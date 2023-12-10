---
layout: post
title: A New Application Architecture
author: Brian Richardson <brian@cubik.ca>
description: Cloud-Native Event-Driven Architecture
date: 2023-12-09T21:51:00Z-07
categories: architecture
published: true
---
# A New Application Architecture
## Cloud-Native Event-Driven Architecture

You would be forgiven for thinking that the subtitle is nothing more than buzzwords and
vaporware. But I assure you it is not. This article will present my vision for a new (and
affordable!) application architecture that will give great returns in resiliency, scalability,
performance, responsiveness, maintainability, and user satisfaction. What I am selling is
not pie in the sky; it really works!

### PaaS and SaaS, not IaaS

The journey to Cloud-Native has been a long one. This is not something that will happen
overnight. For our organization, it has been a journey that has taken more than 6 years now,
and is only now starting to change from the straight "lift-and-shift" approach that gets
into the cloud. Now that most of our infrastructure is there, we can start replacing servers
with special-purpose services. For example, this article will make reference to Memphis.dev,
a SaaS message broker that replaces an entire RabbitMQ cluster. That cluster currently
requires 3 VMs that must be maintained on an ongoing basis: Erlang releases, RabbitMQ releases,
Linux patching. So the savings of replacing that cluster with a SaaS offering goes beyond
just the monetary savings (of which there is a modest amount), but also frees up a little
bit of sysadmin time too every month.

### Scalability

As an architect, perhaps my biggest challenge is preparing for large releases and new business.
I worry that when the business comes to me and asks about new business opportunities that I
will be limited by the existing infrastructure and architecture, and that no amount of
throwing money at the problem will solve it faster. There is one very good solution, and
that is to build scalable applications from the beginning.

### Resiliency
:was
How many of your applications run with little to no redundancy? Are there databases in your
organization that would cause irreparable harm if they were to be corrupted or damaged?
How frequent are your planned maintenance outages? How about your unplanned ones? All of these
questions refer to resiliency, the ability to withstand difficult conditions, such as network
outages, hardware failures, viruses and malware, etc.

## Architecture Diagram

![Cloud-Native Application Architecture 2024](/assets/images/cloud-native-app-architecture-2024.svg "Cloud-Native Application Architecture 2024")

This diagram shows all of the application components and PaaS/SaaS offerings used by a simple web-based voting application
written using this architecture. I'll review each of the components in turn below.

### Kubernetes

Containerization is a key component of cloud-native architectures. The ability to deploy new VMs quickly is critical to
the success of such an application. Kubernetes is what is called a _container orchestrator_. It manages _nodes_ and creates
or destroys nodes as resource requirements (and specified limits) require. By orchestrating containers in an autoscaling
mode, the orchestrator is capable of managing compute costs far better than any automation you might write for your IaaS.

At a lower level, a _node_ is home to many _pods_, a running container image. Each pod is analagous to a docker image you
might run on your local workstation. Nodes can be further organized into _node pools_ (depending on your Kubernetes
implementation). In such implementations, node pools can have their own properties. For example, in Azure Kubernetes
Services, it is possible to build nodes of different types (e.g. ARM64, different memory sizes) for each node pool.

Kubernetes is a huge topic, and I won't spend a lot of time talking about it. But it is at the heart of this architecture, and
its cost savings are entirely dependent on the ability of the orchestrator to properly manage compute resources.

### Redis

Redis serves as shared memory for the entire cluster. While it can be used to share data between application components,
that is not the intended usage. Redis is typically used to store state information for application sessions, provide
realtime notification systems and various support use cases along those lines. It's not a critical part of the infrastructure,
but it is noteworthy in that many application components rely on having session state available.

### Memphis.dev
[Memphis.dev](https://memphis.dev) provides a modern message broker with pub/sub conventions over websockets. It is in early
development still, but shows a ton of promise. It was really easy to spin up a new broker, configure it with S3 storage to
tier off older messages from the broker, and create a new _station_ (roughly analagous to a Kafka or Pulsar topic). The code,
too, is very easy to write (more on that later). It is expected that Memphis will provide AMQP at some point, so eventually
it will be a complete replacement for [RabbitMQ](https://rabbitmq.org).

### EventStore DB

[EventStoreDB](https://eventstore.com) is a database dedicated to storing event-sourced aggregates in individual streams.
If that sounds niche, you're right. But it's a good thing it exists, because having the ability to subscribe to a filtered
subset of all events in the system is ideal for your database workers. I will spend more time on the code later, but you
should consider this your master data source. It is long-term immutable storage that contains every event your applications
have ever processed. In disaster scenarios, it will be possible to replay events from here to recreate databases, etc. Audit
will really like this, too, since an event contains audit information at a level of detail that will make even auditors
smile 😂.

### MongoDB

[MongoDB](https://mongodb.com) is a popular _document database_, a database dedicated to storing JSON documents. Effectively, this makes a document database object storage (though an object is more than just its state). Storing objects in their native
structure reduces code complexity as there is no more need to translate objects to relational storage. While there are good
libraries (such as `EntityFramework`) that do this relatively transparently, it is still extra processing that can be
eliminated. The diagram above refers to a service called `Azure MongoDB for CosmosDB`. This service uses CosmosDB
infrastructure, but provides the MongoDB 4.2 wire protocol. This seems to be the best option for Azure customers. For others,
there is the original `MongoDB Atlas`.

## The Application

To illustrate the use of all of these components, I have developed a small voting application that allows users to vote
on their favourite photographs. It's sufficiently complex to demonstrate the benefits of this architecture, but easy to
understand. I will walk through the development process now so you can see how Domain-Driven Design and Event Storming
activities contribute to the overall solution, and produce a final product that lives up to the hype I've given it. I will
share the entire application at a later time, but for now I'll just discuss individual components and highlight key
code.

### Domain-Driven Design

The Voting domain is relatively straightforward. `Voters` cast `Ballots` that contain `Ratings` for `Candidates` in one or
more `Competitions`. These aggregates are organized under a single `Election` aggregate. In this domain, Voters, Candidates,
Competitions, and Ballots are all `Aggregate Roots`, while the Rating is a `Value Object`. Let's look at the code for
`AggregateRoot` first:

```c#
public abstract class AggregateRoot<TModel, TId>(AggregateId<TId> id) : AggregateRoot
    where TModel : AggregateRoot<TModel, TId>
    where TId : AggregateId<TId>, IEqualityComparer<TId>, new()
{
    [BsonId]
    public AggregateId<TId> Id { get; protected set; } = id;
}
```

This is a short class, but it contains a lot more than it might seem. Let's start by looking at the type parameters,
`TModel` and `TId`. TModel is exactly what you'd expect, a POCO containing all properties for the model. Thus, we use
`AggregateRoot<TModel, TId>` as a _decorator_. Let's look a little closer at that TId parameter, which extends
`AggregateId<T>`. Here's the code for `AggregateId`:

```c#
public abstract class AggregateId<T> : Value<T> where T : AggregateId<T>, IEqualityComparer<T>, new()
{
    protected Ulid Value { get; }

    protected AggregateId(Ulid? value)
    {
        if (!value.HasValue) throw new ArgumentNullException(nameof(value), "The Id cannot be empty");
        Value = value.Value;
    }

    protected AggregateId(string? value)
    {
        if (value is null or null) throw new ArgumentNullException(nameof(value), "The Id cannot be empty");
        Value = Ulid.Parse(value);
    }

    // `Ulid` raw value
    public static implicit operator Ulid?(AggregateId<T>? self) => self?.Value;
    
    // `Ulid` represented as `string`
    public static implicit operator string?(AggregateId<T>? self) => self?.Value.ToString();

    // interchangeable with T
    public static implicit operator T(AggregateId<T> self) => (T) self;

    // `Ulid` represented as `string`
    public override string? ToString() => Value.ToString();
}
```

The Id property is basically a `Value` object, but it is of a specific type (here, a [ULID](https://github.com/ulid/spec)).
It contains the implicit operators that allow it to be treated equally as a ULID, a string, and its base type. One other
notable feature of the Id objects is that they can be compared for equality. Let's look at the `Value` code as well:

```c#
public class Value<T> where T : Value<T>, IEqualityComparer<T>
{
    // ReSharper disable once StaticMemberInGenericType - it is private and not called from outside this file
    private static readonly Member[] Members = GetMembers().ToArray();

    // two value objects are identical if all primitive fields and properties are equal
    public override bool Equals(object? obj)
    {
        if (obj is null) return false;
        if (ReferenceEquals(this, obj)) return true;
        return obj.GetType() == typeof(T) && Members.All(
            m =>
            {
                var objValue = m.GetValue(obj);
                var thisValue = m.GetValue(this);
                return m.IsNonStringEnumerable
                    ? GetEnumerableValues(objValue).SequenceEqual(GetEnumerableValues(thisValue))
                    : objValue?.Equals(thisValue) ?? thisValue == null;
            });
    }

    // ... omitted GetEnumerableValues()
    // ... omitted GetMembers()

    // a pseudo-random hashing function
    public override int GetHashCode()
    {
        return CombineHashCodes(Members.Select(m => m.IsNonStringEnumerable
                    ? CombineHashCodes(GetEnumerableValues(m.GetValue(this)))
                    : m.GetValue(this)));
    }

    // a pseudo-random function
    private static int CombineHashCodes(IEnumerable<object?> values)
    {
        return values.Aggregate(17, (current, value) => current * 59 + (value?.GetHashCode() ?? 0));
    }

    // value objects can be compared for equality
    public static bool operator ==(Value<T>? left, Value<T>? right)
    {
        return Equals(left, right);
    }

    public static bool operator !=(Value<T>? left, Value<T>? right)
    {
        return !Equals(left, right);
    }

    // implicitly convert to string by calling ToString(); this isn't useful in the base class since it returns
    // a pretty-printed string representation of the entire object. But other subclasses might do something different
    // so we'll allow the implicit conversion for comparisons
    public static implicit operator string?(Value<T>? value) => value?.ToString();

    // ... omitted ToString()

    // ... omitted struct Member
}
```

This code mostly ensures that Value Objects are immutable and equatable. Two value objects are considered equal exactly when
all property values are equal. Between the code in Value and AggregateId, we have a non-null equatable key that can be
used to identify an individual aggregate.

There is one final bit of code to look at before proceeding with building the domain model: the abstract, non-generic
`AggregateRoot` class:

```c#
public abstract class AggregateRoot
{
    // the current version of this aggregate root. must match the current version in the event store.
    public int Version { get; private set; } = -1;
    // handle an incoming event
    protected abstract void When(Event @event);
    // validate the state after every event
    protected abstract void EnsureValidState();

    // apply an event to the aggregate
    public void Apply(Event @event)
    {
        When(@event);
        EnsureValidState();
        Version++;
    }

    // load the history of events from the event store
    public void Load(IEnumerable<Event> history)
    {
        foreach (var ev in history)
        {
            When(ev);
            Version++;
        }
    }
}
```

This class defines the abstract methods `When()` and `EnsureValidState()`. A `When()` implementation includes event
handlers for every event to which your domain class responds. `EnsureValidState()` can be used to throw an exception
after every event is applied if the resulting state is not valid. For example, it is necessary for domain classes to
provide a public parameterless constructor. This is accomplished by initializing the Id property with an empty ULID.
Clearly, this is an invalid state, so we want to ensure that the Id field is valid in this method.

With the description of the domain and these three base classes, we can now proceed to create our domain classes. Here's
an example of the `Election` class:

```c#
public class Election(ElectionId id, ElectionName name, ElectionDescription? description = null, ElectionStartDate? startDate = null, 
    ElectionEndDate? endDate = null) : AggregateRoot<Election, ElectionId>(id)
{
    public ElectionName Name { get; protected set; } = name;
    public ElectionDescription? Description { get; protected set; } = description;
    public ElectionStartDate? StartDate { get; protected set; } = startDate;
    public ElectionEndDate? EndDate { get; protected set; } = endDate;

    public List<Competition> Competitions { get; } = [];
    public List<Voter> Voters { get; } = [];
    public List<Ballot> Ballots { get; } = [];
    
    protected override void When(Common.Event.Event @event)
    {
        switch (@event)
        {
            case ElectionNameChanged ev:
                ChangeElectionName(ev.Name);
                break;
            case ElectionDescriptionChanged ev:
                ChangeElectionDescription(ev.Description);
                break;
            case ElectionStarted ev:
                StartElection(ev.StartDate);
                break;
            case ElectionEnded ev:
                EndElection(ev.EndDate); 
                break;
            case VoterVoted ev:
                RegisterVoter(ev.Id, ev.Name, ev.Email);
                IssueBallot();
                break;
            case CompetitionAdded ev:
                AddCompetition(ev.Id, ev.Name, ev.Description);
                break;
            case CompetitionRemoved ev:
                RemoveCompetition(ev.Id);
                break;
            case ElectionCreated ev:
                InitializeElection(ev.Id, ev.Name, ev.Description);
                break;
        }
    }

    private Election(ElectionId electionId) : this(electionId, ElectionName.FromString(string.Empty)) {}

    public Election() : this(ElectionId.FromUlid(Ulid.Empty)) {}

    private void InitializeElection(string electionId, string n, string? d)
    {
        Id = ElectionId.FromString(electionId);
        Name = ElectionName.FromString(n);
        Description = d == null ? null : ElectionDescription.FromString(d);
    }

    private void ChangeElectionName(string n) => Name = ElectionName.FromString(n);

    private void ChangeElectionDescription(string? d) =>
        Description = d != null ? ElectionDescription.FromString(d) : null;

    private void StartElection(DateTime sd) => StartDate = ElectionStartDate.FromDateTime(sd);

    private void EndElection(DateTime ed) => EndDate = ElectionEndDate.FromDateTime(ed);

    private void RegisterVoter(string voterId, string n, string email)
    {
        var voter = new Voter(VoterId.FromString(voterId),
            VoterName.FromString(n),
            VoterEmail.FromString(email));
        Voters.Add(voter);        
    }

    private void AddCompetition(string competitionId, string n, string? d)
    {
        var competition = new Competition(
            CompetitionId.FromString(competitionId),
            CompetitionName.FromString(n),
            CompetitionDescription.FromString(d));
        Competitions.Add(competition);
    }

    private void RemoveCompetition(string competitionId)
    {
        var theId = CompetitionId.FromString(competitionId);
        var competition = Competitions.SingleOrDefault(c => c.Id == theId);
        if (competition == null) return;
        Competitions.Remove(competition);
    }

    private void IssueBallot()
    {
        var ballot = new Ballot(BallotId.FromUlid(Ulid.NewUlid()));
        Ballots.Add(ballot); 
    }

    protected override void EnsureValidState()
    {
        if (Id == Ulid.Empty) throw new InvalidOperationException("Election Id has no value");
        if (Name == string.Empty) throw new InvalidOperationException("Election Name is blank");
    }
}
```

### Event Storming

Populating the `When()` method is best done using an activity called _Event Storming_. This activity is generally as you'd
expect: to attempt to enumerate all of the events that can be applied to a domain object. The event storming activity
should take place immediately following DDD so that the `When()` method can be written as well. Events are straightforward
as well:

```c#
public abstract class Event(string id)
{
    public string Id { get; } = id;
}
```

That's it! An event is related to a single aggregate identified by its id field. Note that the Event project must use
primitives; it cannot refer to the domain project so that a circular reference is not created. An event contains all of
the information required for the domain object to apply the event. But they are simply POCOs:

```c#
public class ElectionCreated(string id, string name, string? description = null) : Common.Event.Event(id)
{
    public string Name { get; } = name;
    public string? Description { get; } = description;
}
```

### Worker Services

The base worker service is quite involved, so I will walk through it a method at a time.

```c#
public abstract class WorkerBase<T,TId>(MemphisClient memphis, IOptions<EsdbOptions> esdbOptions) : BackgroundService 
    where T : AggregateRoot<T,TId>, new() 
    where TId : AggregateId<TId>, IEqualityComparer<TId>, new()
{
    private EventStoreClient? _esdb;
    private readonly EsdbOptions _esdbOptions = esdbOptions.Value;
    private UserCredentials? _credentials;

    // ... omitted
}
```

Here, we define a generic worker that will consume events from Memphis and output events to an EventStore stream. We receive
a copy of all events that are sent to this application's station. The method `ProcessMessage()` is called for every received
message from Memphis:

```c#
private async Task ProcessMessage(MemphisMessage message)
{
     var data = message.GetData();
     var clrType = message.GetHeaders()["Clr-Type"];
     if (clrType == null) return;
     var assembly = Assembly.Load("Vote.Event");
     var eventType = assembly.GetType(clrType);
     if (eventType == null) return; 
     var json = Encoding.UTF8.GetString(data);
     if (eventType.Namespace != "Vote.Event") return;
     var @event = ExtractEvent(json, eventType);
     if (@event == null) return;
     var aggregate = await LoadAggregate(@event.Id);
     aggregate?.Apply(@event);
     var streamName = $"voteapp-{typeof(T).Name.ToLower()}-{@event.Id}";
     var eventData = JsonConvert.SerializeObject(@event);
     var metadata = JsonConvert.SerializeObject(new { ClrType = clrType });
     if (_esdb == null) return;
     if (aggregate == null) return;
     await _esdb.AppendToStreamAsync(streamName, 
         aggregate.Version == 0 ? StreamRevision.None : StreamRevision.FromInt64(aggregate.Version - 1), 
         userCredentials: _credentials, 
         eventData: new List<EventData> {
             new (Uuid.NewUuid(), @event.GetType().Name, 
             Encoding.UTF8.GetBytes(eventData), 
             Encoding.UTF8.GetBytes(metadata))
         });
     message.Ack();        
}
```

The message data is a JSON representation of some event in the `Vote.Event` namespace. The exact type is stored in the
`Clr-Type` message header. As this handler doesn't currently accept events outside this namespace, we can simply return
if the event did not originate from this application. Given the JSON and the CLR type, it is an easy task to produce an
object of that type from the JSON data. The details of that are in `ExtractEvent()`:

```c#
private Common.Event.Event? ExtractEvent(string json, Type type)
{
    if (type.Namespace != "Vote.Event") return null;
    return JsonConvert.DeserializeObject(json, type) as Common.Event.Event;
}
```

This service will work with the master data. Therefore, it loads the entire event history for the aggregate from EventStore
before applying and persisting the new event:

```c#
private async Task<T?> LoadAggregate(string id)
{
    var streamName = $"voteapp-{nameof(T).ToLower()}-{id}";
    UserCredentials? credentials = _esdbOptions.Username != null && _esdbOptions.Password != null
        ? new UserCredentials(_esdbOptions.Username, _esdbOptions.Password)
        : null;
    EventStoreClient.ReadStreamResult stream;
    var s = _esdb?.ReadStreamAsync(Direction.Forwards, streamName, StreamPosition.Start,
        userCredentials: credentials);
    if (s == null) return null;
    stream = s;
    var aggregate = Activator.CreateInstance<T>();
    var history = new List<Common.Event.Event>();
    if (await stream.ReadState == ReadState.Ok)
        await stream.ForEachAwaitAsync(ev =>
        {
            var eventJson = Encoding.UTF8.GetString(ev.Event.Data.ToArray());
            var metadataJson = Encoding.UTF8.GetString(ev.Event.Metadata.ToArray());
            dynamic? metadata = JsonConvert.DeserializeObject(metadataJson);
            var assembly = Assembly.Load("Vote.Event");
            if (metadata == null) return Task.CompletedTask;
            var type = assembly.GetType(metadata.ClrType.ToString());
            if (type == null) return Task.CompletedTask;
            Common.Event.Event @event = ExtractEvent(eventJson, type);
            history.Add(@event);
            return Task.CompletedTask;
        });
    aggregate.Load(history);
    return aggregate;
}

```

Note that all aggregate roots require a public parameterless constructor. This means that you will need to initialize some
fields with undesirable default values, so ensure that you update your `EnsureValidState()` method accordingly. To illustrate,
consider what will happen if the aggregate root does not exist. No event will ever be applied to initial object, which
was created with `Activator.CreateInstance<T>()`. In other words, `new T()`. If you examine the code for `Election`, you will see that the public parameterless constructor provides `Ulid.Empty` to the base class, and thus is in an invalid state from
the beginning. This will only cause an error if an event is applied to it that results in an invalid state, so ensure that
your event handlers consider this scenario too. In this case, it is necessary to also assign the `Id` property from the
event, since it is current `Ulid.Empty`.

It's worth noting how simple the main method is for this service:

```c#
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    var consumer = await memphis.CreateConsumer(new MemphisConsumerOptions
    {
        StationName = "votes",
        ConsumerName = $"{typeof(T).Name}Consumer",
        ConsumerGroup = "Vote.Service"
    }, cancellationToken: stoppingToken);
    if (_esdbOptions.ConnectionString == null)
        throw new Exception("Unable to start Election Worker: ESDB connection failed");
    if (_esdbOptions.Username != null && _esdbOptions.Password != null)
        _credentials = new UserCredentials(_esdbOptions.Username, _esdbOptions.Password);
    var esdbSettings = EventStoreClientSettings.Create(_esdbOptions.ConnectionString);
    _esdb = new EventStoreClient(esdbSettings);
    consumer.MessageReceived += OnMessageReceived;
    await consumer.ConsumeAsync(stoppingToken);
}
```

This method will run until `stoppingToken` is cancelled, calling `ProcessMessage` for each message received.

Finally, we just need concrete implementations for each Aggregate Root:

```c#
public class ElectionWorker(MemphisClient memphis, IOptions<EsdbOptions> esdbOptions)
    : WorkerBase<Election, ElectionId>(memphis, esdbOptions);
```

The quintessential 1-line service - there's not even a class body! Add each of these hosted services to the root worker
service:

```c#
var builder = Host.CreateApplicationBuilder(args);
builder.Logging.AddConsole();
var memphisOpts = MemphisClientFactory.GetDefaultOptions();
builder.Configuration.GetSection("Memphis.dev").Bind(memphisOpts);
builder.Services.Configure<EsdbOptions>(builder.Configuration.GetSection("Esdb"));
builder.Services.Configure<MongoOptions>(builder.Configuration.GetSection("MongoDb"));
var memphis = await MemphisClientFactory.CreateClient(memphisOpts);
builder.Services.AddSingleton(memphis);
builder.Services.AddHostedService<BallotWorker>();
builder.Services.AddHostedService<BallotMongoDbWorker>();
builder.Services.AddHostedService<CandidateWorker>();
builder.Services.AddHostedService<CandidateMongoDbWorker>();
builder.Services.AddHostedService<CompetitionWorker>();
builder.Services.AddHostedService<CompetitionMongoDbWorker>();
builder.Services.AddHostedService<ElectionWorker>();
builder.Services.AddHostedService<ElectionMongoDbWorker>();
builder.Services.AddHostedService<VoterWorker>();
builder.Services.AddHostedService<VoterMongoDbWorker>();

var host = builder.Build();
host.Run();
```

### Database Workers

In the above code, there's still more hosted services: the `MongoDbWorker`s. If the `Worker` is a generic dispatcher and
recorder of events, the `MongoDbWorker` is a generic recorder of aggregate state. Loading an aggregate from the master event
source every time someone requests it is computationally expensive and inefficient. The `MongoDbWorker` receives updates
from EventStore DB every time a new event is added to a stream. (Well, the subscription can actually employ filters, so
that's not strictly true. More later.) A more efficient implementation would not read the whole aggregate back, but would
simply apply a delta based on the type of event. There are benefits to both approaches, so you can decide for yourself 🙂.

```c#
public class MongoDbWorker<TAggregate, TId>(IOptions<MongoOptions> mongoOptions, IOptions<EsdbOptions> esdbOptions) : BackgroundService
    where TAggregate : AggregateRoot<TAggregate, TId>, new()
    where TId : AggregateId<TId>, IEqualityComparer<TId>, new()
{
    private EventStoreClient? _esdb;
    private MongoClient? _mongo;
    private readonly EsdbOptions _esdbOptions = esdbOptions.Value;
    private readonly MongoOptions _mongoOptions = mongoOptions.Value;
    private UserCredentials? _credentials;

    // ... omitted
}
```

Once again, a generic service requiring only the types of the aggregate and aggregate id.

```c#
 private async Task EventAppeared(StreamSubscription subscription, ResolvedEvent resolvedEvent,
        CancellationToken cancellationToken)
{
    if (_mongo == null) return;
    var assembly = Assembly.Load("Vote.Event");
    var metadataJson = Encoding.UTF8.GetString(resolvedEvent.OriginalEvent.Metadata.ToArray());
    dynamic? metadata = JsonConvert.DeserializeObject(metadataJson, typeof(JObject));
    if (metadata == null) return;
    var eventType = assembly.GetType(metadata.ClrType.ToString());
    var eventJson = Encoding.UTF8.GetString(resolvedEvent.OriginalEvent.Data.ToArray());
    var @event = ExtractEvent(eventJson, eventType);
    if (@event != null)
    {
        var aggregate = await LoadAggregate(@event.Id);
        aggregate.Apply(@event);
        var collection = _mongo.GetDatabase("vote").GetCollection<TAggregate>($"{typeof(TAggregate).Name.ToLower()}");
        var filter = new FilterDefinitionBuilder<TAggregate>().Eq("Id", @event.Id);
        await collection.ReplaceOneAsync(filter, aggregate, new ReplaceOptions {IsUpsert = true}, cancellationToken: cancellationToken);
        var checkpoints = _mongo.GetDatabase("vote").GetCollection<Checkpoint>("checkpoint");
        var checkpointFilter =
            new FilterDefinitionBuilder<Checkpoint>().Eq(cp => cp.WorkerName, typeof(TAggregate).Name);
        await checkpoints.ReplaceOneAsync(checkpointFilter,
            new Checkpoint
            {
                WorkerName = typeof(TAggregate).Name,
                Position = (resolvedEvent.OriginalPosition ?? Position.Start).ToString()
            }, new ReplaceOptions { IsUpsert = true }, cancellationToken: cancellationToken);
    }
}
```

This service will call `EventAppeared` for every event matching the filter subscription. The filter for each
MongoDbWorker<TAggregate, TId> is a stream prefix of $"vote-{typeof(TAggregate).Name.ToLower()}". In other words, it will
receive all events generated by this application that correspond with an aggregate of TAggregate. This might need a little
bit of tweaking, but it works well for this simple application.

You might have missed it if you blinked, but that's all the CRUD code this application will ever need for MongoDB. Let's
look a little more carefully.

1. Load the Aggregate from Event Store DB (`await LoadAggregate(@event.Id)`)
2. Apply the current event to the aggregate (`aggregate.Apply(@event)`)
3. Update MongoDB with the current state of the aggregate (`await collection.ReplaceOneAsync(filter, aggregate, new ReplaceOptions {IsUpsert = true}, cancellationToken: cancellationToken)`)

Admittedly, this is a little slower than simply applying the event's update directly to the database, but only if EventStoreDBis slow, which it is not, at least not at this scale. I'm quite confident that there is less than 250ms of total latency
between API entry and MongoDB update in this system. I can already hear the objections though! "But the aggregate will
be out-of-date when I read it, because it takes less than 250ms to call the get method after posting my update". The 
statement itself is true: the aggregate does have a 250ms latency in MongoDB. But the spirit of the objection overlooks
that this is an easy problem to overcome. Why do you need to reread the database? The client already knows the current state!
Sometimes we only need to change our thinking a little. (More on this later when I talk about the web client)

### Voting API

The API itself is a very simple façade that copies request parameters into event objects and forwards them to the Memphis
station. Here's the start of the `ElectionController`:

```c#
[ApiController, Route("{controller=Election}/{action=Index}/{id?}")]
public class ElectionController(MemphisClient memphis) : ControllerBase
{
    private readonly MemphisProducer _producer = memphis.CreateProducer(new MemphisProducerOptions
    {
        StationName = "vote",
        ProducerName = "ElectionProducer"
    }).GetAwaiter().GetResult();

    [HttpGet]
    public IActionResult Index([FromRoute] string? id = null)
    {
        return NoContent(); // read from Mongo, not yet implemented
    }

    [HttpPost]
    public async Task<IActionResult> Created([FromRoute] string id)
    {
        NameValueCollection headers = [];
        string json = await new StreamReader(Request.Body).ReadToEndAsync();
        headers.Add("Clr-Type", typeof(ElectionCreated).FullName);
        dynamic? obj = JsonConvert.DeserializeObject(json);
        string? name = obj?["name"];
        if (name == null) return BadRequest("name is required");
        string? description = obj?["description"];
        if (description == null) return BadRequest("description is required");
        var @event = new ElectionCreated(id, name, description);
        await _producer.ProduceAsync(Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(@event)), headers, asyncProduceAck: true);
        return Ok();
    }

    [HttpPut]
    public async Task<IActionResult> Name([FromRoute] string id)
    {
        NameValueCollection headers = [];
        string json = await new StreamReader(Request.Body).ReadToEndAsync();
        headers.Add("Clr-Type", typeof(ElectionNameChanged).FullName);
        dynamic? obj = JsonConvert.DeserializeObject(json);
        string? name = obj?["name"];
        if (name == null) return BadRequest("name is required");
        var @event = new ElectionNameChanged(id, name);
        await _producer.ProduceAsync(Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(@event)), headers, asyncProduceAck: true);
        return Ok();
    }

    // ...
}
```

This can almost certainly be turned into a generic controller base as well, as was done with the dispatcher and database
workers.

That's a lot to take in for one day! The web client has some interesting commentary as well, and introduces .NET Blazor for
those who have not seen it before. Stay tuned, and thanks for reading! If you like this article or others I've written,
please consider using the "❤️ Sponsor" link at the top of the website.
