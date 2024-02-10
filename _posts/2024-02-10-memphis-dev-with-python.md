# Using Memphis.dev Message Broker with Python

Memphis.dev is a new cloud-based message broker that seeks to replace existing
RabbitMQ and Apache Kafka installations. It has SDK support for all major
programming languages and is very easy to use. We'll continue with the
`PhotoVote` application and create the necessary producers to send events from
our system to Memphis.dev.

## The PhotoVote API
The PhotoVote API implementation is rather straightforward, as all we really
need are the endpoint definitions and a little bit of code to forward the events
to Memphis. Once the events arrive in Memphis, a worker program will receive the
events and write them to EventStoreDB, from where another worker will receive
events and write to MongoDB. Let's look at the main API program first:

```python
# api.py

from fastapi import FastAPI
from server.routers.ballot import router as ballot_router
from server.routers.candidate import router as candidate_router
from server.routers.competition import router as competition_router
from server.routers.election import router as election_router
from server.routers.voter import router as voter_router
from dotenv import load_dotenv

router = FastAPI()

load_dotenv(".env")
router.include_router(ballot_router, prefix="/ballot")
router.include_router(candidate_router, prefix="/candidate")
router.include_router(competition_router, prefix="/competition")
router.include_router(election_router, prefix="/election")
router.include_router(voter_router, prefix="/voter")
```
I think this code is quite self-explanatory. We import all of our routers, and
attach them to the main FastAPI instance. Before we do that, we load the
environment variables from `.env`.

We can look at one of the routers in more detail but it, too, is quite simple:

```python

from fastapi import APIRouter
from fastapi.responses import JSONResponse

from PhotoVote.Event.Voter import VoterRegistered
from .Router import Router

router = APIRouter()
voter_router = Router("voter", ["election"])


@router.post("/")
async def voter_registered(registered: VoterRegistered) -> JSONResponse:
    await voter_router.publish_event(registered)
    return JSONResponse(status_code=200, content={})
```

As you can see, the bulk of the work is being done by the `publish_event` method
in `Router`, so let's look at that:

```python
from typing import List

from memphis import Memphis, Headers
from uuid import uuid4

from PhotoVote.Event import Event
import os


class Router:
    _name: str
    _stations: List[str]
    _producer: Memphis = None

    def __init__(self, name: str, stations: List[str]):
        self._name = name
        self._stations = stations
        self._memphis: Memphis = Memphis()

    async def publish_event(self, event: Event):
        if self._producer is None:
            memphis: Memphis = Memphis()
            await memphis.connect(
                host=os.getenv('MEMPHIS_HOST'),
                username=os.getenv('MEMPHIS_USERNAME'),
                password=os.getenv('MEMPHIS_PASSWORD'),
                account_id=int(os.getenv('MEMPHIS_ACCOUNT_ID'))
            )
            for station in self._stations:
                await memphis.station(station)
            self._producer = await memphis.producer(self._stations, f'{self._name}-{uuid4()}')
        headers = Headers()
        headers.add('Event-Type', event.__class__.__module__)
        await self._producer.produce(message=event.model_dump(), headers=headers)
```

OK, now we have some actual code! Let's see what it does. First, we take a base
name for the producer (actual producers will have a UUID appended to it).
Individual producers will have unique names so that we can see all the instances
that are attached in the Memphis.dev console. We also take a list of stations to
which we will publish. This application only requires one station (since we will
only record entire `Election` documents in MongoDB). It would be entirely
possible to publish some or all events to another station, where they could be
picked up by another application. In this case, you would just add the
additional station to the initialization of `Router`.

Next, we lazily initialize a Memphis connection. Here, we will read the values
that were in .env, and call `connect()`. Once connected, we will create any
stations that are required, and then create a producer for those stations.

Now this code can publish events! Calling the `publish_event` method will
publish any `Event` subclass to Memphis and record its type so that it can be
reconstructed by the consumer.

That's it! Creating the producer is extremely simple. There will be a router for
each aggregate type. Each router will have an endpoint for each event type that
is handled by that aggregate. And each endpoint method will use the `Router`
class to publish its event to Memphis. The next article in this series will
examine the EventStoreDB worker that will write events into separate streams for
each aggregate instance, and then one last article will examine how we listen to
EventStoreDB for updates and create the corresponding update in MongoDB.
