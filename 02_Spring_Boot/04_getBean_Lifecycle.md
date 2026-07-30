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

Developer


Ã¢â€ â€œ


ApplicationContext


Ã¢â€ â€œ


BeanFactory


Ã¢â€ â€œ


Bean Instance


The developer interacts only with the ApplicationContext.

The ApplicationContext delegates all bean management responsibilities to the BeanFactory.


---
## BeanFactory Architecture

Spring builds its BeanFactory using layered abstractions.

BeanFactory (Interface)

       Ã¢â€â€š

       Ã¢â€“Â¼

AbstractBeanFactory

       Ã¢â€â€š

       Ã¢â€“Â¼

AbstractAutowireCapableBeanFactory

       Ã¢â€â€š

       Ã¢â€“Â¼

DefaultListableBeanFactory


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

               BeanFactory

                    Ã¢â€â€š

     Ã¢â€Å’Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€Â¼Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€Â

     Ã¢â€“Â¼                     Ã¢â€“Â¼                    Ã¢â€“Â¼

BeanDefinition   Singleton Cache   Dependency Resolver

  Registry


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

     Ã¢â€ â€œ

UserService Instance


emailService

     Ã¢â€ â€œ

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


       Ã¢â€â€š

       Ã¢â€“Â¼

ApplicationContext


       Ã¢â€â€š

       Ã¢â€“Â¼

AbstractBeanFactory.getBean()


       Ã¢â€â€š

       Ã¢â€“Â¼

Check Singleton Cache


       Ã¢â€â€š

Found?

Ã¢â€â€š

Ã¢â€Å“Ã¢â€â‚¬Ã¢â€â‚¬ Yes

Ã¢â€â€š      Ã¢â€â€š

Ã¢â€â€š      Ã¢â€“Â¼

Ã¢â€â€š   Return Existing Bean

Ã¢â€â€š

Ã¢â€â€Ã¢â€â‚¬Ã¢â€â‚¬ No


       Ã¢â€â€š

       Ã¢â€“Â¼

Read BeanDefinition


       Ã¢â€â€š

       Ã¢â€“Â¼

AbstractAutowireCapableBeanFactory.createBean()


       Ã¢â€â€š

       Ã¢â€“Â¼

Choose Constructor


       Ã¢â€â€š

       Ã¢â€“Â¼

Resolve Constructor Dependencies


       Ã¢â€â€š

       Ã¢â€“Â¼

Instantiate Bean


       Ã¢â€â€š

       Ã¢â€“Â¼

Populate Properties


       Ã¢â€â€š

       Ã¢â€“Â¼

Initialize Bean


       Ã¢â€â€š

       Ã¢â€“Â¼

Return Bean


       Ã¢â€â€š

       Ã¢â€“Â¼

AbstractBeanFactory


       Ã¢â€â€š

       Ã¢â€“Â¼

Store Singleton (if singleton scope)


       Ã¢â€â€š

       Ã¢â€“Â¼

Return Bean to ApplicationContext


       Ã¢â€â€š

       Ã¢â€“Â¼

Return Bean to Developer


---
## Recursive Dependency Resolution

Eg:

public UserService(

       UserRepository repository,

       NotificationService notificationService

)


Spring Performs:


getBean(UserService)

Ã¢â€ â€œ

Need UserRepository

Ã¢â€ â€œ

getBean(UserRepository)

Ã¢â€ â€œ

Create UserRepository

Ã¢â€ â€œ

Return

Ã¢â€ â€œ

Need NotificationService

Ã¢â€ â€œ

Find Candidates

Ã¢â€ â€œ

Apply @Qualifier / @Primary

Ã¢â€ â€œ

Create EmailService

Ã¢â€ â€œ

Return

Ã¢â€ â€œ

Create UserService

Ã¢â€ â€œ

Return


Spring recursively calls getBean() until every dependency is satisfied.


---
## Bean Creation Phases

Bean creation is not a single step.


It consists of multiple phases:

Read BeanDefinition

Ã¢â€ â€œ

Choose Constructor

Ã¢â€ â€œ

Resolve Dependencies

Ã¢â€ â€œ

Instantiate Object

Ã¢â€ â€œ

Inject Dependencies

Ã¢â€ â€œ

Populate Properties

Ã¢â€ â€œ

Initialize Bean

Ã¢â€ â€œ

Return Bean


---
## Mental Model

Developer

Ã¢â€ â€œ

ApplicationContext

Ã¢â€ â€œ

AbstractBeanFactory

Ã¢â€ â€œ

Singleton Cache

Ã¢â€ â€œ

BeanDefinition Registry

Ã¢â€ â€œ

AbstractAutowireCapableBeanFactory

Ã¢â€ â€œ

Reflection

Ã¢â€ â€œ

Dependency Injection

Ã¢â€ â€œ

Bean Initialization

Ã¢â€ â€œ

Singleton Cache

Ã¢â€ â€œ

Developer receives Bean
