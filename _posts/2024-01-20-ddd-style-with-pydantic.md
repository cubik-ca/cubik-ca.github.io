# DDD Style with Pydantic models

The Domain-Driven Design code style is very strongly typed, defining types for each property to differentiate them from
other primitives of the same type. Python allows us to create DDD models very easily and produces the same clean code
that you would find in a compiled-language implementation such as C# or Java.

## The scenario

To demonstrate, I will once again be using the `PhotoVote` concept I've introduced in previous articles. `PhotoVote` is a
simple webapp that allows administrators to post photos that users will rate. It's a non-trivial application domain, but
is sufficiently simple that we can examine most of the code. A quick summary of the `PhotoVote` domain aggregates:

* A `Voter` represents a non-administrative user of the system. We record their name and email as well as whether they
have voted
* A `Ballot` represents a `Voter`'s ratings of the `Candidate`s in each `Competition`.
* A `Competition` is a category within an `Election` containing one or more `Candidate`s.
* A `Candidate` has a name, description, and image

## A Domain Aggregate

There are 5 total domain aggregates (`Ballot`, `Candidate`, `Competition`, `Election`, and `Voter`). Let's have a closer look
at the source code for `Election` so that you can see how simple it is to create strongly-typed DDD aggregates with Pydantic:

```python
from typing import Optional, List

from PhotoVote.Domain import AggregateRoot
from PhotoVote.Domain.Ballot import Ballot
from PhotoVote.Domain.Competition import Competition
from PhotoVote.Domain.Election import ElectionId, ElectionName, ElectionDescription
from PhotoVote.Domain.Voter import Voter


class Election(AggregateRoot[ElectionId]):
    name: ElectionName = ElectionName("")
    description: Optional[ElectionDescription] = None
    competitions: List[Competition] = []
    voters: List[Voter] = []
    ballots: List[Ballot] = []
```

This is Pydantic style, but where is the `BaseModel` inheritance? For that, we will have to dig a little deeper into the
`AggregateRoot` source:

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

TId = TypeVar('TId')


class AggregateRoot(BaseModel, Generic[TId]):
    id: TId
```

That's rather short, but it defines an aggregate as a model containing a property `id` of the specified type. So, for
example, `Election` declares itself to extend `AggregateRoot[ElectionId]`, so the `id` property in this case is of type
`ElectionId`. Let's quickly look at `ElectionId` for completeness:

```python
from PhotoVote.Domain import AggregateId


class ElectionId(AggregateId):
    def __init__(self, value: str):
        super().__init__(value)
```

Pretty straightforward, but once again we see that there's a base class at work, so let's look there too:

```python
from pydantic import model_validator, RootModel, Field
from pydantic_core.core_schema import ValidationInfo
from ulid import ULID


class AggregateId(RootModel):
    root: str = Field(ULID.from_int(0), description="Aggregate ID")

    @model_validator(mode='before')
    def validate_id(self, v: ValidationInfo):
        if not isinstance(self, str):
            raise ValueError("Invalid aggregate id (must be a str)")
        try:
            ULID.from_str(self)
        except ValueError:
            raise ValueError("Invalid aggregate id (must be a valid ULID)")
        return self
```

This base class will validate any `AggregateId` type before assigning its value, by ensuring it is a `str` representation
of a valid ULID. So now we can distinguish between various ULID strings and know what kind of object they identity, and we
cannot easily make the mistake of comparing an `ElectionId` to, say, a `VoterId`. By inheriting from `RootModel`, we solve
the problem of serialization quite nicely.

By composing our domain model in Pydantic, we end up with an easily serializable domain model with all of the validations
provided by Pydantic. This provides clean, minimal code that easily converts to and from JSON, as follows:

```python
election = Election(json.loads(election_json))
election_json = election.model_dump_json()
```

And, as you might expect, you will end up with an Election JSON that looks like:

```json
{
  "id": "SOMEULID",
  "name": "Election Name",
  "description": "Election Description",
  "voters": [],
  "competitions": [],
  "ballots": []
}
```

In summary, Pydantic makes the task of creating the many required user-defined types in a DDD model easy. By inheriting
from `RootModel`, we can create named types for the various primitive properties that make up an `Election`. In the next
article, I will continue the DDD development by introducing the `Event` classes, and show how to implement event sourcing
using Pydantic models.
