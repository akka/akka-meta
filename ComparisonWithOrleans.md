# Orleans and Akka Actors: A Comparison

## Step-By-Step Review

This overview walks along the Technical Report [MSR-TR-2014-41](http://research.microsoft.com/apps/pubs/default.aspx?id=210931) by *Philip A. Bernstein, Sergey Bykov, Allan Geller, Gabriel Kliot, Jorgen Thelin (Microsoft Research)*, for another comparison focusing on Erlang and Riak see [Christopher Meiklejohn’s article](http://christophermeiklejohn.com/orleans.html).

Thanks to Gabriel Kliot for reviewing and providing additional insight, this text would be a lot less accurate without his help.

### Introduction

The most interesting aspect in this section is the difference in primary focus between the two projects:

  * The primary focus of Orleans is to simplify distributed computing and allow non-experts to write efficient, scalable and reliable distributed services.
  
  * Akka is a toolkit for building distributed systems, offering the full power but also exposing the inherent complexity of this domain.

Both projects intend to be complete solutions, meaning that Orleans’ second priority is to allow experienced users to control the platform in more detail and adapt it to a wide range of use-cases, while Akka also raises the level of abstraction and offers simplified but very useful abstraction.

Another difference is that of design methodology:

  * The guiding question for Orleans is “what is the default behavior that is most natural and easy for non-experts?” The second question is then how the expert can make their own decision.
  
  * Akka’s guiding question is “what is the minimal abstraction that we can provide without compromises?” This means that “good default” for us is not driven by what users might expect, but what we think users will find most useful for reasoning about their program once they have understood the abstraction—familiarity is not a goal in itself.
  
#### Lifecycle

  * Orleans Grains do not have a lifecycle, can neither be started nor stopped. Consequently they also cannot fail and be restarted, therefore Orleans does not offer tools for software failure handling—the failure handling aspects concentrate on recovering from hardware crashes.
  
    On the other hand Grain _Activations_ do have a lifecycle and corresponding lifecycle hooks that can be used by the programmer to react to activation or deactivation.
  
  * Akka Actors implement the full model including defined lifecycle beginning and end, these are explicit operations. Persistent Actors support extending the lifecycle of a logical unit of computation beyond the lifecycle of the running process instance. Restarting an Actor provides powerful means for automatic service recovery.
  
#### Automatic Creation

  * Orleans Grains are created automatically whenever needed, which implies that Activation initialization should be careful in the externally visible effects it has—initialization activities with persistent effects should be made explicit interface methods that are invoked by the client. The automatic creation frees the user from having to consider the need for Grain creation.
  
  * Akka actors are explicitly created by their parent, implementing mandatory parental supervision. This allows precise control over when initialization actions are performed and which exact actor class is being created.
  
#### Virtual Actor Space

  * In Orleans each type of Grain corresponds to a practically infinite space of Grain instances that conceptually all exist from the beginning of the universe to its end. The relation to the physical Actors that implement the Grains is explained similar to virtual and physical memory, where swapping in and out corresponds to activation/deactivation of Grain (in other respects the analogy does not hold, like in that all Grains of a given type exist always whereas virtual memory needs to be mapped explicitly).
  
  * Akka’s explicit lifecycle management requires that all currently running Actors must have been explicitly started in the past and will have to be explicitly stopped in the future. Higher-level abstractions like ClusterSharding provide the ability to opt into an Orleans-like model where instances can be addressed by their logical name instead of a concrete ActorRef, but the API for accessing them is provided by other Actors that perform the necessary lookups and initialization on behalf of the user.

#### Automatic Actor Placement and Load Balancing

  * Orleans Grains are deployed on silos that can span multiple cluster nodes and the user has no direct influence on their precise placement or load-dependent movement. This means that making use of elastically provisioned resources is fully automatic and built into the model.
  
  * Elastic deployment and load balancing are features that Akka users opt into by using ClusterSharding or cluster-aware routers with remote deployment. Otherwise Actors are created explicitly on a given node (this can be influenced using the configuration file in order to allow deployment decisions to be taken by the operations personnel and not the programmer).
  
### Programming Model

#### Virtual Actors

  * Orleans Grains are identified by their nominal type and a 128-bit GUID, by a long, by a String, or by a combination of the three.
  
  * Akka Actors are identified by their ActorRef, which consists of the ActorPath (including physical network location and the names of all its ancestors) and its UID (a 32-bit integer that disambiguates different incarnations on the same ActorPath).

Actors as well as Grains do not share memory with their peers, both only communicate via message-passing.

##### Perpetual Existence

discussed above

##### Automatic Instantiation

  * An Orleans Grain may at any given time have one or more activations (active instances) depending on deployment mode: if automatic scale-out (see below) is enabled for a Grain type then the programmer will have to consider that there may be multiple copies running with the same identifier.
  
  * An Akka ActorRef refers to an Actor that must have been previously instantiated (otherwise there would not be an ActorRef but only an ActorPath available) and that may have already terminated. There will never be two Actors running with the same ActorRef. When using ClusterSharding there can be exactly zero or one instances currently running for each logical Actor name at any given point in time (unless the user explicitly configures the cluster to tolerate operation under split-brain scenarios).

see also above

##### Location Transparency

  * An Orleans Grain does not have knowledge of its physical location which can also transparently change during the runtime of the system. Users of a Grain’s function do not need to know the location of the Grain activation they are talking to. Grain references can also be sent as part of messages or persisted.
  
  * In Akka the central abstraction for Location Transparency is the ActorRef: it is a self-contained serializable representation of an Actor location that enables other Actors to send messages to it. ActorRefs can also be sent as part of messages to introduce Actors to one another.
  
##### Automatic Scale-Out

  * Orleans Grains can be configured as *stateless worker* which allows multiple activations to be active at any given time. The runtime manages the scaling factor in response to the workload.
  
  * In Akka cluster-aware routers offer the same functionality as the stateless worker mode of Orleans with the difference that here the provided abstraction exposes available resources to the program as the cluster grows but Akka itself does not contain the logic or ability to spin up additional nodes in response to increased load. This is left to other tools that manage server fleets.
  
#### Actor Interfaces

  * The API provided by Orleans uses code generation in order to present Actor interactions exactly like normal method invocations—with the only difference that all methods are required to return their result asynchronously. Composition of results is performed using the async–await language features of C# and friends.
  
  * Akka distinguishes Actor interactions from normal method invocations by requiring the use of the ActorRef’s `!` or `tell` methods in order to send messages. These method return no result value, receiving a reply from the Actor entails providing an ActorRef where the response is sent to.

In this area the differences between Akka and Orleans are very pronounced and mirror the different intent stated up-front: while Orleans aims for convenience and uses syntax that is familiar to OOP practitioners, Akka makes messaging very explicit and requires the user to define message classes. There is one instance of providing a similar model as async–await, namely the interaction between a PersistentActor and its Journal using the `persist` and `persistAsync` methods, but this API leads to a stronger coupling of the two components than would otherwise be the case. For a PersistentActor this is intended since it cannot function without the Journal, but for Grain interactions users now have to choose between blocking the grain in an `await` call (in a logical sense, but still!) or permitting other invocations to be processed concurrently “during” the `await` call—which leads to much of the same race conditions and problems that shared-state concurrency has. In Akka the interaction of multiple Actors is clearly demarcated by sending messages, and if and Actor ignores all incoming requests until a certain reply has been received then that is an explicit choice that the programmer makes.

Another consequence of using interface methods for messaging is that all interactions default to the request–response pattern whereas explicit messaging naturally allows more streamlined message flows through multiple stations.

#### Actor References

  * Orleans Actor references do not contain any location information, all invocations will be resolved using the distributed directory when needed.
  
  * Akka ActorRef contains all information that is necessary in order to send a message to the Actor it designates, including interactions with Actors outside of the current cluster.

#### Promises

  * Orleans relies upon the async–await language feature to formulate continuations that are invoked asynchronously. This means that the user need only ensure that methods return Promises and can then write code as if it was synchronous.
  
  * Instead of trying to offer a familiar (synchronous) user experience Akka makes handling asynchronous results explicit. In the case of the `ask`-pattern—which is analogous to how every invocation works in Orleans—the user should transform the Future explicitly or send its value to an Actor by way of the `pipeTo` pattern.

We found that making asynchronous code look synchronous invites many subtle possibilities of making mistakes, either concerning data consistency (i.e. transparently interleaving different invocations by way of their scheduled closures) or liveness (by blocking a Grain from processing any message until some response has been received). The nature of distributed systems is that messages are sent one-way and with at-most-once semantics, that is what physics gives us, and suggesting that there will always be a reply often turns out to be a pitfall, especially for unexperienced users.

#### Turns

  * In Orleans a Grain is scheduled in terms of non-overlapping *turns*, meaning that no two threads run that Grain’s code concurrently at any given time. This is also true for continuations formed via the usage of async–await.
  
  * Akka schedules Actors similarly, upon each assignment to a (single) Thread an Actor may process up to a configurable limit of messages and it is guaranteed that an Actor is running on at most one Thread at any given time. While it is possible to specify an ExecutionContext that runs continuations within the Actor’s schedule, the default executors do not respect this constraint and will typically run Future continuations concurrently with the Actor that scheduled them.
  
#### Persistence

  * Orleans provides persistence based on snapshots that are managed by the Grain explicitly. This maps well to a database-backed model where each Grain is responsible for a conceptual “row”.
  
  * Akka Persistence is based upon event-sourcing, which means that the PersistentActor emits events that represent state changes to be applied. Snapshots may be used as an optimization to shorten recovery time.

In both cases the persistence provider is configurable and users may choose to persist Actor state independently by other means as well. The biggest difference between snapshots and event-sourcing is that the event history of a PersistentActor contains valuable information that is lost when persisting only snapshots: each event has a well-defined meaning in the business domain and thus the event log retains its meaning as the internal representation of the Actor’s state changes over time while snapshots will become invalid and cannot be mined for additional insights later.

#### Timers and Reminders

  * Orleans provides transient timers that are local to a Grain activation and cease to exist when the Grain is passivated. It also provides reliable persistent reminders that will send an invocation to a Grain independently of whether it is currently active or not.
  
  * Akka includes only transient timers at this point, reliable reminders need to be implemented using third-party extensions like Akka Quartz.

### Runtime Implementation

#### Overview

Both systems consist of similar core components: Orleans’ Execution corresponds to akka-actor in that it is responsible for the actual execution of Actors, Messaging corresponds to akka-remote in that it manages TCP connections between the cluster nodes, and Hosting corresponds to akka-cluster plus ClusterSharding. One difference is that akka-actor also allows purely local message sends that are optimized to local reference passing whereas Orleans invocations are always sent via the Messaging component that consults with the Hosting component as to where the recipient currently resides. Akka’s lack of indirection allows substantial performance optimizations in this regard.

#### Distributed Directory

  * Orleans uses a one-hop distributed hash table that maps GUIDs to their current activations’ locations by way of partitioning the GUID hash space onto the cluster nodes. Local caches are used to optimize away the network round-trip in 90% of the cases.
  
  * Akka uses a sharding approach where the hash space of the logical Actor names is partitioned into shards of equal size and a ClusterSingleton coordinates the placement of these shards onto cluster nodes.

While Orleans’ Hosting has the ability to relocate individual Actors, it must pay for this ability by a hash table that grows with the number of Grains used (though there are mechanisms to compact it when needed). Akka’s hash-based approach in contrast distributes the full location information (on an as-needed basis) because its size is bounded by the configurable number of shards, which allows it to scale further than Orleans in the number of Actors that are running; the price here is that individual Actor placement is not possible.

#### Strong Isolation

  * Orleans ensures strong Grain isolation by copying all messages using the Serialization subsystem; application code can explicitly opt out of this behavior by using a marker class for wrapping invocation arguments.
  
  * Akka does not copy messages unless sending over a remote network link, trusting the user to not use mutable objects in Actor messages.
  
#### Asynchrony

Both libraries implement a purely asynchronous programming model.

#### Single-Threading

Both libraries map Actors to Threads in an N:M fashion with N typically being much larger than M.

#### Cooperative Multi-Tasking

Both libraries allow Actors to process one message at a time without interruption or time limit.

#### Serialization

  * Orleans contains code generators for emitting the serialization and deserialization code for all Grain invocations.
  
  * Akka’s Serialization subsystem is fully configurable, users can use any library for mediating between objects and ByteString that they like.
  
#### Reliability

This section describes in some detail how Orleans’ Hosting works, which is not very different from how Akka’s ClusterSharding works.

#### Eventual Consistency

  * Orleans’ Distributed Directory and Hosting service are eventually consistent, meaning that multiple activations of a *single activation* Grain may coexist during cluster topology changes (especially after failures).
  
  * An Akka Cluster depends on explicit removal of failed nodes according to a user-selected or user-implemented strategy, and this strategy determines whether in case of a network split the Cluster will favor availability (as Orleans does) or consistency (which would mean that the cluster shuts down completely in certain cases). While ClusterSharding is moving shards in response to load changes or failures certain logical Actors names may temporarily be unavailable, with their incoming messages being held and/or re-routed (within limits); ClusterSharding will never create two instances for a single logical Actor if the Cluster is configured to favor consistency as explained before.

#### Messaging Guarantees

  * Orleans guarantees *at-most-once* semantics, but it does not include sequence numbers to ensure per-pair message ordering, meaning that invocations to a Grain that are issued without waiting for the reply of each previous one before sending the next may be received out of order by that Grain.
  
  * Akka also offers *at-most-once* guarantees for normal Actor message sends (non-persistent exactly-once for supervision notifications) and does guarantee message ordering between each sender–recipient pair.

Both systems include opt-in support for *at-least-once* semantics with the inherent caveat that the recipient may receive duplicate messages—none of them include de-duplication.

It is interesting to see how all implementation and the model differ in this basic choice, Erlang provides neither resends nor ordering while the Actor Model by Carl Hewitt postulates *exactly-once* messaging   without ordering guarantees.

## Summary and Interpretation

While there is some overlap between the implementation of Orleans and Akka it is clear that both pursue completely different goals:

  * Orleans offers a programming model that integrates seamlessly into non-distributed methodology and programmer skills, it allows scaling out beyond the limits of a single computer without having to deal with the difficulties of writing a distributed application. This is achieved by making a set of implementation choices—like at-least-once delivery which is based upon the request–response style of using normal method invocations and so on—and offering a restricted set of tools to the user that can then be used without needing to understand the underlying technology. From this follow all the implications seen above, including the lack of a lifecycle for Grains, which is abstracted away.
  
    Since this abstraction achieves elasticity and resilience, allowing user code to responsive at all times, Orleans clearly is an implementation of the spirit embodied by the [Reactive Manifesto](http://reactivemanifesto.org/).
  
  * Akka provides a very simple and effective abstraction for modeling distributed systems—the Actor Model—and offers this low-level tool together with higher abstractions to the user. The philosophy is that the user must understand distributed programming in order to make their own choices regarding implementation trade-offs, for example whether the basic at-most-once message delivery semantics are enough or whether the additional cost for achieving at-least-once semantics is justified by the use-case at hand. While it is possible to build a system that offers equivalent capabilities to Orleans—which in large part is provided by Akka’s ClusterSharding module and its supporting abstractions (Remoting, Clustering, Persistence, etc.)—this is an add-on and users always have the freedom to drop down to lower levels of abstraction where performance or the desired semantics require it; Akka is not as opinionated as Orleans. One central feature that Akka provides which is absent in Orleans is support for software failure management through supervision, which together with the ability to dynamically create and destroy Actor hierarchies allows resilience beyond recovery from node failure.
    
    While being quite different from Orleans, Akka is also an implementation of the spirit embodied by the Reactive Manifesto.

Besides these differences in the guiding principles of both tools one particular difference deserves further elaboration. The modeling of Actors via strongly typed interfaces and proxy objects is the most prominent difference between Orleans and Akka; the latter includes the same functionality via the `akka.actor.TypedActor` feature, but that has been deprecated for a long time, awaiting its replacement by an equally type-safe but explicitly message-oriented abstraction. We have observed that offering classical method-based interaction between objects takes away many of the advantages of the purely message-driven nature of the Actor Model, in particular it is impossible to work with the underlying stream of messages for routing, aggregation, transformation and so on because this essential nature of the interaction is hidden from the programming model, it is abstracted away. Another manifestation of this difference is that TypedActor and Orleans Grains have as many entry points into the behavior of an object (Actor) as there are exposed methods while in normal Actors there is just one behavior that receives the next message at any given time. This leads to the loss of dynamic behavior changes (Grains cannot offer the equivalent of Akka’s `context.become()` and TypedActor has the same restriction). This also shines through in the rough edges around reentrancy for in-Grain continuations using async–await: while a classical Actor can—and must—freely choose which requests to ignore or buffer and which to answer during the period where another Actor’s response is awaited, Grains can only work in either the fully blocked or fully reentrant modes, limiting the user’s choices to a safe one and a fast one.

The usual and valid critique of the Actor Model, in particular in Akka, is that due to its dynamic nature and due to this single message entry point into an Actor’s behavior we forego all static type checking of Actor interactions—the ActorRef excepts `Any` message and the behavior is a function from `Any` to `Unit`. This is however not a fundamental restriction as the nascent [Akka Typed](http://doc.akka.io/docs/akka/snapshot/scala/typed.html) project shows: while it took a few tries to arrive at a formulation that works, we now have demonstrated that fully type-safe Actor interactions are possible without performance overhead and very little syntactic difference (with Scala’s case classes; in Java defining the required message classes involves more boilerplate). Using this new model we can transform a classical interface into an Actor definition by changing method declarations into class definitions—keeping the arguments as fields—and adding a reply-to ActorRef in order to transmit the return value if there is one; then the role of the `.` method selection is taken by the `!` message send operator. For this reason it might be time to rethink the valuation of classical interfaces and the Actor Model.

## Closing Remarks

This investigation has been sparked by the question of whether Akka.NET shall implement the «Virtual Actor» abstraction that Orleans offers, in addition to the classical Actor Model copied from Akka. After having seen the differences in goals and philosophy I would like to suggest that Orleans use the terms «Grain» and «Silo» to describe the undoubtedly useful abstraction that it provides, but refrain from using «Virtual Actor» terminology for two reasons:

  * Orleans’ explicit goal is to hide the Actor-inspired implementation details and offer a hassle-free “elastic object” programming model that does not require distributed programming expertise.
  
  * Grains are not Actors according to Carl Hewitt’s definition since they lack the ability to create other Actors dynamically—all Grains already “exist” before the program starts—and while “virtual memory” is still “memory” the abstraction offered by Orleans is differing from the Actor Model (for valid reasons).

---

Dr. Roland Kuhn

*Akka Tech Lead*

April 2015