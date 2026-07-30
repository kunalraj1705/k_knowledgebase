# BeanFactory vs ApplicationContext


##Learning Objective

* Understand the responsibilities of ApplicationContext and BeanFactory, how Spring locates and creates beans, and how recursive dependency resolution works.


## Why does Spring have both BeanFactory and ApplicationContext?

* A factory should focus on creating and managing beans.
* A client should not directly interact with the factory.
* Spring introduces ApplicationContext as a higher-level abstraction that hides the internal complexity of the BeanFactory.
* Mental Model

  *                 Developer
  *                     Ã¢â€â€š
  *                     Ã¢â€“Â¼
  *          ApplicationContext
  *        (Public API / Showroom)
  *                     Ã¢â€â€š
  *                     Ã¢â€“Â¼
  *               BeanFactory
  *          (Bean Management Unit)
  *                     Ã¢â€â€š
  *     Ã¢â€Å’Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€Â¼Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€Â
  *     Ã¢â€“Â¼                     Ã¢â€“Â¼                    Ã¢â€“Â¼
  * BeanDefinition   Singleton Cache   Dependency Resolver
  *   Registry

Eg:

You want to buy a car, you go to showroom and get the car. You don't know how you car was built.

Here Showroom represent the ApplicationContext, Car factory represent the BeanFactory.


---
## Responsibilities

* ApplicationContext

  * Responsibilities:

    * Public entry point for developers
    * Delegates bean requests to BeanFactory
    * Event publishing
    * Resource loading
    * Environment \& Properties
    * Internationalization (i18n)

   ApplicationContext coordinates but does not perform bean creation.

* BeanFactory

  * Responsibilities:
  * Stores Bean Definitions
  * Creates Beans
  * Resolves Dependencies
  * Maintains Singleton Cache
  * Manages Bean Lifecycle

  BeanFactory is the core IoC container.

---
## Internal Components of BeanFactory

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
     *        Ã¢â€ â€œ
     * UserService Instance
     *
     * emailService
     *        Ã¢â€ â€œ
     * EmailService Instance
   * Every getBean() call first checks this cache.
3. Dependency Resolver

   * Determines:

     * Which constructor to use
     * Constructor arguments
     * Qualifier selection
     * Primary selection
     * Circular dependency detection


---
## Bean Retrieval Algorithm

getBean(UserService)


       Ã¢â€â€š

       Ã¢â€“Â¼

Check Singleton Cache


       Ã¢â€â€š

Found?

Ã¢â€â€š

Ã¢â€Å“Ã¢â€â‚¬Ã¢â€â‚¬ Yes Ã¢â€ â€™ Return Existing Bean

Ã¢â€â€š

Ã¢â€â€Ã¢â€â‚¬Ã¢â€â‚¬ No


       Ã¢â€â€š

       Ã¢â€“Â¼

Read BeanDefinition


       Ã¢â€â€š

       Ã¢â€“Â¼

Choose Constructor


       Ã¢â€â€š

       Ã¢â€“Â¼

Resolve Constructor Dependencies


       Ã¢â€â€š

       Ã¢â€“Â¼

Create Bean


       Ã¢â€â€š

       Ã¢â€“Â¼

Initialize Bean


       Ã¢â€â€š

       Ã¢â€“Â¼

Singleton?


       Ã¢â€â€š

Yes

       Ã¢â€â€š

       Ã¢â€“Â¼

Store in Singleton Cache


       Ã¢â€â€š

       Ã¢â€“Â¼

Return Bean


---
## Recursive Dependency Resolution

Eg:


UserService(

   UserRepository repository,

   NotificationService notificationService

)


Spring does not create UserService immediately.


Instead:


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


Resolve @Qualifier/@Primary


Ã¢â€ â€œ


Create EmailService


Ã¢â€ â€œ


Return


Ã¢â€ â€œ


Create UserService


Ã¢â€ â€œ


Return


NOTE: Spring recursively calls getBean() for dependencies until the dependency graph is fully satisfied.


---
## Constructor Selection

1. One Constructor : Spring automatically selects it.

   1. public UserService(UserRepository repo)
2. Multiple Constructors: Spring uses the constructor marked with @Autowired (or applies its constructor resolution rules if no annotation is present).

   1. UserService()
   2. UserService(UserRepository)
   3. UserService(UserRepository, EmailService)


---
## Bean Lookup Strategy

Spring instead maintains bean metadata (primarily indexed by bean name) and resolves candidates by type when needed, applying rules like:


Match by type

Apply @Qualifier

Apply @Primary

Throw exception if ambiguous


---
## Mental Model

Developer


Ã¢â€ â€œ


ApplicationContext


Ã¢â€ â€œ


BeanFactory


Ã¢â€ â€œ


Singleton Cache


Ã¢â€ â€œ


BeanDefinition Registry


Ã¢â€ â€œ


Dependency Resolution


Ã¢â€ â€œ


Bean Creation


Ã¢â€ â€œ


Bean Initialization


Ã¢â€ â€œ


Singleton Cache


Ã¢â€ â€œ


Return Bean


---
## Spring BeanFactory Architecture

Spring does not use a single BeanFactory implementation. Instead, it builds functionality through layered abstractions.

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


Each layer adds a specific responsibility while reusing functionality from the previous layer.


---
## Responsibilities of Each Layer

BeanFactory (Interface)


* Defines the contract for the IoC container.
* Example methods:

  * getBean()
  * containsBean()
  * isSingleton()


  It defines what a BeanFactory should do, not how.


* AbstractBeanFactory

  * Acts as the coordinator for bean retrieval.
  * Responsibilities:

    * Implements the getBean() workflow
    * Checks the Singleton Cache
    * Retrieves BeanDefinitions
    * Coordinates bean creation
    * Delegates actual bean creation to AbstractAutowireCapableBeanFactory

   It does not instantiate objects directly.


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

   This is where Spring actually creates objects.


* DefaultListableBeanFactory

  * The default BeanFactory implementation used by Spring.
  * Additional responsibilities:

    * Maintains the BeanDefinition registry
    * Resolves beans by type
    * Handles multiple implementations
    * Applies @Primary
    * Applies @Qualifier
    * Supports efficient bean lookup

---
ApplicationContext

       Ã¢â€â€š

       Ã¢â€“Â¼

AbstractBeanFactory.getBean()

       Ã¢â€â€š

       Ã¢â€“Â¼

Singleton Cache

       Ã¢â€â€š

Found?

Ã¢â€â€š

Ã¢â€Å“Ã¢â€â‚¬Ã¢â€â‚¬ Yes Ã¢â€ â€™ Return Bean

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

Resolve Dependencies

       Ã¢â€â€š

       Ã¢â€“Â¼

Instantiate Object

       Ã¢â€â€š

       Ã¢â€“Â¼

Inject Dependencies

       Ã¢â€â€š

       Ã¢â€“Â¼

Initialize Bean

       Ã¢â€â€š

       Ã¢â€“Â¼

Store Singleton

       Ã¢â€â€š

       Ã¢â€“Â¼

Return Bean


---
## Interview Discussion


Q1. Why doesn't AbstractBeanFactory instantiate beans directly?


Answer: Because object creation is a specialized concern. AbstractBeanFactory coordinates bean retrieval, while AbstractAutowireCapableBeanFactory encapsulates the complex logic for constructor resolution, dependency injection, initialization, and lifecycle management. This follows the Single Responsibility Principle.


Q2. Why did Spring choose inheritance (AbstractBeanFactory Ã¢â€ â€™ AbstractAutowireCapableBeanFactory Ã¢â€ â€™ DefaultListableBeanFactory) instead of putting everything into one class?


Answer: Each layer introduces a distinct capability without duplicating existing logic. This makes the framework extensible, easier to maintain, and aligned with the Open/Closed Principle.
