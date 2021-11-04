# Akka cheatsheet (untyped)
---
### Table of Contents
1. [Actor model](#actor-model)
2. [Message](#message)
3. [Actor communication and state](#actor-communication-and-state)
    1. [Forwarding](#forwarding)
    2. [Actor lifecycle](#actor-lifecycle)
        1. [Creation](#creation)
        2. [Deletion](#deletion)
    3. [Supervision](#supervision)
    4. [Actor behavior](#actor-behavior)
        1. [Hot-swapping method](#hot-swapping-method)
        2. [Behavior stack](#behavior-stack)
        3. [Stashing](#stashing)
        4. [Combining both](#combining-both)
    5. [Ask pattern](#ask-pattern)
4. [Routers & dispatchers](#routers-&-dispatchers)
    1. [Routers](#routers)
        1. [Router types](#router-types)
        2. [Creation](#creation)
        3. [Config](#config)
    2. [Dispatchers](#dispatchers)
        1. [Dispatcher types](#dispatcher-types)
        2. [Usage](#usage)
        3. [Config](#config)
5. [Cluster](#cluster)
    1. [Joining a cluster](#joining-a-cluster)
    2. [Cluster management](#cluster-management)
    3. [Communication](#communication)
    4. [Cluster failure](#cluster-failure)
    5. [Healing a cluster](#healing-a-cluster)
6. [Cluster sharding](#cluster-sharding)
    1. [Stateless applications issues](#stateless-applications-issues)
    2. [Database sharding](#database-sharding)
    3. [Communication](#communication)
        1. [Extractors](#extractors)
        2. [Balancing shards](#balancing-shards)
    4. [Implementation](#implementation)
        1. [Create](#create)
        2. [Find](#find)
        3. [Proxy](#proxy)
    5. [Stateful applications pros](#stateful-applications-pros)
        1. [Implementation](#implementation)
        2. [Working asynchronously](#working-asynchronously)
        3. [Real case example](#real-case-example)
    6. [Passivation](#passivation)
    7. [Rebalancing](#rebalancing)
        1. [Configuration](#configuration)
        2. [Remembering entities](#remembering-entities)
            1. [Implementation](#implementation)
        3. [Delaying deployment](#delaying-deployment)

---

### Actor model

An actor is a class which look like this:

```scala
object Waiter {
  def props(coffeeHouse: ActorRef, barista: ActorRef): Props = Props(new Waiter(coffeeHouse, barista))
}

...

class Waiter(coffeeHouse: ActorRef, barista: ActorRef) extends Actor with ActorLogging {
  override def receive(): Receive = ???
}
```

An actor has a companion object with a special method `props` which explains how to instantiate it.


`ActorLogging` is a trait which includes actor logging functionalities. Ie: `log.info(s"This is a log message")`

### Message

A message represents an action for an **Actor** to do. We also speak about *behavior*.

Express them with a `case class` if it takes parameters or a `case object` otherwise.

```scala
object Waiter {
  def props(coffeeHouse: ActorRef, barista: ActorRef): Props = Props(new Waiter(coffeeHouse, barista))
  case class ServeCoffee(coffee: Coffee) // case class because of parameter
}
```

Inside the actor class, you'll have to define abstract method `receive` which is made to handle messages from different actors.
In most case you have to use pattern matching to interpret these messages:

```scala
override def receive(): Receive = {
  case ServeCoffee(coffee) => coffeeHouse ! ApproveCoffee(coffee, sender())
  case ComplaintFromCustomer() => ...
}
```

`!` is used to send a message to an **actor**. Here we send `ApproveCoffee` message to `coffeeHouse` actor.
`sender()` can help you to identify the sender of a message your actor received.

### Actor communication and state

##### Forwarding

Can be used to skip an intermediary actor (act as a middleman) and deliver a message to another actor with the original sender as parameter:

`cook.forward(PrepareCoffee(coffee, customer))`. In the current context, the `PrepareCoffee` message is usually sent by a **waiter** actor to the **cook**. In that particular case, we are another actor and we want to skip the **waiter** and send an order directly to the **cook**.
On its side, the cook will see the **waiter** as the `sender()` and act like if nothing different happened.

#### Actor lifecycle

##### Creation

Actors can't be created inside a companion object because they depends on implicit **context** which is related to **Actor** trait.

The first will be created by the **actor-system** this way:

`system.actorOf(NewActor.props(params), "new actor name")`

Inside an actor, you can create a child actor by using:

`context.actorOf(NewActor.props(params), "new actor name")`

To get notified when a child actor is terminated/stopped you have to watch it:

`context.watch(actor)`. This command is commonly called directly after the child actor being started.

##### Deletion

`context.stop(actor)`. When called, all children are stopped and a `Terminated` event is fired and received by the watching actor:

```scala
case Terminated(customer) => log.info(s"The actor $customer is terminated !")
```
If it's not handled then a `DeathPactException` is thrown.

Moreover the `stop()` command will stop the actor after the message it's currently processing. There are several ways to stop an actor like special messages:
* actor ! PoisonPill
* actor ! Kill

Those two messages are enqueued into the actor mailbox and thus stop the actor when it deals with one of them.
The last one will throw an `ActorKilledException`

#### Supervision

The idea is to handle errors/exceptions and define a resulting strategy on how to proceed after.

There are two different default strategies: `OneForOneStrategy` and `AllForOneStrategy`. The first provides a strategy for a faulty **child** actor and the latter, a strategy for all **children** (even if they aren't faulty).

For most exceptions the default strategy is to restart the faulty actor and continue.

```scala
// Override restarts on CaffeineException by stopping the faulty actor
override def supervisorStrategy: SupervisorStrategy = {
  val decider: SupervisorStrategy.Decider = {
    case CaffeineException => Stop
    case FrustratedException(expectedCoffee, guest) =>
      cook.forward(PrepareCoffee(expectedCoffee, guest))
      SupervisorStrategy.Restart
  }

  OneForOneStrategy()(decider.orElse(super.supervisorStrategy.decider))
}
```

In the above case we decided to stop the actor when a **CaffeineException** is triggered and to override restart of the other exception by forwarding a message to the cook. The latter provides a *self-healing* scenario to our workflow (global application isn't stopped and exception managed properly).

`OneForOneStrategy()(decider.orElse(super.supervisorStrategy.decider))`

Here we override the `OneForOneStrategy` by only adding our exceptions thanks to `orElse` which will apply the default **decider** if exceptions do not match.

#### Actor behavior

Sometimes you would like to change an actor behavior for specific reasons.

##### Hot-swapping method
```scala
override def receive: Receive = ready
def ready: Receive = {
  case _ => context.become(busy)
  // Send Done msg to self when done.
}
def busy: Receive = {
  case Done => context.become(ready)
}
```

##### Behavior stack
```scala
def ready: Receive = {
  case _ => context.become(busy, discardOld = false)
  // Send Done msg to self when done.
}
def busy: Receive = {
  case Done => context.unbecome()
}
```
Here the idea is to keep old state behavior in a stack.
When `Done` is handled, the old state is set back.
> Be careful not to create memory-leaks !

##### Stashing

The idea is to also keep messages that are not handled by our actor in order to process them later:

```scala
def ready: Receive = {
  case _ => context.become(busy)
  // Send Done msg to self when done.
}
def busy: Receive = {
  case Done =>
    unstashAll()          // stashed messages will be added to the actor mailbox.
    context.become(ready)
  case _ =>
    stash()
}
```
> Stash size can be set in config `akka.actor.default-mailbox.stash-capacity`

##### Combining both

```scala
// Barista actor
override def receive: Receive = ready
private def ready: Receive = {
  case PrepareCoffee(coffee, guest) =>
    timers.startSingleTimer(
      "coffee-prepared",
      CoffeePrepared(selectCoffee(coffee), guest),
      prepareCoffeeDuration
    )
    context.become(busy(sender()))
}
private def busy(waiter: ActorRef): Receive = {
  case coffeePrepared: CoffeePrepared =>
    waiter ! coffeePrepared
    unstashAll()
    context.become(ready)
  case _ => stash()
}
```

> When using timers, be sure to define its value into a `lazy` variable otherwise you'll probably get a `NullPointerException` due to initialization order.

#### Ask pattern

Send a message _(request)_ and expect an answer _(response)_.
In order to do this, use *ask* _?_ operator (which is a `Future`). Be sure to import it with: `import akka.pattern.ask`:

```scala
import akka.pattern.ask

...

val response = actor ? PingMessage

response.mapTo[PongMessage] onComplete {
  case Success() => ???
  case Failure() => ???
}
```

or in an actor:

```scala
actor ? PingMessage pipeTo self
```

Here we _pipe_ the response of the `Future` to ourself.

> **Hint**: In **actor** code **only** use _ask_ in combination with _pipeTo_ to not break the single thread illusion. More concretely always use _pipeTo_ with `Futures` inside **actors**.

_Ex: Getting the status of an **actor** (CoffeeHouse)_


CoffeeHouse:
```scala
object CoffeeHouse {
  ...
  case object GetStatus
  case class Status(guestCount: Int)
}

class CoffeeHouse ... {
  ...
  override def receive: Receive = {
    ...
    case GetStatus => sender() ! Status(guestBook.size)
  }
}

```
CoffeeHouseApp (the actor-system):
```scala
object CoffeeHouseApp {
  ...
  def main(args: Array[String]): Unit = {
    ...
    val coffeeHouseApp = new CoffeeHouseApp(system)(statusTimeout)
    coffeeHouseApp.run()
  }

  class CoffeeHouseApp(system: ActorSystem)(implicit statusTimeout: Timeout) {
    import system.dispatcher
    ...
    protected def status(): Unit = {
      (coffeeHouse ? CoffeeHouse.GetStatus).mapTo[CoffeeHouse.Status] onComplete {
        case Success(status) => log.info(s"Number of guests: ${status.guestCount}")
        case Failure(error) => log.error(s"An error occurred while trying to get the status: $error")
      }
    }
  }
}
```

We're asking the **actor** `CoffeeHouse` by sending it a `GetStatus` message which will respond with the number of **guests**. Then, we map the **Future** into a `Status` case class which will handle a parameter corresponding to this number.

### Routers & dispatchers

#### Routers

The router routes messages to routees.
Routees are actors that can process messages in parallel.

There are many routing strategies provided by *Akka*:

| Strategy | Purpose |
|----------|---------|
|RandomRoutingLogic |Choose an actor randomly |
|RoundRobinRoutingLogic |Choose an actor by order in a pool |
|SmallestMailboxRoutingLogic |Choose the smallest mailbox actor (local actors only) |
|ConsistentHashingRoutingLogic |Choose an actor considering a given pattern |
|BroadcastRoutingLogic |Choose every actors |
|ScatterGatherFirstCompletedRoutingLogic |Choose also everyone and select the first one which responds |
|TailChoppingRoutingLogic |Choose everyone too but ask each one independently (delay between all asks)|

##### Router types

| Type | Role |
| ---  | ---  |
|Pool Router |Create and supervise routees with a given configuration |
|Group Router |Each routee has its own parent and the router only routes messages to them |

> *Hint: `Kill` and `PoisonPill` cannot be interpreted by the routees*

##### Creation

There are basically two different ways of creating a router.
One way using the provided config and another way providing directly the router strategy:

```scala
context.actorOf(Props(new SomeActor).withRouter(FromConfig()), "some-actor-name") // 1
context.actorOf(FromConfig.props(SomeActor.props()), "some-actor-name")           // 1 bis
context.actorOf(RoundRobinPool(4).props(Props(new SomeActor)), "some-actor-name") // 2
```

##### Config

Here is an example of configuration:

```yml
akka {
  actor {
    deployment {
      /coffee-house/barista {     // represents /actor-system/actor
        router = round-robin-pool
        nr-of-instances = 4
      }
    }
  }
}
```

#### Dispatchers
Dispatchers are basically an entity which can control how many **threads** are allocated to **actors** for a given amount of time.

##### Dispatcher types

| Type | Role |
| ---  | ---  |
|Dispatcher |Share threads from a pool of threads (the default one) |
|PinnedDispatcher |Each actor gets its own thread (rarely used)|
|CallingThreadDispatcher |Use only one thread (only used for test purposes)| 

##### Usage

Generally speaking an other dispatcher from the default one is used for long running blocking operations like a big SQL query or any other type of process which requires a long execution. Thus you'll have to use a specific dispatcher for this kind of operation.
The default one is used for really fast operations.

```scala
context.actorOf(Props(new SomeActor).withDispatcher("my-dispatcher")
```

##### Config

Here is an optimized dispatcher configuration exploiting the maximum of a machine. In that case my computer has 6 physical cores:

```yml
akka {
  actor {
    deployment {
      /coffee-house/barista {
        router = round-robin-pool
        nr-of-instances = 12
      }
    }
    default-dispatcher {
      fork-join-executor {
        parallelism-min = 4
        parallelism-factor = 3.0 // Nb of physical cores / 2
        parallelism-max = 18     // Limit is = nb physical * parallelism-factor
      }
    }
  }
}
```
Here, **parallelism** keyword represents **threads**.

You can recognize a good configuration when there's **no spiky** behavior in your processed messages diagram (oscillating value of operation per second).

> *One other fact:* it's preferable not to have the **same** values into `nr-of-instances` and `parallelism-max` because it means your routees will fight together with internal actors (whose belong to Akka internals) to get the thread execution time allowed. **You have to leave some extra-threads**.

### Cluster

In *Akka cluster* stack we distinguish **three** important notions:

|Notion |Purpose |
| ---   |---     |
|Cluster aware routers |Distributes **application instances** across cluster machines |
|Cluster sharding |Distributes **actors** across cluster machines (caching actors to prevent database reads)|
|Distributed data | In-memory, **replicated** data storage for each node |

In order to activate clusters, update your `application.conf` with:

```yml
akka {
  actor {
    provider = cluster
  }
}
```

Then, you are going to interact with remote actors. They are accessible through an address of this form: `akka://<ActorSystem>@<Hostname>:<Port>/<ActorPath>`.
To **enable** remote addresses, you have to set a remote transport protocol (*Artery*):

```yml
akka {
  remote {
    artery {
      enabled=on
      transport=tcp                  // Or aeron-udp / tls-tcp
      canonical {
        hostname="127.0.0.1"
        port=2551
      }
    }
  }
}
```

Each instance of the actor system that connects to the cluster is associated to an `uid` which is randomly generated and represents it's lifecycle.

#### Joining a cluster

To join an existing cluster, you have to retrieve its `seed nodes`:

> Deprecated
```yml
akka {
  cluster {
    seed-nodes = [
      "akka://ClusterSystem@127.0.0.1:2551",
      "akka://ClusterSystem@127.0.0.1:2552",
      "akka://ClusterSystem@127.0.0.1:2553",
    ]
  }
}
```

Or, alternatively you can use [Akka Cluster Bootstrap](https://doc.akka.io/docs/akka-management/current/bootstrap/) technique to **dynamically** assign seed nodes.

#### Cluster management

[Akka HTTP Cluster Management](https://doc.akka.io/docs/akka-management/current/cluster-http-management.html) provides a set of HTTP endpoints to manage the cluster.

```yml
akka {
  management {
    http {
      hostname = "127.0.0.1"
      port = 8558
      route-providers-read-only = true // Or false to w access to this endpoint.
    }
  }
}
```

To start use: `Akka management`: `AkkaManagement(system).start()` after actor system has been created.

**Endpoints**:

|Method|Route|Description|
|---|---|---|
|GET|/cluster/members/|Exposes the status of the cluster, membership, unreachable nodes, etc.|
|PUT|/cluster/members/{address}|Initiate a change to the cluster membership|
|GET|/cluster/shards/{name}|Exposes the shard information|

#### Communication

Each node/member of the cluster interact with each other through a `gossip` protocol.
They **inform** others about their own **status** at a given time and when **all nodes** are **aware** of each others it's called `gossip convergence`.

- It begins with the `Joining` state, then once all nodes have seen the new one trying to join them, the **leader** sets it to `Up`.
- When **leaving** the cluster, a node will move itself to the `Leaving` state
- After leaving, once **convergence** is reached (every one is aware of the leave), the leader sets it to `Exiting`
- Then the node sets it as `Removed` and must be restarted before it can rejoin

In order to gossip, each node interact with others thanks to a `heartbeat` message defining the node status.

#### Cluster failure

In case of **network failure** resulting in a network partition, you'll have to manually `Down` an `Unreachable` member.

To do so, you can use `Akka Management` by sending a **form-data** HTTP **PUT** to:
`<akka-management-address>/cluster/members/<member-address>` with the following body: `operation=down`.

> WARNING: Be sure to `Terminate` the member before *downing* it in order to avoid a `Split Brain` condition.

#### Healing a cluster

- Your nodes are `Up` and it's okay
- A problem occurs and a node become `Unreachable` as your **Akka Management dashboard** shows you at [http://localhost:8558/cluster/members]()
- You have to terminate the member/node:
    - Find its PID: `lsof -i :2551 | grep LISTEN | awk '{print $2}'`.
    - Kill it: `kill -9 <pid>`
- You have to `Down` the node: `curl -w '\n' -X PUT -H 'Content-Type: multipart/form-data' -F operation=down http://localhost:8559/cluster/members/{address}`
- Then, restart the node !


There are different strategies to make your cluster self-healing by resolving split brain thanks to **Lightbend Split Brain Resolver**:
- **static-quorum**: specifies the **number of nodes** in the cluster (static)
- **keep-majority**: tracks the size of the cluster and keep the **majority** which is re-calculated when adding/removing nodes
- **keep-oldest**: keep the **oldest** node sub-cluster
- **keep-referee**: keep all nodes which communicate to a designated node called the **referee**
- **down-all**: if any node become `Unreachable`, terminate all nodes
- **lease-majority**: (for **kubernetes**) similar to *keep-majority*

### Cluster sharding

Sharding provides a way to serve reads directly from memory, rather than the database.

#### Stateless applications issues

These are applications which doesn't rely on a context/state to work. Basically there's **no state** in the application.
Although they are not completely *stateless* since their database(s) keeps state anyway !
This kind of application provides strong consistency, it means a **single source of truth**: the **database**.

> WARNING: It also means a **single point of contention**: the source of truth becomes a **bottleneck**. It can also be a **single point of failure** !

#### Database sharding

We've previously seen **cons** of using **stateless applications** due to their single point of contention. **Sharding** can help to solve this issue by *replicating data* into *subsets* called **shards** (also known as *partitions*).
Shards are accessed by a **shard coordinator** which routes all requests to corresponding shard.

Shards are distributed into **shard regions** (usually one shard region per JVM for a type of entity / one shard region by node).

#### Communication

`Entity` is a reference to an `Actor` and are distributed into `Shards`.
In order to process incoming messages, an `Extractor` function is used to divide it into:
- An entity id
- A `Message` which will be passed to the `Entity` actor

**Inside the actor**, its `entity id` can be accessed via: `context.self.path.name`.
Basically, if you want to generate a random ID based on the entity's name:

```scala
private val entityId = UID.fromString(context.self.path.name)
```

##### Extractors

- *Entity Id*

The crucial information in order to **communicate** is the `Entity Id`.

> A common *pattern* is to create an **envelope** that contains the **entity id** and the associated **message**. Otherwise you'll have to **extract** the **entity id** from the message itself.
```scala
case class Envelope(entityId: String, message: Any)
val idExtractor: ExtractEntityId = {
  case Envelope(id, msg) => (id, msg)
}
```

You may want to extract the shard from an envelope:

- *Shard Id*

Once again, extract your shard id this way, with the `Hashcode modulo method`:
```scala
val shardIdExtractor: ExtractShardId = {
  case Envelope(id, _) =>
    (Math.abs(id.hashCode % totalShards)).toString
}
```

> **Best practice: 10 shards per node**


##### Balancing shards

The idea is to have an **even** distribution of **entities** in **each shard** of **each node** to avoid *hotspots* (unbalanced distribution: too many entities in a single shard).

#### Implementation

##### Create
...a shard for a specific actor type:

```scala
val shardActorRefs = ClusterSharding(actorSystem).start(
  "myShardedActors",                    // Name for the type of actors being sharded
  MyShardedActor.props(),
  ClusterShardingSettings(actorSystem),
  idExtractor,                          // Entity Id extractor function
  shardIdExtractor                      // Shard Id extractor function
)
...
shardActorRefs ! Envelope(entityId, someMessage)
```

##### Find
...an existing *shard region*:
```scala
val shardActorRefs = ClusterSharding(actorSystem).shardRegion("myShardedActors")
```
Useful when you don't want to recreate it.

##### Proxy
Used when there's **no shard** on the **local node** (it won't create any *shard region*):

```scala
val proxyActorRefs = ClusterSharding(actorSystem).startProxy(
  "myShardedActors",
  None,
  idExtractor,
  shardIdExtractor
)
```

#### Stateful applications pros

Stateless applications have their database as a *single point of contention*.

In stateful applications, each **sharded actor** handles messages one at a time. Thus, actors are a *point of contention*. However the contention is **distributed across many** actors as the cluster can be **scaled** up (or down) as necessary.

**It means no more contention !**

##### Implementation

It consists at adding a **state** into the `Actor`:

```scala
class MyActor extends Actor {
  private var state = ...
  override def receive: Receive = {
    case GetState => sender() ! state
    case UpdateState(newState) =>
      state = newState
      database.write(state)
  }
}
```

> Best practice: is to use **wrappers** such as `Actors` or `Futures` to avoid **blocking** operations to database.
```scala
class NonBlockingDB {
  def write(state: MyState): Future[Done]
  def read(id: MyId): Future[MyState]
}
```

##### Working asynchronously

The actor below won't process incoming messages until its **state** has been **loaded**:

```scala
class MyActor extends Actor with Stash {
  var state: Option[State] = None // No value in state by default

  nonBlockingDB
    .read(id)                     // read() is asynchronous and results into a Future
    .map(state =>
      StateLoaded(state)          // Encapsulate extracted state into a case class
    ) pipeTo self                 // Return state to ourself

  override def receive: Receive = waitingForState     // Delay receiving messages

  def waitingForState: Receive = {
    case StateLoaded(s) =>
      state = Some(s)
      unstashAll()
      context.become(running)            // Or another one behavior
    case Status.Failure(e) => throw e    // throw exception and restart the Actor (by default)
    case _ =>
      stash()
  }

}
```

> Notice that when an **exception** occurs, the actor is **restarted** thanks to the default strategy of exceptions.
However, all **messages previously received** in the mailbox which have been **stashed** are **not lost** and will be processed once the **actor** is restarted !

##### Real case example

```scala
class OrderActor(repository: OrderRepository) extends Actor with ActorLogging with Stash {
  import context.dispatcher
  import OrderActor._

  private var state: Option[Order] = None
  private val orderId: OrderId = OrderId(UUID.fromString(context.self.path.name))

  repository.find(orderId).map(OrderLoaded.apply) pipeTo self         // 1. Asynchronously pick data from DB and send OrderLoaded to ourself when complete

  override def receive: Receive = loading                             // 2. We asked DB so we start loading data

  private def loading: Receive = {
    case OrderLoaded(order) =>                                        // 3. We've got a response from DB
      unstashAll()
      state = order                                                   // 4. State is updated and behavior changed to running
      context.become(running)
    case Status.Failure(e) =>
      log.error(e, s"[$orderId] FAILURE: ${e.getMessage}")
      throw e                                                         // In order to restart actor
    case _ => stash()
  }

  private def running: Receive = {
    case OpenOrder(server, table) =>                                  // Here we rely on the state
      log.info(s"[$orderId] OpenOrder($server, $table)")
      state match {
        case Some(order) => duplicateOrder(orderId) pipeTo sender()   // 8. State isn't empty, do some stuff
        case None =>
          context.become(waiting)                                     // 5. Wait for DB write since there's no state
          openOrder(orderId, server, table).pipeTo(self)(sender())    // 6. Asynchronous write call to DB (Future) and passing a reference to the original sender() as param
      }

    ...
  }

  private def waiting: Receive = {
    case evt @ OrderOpened(order) =>                                  // 6.2 Once openOrder has been completed, store state
      state = Some(order)
      unstashAll()
      sender() ! evt
      context.become(running)                                         // 7. Re-send event and go back to running behavior
    ...
    case failure @ Status.Failure(e) =>
      log.error(e, s"[$orderId] FAILURE: ${e.getMessage}")
      sender() ! failure
      throw e
    case _ => stash()
  }

  private def openOrder(orderId: OrderId, server: Server, table: Table): Future[OrderOpened] = {    // 6.1 Asynchronous DB write and sent back OrderOpened message
    repository.update(Order(orderId, server, table, Seq.empty)).map(OrderOpened.apply)
  }
}

```

#### Passivation

Basically, it's removing **idle** actors from memory.
Passivation **removes** actor from memory if it hasn't processed a message since a given configured **time**.

`akka.cluster.sharding.passivate-idle-entity-after = 120 seconds` 120s by default.

You can **force** an actor to `Passivate` this way:
```scala
import ShardRegion.Passivate
context.parent ! Passivate(stopMessage = Stop)
```

* **Early passivation** can result in **more** frequent **database access**. Thus, it can reduce or eliminate the benefit of stateful actors.
* **Late passivation** can result in **high memory usage**. It can make the application fragile or expensive to operate.

> Best practices:
* Set *passivation time* at the **determined time an actor is active**.
* **Watch** your **memory usage**. If it's **too high**, then **shorten** the *passivation time*.

#### Rebalancing

Happens at node creation/deletion.

It is the operation of **moving** shards into another `Shard region`.
Messages to an `Entity` on the moving `Shard` are buffered.
**Theorically**, once the move has been done, the messages will be *delivered* (it's not guaranteed).

##### Configuration

```yaml
least-shard-allocation-strategy {
  rebalance-threshold = 1  # Difference of shards > 1 (ie: One node has 3 shards and another one has 5 shards, 5 - 3 > 1, rebalances, otherwise not)
  max-simultaneous-rebalance = 3
}
```

##### Remembering entities

When a shard has been migrated, its entities have been too.
You can choose to **restart** those entities with the following configuration:
`ClusterShardingSettings(system).withRememberEntities(true)` for a specific type of entity, or globally:
`akka.cluster.sharding.remember-entities = on`.

**It will disable automatic passivation !**
> Notice: However you can still passivate an actor by sending its parent a `Passivate` [message](#passivation).

> Notice: The list of **entities** can be **recovered** even after a full **cluster restart** !

###### Implementation

Adapt `ExtractShardId` function by adding a case:

```scala
val shardIdExtractor: ExtractShardId = {
  case Envelope(entityId, _) =>
    Math.abs(entityId.hashCode % 30).toString
  case ShardRegion.StartEntity(entityId) =>
    Math.abs(entityId.hashCode % 30).toString
}
```

*Remembering entities* isn't cheap. It requires all nodes in the cluster to be aware of all running entities !

> Best practice: Limit *Remember entities* to cases where there's a small number of active entities.

**Use cases**:
- When entity has scheduled process that may not have completed
- When time to restart an entity is too long (long to hydrate)
- When resource savings of passivating are insignificant

##### Delaying deployment
*...to avoid first node handling all of the load and thus, become unresponsive*

If `Remembering entities` is enabled, the problem is even more severe because it will also start the remembered entities !
If it succeeds, rebalancing will take a long time.

To prevent a node to handle **all of the load**, here is a configuration you can set:
`akka.cluster.min-nr-of-members = 3`

Members remain in a `Joining` state until the minimum requirement is met.