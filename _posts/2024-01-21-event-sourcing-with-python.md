# Event Sourcing with Python

This article begins a series to adapt the teachings of Alexey Zimarev's excellent book "Domain-Driven Design with .NET Core".
It is perhaps the earliest work on Event Sourcing I have seen, and it was written in my native .NET. However, I've since
learned Python for various reasons, and I'm working through translating the C# into Python. This has some interesting
challenges that must be overcome, mostly the lack of reflection. However, the use of Pydantic simplifies many things which
makes the Python implementation rather nice to read.

## The process

The process of taking our DDD model and adding support for Event Sourcing is nice and methodical. TLDR:

1. Modify the AggregateRoot class with the methods `when()`, `ensure_valid_state()` (both abstract methods), `apply()`, and
`load()` (written in terms of when())
2. Create the `Event` base class defining a model with a single ULID `id` property
3. Mirror the `Domain` package structure
4. Do Event Storming and put the results in the package structure. Every event will extend `Event`
5. Write `when()` and `ensure_valid_state()` methods for each aggregate

### The updated `AggregateRoot` class

```python
from abc import abstractmethod
from typing import Generic, TypeVar, List
from pydantic import BaseModel

from PhotoVote import Event

TId = TypeVar('TId')


class AggregateRoot(BaseModel, Generic[TId]):
    id: TId
    version: int = -1

    @abstractmethod
    def when(self, event: Event):
        pass

    @abstractmethod
    def ensure_valid_state(self):
        pass

    def apply(self, event: Event):
        self.when(event)
        self.ensure_valid_state()

    def load(self, history: List[Event]):
        for event in history:
            self.when(event)
            version = version + 1
        self.ensure_valid_state()
```

We have added the `when()` and `ensure_valid_state()` abstract methods, indicating our intent to add these methods to
all of our aggregate classes. Additionally we have added the `apply()` and `load()` methods for handling incoming
events. We'll look at the implementation of those methods later on. For now, it's sufficient to know that we call
`apply()` on an aggregate to process an incoming event, and `load()` to load the entire event stream from EventStoreDB.
Most notably, this is a substantial difference from C#. By using Pydantic, the properties of our aggregate root are
public and mutable. I guess Python kind of works on the honor system :) We should not update the properties directly,
but rather only through the `apply()` method, so ensure that every conceivable modification to the object is covered by
your events.

### The event base class

```python
from pydantic import BaseModel, Field, model_validator
from pydantic_core.core_schema import ValidationInfo
from ulid import ULID


class Event(BaseModel):
    id: str = Field(str(ULID.from_int(0)), description="Event ID")

    @model_validator(mode='before')
    def validate_event_id(self, v: ValidationInfo):
        if not isinstance(self, str):
            raise ValueError("Event ID must be a string")
        try:
            ULID.from_str(self)
        except ValueError:
            raise ValueError("Event ID must be a valid ULID")
```

Like `AggregateRoot`, we are simply defining a model class that contains a single property, `id`, which must be a valid
string representation of a ULID. This is an arbitrary decision, but it does address the need for the client to be able
to generate unique event ids. We don't currently use the event ids for anything more than providing an identifier to
distinguish one event from another. One possibility is that the event id could be used for dedupe at the API level.

### Mirror the `Domain` package structure

The package structure for the Domain provides us with the necessary structure to aid our Event Storming process. An
event specifically targets one or more aggregates. By knowing what aggregate an event is primarily associated with, we
can better organize our events and keep the Event Storming process from becoming unmanageable. Also note that an event
can affect more than one aggregate. In that case, you have some choice about how to organize, but it should generally be
reasonably obvious which aggregate an event "belongs" to.


### Event Storming

Now that we have a structure in place to record our events, we simply need to make some classes that extend event. The
name should be descriptive of the event (e.g. `BallotCreated`, `CompetitionAdded`), and the properties of the event are
all of the required values needed for the aggregates to process the event, for example:

```python
from PhotoVote.Event import Event

class BallotCreated(Event):
    election_id: str
    ballot_id: str
    voter_id: str
```

In order to insert a new ballot record in the `Election` aggregate, we need to know which election the ballot belongs
to, the unique identifier for the new ballot, and the voter that created the ballot (so we can verify their
registration). Unfortunately, this provides an audit trail that could be used to trace votes to voters, but it won't be
stored in the cache. A proper public voting application would have to overcome this link by using some impressive
cryptography.

Event properties are always primitives. While we use DDD classes in the domain model, we don't need to keep the event
within the application domain once it is persisted to EventStoreDB.

### The `when()` and `ensure_valid_state()` methods

The last step is to process all of the events. This is accomplished by overriding (as you must) the `when()` and
`ensure_valid_state()` methods. The `when()` method is simply an `if`-`elif` chain that handles each event type
individually, like this:

```python
class Ballot(AggregateRoot[BallotId]):
    votes: Dict[CompetitionId, Dict[CandidateId, Rating]]
    is_cast: bool = False
    
    def __init__(self, **data):
        super().__init__(**data)
        self.votes = {}

    def when(self, event: Event):
        if isinstance(event, BallotCreated):
            self._handle_ballot_created(event)
        elif isinstance(event, BallotCandidateRated):
            self._handle_ballot_candidate_rated(event)
        else:
            # This is a matter of some debate, too. Do we simply ignore events that aren't intended for us? Or is it
            # a program error to submit events to aggregates that don't process them?
            print(f"WARN: Unknown event type {format(type(self))} received by {self.__class__.__name__}-{self.id}")
            pass

    def ensure_valid_state(self):
        if self.id == AggregateId.empty_ulid():
            raise InvalidStateError("Ballot ID is required")
        if self.is_cast and self.votes.__len__() == 0:
            raise InvalidStateError("Cannot cast an empty ballot")

    def _handle_ballot_created(self, created: BallotCreated):
        self.id = BallotId(created.ballot_id)

    def _handle_candidate_rated(self, rated: BallotCandidateRated):
        if rated.competition_id not in self.votes:
            self.votes[rated.competition_id] = {}
        self.votes[rated.competition_id][rated.candidate_id] = Rating(rated.rating)
```

This is not quite complete, as it doesn't (e.g.) check to see if a ballot has been cast before accepting a candidate
rating. But this is the framework that all aggregates must adhere to for the event sourcing mechanism to work. In a 
production setting, it would be important to test the domain model and event sourcing by ensuring that events produce 
the correct domain state. For the purposes of this demonstration, however, I will forego testing.

It doesn't seem like much to rewrite traditional getters and setters with this framework, as it is undoubtedly more
verbose. However, the payoff is coming in the next section when we start writing Event Store workers and start receiving
events through Memphis.dev. It's not as obvious in Python, but in C# this mechanism is publicly enforced, not even
allowing for properties to be updated. In Python, as I've observed, this is simply a matter of good behaviour.

Thanks for following along thus far! I promise it will get more exciting when we bring in the API and infrastructure and
show how it all fits together.
