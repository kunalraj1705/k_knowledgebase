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