# 10. Spring Events Internals

---

# Learning Objective

By the end of this module, you will understand:

- Why Event Driven Architecture exists
- How Spring implements the Event Framework
- How events are published and dispatched
- How `@EventListener` works internally
- How Spring discovers listener methods
- How events are matched and invoked
- How transactional events work
- Spring Boot lifecycle events
- Internal Spring classes responsible for event processing
- Design principles behind the Spring Event Framework

---

# Prerequisites

Before starting this module, you should understand:

- IoC Container
- Bean Lifecycle
- BeanPostProcessor
- Reflection
- Spring AOP
- Proxy Architecture
- ApplicationContext

---

# Why This Feature Exists

## Problem Statement

Consider the following implementation.

```java
@Service
public class OrderService {

    @Autowired
    private EmailService emailService;

    @Autowired
    private InvoiceService invoiceService;

    @Autowired
    private AnalyticsService analyticsService;

    public void placeOrder(Order order){

        save(order);

        emailService.send(order);

        invoiceService.generate(order);

        analyticsService.record(order);

    }

}
```

Looks simple.

But after six months, the business adds more requirements.

```
Order Created

↓

Email

↓

Invoice

↓

Analytics

↓

SMS

↓

Loyalty Points

↓

Warehouse

↓

Audit

↓

Recommendation Engine

↓

Fraud Detection
```

Now every new requirement forces us to modify `OrderService`.

Eventually the service becomes tightly coupled to every downstream component.

Problems:

- High coupling
- Difficult testing
- Difficult maintenance
- Violates Open/Closed Principle
- Every new feature modifies existing business logic

---

# Solution

Instead of directly calling every service,
publish an event.

```java
@Service
public class OrderService {

    @Autowired
    private ApplicationEventPublisher publisher;

    public void placeOrder(Order order){

        save(order);

        publisher.publishEvent(new OrderPlacedEvent(order));

    }

}
```

Now interested components subscribe independently.

```
OrderService

        │

        ▼

OrderPlacedEvent

        │

        ├────────► Email

        ├────────► Invoice

        ├────────► Analytics

        ├────────► SMS

        ├────────► Warehouse

        ├────────► Loyalty

        └────────► Audit
```

Notice:

The publisher has no knowledge of listeners.

This is the Observer Pattern.

---

# Evolution

Early Spring versions required implementing:

```java
ApplicationListener<ApplicationEvent>
```

Example

```java
@Component
public class EmailListener
        implements ApplicationListener<OrderPlacedEvent> {

    @Override
    public void onApplicationEvent(OrderPlacedEvent event){

    }

}
```

Although powerful, this approach was verbose.

Spring later introduced:

```java
@EventListener
public void sendEmail(OrderPlacedEvent event){

}
```

The annotation provides the same functionality while significantly simplifying the programming model.

Internally, Spring still converts the annotated method into an `ApplicationListener`.

The developer sees a simpler API, while the framework preserves the original architecture.

---

# Core Components

The Spring Event Framework is built around a small number of infrastructure classes.

---

## ApplicationEventPublisher

Responsibility:

Publish events.

Developer interacts with this interface.

Example:

```java
publisher.publishEvent(event);
```

It does **not**

- find listeners
- invoke methods
- execute reflection

Its only responsibility is publishing.

---

## ApplicationEventMulticaster

Responsibility:

Dispatch events to all matching listeners.

Responsibilities include:

- lookup listeners
- iterate listeners
- invoke listener adapters
- support synchronous and asynchronous dispatch

It is the execution engine of the event framework.

---

## ApplicationListener

Original Spring listener interface.

```java
public interface ApplicationListener<E>{

    void onApplicationEvent(E event);

}
```

All modern event listeners are ultimately adapted to this interface.

---

## @EventListener

Provides an annotation-based programming model.

Example:

```java
@EventListener
public void sendEmail(OrderPlacedEvent event){

}
```

Spring discovers this annotation during container startup.

---

## EventListenerMethodProcessor

Responsible for:

- scanning beans
- discovering `@EventListener`
- creating listener adapters
- registering listeners

This class is implemented as a BeanPostProcessor.

---

## ApplicationListenerMethodAdapter

Wraps every discovered listener method.

Stores metadata such as:

- Bean Name
- Listener Method
- Event Type
- Annotation Metadata

Responsible for invoking the listener during runtime.

---

# Internal Architecture

```
Developer

        │

        ▼

publishEvent()

        │

        ▼

ApplicationEventPublisher

        │

        ▼

ApplicationEventMulticaster

        │

        ▼

ApplicationListenerMethodAdapter

        │

        ▼

ApplicationContext.getBean()

        │

        ▼

Reflection.invoke()

        │

        ▼

Listener Method
```

Notice the responsibilities.

Publisher

↓

Publish

Multicaster

↓

Dispatch

Adapter

↓

Invoke

ApplicationContext

↓

Resolve Bean

Reflection

↓

Execute Method

Each component has exactly one responsibility.

---

# Startup Phase

Unlike many frameworks,
Spring performs almost all expensive work during startup.

---

## Stage 1

ApplicationContext starts.

Infrastructure beans are created.

---

## Stage 2

BeanPostProcessors are registered.

One of them is:

```
EventListenerMethodProcessor
```

---

## Stage 3

Application beans are created.

Example:

```java
@Component
class EmailListener{

    @EventListener
    public void sendEmail(OrderPlacedEvent event){

    }

}
```

---

## Stage 4

EventListenerMethodProcessor inspects every bean.

Reflection is used to discover:

- annotations
- methods
- parameter types

---

## Stage 5

For every discovered listener method,

Spring creates:

```
ApplicationListenerMethodAdapter
```

Conceptually:

```
Bean Name

↓

emailListener

Method

↓

sendEmail()

Event Type

↓

OrderPlacedEvent
```

---

## Stage 6

The listener adapter is registered with:

```
ApplicationEventMulticaster
```

Conceptual Registry

```
OrderPlacedEvent

        │

        ├────────► EmailListener

        ├────────► InvoiceListener

        └────────► AuditListener
```

Startup processing is now complete.

No further annotation scanning is required.

---

# Runtime Phase

Suppose the application executes:

```java
publisher.publishEvent(new OrderPlacedEvent(order));
```

Execution Flow

```
publishEvent()

        │

        ▼

ApplicationEventPublisher

        │

        ▼

ApplicationEventMulticaster

        │

        ▼

Lookup Listener Registry

        │

        ▼

Matching Listener Adapters

        │

        ▼

ApplicationContext.getBean(beanName)

        │

        ▼

Reflection.invoke()

        │

        ▼

Listener Executes
```

Notice that runtime performs only:

- lookup
- bean resolution
- invocation

Everything else was completed during startup.

---

# Mental Model

Think of an airport announcement system.

```
Airport

↓

Announcement

↓

Passengers
```

The airport announces:

> Flight AI-202 is boarding.

It does not know:

- who is listening
- who boards
- who ignores the announcement

The announcement is simply published.

Interested passengers react.

Spring Events work exactly the same way.

---

# Module Summary

In this chapter, we learned:

- Why Event Driven Architecture exists
- Observer Pattern
- Spring Event Framework components
- Responsibilities of each component
- Startup processing
- Runtime execution
- High-level event architecture


---
# Event Listener Discovery Internals

Everything we learned so far explains **why** the Event Framework exists.

Now let's understand **how Spring discovers an `@EventListener` method**.

---

# Problem Statement

Suppose a developer writes:

```java
@Component
public class EmailListener {

    @EventListener
    public void sendEmail(OrderPlacedEvent event){

        System.out.println("Sending Email");

    }

}
```

Question:

How does Spring know this method should execute when
`OrderPlacedEvent` is published?

Java does not understand:

```
@EventListener
```

So who discovers it?

---

# Architecture

```
ApplicationContext

        │

        ▼

Create Bean

        │

        ▼

BeanPostProcessor

        │

        ▼

EventListenerMethodProcessor

        │

        ▼

Reflection

        │

        ▼

Find @EventListener

        │

        ▼

Create Listener Adapter

        │

        ▼

Register with Multicaster
```

Notice:

The listener is discovered **during startup**, not runtime.

---

# Why BeanPostProcessor?

Spring already has a lifecycle hook after every bean is created.

```
Bean Created

↓

BeanPostProcessor
```

Instead of creating a brand-new scanning mechanism,

Spring simply reused the existing extension point.

This is excellent framework design.

---

# Startup Phase

## Stage 1

ApplicationContext creates a bean.

```
EmailListener
```

---

## Stage 2

BeanPostProcessor receives the bean.

Conceptually:

```java
postProcessAfterInitialization(bean)
```

---

## Stage 3

EventListenerMethodProcessor inspects every method.

Reflection discovers:

```
Method

↓

Annotation

↓

Parameter Type
```

Suppose it finds:

```java
@EventListener
public void sendEmail(OrderPlacedEvent event)
```

---

## Stage 4

Spring extracts metadata.

Conceptually:

```
Bean Name

↓

emailListener

Method

↓

sendEmail()

Event Type

↓

OrderPlacedEvent
```

Notice:

Spring is **not storing the object**.

It is storing metadata.

---

# Why Metadata?

Suppose Spring stored:

```
EmailListener Object
```

Immediately a problem appears.

What if the bean is:

```
Prototype

Request Scope

Session Scope

Proxy
```

The cached object would become incorrect.

Instead Spring stores:

```
Bean Name

↓

Method

↓

Event Type
```

Whenever execution is required,

Spring asks the container:

```
ApplicationContext.getBean(beanName)
```

The IoC container returns the correct managed instance.

---

# ApplicationListenerMethodAdapter

Every discovered listener becomes one adapter.

Suppose we have:

```java
@Component
class OrderListener{

    @EventListener
    void email(OrderPlacedEvent event){}

    @EventListener
    void invoice(OrderPlacedEvent event){}

    @EventListener
    void analytics(OrderPlacedEvent event){}

}
```

Question:

How many beans?

Answer:

```
One Bean
```

Question:

How many listener adapters?

Answer:

```
Three
```

Conceptually:

```
OrderListener Bean

        │

        ├────────► Adapter #1

        │          email()

        │

        ├────────► Adapter #2

        │          invoice()

        │

        └────────► Adapter #3

                   analytics()
```

Spring manages objects as units of lifecycle,

but methods as units of behavior.

---

# Listener Registry

Once every adapter is created,

Spring registers it.

Conceptually:

```
OrderPlacedEvent

        │

        ├────────► Email Adapter

        │

        ├────────► Invoice Adapter

        │

        └────────► Analytics Adapter
```

Notice:

No listener execution has happened yet.

Only registration.

---

# Runtime Execution

Suppose the application executes:

```java
publisher.publishEvent(new OrderPlacedEvent());
```

Flow:

```
ApplicationEventPublisher

        │

        ▼

ApplicationEventMulticaster

        │

        ▼

Lookup Registry

        │

        ▼

Email Adapter

Invoice Adapter

Analytics Adapter
```

Each adapter executes independently.

---

# Adapter Invocation

Question:

Why doesn't the multicaster directly invoke reflection?

Because that would violate Single Responsibility Principle.

Instead:

```
ApplicationEventMulticaster

↓

Dispatch

----------------------------

ApplicationListenerMethodAdapter

↓

Invocation
```

The adapter is responsible for:

- obtaining the bean
- preparing arguments
- evaluating metadata
- invoking reflection

The multicaster only dispatches.

---

# Bean Resolution

The adapter stores:

```
Bean Name

↓

Method

↓

Event Type
```

When execution starts:

```
ApplicationContext

↓

getBean(beanName)

↓

Managed Bean

↓

Reflection.invoke()
```

Notice:

The adapter never performs:

```
new EmailListener()
```

It always asks the IoC container.

This guarantees:

- Singleton
- Prototype
- Scoped Proxy
- AOP Proxy

all continue working.

---

# Event Matching

Suppose:

```java
@EventListener
public void email(OrderPlacedEvent event)
```

Spring stores:

```
OrderPlacedEvent
```

Later:

```java
publisher.publishEvent(new OrderPlacedEvent())
```

The registry immediately finds:

```
OrderPlacedEvent

↓

Email Adapter
```

No annotation scanning.

No reflection lookup.

Only metadata lookup.

---

# Polymorphism

Suppose:

```
ApplicationEvent

        ▲

OrderPlacedEvent

        ▲

PremiumOrderPlacedEvent
```

Listener:

```java
@EventListener
public void handle(OrderPlacedEvent event)
```

Published:

```java
new PremiumOrderPlacedEvent()
```

Will Spring invoke the listener?

Yes.

Because:

```
PremiumOrderPlacedEvent

IS-A

OrderPlacedEvent
```

Spring follows Java's inheritance rules.

Conceptually:

```java
listenerType.isAssignableFrom(event.getClass())
```

No custom matching algorithm is required.

Spring simply reuses Java's type system.

---

# Internal Data Structures

Conceptually the registry looks like:

```
Event Type

↓

Listener Adapter

↓

Bean Name

↓

Method
```

or

```
Map

↓

Event Type

↓

List<ApplicationListener>
```

The exact implementation differs,

but conceptually this is how lookup works.

---

# Internal Working Summary

```
Application Starts

        │

        ▼

Create Beans

        │

        ▼

BeanPostProcessor

        │

        ▼

Discover @EventListener

        │

        ▼

Create Listener Adapter

        │

        ▼

Register Adapter

────────────────────────────

Runtime

────────────────────────────

publishEvent()

        │

        ▼

Lookup Registry

        │

        ▼

ApplicationListenerMethodAdapter

        │

        ▼

ApplicationContext.getBean()

        │

        ▼

Reflection.invoke()

        │

        ▼

Listener Executes
```

---

# Design Decisions

## Why BeanPostProcessor?

Reuse an existing lifecycle extension point instead of inventing another one.

---

## Why Listener Adapter?

Encapsulates invocation logic while keeping the multicaster simple.

---

## Why Store Bean Names?

Supports every bean scope and proxy mechanism.

---

## Why Startup Discovery?

Reflection is expensive.

Do it once.

Reuse forever.

---

## Why Java Polymorphism?

Spring builds upon the Java language instead of introducing a separate event matching system.

---

# Engineering Principles

### EP-063

Well-designed frameworks reuse existing extension points instead of introducing new lifecycle phases.

---

### EP-064

Frameworks manage objects as units of lifecycle while registering individual methods as units of behavior.

---

### EP-065

Frameworks wrap reflection metadata inside richer abstractions to support future extensibility.

---

### EP-068

Dispatchers coordinate execution; adapters encapsulate invocation logic.

---

### EP-069

Framework infrastructure retrieves managed objects through the IoC container instead of caching concrete instances.

---

### EP-070

Frameworks trade startup work for runtime efficiency.

---

### EP-071

Frameworks build upon the language's type system rather than introducing parallel compatibility rules.

---

# Interview Questions

### Beginner

How does Spring discover an `@EventListener`?

---

### Intermediate

Why does Spring use `ApplicationListenerMethodAdapter`?

---

### Advanced

Why does Spring store bean names instead of bean instances inside listener adapters?

Explain the architectural advantages.

---
# Transactional Event Internals

---

# Problem Statement

Suppose we have:

```java
@Service
public class OrderService {

    @Autowired
    private ApplicationEventPublisher publisher;

    @Transactional
    public void placeOrder(Order order){

        orderRepository.save(order);

        publisher.publishEvent(new OrderPlacedEvent(order));

    }

}
```

Listener:

```java
@Component
public class EmailListener {

    @EventListener
    public void sendEmail(OrderPlacedEvent event){

        sendConfirmationMail();

    }

}
```

Looks correct.

Now consider this:

```java
@Transactional
public void placeOrder(){

    saveOrder();

    publisher.publishEvent(new OrderPlacedEvent());

    throw new RuntimeException();

}
```

Execution:

```
Begin Transaction

↓

Save Order

↓

Publish Event

↓

Email Sent

↓

Exception

↓

Rollback
```

Problem:

The customer receives a confirmation email even though the order never existed.

Business consistency is broken.

---

# Solution

Spring introduced:

```java
@TransactionalEventListener
public void sendEmail(OrderPlacedEvent event){

}
```

Instead of executing immediately,

Spring waits until the transaction reaches the configured phase.

---

# Internal Architecture

```
Business Service

        │

@Transactional

        │

        ▼

publishEvent()

        │

        ▼

ApplicationEventPublisher

        │

        ▼

ApplicationEventMulticaster

        │

        ▼

Transactional Listener Adapter

        │

        ▼

TransactionSynchronizationManager

        │

        ▼

Commit / Rollback Callback

        │

        ▼

Invoke Listener
```

Notice something important.

The Event Framework did not change.

The Transaction Framework did not change.

Spring composed the two systems together.

---

# Internal Working

Stage 1

Business transaction starts.

```
Begin Transaction
```

---

Stage 2

Business data is modified.

```
Insert Order
```

---

Stage 3

Business publishes event.

```
publishEvent()
```

---

Stage 4

ApplicationEventMulticaster dispatches to the listener adapter.

---

Stage 5

The adapter detects:

```
@TransactionalEventListener
```

Instead of invoking immediately,

it registers a callback with

```
TransactionSynchronizationManager
```

---

Stage 6

Transaction finishes.

If COMMIT

↓

Execute Listener

If ROLLBACK

↓

Discard Callback

---

# Transaction Phases

## AFTER_COMMIT (Default)

```
Business Logic

↓

Commit

↓

Execute Listener
```

Typical Use Cases

- Email
- SMS
- Kafka Events
- Cache Update
- External API

---

## BEFORE_COMMIT

```
Business Logic

↓

Listener

↓

Commit
```

Use Cases

- Validation
- Final Data Enrichment
- Integrity Checks

---

## AFTER_ROLLBACK

```
Rollback

↓

Execute Listener
```

Use Cases

- Failure Notification
- Inventory Release
- Compensation Logic
- Audit Logging

---

## AFTER_COMPLETION

```
Commit

or

Rollback

↓

Execute Listener
```

Runs regardless of transaction outcome.

---

# Spring Boot Lifecycle Events

Spring Boot itself uses the Event Framework.

Application Startup

```
ApplicationStartingEvent

↓

ApplicationEnvironmentPreparedEvent

↓

ApplicationContextInitializedEvent

↓

ApplicationPreparedEvent

↓

Context Refreshed

↓

ApplicationStartedEvent

↓

ApplicationReadyEvent
```

If startup fails

```
ApplicationFailedEvent
```

These events allow infrastructure components to react to different startup stages without modifying the startup pipeline.

---

# Complete Internal Architecture

```
Developer

        │

        ▼

publishEvent()

        │

        ▼

ApplicationEventPublisher

        │

        ▼

ApplicationEventMulticaster

        │

        ▼

Lookup Listener Registry

        │

        ▼

ApplicationListenerMethodAdapter

        │

        ▼

Resolve Bean

        │

        ▼

ApplicationContext.getBean()

        │

        ▼

Managed Bean

        │

        ▼

Reflection.invoke()

        │

        ▼

Business Method
```

If transactional:

```
ApplicationListenerMethodAdapter

↓

TransactionSynchronizationManager

↓

AFTER_COMMIT

↓

Invoke Listener
```

---

# Framework Design Decisions

## Why Publisher and Multicaster are separate?

Publisher publishes.

Multicaster dispatches.

Single Responsibility Principle.

---

## Why BeanPostProcessor?

Reuse existing lifecycle extension point.

No additional startup mechanism required.

---

## Why Listener Adapters?

Encapsulate invocation logic.

Supports future extension.

- Conditions
- Ordering
- Async
- Transactional Events

without changing the multicaster.

---

## Why Store Bean Name?

Supports

- Singleton
- Prototype
- Scoped Beans
- Proxies

Always obtain the latest managed bean.

---

## Why Startup Discovery?

Reflection is expensive.

Perform once.

Reuse metadata forever.

---

## Why Java Type Matching?

Reuse inheritance.

No custom matching engine.

---

## Why TransactionSynchronizationManager?

Keeps the Event Framework independent from Transaction Management.

Both frameworks remain reusable.

---

# Internal Working Summary

Startup

```
ApplicationContext

↓

Infrastructure Beans

↓

BeanPostProcessors

↓

Application Beans

↓

Discover @EventListener

↓

Create Listener Adapters

↓

Register Registry
```

Runtime

```
publishEvent()

↓

Publisher

↓

Multicaster

↓

Registry Lookup

↓

Listener Adapter

↓

ApplicationContext.getBean()

↓

Reflection

↓

Business Method
```

Transactional

```
publishEvent()

↓

Register Transaction Callback

↓

Commit

↓

Invoke Listener
```

---

# Mental Models

## Airport Announcement

```
Airport

↓

Announcement

↓

Passengers
```

Airport never knows who reacts.

---

## Courier Service

Normal Event

```
Package Ready

↓

Deliver
```

Transactional Event

```
Package Ready

↓

Hold Delivery

↓

Payment Success?

↓

YES → Deliver

NO → Cancel
```

---

# Source Code Mapping

| Class | Responsibility |
|---------|---------------|
| ApplicationEventPublisher | Publishes events |
| ApplicationEventMulticaster | Dispatches events |
| SimpleApplicationEventMulticaster | Default multicaster implementation |
| EventListenerMethodProcessor | Discovers `@EventListener` |
| ApplicationListenerMethodAdapter | Invokes listener methods |
| TransactionalApplicationListenerMethodAdapter | Integrates listeners with transactions |
| TransactionSynchronizationManager | Registers transaction callbacks |
| AbstractPlatformTransactionManager | Executes synchronization callbacks |

---

# Engineering Principles

### EP-058

An event represents something that has already happened, not a request that must be handled.

### EP-059

Frameworks preprocess metadata during startup and execute precomputed decisions during runtime.

### EP-060

Runtime execution should rely on precomputed metadata rather than repeated discovery.

### EP-061

Frameworks expose simple public APIs while delegating execution to specialized internal components.

### EP-063

Reuse existing extension points instead of introducing new lifecycle phases.

### EP-064

Manage objects as units of lifecycle and methods as units of behavior.

### EP-065

Wrap reflection metadata inside richer abstractions.

### EP-068

Dispatchers coordinate execution; adapters encapsulate invocation logic.

### EP-069

Always retrieve managed objects through the IoC container.

### EP-070

Trade startup work for runtime efficiency.

### EP-071

Build upon the language's type system.

### EP-073

Compose independent subsystems through extension points.

### EP-074

Expose lifecycle events so independent components can react without coupling.

### EP-075

Core framework components should remain unaware of extension-specific behavior.

---

# Interview Questions

## Beginner

- What is Spring Event Framework?
- Difference between `ApplicationEventPublisher` and `ApplicationEventMulticaster`.
- What is `@EventListener`?

---

## Intermediate

- How does Spring discover `@EventListener`?
- What is `ApplicationListenerMethodAdapter`?
- Why are listeners discovered during startup?
- Why does Spring store bean names instead of bean instances?
- Explain event matching using polymorphism.

---

## Advanced

- Explain the complete execution flow from `publishEvent()` to listener invocation.
- Why are Publisher and Multicaster separate?
- Explain `@TransactionalEventListener` internals.
- Explain `TransactionSynchronizationManager`.
- How does Spring Boot use the Event Framework during startup?