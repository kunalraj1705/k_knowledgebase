# Module 17 – Advanced Spring Data JPA & Hibernate

---

# Objective

Understand how Spring Data JPA works internally beyond CRUD operations by deriving Hibernate's architecture from first principles instead of memorizing annotations.

This module focuses on:

- JPQL Internals
- SQL Generation
- Query Optimization
- Fetch Strategies
- Performance Optimization
- Dynamic Query Building
- Caching
- Locking
- Production Design Decisions

Rather than learning *what* annotations exist, we learn *why* they exist.

---

# Why Advanced Spring Data JPA?

Module 16 taught us how JPA manages entities.

Now the question becomes:

How does Hibernate actually execute queries efficiently?

Consider this JPQL:

```java
SELECT u
FROM User u
WHERE u.age > 25
```

Questions naturally arise:

- How does Hibernate understand this query?
- How does it know which table to use?
- How does it convert JPQL into SQL?
- How does it optimize repeated execution?
- Why doesn't it concatenate SQL strings?
- Why is JPQL database independent?

This module answers all of these questions.

---

# Internal Architecture

```

Application
│
▼
Service
│
▼
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
JPQL Parser
│
▼
Query Translator
│
▼
SQL Generator
│
▼
JDBC
│
▼
Database

```

Notice something.

Module 16 stopped at Hibernate.

Module 17 opens Hibernate and studies its internal components.

---

# Why JPQL?

Databases understand SQL.

Applications understand Java Objects.

Suppose we have

```java
@Entity
class User{

    Long id;

    String name;

}
```

The database has

```

users

id

name

```

The application thinks in terms of

```

User

id

name

```

The database thinks in terms of

```

users

id

name

```

Someone must translate between the two.

That is Hibernate's responsibility.

Instead of writing

```sql
SELECT *
FROM users
WHERE id = 1;
```

we write

```java
SELECT u
FROM User u
WHERE u.id = :id
```

Hibernate converts it into SQL.

---

# JPQL

JPQL stands for

```

Java Persistence Query Language

```

Characteristics

- Object Oriented
- Entity Based
- Relationship Aware
- Database Independent

JPQL never queries tables directly.

It queries entities.

Instead of

```sql
users
```

we write

```java
User
```

Instead of

```sql
department_id
```

we write

```java
user.department
```

Hibernate resolves everything using entity metadata.

---

# JPQL Internal Flow

JPQL is **not** executed directly.

Hibernate first parses it.

```

JPQL

↓

Parser

↓

Query Model

↓

Entity Metadata

↓

SQL Generator

↓

Database Dialect

↓

SQL

↓

JDBC

↓

Database

```

Every stage has a different responsibility.

---

# Step 1 – Parsing

Suppose we execute

```java
SELECT u
FROM User u
WHERE u.age > 25
```

Hibernate does not immediately generate SQL.

Instead it parses the query.

Conceptually

```

SELECT

↓

Entity = User

↓

Alias = u

↓

Condition

↓

age > 25

```

This creates an internal query model.

---

# Why Parsing?

Imagine Hibernate generated SQL by string replacement.

Example

```

Replace User

↓

users

↓

Replace age

↓

age

↓

Done

```

This approach would fail for

- Nested queries
- Functions
- Joins
- Aggregations
- Aliases
- Subqueries

Instead Hibernate builds a structured representation first.

Exactly like a compiler.

---

# Query Model

After parsing

Hibernate creates something conceptually similar to

```

SELECT

↓

FROM User

↓

WHERE

↓

age > 25

↓

ORDER BY

↓

LIMIT

```

This model is database independent.

No SQL has been generated yet.

---

# Entity Metadata

How does Hibernate know

```

User

↓

users
```

Answer

Entity Metadata.

During startup

Hibernate scans

```java
@Entity
@Table(name="users")
```

and builds metadata.

Conceptually

```

User

↓

Table = users

↓

id

↓

Primary Key

↓

Columns

↓

Relationships

```

Later

JPQL uses this metadata.

---

# SQL Generation

Once the query model is complete

Hibernate asks

```

Which Database?

```

Suppose

```

MySQL

```

Generate MySQL SQL.

Suppose

```

PostgreSQL

```

Generate PostgreSQL SQL.

Suppose

```

Oracle

```

Generate Oracle SQL.

This is why JPQL remains portable.

Hibernate adapts SQL generation using the configured database dialect.

---

# Database Dialect

A Dialect teaches Hibernate how a database speaks.

Examples

```

MySQL Dialect

PostgreSQL Dialect

Oracle Dialect

SQL Server Dialect

```

Hibernate asks the dialect

```

How do I write LIMIT?

```

MySQL

```sql
LIMIT 20
```

Oracle

Uses different syntax.

Application code never changes.

Only the dialect changes.

---

# Query Plan Cache

Parsing JPQL is expensive.

Suppose every request executes

```java
SELECT u
FROM User u
WHERE u.id = :id
```

If Hibernate parsed the query every time

```
Request 1

↓

Parse JPQL

↓

Generate Query Model

↓

Generate SQL

----------------------

Request 2

↓

Parse Again

↓

Generate Query Model

↓

Generate SQL

----------------------

Request 3

↓

Parse Again
```

The same work is repeated for every request.

This is inefficient.

---

# Why Query Plan Cache Exists?

Instead of repeatedly parsing JPQL,

Hibernate parses it only once.

```
JPQL

↓

Already Parsed?

↓

Yes

↓

Reuse Query Plan

↓

Generate SQL
```

The parsing cost is paid only once.

Future executions reuse the cached query structure.

Only parameter values change.

Example

```java
SELECT u
FROM User u
WHERE u.id = :id
```

Execution 1

```
:id = 10
```

Execution 2

```
:id = 50
```

Execution 3

```
:id = 120
```

The query structure is identical.

Only parameters change.

Therefore Hibernate reuses the same Query Plan.

---

# Internal Flow

```
JPQL

↓

Parser

↓

Query Plan

↓

Store in Cache

↓

Future Requests

↓

Reuse Query Plan

↓

Generate SQL

↓

Execute
```

This significantly reduces CPU overhead.

---

# Native Queries

JPQL cannot express every database feature.

Examples

- Database specific functions
- Recursive queries
- Window functions
- Vendor specific syntax
- Stored Procedures

In such cases

Hibernate allows Native SQL.

Example

```java
@Query(
    value = "SELECT * FROM users",
    nativeQuery = true
)
```

Now Hibernate skips JPQL parsing.

Flow

```
Native SQL

↓

Hibernate

↓

JDBC

↓

Database
```

Hibernate does not translate Native SQL.

It sends it directly to the database.

---

# JPQL vs Native Query

| JPQL | Native SQL |
|------|------------|
| Entity Oriented | Table Oriented |
| Portable | Vendor Specific |
| Parsed by Hibernate | Executed Directly |
| Recommended | Use Only When Necessary |

---

# Why Prefer JPQL?

Suppose the company migrates

```
MySQL

↓

PostgreSQL
```

JPQL

```
No Changes
```

Native SQL

```
May Require Changes
```

JPQL protects the application from vendor specific syntax.

---

# JOIN

Suppose

```
User

↓

Department
```

We want users belonging to

```
Engineering
```

JPQL

```java
SELECT u
FROM User u
JOIN u.department d
WHERE d.name='Engineering'
```

Purpose

```
Filtering
```

JOIN allows filtering based on related entities.

However

JOIN does **not** initialize lazy relationships.

After executing

```
User

↓

Department Proxy
```

The Department is still lazily loaded.

---

# JOIN FETCH

Suppose every API response needs

- User
- Department

Using JOIN

Later

```java
user.getDepartment()
```

Hibernate executes another SQL.

With

100 Users

```
Users Query

↓

100 Additional Queries
```

This is the N+1 problem.

JOIN FETCH solves it.

```java
SELECT u
FROM User u
JOIN FETCH u.department
```

Now

```
Users

+

Departments

↓

One SQL
```

The Department is initialized immediately.

---

# JOIN vs JOIN FETCH

| JOIN | JOIN FETCH |
|------|------------|
| Used for Filtering | Used for Filtering + Loading |
| Association remains Lazy | Association initialized |
| May produce N+1 | Eliminates N+1 |
| Child loaded later | Child loaded immediately |

---

# N+1 Query Problem

Suppose

```
100 Users
```

First Query

```sql
SELECT *
FROM users;
```

Later

```java
for(User u : users){

    u.getDepartment();

}
```

Hibernate executes

```
SELECT department
WHERE id=1

SELECT department
WHERE id=2

SELECT department
WHERE id=3

...

100 Queries
```

Total

```
1 + 100

=

101 Queries
```

This is called

```
N+1 Query Problem
```

---

# Solving N+1

Possible solutions

- JOIN FETCH
- DTO Projection
- Batch Fetching
- EntityGraph (covered later)

There is no universal solution.

Choose according to the use case.

---

# Cartesian Product Problem

Suppose

```
User

↓

Orders

↓

Addresses
```

Both are collections.

Now we write

```java
JOIN FETCH orders

JOIN FETCH addresses
```

Hibernate produces one SQL.

Conceptually

```
User

↓

Order1

↓

Address1

↓

Order1

↓

Address2

↓

Order2

↓

Address1

↓

Order2

↓

Address2
```

Rows multiply.

This multiplication is called

```
Cartesian Product
```

Instead of reducing work

we increased it.

More JOIN FETCH is **not** always better.

---

# KRB Academy Summary

Hibernate treats JPQL like a programming language rather than a SQL string. It parses JPQL into an internal query model, resolves entity metadata, generates database-specific SQL using the configured dialect, and caches the parsed query through the Query Plan Cache.

JOIN and JOIN FETCH solve different problems. JOIN filters entities while JOIN FETCH eagerly loads associations to eliminate the N+1 Query Problem. However, JOIN FETCH should be used carefully because fetching multiple collections can lead to Cartesian Products and increased memory usage.

---

# DTO Projection

---

# Why DTO Projection?

Suppose our User entity contains

```java
@Entity
public class User{

    @Id
    private Long id;

    private String firstName;

    private String lastName;

    private String email;

    private String phone;

    private String address;

    private String password;

    @OneToMany
    private List<Order> orders;

}
```

Now suppose an API returns

```
User List
```

The UI only displays

- Name
- Email

Question

Should Hibernate load

- Password
- Address
- Orders
- Phone

No.

Those fields are never used.

Loading unnecessary data increases

- Heap Usage
- Network Traffic
- Persistence Context Size
- Dirty Checking Cost

This violates one of our engineering principles.

```
Load only what you need.
```

---

# What is DTO Projection?

DTO stands for

```
Data Transfer Object
```

Instead of returning an Entity

```java
User
```

Hibernate directly creates

```java
UserDto
```

Example

```java
public record UserDto(

        Long id,

        String name,

        String email

){}
```

JPQL

```java
SELECT new
com.krbacademy.dto.UserDto(

u.id,

u.name,

u.email

)

FROM User u
```

Hibernate only selects

```
id

name

email
```

Nothing else.

---

# Internal Flow

```
JPQL

↓

SQL

↓

ResultSet

↓

DTO Constructor

↓

Application
```

Notice something.

No Entity is created.

---

# Entity Flow

```
Database

↓

Entity

↓

Persistence Context

↓

Dirty Checking

↓

Application
```

---

# DTO Flow

```
Database

↓

DTO

↓

Application
```

Persistence Context is skipped.

Dirty Checking is skipped.

Snapshot creation is skipped.

---

# Why is DTO Projection Faster?

Suppose

```
User

20 Columns
```

API requires

```
3 Columns
```

Entity

```
Load

20 Columns

↓

Persistence Context

↓

Dirty Checking

↓

Application
```

DTO

```
Load

3 Columns

↓

DTO

↓

Application
```

Less work.

Less memory.

Faster serialization.

---

# DTO Projection Characteristics

- Immutable
- Read Only
- Not Managed
- No Dirty Checking
- No Persistence Context
- Better Performance

---

# Interface Projection

Spring provides another projection mechanism.

Instead of DTO

Create

```java
public interface UserProjection{

    Long getId();

    String getName();

    String getEmail();

}
```

Repository

```java
List<UserProjection>

findAllProjectedBy();
```

Spring generates the implementation automatically.

Developer writes only the interface.

---

# Internal Flow

```
Database

↓

ResultSet

↓

Spring Proxy

↓

UserProjection

↓

Application
```

Notice

There is no implementation class.

Spring creates one dynamically.

---

# Why Interface Projection?

Advantages

- Less Boilerplate
- Faster Development
- Very Convenient
- Good for Small Read Models

---

# DTO vs Interface Projection

| DTO | Interface Projection |
|------|----------------------|
| Constructor Based | Proxy Based |
| Immutable | Dynamic Proxy |
| Better Performance | Less Boilerplate |
| Explicit API Contract | Convenient |
| Best for Public APIs | Best for Internal Queries |

---

# Which should we prefer?

Small Internal Project

```
Interface Projection
```

Enterprise REST API

```
DTO Projection
```

Reason

DTOs provide

- Stable API Contracts
- Better Serialization
- Better Performance
- Clear Ownership

---

# Why not return Entities from REST APIs?

Suppose Entity contains

```
Password

Salary

Internal Flags
```

Returning Entity

↓

Sensitive Data Exposure

Also

Lazy Loading

↓

Serialization Problems

Entities belong to

```
Persistence Layer
```

DTOs belong to

```
API Layer
```

Different responsibilities.

---

# KRB Academy Summary

DTO Projection is the preferred approach for read-only APIs because it loads only the required columns while avoiding Persistence Context, Dirty Checking, and unnecessary object creation.

Interface Projection provides a lightweight alternative where Spring generates runtime proxies automatically. DTO Projection is generally preferred for enterprise APIs because it creates explicit, stable contracts between the backend and its consumers.

---

# Criteria API

---

# Why Criteria API?

Suppose we have a search screen.

Filters

```
Name

Department

Salary

Experience

City
```

The user may select

```
Only Name
```

or

```
Name + Department
```

or

```
Salary + Experience
```

or

```
All Filters
```

Question

Should we create

```java
findByName()

findByDepartment()

findBySalary()

findByNameAndDepartment()

findByDepartmentAndSalary()

findBySalaryAndExperience()

...
```

No.

As the number of filters grows, the number of repository methods grows exponentially.

This becomes impossible to maintain.

---

# What is Criteria API?

Criteria API allows queries to be built dynamically at runtime.

Instead of writing SQL strings,

we construct a query object.

Conceptually

```
Query

↓

Add Predicate

↓

Add Predicate

↓

Add Predicate

↓

Generate SQL
```

Instead of modifying SQL,

we modify objects.

---

# Criteria API Flow

```
Application

↓

CriteriaBuilder

↓

CriteriaQuery

↓

Root

↓

Predicate

↓

Hibernate

↓

SQL

↓

Database
```

Everything is represented as Java objects.

---

# CriteriaBuilder

CriteriaBuilder is the factory responsible for creating query components.

It creates

- Predicates
- Expressions
- Conditions
- Sorting
- Aggregations

Think of it as

```
Object Factory

↓

Creates Query Objects
```

It does not execute queries.

---

# CriteriaQuery

CriteriaQuery represents the query being built.

Conceptually

```
SELECT

↓

FROM

↓

WHERE

↓

ORDER BY

↓

GROUP BY
```

Every clause is represented as an object.

---

# Root

Root represents the entity being queried.

Example

```java
Root<User> root
```

Conceptually

```
User Table
```

Every field reference starts from Root.

Example

```java
root.get("name")
```

means

```
User.name
```

---

# Predicate

Predicate represents a single condition.

Example

```java
cb.equal(root.get("department"),"IT")
```

This creates

```
Department = IT
```

Another

```java
cb.greaterThan(root.get("salary"),50000)
```

creates

```
Salary > 50000
```

Each Predicate is independent.

---

# Why Predicate?

Instead of concatenating SQL

```
WHERE

+

AND

+

OR

+

AND
```

Hibernate creates objects.

Conceptually

```
Predicate

↓

Predicate

↓

Predicate

↓

Combine
```

Objects are much easier to manipulate than SQL strings.

---

# Dynamic Query Building

Suppose

```
Name

Optional

Department

Optional

Salary

Optional
```

Application checks

```
Provided?

↓

Yes

↓

Create Predicate

↓

No

↓

Skip
```

Finally

Hibernate combines all available predicates.

This gives us complete flexibility.

---

# Internal Flow

```
Request

↓

Read Filters

↓

Create Predicates

↓

CriteriaQuery

↓

Hibernate

↓

Generate SQL

↓

Database
```

Notice

SQL is generated only after the object model is complete.

---

# Why is this better than String Concatenation?

Suppose

```
WHERE

name='Kunal'

AND

salary>50000
```

Using strings

```
Forgot AND

↓

Invalid SQL
```

Forgot spaces

↓

Invalid SQL

Misspelled column

↓

Runtime Failure

Object based queries avoid these problems.

---

# Specifications

Criteria API is powerful.

But applications often repeat the same predicates.

Example

```
Department = IT

Salary > 50000

Experience > 5
```

Multiple APIs may reuse them.

Instead of rewriting them,

Spring introduced Specifications.

---

# What is Specification?

A Specification represents

```
One Business Rule
```

Example

```
Department Specification

Salary Specification

Experience Specification
```

Each Specification creates one Predicate.

---

# Combining Specifications

Specifications can be combined.

```
Department

AND

Salary

AND

Experience
```

or

```
Department

OR

Experience
```

Spring builds one Criteria Query automatically.

---

# Internal Flow

```
Specification

↓

Predicate

↓

CriteriaQuery

↓

Hibernate

↓

SQL

↓

Database
```

Specification is simply a reusable Predicate builder.

---

# Why Specifications?

Suppose

10 APIs

Need

```
Department = IT
```

Without Specification

Copy

Copy

Copy

Copy

With Specification

```
DepartmentSpecification

↓

Reuse Everywhere
```

Code duplication disappears.

---

# Criteria API vs Specification

| Criteria API | Specification |
|--------------|---------------|
| Low Level API | Built on Criteria API |
| More Flexible | More Reusable |
| Verbose | Cleaner |
| Manual Query Construction | Business Rule Composition |

---

# KRB Academy Summary

Criteria API represents SQL queries as Java object graphs rather than strings. Hibernate generates SQL only after the query model has been fully constructed.

Specifications build on top of Criteria API by encapsulating reusable business predicates. They allow applications to compose complex search queries dynamically while avoiding code duplication and keeping repositories maintainable.

---

# Pagination

---

# Why Pagination?

Suppose the Users table contains

```
10,000,000 Rows
```

A client calls

```http
GET /users
```

Question

Should Hibernate load

```
10 Million Users
```

into memory?

No.

Problems

- Huge Heap Usage
- Long Response Time
- High Network Traffic
- Increased Garbage Collection
- Application May Crash

Instead, we return only a small subset of records.

This is called

```
Pagination
```

---

# What is Pagination?

Pagination divides a large dataset into smaller pages.

Example

```
Page Size = 20
```

Database returns

```
Rows 1-20

↓

Rows 21-40

↓

Rows 41-60
```

instead of every row.

---

# Internal Flow

```
Client

↓

Page Number

↓

Repository

↓

Hibernate

↓

LIMIT / OFFSET

↓

Database

↓

20 Rows

↓

Application
```

The database performs the pagination.

The application only receives the required records.

---

# Offset Pagination

Offset Pagination skips records.

SQL

```sql
SELECT *
FROM users
ORDER BY id
LIMIT 20 OFFSET 40;
```

Meaning

```
Skip

40 Rows

↓

Return

Next 20
```

---

# Why does OFFSET become slower?

Suppose

```
Page = 5000

Page Size = 20
```

Hibernate generates

```sql
LIMIT 20 OFFSET 99980
```

Conceptually

```
Read

↓

Row 1

↓

Row 2

↓

...

↓

Row 99,980

↓

Discard

↓

Return

20 Rows
```

The client still receives

```
20 Rows
```

But the database processed

```
100,000 Rows
```

Most of the work was wasted.

---

# Keyset Pagination

Instead of

```sql
OFFSET 100000
```

Use

```sql
WHERE id > 100000
LIMIT 20
```

Conceptually

```
Locate ID

100000

↓

Read

100001

↓

100002

↓

100003
```

No skipping.

No discarding.

Only useful work.

---

# Offset vs Keyset

| Offset | Keyset |
|---------|---------|
| Uses OFFSET | Uses WHERE |
| Slower for large pages | Nearly Constant Performance |
| Supports Random Pages | Next / Previous Navigation |
| Reads Discarded Rows | Reads Only Required Rows |

---

# Sorting

---

# Why Sorting?

Suppose we execute

```sql
SELECT *
FROM users;
```

Question

Will the database always return rows in the same order?

No.

SQL does not guarantee ordering unless

```sql
ORDER BY
```

is specified.

---

# Internal Flow

```
Query

↓

ORDER BY

↓

Database

↓

Sorted Rows

↓

Hibernate

↓

Application
```

Sorting belongs to the database.

---

# Stable Ordering

Suppose

```
Salary

50000

50000

50000
```

Query

```sql
ORDER BY salary
```

Question

Which employee comes first?

The database does not know.

The order among equal values is undefined.

Instead use

```sql
ORDER BY salary,
id
```

Now every row has a unique position.

---

# Database Sorting

Never load

```
10 Million Rows
```

into Java

and call

```java
Collections.sort()
```

Instead

```sql
ORDER BY
```

lets the database perform the sorting.

The database can

- Use Indexes
- Optimize Execution Plans
- Sort Efficiently

---

# Index Assisted Sorting

Suppose

```sql
ORDER BY salary
```

No index exists.

Database

```
Read

↓

Sort

↓

Return
```

If an index exists

```
Index

↓

Read In Order

↓

Return
```

No runtime sorting required.

---

# Lost Update Problem

Suppose

```
Balance

₹10,000
```

Transaction A

Withdraw

₹2,000

Transaction B

Withdraw

₹3,000

Both read

```
₹10,000
```

A updates

```
₹8,000
```

B updates

```
₹7,000
```

Correct balance should be

```
₹5,000
```

One update was lost.

This is called

```
Lost Update Problem
```

---

# Why Locking Exists?

Locking prevents multiple transactions from corrupting shared data.

Hibernate supports

- Optimistic Locking
- Pessimistic Locking

---

# Optimistic Locking

Optimistic Locking assumes

```
Conflicts are Rare
```

Uses

```java
@Version
private Long version;
```

Initial State

```
Balance

10000

Version

1
```

Transaction A

```
Version = 1

↓

Update

↓

Version = 2
```

Transaction B

Still expects

```
Version = 1
```

Update fails.

Hibernate throws

```
OptimisticLockException
```

---

# Internal Flow

```
Read Entity

↓

Read Version

↓

Modify

↓

UPDATE

WHERE Version = ?

↓

Rows Updated?

↓

Yes

Commit

↓

No

OptimisticLockException
```

---

# Advantages

- No Database Locks
- Better Throughput
- Better Concurrency

Best For

- User Profiles
- Product Catalogs
- HR Systems
- CRM

---

# Pessimistic Locking

Pessimistic Locking assumes

```
Conflicts are Expected
```

Hibernate acquires a database lock.

```java
@Lock(
LockModeType.PESSIMISTIC_WRITE
)
```

Conceptually

```sql
SELECT *
FROM account
FOR UPDATE;
```

Another transaction

↓

Must Wait

Until

↓

Commit

---

# Advantages

- Prevents Lost Updates
- Strong Consistency

Disadvantages

- Blocking
- Deadlocks
- Lower Throughput

Best For

- Banking
- Payments
- Inventory
- Seat Booking

---

# Optimistic vs Pessimistic

| Optimistic | Pessimistic |
|------------|-------------|
| Version Based | Database Lock |
| No Blocking | Blocking |
| Higher Throughput | Higher Consistency |
| Retry Required | Wait Required |

---

# KRB Academy Summary

Pagination prevents applications from loading unnecessary data by delegating page calculation to the database. Offset Pagination is simple but becomes slower as offsets grow, while Keyset Pagination avoids unnecessary work by continuing from the last known key.

Sorting should always be performed by the database using deterministic ordering. Locking protects shared data from concurrent modifications. Optimistic Locking detects conflicts using a version column, whereas Pessimistic Locking prevents conflicts through database row locks.

---

# Batch Fetching

---

# Why Batch Fetching?

Suppose we have the following relationship

```java
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;
```

Now we load

```
100 Users
```

Initially

```
Users

↓

Loaded
```

Orders are not loaded because they are lazy.

Later

```java
user.getOrders();
```

Hibernate executes

```
SELECT Orders

WHERE user_id = 1

↓

SELECT Orders

WHERE user_id = 2

↓

SELECT Orders

WHERE user_id = 3

↓

...

↓

100 Queries
```

Again

```
N+1 Problem
```

---

# Why not make it EAGER?

Many developers think

```
Problem

↓

Use EAGER
```

Wrong.

EAGER loads data even when it is never used.

Suppose

90% of APIs

Never access

```
Orders
```

Still Hibernate loads them.

Result

- Higher Memory
- Slower Queries
- Bigger Result Sets

The better solution is

```
Keep LAZY

+

Optimize Lazy Loading
```

This is exactly what Batch Fetching does.

---

# What is Batch Fetching?

Instead of loading

```
Orders

↓

One User At A Time
```

Hibernate groups multiple lazy loads together.

Example

```
Batch Size = 20
```

Instead of

```
20 SQL Queries
```

Hibernate executes

```sql
SELECT *
FROM orders
WHERE user_id IN (1,2,3,4,...20)
```

One SQL

Twenty Users

---

# Internal Flow

Without Batch Fetching

```
Users

↓

Access Orders

↓

100 SQL Queries
```

With Batch Fetching

```
Users

↓

Collect IDs

↓

WHERE id IN(...)

↓

5 SQL Queries

Instead of

100
```

---

# Configuring Batch Fetching

Entity Level

```java
@OneToMany(fetch = FetchType.LAZY)

@BatchSize(size = 20)

private List<Order> orders;
```

Global Configuration

```properties
hibernate.default_batch_fetch_size=20
```

Hibernate automatically batches lazy loading.

---

# Why does this improve performance?

Database Round Trips

Without

```
100 Network Calls
```

With Batch Fetching

```
5 Network Calls
```

Network latency is often much slower than executing SQL itself.

Reducing round trips improves performance significantly.

---

# JDBC Batching

---

# Why JDBC Batching?

Suppose

```java
for(User user : users){

    repository.save(user);

}
```

1000 Users

Hibernate executes

```
INSERT

↓

INSERT

↓

INSERT

↓

...

↓

1000 Times
```

Every INSERT

↓

One Database Call

Very expensive.

---

# What is JDBC Batching?

Hibernate groups multiple write operations together.

Instead of

```
INSERT

↓

INSERT

↓

INSERT
```

Hibernate sends

```
50 INSERTS

↓

One Network Call
```

---

# Configuration

```properties
hibernate.jdbc.batch_size=50
```

Now

```
1000 Inserts

↓

20 Batches

↓

20 Database Calls
```

instead of

```
1000 Calls
```

---

# Internal Flow

Without Batching

```
Application

↓

INSERT

↓

Database

↓

INSERT

↓

Database
```

With JDBC Batching

```
Application

↓

Batch Buffer

↓

50 INSERTS

↓

Database
```

---

# Batch Fetching vs JDBC Batching

| Batch Fetching | JDBC Batching |
|---------------|---------------|
| Read Optimization | Write Optimization |
| SELECT | INSERT / UPDATE / DELETE |
| Reduces N+1 | Reduces Network Calls |
| Lazy Loading | Bulk Writes |

---

# Second-Level Cache

---

# Why First-Level Cache isn't enough?

Persistence Context

↓

Transaction Ends

↓

Cache Destroyed

Next Request

↓

Database Again

This means

Every transaction starts with an empty cache.

---

# What is Second-Level Cache?

Second-Level Cache is shared across

```
All EntityManagers
```

Scope

```
SessionFactory
```

Architecture

```
SessionFactory

↓

Second-Level Cache

↓

EntityManager 1

EntityManager 2

EntityManager 3
```

Every transaction can reuse cached entities.

---

# Internal Flow

```
Find User

↓

Second-Level Cache

↓

Found?

↓

Yes

↓

Return Entity

↓

No

↓

Database

↓

Store In Cache

↓

Return Entity
```

---

# Good Candidates

- Countries
- States
- Roles
- Product Categories
- Configuration
- Permissions

---

# Bad Candidates

- Inventory
- Wallet Balance
- Stock Prices
- Payment Status
- Shopping Cart

Frequently changing data creates cache invalidation overhead.

---

# First-Level vs Second-Level Cache

| First-Level | Second-Level |
|-------------|--------------|
| EntityManager Scope | SessionFactory Scope |
| Private | Shared |
| Default | Optional |
| Transaction Lifetime | Application Lifetime |

---

# Query Cache

---

# Why Query Cache?

Suppose

```java
findByDepartment("Engineering")
```

returns

```
User 1

User 5

User 9
```

Question

Does Second-Level Cache store this query?

No.

Second-Level Cache stores

```
Entities
```

Not

```
Query Results
```

Hibernate still executes the SQL to determine

Which Users belong to Engineering.

---

# What does Query Cache store?

It stores

```
Query

↓

Matching IDs

↓

[1,5,9]
```

Not

```
Entire User Objects
```

---

# Internal Flow

```
Execute Query

↓

Query Cache

↓

IDs

↓

Second-Level Cache

↓

Entities

↓

Application
```

Both caches work together.

---

# Why Query Cache depends on Second-Level Cache?

Suppose Query Cache returns

```
1

5

9
```

Hibernate still needs

```
User 1

User 5

User 9
```

Without Second-Level Cache

Hibernate loads every entity from the database.

Therefore

Query Cache alone provides little benefit.

---

# Second-Level Cache vs Query Cache

| Second-Level Cache | Query Cache |
|--------------------|-------------|
| Stores Entities | Stores Query Results |
| Entity Based | Query Based |
| Can Work Alone | Best with Second-Level Cache |

---

# When should Query Cache be used?

Good

- Countries
- Roles
- Configuration
- Reports
- Read Mostly Data

Avoid

- Orders
- Inventory
- Payments
- Live Dashboards

---

# KRB Academy Summary

Batch Fetching reduces database round trips while preserving Lazy Loading. Instead of loading associations one entity at a time, Hibernate groups multiple lazy loads into fewer SQL statements.

JDBC Batching optimizes write operations by grouping INSERT, UPDATE and DELETE statements into batches, significantly reducing network overhead.

Second-Level Cache shares entities across transactions, while Query Cache stores query result identifiers. Query Cache complements, but does not replace, the Second-Level Cache.

---

# Repository Customization

---

# Why Repository Customization?

Spring Data JPA provides

```java
JpaRepository
```

with common CRUD operations.

Example

```java
save()

findById()

findAll()

delete()
```

Question

What if our application requires

- Complex JPQL
- Criteria API
- Stored Procedures
- Bulk Updates
- Vendor Specific SQL
- Custom Business Queries

Should we put everything inside the Service?

No.

Repositories are responsible for

```
Persistence Logic
```

Services are responsible for

```
Business Logic
```

Responsibilities should remain separate.

---

# Custom Repository

Spring allows creating a custom repository.

Architecture

```
Service

↓

UserRepository

↓

JpaRepository

+

UserRepositoryCustom

↓

EntityManager

↓

Hibernate

↓

Database
```

The default CRUD methods remain available while complex persistence logic is moved into the custom repository.

---

# Why EntityManager?

Question

Inside a custom repository should we use

```
DriverManager

↓

JDBC
```

or

```
EntityManager
```

Always use

```
EntityManager
```

EntityManager already manages

- Transactions
- Persistence Context
- Dirty Checking
- Flush
- JPQL
- Criteria API
- Native Queries

Using raw JDBC bypasses all JPA features.

---

# Internal Flow

```
Application

↓

Service

↓

Custom Repository

↓

EntityManager

↓

Hibernate

↓

JDBC

↓

Database
```

EntityManager is the entry point to JPA.

---

# Extending SimpleJpaRepository

Every JpaRepository implementation is backed by

```
SimpleJpaRepository
```

Architecture

```
JpaRepository

↓

Repository Proxy

↓

SimpleJpaRepository

↓

EntityManager
```

Question

Suppose every repository should

- Audit saves
- Apply Soft Delete
- Encrypt fields
- Add Logging

Should we copy code into every repository?

No.

Instead extend

```
SimpleJpaRepository
```

Once.

Every repository inherits the behavior.

---

# Why Interfaces?

Question

Why do we write

```java
interface UserRepository
extends JpaRepository<User,Long>
```

instead of

```java
new SimpleJpaRepository()
```

Spring follows

```
Program to an Interface

Not an Implementation
```

Benefits

- Loose Coupling
- Proxy Generation
- Easy Testing
- Runtime Customization
- Better Extensibility

---

# Repository Delegation

```
Application

↓

Service

↓

JpaRepository

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

Every layer has

```
One Responsibility
```

No layer performs the work of another.

---

# Production Best Practices

---

# Use DTOs for REST APIs

Instead of

```
Entity

↓

JSON
```

Prefer

```
DTO

↓

JSON
```

Benefits

- Better Security
- Smaller Responses
- Stable API Contracts
- Better Performance

---

# Prefer LAZY Loading

Never make every relationship

```
EAGER
```

Lazy loading

↓

Load Only When Needed

Combine with

```
Batch Fetching
```

for better performance.

---

# Use JOIN FETCH Carefully

Good

```
User

+

Department

Always Needed
```

Bad

```
Multiple Collections

Large Result Sets

Pagination
```

JOIN FETCH is a tool.

Not a default strategy.

---

# Prefer Specifications

Suppose

15 Optional Filters

Without Specifications

```
Hundreds

of Repository Methods
```

With Specifications

```
One Specification

↓

Reusable

↓

Composable
```

Better architecture.

---

# Push Work to the Database

Database should perform

- Filtering
- Sorting
- Pagination
- Aggregations
- Joins

Application should perform

- Business Logic

---

# Cache Carefully

Cache

- Countries
- Roles
- States
- Configuration

Avoid Caching

- Orders
- Inventory
- Wallet Balance
- Payments

---

# Choose Locking by Contention

Low Conflict

↓

Optimistic Locking

High Conflict

↓

Pessimistic Locking

Never choose a locking strategy without understanding application behavior.

---

# Complete Hibernate Philosophy

Everything we learned in Module 17 follows one philosophy.

## 1. Load only what you need.

Use

- DTO Projection
- Interface Projection
- Lazy Loading

---

## 2. Load it only when needed.

Use

- Lazy Associations
- Batch Fetching
- JOIN FETCH only when appropriate

---

## 3. Let the database perform database work.

Database

↓

Filtering

Sorting

Pagination

Joins

Aggregations

Application

↓

Business Logic

---

## 4. Reduce database round trips.

Use

- Batch Fetching
- JDBC Batching
- Second-Level Cache
- Query Cache

---

## 5. Reuse Components.

Use

- Specifications
- Custom Repositories
- SimpleJpaRepository
- Query Plan Cache

---

## 6. Program Against Abstractions.

JpaRepository

↓

EntityManager

↓

Hibernate

↓

JDBC

↓

Database

Every layer delegates.

---

# KRB Academy Summary

Module 17 explored the advanced internals of Spring Data JPA and Hibernate.

Rather than memorizing annotations, we derived why each feature exists and what engineering problem it solves.

We learned that Hibernate is designed around delegation, abstraction, object modeling and performance optimization.

Every feature—from JPQL to Specifications, Pagination, Locking, Batching and Caching—exists to solve a specific scalability or maintainability problem.

Understanding these trade-offs is what differentiates a developer who can use JPA from an engineer who can design high-performance persistence layers.

---

# Engineering Principles and Interview Readiness

## Engineering Principles Learned

### Principle 1
High-level object-oriented queries should remain independent of database implementation details.

### Principle 2
Separate business queries from database-specific SQL syntax through the database dialect.

### Principle 3
Parse once and reuse the query plan instead of reparsing the same JPQL repeatedly.

### Principle 4
Prefer JPQL over native SQL whenever possible to keep the application database independent.

### Principle 5
JOIN filters data, while JOIN FETCH loads associated data.

### Principle 6
Performance problems are often caused by excessive database round trips rather than slow SQL execution.

### Principle 7
Choose fetch strategies based on the business use case, not a one-size-fits-all default.

### Principle 8
Reducing SQL count should never come at the cost of creating excessively large result sets.

### Principle 9
Applications should retrieve only the data required by the current use case.

### Principle 10
Persistence models and API models should remain independent.

### Principle 11
Represent queries as objects rather than strings.

### Principle 12
Business rules should be reusable instead of duplicated across repositories.

### Principle 13
Avoid processing data that will ultimately be discarded.

### Principle 14
Stable pagination requires deterministic ordering.

### Principle 15
Delegate sorting and pagination to the database.

### Principle 16
Detect conflicts when they are rare and prevent them when conflicts are expected.

### Principle 17
Optimize communication between the application and the database by reducing unnecessary round trips.

### Principle 18
Cache stable data and avoid caching highly volatile data.

### Principle 19
Different caches solve different problems: entity cache and query cache serve different purposes.

### Principle 20
Repositories encapsulate persistence logic while services encapsulate business logic.

### Principle 21
Use EntityManager instead of bypassing JPA with raw JDBC.

### Principle 22
Program against abstractions rather than implementations.

### Principle 23
Centralize cross-cutting repository behavior by extending SimpleJpaRepository instead of duplicating code.

### Principle 24
Load only the data required by the current use case.

### Principle 25
Delegate filtering, sorting, and pagination to the database.

### Principle 26
Choose the simplest solution that satisfies the performance requirements.

### Principle 27
Every optimization introduces trade-offs that must be understood.

## Interview Questions

- What is JPQL and why does Hibernate parse it?
- What is Query Plan Cache?
- What is the difference between JPQL and Native SQL?
- What is the difference between JOIN and JOIN FETCH?
- What is the N+1 query problem and how is it avoided?
- What is a Cartesian Product?
- What is DTO Projection and why is it useful?
- What is Interface Projection?
- Why does Criteria API exist and how do Specifications help?
- What is the difference between Offset and Keyset Pagination?
- What is the Lost Update Problem?
- What is Optimistic Locking versus Pessimistic Locking?
- What is Batch Fetching versus JDBC Batching?
- What is the difference between First-Level, Second-Level, and Query Cache?
- Why is Repository Customization important in Spring Data JPA?

## Senior Interview Tips

- Always explain why a feature exists before explaining how it works.
- Focus on trade-offs rather than memorizing annotations.
- Mention the performance implications of every design decision.
- Relate Hibernate internals to Spring's delegation architecture.
- Explain how each optimization reduces memory use, database round trips, or CPU overhead.

## KRB Academy Interview Readiness

After completing this module, you should confidently explain:

- JPQL internals and SQL generation
- Query parsing and Query Plan Cache
- JOIN versus JOIN FETCH
- DTO and Interface Projection
- Criteria API and Specifications
- Pagination, sorting, and locking
- Batch Fetching, JDBC Batching, and caching
- Repository customization and production best practices

---

# Module 17 Complete

## Key Takeaway

> **Load only what you need, load it only when required, let the database do database work, minimize round trips, reuse components, and depend on abstractions rather than implementations.**

This single philosophy connects every advanced Hibernate feature and forms the foundation of scalable, maintainable, and high-performance enterprise applications.

---