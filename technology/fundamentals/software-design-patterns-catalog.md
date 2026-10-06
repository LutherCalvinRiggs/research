# Software Design Patterns: A Comprehensive Catalog

**Source:** Compiled from several catalogs: [Azure Architecture Center, Cloud Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/), Fowler's Patterns of Enterprise Application Architecture (category list via [this index](https://www.bhaumiknagar.com/list-of-enterprise-application-architecture-patterns/)), the [awesome-design-patterns](https://github.com/DovAmir/awesome-design-patterns) index, plus the classic Gang of Four, POSA, DDD, and messaging catalogs
**Saved:** 2026-10-06
**Tags:** technology, fundamentals, design-patterns, architecture, catalog, gof

> **How this was checked.** Re-checked against live sources this session: the Azure list (44 patterns) and Fowler's PEAA categories. The Gang of Four 23 are long-standing canon. The other groups (concurrency, DDD, messaging, reliability, front end, testing, delivery, security, AI, principles, anti-patterns) come from well-known catalogs and my own knowledge. I did not re-fetch each one. Treat those descriptions as a good starting point, not as a quotation of any source.

---

## TL;DR
A single reference of **295 named design ideas in 18 groups**, from class-level patterns (Strategy, Observer) up to system-level patterns (Saga, Circuit Breaker) and newer AI-agent patterns (Prompt Chaining, Orchestrator-Workers). Each entry has a one-line intent, a "use when" hint, aliases, and related entries. The same data powers the pattern gallery website.

## Key Concepts & Terms
- **Design pattern**: A named, reusable solution to a problem that keeps coming back in a given context. It is a template for a solution, not finished code.
- **Gang of Four (GoF)**: The authors of *Design Patterns* (1994). Their 23 patterns are 5 creational, 7 structural, and 11 behavioral.
- **Architecture pattern**: A pattern for the shape of a whole system, such as Layered, Hexagonal, or Event-Driven.
- **PEAA**: Martin Fowler's *Patterns of Enterprise Application Architecture* (2002). It covers data access, web layers, state, and locking in business apps.
- **Cloud design pattern**: A pattern for apps that run on many machines and must handle failure, such as Retry or Sharding.
- **Principle**: A guideline that sits behind patterns, such as SOLID, DRY, or YAGNI. A principle tells you what to value. A pattern gives you a shape to use.
- **Anti-pattern**: A common solution that looks fine but causes harm, such as God Object or Big Ball of Mud.

## Main Arguments & Takeaways
- **Patterns work at different scales.** GoF patterns shape classes. PEAA and DDD shape one app. Cloud, messaging, and architecture patterns shape whole systems. Pick the scale first, then the pattern.
- **The same idea repeats at larger scales.** Observer becomes Publisher-Subscriber and then Event-Driven Architecture. Proxy becomes Sidecar and then Service Mesh. Facade becomes Remote Facade and then API Gateway. Adapter becomes Anti-Corruption Layer.
- **AI-agent patterns reuse older ideas.** In my own mapping: Prompt Chaining is close to Pipes and Filters, Routing to Message Router, Parallelization to Scatter-Gather, and Orchestrator-Workers to Process Manager. This is my observation, not a claim from the sources.
- **Language features can replace patterns.** As far as I know, Peter Norvig (1996) argued that many GoF patterns become trivial in dynamic languages. In a language with functions as values, Strategy, Command, and Iterator often shrink to a function, a closure, or a generator.
- **Patterns have costs.** Every pattern adds indirection. Use KISS and YAGNI first. The Rule of Three and the Anti-patterns group exist to stop early, wrong abstraction.
- **Reliability patterns work as a set.** Timeout, Retry with backoff and jitter, Circuit Breaker, Bulkhead, and Rate Limiting each cover a different failure. Used alone, some can make an outage worse. Retry without backoff is a common cause of retry storms.
- **Some pairs are real choices, not duplicates.** Active Record vs Data Mapper, Choreography vs Orchestration (Process Manager), and Optimistic vs Pessimistic locking each trade simplicity for control.
- **Modular Monolith is a common alternative to Microservices.** It is often a safer first step, because module borders are easier to change than service borders.

## Notable Quotes
No quotes. This note is a compiled reference, not a single source.

## Questions & Gaps
- This list is broad but **not exhaustive**. It leaves out mobile (iOS and Android), game, IoT, big data, ML-ops, and language-specific idioms (Go, Rust, Python). The awesome-design-patterns index covers these.
- **Popularity is unmeasured.** Which of these 295 entries do teams actually use? A usage survey would help rank them.
- **Boundaries are fuzzy.** Some entries could sit in two groups (Saga, Event Sourcing, MVC). I placed each one where its source puts it, or where it is most often discussed.
- **AI-agent patterns change fast.** Treat that group as a snapshot. Check current sources before you rely on it.
- Which patterns survive well in your own stack? A follow-up note could pair each pattern with a real example from your projects.

## How to use this catalog
- Start from the **problem**, not the pattern name. Scan the "use when" lines in the group that matches your scale.
- Follow the **related** links to compare close neighbors before you choose.
- Check the **Principles** and **Anti-patterns** groups before you add a pattern.

---

## The Catalog

Format: **Name** (aliases): intent. *Hint:* when to use it. Source key at the end of each entry.

Source keys: `GOF` = Design Patterns, Gamma, Helm, Johnson, Vlissides (1994); `POSA` = Pattern-Oriented Software Architecture (Buschmann et al., 1996 to 2007); `PEAA` = Patterns of Enterprise Application Architecture, Fowler (2002); `DDD` = Domain-Driven Design, Evans (2003); `AZURE` = Azure Architecture Center, Cloud Design Patterns; `EIP` = Enterprise Integration Patterns, Hohpe and Woolf (2003); `MS` = microservices.io, Richardson; `ANTHROPIC` = Anthropic, Building Effective Agents (2024); `SOLID` = Robert C. Martin; `COMMON` = Common industry practice

### Creational (8)

_Ways to create objects without tying code to concrete classes._

- **Abstract Factory** (Kit): Create families of related objects without naming their concrete classes. *Hint:* Use when code must work with several product families, such as light and dark UI kits, and the pieces must match. `GOF`
- **Builder**: Build a complex object step by step. The same steps can make different results. *Hint:* Use when a constructor would need many optional arguments. `GOF`
- **Factory Method** (Virtual Constructor): Let a method create objects. Subclasses decide which class to create. *Hint:* Use when a class cannot know in advance which objects it must create. `GOF`
- **Prototype** (Clone): Create new objects by copying an existing object. *Hint:* Use when making an object from scratch costs more than copying one. `GOF`
- **Singleton**: Make sure a class has one instance and give global access to it. *Hint:* Use rarely. A shared logger can fit. Dependency injection is often better. `GOF`
- **Object Pool** (Resource Pool): Keep a set of ready objects and reuse them instead of making new ones. *Hint:* Use when objects are costly to create, such as database connections. `COMMON`
- **Dependency Injection** (DI, Inversion of Control): Give an object its dependencies from outside. The object does not create them. *Hint:* Use to swap parts in tests and to lower coupling. `COMMON`
- **Lazy Initialization** (Lazy Instantiation): Delay the creation of an object until the first time it is needed. *Hint:* Use when setup is costly and the object may never be used. `COMMON`

### Structural (7)

_Ways to compose classes and objects into larger structures._

- **Adapter** (Wrapper): Make one interface work where a different interface is expected. *Hint:* Use to connect a legacy or third-party class to your code. `GOF`
- **Bridge** (Handle/Body): Split an abstraction from its implementation so each can change on its own. *Hint:* Use when a class has two independent dimensions, such as shape and renderer. `GOF`
- **Composite** (Object Tree): Treat single objects and groups of objects the same way, using a tree. *Hint:* Use for part-whole trees, such as files and folders or UI widgets. `GOF`
- **Decorator** (Wrapper): Add behavior to an object by wrapping it. The wrapper has the same interface. *Hint:* Use to add features at run time without many subclasses. `GOF`
- **Facade**: Give one simple interface to a complex group of classes. *Hint:* Use to hide a messy subsystem behind a few clear calls. `GOF`
- **Flyweight** (Cache): Share common state between many small objects to save memory. *Hint:* Use when you need huge numbers of similar objects. `GOF`
- **Proxy** (Surrogate): Provide a stand-in that controls access to another object. *Hint:* Use for lazy loading, access control, caching, or remote calls. `GOF`

### Behavioral (14)

_Ways objects share work and talk to each other._

- **Chain of Responsibility** (CoR, Chain of Command): Pass a request along a chain of handlers until one handles it. *Hint:* Use for middleware, event bubbling, and validation steps. `GOF`
- **Command** (Action, Transaction): Turn a request into an object. You can then queue, log, or undo it. *Hint:* Use for undo and redo, job queues, and macros. `GOF`
- **Interpreter**: Define a grammar as classes and evaluate sentences in that language. *Hint:* Use for small languages, such as rules or query filters. `GOF`
- **Iterator** (Cursor): Walk through the items of a collection without showing how it is built. *Hint:* Use when the same loop must work on lists, trees, or streams. `GOF`
- **Mediator** (Controller): Put the communication between objects in one place so they do not talk to each other directly. *Hint:* Use when many objects depend on each other in a tangle, such as dialog widgets. `GOF`
- **Memento** (Snapshot, Token): Save an object's state so you can restore it later without breaking encapsulation. *Hint:* Use for snapshots, undo, and checkpoints. `GOF`
- **Observer** (Listener, Dependents, Publish-Subscribe): Let objects subscribe to another object and get told when it changes. *Hint:* Use when a change in one object must update others. `GOF`
- **State** (Objects for States): Let an object change its behavior when its internal state changes. *Hint:* Use when an object has many modes with different rules. `GOF`
- **Strategy** (Policy): Put each of several algorithms in its own class so you can swap them. *Hint:* Use when you pick a rule at run time, such as sort order or pricing. `GOF`
- **Template Method**: Define the steps of an algorithm in a base class. Subclasses fill in some steps. *Hint:* Use when several classes share one flow but differ in a few steps. `GOF`
- **Visitor** (Double Dispatch): Add new operations to a family of classes without changing the classes. *Hint:* Use when node types are stable and you add operations often. `GOF`
- **Null Object** (Neutral Object): Use an object that does nothing in place of a null reference. *Hint:* Use to remove many null checks. `COMMON`
- **Specification**: Describe a business rule as an object. Combine rules with and, or, and not. *Hint:* Use for filters and validation that you must reuse and combine. `DDD`
- **Interceptor** (Middleware): Hook into the flow of a call so you can add actions before or after it. *Hint:* Use for logging, auth, and metrics around calls. `POSA`

### Concurrency (15)

_Ways to run work at the same time and keep it safe._

- **Active Object**: Give an object its own thread. Callers send requests to a queue and get a future back. *Hint:* Use to keep calls non-blocking while object state stays safe. `POSA`
- **Actor Model** (Actors): Build a system from actors. Each actor owns its state and talks only by messages. *Hint:* Use for highly concurrent systems, such as chat servers or telecom. `COMMON`
- **Balking**: Ignore a call when the object is not in the right state for it. *Hint:* Use when a repeated call is useless, such as save with no changes. `COMMON`
- **Double-Checked Locking** (DCL): Test a condition before and after taking a lock to avoid the lock cost. *Hint:* Use for lazy setup in threaded code. It is easy to get wrong, so prefer language support. `POSA`
- **Future/Promise** (Promise, Deferred, Task): Return a placeholder for a value that is not ready yet. *Hint:* Use for async results, such as network calls. `COMMON`
- **Guarded Suspension**: Make a caller wait until a condition is true before it continues. *Hint:* Use when a thread needs a resource that is not ready. `COMMON`
- **Monitor Object** (Monitor): Allow only one thread at a time inside an object's methods. Threads can wait for conditions. *Hint:* Use to protect shared state with simple rules. `POSA`
- **Producer-Consumer** (Bounded Buffer): Separate the code that makes work from the code that handles it. Join them with a shared queue. *Hint:* Use to smooth speed differences between two stages. `COMMON`
- **Proactor**: Start async operations and run a handler when each one completes. *Hint:* Use for high-performance I/O when the OS completes the I/O for you. `POSA`
- **Reactor** (Event Loop, Dispatcher): Wait for events on many sources and send each event to its handler. *Hint:* Use for event-driven servers, such as Node.js or nginx. `POSA`
- **Read-Write Lock** (RW Lock): Let many readers share a resource, but give a writer exclusive access. *Hint:* Use when reads are common and writes are rare. `COMMON`
- **Scheduler**: Control when and in what order threads or tasks run. *Hint:* Use to apply a policy, such as priority or fairness. `COMMON`
- **Thread Pool** (Worker Pool): Keep a fixed set of worker threads and give them tasks from a queue. *Hint:* Use to avoid the cost of one thread per task. `COMMON`
- **Thread-Specific Storage** (Thread-Local Storage): Give each thread its own copy of a variable. *Hint:* Use for per-thread context, such as a request ID, without locks. `POSA`
- **Half-Sync/Half-Async**: Use async handling at the low level and sync code at the high level. Join them with a queue. *Hint:* Use when you want simple blocking code on top of an event-driven I/O layer. `POSA`

### Architecture (19)

_Large-scale ways to organize a whole system or app._

- **Layered Architecture** (N-Tier, Layers): Split the system into layers. Each layer uses only the layer below it. *Hint:* Use as a default for business apps. Watch for layers that only pass calls through. `POSA`
- **Client-Server**: Split work between clients that ask and servers that answer. *Hint:* Use for web apps and APIs. `COMMON`
- **Hexagonal Architecture** (Ports and Adapters): Put the domain in the center. Talk to the outside through ports and adapters. *Hint:* Use to keep business rules free of frameworks and databases. `COMMON`
- **Clean Architecture**: Arrange code in rings so dependencies point inward toward business rules. *Hint:* Use for long-lived apps that must outlast framework changes. `COMMON`
- **Onion Architecture**: Put the domain model at the core. Each outer layer depends only on inner ones. *Hint:* Use when the domain is rich and must stay independent. `COMMON`
- **Microkernel Architecture** (Plug-in Architecture): Keep a small core. Add features as plug-ins. *Hint:* Use for IDEs, browsers, and products that need extension. `POSA`
- **Event-Driven Architecture** (EDA): Components react to events instead of calling each other directly. *Hint:* Use when many parts must react to changes and stay loosely coupled. `COMMON`
- **Microservices Architecture** (Microservices): Build an app from small services. Each owns one capability and its data. *Hint:* Use when teams must deploy on their own. Expect high operating cost. `COMMON`
- **Modular Monolith**: Build one deployable unit with strict module borders inside. *Hint:* Use as a safer step before microservices, or instead of them. `COMMON`
- **Service-Oriented Architecture** (SOA): Expose business functions as reusable services over a network. *Hint:* Use in large enterprises that share services across systems. `COMMON`
- **Serverless Architecture** (FaaS): Run code as short functions on a platform that manages the servers. *Hint:* Use for spiky load and glue code. `COMMON`
- **Space-Based Architecture** (Tuple Space, Cloud Architecture Pattern): Spread processing and data across in-memory grids to avoid a central database limit. *Hint:* Use for extreme load with big peaks. `COMMON`
- **Broker Architecture**: Place a broker between clients and servers to route requests. *Hint:* Use to hide where services run. `POSA`
- **Blackboard**: Let many specialist parts work on a shared data store until a solution forms. *Hint:* Use for hard problems with no fixed algorithm, such as speech recognition. `POSA`
- **Model-View-Controller** (MVC): Split an app into data (model), display (view), and input handling (controller). *Hint:* Use in UI and web frameworks. `COMMON`
- **Model-View-Presenter** (MVP): Move UI logic to a presenter. The view stays passive. *Hint:* Use when you want to test UI logic without the UI. `COMMON`
- **Model-View-ViewModel** (MVVM): Bind the view to a view model that exposes state and commands. *Hint:* Use in UI frameworks with data binding, such as WPF, SwiftUI, or Vue. `COMMON`
- **Model-View-Update** (MVU, Elm Architecture): Keep state in one model. Events make messages. A pure update function makes the next state. *Hint:* Use for predictable UI state. `COMMON`
- **Entity Component System** (ECS): Build game objects from plain data components. Systems process every entity that has matching components. *Hint:* Use in games and simulations with many objects. `COMMON`

### Enterprise application (49)

_Patterns for business apps: data access, web layers, state, and locks._

- **Transaction Script**: Organize business logic as one procedure for each request. *Hint:* Use for simple logic. `PEAA`
- **Domain Model**: Model the business as objects that hold both data and behavior. *Hint:* Use when business rules are complex. `PEAA`
- **Table Module**: Use one object to handle all rows of a table or view. *Hint:* Use when the UI works with table-shaped data. `PEAA`
- **Service Layer**: Define the app's boundary as a set of operations that coordinate domain logic. *Hint:* Use when many clients need the same operations. `PEAA`
- **Table Data Gateway**: One object per table handles all SQL for that table. *Hint:* Use for simple data access with no object model. `PEAA`
- **Row Data Gateway**: One object per row exposes its fields and holds the SQL access. *Hint:* Use when you want row-level objects without domain logic. `PEAA`
- **Active Record**: An object wraps a database row, holds domain logic, and saves itself. *Hint:* Use for simple domains. Rails uses this. `PEAA`
- **Data Mapper**: A separate layer moves data between objects and the database so each stays independent. *Hint:* Use when the domain model and the schema differ. `PEAA`
- **Unit of Work**: Track changes during a business transaction and write them together. *Hint:* Use to avoid many small writes and keep changes consistent. `PEAA`
- **Identity Map**: Load each database row only once per session. Return the same object each time. *Hint:* Use to avoid duplicate objects for one row. `PEAA`
- **Lazy Load**: Delay loading related data until you need it. *Hint:* Use when related data is large or rarely used. `PEAA`
- **Identity Field**: Store the database key in the object so you can link object and row. *Hint:* Use in any object-to-table mapping. `PEAA`
- **Foreign Key Mapping**: Map a link between objects to a foreign key between tables. *Hint:* Use for one-to-many and one-to-one links. `PEAA`
- **Association Table Mapping**: Save many-to-many links in a link table. *Hint:* Use for many-to-many relations. `PEAA`
- **Dependent Mapping**: Let the owner class save its dependent child objects. *Hint:* Use when children have no life outside the owner. `PEAA`
- **Embedded Value**: Map an object into columns of its owner's table. *Hint:* Use for small value objects, such as a date range. `PEAA`
- **Serialized LOB**: Save a graph of objects as one serialized value in a single column. *Hint:* Use for hierarchies you never query by part. `PEAA`
- **Single Table Inheritance**: Store a class hierarchy in one table. *Hint:* Use when you want simple queries and few joins. `PEAA`
- **Class Table Inheritance**: Give each class in the hierarchy its own table. *Hint:* Use when you want a clean schema that matches the classes. `PEAA`
- **Concrete Table Inheritance**: Give each concrete class its own table with all of its fields. *Hint:* Use when you rarely query across the hierarchy. `PEAA`
- **Inheritance Mappers**: Organize mapper classes to handle a class hierarchy. *Hint:* Use when you map inheritance with Data Mapper. `PEAA`
- **Metadata Mapping**: Describe the object-to-table map as data. Generic code reads it. *Hint:* Use to avoid writing mapping code by hand. `PEAA`
- **Query Object**: Represent a database query as an object. *Hint:* Use to build queries without writing SQL strings. `PEAA`
- **Repository**: Give a collection-like interface to domain objects and hide how they are stored. *Hint:* Use to keep domain code free of data access detail. `PEAA`
- **Page Controller**: One controller handles the request for one page or action. *Hint:* Use for simple sites. `PEAA`
- **Front Controller**: One handler receives all requests and dispatches them. *Hint:* Use to share security and routing logic. `PEAA`
- **Template View**: Render output by placing markers in an HTML page. *Hint:* Use for most server-side pages. `PEAA`
- **Transform View**: Transform domain data into output, one element at a time. *Hint:* Use when output comes from XML or JSON transforms. `PEAA`
- **Two-Step View**: Make a logical page first. Then render it to HTML in a second step. *Hint:* Use when many pages must share one look. `PEAA`
- **Application Controller**: Centralize decisions about screen flow and which command runs. *Hint:* Use for complex wizards and flows. `PEAA`
- **Remote Facade**: Give a coarse interface to fine-grained objects to cut network calls. *Hint:* Use at a remote boundary. `PEAA`
- **Data Transfer Object** (DTO): Carry data between processes in one object to reduce calls. *Hint:* Use with a remote interface. `PEAA`
- **Optimistic Offline Lock** (Optimistic Locking): Detect conflicts at commit by checking a version. *Hint:* Use when conflicts are rare. `PEAA`
- **Pessimistic Offline Lock** (Pessimistic Locking): Lock the data when editing starts so no one else can edit. *Hint:* Use when conflicts are likely or costly. `PEAA`
- **Coarse-Grained Lock**: Lock a group of related objects with one lock. *Hint:* Use to keep a whole aggregate consistent. `PEAA`
- **Implicit Lock**: Let framework code take locks so developers cannot forget. *Hint:* Use to remove locking mistakes. `PEAA`
- **Client Session State**: Keep session data on the client. *Hint:* Use when servers must stay stateless. `PEAA`
- **Server Session State**: Keep session data in server memory. *Hint:* Use for small, fast sessions. `PEAA`
- **Database Session State**: Keep session data in database tables. *Hint:* Use when sessions must survive server restarts. `PEAA`
- **Gateway**: An object that wraps access to an external system or resource. *Hint:* Use to isolate outside APIs. `PEAA`
- **Mapper**: An object that sets up communication between two independent objects. *Hint:* Use when neither object should know the other. `PEAA`
- **Layer Supertype**: A shared base class for all classes in a layer. *Hint:* Use to put common code in one place. `PEAA`
- **Separated Interface**: Put an interface in one package and its implementation in another. *Hint:* Use to break dependency cycles between layers. `PEAA`
- **Registry** (Service Locator): A well-known object that others use to find common objects or services. *Hint:* Use when you cannot pass a reference. Prefer injection. `PEAA`
- **Money**: Represent an amount and a currency together, with correct rounding. *Hint:* Use for any money value. `PEAA`
- **Special Case**: A subclass that gives special behavior for one case, such as an unknown customer. *Hint:* Use to remove null checks and case logic. `PEAA`
- **Plugin**: Link classes at configuration time instead of compile time. *Hint:* Use when behavior must change between environments. `PEAA`
- **Service Stub**: Replace a hard-to-test service with a simple local one in tests. *Hint:* Use to test without the real service. `PEAA`
- **Record Set**: An in-memory copy of tabular data. *Hint:* Use with table-based UIs. `PEAA`

### Domain-driven design (13)

_Patterns for modeling a business domain and its borders._

- **Ubiquitous Language**: Use one shared language between developers and domain experts, in both code and talk. *Hint:* Use from day one of any domain model. `DDD`
- **Bounded Context**: Set a clear border in which one model and its language apply. *Hint:* Use to stop one big model from growing out of control. `DDD`
- **Context Map**: Document how bounded contexts relate and share data. *Hint:* Use to see team and model dependencies. `DDD`
- **Entity**: An object defined by its identity, which continues through changes. *Hint:* Use for things that have a life, such as an order or a user. `DDD`
- **Value Object**: An immutable object defined by its values, not by identity. *Hint:* Use for amounts, dates, and addresses. `DDD`
- **Aggregate**: A cluster of objects treated as one unit for changes. One root controls access. *Hint:* Use to set clear consistency borders. `DDD`
- **Domain Event**: Record something meaningful that happened in the domain. *Hint:* Use to tell other parts about a change. `DDD`
- **Domain Service**: Put logic that does not fit one entity or value object in a stateless service. *Hint:* Use for operations that span several objects. `DDD`
- **Shared Kernel**: Two contexts share a small part of the model and agree to change it together. *Hint:* Use when two teams are closely linked. `DDD`
- **Customer-Supplier**: An upstream context serves a downstream context and plans for its needs. *Hint:* Use when the downstream team can influence the upstream plan. `DDD`
- **Conformist**: The downstream context accepts the upstream model as it is. *Hint:* Use when upstream will not change for you. `DDD`
- **Open Host Service**: Offer a clear public protocol for other contexts. *Hint:* Use when many contexts need the same access. `DDD`
- **Published Language**: Use a documented shared language for exchange, such as a schema. *Hint:* Use with Open Host Service. `DDD`

### Cloud and distributed (44)

_Patterns for apps that run on many machines in the cloud._

- **Ambassador**: Run a helper next to a client that makes network calls on its behalf. *Hint:* Use to add retry, monitoring, or TLS for clients you cannot change. `AZURE`
- **Anti-Corruption Layer** (ACL): Place a translation layer between a new system and a legacy system. *Hint:* Use so the legacy design does not leak into the new model. `AZURE`
- **Asynchronous Request-Reply**: Accept a request at once and let the client check for the result later. *Hint:* Use when back-end work takes a long time. `AZURE`
- **Backends for Frontends** (BFF): Build one back end for each kind of client. *Hint:* Use when web and mobile need different APIs. `AZURE`
- **Bulkhead**: Separate resources into pools so one failure does not drain all of them. *Hint:* Use to contain a failure to one part. `AZURE`
- **Cache-Aside** (Lazy Loading Cache): Load data into a cache on demand when a read misses. *Hint:* Use for read-heavy data that changes slowly. `AZURE`
- **Choreography**: Let services react to each other's events with no central controller. *Hint:* Use for loose coupling between a few services. `AZURE`
- **Circuit Breaker**: Stop calls to a failing service for a while so it can recover. *Hint:* Use when a remote service can fail for a long time. `AZURE`
- **Claim Check**: Store a large payload elsewhere and send only a reference in the message. *Hint:* Use to keep messages small. `AZURE`
- **Compensating Transaction**: Undo earlier steps of a multi-step job when a later step fails. *Hint:* Use when you cannot use one atomic transaction. `AZURE`
- **Competing Consumers**: Let several workers read from one queue and share the load. *Hint:* Use to scale message handling. `AZURE`
- **Compute Resource Consolidation**: Group several tasks into one compute unit to save cost. *Hint:* Use when separate units waste capacity. `AZURE`
- **CQRS** (Command Query Responsibility Segregation): Use separate models for reading data and for writing data. *Hint:* Use when read and write needs differ a lot. `AZURE`
- **Deployment Stamps** (Scale Units): Deploy many independent copies of the full stack. *Hint:* Use for multi-tenant scale and regional isolation. `AZURE`
- **Event Sourcing**: Store every change as an event in an append-only log. Rebuild state by replaying it. *Hint:* Use when you need a full audit trail or time travel. `AZURE`
- **External Configuration Store**: Keep configuration in a central store outside the deployment package. *Hint:* Use to change settings without redeploying. `AZURE`
- **Federated Identity**: Let an external identity provider handle sign-in. *Hint:* Use to avoid building your own login. `AZURE`
- **Gatekeeper**: Put a dedicated host in front of back ends to validate requests. *Hint:* Use to shield trusted services from attack. `AZURE`
- **Gateway Aggregation**: Combine several back-end calls into one request at a gateway. *Hint:* Use to cut round trips from clients. `AZURE`
- **Gateway Offloading**: Move shared tasks, such as TLS or auth, to a gateway. *Hint:* Use to keep services simple. `AZURE`
- **Gateway Routing**: Route requests to many services through one endpoint. *Hint:* Use to hide service layout from clients. `AZURE`
- **Geode**: Run back ends in many regions so any node can serve any client. *Hint:* Use for low latency and high availability worldwide. `AZURE`
- **Health Endpoint Monitoring** (Health Check): Expose health checks that tools can poll. *Hint:* Use to detect failed instances. `AZURE`
- **Idempotent Consumer** (Idempotent Receiver): Make repeated delivery of a message safe, with the same result as one delivery. *Hint:* Use with at-least-once messaging. `AZURE`
- **Index Table**: Build extra indexes over fields that queries often use. *Hint:* Use when the store has no secondary indexes. `AZURE`
- **Leader Election**: Choose one instance to coordinate the others. *Hint:* Use when tasks need one coordinator. `AZURE`
- **Materialized View**: Precompute a view of data shaped for the queries you run. *Hint:* Use when the stored data is poorly shaped for reads. `AZURE`
- **Messaging Bridge**: Connect two messaging systems that cannot talk to each other. *Hint:* Use during migration or between vendors. `AZURE`
- **Pipes and Filters** (Pipeline): Break complex processing into small steps joined by channels. *Hint:* Use for reusable processing steps. `AZURE`
- **Priority Queue**: Process higher-priority requests before others. *Hint:* Use when some work must finish sooner. `AZURE`
- **Publisher-Subscriber** (Pub/Sub): Announce events to many consumers without coupling senders to receivers. *Hint:* Use to broadcast changes. `AZURE`
- **Quarantine**: Check outside assets against quality gates before using them. *Hint:* Use for third-party packages and images. `AZURE`
- **Queue-Based Load Leveling**: Put a queue between a task and a service to absorb bursts. *Hint:* Use when load is spiky. `AZURE`
- **Rate Limiting**: Control how fast clients use a resource to avoid throttling errors. *Hint:* Use to protect shared services. `AZURE`
- **Retry**: Try a failed operation again when the failure is likely temporary. *Hint:* Use for brief network faults. `AZURE`
- **Saga**: Keep data consistent across services with a chain of local transactions and compensations. *Hint:* Use for business flows that span services. `AZURE`
- **Scheduler Agent Supervisor**: Coordinate distributed steps with a scheduler, agents, and a supervisor. *Hint:* Use for long workflows that must recover. `AZURE`
- **Sequential Convoy**: Process related messages in order without blocking other groups. *Hint:* Use when order matters inside a group. `AZURE`
- **Sharding** (Horizontal Partitioning): Split a data store into horizontal partitions. *Hint:* Use when one node cannot hold the data or load. `AZURE`
- **Sidecar**: Run helper components in a separate process beside the main app. *Hint:* Use for logging, proxying, and config in any language. `AZURE`
- **Static Content Hosting**: Serve static files from cheap object storage or a CDN. *Hint:* Use for assets and static sites. `AZURE`
- **Strangler Fig**: Replace a legacy system piece by piece behind a routing layer. *Hint:* Use for safe, gradual migration. `AZURE`
- **Throttling**: Limit resource use by apps, tenants, or services to keep the system up. *Hint:* Use to survive load peaks. `AZURE`
- **Valet Key** (Pre-signed URL): Give clients a limited token for direct access to a resource. *Hint:* Use for direct uploads and downloads. `AZURE`

### Messaging and microservices (21)

_Patterns for services that talk by messages and events._

- **API Gateway**: Provide one entry point for clients. It routes to services and handles shared tasks. *Hint:* Use in front of a microservice system. `MS`
- **Service Discovery** (Service Registry): Let services find each other's network addresses at run time. *Hint:* Use when instances start and stop often. `MS`
- **Database per Service**: Give each service its own private data store. *Hint:* Use to keep services loosely coupled. `MS`
- **Service Mesh**: Run a proxy beside every service to handle traffic, security, and metrics. *Hint:* Use for many services that need uniform policy. `MS`
- **Transactional Outbox** (Outbox): Write the event to a table in the same transaction as the data. Publish it afterward. *Hint:* Use to avoid losing events or sending ghost events. `MS`
- **Message Router** (Content-Based Router): Send each message to a channel based on its content. *Hint:* Use when one input must reach different consumers. `EIP`
- **Message Translator**: Convert a message from one format to another. *Hint:* Use between systems with different formats. `EIP`
- **Message Filter**: Drop messages that a consumer does not want. *Hint:* Use to cut noise. `EIP`
- **Splitter**: Break one message into several parts. *Hint:* Use to handle each part on its own. `EIP`
- **Aggregator**: Combine related messages into one. *Hint:* Use to collect replies or parts. `EIP`
- **Scatter-Gather**: Send a request to many recipients and collect the replies. *Hint:* Use to get the best or all answers. `EIP`
- **Dead Letter Queue** (DLQ, Dead Letter Channel): Move messages that cannot be processed to a side queue for review. *Hint:* Use so one bad message does not block the flow. `EIP`
- **Correlation Identifier**: Put an ID in each message so replies match requests. *Hint:* Use for async request and reply. `EIP`
- **Request-Reply**: Send a request on one channel and get the answer on another. *Hint:* Use for two-way messaging. `EIP`
- **Wire Tap**: Copy messages passing on a channel so you can inspect them. *Hint:* Use for debug and audit. `EIP`
- **Content Enricher**: Add missing data to a message from another source. *Hint:* Use when a message lacks fields a consumer needs. `EIP`
- **Process Manager**: Track the state of a multi-step flow and route messages through it. *Hint:* Use for flows with branches and waits. `EIP`
- **Event Notification**: Send a small event that says something happened. Receivers fetch details if needed. *Hint:* Use for low coupling and small events. `COMMON`
- **Event-Carried State Transfer**: Put the changed data in the event so receivers need no call back. *Hint:* Use to cut calls between services. `COMMON`
- **Polling Consumer**: A consumer asks for messages when it is ready. *Hint:* Use to control the pace of work. `EIP`
- **Canonical Data Model**: Use one shared message format between all systems. *Hint:* Use to avoid many point-to-point translators. `EIP`

### Reliability and distributed systems (16)

_Patterns that keep systems working when parts fail._

- **Timeout**: Stop waiting for a call after a fixed time. *Hint:* Use on every remote call. `COMMON`
- **Fallback**: Return a default or cached answer when a call fails. *Hint:* Use when a partial answer is better than none. `COMMON`
- **Graceful Degradation**: Keep the core features working when parts fail. Turn off the extras. *Hint:* Use for user-facing systems. `COMMON`
- **Load Shedding**: Reject some requests on purpose when the system is overloaded. *Hint:* Use to protect latency for the rest. `COMMON`
- **Backpressure**: Let a slow consumer tell the producer to slow down. *Hint:* Use in streams and queues. `COMMON`
- **Hedged Request**: Send the same request to a second server if the first is slow. Use the first reply. *Hint:* Use to cut tail latency. `COMMON`
- **Exponential Backoff with Jitter** (Backoff and Jitter): Wait longer after each failure and add random delay. *Hint:* Use with retries to avoid retry storms. `COMMON`
- **Heartbeat**: Send a regular signal to show that a node is alive. *Hint:* Use for failure detection. `COMMON`
- **Write-Ahead Log** (WAL): Write each change to an append-only log before you apply it. *Hint:* Use for crash recovery in databases. `COMMON`
- **Two-Phase Commit** (2PC): A coordinator asks all parties to prepare and then to commit together. *Hint:* Use when you need atomic commits across stores and can accept blocking. `COMMON`
- **Consistent Hashing**: Map keys and nodes onto a ring so few keys move when nodes change. *Hint:* Use for caches and sharded stores. `COMMON`
- **Quorum**: Require a majority of nodes to agree before an action counts. *Hint:* Use for replicated data and leader choice. `COMMON`
- **Gossip Protocol** (Epidemic Protocol): Nodes share state by telling random peers, which tell others. *Hint:* Use for membership and failure detection at scale. `COMMON`
- **Lease**: Grant a right for a limited time. It expires unless renewed. *Hint:* Use for locks and leaders that must recover from crashes. `COMMON`
- **Fencing Token**: Give each lock grant a rising number. Resources reject older numbers. *Hint:* Use to stop a stale lock holder from writing. `COMMON`
- **CRDT** (Conflict-Free Replicated Data Type): Use data types whose copies can merge in any order and still agree. *Hint:* Use for offline-first and multi-writer data. `COMMON`

### Data and caching (6)

_Patterns for storing, caching, and tracking data changes._

- **Read-Through Cache**: The cache loads missing data from the store by itself. *Hint:* Use to keep loading logic out of the app. `COMMON`
- **Write-Through Cache**: Write to the cache and the store together. *Hint:* Use when reads must see fresh data. `COMMON`
- **Write-Behind Cache** (Write-Back): Write to the cache first and save to the store later. *Hint:* Use for fast writes. You risk losing data on a crash. `COMMON`
- **Refresh-Ahead Cache**: Reload hot cache entries before they expire. *Hint:* Use to avoid slow misses on popular keys. `COMMON`
- **Soft Delete**: Mark a row as deleted instead of removing it. *Hint:* Use for undo and audit. `COMMON`
- **Audit Log**: Record who changed what and when, in an append-only store. *Hint:* Use for compliance and debugging. `COMMON`

### Front end and rendering (23)

_Patterns for UI code, rendering, and web performance._

- **Module**: Group code and data in a file with a private scope and a public interface. *Hint:* Use to avoid global variables. `COMMON`
- **Mixin**: Add shared methods to many classes without a deep inheritance tree. *Hint:* Use for small reusable behavior. `COMMON`
- **Higher-Order Component** (HOC): A function that takes a component and returns an enhanced component. *Hint:* Use to share logic across components. `COMMON`
- **Render Props**: Pass a function as a prop that a component calls to render. *Hint:* Use to share logic and let the caller control the output. `COMMON`
- **Hooks**: Functions that let a component use state and effects and share logic. *Hint:* Use in React to reuse stateful logic. `COMMON`
- **Container/Presentational** (Smart/Dumb Components): Split components into ones that fetch data and ones that only draw. *Hint:* Use to separate data from view. `COMMON`
- **Compound Components**: Several components work together and share hidden state. *Hint:* Use for tabs, menus, and accordions. `COMMON`
- **Provider** (Context): Share data with a whole component tree without passing props down. *Hint:* Use for themes, auth, and settings. `COMMON`
- **Flux** (Redux): Send actions through a dispatcher to stores. Views read from stores. *Hint:* Use for one-way data flow. Redux follows this idea. `COMMON`
- **Client-Side Rendering** (CSR): Build the page in the browser with JavaScript. *Hint:* Use for app-like pages behind a login. `COMMON`
- **Server-Side Rendering** (SSR): Build the HTML on the server for each request. *Hint:* Use for fast first paint and SEO with fresh data. `COMMON`
- **Static Site Generation** (SSG): Build the HTML once at build time. *Hint:* Use for content that rarely changes. `COMMON`
- **Incremental Static Regeneration** (ISR): Rebuild static pages after build, one page at a time, when data changes. *Hint:* Use for large sites with changing content. `COMMON`
- **Streaming SSR**: Send HTML to the browser in chunks as it becomes ready. *Hint:* Use to show content sooner on slow data. `COMMON`
- **Islands Architecture**: Render a page as static HTML with small interactive islands. *Hint:* Use to ship less JavaScript. `COMMON`
- **Progressive Hydration** (Selective Hydration): Add interactivity to parts of the page one at a time. *Hint:* Use to cut start-up cost. `COMMON`
- **Lazy Loading**: Load code or assets only when needed. *Hint:* Use for below-the-fold content and rare routes. `COMMON`
- **Code Splitting**: Split the bundle into pieces that load on demand. *Hint:* Use to shrink the first download. `COMMON`
- **List Virtualization** (Windowing): Render only the rows that are visible. *Hint:* Use for very long lists. `COMMON`
- **Memoization**: Cache the result of a pure function for the same inputs. *Hint:* Use for costly computations and renders. `COMMON`
- **Debounce and Throttle**: Limit how often a function runs when events fire fast. *Hint:* Use for scroll, resize, and typing. `COMMON`
- **Optimistic UI**: Show the result of an action at once and fix it if the server rejects it. *Hint:* Use to make the app feel fast. `COMMON`
- **Micro Frontends**: Split a web app into parts that separate teams build and deploy. *Hint:* Use for large apps with many teams. `COMMON`

### Testing (8)

_Patterns for writing tests that stay useful._

- **Test Double** (Mock, Stub, Fake, Spy): A stand-in for a real part during a test. Kinds include dummy, stub, spy, mock, and fake. *Hint:* Use to isolate the code under test. `COMMON`
- **Arrange-Act-Assert** (AAA, Given-When-Then): Structure a test as setup, one action, and then checks. *Hint:* Use to keep tests readable. `COMMON`
- **Page Object**: Wrap a page's elements and actions in a class for UI tests. *Hint:* Use so UI changes need one fix. `COMMON`
- **Test Data Builder** (Object Mother): Build test objects with a fluent builder and sensible defaults. *Hint:* Use when tests need varied data. `COMMON`
- **Golden Master** (Approval Testing, Snapshot Testing): Record the output of existing code and compare future output to it. *Hint:* Use to test legacy code with no tests. `COMMON`
- **Property-Based Testing** (PBT): State rules that must hold for all inputs. A tool generates many inputs. *Hint:* Use to find edge cases you did not think of. `COMMON`
- **Contract Testing** (Consumer-Driven Contracts): Check that a client and a service agree on the shape of their messages. *Hint:* Use between teams and services. `COMMON`
- **Mutation Testing**: Change the code on purpose and check that tests fail. *Hint:* Use to measure test quality. `COMMON`

### Delivery and operations (8)

_Patterns for releasing and running software safely._

- **Blue-Green Deployment**: Run two full environments. Switch traffic from the old one to the new one. *Hint:* Use for fast release and rollback. `COMMON`
- **Canary Release**: Send a small share of traffic to the new version first. *Hint:* Use to catch problems early. `COMMON`
- **Rolling Deployment**: Replace instances of the old version one by one. *Hint:* Use when you cannot afford a full second environment. `COMMON`
- **Feature Flag** (Feature Toggle): Turn a feature on or off at run time without a deploy. *Hint:* Use to separate deploy from release. `COMMON`
- **Dark Launch**: Release a feature to production hidden from users. Test it with real traffic. *Hint:* Use to check load and bugs before launch. `COMMON`
- **Immutable Infrastructure**: Never change a running server. Replace it with a new one. *Hint:* Use to avoid configuration drift. `COMMON`
- **GitOps**: Keep the desired system state in Git. An agent applies it. *Hint:* Use for audited, repeatable changes. `COMMON`
- **Infrastructure as Code** (IaC): Define servers and networks in code files. *Hint:* Use to review, test, and repeat setups. `COMMON`

### Security (6)

_Patterns for protecting systems and data._

- **Role-Based Access Control** (RBAC): Give permissions to roles. Assign roles to users. *Hint:* Use to manage access at scale. `COMMON`
- **Zero Trust**: Trust no request by default. Check every request. *Hint:* Use for networks with no safe inside. `COMMON`
- **Defense in Depth**: Use several layers of protection, so one failure is not enough. *Hint:* Use for any system with real risk. `COMMON`
- **Secrets Management**: Store keys and passwords in a managed vault. Do not put them in code. *Hint:* Use for all credentials. `COMMON`
- **Token-Based Authentication** (JWT, Bearer Token): Give the client a signed token after sign-in. The client sends it with each call. *Hint:* Use for stateless APIs. `COMMON`
- **Input Validation at the Boundary** (Allow-list Validation): Check and clean all outside input where it enters the system. *Hint:* Use for every public interface. `COMMON`

### AI and agents (13)

_Patterns for apps built on language models._

- **Prompt Chaining**: Split a task into a fixed series of model calls. Each call uses the last output. *Hint:* Use when a task has clear steps and you can add checks between them. `ANTHROPIC`
- **Routing**: Classify an input and send it to a specialized prompt or model. *Hint:* Use when different inputs need different handling. `ANTHROPIC`
- **Parallelization** (Sectioning, Voting): Run model calls at the same time, either on parts of a task or as repeated votes. *Hint:* Use for speed or for higher confidence. `ANTHROPIC`
- **Orchestrator-Workers**: A central model splits a task and gives parts to worker models. It then merges the results. *Hint:* Use when you cannot predict the subtasks in advance. `ANTHROPIC`
- **Evaluator-Optimizer**: One model makes a draft. Another model critiques it. Repeat until it is good. *Hint:* Use when you have clear quality criteria. `ANTHROPIC`
- **ReAct** (Reason and Act): The model alternates between reasoning steps and tool actions. *Hint:* Use for tasks that need tool results to decide the next step. `COMMON`
- **Plan-and-Execute**: The model writes a full plan first. Then it runs the steps. *Hint:* Use for long tasks that benefit from a plan. `COMMON`
- **Reflection** (Self-Critique): The model reviews its own output and fixes it. *Hint:* Use to raise quality with one extra pass. `COMMON`
- **Tool Use** (Function Calling): Let the model call functions or APIs and read the results. *Hint:* Use to give the model data and actions. `COMMON`
- **Retrieval-Augmented Generation** (RAG): Fetch relevant documents and add them to the prompt before the model answers. *Hint:* Use to ground answers in your own data. `COMMON`
- **Human-in-the-Loop**: Pause the agent for a person to approve or correct key steps. *Hint:* Use for risky or irreversible actions. `COMMON`
- **Guardrails**: Check model inputs and outputs against rules and block bad ones. *Hint:* Use for safety, format, and policy limits. `COMMON`
- **LLM-as-Judge**: Use a model to score or compare the outputs of another model. *Hint:* Use for automated evaluation at scale. `COMMON`

### Principles (15)

_Guidelines that sit behind many patterns. These are not patterns themselves._

- **Single Responsibility Principle** (SRP): A class should have one reason to change. *Hint:* Apply when a class mixes unrelated jobs. `SOLID`
- **Open-Closed Principle** (OCP): Code should be open to extension and closed to change. *Hint:* Apply when new cases keep forcing edits to old code. `SOLID`
- **Liskov Substitution Principle** (LSP): A subtype must work anywhere its base type is expected. *Hint:* Apply when you design inheritance. `SOLID`
- **Interface Segregation Principle** (ISP): Do not force clients to depend on methods they do not use. *Hint:* Apply when interfaces grow large. `SOLID`
- **Dependency Inversion** (DIP): High-level code and low-level code should both depend on abstractions. *Hint:* Apply to decouple business rules from details. `SOLID`
- **DRY** (Don't Repeat Yourself): Keep one source of truth for each piece of knowledge. *Hint:* Apply to knowledge, not to code that merely looks alike. `COMMON`
- **KISS** (Keep It Simple): Keep designs as simple as the problem allows. *Hint:* Apply before you add a pattern. `COMMON`
- **YAGNI** (You Aren't Gonna Need It): Do not build a feature until you need it. *Hint:* Apply to speculative generality. `COMMON`
- **Law of Demeter** (Principle of Least Knowledge): A unit should talk only to its close friends, not to strangers. *Hint:* Apply to long call chains. `COMMON`
- **Composition over Inheritance**: Build behavior by combining objects rather than by extending classes. *Hint:* Apply when a hierarchy grows deep. `GOF`
- **Separation of Concerns** (SoC): Split a program into parts that each handle one concern. *Hint:* Apply at every level, from functions to services. `COMMON`
- **Tell, Don't Ask**: Tell objects what to do. Do not pull their data to decide for them. *Hint:* Apply when you see getters feeding outside logic. `COMMON`
- **Principle of Least Astonishment** (POLA): A system should behave the way users and developers expect. *Hint:* Apply to API and UI design. `COMMON`
- **Least Privilege** (PoLP): Give each part only the access it needs. *Hint:* Apply to users, services, and keys. `COMMON`
- **Rule of Three**: Wait until you see the same code three times before you abstract it. *Hint:* Apply to avoid early, wrong abstractions. `COMMON`

### Anti-patterns (10)

_Common bad solutions. Learn to spot them._

- **God Object** (Blob): One class knows or does too much. *Hint:* Split it by responsibility. `COMMON`
- **Spaghetti Code**: Control flow is tangled and hard to follow. *Hint:* Add structure and break it into functions. `COMMON`
- **Big Ball of Mud**: A system with no clear structure that grows by patches. *Hint:* Add boundaries slowly, for example with a Strangler Fig. `COMMON`
- **Golden Hammer**: Use a favorite tool or pattern for every problem. *Hint:* Match the tool to the problem. `COMMON`
- **Lava Flow**: Dead or unclear code stays because no one dares to remove it. *Hint:* Remove it with tests as a safety net. `COMMON`
- **Shotgun Surgery**: One change forces small edits in many places. *Hint:* Group related logic in one place. `COMMON`
- **Premature Optimization**: Optimize before you know where the time goes. *Hint:* Measure first. `COMMON`
- **Anemic Domain Model**: Domain objects hold data only. All logic lives in services. *Hint:* Move behavior into the domain objects. `COMMON`
- **Cargo Cult Programming**: Copy code or patterns without understanding why they work. *Hint:* Learn the reason before you copy. `COMMON`
- **Distributed Monolith**: Services are split but still change and deploy together. *Hint:* Fix the coupling or merge them. `COMMON`

---

## Related Notes
- [System Design Playbook — 16 Core Patterns](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/system-design-playbook-neo-kim.md): production system-design explainers that apply many of the cloud and architecture patterns in this catalog.
- [PatternsDev/skills — Agent Skills for JavaScript, React, and Vue](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/patterns-dev-skills-overview.md): the front-end and rendering patterns (CSR, SSR, ISR, hydration) come from the same patterns.dev material.
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md): the concepts behind the AI and agents group of patterns.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md): applies the State pattern idea to agent control.
- [How Complex Systems Fail — Cook (1998)](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/how-complex-systems-fail-cook-1998.md): the failure theory behind the reliability and resilience patterns.
