\# BeanFactory vs ApplicationContext



\##Learning Objective

* Understand the responsibilities of ApplicationContext and BeanFactory, how Spring locates and creates beans, and how recursive dependency resolution works.



\## Why does Spring have both BeanFactory and ApplicationContext?

* A factory should focus on creating and managing beans.
* A client should not directly interact with the factory.
* Spring introduces ApplicationContext as a higher-level abstraction that hides the internal complexity of the BeanFactory.
* Mental Model

  * &#x20;                Developer
  * &#x20;                    │
  * &#x20;                    ▼
  * &#x20;         ApplicationContext
  * &#x20;       (Public API / Showroom)
  * &#x20;                    │
  * &#x20;                    ▼
  * &#x20;              BeanFactory
  * &#x20;         (Bean Management Unit)
  * &#x20;                    │
  * &#x20;    ┌───────────────┼────────────────┐
  * &#x20;    ▼               	  ▼               	 ▼
  * BeanDefinition   Singleton Cache   Dependency Resolver
  * &#x20;  Registry

Eg:

You want to buy a car, you go to showroom and get the car. You don't know how you car was built.

Here Showroom represent the ApplicationContext, Car factory represent the BeanFactory.



\--------------------------------------------------------------------------------



\## Responsibilities

* ApplicationContext

  * Responsibilities:

    * Public entry point for developers
    * Delegates bean requests to BeanFactory
    * Event publishing
    * Resource loading
    * Environment \& Properties
    * Internationalization (i18n)

&#x20;   ApplicationContext coordinates but does not perform bean creation.

* BeanFactory

  * Responsibilities:
  * Stores Bean Definitions
  * Creates Beans
  * Resolves Dependencies
  * Maintains Singleton Cache
  * Manages Bean Lifecycle

&#x20;  BeanFactory is the core IoC container.

\------------------------------------------------------------------------------

\## Internal Components of BeanFactory

1. BeanDefinition Registry

   * Stores metadata required to create beans.
   * Example metadata:

     * Bean Class	
     * Scope
     * Constructor
     * Dependencies
     * Lazy
     * Primary
     * Qualifier
   * Think of it as a Blueprint Library.
2. Singleton Bean Cache

   * Stores already-created singleton objects.

     * userService
     * &#x20;       ↓
     * UserService Instance
     * 
     * emailService
     * &#x20;       ↓
     * EmailService Instance
   * Every getBean() call first checks this cache.
3. Dependency Resolver

   * Determines:

     * Which constructor to use
     * Constructor arguments
     * Qualifier selection
     * Primary selection
     * Circular dependency detection



\------------------------------------------------------------------------------

\## Bean Retrieval Algorithm

getBean(UserService)



&#x20;       │

&#x20;       ▼

Check Singleton Cache



&#x20;       │

Found?

│

├── Yes → Return Existing Bean

│

└── No



&#x20;       │

&#x20;       ▼

Read BeanDefinition



&#x20;       │

&#x20;       ▼

Choose Constructor



&#x20;       │

&#x20;       ▼

Resolve Constructor Dependencies



&#x20;       │

&#x20;       ▼

Create Bean



&#x20;       │

&#x20;       ▼

Initialize Bean



&#x20;       │

&#x20;       ▼

Singleton?



&#x20;       │

Yes

&#x20;       │

&#x20;       ▼

Store in Singleton Cache



&#x20;       │

&#x20;       ▼

Return Bean



\-------------------------------------------------------------------



\## Recursive Dependency Resolution

Eg:



UserService(

&#x20;   UserRepository repository,

&#x20;   NotificationService notificationService

)



Spring does not create UserService immediately.



Instead:



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



Resolve @Qualifier/@Primary



↓



Create EmailService



↓



Return



↓



Create UserService



↓



Return



NOTE: Spring recursively calls getBean() for dependencies until the dependency graph is fully satisfied.



\-------------------------------------------------------------------------



\## Constructor Selection

1. One Constructor : Spring automatically selects it.

   1. public UserService(UserRepository repo)
2. Multiple Constructors: Spring uses the constructor marked with @Autowired (or applies its constructor resolution rules if no annotation is present).

   1. UserService()
   2. UserService(UserRepository)
   3. UserService(UserRepository, EmailService)



\-------------------------------------------------------------------------

\## Bean Lookup Strategy

Spring instead maintains bean metadata (primarily indexed by bean name) and resolves candidates by type when needed, applying rules like:



Match by type

Apply @Qualifier

Apply @Primary

Throw exception if ambiguous



\------------------------------------------------------------------------

\## Mental Model

Developer



↓



ApplicationContext



↓



BeanFactory



↓



Singleton Cache



↓



BeanDefinition Registry



↓



Dependency Resolution



↓



Bean Creation



↓



Bean Initialization



↓



Singleton Cache



↓



Return Bean



\------------------------------------------------------------------------------------



\## Spring BeanFactory Architecture

Spring does not use a single BeanFactory implementation. Instead, it builds functionality through layered abstractions.

BeanFactory (Interface)

&#x20;       │

&#x20;       ▼

AbstractBeanFactory

&#x20;       │

&#x20;       ▼

AbstractAutowireCapableBeanFactory

&#x20;       │

&#x20;       ▼

DefaultListableBeanFactory



Each layer adds a specific responsibility while reusing functionality from the previous layer.



\--------------------------------------------------------------------------------



\## Responsibilities of Each Layer

BeanFactory (Interface)



* Defines the contract for the IoC container.
* Example methods:

  * getBean()
  * containsBean()
  * isSingleton()



&#x20;  It defines what a BeanFactory should do, not how.



* AbstractBeanFactory

  * Acts as the coordinator for bean retrieval.
  * Responsibilities:

    * Implements the getBean() workflow	
    * Checks the Singleton Cache
    * Retrieves BeanDefinitions
    * Coordinates bean creation
    * Delegates actual bean creation to AbstractAutowireCapableBeanFactory

&#x20;   It does not instantiate objects directly.



* AbstractAutowireCapableBeanFactory

  * Responsible for creating beans.
  * Responsibilities:

    * Select constructor
    * Resolve dependencies
    * Instantiate objects using Reflection
    * Perform constructor/field/setter injection
    * Populate bean properties
    * Execute initialization callbacks
    * Apply BeanPostProcessors

&#x20;   This is where Spring actually creates objects.



* DefaultListableBeanFactory

  * The default BeanFactory implementation used by Spring.
  * Additional responsibilities:

    * Maintains the BeanDefinition registry
    * Resolves beans by type
    * Handles multiple implementations
    * Applies @Primary
    * Applies @Qualifier
    * Supports efficient bean lookup

\----------------------------------------------------------------------------------

ApplicationContext

&#x20;       │

&#x20;       ▼

AbstractBeanFactory.getBean()

&#x20;       │

&#x20;       ▼

Singleton Cache

&#x20;       │

Found?

│

├── Yes → Return Bean

│

└── No

&#x20;       │

&#x20;       ▼

Read BeanDefinition

&#x20;       │

&#x20;       ▼

AbstractAutowireCapableBeanFactory.createBean()

&#x20;       │

&#x20;       ▼

Choose Constructor

&#x20;       │

&#x20;       ▼

Resolve Dependencies

&#x20;       │

&#x20;       ▼

Instantiate Object

&#x20;       │

&#x20;       ▼

Inject Dependencies

&#x20;       │

&#x20;       ▼

Initialize Bean

&#x20;       │

&#x20;       ▼

Store Singleton

&#x20;       │

&#x20;       ▼

Return Bean



\---------------------------------------------------------------------------------

\## Interview Discussion



Q1. Why doesn't AbstractBeanFactory instantiate beans directly?



Answer: Because object creation is a specialized concern. AbstractBeanFactory coordinates bean retrieval, while AbstractAutowireCapableBeanFactory encapsulates the complex logic for constructor resolution, dependency injection, initialization, and lifecycle management. This follows the Single Responsibility Principle.



Q2. Why did Spring choose inheritance (AbstractBeanFactory → AbstractAutowireCapableBeanFactory → DefaultListableBeanFactory) instead of putting everything into one class?



Answer: Each layer introduces a distinct capability without duplicating existing logic. This makes the framework extensible, easier to maintain, and aligned with the Open/Closed Principle.

