# Introduction

> *A single dramatic software failure can cost a company millions of dollars - but can be avoided with simple changes to design and architecture.*

Build systems that survive the real world, avoid downtime, implement continuous delivery, and make cloud-native applications resilient.
Examine ways to architect, design, and build software - particularly distributed systems that stands up to a flash mob, a Slashdotting, or a link on Reddit.

This section will present what I've understood, what I thought and my personal notes on a book that targets architects, designers and developers of distributed systems (including websites, web services and EAI projects (among others)).
Every one of this system has to be available, otherwise the company will lose money:

1. When the website is down, new clients cannot be targetd and existing customers won't be able to catch necessayr information.
2. When web services or the Enterprise Application Integration (EAI) system will be down, data flow will not work as intended and will surely increase support, will block employees in doing what that are payed for and everyone will have to go home for a day because softwares have stop working accordingly.

The main point is also to **aim at the right target**.
Based on available resources, we need to design systems that operates at low cost and high quality: the goal is to build softwares that are cheap to build, good for users and cheap to operate, by requesting little to no time when an evolution or new functionality is needed, and evolves and follows continuous techniques and state-of-the-art methdologies.

# Create stability

> *Bugs will happen. They cannot be eliminated, so they must be survived instead.*
> The worst problem is that a bug in one system can propagate to all the other affected systems.
> *How do we prevent bugs in one system from affecting everything else ?*

> Inside every enterprise today is a mesh of interconnected, interdependant systems.
> The cannot - must not - allow bugs to cause a chain of failures.

A robust system keeps processing transactions, even when transient impulses, persistent stresses, or component failures dirupst normal processing.
This is what most people meant by "stability".
It's not just that your individual servers or applications stay up and running, but rather that the user can still get work done.

* **Extending your life span**. The major dangers to your system's longevity are memory leaks and data growth. Both kinds of sludge will kill your system in production. Both are rarely caught in production: the trouble is that applications never run long enouth in the development environment to reveal their longevity bugs. These kind of bugs are not caught by load testing either, as a load test runs for a specified period of time. The only solution is to run your own longevity tests. [26]
* **Failure modes**. Sudden impulses and excessive strain can both trigger catastrophic failure. In either case, some component of the system will start to fail before everything else does. Once you accept that failure will happen, you have the ability to design your system's reaction to specific failures, by creating failure modes that contain the damage and protect the rest of the system. This sort of self-protection determines the while system's resilience. These *crumple zones* allow to decide what features of the system are indispensable and build failure modes that keep cracks away. If you do not design your failure modes, than you'll get whatever unpredictable - and usually dangerous - ones happen to emerge.
* **Stopping crack propagation**. This is probably the most complicated thing to anticipate, as it needs to have a good feeling of what could go wrong. And actually, *anything* could go wrong, from the OS to the database connection, an code exception that is not well caught, ...

## Stability antipatterns

### Chain of failures

One small issue lead to another, which lead to another, and so on.
If you try to estimated the probability of that exact chain of events occuring, it would look incredibly improbable.
But it looks improbable only if you consider the probability of each event individually: the combination of events that cause a failure is not independant.
A failure in one point or layer actually increases the probability of other failures.
If a database gets slow, thant the application servers are *most likely* to run out of memory: because the layers are coupled, the events of not independent.

One way to prepare for every possible failure is to look at every external call, every I/O, every use of resources, and every excpected outcome and ask: *"What are all the ways this can go wrong?"*
Think about different types of impulse and stress that can be applied :

* What if I can't make the initial connection ?
* What if it takes ten minutes to make a connection ?
* What if it can make the connection, and then get disconnected ?
* What if it doesn't get a response from the other end ?
* What if it takes two minutes to respond to my query ?
* What if 10.000 requests come in at the same time ?
* WHat if the disk is full when the application tries to log a message ?

It is not possible to have an exhaustive list of everything that could go wrong.
One thing is sure : faults will happen. They can never be completely prevented.

### Integration points

Every new application is an integration project with some combination of HTML, front-end apps, mobile app, API, or all of the above.
The context diagram for any project will fall between one of these two patterns:

1. The *butterfly* has a centralized system, with a lot of feeds and connections fanning int o on one side, and a large fan out on the other side.
2. The *spiderweb* is a style with many boxes and dependencies. If you've been diligent, the calls will soap through boundaries and layers before being sent to a new layer. If not, the web will be chaotic. The feature common to all of these is that the connecions outnumber the services. A butterfly has 2N connections; a spiderweb might have up to 2^N, and yours falls somewhere in between.

All these connections are integrations points, and every single one of them is out to destroy your system.
The more you move toward a large number of smaller devices, the more we go API first, the worse this is going to get.

The simplest failure mode occurs when the remote system refuses connections.
The calling system must deal with connection failures.
This shouldn't be a problem, since every programming language allows to deal with this kind of error.
What could take time, though, is to discover that you can't connect; for example when the destination server is hammered with connection requests.

When using socked-based connection, the port itself has a "listen queue" that defines how many pending connections (SYN sent, but no SYN/ACK replied) are allowed by the network stack.
Once that listen queue is full, further connection attempts are refused quickly.
The worst place is to be **in** the listen queue, as it can takes more than ten minutes for the sender to realise that there was a problem.

Network failures can hit you in two ways : fast or slow.
Fast network failures cause immediate exceptions in the calling code, in the form of a *connection refused*, it takes a few miiliseconds to come back to the caller.
Slow failures such as a dropped ACK, let threads block for minutes before throwing exceptions.
The blocked thread can't process other transactions, so overall capacity is reduced.
If all threads are blocked, then the server is down.

The most effective stability patterns to combat integrations points failures are *Circuit breaker* and *Decoupling middlewares*.
Other defensive patterns are *Timeouts* and *handshaking*.

### Cascading failures

An obvious example is a database failure.
If an entire database cluster goes dark, then any application that calls the database is going to experience problems of some kinds.
What happen next depends on how the caller is written.
Preventing cascading failures is the key to resilience.

### Users

As traffic grows, it will eventually surpass your capacity.
How does your system react to excessive demand ? If you're running in the cloud, then autoscaling is your friend.
It's not hard to up a huge bill by autoscaling buggy applications.

The best thing to defense against users is to test agressively, as some users might want some things that are expensive to serve.
Aim for the best.

Remember that users consume memory: each user session requires memory. Minimize that footprint to improve your capacity ("capacity" is the maximum throughput your system can sustain under a given workload while maintaining acceptable performance. When a transaction takes too long to execute, it means that the demand on your system exceeds its capacity).

### Infrastructure (73)

> If you have little time to preparen you can either set aside a portion of your infrastructure or provision new cloud resources to handle the promotion or traffic surge.
> This works only if the extraordinary traffic is directed at a portion of the system.
> In this case, event if the dedicated portion melts down, at least the rest of the system's regular behavior is available.
> The adivce is to "pre-autoscale" by upping the configuration before the marketing event goes out.

As the number of servers grows, then a different communication strategy is needed. Depending on your infrastructure, you can replace point-to-point communication with one of the following: UDP broadcasts, TCP or UDP multicast, Publish/subscribe messaging or message queues. Of course, *publish/subscribe* mecanisms come at a serious infrastructure cost.

If you provide a service, you would expect a "normal" workload. In reality, services serve unpredictable users.
To improve this,

1. Use capacity modeling, to make sure you're at least in the ballpark
2. Don't just test your system with your usual workloads: see what happens if you take the number of calls the front end could possibly make, double it, and direct it all against your most expensive transaction. If your system is resilient, it might slow down - event start to fail fast if it can't process transactions within the allowed time.
3. If you can, use autoscaling to react to surging demand.

### Databases

A *dogpile* can occur in several different situations:

* When booting up several servers, such as after a code upgrade and restart
* When a cron job triggers at midnight (or on the hour for any hour, really)
* When the configuration management system pushes out a change.

For example, Reddit faces an incident where every new instances (and the main server was just rebooted...) had their caches empty and all made the same queries to the databases, which led to a dogpile on the infrastructure.

Slow responses trigger cascading failures and slow responses cause more traffic.
Hunt for memory leaks or resources contention.

In conclusions:

* Use realistic data volumes: typical development and test data are too small to exhibit this kind of problem. You need production-sized data sets to see what happens when your query returns a million rows that you turn into objects.
* Paginate at the front end: requests should include a parameter for the first item and the count
* Don't rely on data producers: Even when you think a query will never have more than a handful of results, it could change without warning because of some other parts of the system. **The only sensible numbers are "zero", "one" and "lots", so unless your query selects exactly one row, it has the potential to return too many**.

## Stability patterns

### Timeouts

Networks are fallible: the wire could be broken, some switch or router along the way could be broken, or the computer you are addressing could be broken. Well-placed timeouts provide fault-isolation.
Handling all the possible timeouts in your code adds complexity, but if it is done well, it adds resilience.
For example, if the operation failed because of any significant problem, it's likely to fail again.
Problems on network or with other servers tend to last a while, so fast-retries are very likely to fail again.

Language runtimes to use callbacks or reactive programming styles let you specify timeouts more easily.

Remember :

* Apply timeouts to integration points, blocked threads and slow responses
* Apply timeouts to recover from unexpected failures
* Consider delayed retries.

All of these can be define with a Circuit Breaker pattern (see [https://pypi.org/project/pybreaker/](https://pypi.org/project/pybreaker/)).

### Bulkheads

(In a ship, to avoid sinking by spreading water across all compartments, by sealing every one of them independently from each other).

### Fail fast

If slow responses are worse than no response, the worst must surely be a slow *failure* response.
Even when failing fast, be sure to report a system failure (resources not available, ...) differently than an application failure.
This pattern improves overall stability by avoiding slow responses.

* Avoid slow responses and fail fast - don't wait until your system doesn't meet its SLA to inform the callers.
* Validate input : don't bother checking a database connection if a user parameter is missing right from the start.

### Handshaking

Handshaking is all about the letting the server protect itself by trottling its own workload. Instead of being victim to whatever demands are made upon it, the server should have a way to reject incoming work.

# Deliver your system

* Use caching carefully (page 67)
* Unbalanced capacities (page 75)
* Test harness (page 113)
* Handle others'version (page 270)
* Fail fast (page 106)
* Circuit breaker
* Governor (page 123) with Hysteresis

# Solve systemic problems

# Conclusions


# Must read

* *Inviting disaster*, by James R. Chiles.
