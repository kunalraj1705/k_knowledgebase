# Module 18 - Advanced Spring Boot Internals

---

# Objective

Understand how Spring Boot works internally by deriving its architecture from first principles instead of memorizing annotations.

This module focuses on:

- Why Spring Boot exists
- `@SpringBootApplication`
- `SpringApplication`
- Application startup
- ApplicationContext creation
- Environment and Property Sources
- Profiles
- Auto-Configuration
- Conditional Configuration
- Spring Boot Starters
- Configuration Properties and Binding
- Bean Lifecycle
- Embedded Server
- Application Events
- Actuator
- Startup optimization
- Lazy initialization
- Failure Analysis
- Custom Auto-Configuration
- Custom Spring Boot Starters
- Production architecture
- Readiness and Liveness
- Observability
- AOP Proxies
- Transaction boundaries
- Thread pools and connection pools
- Graceful shutdown

The goal is to understand what Spring Boot is actually doing when we execute:

```java
SpringApplication.run(Application.class, args);
```

---

# Why Spring Boot?

Before Spring Boot, building a Spring application required significant manual configuration.

A typical application required configuration for:

- ApplicationContext
- Component scanning
- DataSource
- TransactionManager
- EntityManager
- DispatcherServlet
- JSON conversion
- Embedded or external web server
- Application properties
- Infrastructure beans

The developer had to understand and configure much of this infrastructure manually.

This created problems:

- Large configuration
- Repeated setup
- Boilerplate
- Difficult environment management
- Dependency compatibility problems
- More configuration errors

Spring Boot was created to simplify this.

---

# Declarative Programming

Spring Boot follows the same declarative philosophy we have seen throughout Spring.

Instead of manually creating infrastructure:

```java
DataSource dataSource = ...;
EntityManagerFactory factory = ...;
DispatcherServlet servlet = ...;
```

we declare our intent:

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Spring Boot determines the required infrastructure based on:

- Dependencies
- Configuration
- Application type
- Existing beans
- Conditions

The developer declares **what the application needs**.

Spring Boot determines **how the infrastructure should be assembled**.

---

# Core Philosophy

Spring Boot follows:

```text
Convention over Configuration
```

But this does **not** mean:

```text
Convention instead of Configuration
```

Instead:

```text
Sensible Default
       ↓
Application Starts
       ↓
Need Custom Behavior?
       ↓
Explicit Configuration
       ↓
Boot Backs Off
```

This gives Spring Boot its central philosophy:

```text
Convention
    +
Customization
```

---

# Internal Architecture

```text
Application
      │
      ▼
SpringApplication
      │
      ▼
Environment
      │
      ▼
ApplicationContext
      │
      ├───────────────┐
      ▼               ▼
Component Scan   Auto-Configuration
      │               │
      │          Condition Evaluation
      │               │
      └───────┬───────┘
              ▼
       Bean Definitions
              │
              ▼
        Bean Creation
              │
              ▼
       Dependency Injection
              │
              ▼
       Bean Initialization
              │
              ▼
      ApplicationContext Ready
              │
              ▼
      Embedded Web Server
              │
              ▼
       Application Ready
```

---

# `@SpringBootApplication`

Most Spring Boot applications begin with:

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(
            Application.class,
            args
        );
    }
}
```

Conceptually:

```text
@SpringBootApplication
        │
        ├── @SpringBootConfiguration
        │
        ├── @EnableAutoConfiguration
        │
        └── @ComponentScan
```

Therefore:

```text
@SpringBootApplication
        │
        ├── Configuration
        ├── Auto-Configuration
        └── Component Discovery
```

---

# `@SpringBootConfiguration`

`@SpringBootConfiguration` identifies the class as a Spring Boot configuration class.

Conceptually it is a specialized form of:

```java
@Configuration
```

Therefore:

```java
@SpringBootConfiguration
public class Application {
}
```

tells Spring:

> This class is a source of application configuration.

---

# `@ComponentScan`

`@ComponentScan` tells Spring where to search for components.

Examples:

```java
@Component
@Service
@Repository
@Controller
@RestController
@Configuration
```

Suppose:

```text
com.krbacademy
│
├── Application
├── controller
├── service
├── repository
└── configuration
```

Spring can scan the package hierarchy and discover Spring-managed components.

Important:

Component scanning does **not** immediately mean every object is instantiated.

First, Spring discovers and registers bean definitions.

Bean creation happens later during the ApplicationContext lifecycle.

---

# Component Scan Flow

```text
@SpringBootApplication
        │
        ▼
@ComponentScan
        │
        ▼
Determine Base Package
        │
        ▼
Scan Classes
        │
        ▼
Identify Spring Components
        │
        ▼
Register Bean Definitions
```

---

# Engineering Principle 1

> Discover configuration and components before creating managed objects so the container can build the application dependency graph systematically.

---

# `@EnableAutoConfiguration`

This is the feature that gives Spring Boot its automatic configuration capability.

It tells Spring Boot:

> Examine the application environment and configure appropriate infrastructure automatically.

For example, if the application contains web dependencies:

```text
Spring MVC
Servlet API
Embedded Server
```

Spring Boot can configure appropriate web infrastructure.

If the application contains JPA dependencies:

```text
JPA
Hibernate
Database Driver
```

Spring Boot can configure appropriate persistence infrastructure.

---

# Conditional Configuration

Auto-Configuration is **not unconditional**.

Spring Boot does not simply say:

```text
JPA dependency present
        ↓
Create everything
```

Instead it evaluates conditions:

```text
Dependency Present?
       ↓
Existing Bean?
       ↓
Property Enabled?
       ↓
Application Type?
       ↓
Conditions Match?
       ↓
Apply Configuration
```

Spring Boot provides conditions such as:

```java
@ConditionalOnClass
@ConditionalOnBean
@ConditionalOnMissingBean
@ConditionalOnProperty
@ConditionalOnWebApplication
```

---

# `@ConditionalOnClass`

```java
@ConditionalOnClass(DataSource.class)
```

Meaning:

> Apply this configuration only if `DataSource` is available on the classpath.

```text
DataSource.class available?
        │
    ┌───┴───┐
   YES      NO
    │        │
    ▼        ▼
 Apply     Skip
```

---

# `@ConditionalOnBean`

```java
@ConditionalOnBean(DataSource.class)
```

Meaning:

> Apply this configuration only when a matching DataSource bean already exists.

This allows one configuration to depend on infrastructure created elsewhere.

---

# `@ConditionalOnMissingBean`

This is one of the most important Spring Boot conditions.

```java
@Bean
@ConditionalOnMissingBean
public PaymentClient paymentClient() {
    return new PaymentClient();
}
```

Conceptually:

```text
Does PaymentClient already exist?
        │
    ┌───┴───┐
   YES      NO
    │        │
    ▼        ▼
  Skip     Create
```

This is the **back-off mechanism**.

If the application provides its own bean:

```java
@Bean
PaymentClient customPaymentClient() {
    return new CustomPaymentClient();
}
```

Boot backs off and the application bean is used.

---

# Engineering Principle 2

> Framework defaults should provide sensible behavior without preventing explicit application configuration.

---

# `@ConditionalOnProperty`

```java
@ConditionalOnProperty(
    name = "feature.audit.enabled",
    havingValue = "true"
)
```

Configuration becomes active only when:

```properties
feature.audit.enabled=true
```

Useful for:

- Feature flags
- Optional infrastructure
- Environment-specific behavior
- Optional integrations

---

# `@ConditionalOnWebApplication`

Some configuration should only exist in web applications.

```text
Web Application?
       │
   ┌───┴───┐
  YES      NO
   │        │
   ▼        ▼
Apply     Skip
```

---

# Engineering Principle 3

> Infrastructure should be activated according to the actual application environment rather than being created unconditionally.

---

# Auto-Configuration Internal Flow

```text
@SpringBootApplication
        │
        ▼
@EnableAutoConfiguration
        │
        ▼
Discover Auto-Configuration Candidates
        │
        ▼
Load Candidate Configurations
        │
        ▼
Evaluate Conditions
        │
        ├── Condition TRUE
        │       ↓
        │   Apply Configuration
        │
        └── Condition FALSE
                ↓
              Skip
```

---

# Spring Boot Starters

Example:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

A Starter provides a curated collection of dependencies required for a capability.

---

# Why Starters Exist?

Without a starter, developers might manually add:

```text
Spring MVC
Servlet API
JSON support
Embedded server
Supporting dependencies
```

This creates:

- Dependency management complexity
- Version conflicts
- Configuration mistakes

Starters simplify dependency management.

---

# Starter vs Auto-Configuration

### Starter

Answers:

> What dependencies should the application have?

### Auto-Configuration

Answers:

> What infrastructure should Spring configure based on those dependencies and the environment?

```text
Starter
   ↓
Dependencies
   ↓
Classpath
   ↓
Auto-Configuration
   ↓
Conditions
   ↓
Beans
```

---

# Engineering Principle 4

> Dependency selection and runtime configuration are separate responsibilities.

---

# SpringApplication

`SpringApplication` is the central bootstrap component.

When we execute:

```java
SpringApplication.run(
    Application.class,
    args
);
```

Spring Boot uses `SpringApplication` to orchestrate application startup.

It coordinates:

- Application type detection
- Environment preparation
- ApplicationContext creation
- Application listeners
- Application initializers
- Configuration loading
- Context refresh
- Application lifecycle events
- Application startup completion

---

# Spring Boot Startup Flow

```text
main()
   │
   ▼
SpringApplication.run()
   │
   ▼
Create SpringApplication
   │
   ▼
Determine Application Type
   │
   ▼
Prepare Environment
   │
   ▼
Create ApplicationContext
   │
   ▼
Load Configuration
   │
   ▼
Component Scan
   │
   ▼
Auto-Configuration
   │
   ▼
Evaluate Conditions
   │
   ▼
Register Bean Definitions
   │
   ▼
Create Beans
   │
   ▼
Initialize Beans
   │
   ▼
Refresh ApplicationContext
   │
   ▼
Start Embedded Server
   │
   ▼
Application Ready
```

---

# Application Type Detection

Spring Boot determines whether the application is:

```text
Servlet Web Application
Reactive Web Application
Non-Web Application
```

Each type requires different infrastructure.

### Servlet

```text
Spring MVC
     ↓
Servlet API
     ↓
Embedded Tomcat
```

### Reactive

```text
Spring WebFlux
     ↓
Reactive Infrastructure
     ↓
Reactive Server
```

### Non-Web

No HTTP server is required.

---

# Engineering Principle 5

> Application infrastructure should match the application's execution model.

---

# Environment

The `Environment` represents the application's configuration space.

Configuration can come from:

- `application.properties`
- `application.yml`
- Profile-specific configuration
- Environment variables
- System properties
- Command-line arguments

```text
Configuration Sources
        │
        ▼
Environment
        │
        ▼
Application
```

---

# Property Sources

The Environment contains multiple `PropertySources`.

Conceptually:

```text
Command-Line Arguments
          │
System Properties
          │
Environment Variables
          │
application.properties
          │
application.yml
          │
Profile Configuration
          │
          ▼
      Environment
```

Spring Boot applies its defined property precedence rules when the same property exists in multiple sources.

Example:

```properties
server.port=8080
```

but:

```bash
java -jar application.jar --server.port=9090
```

The command-line value can override the file value.

---

# Engineering Principle 6

> Configuration should be externalized so the same application artifact can run across different environments without code changes.

---

# Profiles

Spring Boot supports environment-specific configuration:

```text
application.yml
application-dev.yml
application-test.yml
application-prod.yml
```

Conceptually:

```text
Common Configuration
        +
Active Profile
        ↓
Effective Configuration
```

Profiles should represent meaningful environment differences rather than arbitrary application branching.

---

# Configuration Properties

Suppose:

```yaml
app:
  payment:
    timeout: 5000
    retry-count: 3
    enabled: true
```

Instead of scattering many `@Value` fields, use:

```java
@ConfigurationProperties(prefix = "app.payment")
public class PaymentProperties {

    private int timeout;
    private int retryCount;
    private boolean enabled;
}
```

---

# Configuration Binding

```text
application.yml
      │
      ▼
Property Sources
      │
      ▼
Environment
      │
      ▼
Configuration Binder
      │
      ▼
PaymentProperties
      │
      ▼
Application
```

The Binder maps external configuration into Java objects.

Example:

```yaml
app:
  payment:
    retry-count: 3
```

maps conceptually to:

```java
retryCount = 3;
```

Spring Boot supports relaxed binding and type conversion.

---

# `@Value` vs `@ConfigurationProperties`

| `@Value` | `@ConfigurationProperties` |
|---|---|
| Individual properties | Related property group |
| Simple | Structured |
| Good for small values | Better for larger configuration |
| Scattered when overused | Centralized |
| Less configuration modeling | Type-safe configuration model |

---

# Engineering Principle 7

> Related configuration should be represented as a structured, type-safe object instead of being scattered throughout the application.

---

# Configuration Validation

Invalid configuration should fail early.

Example:

```yaml
app:
  payment:
    timeout: -1
    retry-count: -5
```

Conceptually:

```text
Application Startup
        │
        ▼
Bind Configuration
        │
        ▼
Validate
        │
   ┌────┴────┐
 Valid     Invalid
   │          │
   ▼          ▼
Continue    Fail Fast
```

Example:

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public class PaymentProperties {

    @Min(100)
    private int timeout;

    @Min(0)
    private int retryCount;
}
```

---

# Engineering Principle 8

> Invalid configuration should be detected as early as possible, preferably before the application begins serving traffic.

---

# ApplicationContext

The ApplicationContext is the central container that manages:

- Bean Definitions
- Bean Instances
- Dependency Injection
- Bean Lifecycle
- Events
- Resources
- Environment
- Configuration

Conceptually:

```text
ApplicationContext
       │
       ├── Bean Definitions
       ├── Environment
       ├── Event System
       └── BeanFactory
```

---

# Why ApplicationContext Exists

Suppose:

```text
Controller
   ↓
Service
   ↓
Repository
   ↓
DataSource
```

Without a container, developers manually construct the entire object graph.

Spring instead determines:

```text
What needs to be created?
        ↓
What does it depend on?
        ↓
What order should dependencies be created?
        ↓
How should objects be initialized?
```

---

# Bean Definition vs Bean Instance

A **Bean Definition** is metadata describing a bean.

A **Bean Instance** is the actual Java object.

```text
Bean Definition
      │
      ▼
Bean Class
      │
      ▼
Bean Instance
```

Spring needs the definitions before creating the object graph.

```text
Bean Definitions
      ↓
Dependency Graph
      ↓
Bean Creation
```

---

# Engineering Principle 9

> Configuration describes the object graph; the container is responsible for constructing and managing that object graph.

---

# Bean Lifecycle

A simplified lifecycle is:

```text
Bean Definition
      ↓
Instantiate Bean
      ↓
Populate Properties / Dependency Injection
      ↓
Aware Callbacks
      ↓
postProcessBeforeInitialization()
      ↓
@PostConstruct
      ↓
InitializingBean / init-method
      ↓
postProcessAfterInitialization()
      ↓
Final Bean
```

If AOP applies, the final exposed object may be a proxy.

---

# BeanPostProcessor

A common mistake is:

> BeanPostProcessor runs only after the bean is fully initialized.

That is incomplete.

A BeanPostProcessor participates around initialization:

```java
postProcessBeforeInitialization()
```

and:

```java
postProcessAfterInitialization()
```

The important distinction is:

```text
BeanPostProcessor instance
        ↓
Registered before normal bean creation
```

while:

```text
Individual Bean
        ↓
Instantiated
        ↓
Dependency Injection
        ↓
Before Initialization
        ↓
Initialization
        ↓
After Initialization
```

---

# Why Register BeanPostProcessors First?

If Spring did:

```text
Create Bean
    ↓
Register Processor
    ↓
Too Late
```

the processor could not participate in creation of that bean.

Instead:

```text
Register BeanPostProcessors
    ↓
Create Application Bean
    ↓
Process Bean
```

---

# Constructor Injection

Example:

```java
@Service
public class PaymentService {

    private final PaymentRepository repository;

    public PaymentService(
            PaymentRepository repository) {

        this.repository = repository;
    }
}
```

Benefits:

- Required dependencies are explicit
- Fields can be `final`
- Easier unit testing
- Object cannot exist without required dependencies
- Better immutability

---

# Engineering Principle 10

> Required dependencies should be supplied during object construction so invalid objects are difficult to create.

---

# Circular Dependencies

Example:

```text
ServiceA
   ↓
ServiceB
   ↓
ServiceA
```

This creates:

```text
A → B → A
```

Constructor injection exposes this problem clearly.

Circular dependencies often indicate incorrect responsibility boundaries.

Possible solutions:

- Extract shared behavior
- Introduce a third component
- Redesign responsibilities
- Use domain events where appropriate

---

# Engineering Principle 11

> Circular dependencies often indicate that responsibilities are incorrectly coupled.

---

# ApplicationContext Refresh

A major part of Spring startup is:

```java
refresh()
```

Conceptually:

```text
ApplicationContext
      ↓
refresh()
      ↓
Prepare BeanFactory
      ↓
Load Bean Definitions
      ↓
Register BeanPostProcessors
      ↓
Instantiate Required Beans
      ↓
Initialize Beans
      ↓
Publish Context Events
      ↓
Context Ready
```

Refresh transitions the context from configuration metadata to an operational object graph.

---

# BeanFactory vs ApplicationContext

```text
BeanFactory
    ↓
Basic IoC Container

ApplicationContext
    ↓
Advanced Container
```

ApplicationContext builds on BeanFactory capabilities and adds:

- Application events
- Message resolution
- Resource loading
- Environment integration
- Application-level features

---

# Engineering Principle 12

> Higher-level abstractions should build on lower-level mechanisms while adding application-level capabilities.

---

# Embedded Web Server

For a typical Spring Boot web application:

```text
SpringApplication
      ↓
ApplicationContext
      ↓
Web Server Infrastructure
      ↓
Embedded Tomcat
```

Traditional deployment:

```text
Build WAR
    ↓
Install Tomcat
    ↓
Deploy WAR
    ↓
Start Tomcat
```

Spring Boot commonly uses:

```text
Build JAR
    ↓
java -jar application.jar
    ↓
Start Embedded Server
    ↓
Application Running
```

---

# Embedded Tomcat Architecture

```text
Application
      ↓
Spring Boot
      ↓
Embedded Tomcat
      ↓
Servlet Container
      ↓
DispatcherServlet
      ↓
Controller
```

Tomcat does not directly execute the controller.

Tomcat provides the Servlet infrastructure.

Spring MVC's `DispatcherServlet` handles request dispatching.

---

# HTTP Request Flow

```text
Client
  ↓
HTTP Request
  ↓
Tomcat
  ↓
Servlet Container
  ↓
DispatcherServlet
  ↓
HandlerMapping
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
Database
```

---

# Engineering Principle 13

> Infrastructure layers should have clear responsibilities: the server handles transport, Spring MVC handles dispatching, and application services handle business logic.

---

# Application Events

Spring Boot exposes lifecycle events.

Conceptually:

```text
Application Starting
        ↓
Environment Prepared
        ↓
Application Context Initialized
        ↓
Context Refreshed
        ↓
Application Started
        ↓
Application Ready
```

Example:

```java
@EventListener
public void handleReady(ApplicationReadyEvent event) {
    // Application is ready
}
```

---

# `@PostConstruct` vs `ApplicationReadyEvent`

| `@PostConstruct` | `ApplicationReadyEvent` |
|---|---|
| Bean initialization stage | Application startup completion |
| Runs for a bean | Application lifecycle event |
| Before application is fully ready | After application is ready |
| Good for bean initialization | Good for startup-complete tasks |

Do not use lifecycle events as an excuse to execute huge startup workloads.

---

# Engineering Principle 14

> Execute work at the lifecycle stage that matches its actual responsibility.

---

# Actuator

Spring Boot Actuator provides production-oriented monitoring and management capabilities.

It can expose:

- Health
- Metrics
- Application information
- Environment information
- Bean information
- Request mappings
- Loggers

---

# Why Actuator?

Production systems need answers to:

```text
Is the application healthy?
Is the database reachable?
How much memory is being used?
What is the request rate?
What is the request latency?
Which endpoints are registered?
```

---

# Actuator Architecture

```text
Monitoring System
       ↓
Actuator Endpoint
       ↓
Spring Boot Management Infrastructure
       ↓
Application / JVM / Dependencies
```

---

# Metrics

Spring Boot integrates with Micrometer for application metrics.

Typical metrics:

```text
HTTP Requests
Request Duration
JVM Memory
CPU
Threads
Connection Pools
Database Metrics
```

---

# Actuator Security

Management endpoints can expose sensitive information.

Production systems should consider:

- Authentication
- Authorization
- Network restrictions
- Endpoint exposure
- Sensitive configuration
- Operational access control

---

# Engineering Principle 15

> Operational interfaces should be protected according to their sensitivity and exposure requirements.

---

# Auto-Configuration Discovery

Modern Spring Boot uses:

```text
META-INF/spring/
org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

to discover Auto-Configuration candidates.

The process is:

```text
AutoConfiguration.imports
        ↓
Auto-Configuration Candidates
        ↓
Condition Evaluation
        ↓
Matching Configurations
        ↓
Bean Definitions
```

---

# Discovery Does Not Mean Activation

A discovered Auto-Configuration class is only a candidate.

```text
Candidate Found
      ↓
Conditions Evaluated
      ↓
Condition Matches?
      │
   ┌──┴──┐
  YES    NO
   │      │
   ▼      ▼
Apply    Skip
```

---

# Why This Architecture?

Suppose:

```text
100 Auto-Configuration Classes
```

but the application needs only:

```text
10
```

Unconditional configuration would create:

- Unnecessary beans
- Unnecessary memory usage
- Unnecessary initialization
- Unwanted infrastructure
- Configuration conflicts

Conditional evaluation prevents this.

---

# Condition Evaluation Report

Spring Boot can provide information about why Auto-Configuration was applied or skipped.

This helps answer:

```text
Why did Spring create this bean?
Why did Spring not create this bean?
Why is this Auto-Configuration active?
Why is my configuration being ignored?
```

Debug output can be enabled with:

```properties
debug=true
```

Conceptually:

```text
Positive Matches
      ↓
Configurations Applied

Negative Matches
      ↓
Configurations Skipped
```

---

# Engineering Principle 16

> Framework behavior should be observable enough that developers can determine why a configuration was applied or skipped.

---

# Custom Auto-Configuration

Suppose we build:

```text
KRB Observability Starter
```

We want another application to simply add the dependency.

Architecture:

```text
Application
      ↓
Add Starter
      ↓
Dependencies Added
      ↓
Auto-Configuration Discovered
      ↓
Conditions Evaluated
      ↓
Beans Created
```

Example:

```java
@AutoConfiguration
@ConditionalOnClass(KrbClient.class)
public class KrbClientAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public KrbClient krbClient() {
        return new KrbClient();
    }
}
```

---

# Custom Auto-Configuration Design

The logic is:

```text
KrbClient class available?
        │
       YES
        ↓
Does KrbClient bean already exist?
        │
   ┌────┴────┐
  YES       NO
   │         │
   ▼         ▼
 Skip      Create
```

A reusable library should:

```text
No Application Configuration
        ↓
Provide Default

Application Configuration
        ↓
Back Off
```

---

# Engineering Principle 17

> Reusable infrastructure should provide defaults without taking ownership away from the application.

---

# Configuration Binding Internals

The flow is:

```text
Configuration File
        ↓
PropertySource
        ↓
Environment
        ↓
Binder
        ↓
ConfigurationProperties Object
```

The Binder maps external configuration into Java objects.

Example:

```text
app.payment.retry-count
            ↓
        retryCount
```

Spring Boot supports relaxed binding and type conversion.

---

# Why Binder?

Without a Binder, developers would manually retrieve and convert configuration values.

The Binder centralizes conversions such as:

```text
String
   ↓
Integer
   ↓
Boolean
   ↓
Duration
   ↓
Data Size
   ↓
Java Types
```

---

# Startup Performance

Large applications may contain:

```text
500+ Beans
100+ Dependencies
Many Auto-Configurations
Database
Kafka
Redis
Security
Observability
```

Startup involves:

```text
Class Loading
      ↓
Configuration Processing
      ↓
Condition Evaluation
      ↓
Bean Creation
      ↓
Dependency Injection
      ↓
Proxy Creation
      ↓
Initialization
      ↓
Embedded Server
```

---

# Why Startup Performance Matters

Suppose:

```text
Application Startup = 90 seconds
```

This affects:

- Deployment time
- Kubernetes pod replacement
- Auto Scaling
- Rolling Deployments
- Failure Recovery
- Developer Productivity

---

# Engineering Principle 18

> Application startup time directly affects deployment speed, scaling speed and recovery time.

---

# Measuring Startup

Before optimizing:

> Where is the time actually being spent?

Use:

```text
Measure
   ↓
Identify Bottleneck
   ↓
Optimize Actual Bottleneck
   ↓
Validate
```

Do not blindly enable lazy initialization or remove dependencies.

---

# Common Startup Bottlenecks

- Large dependency graph
- Excessive component scanning
- Too many beans
- Heavy `@PostConstruct` logic
- External service calls
- Database initialization
- Large object creation
- Unnecessary Auto-Configuration
- Expensive proxy creation

---

# Lazy Initialization

### Eager

```text
Application Start
      ↓
Create Bean
      ↓
Initialize Bean
      ↓
Application Ready
```

### Lazy

```text
Application Start
      ↓
Bean Definition
      ↓
Application Ready

Later

Request
      ↓
Bean Required
      ↓
Create Bean
      ↓
Initialize Bean
```

Lazy initialization can improve startup time but may increase first-request latency.

---

# Lazy Initialization Trade-Off

Without lazy initialization:

```text
Startup
 ↓
Failure detected
```

With lazy initialization:

```text
Startup
 ↓
Success

First Request
 ↓
Bean Creation
 ↓
Failure
```

Therefore:

> Faster startup does not necessarily mean faster application behavior overall.

---

# Engineering Principle 19

> Startup latency and runtime latency are separate optimization problems.

---

# Component Scanning Optimization

Suppose:

```text
com.company
```

contains:

```text
10,000 classes
```

but the application only needs:

```text
com.company.payment
```

Scanning the entire package hierarchy can create unnecessary discovery work.

Good boundaries:

```text
com.krbacademy.payment
│
├── controller
├── service
├── repository
└── configuration
```

---

# Engineering Principle 20

> Component scanning boundaries should reflect application ownership and dependency boundaries.

---

# Auto-Configuration Optimization

If a specific Auto-Configuration is unnecessary, it may sometimes be excluded:

```java
@SpringBootApplication(
    exclude = {
        SomeAutoConfiguration.class
    }
)
public class Application {
}
```

But exclusions should be used carefully.

```text
Measure
 ↓
Understand
 ↓
Exclude Only When Justified
```

---

# Engineering Principle 21

> Disable framework infrastructure only when you understand why it is unnecessary.

---

# External Calls During Startup

Bad example:

```java
@PostConstruct
public void initialize() {
    paymentClient.loadConfiguration();
}
```

Now startup depends on:

```text
Application
    ↓
External Service
    ↓
Network
    ↓
Response
    ↓
Bean Initialization
```

If the dependency is mandatory, failing startup may be correct.

If optional, consider:

- Background initialization
- Retry
- Caching
- Graceful degradation
- Lazy initialization

---

# Engineering Principle 22

> Startup dependencies should reflect actual application readiness requirements.

---

# Database Initialization

Database initialization can affect startup:

```text
Application
      ↓
DataSource
      ↓
Database Connection
      ↓
Schema Initialization
      ↓
Migration
      ↓
Application Ready
```

Large migrations can:

- Slow startup
- Hold locks
- Affect traffic
- Complicate multi-instance deployments

Database migrations should therefore be treated as a deployment concern, not merely an application startup concern.

---

# Kubernetes Startup

Suppose:

```text
Startup Time = 60 seconds
```

A replacement pod may need:

```text
Pod
 ↓
Application Startup
 ↓
60 seconds
 ↓
Ready
```

Proper readiness handling is therefore critical.

---

# Readiness vs Liveness

### Liveness

> Should this process be restarted?

### Readiness

> Can this instance safely receive traffic?

An application may be:

```text
Liveness = UP
Readiness = NOT READY
```

For example, the process is running but a mandatory dependency is not yet available.

---

# Production Architecture

```text
Kubernetes
    │
    ├── Liveness
    │       ↓
    │   Process Healthy?
    │
    └── Readiness
            ↓
       Can Receive Traffic?
```

Spring Boot Actuator can integrate with application health and availability concepts used for these probes.

---

# Engineering Principle 23

> Process health and traffic readiness are different operational concerns.

---

# Application Events

Spring Boot provides lifecycle events such as:

```text
ApplicationStartingEvent
ApplicationEnvironmentPreparedEvent
ApplicationContextInitializedEvent
ApplicationPreparedEvent
ApplicationStartedEvent
ApplicationReadyEvent
```

Events allow components to react to lifecycle stages without direct coupling.

---

# `ApplicationReadyEvent`

Example:

```java
@EventListener
public void onApplicationReady(
        ApplicationReadyEvent event) {

    // Application startup completed
}
```

Useful when logic should execute after application startup.

But:

```text
Lifecycle Event
    ≠
Free Background Execution
```

Expensive work still has a cost.

---

# Engineering Principle 24

> Lifecycle events provide timing and decoupling; they do not remove the cost of the work being executed.

---

# Failure Analysis

Spring Boot attempts to make common startup failures easier to diagnose.

Typical categories:

- Missing Bean
- Invalid Property
- Port Already In Use
- Database Configuration
- Dependency Problem

Conceptually:

```text
Exception
    ↓
Failure Analysis
    ↓
Description
    ↓
Possible Action
```

---

# Why Failure Analysis Matters

Compare:

```text
BeanCreationException
```

with:

```text
Application failed because required
DataSource configuration is missing.
Configure the datasource URL or provide
a DataSource bean.
```

The second message reduces diagnosis time.

---

# Engineering Principle 25

> Good frameworks should turn low-level failures into actionable diagnostic information.

---

# Startup Failure vs Runtime Failure

Bad:

```text
Application starts
      ↓
Traffic accepted
      ↓
Database operation
      ↓
Failure
```

Better:

```text
Application Startup
      ↓
Validate Required Infrastructure
      ↓
Failure
      ↓
Application Never Becomes Ready
```

---

# Engineering Principle 26

> Required infrastructure failures should prevent the application from becoming ready.

---

# Custom Spring Boot Starter

Suppose we create:

```text
KRB Observability Starter
```

Application developers should only need to add the dependency.

Then Spring Boot automatically provides:

```text
Metrics
Logging
Tracing
Health Integration
```

Architecture:

```text
KRB Starter
      ↓
Dependencies
      ↓
AutoConfiguration.imports
      ↓
KrbObservabilityAutoConfiguration
      ↓
Conditions
      ↓
Beans
```

---

# Custom Auto-Configuration Example

```java
@AutoConfiguration
@ConditionalOnClass(KrbMetricsClient.class)
public class KrbObservabilityAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public KrbMetricsClient krbMetricsClient() {
        return new KrbMetricsClient();
    }
}
```

The application gets a default implementation.

But:

```java
@Bean
KrbMetricsClient customClient() {
    return new CustomMetricsClient();
}
```

causes the default configuration to back off.

---

# Custom Starter Design Rules

A good starter should:

```text
Provide Defaults
      ↓
Be Conditional
      ↓
Be Configurable
      ↓
Back Off
      ↓
Avoid Surprising Side Effects
```

A bad starter:

```text
Force Beans
Override Application
Perform Network Calls
Create Heavy Infrastructure
```

---

# Engineering Principle 27

> A reusable Spring Boot starter should be predictable, conditional, configurable and non-invasive.

---

# Configuration Design for Starters

Example:

```yaml
krb:
  observability:
    enabled: true
    endpoint: https://metrics.example.com
    timeout: 2s
```

Use:

```java
@ConfigurationProperties(prefix = "krb.observability")
public class KrbObservabilityProperties {

    private boolean enabled;
    private String endpoint;
    private Duration timeout;
}
```

Flow:

```text
Configuration
      ↓
Binder
      ↓
Properties Object
      ↓
Auto-Configuration
      ↓
Infrastructure
```

---

# Conditional Starter Configuration

Example:

```java
@AutoConfiguration
@EnableConfigurationProperties(
    KrbObservabilityProperties.class
)
@ConditionalOnProperty(
    prefix = "krb.observability",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true
)
public class KrbObservabilityAutoConfiguration {
}
```

Conceptually:

```text
Enabled?
   │
 ┌─┴─┐
YES NO
 │   │
 ▼   ▼
Run Skip
```

---

# Production Concern: Network Calls in Starters

A starter should be especially careful about:

```text
Application Startup
      ↓
Starter
      ↓
Network Call
```

A reusable library does not know whether the external system is available.

Therefore libraries should avoid hidden startup side effects.

---

# Engineering Principle 28

> Libraries should minimize hidden side effects because the consuming application cannot easily reason about behavior it did not explicitly request.

---

# Production Application Architecture

A production Spring Boot application can be viewed as:

```text
Application
     │
     ▼
SpringApplication
     │
     ▼
Environment
     │
     ▼
ApplicationContext
     │
     ├── Component Scanning
     ├── Auto-Configuration
     ├── Configuration Binding
     └── Bean Lifecycle
     │
     ▼
Infrastructure Ready
     │
     ▼
Embedded Server
     │
     ▼
Readiness
     │
     ▼
Traffic
```

The important concept is:

```text
Running
    ≠
Ready
```

---

# Readiness Design

Not every dependency must necessarily be required for readiness.

Suppose:

```text
Primary Database
```

is mandatory.

But:

```text
Analytics Service
```

is optional.

Then:

```text
Database Down
    ↓
Application NOT READY
```

while:

```text
Analytics Down
    ↓
Application MAY remain READY
```

if business requirements allow graceful degradation.

---

# Engineering Principle 29

> Readiness should represent the minimum set of conditions required for the application to safely serve traffic.

---

# Observability Architecture

Production systems require more than logs.

A useful observability model is:

```text
Logs
+
Metrics
+
Traces
+
Health
```

Conceptually:

```text
Application
    │
    ├── Logs
    ├── Metrics
    ├── Traces
    └── Health
           │
           ▼
      Observability
```

---

# Logs

Logs answer:

> What happened?

Example:

```text
Payment request failed
Transaction ID = 12345
Reason = Timeout
```

---

# Metrics

Metrics answer:

> How much? How often? How long?

Examples:

```text
Request Count
Error Rate
Latency
CPU
Memory
Thread Count
Connection Pool Usage
```

---

# Traces

Distributed tracing answers:

> Where did the request spend its time?

Example:

```text
API Gateway
    ↓
Payment Service
    ↓
Fraud Service
    ↓
Database
```

A trace can reveal:

```text
Gateway        10 ms
Payment        40 ms
Fraud          800 ms
Database       20 ms
```

The bottleneck becomes visible.

---

# Engineering Principle 30

> Logs explain events, metrics reveal system behavior, and traces explain request execution across distributed components.

---

# Actuator in Production

Actuator can expose:

```text
Health
Metrics
Info
Mappings
Loggers
```

But production exposure must be controlled.

Avoid:

```text
Expose Everything
     ↓
Public Internet
```

Prefer:

```text
Management Endpoints
        ↓
Authentication
        ↓
Authorization
        ↓
Restricted Network
```

---

# Engineering Principle 31

> Management interfaces should have a smaller security boundary than public business APIs.

---

# Configuration Architecture in Production

A production application should separate:

```text
Application Code
```

from:

```text
Environment Configuration
```

For example:

```text
Same Application Artifact

        │
        ├── Development Configuration
        ├── Testing Configuration
        └── Production Configuration
```

---

# Configuration Hierarchy

Conceptually:

```text
Defaults
   ↓
Configuration Files
   ↓
Profile Configuration
   ↓
Environment Variables
   ↓
Command-Line Arguments
   ↓
Effective Configuration
```

The exact precedence depends on Spring Boot's configuration rules.

The important principle is:

> More environment-specific configuration can override less specific defaults.

---

# Secrets

Do not place sensitive credentials directly in source code.

Bad:

```java
String password = "MyProductionPassword";
```

Also avoid committing production secrets into source-controlled configuration.

Instead use appropriate secret-management mechanisms.

Conceptually:

```text
Secret Store
     ↓
Runtime Configuration
     ↓
Spring Environment
     ↓
Application
```

---

# Engineering Principle 32

> Configuration and secrets should be externalized from application code and managed according to their sensitivity.

---

# Profiles in Production

Profiles are useful for meaningful environment differences.

However, profiles should not become a replacement for proper deployment configuration management.

---

# Engineering Principle 33

> Profiles should express meaningful environment or deployment differences, not become a mechanism for arbitrary application branching.

---

# Dependency Management

Spring Boot starters simplify dependencies.

But every dependency introduces:

```text
Classpath
+
Memory
+
Startup Work
+
Security Surface
+
Maintenance
```

Adding a dependency can also activate additional Auto-Configuration.

---

# Engineering Principle 34

> A dependency is not free; every library adds operational and maintenance cost.

---

# Production Bean Design

A Spring bean should ideally have:

- Clear responsibility
- Explicit dependencies
- Predictable initialization
- Minimal side effects
- Appropriate lifecycle

Avoid:

```java
@PostConstruct
public void initialize() {

    // Huge database operation
    // External API call
    // Complex business process
}
```

---

# Engineering Principle 35

> Bean initialization should establish object readiness, not execute large business workflows.

---

# AOP and Proxies

Spring Boot applications frequently use:

```text
Transactions
Security
Caching
Logging
Metrics
Tracing
```

Many of these capabilities are implemented using proxies and interceptors.

Conceptually:

```text
Client
  ↓
Proxy
  ↓
Interceptor
  ↓
Target Bean
```

For example:

```text
@Transactional
      ↓
Proxy
      ↓
Transaction Interceptor
      ↓
Service Method
```

---

# Why Proxies Matter

The object injected into another bean may not be the raw target object.

Instead:

```text
Injected Reference
      ↓
Proxy
      ↓
Target Object
```

This explains many Spring behaviors.

---

# Self Invocation Problem

Consider:

```java
@Service
public class PaymentService {

    @Transactional
    public void payment() {
        ...
    }

    public void process() {
        payment();
    }
}
```

The call:

```text
process()
```

then:

```text
payment()
```

is a direct call within the same object.

It does not necessarily pass through the Spring proxy.

Conceptually:

```text
External Caller
      ↓
Proxy
      ↓
process()

Inside Target
      ↓
this.payment()
      ↓
Target directly
```

Therefore proxy-based interception may not occur for the internal call.

---

# Engineering Principle 36

> Proxy-based behavior applies when execution passes through the proxy; direct self-invocation can bypass proxy interception.

---

# Transaction Boundaries

A common production design is:

```text
Controller
    ↓
Service
    ↓
Repository
```

Transaction boundaries typically belong around business operations.

Example:

```java
@Transactional
public void processPayment() {

    validate();
    reserve();
    persist();
}
```

---

# Avoid Huge Transactions

Bad:

```text
@Transactional
process100000Records()
```

Potential problems:

- Large Persistence Context
- Long Database Locks
- Memory Growth
- Long Transaction Duration
- Increased Rollback Cost

---

# Engineering Principle 37

> Transaction boundaries should represent business consistency requirements, not simply method boundaries.

---

# Thread Pools and Connection Pools

Spring Boot applications often use:

- Web request threads
- Async executors
- Scheduled tasks
- Database connection pools

A common mistake is:

```text
More Threads
=
More Throughput
```

Not necessarily.

If the database supports:

```text
50 connections
```

and the application creates:

```text
500 worker threads
```

many threads compete for limited database capacity.

---

# Resource Flow

```text
Requests
    ↓
Application Threads
    ↓
Connection Pool
    ↓
Database
```

The bottleneck may be the database rather than CPU.

---

# Engineering Principle 38

> Concurrency should be sized according to downstream capacity, not merely application CPU capacity.

---

# Graceful Shutdown

Production applications must also shut down correctly.

Simplified flow:

```text
Shutdown Signal
      ↓
Application Context
      ↓
Stop Accepting New Work
      ↓
Complete / Cancel Existing Work
      ↓
Destroy Beans
      ↓
Close Resources
      ↓
Process Exit
```

This matters during:

- Deployments
- Kubernetes pod termination
- Scaling
- Failover

---

# Why Graceful Shutdown?

If a payment request is executing and the process is killed immediately:

```text
Request
   ↓
Process Killed
   ↓
Operation Interrupted
```

Potential consequences:

- Failed requests
- Partial processing
- Resource leaks
- Unclear transaction state

Graceful shutdown gives the application an opportunity to finish appropriate work and release resources.

---

# Engineering Principle 39

> Production applications should be designed for both graceful startup and graceful shutdown.

---

# Complete Production Mental Model

```text
                    Application
                         │
                         ▼
                SpringApplication
                         │
                         ▼
                    Environment
                         │
                         ▼
                 ApplicationContext
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     Components     Auto-Config     Properties
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                    Bean Graph
                         │
                         ▼
                 Bean Lifecycle
                         │
                         ▼
                 AOP / Proxies
                         │
                         ▼
                 Application Ready
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        Logs          Metrics         Traces
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                  Observability
                         │
                         ▼
                    Production
```

---

# Complete Spring Boot Startup Architecture

```text
main()
  │
  ▼
SpringApplication.run()
  │
  ▼
Create SpringApplication
  │
  ▼
Determine Application Type
  │
  ▼
Prepare Environment
  │
  ├── Configuration Files
  ├── Profiles
  ├── Environment Variables
  ├── System Properties
  └── Command-Line Arguments
  │
  ▼
Create ApplicationContext
  │
  ├── Component Scan
  │
  └── Auto-Configuration
          │
          ▼
      Discover Candidates
          │
          ▼
      Evaluate Conditions
          │
          ▼
      Register Bean Definitions
  │
  ▼
Register BeanPostProcessors
  │
  ▼
Create Beans
  │
  ▼
Dependency Injection
  │
  ▼
Bean Initialization
  │
  ├── Before Initialization
  ├── @PostConstruct
  ├── Initialization Callbacks
  └── After Initialization
  │
  ▼
AOP / Proxy Handling
  │
  ▼
ApplicationContext Ready
  │
  ▼
Embedded Server
  │
  ▼
Application Started
  │
  ▼
Application Ready
  │
  ▼
Traffic
  │
  ├── Logs
  ├── Metrics
  ├── Traces
  └── Health
```

---

# Engineering Principles - Complete Module

### Principle 1

> Discover configuration and components before creating managed objects so the container can build the application dependency graph systematically.

### Principle 2

> Framework defaults should provide sensible behavior without preventing explicit application configuration.

### Principle 3

> Infrastructure should be activated according to the actual application environment.

### Principle 4

> Dependency selection and runtime configuration are separate responsibilities.

### Principle 5

> Application infrastructure should match the application's execution model.

### Principle 6

> Configuration should be externalized so the same application artifact can run across different environments.

### Principle 7

> Related configuration should be represented as a structured, type-safe object.

### Principle 8

> Invalid configuration should be detected as early as possible.

### Principle 9

> Configuration describes the object graph; the container constructs and manages it.

### Principle 10

> Required dependencies should be supplied during object construction.

### Principle 11

> Circular dependencies often indicate incorrect responsibility boundaries.

### Principle 12

> Higher-level abstractions should build on lower-level mechanisms.

### Principle 13

> Infrastructure layers should have clearly separated responsibilities.

### Principle 14

> Execute work at the lifecycle stage that matches its responsibility.

### Principle 15

> Operational interfaces should be protected according to their sensitivity.

### Principle 16

> Framework behavior should be observable enough to explain why configuration was applied or skipped.

### Principle 17

> Reusable infrastructure should provide defaults without taking ownership away from the application.

### Principle 18

> Application startup time affects deployment, scaling and recovery.

### Principle 19

> Startup latency and runtime latency are separate optimization problems.

### Principle 20

> Required infrastructure failures should preferably occur before traffic is accepted.

### Principle 21

> Component scanning boundaries should reflect application boundaries.

### Principle 22

> Disable framework infrastructure only when its behavior is understood.

### Principle 23

> Startup dependencies should reflect actual readiness requirements.

### Principle 24

> Liveness and readiness represent different operational questions.

### Principle 25

> Lifecycle events provide timing and decoupling, not free execution.

### Principle 26

> Failure information should be actionable.

### Principle 27

> Required infrastructure failures should prevent the application from becoming ready.

### Principle 28

> A reusable Spring Boot starter should be predictable, conditional, configurable and non-invasive.

### Principle 29

> Libraries should minimize hidden side effects.

### Principle 30

> Readiness should represent the minimum conditions required to safely serve traffic.

### Principle 31

> Logs explain events, metrics reveal system behavior, and traces explain distributed request execution.

### Principle 32

> Management interfaces should have a smaller security boundary than public business APIs.

### Principle 33

> Configuration and secrets should be externalized from application code.

### Principle 34

> Profiles should express meaningful environment or deployment differences.

### Principle 35

> A dependency is not free; every library adds operational and maintenance cost.

### Principle 36

> Bean initialization should establish object readiness, not execute large business workflows.

### Principle 37

> Proxy-based behavior applies when execution passes through the proxy; direct self-invocation can bypass proxy interception.

### Principle 38

> Transaction boundaries should represent business consistency requirements, not simply method boundaries.

### Principle 39

> Concurrency should be sized according to downstream capacity.

### Principle 40

> Production applications should be designed for both graceful startup and graceful shutdown.

---

# KRB Academy Summary

Spring Boot is not magic.

It is a collection of mechanisms that automatically assemble Spring infrastructure based on:

```text
Dependencies
+
Configuration
+
Application Type
+
Existing Beans
+
Conditions
```

The fundamental architecture is:

```text
SpringApplication
       ↓
Environment
       ↓
ApplicationContext
       ↓
Component Scan
       +
Auto-Configuration
       ↓
Bean Definitions
       ↓
Bean Creation
       ↓
Dependency Injection
       ↓
Bean Lifecycle
       ↓
AOP / Proxies
       ↓
Embedded Infrastructure
       ↓
Application Ready
```

Spring Boot provides sensible defaults, but those defaults are designed to back off when explicit application configuration is present.

That gives us:

```text
Convention
    +
Customization
```

For production systems, the mental model must go further:

```text
Build
 ↓
Configure
 ↓
Start
 ↓
Validate
 ↓
Become Ready
 ↓
Serve
 ↓
Observe
 ↓
Scale
 ↓
Shutdown Gracefully
```

The senior engineer's responsibility is not merely to make the application run.

It is to understand the trade-offs behind:

- Startup
- Configuration
- Auto-Configuration
- Bean Lifecycle
- Proxies
- Transactions
- Concurrency
- Observability
- Readiness
- Failure
- Shutdown

---

# Senior Interview Questions

## Q1. Why was Spring Boot created?

Spring Boot was created to reduce repetitive Spring configuration, simplify dependency management, provide sensible defaults and make applications easier to run and deploy.

## Q2. What does `@SpringBootApplication` contain?

Conceptually:

```text
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

## Q3. What is Auto-Configuration?

Auto-Configuration automatically configures Spring infrastructure based on:

- Classpath dependencies
- Existing beans
- Properties
- Application type
- Conditional rules

## Q4. Is Auto-Configuration unconditional?

No. It is heavily based on conditional configuration.

## Q5. Where does modern Spring Boot discover Auto-Configuration candidates?

From:

```text
META-INF/spring/
org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

## Q6. What is `@ConditionalOnClass`?

It activates configuration when a specified class is available on the application's classpath.

## Q7. What is `@ConditionalOnMissingBean`?

It allows Spring Boot to create a default bean only when the application has not already provided a matching bean.

## Q8. What is the back-off mechanism?

When application configuration already provides infrastructure that Auto-Configuration would otherwise create, the Auto-Configuration can back off.

## Q9. What is the difference between a Starter and Auto-Configuration?

Starter provides a curated dependency set.

Auto-Configuration configures runtime infrastructure.

## Q10. What is `SpringApplication`?

`SpringApplication` is the central Spring Boot startup orchestrator responsible for coordinating environment preparation, ApplicationContext creation, configuration processing, lifecycle events and application startup.

## Q11. What is the Environment?

The Environment represents the application's configuration space and provides access to properties from multiple sources.

## Q12. What are PropertySources?

PropertySources are the different sources from which Spring obtains configuration, such as configuration files, environment variables, system properties, command-line arguments and profile-specific configuration.

## Q13. What is `@ConfigurationProperties`?

It provides structured and type-safe binding of external configuration into Java objects.

## Q14. Why validate configuration during startup?

To fail fast when configuration is invalid instead of allowing the application to start in an invalid state.

## Q15. What is ApplicationContext?

ApplicationContext is Spring's central application container that manages bean definitions, bean instances, dependency injection, lifecycle, events, environment and other infrastructure.

## Q16. What is the difference between BeanDefinition and Bean instance?

BeanDefinition is metadata describing how a bean should be created and managed.

Bean instance is the actual Java object created by the container.

## Q17. When does BeanPostProcessor run?

BeanPostProcessors are registered before normal application beans are created.

For each bean, their callbacks participate around initialization:

```text
postProcessBeforeInitialization()
        ↓
Initialization callbacks
        ↓
postProcessAfterInitialization()
```

## Q18. Does BeanPostProcessor run before bean creation?

No.

The BeanPostProcessor itself is registered before normal bean creation, but it processes an individual bean only after that bean has been instantiated.

## Q19. Why is constructor injection preferred?

It makes required dependencies explicit, supports immutability, improves testability and prevents construction of an object without its required dependencies.

## Q20. What is a circular dependency?

A circular dependency occurs when two or more beans depend on each other directly or indirectly.

Example:

```text
A → B → A
```

## Q21. Why are circular dependencies usually considered a design problem?

They indicate strong coupling and often suggest that responsibilities should be separated or redesigned.

## Q22. What is the difference between BeanFactory and ApplicationContext?

BeanFactory provides core IoC container functionality.

ApplicationContext builds on it and provides additional application-level capabilities such as events, resource loading and environment integration.

## Q23. What is the role of embedded Tomcat?

Embedded Tomcat provides the Servlet container and web-server infrastructure required to run a Spring MVC application without deploying to an externally installed Tomcat server.

## Q24. Does Tomcat call the Controller directly?

No.

The request enters Tomcat's Servlet container and is dispatched through Spring MVC's `DispatcherServlet`.

## Q25. `@PostConstruct` vs `ApplicationReadyEvent`?

`@PostConstruct` belongs to bean initialization.

`ApplicationReadyEvent` represents the application lifecycle stage where startup has completed and the application is ready.

## Q26. What is Spring Boot Actuator?

Actuator provides production-oriented monitoring and management capabilities such as health, metrics and application information.

## Q27. Why should Actuator endpoints be secured?

Because management endpoints may expose sensitive operational information.

## Q28. How would you troubleshoot slow Spring Boot startup?

First measure startup and identify where time is being spent. Then investigate bean initialization, Auto-Configuration, component scanning, external calls, database initialization and expensive lifecycle callbacks.

## Q29. Does lazy initialization reduce the cost of bean creation?

Not necessarily. It changes when the bean is created. The cost may move from startup to the first request.

## Q30. Why can lazy initialization be dangerous?

A required bean may fail during the first request instead of failing during startup.

## Q31. What is the difference between liveness and readiness?

Liveness determines whether a process should be restarted.

Readiness determines whether an instance should receive traffic.

## Q32. Why is readiness important in Kubernetes?

A newly started application may be alive but not yet capable of safely handling requests. Readiness prevents traffic from reaching it until it is ready.

## Q33. How would you design a Spring Boot Starter?

Use:

```text
Starter
 ↓
Dependencies
 ↓
Auto-Configuration
 ↓
Conditions
 ↓
Default Beans
 ↓
Back-Off
```

Also provide configuration properties and avoid hidden side effects.

## Q34. Why use `@ConditionalOnMissingBean` in a Starter?

It allows the application to replace the library's default implementation.

## Q35. Why should a Starter avoid network calls during startup?

Because the library should not introduce hidden availability dependencies into applications that consume it.

## Q36. Why is failure analysis important?

It converts low-level startup failures into actionable diagnostic information and reduces troubleshooting time.

## Q37. Why can adding a dependency affect startup?

A dependency can add classes, activate Auto-Configuration, create additional beans and increase startup work.

## Q38. What is self-invocation in Spring?

Self-invocation occurs when one method directly calls another method on the same target object. Because the call does not pass through the Spring proxy, proxy-based features such as transactions may not be applied as expected.

## Q39. Why should transactions not always cover huge operations?

Large transactions can cause long locks, large persistence contexts, memory growth, longer rollback and reduced concurrency.

## Q40. Why isn't increasing thread count always beneficial?

Because downstream resources such as databases, connection pools and external services may have lower capacity.

## Q41. What is graceful shutdown?

Graceful shutdown allows the application to stop accepting new work, complete appropriate existing work, release resources and then terminate.

---

# Senior Interview Tips

## Startup

Do not answer:

> Spring Boot creates beans and starts Tomcat.

Use:

```text
SpringApplication
 ↓
Environment
 ↓
ApplicationContext
 ↓
Configuration
 ↓
Auto-Configuration
 ↓
Conditions
 ↓
Bean Definitions
 ↓
Bean Creation
 ↓
Initialization
 ↓
Embedded Server
 ↓
Application Ready
```

---

## Auto-Configuration

A strong answer:

> Spring Boot discovers Auto-Configuration candidates and evaluates conditions based on the classpath, existing beans, properties and application type. Matching configurations contribute bean definitions while non-matching configurations are skipped.

---

## Back-Off

Always mention:

```java
@ConditionalOnMissingBean
```

Then:

```text
Boot Default
      ↓
Custom Application Bean
      ↓
Boot Backs Off
```

---

## Bean Lifecycle

Be precise:

```text
Instantiate
 ↓
Dependency Injection
 ↓
postProcessBeforeInitialization()
 ↓
@PostConstruct
 ↓
Initialization
 ↓
postProcessAfterInitialization()
 ↓
Final Bean / Proxy
```

Important:

> BeanPostProcessors are registered before normal beans are created, but their callbacks process individual beans around initialization.

---

## Startup Optimization

Never say:

> Enable lazy initialization.

Instead:

> First measure startup to identify whether the bottleneck is bean initialization, classpath scanning, Auto-Configuration, external calls or database initialization. Then optimize the actual bottleneck.

---

## Kubernetes

Remember:

```text
Liveness
    ↓
Should I restart?

Readiness
    ↓
Should I receive traffic?
```

---

## Transactions

Think:

```text
Business Consistency
        ↓
Transaction Boundary
```

not:

```text
Every Method
        ↓
@Transactional
```

---

## Concurrency

Always ask:

```text
How much can the downstream system handle?
```

before increasing application concurrency.

---

## Custom Starter

Your answer should include:

```text
@AutoConfiguration
+
Conditions
+
@ConfigurationProperties
+
@ConditionalOnMissingBean
+
Back-Off
+
Minimal Side Effects
```

And mention:

> The consuming application must remain in control.

---

## Production Architecture

Don't stop at:

> Spring Boot starts Tomcat.

Explain:

```text
Configuration
 ↓
ApplicationContext
 ↓
Bean Graph
 ↓
Infrastructure
 ↓
Readiness
 ↓
Traffic
 ↓
Observability
 ↓
Graceful Shutdown
```

---

# KRB Academy Interview Readiness

After completing Module 18, you should confidently explain:

- Why Spring Boot exists
- Declarative configuration
- `@SpringBootApplication`
- `@SpringBootConfiguration`
- `@ComponentScan`
- `@EnableAutoConfiguration`
- Conditional Configuration
- `@ConditionalOnClass`
- `@ConditionalOnBean`
- `@ConditionalOnMissingBean`
- `@ConditionalOnProperty`
- `@ConditionalOnWebApplication`
- Starter Dependencies
- Starter vs Auto-Configuration
- `SpringApplication`
- Application startup
- Application type detection
- Environment
- Property Sources
- Profiles
- Configuration Binding
- Binder
- `@ConfigurationProperties`
- Configuration Validation
- ApplicationContext
- BeanDefinition
- Bean Instance
- Bean Lifecycle
- BeanPostProcessor
- Constructor Injection
- Circular Dependencies
- BeanFactory vs ApplicationContext
- ApplicationContext refresh
- Embedded Tomcat
- DispatcherServlet relationship
- Application Events
- `ApplicationReadyEvent`
- Actuator
- Health
- Metrics
- Startup optimization
- Lazy Initialization
- Auto-Configuration discovery
- `AutoConfiguration.imports`
- Condition Evaluation
- Custom Auto-Configuration
- Custom Spring Boot Starters
- Failure Analysis
- Fail-Fast Startup
- Kubernetes Readiness
- Kubernetes Liveness
- Observability
- Configuration and Secrets
- AOP Proxies
- Self Invocation
- Transaction Boundaries
- Thread Pools
- Connection Pools
- Graceful Shutdown
- Production Spring Boot Architecture

---

# Module 18 - Final Takeaway

> **Spring Boot is not just a framework for starting a Spring application. It is a runtime orchestration layer that assembles configuration, infrastructure, beans, lifecycle management, observability and operational behavior into a production application.**

The senior engineer's mental model should be:

```text
Understand the defaults
        ↓
Understand the conditions
        ↓
Understand the lifecycle
        ↓
Measure the runtime
        ↓
Control the trade-offs
        ↓
Design for failure
        ↓
Design for production
```

The final production mental model is:

```text
Build
 ↓
Configure
 ↓
Start
 ↓
Validate
 ↓
Become Ready
 ↓
Serve
 ↓
Observe
 ↓
Scale
 ↓
Shutdown Gracefully
```

---