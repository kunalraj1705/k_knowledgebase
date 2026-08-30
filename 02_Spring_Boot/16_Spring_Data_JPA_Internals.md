# Module 16 - Spring Data JPA Internals

---

# Objective

Understand how Spring Data JPA works internally by deriving its architecture from first principles instead of memorizing annotations. Learn how the Persistence Context manages entities, how Dirty Checking works, why EntityManager exists, and how Hibernate translates object changes into SQL.

---

# Why JPA?

Before JPA, every database interaction required manual JDBC code.

```java
Connection connection = DriverManager.getConnection(...);

PreparedStatement statement =
connection.prepareStatement("SELECT * FROM users WHERE id=?");

ResultSet rs = statement.executeQuery();

User user = new User();
user.setId(...);
user.setName(...);
```

Problems:

- Boilerplate code
- Manual object mapping
- Connection management
- Resource leaks
- Repeated SQL
- Difficult maintenance

Just like Spring MVC simplified HTTP handling and Spring Transactions simplified transaction management, JPA simplifies persistence.

---

# Declarative Programming

Instead of writing SQL manually:

```java
@Entity
public class User {

@Id
private Long id;

private String name;

}
```

```java
public interface UserRepository extends JpaRepository<User, Long> {

}
```

Spring and Hibernate manage persistence infrastructure.

---

# Internal Architecture

```
Repository Proxy
        │
        ▼
SimpleJpaRepository
        │
        ▼
EntityManager
        │
        ▼
Hibernate
        │
        ▼
JDBC
        │
        ▼
Database
```

Every layer has a single responsibility.

---

# EntityManager

EntityManager is the central persistence coordinator.

Responsibilities:

- Persist entities
- Find entities
- Remove entities
- Manage Persistence Context
- Dirty Checking
- Flush changes
- Delegate persistence work to Hibernate

EntityManager does **not** execute SQL directly.

---

# Persistence Context

Persistence Context is **not just a cache**.

It is a transaction-scoped unit of work responsible for:

- Managing entity lifecycle
- Maintaining entity identity
- First-Level Cache
- Dirty Checking
- Snapshot Management
- Database Synchronization

Think of it as:

```
ApplicationContext → Beans

SecurityContext → Authentication

PersistenceContext → Entities
```

---

# First-Level Cache

```
find(User,10)

↓

Persistence Context

↓

Found?

↓

Yes

↓

Return Same Object
```

Database is called only once.

---

# Entity Identity

Inside one Persistence Context:

```java
User u1 = entityManager.find(User.class,10L);

User u2 = entityManager.find(User.class,10L);
```

Both variables reference the same Java object.

```
u1 ───────┐
        │
        ▼
User Object
        ▲
        │
u2 ───────┘
```

---

# Dirty Checking

Hibernate stores:

```
Managed Entity

↓

Snapshot
```

When Flush occurs:

```
Current Entity

↓

Compare

↓

Generate UPDATE SQL
```

Developer never writes update statements.

---

# Dirty Checking Flow

```
Find Entity

↓

Managed Entity

↓

Modify Java Object

↓

Flush

↓

Dirty Checking

↓

Generate UPDATE SQL

↓

Commit
```

---

# Flush

Flush synchronizes Persistence Context with the database.

Flush does:

- Dirty Checking
- Generate SQL
- Execute SQL

Flush does **not**:

- Commit transaction
- End transaction
- Detach entities

---

# Commit

Commit performs:

```
Flush

↓

Database Commit

↓

Transaction Ends
```

Commit makes changes permanent.

---

# Entity Lifecycle

```
new User()

↓

Transient

↓

persist()

↓

Managed

↓

detach()

↓

Detached

↓

merge()

↓

Managed

↓

remove()

↓

Removed
```

---

# Transient

Created using:

```java
User user = new User();
```

Characteristics:

- Exists only in JVM
- Not managed
- No Dirty Checking
- No database row

---

# Managed

Created using:

```java
entityManager.persist(user);
```

or

```java
entityManager.find(...)
```

Characteristics:

- Managed by Persistence Context
- Dirty Checking enabled
- Lifecycle tracked
- Automatically synchronized

---

# Detached

Created using:

```java
entityManager.detach(user);
```

Characteristics:

- Java object still exists
- Persistence Context no longer tracks it
- Dirty Checking disabled

---

# Removed

Created using:

```java
entityManager.remove(user);
```

Entity will be deleted during Flush.

---

# persist()

Purpose:

Register a new entity.

```
Transient

↓

Managed

↓

INSERT
```

persist() always means:

> This is a new entity.

---

# merge()

Purpose:

Synchronize detached entity.

```
Detached Entity

↓

Copy State

↓

Managed Entity

↓

Dirty Checking
```

merge() returns the managed object.

---

# Why merge() creates a managed copy?

Only one managed instance can exist inside one Persistence Context.

```
Detached Entity

↓

Copy Values

↓

Managed Entity
```

Avoids duplicate managed objects.

---

# Lazy Loading

Instead of loading child data immediately:

```
User

↓

Orders Proxy
```

Actual SQL executes only when:

```java
user.getOrders();
```

---

# Lazy Proxy

```
Orders Proxy

↓

Loaded?

↓

No

↓

Execute SQL

↓

Replace Proxy

↓

Return Collection
```

---

# N+1 Query Problem

```
Load Users

↓

1 Query

↓

Loop

↓

Load Orders

↓

100 Queries
```

Total:

```
1 + N Queries
```

Avoid using:

- JOIN FETCH
- EntityGraph
- DTO Projection
- Batch Fetching

---

# EAGER vs LAZY

Never choose one strategy globally.

Choose based on the use case.

Bad:

```
Always LAZY

Always EAGER
```

Correct:

```
Use Case Driven Fetching
```

Load exactly what the current API requires.

---

# LazyInitializationException

```
Transaction Starts

↓

Persistence Context

↓

Orders Proxy

↓

Transaction Ends

↓

Persistence Context Destroyed

↓

user.getOrders()

↓

LazyInitializationException
```

Proxy requires Persistence Context to execute SQL.

---

# Initialized Proxy

If accessed inside transaction:

```java
user.getOrders().size();
```

Then:

```
Proxy

↓

Load Orders

↓

Real Collection

↓

Transaction Ends

↓

Still Accessible
```

Already-loaded data remains in memory.

---

# Repository Internals

```
UserRepository

↓

Repository Proxy

↓

SimpleJpaRepository

↓

EntityManager

↓

Hibernate

↓

JDBC

↓

Database
```

Repository interface has no implementation written by the developer.

Spring creates it dynamically during startup.

---

# EntityManager Lifecycle

EntityManager is **not Singleton**.

Each transaction receives its own EntityManager.

```
Request T1

↓

EntityManager 1

↓

Persistence Context 1

-----------------------

Request T2

↓

EntityManager 2

↓

Persistence Context 2
```

Reason:

- Thread Safety
- Independent Transactions
- Independent Dirty Checking
- Independent Entity Lifecycle

---

# Complete Request Lifecycle

```
Application Starts

↓

@EnableJpaRepositories

↓

Repository Interfaces Found

↓

Repository Proxies Created

↓

Application Ready

---------------------------------

Incoming Request

↓

Tomcat

↓

Security Filters

↓

DispatcherServlet

↓

Controller

↓

@Transactional Proxy

↓

TransactionInterceptor

↓

EntityManager Created

↓

Persistence Context Created

↓

Repository Proxy

↓

SimpleJpaRepository

↓

EntityManager

↓

Hibernate

↓

JDBC

↓

Database

↓

Managed Entity Returned

↓

Business Logic

↓

Entity Modified

↓

Flush

↓

Dirty Checking

↓

Generate SQL

↓

Commit

↓

Close EntityManager

↓

Destroy Persistence Context
```

---

# Engineering Principles Learned

### Principle 1

High-level repository abstractions should delegate persistence work to specialized infrastructure.

---

### Principle 2

Persistence Context is the authoritative owner of managed entities.

---

### Principle 3

A managed entity represents a live transaction-scoped view of a database row.

---

### Principle 4

Delay expensive database operations until Flush.

---

### Principle 5

Synchronization (Flush) and Persistence (Commit) are different responsibilities.

---

### Principle 6

Only managed entities participate in Dirty Checking.

---

### Principle 7

Framework-managed objects should remain authoritative. Detached objects are synchronized through merge(), not adopted directly.

---

### Principle 8

Lazy loading avoids loading data that may never be used.

---

### Principle 9

Performance should be optimized based on access patterns, not individual queries.

---

### Principle 10

Fetch strategy should be driven by business use cases, not by a universal default.

---

# KRB Academy Summary

Spring Data JPA follows the same architectural principles seen throughout Spring:

- Delegation
- Proxy Pattern
- Context Ownership
- Managed Lifecycle
- Runtime Lookup
- Lazy Execution
- Separation of Responsibilities

The Persistence Context is the heart of JPA. It manages entity identity, lifecycle, dirty checking, caching, and synchronization with the database, allowing developers to work with Java objects while Hibernate transparently generates the minimum SQL required.

---

# Interview Questions

## Q1. What is JPA?

JPA (Java Persistence API) is a specification that defines the standard APIs and contracts for object-relational mapping (ORM) in Java applications. It defines interfaces such as EntityManager, entity lifecycle, annotations, and persistence behavior, but it does not provide an implementation.

---

## Q2. What is Hibernate?

Hibernate is a JPA provider (implementation) that implements the JPA specification. It provides features such as SQL generation, Dirty Checking, Lazy Loading, Caching, Proxy creation, and JDBC interaction.

---

## Q3. Difference between JPA and Hibernate?

| JPA                   | Hibernate                         |
| --------------------- | --------------------------------- |
| Specification         | Implementation                    |
| Defines contracts     | Implements contracts              |
| Vendor independent    | Vendor specific                   |
| Defines EntityManager | Implements EntityManager behavior |
| Defines annotations   | Processes annotations             |
| No SQL generation     | Generates SQL                     |
| No Dirty Checking     | Implements Dirty Checking         |
| No Lazy Loading       | Implements Lazy Loading           |

---

## Q4. Why does Spring use JPA instead of Hibernate directly?

Spring depends on the JPA specification rather than Hibernate directly. This follows the Dependency Inversion Principle and allows the JPA provider (Hibernate, EclipseLink, OpenJPA, etc.) to be replaced without changing business code.

Architecture:

Application

↓

Spring Data JPA

↓

JPA Specification

↓

Hibernate / EclipseLink

↓

JDBC

↓

Database

---

## Q5. What is EntityManager?

EntityManager is the central JPA component responsible for managing the Persistence Context. It performs operations such as persist(), find(), merge(), remove(), flush(), and delegates persistence work to Hibernate.

---

## Q6. What is the Persistence Context?

Persistence Context is a transaction-scoped unit of work managed by the EntityManager. It manages entity lifecycle, entity identity, first-level caching, dirty checking, snapshots, and synchronization with the database.

---

## Q7. What is the First-Level Cache?

The First-Level Cache is the cache inside the Persistence Context. Repeated lookups of the same entity within the same Persistence Context return the same managed object without executing additional SQL.

---

## Q8. Why does entityManager.find() return the same object reference?

Because the Persistence Context maintains exactly one managed instance for each database row. Multiple calls to find() return the same managed object to maintain consistency.

---

## Q9. What is Dirty Checking?

Dirty Checking is Hibernate's mechanism for detecting changes made to managed entities. During Flush, Hibernate compares the current entity with its snapshot and automatically generates UPDATE SQL if changes are detected.

---

## Q10. Why don't we call save() after modifying a managed entity?

Managed entities are tracked by the Persistence Context. Hibernate automatically detects changes through Dirty Checking and synchronizes them during Flush or Commit.

---

## Q11. Difference between Flush and Commit?

### Flush

- Synchronizes Persistence Context with Database
- Executes SQL
- Transaction remains active
- Changes can still be rolled back

### Commit

- Flushes pending changes
- Commits database transaction
- Makes changes permanent
- Ends transaction

---

## Q12. Explain the Entity Lifecycle.

Transient

↓

Managed

↓

Detached

↓

Removed

---

## Q13. Difference between persist() and merge()?

persist()

- Used for new entities
- Makes Transient entity Managed
- Generates INSERT

merge()

- Used for Detached entities
- Copies state into a Managed entity
- Generates INSERT or UPDATE depending on existence

---

## Q14. Difference between Managed and Detached Entity?

Managed

- Inside Persistence Context
- Dirty Checking enabled
- Automatically synchronized

Detached

- Outside Persistence Context
- Dirty Checking disabled
- Changes are not tracked

---

## Q15. What is Lazy Loading?

Lazy Loading delays loading of related entities until they are actually accessed. Hibernate initially returns a proxy and executes SQL only when the relationship is used.

---

## Q16. What is the N+1 Query Problem?

One query loads parent entities and then N additional queries are executed to load child entities lazily, resulting in excessive database calls and degraded performance.

---

## Q17. What is LazyInitializationException?

LazyInitializationException occurs when a lazy proxy attempts to load data after the Persistence Context has already been closed.

---

## Q18. How does JpaRepository work internally?

```
Repository Interface

↓

Repository Proxy

↓

SimpleJpaRepository

↓

EntityManager

↓

Hibernate

↓

JDBC

↓

Database
```

---

## Q19. What is SimpleJpaRepository?

SimpleJpaRepository is Spring Data JPA's default implementation of JpaRepository. It implements standard CRUD operations by delegating persistence work to the EntityManager.

---

## Q20. Is EntityManager a Singleton?

No.

EntityManager is not thread-safe.

A new EntityManager (and Persistence Context) is associated with each transaction.

---

## Q21. Explain the complete JPA request lifecycle.

```
Controller

↓

@Transactional Proxy

↓

TransactionInterceptor

↓

EntityManager Created

↓

Persistence Context Created

↓

Repository Proxy

↓

SimpleJpaRepository

↓

EntityManager

↓

Hibernate

↓

JDBC

↓

Database

↓

Managed Entity Returned

↓

Business Logic

↓

Dirty Checking

↓

Flush

↓

Generate SQL

↓

Commit

↓

Close EntityManager
```

---

# Senior Interview Tips

- Always explain **why** JPA exists before discussing Hibernate.
- Differentiate **Specification vs Implementation**.
- Never confuse **Flush** with **Commit**.
- Explain that **Dirty Checking** works only for **Managed Entities**.
- Mention that **Persistence Context** is much more than a cache—it manages identity, lifecycle, snapshots, synchronization, and caching.
- Explain why **EntityManager is not thread-safe**.
- Avoid recommending `FetchType.EAGER` as a universal solution to LazyInitializationException or the N+1 problem. Discuss use-case-driven fetching with `JOIN FETCH`, `@EntityGraph`, or DTO projections instead.

---

# KRB Academy Interview Readiness

After completing this module, you should be able to confidently explain:

- Why JPA exists
- JPA vs Hibernate
- EntityManager
- Persistence Context
- Dirty Checking
- Flush vs Commit
- Entity Lifecycle
- persist() vs merge()
- Lazy Loading
- N+1 Query Problem
- LazyInitializationException
- Repository Internals
- SimpleJpaRepository
- Complete JPA Architecture

