# 11. Spring Async Internals

---

# Learning Objective

By the end of this module, you will understand:

- Why Spring introduced asynchronous execution
- Why `@Async` is implemented using Spring AOP
- How `@EnableAsync` enables the async infrastructure
- How `AsyncAnnotationBeanPostProcessor` discovers `@Async`
- How Spring creates async proxies
- Role of `AsyncExecutionInterceptor`
- Internal working of `TaskExecutor`
- Thread Pool architecture
- Why self-invocation bypasses `@Async`
- Exception handling in asynchronous execution
- `Future` and `CompletableFuture`
- ThreadPoolTaskExecutor internals
- Thread pool sizing and queue management
- Production considerations and common pitfalls

---

# Prerequisites

Before starting this module, you should understand:

- IoC Container
- Bean Lifecycle
- BeanPostProcessor
- Spring AOP
- Proxy Architecture
- Advisor
- Advice
- Interceptor
- Spring Events

---

# Why This Feature Exists

## Problem Statement

Consider the following implementation.

```java
@Service
public class ReportService {

    public void generateReport(){

        // Generate report
        // Takes 30 seconds

    }

}
```

Controller

```java
@GetMapping("/report")
public String report(){

    reportService.generateReport();

    return "Done";

}
```

Execution Flow

```
Client

↓

Controller

↓

ReportService

↓

Generate Report

↓

30 Seconds

↓

Return Response
```

Problem

The client waits for the report generation to complete.

Business Requirement

Generate the report in the background and immediately return the response.

---

# Traditional Java Solution

```java
new Thread(() -> generateReport()).start();
```

Looks simple.

But now the developer becomes responsible for:

- Thread Creation
- Thread Lifecycle
- Thread Cleanup
- Thread Reuse
- Exception Handling
- Resource Management

This violates the principle of separating business logic from infrastructure concerns.

---

# Spring Solution

Instead of manually creating threads,

Spring allows the developer to simply express intent.

```java
@Async
public void generateReport(){

}
```

Developer responsibility

```
Business Logic
```

Spring responsibility

```
Thread Management

↓

Scheduling

↓

Thread Reuse

↓

Exception Handling
```

---

# Why Not Create Threads Manually?

Suppose every request creates a new thread.

```
1000 Requests

↓

1000 Threads
```

Problems

- High Memory Consumption
- Native Thread Creation Cost
- Context Switching Overhead
- Poor Scalability
- Difficult Monitoring

Creating a thread is expensive because it requires:

- JVM Thread Object
- Native Operating System Thread
- Stack Memory
- Scheduler Registration

Instead of repeatedly creating and destroying threads,

Spring reuses existing worker threads.

---

# Solution Architecture

```
Developer

↓

@Async

↓

Spring Proxy

↓

AsyncExecutionInterceptor

↓

TaskExecutor

↓

Thread Pool

↓

Worker Thread

↓

Business Method
```

Notice

The developer never interacts with threads directly.

---

# Evolution

Initially developers used

```java
new Thread(...)
```

Later

```java
ExecutorService
```

Although better,

every project implemented thread pools differently.

Spring introduced

```java
@Async
```

to standardize asynchronous execution.

Internally,

Spring still delegates to Java's Executor framework,

but hides the infrastructure behind a simple annotation.

---

# Core Components

## @EnableAsync

Enables asynchronous processing.

Registers the async infrastructure during container startup.

---

## AsyncAnnotationBeanPostProcessor

Discovers

```
@Async
```

during bean creation.

Creates proxies for async beans.

---

## AsyncAnnotationAdvisor

Responsible for identifying methods requiring asynchronous execution.

Supplies the AsyncExecutionInterceptor.

---

## AsyncExecutionInterceptor

Intercepts method calls.

Creates asynchronous tasks.

Delegates execution to the TaskExecutor.

It never creates threads directly.

---

## TaskExecutor

Spring abstraction over Java executors.

Responsible for:

- Thread Pool
- Task Queue
- Worker Threads
- Scheduling

---

## ThreadPoolTaskExecutor

Default Spring implementation.

Internally wraps

```
ThreadPoolExecutor
```

---

# Internal Architecture

```
Developer

↓

Proxy

↓

AsyncExecutionInterceptor

↓

TaskExecutor

↓

Thread Pool

↓

Worker Thread

↓

Original Bean
```

Each component has a single responsibility.

---

# Startup Phase

Application Starts

↓

Process Configuration Classes

↓

@EnableAsync

↓

Import Async Configuration

↓

Register AsyncAnnotationBeanPostProcessor

↓

Begin Bean Creation

↓

Reflection

↓

Java Object

↓

Dependency Injection

↓

@PostConstruct

↓

BeanPostProcessor

↓

Detect @Async

↓

Create Proxy

↓

Return Proxy to IoC Container

Startup processing is complete.

No annotation scanning occurs during runtime.

---

# Runtime Phase

Developer invokes

```java
reportService.generateReport();
```

Execution

```
Developer

↓

Async Proxy

↓

AsyncExecutionInterceptor

↓

Create Runnable / Callable

↓

Submit Task

↓

TaskExecutor

↓

Thread Pool

↓

Worker Thread

↓

Business Method
```

The caller immediately returns.

Business logic executes independently.

---

# Engineering Principles

### EP-076

Frameworks centralize infrastructure concerns such as thread management so developers can focus on business logic.

---

### EP-077

Frameworks maximize consistency by reusing existing lifecycle extension points instead of inventing new processing pipelines.

---

### EP-078

Interceptors coordinate execution while delegating resource management to specialized infrastructure components.

---

# Mental Model

Think of ordering food online.

Customer

↓

Place Order

↓

Receive Order Confirmation

↓

Kitchen Starts Cooking

↓

Delivery Later

The customer does not wait in the kitchen.

Likewise,

the caller does not wait for asynchronous execution.

---

# Module Summary

In this chapter we learned:

- Why asynchronous execution exists
- Why Spring uses proxies
- Startup architecture
- Runtime execution
- Core infrastructure components
- Separation of responsibilities


---

# Async Infrastructure Internals

Everything we learned so far explains **why** Spring provides asynchronous execution.

Now let's understand **how Spring builds the async infrastructure during container startup**.

---

# Problem Statement

Suppose the developer writes:

```java
@Configuration
@EnableAsync
public class AppConfig {

}
```

and

```java
@Service
public class ReportService {

    @Async
    public void generateReport(){

    }

}
```

Question:

How does Spring know that asynchronous execution should be enabled?

Java does not understand:

```
@EnableAsync
```

Some component inside Spring must discover it and register the required infrastructure.

---

# Internal Architecture

```
Application Starts

↓

@Configuration

↓

@EnableAsync

↓

@Import

↓

AsyncConfigurationSelector

↓

ProxyAsyncConfiguration

↓

Register Infrastructure Beans

↓

AsyncAnnotationBeanPostProcessor

↓

Bean Creation Begins
```

Notice

Infrastructure is created **before** application beans.

---

# Why @EnableAsync Exists

Suppose Spring enabled async processing automatically.

Every application would contain:

- Async BeanPostProcessor
- Async Advisor
- Async Interceptor
- Thread Pool

even if the application never used

```
@Async
```

This would waste

- Startup Time
- Memory
- Processing

Instead Spring follows an opt-in architecture.

Only applications requiring async execution enable it.

---

# Stage 1

Spring processes configuration classes.

```
@Configuration
```

↓

Configuration Parser

↓

Read Metadata

---

# Stage 2

Spring discovers

```
@EnableAsync
```

Internally,

this annotation contains

```java
@Import(AsyncConfigurationSelector.class)
```

Notice

The annotation itself contains no business logic.

Its only responsibility is

```
Import Configuration
```

---

# Stage 3

Spring loads

```
AsyncConfigurationSelector
```

Responsibility

Choose the correct async configuration.

Conceptually

```
@EnableAsync

↓

AsyncConfigurationSelector

↓

ProxyAsyncConfiguration
```

---

# Stage 4

ProxyAsyncConfiguration registers infrastructure beans.

Conceptually

```
AsyncAnnotationBeanPostProcessor

↓

AsyncAnnotationAdvisor

↓

AsyncExecutionInterceptor
```

At this point,

the async subsystem becomes part of the IoC container.

---

# Bean Creation Begins

Only after infrastructure registration does Spring create application beans.

```
Bean Definition

↓

Reflection

↓

Java Object

↓

Dependency Injection

↓

@PostConstruct

↓

BeanPostProcessor
```

---

# AsyncAnnotationBeanPostProcessor

Suppose we have

```java
@Service
public class ReportService {

    @Async
    public void generateReport(){

    }

}
```

The BeanPostProcessor inspects every bean.

Reflection discovers

```
Method

↓

Annotation

↓

@Async
```

---

# Proxy Creation

If no async annotation exists

```
Return Original Bean
```

If async annotation exists

```
Original Bean

↓

Create Proxy

↓

Add Async Advisor

↓

Return Proxy
```

Notice

The IoC container stores

```
Async Proxy
```

not

```
Original Bean
```

Every dependency injected into the application receives the proxy.

---

# Async Advisor

The advisor contains

```
Pointcut

↓

Methods annotated with @Async

Advice

↓

AsyncExecutionInterceptor
```

Exactly the same architecture as

```
@Transactional

↓

Transaction Advisor

↓

TransactionInterceptor
```

Spring reuses its AOP infrastructure.

---

# Runtime Execution

Suppose

```java
reportService.generateReport();
```

Execution

```
Developer

↓

Proxy

↓

Async Advisor

↓

AsyncExecutionInterceptor
```

The interceptor does not execute the business method.

Instead

```
Create Runnable

↓

Submit Task

↓

TaskExecutor
```

Execution responsibility is delegated.

---

# TaskExecutor

TaskExecutor receives

```
Runnable

or

Callable
```

Responsibilities

- Thread Management
- Queue Management
- Worker Selection
- Thread Reuse

It never performs business logic.

---

# ThreadPoolTaskExecutor

Default Spring implementation

Internally wraps

```
ThreadPoolExecutor
```

Architecture

```
TaskExecutor

↓

Thread Pool

↓

Worker Threads

↓

Queue

↓

Business Method
```

---

# Thread Pool Algorithm

Suppose

```
Core Pool = 5

Maximum Pool = 10

Queue = 100
```

Task arrives

↓

Current Threads < Core Pool

↓

Create Thread

Once Core Pool becomes full

↓

Store Task in Queue

Queue Full?

↓

No

↓

Continue Waiting

Queue Full?

↓

Yes

↓

Current Threads < Maximum Pool

↓

Create Additional Thread

Maximum Pool Reached?

↓

Reject Task

---

# RejectedExecutionHandler

Suppose

```
Core Pool = 5

Maximum Pool = 10

Queue = 100
```

Current State

```
10 Threads Busy

↓

100 Waiting Tasks
```

Task 111 arrives.

No worker available.

No queue space.

No additional thread allowed.

Spring delegates to

```
RejectedExecutionHandler
```

Possible actions

- Reject Exception
- Caller Executes Task
- Discard Task
- Discard Oldest Task

The strategy depends on application requirements.

---

# Why Not Unlimited Queue?

Suppose

```
Queue = Integer.MAX_VALUE
```

Traffic

```
20,000 Requests/sec
```

Workers

```
5 Threads
```

Processing

```
100 Tasks/sec
```

Queue continuously grows.

Eventually

- Heap Consumption
- Garbage Collection
- CPU Saturation
- High Latency
- OutOfMemoryError

Unlimited queues delay failure.

They do not eliminate it.

---

# Backpressure

Healthy systems reject excess work.

Instead of

```
Accept Everything

↓

Crash
```

Spring prefers

```
Reject Excess Work

↓

System Remains Healthy
```

Rejected requests are recoverable.

System crashes are not.

---

# Capacity Planning

Increasing threads does not necessarily increase throughput.

Suppose

```
500 Threads
```

Database

```
30 Connections
```

Result

```
470 Threads Waiting
```

The application becomes slower.

Throughput is limited by

```
Slowest Downstream Dependency
```

Examples

- Database
- Kafka
- Redis
- Payment Gateway
- External APIs

Thread pools must be tuned considering downstream capacity.

---

# Internal Working Summary

```
Application Starts

↓

@EnableAsync

↓

@Import

↓

AsyncConfigurationSelector

↓

ProxyAsyncConfiguration

↓

Register AsyncAnnotationBeanPostProcessor

──────────────────────────

Bean Creation

──────────────────────────

Reflection

↓

@PostConstruct

↓

BeanPostProcessor

↓

Detect @Async

↓

Create Proxy

↓

Register Proxy

──────────────────────────

Runtime

──────────────────────────

Developer

↓

Proxy

↓

AsyncExecutionInterceptor

↓

TaskExecutor

↓

Thread Pool

↓

Queue

↓

Worker Thread

↓

Business Method
```

---

# Framework Design Decisions

## Why @Import?

Separates annotation declaration from infrastructure creation.

---

## Why BeanPostProcessor?

Reuse existing lifecycle extension point.

---

## Why Proxy?

No modification to business code.

---

## Why Thread Pool?

Thread creation is expensive.

Reuse improves performance.

---

## Why Queue Before Maximum Threads?

Absorb short traffic bursts without creating unnecessary threads.

---

## Why Reject Tasks?

Protect the system from resource exhaustion.

---

# Engineering Principles

### EP-079

Proxy-based features apply only to calls entering through the proxy.

---

### EP-080

The IoC container manages proxy objects as the public representation of beans.

---

### EP-084

Execution frameworks absorb short bursts through bounded queues and reject excess work to preserve stability.

---

### EP-085

Bounded systems fail gracefully by applying backpressure.

---

### EP-086

System throughput is constrained by the slowest downstream dependency.

---

### EP-087

Framework features should be opt-in.

---

### EP-088

Framework annotations declare intent while imported configuration builds infrastructure.

---

# Advanced Async Concepts

Everything we have learned so far explains how Spring executes asynchronous methods.

Now we will study the situations where asynchronous execution behaves differently.

---

# Self Invocation

Consider the following service.

```java
@Service
public class ReportService {

    @Async
    public void generateReport() {

        System.out.println("Generating Report");

    }

    public void processReport() {

        generateReport();

    }

}
```

Looks correct.

Question

Will

```
generateReport()
```

execute asynchronously?

Answer

No.

---

# Why?

Execution Flow

```
Developer

↓

Async Proxy

↓

processReport()

↓

Original Object

↓

this.generateReport()
```

Notice

The proxy has already delegated execution to the target object.

The second method call

```
this.generateReport()
```

never re-enters the proxy.

Therefore

```
AsyncExecutionInterceptor
```

is never executed.

The business method executes synchronously.

---

# Why Self Injection Works

Suppose

```java
@Autowired
private ReportService self;
```

Now

```java
self.generateReport();
```

Execution

```
Target Object

↓

Proxy

↓

AsyncExecutionInterceptor

↓

TaskExecutor

↓

Worker Thread

↓

Business Method
```

Notice

The call crosses the proxy boundary.

Therefore interception occurs.

Although functional,

self-injection is generally considered a workaround.

The preferred solution is separating responsibilities into different beans.

---

# Exception Handling

Suppose

```java
@Async
public void process(){

    throw new RuntimeException();

}
```

Question

Who receives the exception?

Execution

```
Caller Thread

↓

Proxy

↓

Return Immediately

────────────────────

Worker Thread

↓

Business Method

↓

Exception
```

The caller has already returned.

Therefore

the exception cannot propagate back through the original call stack.

Spring delegates uncaught exceptions to

```
AsyncUncaughtExceptionHandler
```

---

# Future

Suppose

```java
@Async
public Future<String> process(){

}
```

Execution

```
Caller

↓

Future Returned

↓

Continue Working

↓

Worker Thread

↓

Complete Future

↓

Caller Reads Result Later
```

Future represents

```
A Result That Will Exist Later
```

---

# CompletableFuture

```java
@Async
public CompletableFuture<String> process(){

}
```

Advantages

- Non-blocking
- Callback Support
- Composition
- Better Exception Handling

Execution

```
Caller

↓

CompletableFuture

↓

Continue Processing

↓

Worker Thread

↓

Complete Future
```

---

# Async and Transaction

Suppose

```java
@Async
@Transactional
public void processPayment(){

}
```

Question

Which interceptor executes first?

Answer

```
AsyncExecutionInterceptor

↓

TransactionInterceptor
```

---

# Why?

Transactions are stored using

```
ThreadLocal
```

If the transaction starts before switching threads

```
Caller Thread

↓

Transaction Begins

↓

Worker Thread

↓

Business Logic
```

The worker thread has no transaction.

Instead

Spring performs

```
Caller Thread

↓

AsyncExecutionInterceptor

↓

Worker Thread

↓

TransactionInterceptor

↓

Business Method

↓

Commit
```

The transaction is created inside the thread performing the work.

---

# Complete Runtime Execution

```
Developer

↓

Proxy

↓

Async Advisor

↓

AsyncExecutionInterceptor

↓

Create Runnable

↓

TaskExecutor

↓

Thread Pool

↓

Worker Thread

↓

TransactionInterceptor (Optional)

↓

Business Method

↓

Complete Future

↓

Return Thread To Pool
```

---

# Source Code Walkthrough

Startup

```
SpringApplication.run()

↓

ApplicationContext

↓

@Configuration

↓

@EnableAsync

↓

@Import

↓

AsyncConfigurationSelector

↓

ProxyAsyncConfiguration

↓

Register AsyncAnnotationBeanPostProcessor
```

Bean Creation

```
Bean Definition

↓

Reflection

↓

Java Object

↓

Dependency Injection

↓

@PostConstruct

↓

AsyncAnnotationBeanPostProcessor

↓

Detect @Async

↓

Create Proxy

↓

Register Proxy
```

Runtime

```
Developer

↓

Async Proxy

↓

AsyncExecutionInterceptor

↓

TaskExecutor

↓

ThreadPoolTaskExecutor

↓

ThreadPoolExecutor

↓

Worker Thread

↓

Business Method
```

---

# Complete Internal Architecture

```
Application Startup

        │

        ▼

@EnableAsync

        │

        ▼

@Import

        │

        ▼

AsyncConfigurationSelector

        │

        ▼

ProxyAsyncConfiguration

        │

        ▼

Register AsyncAnnotationBeanPostProcessor

──────────────────────────────────────────

Bean Creation

──────────────────────────────────────────

Reflection

        │

        ▼

@PostConstruct

        │

        ▼

BeanPostProcessor

        │

        ▼

Detect @Async

        │

        ▼

Create Proxy

        │

        ▼

Store Proxy In IoC Container

──────────────────────────────────────────

Runtime

──────────────────────────────────────────

Developer

        │

        ▼

Async Proxy

        │

        ▼

AsyncExecutionInterceptor

        │

        ▼

TaskExecutor

        │

        ▼

ThreadPoolExecutor

        │

        ├────────► Worker Available

        │              │

        │              ▼

        │       Execute Task

        │

        └────────► Queue

                        │

                        ▼

                 Worker Available

                        │

                        ▼

                 Execute Task

                        │

                        ▼

              Complete Future

                        │

                        ▼

             Return Thread To Pool
```

---

# Mental Models

## Restaurant

```
Customer

↓

Order Food

↓

Receive Token

↓

Continue Shopping

↓

Kitchen

↓

Food Ready
```

The customer does not wait inside the kitchen.

The token is equivalent to

```
CompletableFuture
```

---

## Movie Theatre

```
Cashier Busy

↓

Join Queue

↓

Counter Available

↓

Ticket Issued
```

Queue absorbs temporary spikes.

Unlimited queues eventually exhaust resources.

---

## Internal Working Summary

```
@EnableAsync

↓

Register Infrastructure

↓

Create Bean

↓

BeanPostProcessor

↓

Detect @Async

↓

Create Proxy

↓

Developer Calls Method

↓

AsyncExecutionInterceptor

↓

TaskExecutor

↓

ThreadPoolExecutor

↓

Queue

↓

Worker Thread

↓

Business Logic

↓

Future Completion
```

---

# Interview Questions

## Beginner

- What is @Async?
- Why use asynchronous execution?
- Difference between synchronous and asynchronous execution?

---

## Intermediate

- How does Spring implement @Async?
- What is AsyncAnnotationBeanPostProcessor?
- What is AsyncExecutionInterceptor?
- Why does self-invocation fail?
- Difference between Future and CompletableFuture?

---

## Senior

- Explain the complete startup flow of @Async.
- Explain the runtime execution pipeline.
- Why does AsyncExecutionInterceptor delegate to TaskExecutor?
- Why is ThreadPoolExecutor preferred over new Thread()?
- Explain queue management.
- Explain RejectedExecutionHandler.
- Explain backpressure.
- Why does @Async execute before @Transactional?
- Why are transactions thread-bound?

---

# Source Code Mapping

| Spring Class | Responsibility |
|--------------|----------------|
| EnableAsync | Enables async infrastructure |
| AsyncConfigurationSelector | Imports async configuration |
| ProxyAsyncConfiguration | Registers infrastructure beans |
| AsyncAnnotationBeanPostProcessor | Detects @Async |
| AsyncAnnotationAdvisor | Defines pointcut and advice |
| AsyncExecutionInterceptor | Intercepts async methods |
| TaskExecutor | Spring executor abstraction |
| ThreadPoolTaskExecutor | Default implementation |
| ThreadPoolExecutor | Java thread pool implementation |
| AsyncUncaughtExceptionHandler | Handles uncaught async exceptions |

---

# Engineering Principles

### EP-076

Frameworks centralize infrastructure concerns, allowing developers to focus on business logic.

### EP-077

Frameworks maximize consistency by reusing existing lifecycle extension points.

### EP-078

Interceptors coordinate execution while delegating resource management.

### EP-079

Proxy-based features apply only to calls entering through the proxy.

### EP-080

The IoC container exposes proxy objects as managed beans.

### EP-081

Asynchronous execution separates caller and worker thread control flow.

### EP-082

A Future represents a promise for a result that is not yet available.

### EP-083

Thread-bound infrastructure must be initialized in the thread that performs the work.

### EP-084

Execution frameworks absorb temporary load using bounded queues.

### EP-085

Backpressure preserves system stability by rejecting excess work.

### EP-086

System throughput is constrained by the slowest downstream dependency.

### EP-087

Framework features should be opt-in.

### EP-088

Framework annotations declare intent while imported configurations build infrastructure.

---

# Connections

Previous Modules

```
IoC

↓

Bean Lifecycle

↓

BeanPostProcessor

↓

Spring AOP

↓

Spring Events

↓

Spring Async
```

Next Module

```
Spring Scheduling Internals

↓

@EnableScheduling

↓

ScheduledAnnotationBeanPostProcessor

↓

TaskScheduler

↓

Cron Parser

↓

Scheduler Thread
```