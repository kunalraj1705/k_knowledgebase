# The Complete getBean() Lifecycle

## Learning Objective

Understand the complete execution flow of Spring's getBean() method, from the moment a developer requests a bean until a fully initialized object is returned.

## Why is getBean() Important?

Every bean managed by Spring is ultimately created through the getBean() workflow.

Whether Spring creates beans during startup or lazily at runtime, the underlying algorithm remains the same.

Understanding this flow is the foundation for understanding:

* Dependency Injection
* Bean Lifecycle
* Singleton Management
* Circular Dependencies
* Spring Internals

## High-Level Architecture

```text
Developer
  ↓
ApplicationContext
  ↓
BeanFactory
  ↓
Bean Instance
```

The developer interacts only with the ApplicationContext.

The ApplicationContext delegates all bean management responsibilities to the BeanFactory.

---

## BeanFactory Architecture

Spring builds its `BeanFactory` using layered abstractions.

```text
BeanFactory (Interface)
  │
  ▼
AbstractBeanFactory
  │
  ▼
AbstractAutowireCapableBeanFactory
  │
  ▼
DefaultListableBeanFactory
```

Each layer adds one responsibility while reusing the previous implementation.

---

## Responsibilities

BeanFactory (Interface)

* Defines the contract for an IoC container.
* Examples:

  * getBean()
  * containsBean()
  * isSingleton()

   It specifies what should happen.

* AbstractBeanFactory

  * Coordinates bean retrieval.
  * Responsibilities:

    * Receives getBean()
    * Checks Singleton Cache
    * Reads BeanDefinition
    * Delegates bean creation
    * Stores singleton beans

    It does not create objects directly.

* AbstractAutowireCapableBeanFactory

  * Responsible for bean creation.
  * Responsibilities:

    * Select constructor
    * Resolve dependencies
    * Instantiate bean using Reflection
    * Inject dependencies
    * Populate properties
    * Initialize bean
    * Apply BeanPostProcessors
* DefaultListableBeanFactory

  * Concrete implementation used by Spring.
  * Responsibilities:

    * BeanDefinition registration
    * Bean lookup
    * Candidate resolution
    * Type matching
    * @Primary
    * @Qualifier

---

## Internal Components of BeanFactory

```text
BeanFactory
  │
  ├── BeanDefinition Registry
  ├── Singleton Cache
  └── Dependency Resolver
```

---

## BeanDefinition Registry

Stores metadata required to create beans.

Contains:

* Bean Class
* Scope
* Constructor Metadata
* Dependencies
* Lazy
* Primary
* Qualifier

Think of it as a Blueprint Library.

---

## Singleton Cache

Stores already-created singleton bean instances.

userService

      ↓

UserService Instance

emailService

      ↓

EmailService Instance

This is the first place Spring checks during getBean().

---

## Dependency Resolver

Responsible for:

Constructor selection

Dependency discovery

@Qualifier

@Primary

Recursive dependency resolution

Circular dependency detection

---

## Complete getBean() Flow

context.getBean(UserService)

        │

        ▼

ApplicationContext

        │

        ▼

AbstractBeanFactory.getBean()

        │

        ▼

Check Singleton Cache

        │

Found?

│

├── Yes

│      │

│      ▼

│   Return Existing Bean

│

└── No

        │

        ▼

Read BeanDefinition

        │

        ▼

AbstractAutowireCapableBeanFactory.createBean()

        │

        ▼

Choose Constructor

        │

        ▼

Resolve Constructor Dependencies

        │

        ▼

Instantiate Bean

        │

        ▼

Populate Properties

        │

        ▼

Initialize Bean

        │

        ▼

Return Bean

        │

        ▼

AbstractBeanFactory

        │

        ▼

Store Singleton (if singleton scope)

        │

        ▼

Return Bean to ApplicationContext

        │

        ▼

Return Bean to Developer

---

## Recursive Dependency Resolution

Eg:

```java
public UserService(

        UserRepository repository,

        NotificationService notificationService

)

Spring Performs:

getBean(UserService)

↓

Need UserRepository

↓

getBean(UserRepository)

↓

Create UserRepository

↓

Return

↓

Need NotificationService

↓

Find Candidates

↓

Apply @Qualifier / @Primary

↓

Create EmailService

↓

Return

↓

Create UserService

↓

Return

Spring recursively calls getBean() until every dependency is satisfied.

---

## Bean Creation Phases

Bean creation is not a single step.

It consists of multiple phases:

Read BeanDefinition

↓

Choose Constructor

↓

Resolve Dependencies

↓

Instantiate Object

↓

Inject Dependencies

↓

Populate Properties

↓

Initialize Bean

↓

Return Bean

---

## Mental Model

Developer

↓

ApplicationContext

↓

AbstractBeanFactory

↓

Singleton Cache

↓

BeanDefinition Registry

↓

AbstractAutowireCapableBeanFactory

↓

Reflection

↓

Dependency Injection

↓

Bean Initialization

↓

Singleton Cache

↓

Developer receives Bean
```
