# Module 13 - Spring Caching Internals

---

# Overview

Spring Cache provides a declarative mechanism to cache the result of expensive operations. Instead of repeatedly executing the same business logic or querying the database, Spring stores the result in a cache and reuses it for future requests.

Caching improves application performance, reduces database load, decreases response time, and increases overall system scalability.

Spring follows the same architectural philosophy as Transactions, Events, Async, and Scheduling by separating business logic from infrastructure.

---

# Learning Objectives

After completing this module you will understand:

- Why Spring Cache exists
- Internal architecture of Spring Cache
- Role of @EnableCaching
- Cache Proxy
- CacheInterceptor
- CacheManager
- Startup Lifecycle
- Runtime Execution Flow
- Cache Hit
- Cache Miss
- Cache Key Generation
- Cache Storage
- Cache Consistency
- @Cacheable
- @CachePut
- @CacheEvict
- Cache Stampede
- Performance Optimizations

---

# Problem Statement

Many enterprise applications repeatedly execute the same expensive operation.

Example

```java
public Product getProduct(Long id){

    return repository.findById(id);

}
```

Suppose 10,000 users request the same product.

Without caching:

```
Request

↓

Database

↓

Return Product

------------------------

Request

↓

Database

↓

Return Product

------------------------

Request

↓

Database

↓

Return Product
```

Every request performs the same database query even though the result has not changed.

Problems

- High database load
- Increased response time
- Unnecessary CPU usage
- Poor scalability
- Connection pool exhaustion

Caching solves this problem by storing previously computed results.

---

# Declarative Programming

Developer Responsibility

- Business Logic
- Identify cacheable operations

Spring Responsibility

- Cache Lookup
- Cache Storage
- Cache Retrieval
- Key Generation
- Proxy Creation
- Cache Management

Developer only declares:

```java
@Cacheable("products")
public Product getProduct(Long id){

}
```

Spring manages everything else.

---

# Why Spring Cache Exists

Instead of repeatedly querying the database for identical data, Spring introduces a reusable caching infrastructure.

Benefits

- Reduce database load
- Lower response time
- Improve throughput
- Better scalability
- Reduce expensive computations
- Improve user experience

Caching is a trade-off between memory and computation.

---

# Internal Architecture

```
Application Starts
        │
        ▼
@EnableCaching
        │
        ▼
Caching Configuration Imported
        │
        ▼
Infrastructure Beans Registered
        │
        ▼
Bean Definition Creation
        │
        ▼
Bean Creation
        │
        ▼
Cache BeanPostProcessor
        │
        ▼
Create Cache Proxy
        │
        ▼
Application Ready

──────────────────────────────────

Runtime

↓

Method Call

↓

Cache Proxy

↓

Cache Lookup

↓

Cache Hit / Cache Miss

↓

Business Method

↓

Return Result
```

---

# Important Components

## @EnableCaching

Entry point of Spring Cache.

Responsibilities

- Imports caching configuration
- Registers cache infrastructure
- Enables cache annotations

Without this annotation, cache annotations have no effect.

---

## Cache Proxy

Intercepts every method call.

Responsibilities

- Cache Lookup
- Cache Population
- Return Cached Result
- Skip Business Logic when possible

---

## CacheInterceptor

Responsible for:

- Reading cache annotations
- Generating cache key
- Looking up cache
- Invoking business method on cache miss
- Updating cache

---

## CacheManager

Responsible for managing cache instances.

Responsibilities

- Create Cache
- Retrieve Cache
- Manage cache lifecycle

Examples

- ConcurrentMapCacheManager
- RedisCacheManager
- CaffeineCacheManager

---

## Cache

Stores

```
Key

↓

Value
```

Example

```
10

↓

Product Object
```

---

# Startup Lifecycle

### Step 1

Application starts.

---

### Step 2

@EnableCaching imports caching configuration.

---

### Step 3

Caching infrastructure beans are registered.

---

### Step 4

Component scanning creates BeanDefinitions.

---

### Step 5

Beans are created.

- Constructor
- Dependency Injection
- @PostConstruct

---

### Step 6

BeanPostProcessor discovers cache annotations.

---

### Step 7

Cache proxy is created.

---

### Step 8

Proxy bean is stored inside ApplicationContext.

---

# Runtime Execution Flow

```
Developer

↓

Proxy

↓

Cache Lookup

↓

Hit ?

↓

Yes

↓

Return Cached Object

──────────────────────────

No

↓

Business Method

↓

Database

↓

Return Object

↓

Store In Cache

↓

Return Object
```

---

# Cache Hit

Cache already contains requested object.

```
Request

↓

Cache

↓

Return Object
```

Business method is never executed.

Database is never called.

---

# Cache Miss

Requested object is not present.

```
Request

↓

Cache Miss

↓

Business Method

↓

Database

↓

Return Object

↓

Store In Cache

↓

Return Object
```

Future requests become Cache Hits.

---

# Cache Key Generation

Spring must uniquely identify cached data.

Default Strategy

Use method parameters.

Example

```java
getProduct(10)
```

Cache

```
Key = 10

↓

Product Object
```

Method parameters provide deterministic cache lookup.

---

# Cache Storage

Spring stores the entire object.

Not only the identifier.

Example

```
Key = 10

↓

Product Object
```

Reason

The purpose of caching is to avoid database access.

Caching only the ID would still require a database lookup.

Caching is a trade-off between memory and performance.

---

# Cache Consistency

Problem

Database

```
Price = ₹12
```

Cache

```
Price = ₹10
```

Application now returns stale data.

Business Impact

- Wrong Product Price
- Incorrect Orders
- Financial Loss
- Data Inconsistency

Database remains the source of truth.

---

# Cache Update Strategies

## @Cacheable

Used for reading data.

Flow

```
Cache

↓

Hit ?

↓

Return Cached Value

↓

Else

↓

Business Method

↓

Store Result
```

---

## @CachePut

Always executes business logic.

Updates database.

Updates cache with latest value.

Suitable when latest object is already available after update.

---

## @CacheEvict

Updates database.

Removes cache entry.

Next request loads fresh data from database.

Often preferred because it avoids stale cache entries and simplifies consistency.

---

# Cache Stampede

Suppose two requests arrive simultaneously.

```
Request A

↓

Cache Miss

↓

Database

────────────────────

Request B

↓

Cache Miss

↓

Database
```

Both requests execute the same expensive query.

This is called Cache Stampede.

Spring provides

```java
@Cacheable(sync = true)
```

to synchronize cache population for the same key.

---

# Performance Optimizations

Spring performs expensive work during startup.

Examples

- Annotation Scanning
- Proxy Creation
- Metadata Preparation

Runtime performs only

```
Method Call

↓

Cache Lookup

↓

Return Result
```

No repeated annotation scanning.

---

# Complete Execution Flow

```
Application Starts

↓

@EnableCaching

↓

Caching Infrastructure

↓

Bean Definition Creation

↓

Bean Creation

↓

@PostConstruct

↓

BeanPostProcessor

↓

Create Cache Proxy

↓

Store Proxy In IoC

────────────────────────────

Method Call

↓

Proxy

↓

Cache Lookup

↓

Hit ?

↓

Return Cached Object

──────────────

Miss

↓

Business Method

↓

Database

↓

Store In Cache

↓

Return Object
```

---

# Common Interview Questions

- Why is @EnableCaching required?
- How does Spring Cache work internally?
- What is the role of CacheInterceptor?
- Difference between Cache Hit and Cache Miss?
- How are cache keys generated?
- Why does Spring cache the entire object?
- Difference between @Cacheable, @CachePut and @CacheEvict?
- What is Cache Stampede?
- Why is database considered the source of truth?
- What consistency problems exist in caching?

---

# Engineering Principles

## Principle 1

Caching is a trade-off between memory and computation.

---

## Principle 2

Database remains the source of truth.

---

## Principle 3

Frameworks should move expensive work to startup.

---

## Principle 4

Runtime should operate using precomputed infrastructure.

---

## Principle 5

Cache keys must uniquely identify cached results.

---

## Principle 6

Cache consistency should be designed based on business requirements.

---

## Principle 7

Shared infrastructure should manage cross-cutting concerns instead of business code.

---

# Key Takeaways

- Spring Cache reduces expensive operations by storing computed results.
- @EnableCaching registers the caching infrastructure.
- Spring creates a proxy that intercepts cacheable method calls.
- Cache Hit bypasses business logic and database access.
- Cache Miss executes business logic and stores the result.
- Method parameters are used as cache keys by default.
- The complete object is cached to avoid repeated database access.
- Database remains the source of truth.
- @Cacheable reads data, @CachePut updates the cache, and @CacheEvict removes stale cache entries.
- Cache Stampede occurs when multiple requests simultaneously miss the cache and execute the same expensive operation.
- Spring follows the same architectural pattern used in Transactions, Events, Async, and Scheduling. The runtime trigger differs, but the startup lifecycle and infrastructure extension model remain consistent.   