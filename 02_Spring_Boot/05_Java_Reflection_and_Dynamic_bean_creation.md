# Java Reflection and Dynamic Bean Creation

## Learning Objective

Understand how Spring dynamically creates bean instances using Java Reflection, distinguish the responsibilities of the JVM, `ClassLoader`, `Reflection`, and `BeanFactory`, and learn how Reflection and dependency resolution work together during bean creation.

---

## Problem Statement

Spring is a generic framework. It cannot know application-specific classes like UserService, OrderService, or PaymentService while the framework itself is being developed.

If Spring had to create beans without Reflection, it would require hardcoded object creation for every application class, making the framework impossible to generalize.

The challenge is:

    How can Spring create objects whose classes are unknown at compile time?

Java Reflection solves this problem.

---

## Architecture Components

```text
Java Compilation & Runtime Flow

UserService.java
  ↓
javac Compiler
  ↓
UserService.class
  ↓
ClassLoader
  ↓
JVM creates Class<UserService>
  ↓
Spring Component Scan
  ↓
BeanDefinition
  ↓
BeanFactory
  ↓
Reflection
  ↓
Bean Instance
  ↓
Initialization
  ↓
Singleton Cache
```

---

## Core Components

* Java Compiler

  * Converts source code into JVM bytecode.

* ClassLoader

  * Responsibilities:

    * Loads .class files into the JVM.
    * During class loading, the JVM creates the Class<?> object.

* Class Object (Class<?>)

  * Represents a loaded Java class at runtime.
  * Contains metadata such as:

    * Constructors
    * Fields
    * Methods
    * Interfaces
    * Annotations
    * Class name
  * Spring stores the Class<?> reference inside the BeanDefinition.

* BeanFactory

  * Responsible for all bean creation decisions.
  * Responsibilities include:

    * Selecting constructors
    * Resolving dependencies
    * Managing bean lifecycle
    * Choosing implementations
    * Managing singleton cache

* Reflection

  * Java runtime mechanism used to:

    * Inspect classes
    * Discover constructors
    * Discover methods
    * Discover fields
    * Invoke constructors
    * Create objects dynamically
  * Reflection performs execution.
  * It does not make bean creation decisions.

---

## Internal Working

Step 1 — Class Loading

UserService.class

↓

ClassLoader

↓

JVM creates

Class<UserService>

At this stage, only the class metadata exists.

No bean has been created yet.

Step 2 — Spring Creates BeanDefinition

Spring scans application classes.

It reads annotations like:

* Component
* Service
* @Repository

Using the loaded Class<?> object, Spring creates a BeanDefinition containing all metadata required for bean creation.

Step 3 — Bean Requested

Developer writes:

context.getBean(UserService.class);

BeanFactory starts the bean creation process.

Step 4 — Constructor Selection

BeanFactory decides which constructor should be used.

Rules:

* Single constructor → selected automatically.
* Multiple constructors → @Autowired constructor preferred.
* Otherwise → constructor resolution algorithm is applied.

Reflection never makes this decision.

Step 5 — Dependency Resolution

Suppose:

```java
public UserService(

    UserRepository repository,

    EmailService emailService

)

BeanFactory recursively resolves dependencies.

Need UserService

↓

Need UserRepository

↓

getBean(UserRepository)

↓

Need EmailService

↓

getBean(EmailService)

This process continues until the complete dependency graph is resolved.

Step 6 — Reflection Creates Object

Once all constructor arguments are available:

Reflection invokes the selected constructor.

constructor.newInstance(

    repository,

    emailService

)

Bean instance is created.

Step 7 — Spring Continues Bean Lifecycle

After Reflection creates the object:

* Dependency Injection
* Initialization
* BeanPostProcessor
* Lifecycle callbacks
* Singleton caching

are performed by Spring.

---

## Mental Model

Java Responsibilities

Compiler

↓

Bytecode

↓

ClassLoader

↓

Class Object

Java is responsible for loading classes.

Spring Responsibilities

Component Scan

↓

BeanDefinition

↓

BeanFactory

↓

Reflection

↓

Bean Instance

↓

Initialization

Spring is responsible for managing beans.

---

## Reflection vs BeanFactory

BeanFactory (Brain)

Responsible for deciding:

* Which bean
* Which constructor
* Which dependency
* Which implementation
* Bean lifecycle
* Singleton management

Reflection (Hands)

Responsible for executing:

* Constructor invocation
* Object creation
* Field injection
* Method invocation

Reflection executes the BeanFactory's decisions.

---

Interview Explanation

Spring uses Reflection because it cannot know application-specific classes while the framework is being built. During compilation, Java source code is converted into bytecode. The ClassLoader loads this bytecode into the JVM, creating a Class<?> object. Spring scans these classes, reads annotations, and creates BeanDefinitions. When a bean is requested, the BeanFactory selects the appropriate constructor, recursively resolves all dependencies, and finally uses Reflection to invoke the constructor and create the bean instance. Reflection performs object creation, while the BeanFactory makes all bean creation decisions. After creation, Spring completes initialization and lifecycle processing before returning the fully initialized bean.
```
