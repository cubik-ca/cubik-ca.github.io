# Caching the Event Stream

It's taken a bit of effort to get the last details of the PhotoVote application sorted out, but I've made only a few
minor modifications from what I've discussed thus far, and most of it is not affecting the API. So far, we have discussed
Event Sourcing and DDD, and shown how to publish events from our FastAPI application to the Memphis.dev message broker.

## EventStoreDB

From the message broker, our events will be pulled into any workers that are listening on the same station.
Our first order of business is to put the events into EventStoreDB. This serves two purposes: to persist the events
into long-term storage, and to organize events by aggregate. EventStoreDB is a niche database that is specifically
designed for exactly this task. Once it finishes storing the event, it will further propagate it to any EventStoreDB
subscribers. In our case, we have only a single EventStoreDB subscriber, the CosmosDB worker, which we will discuss
later.

### Receiving events from Memphis.dev

Each worker will need to be a Memphis.dev consumer. Setting up a consumer is done as follows:

```python
from typing import TypeVar, List, Callable, Generic, Coroutine, Any
from uuid import uuid4

from aiomisc import Service
from memphis import Memphis, MemphisError
from memphis.consumer import Consumer
from memphis.message import Message


class MemphisWorker(Service):
    _memphis: Memphis
    _consumer: Consumer
    _aggregate_name: str
    # we handle MemphisError here, and we got the context from the caller, so there's no need to send either
    _handler: Callable[[List[Message]], Coroutine[Any, Any, None]]

    def __init__(self,
                 aggregate_name: str,
                 # this is a mouthful, but you are looking for a signature that looks like:
                 # async def process_messages(msgs: List[Message]) -> None
                 handler: Callable[[List[Message]], Coroutine[Any, Any, None]],
                 memphis: Memphis):
        super().__init__()
        self._handler = handler
        self._memphis = memphis
        self._aggregate_name = aggregate_name.lower()

    async def process_messages(self, messages: List[Message], error: MemphisError, context: Any):
        # error will be set in case the last consume() timed out. we don't need to take any action in this case
        # as the consume() has been scheduled periodically in the application startup.
        if not error:
            await self._handler(messages)

    async def start(self):
        await self._memphis.connect(
            host=self.config.memphis_host,
            account_id=self.config.memphis_account_id,
            username=self.config.memphis_username,
            password=self.config.memphis_password,
        )
        consumer_name = f"{self._aggregate_name}-{uuid4()}"
        # self._memphis.consumer() doesn't return Consumer for some reason.
        # noinspection PyTypeChecker
        self._consumer = await self._memphis.consumer(station_name=self._aggregate_name,
                                                      consumer_name=consumer_name,
                                                      batch_size=10,
                                                      batch_max_time_to_wait_ms=1000,
                                                      pull_interval_ms=100,
                                                      consumer_group=self._aggregate_name)
```

This class is the base class for any worker that consumes messages from Memphis.dev. It extends the `aiomisc.Service` class
so that it can run forever in an event loop. An async method is provided to process the message, and the `start()` method
connects to the Memphis.dev message broker and starts consuming messages.

Given this framework, the next step is to create a worker that stores messages in EventStoreDB. From there, subscribers
will receive the events with the knowledge that they are now persisted and thus represent the current model state. The
worker, then, has only this method:

```python
    async def _process_events(self, msgs: List[Message]) -> None:
        for msg in msgs:
            try:
                data = msg.get_data().decode("utf-8")
                obj = json.loads(data)
                if not obj["id"]:
                    raise ValueError("Event id is required")
                headers = msg.get_headers()
                event = self._create_event_instance(headers.get("Event-Type"), obj["id"], obj)
                await self.write_to_eventstore(event)
                await msg.ack()
            except Exception as ex:
                print(ex)
                await msg.nack()
```

We receive a list of messages from the broker, decode them from UTF-8 into a JSON string, and then convert the JSON to a
dictionary. Once we have determined that the event is valid (by reconstructing its object state), we write the event to
EventStoreDB. Let's look a little closer at `_create_event_instance()` and `write_to_eventstore()`.

Creating an event instance from the JSON is a handy utility function that we can use to restore the object model state
from raw JSON. Likewise, we also have similar utility methods for creating aggregates from their JSON. This is the code:

```python
    def _create_event_instance(self, class_name: str, event_id: str, event_dict: Dict) -> Event:
        event_type = self._get_type(class_name)
        event = event_type(id=event_id)
        event = event.model_validate(event_dict)
        return event
```

The `event_dict`, of course, comes from `json.loads()`.

Finally, the `write_to_eventstore()` method:

```python
        async def write_to_eventstore(self, event: Event) -> None:
        try:
            election_id = getattr(event, 'election_id')
            if election_id is None:
                raise ValueError("No election id in event")
            # All events fall under the Election aggregate root. There is no need to also write the
            # other aggregates to their own streams, since they don't stand on their own outside the
            # context of the election to which they belong.
            stream_name = f'Election-{election_id}'
            # There's not much point in loading the event stream since we'll end up just taking the count
            # of items in there as authoritative. Therefore, the fastest approach is to append to the end of the
            # stream blindly. If there were additional validation sources, we could load the aggregate here and
            # do validations here before appending to the stream.
            self.esdb.append_to_stream(stream_name, current_version=StreamState.ANY,
                                       events=[NewEvent(
                                           id=uuid4(),
                                           data=event.model_dump_json().encode('utf8'),
                                           type=event.__class__.__module__)])
        except Exception as ex:
            raise RuntimeError(f"Error: Unable to write to eventstore: {ex}")
```

We simply append the event to the end of a stream named `Election-(id)`. That concludes the EventStoreDB Worker. It is
intended to do as little processing as possible to reduce the latency introduced by doing so. However I believe that
the benefits of using EventStoreDB outweigh the small latency. In the next section, we will cover the last piece of the
application, the CosmosDB worker.

## CosmosDB Worker

The CosmosDB worker will listen for events being stored in EventStoreDB, and then translate the event to a change within
CosmosDB. For example, receiving an `ElectionCreated` event would cause the election document to be created in CosmosDB.
Similarly, receiving an `ElectionNameChanged` event would cause the election document to have its name updated. If you
have followed along thus far, you should see that we can use the domain models directly with CosmosDB. This is the generic
handler we use to apply an event to an aggregate and store it in CosmosDB:

```python
from typing import Optional

from azure.cosmos.aio import ContainerProxy
from azure.cosmos.exceptions import CosmosResourceNotFoundError

from PhotoVote.Domain.Election import Election, ElectionId
from PhotoVote.Event import Event


class Handler:
    _container: ContainerProxy

    def __init__(self, container: ContainerProxy):
        self._container = container

    async def _get(self, election_id: str) -> Optional[Election]:
        try:
            election_dict = await self._container.read_item(election_id, election_id)
            election = Election.model_validate(election_dict)
            return election
        except CosmosResourceNotFoundError:
            return None

    async def _upsert(self, election: Election) -> None:
        election_dict = election.model_dump()
        await self._container.upsert_item(election_dict)

    async def _delete(self, election_id) -> None:
        await self._container.delete_item(election_id, election_id)

    async def handle(self, event: Event):
        election = await self._get(event.election_id)
        if election is None:
            election = Election(id=ElectionId(event.election_id))
        election.apply(event)
        if not election.deleted:
            await self._upsert(election)
        else:
            await self._delete(election.id)
```

It seems shocking, but this is all the database code you need. This code is capable of all required operations ("CRUD") for
any aggregate that extends `Aggregate`. The reasoning is that the document cache always contains the current state of the
aggregate as stored in EventStoreDB. Therefore it is only necessary to apply the incoming event to the existing aggregate
to get its current state. So, a simple load-update-commit pattern works well.

### Multi-threading

The PhotoVote application only requires one aggregate root, so I have deliberately created a multi-threaded application to
show how additional aggregate roots could be handled by the same EventStoreDB and CosmosDB workers:

```python
from aiomisc import entrypoint
from memphis import Memphis

from workers.MemphisWorker import MemphisWorker
from workers.eventstore.Worker import Worker
import threading

memphis: Memphis = Memphis()


def start_worker(worker: MemphisWorker):
    with entrypoint(worker) as loop:
        loop.run_forever()


if __name__ == "__main__":
    # create additional workers here. only the aggregate name argument is different
    election_worker = Worker('Election', memphis)
    # create and start additional aggregate worker threads here
    t1 = threading.Thread(target=start_worker(election_worker), daemon=True)
    t1.start()
    # will block forever, ensure all executable code has been put into a thread
    t1.join()
    # join any remaining threads here

```

This is the startup program for the EventStoreDB worker. As you can see, it is possible to create additional workers and
start additional threads for each aggregate root.

## Conclusion

That's it! I've shown now excerpts from every component in the PhotoVote application. Let's recap everything that we've done:

1. We created a domain model using DDD principles and Event Sourcing techniques.
2. We implemented the domain model using Pydantic to ease the burden of serialization and validation.
3. We created a FastAPI application that publishes events to Memphis.dev.
4. We created a worker that consumes events from Memphis.dev and stores them in EventStoreDB.
5. We created a worker that consumes events from EventStoreDB and stores them in CosmosDB.

This architecture is designed to be rapidly extended. As you can see, adding new functionality is accomplished by adding
the functionality to the domain model in the form of events. The FastAPI application is then updated with the desired
REST API endpoints, and each endpoint set to publish its corresponding event to Memphis.dev. You do not need to write
any code to publish events to EventStoreDB or CosmosDB, as the workers presented in the PhotoVote repository can used
virtually verbatim (with the exception of using a different domain model).

Therefore, the detailed process for adding new functionality is as follows:

1. Model the new functionality as a series of events and updates to domain models.
2. Implement any new aggregates as subclasses of `Aggregate`. If there is a new aggregate root, ensure that you also
   add a new thread to handle that aggregate root in both the EventStoreDB and CosmosDB workers.
3. Add all desired REST API endpoints to the FastAPI application as methods that publish events to Memphis.dev.
4. Add all desired GET methods to the FastAPI application to read the data back from CosmosDB.
