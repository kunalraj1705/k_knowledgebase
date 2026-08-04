# Module 14 - Spring MVC Internals

---

# Overview

Spring MVC is Spring Framework's web framework responsible for handling HTTP requests and generating HTTP responses. It follows the Front Controller design pattern by routing every incoming request through a single entry point called the DispatcherServlet.

Spring MVC separates HTTP infrastructure from business logic, allowing developers to focus only on implementing business functionality while the framework manages request routing, parameter binding, serialization, exception handling, and response generation.

Like every other Spring module, Spring MVC performs expensive work during application startup and keeps runtime request processing lightweight.

---

# Learning Objectives

After completing this module you will understand:

- Why Spring MVC exists
- Front Controller Pattern
- DispatcherServlet
- HandlerMapping
- HandlerAdapter
- Argument Resolution
- HttpMessageConverter
- Exception Resolution
- Spring MVC Startup Lifecycle
- Runtime Request Lifecycle
- Filter vs Interceptor vs AOP
- Request Mapping Registration
- Request Processing Pipeline
- Response Generation
- Spring MVC Architecture

---

# Problem Statement

Without Spring MVC, every Java web application would need to manually perform:

- Listen for HTTP requests
- Parse HTTP protocol
- Extract URL
- Determine HTTP Method
- Find correct controller
- Invoke business method
- Convert request body
- Serialize response
- Handle exceptions
- Generate HTTP response

Example

```java
ServerSocket server = new ServerSocket(8080);

while(true){

    Socket socket = server.accept();

    HttpRequest request = parse(socket);

    if(request.getPath().equals("/products")){

        // Find controller

        // Invoke method

        // Build response

    }

}
```

As the application grows, maintaining this infrastructure becomes difficult.

Spring MVC separates infrastructure from business logic.

---

# Declarative Programming

Developer Responsibility

- Implement business logic
- Define request mappings
- Return domain objects

Spring Responsibility

- Receive HTTP requests
- Route requests
- Parameter binding
- Request body conversion
- Response serialization
- Exception handling
- Request lifecycle management

Developer writes:

```java
@RestController
@RequestMapping("/products")
public class ProductController {

    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id){

        return service.getProduct(id);

    }

}
```

Spring manages everything else.

---

# Why Spring MVC Exists

Spring MVC removes HTTP infrastructure from application code.

Benefits

- Separation of Concerns
- Reduced Boilerplate
- Centralized Request Processing
- Better Maintainability
- Extensible Architecture
- High Performance Runtime

---

# Internal Architecture

```
Browser

↓

Tomcat

↓

Filter Chain

↓

DispatcherServlet

↓

HandlerMapping

↓

HandlerAdapter

↓

Argument Resolver

↓

HttpMessageConverter

↓

Controller

↓

Service

↓

Repository

↓

Database

↓

Controller Returns Object

↓

HttpMessageConverter

↓

Exception Resolver (if required)

↓

HTTP Response

↓

Browser
```

---

# Core Components

## Embedded Tomcat

Responsibilities

- Listen on HTTP Port
- Accept TCP Connections
- Receive HTTP Requests
- Forward requests to DispatcherServlet

---

## DispatcherServlet

Front Controller of Spring MVC.

Responsibilities

- Receive every request
- Delegate to HandlerMapping
- Delegate to HandlerAdapter
- Coordinate request lifecycle
- Return response

DispatcherServlet never contains business logic.

---

## HandlerMapping

Responsibilities

- Maintain request mapping registry
- Match incoming requests
- Return handler metadata

Mapping contains

- URL Pattern
- HTTP Method
- Controller Bean
- Method Reference
- Parameter Metadata
- Return Type
- Consumes
- Produces

---

## HandlerAdapter

Responsibilities

- Invoke controller
- Coordinate parameter resolution
- Coordinate request body conversion
- Coordinate response generation

DispatcherServlet delegates controller invocation to HandlerAdapter.

---

## Argument Resolver

Responsible for resolving

- @PathVariable
- @RequestParam
- @RequestHeader
- @CookieValue
- Other controller parameters

Controller receives fully prepared Java objects.

---

## HttpMessageConverter

Responsible for

- JSON → Java Object
- XML → Java Object
- Java Object → JSON
- Java Object → XML

Spring automatically selects the correct converter based on Content-Type and Accept headers.

---

## Exception Resolver

Responsible for centralized exception handling.

Works together with

- @ControllerAdvice
- @ExceptionHandler

Controllers only throw exceptions.

Spring creates the HTTP error response.

---

# Startup Lifecycle

## Step 1

Application starts.

```java
SpringApplication.run(...)
```

---

## Step 2

Spring Boot processes

```
@SpringBootApplication
```

which imports

```
@EnableAutoConfiguration
```

---

## Step 3

Spring Boot registers MVC infrastructure.

Examples

- DispatcherServlet
- HandlerMapping
- HandlerAdapter
- ArgumentResolvers
- HttpMessageConverters
- ExceptionResolvers

Infrastructure always exists before application metadata is discovered.

---

## Step 4

Component Scanning begins.

Spring discovers

- @RestController
- @Controller
- @Service
- @Repository

BeanDefinitions are created.

---

## Step 5

Beans are instantiated using Reflection.

Dependency Injection occurs.

@PostConstruct executes.

---

## Step 6

BeanPostProcessor executes.

Spring discovers

- @RequestMapping
- @GetMapping
- @PostMapping
- @PutMapping
- @DeleteMapping

Spring extracts metadata and builds the request mapping registry.

Metadata includes

- URL
- HTTP Method
- Controller Bean
- Method Reference
- Parameters
- Return Type
- Consumes
- Produces

---

## Step 7

Application becomes ready.

All mappings have already been registered.

No runtime scanning is required.

---

# Runtime Request Flow

```
Browser

↓

Tomcat

↓

Filter Chain

↓

DispatcherServlet

↓

HandlerMapping

↓

Find Matching Handler

↓

HandlerAdapter

↓

Argument Resolver

↓

HttpMessageConverter

↓

Controller

↓

Service

↓

Repository

↓

Database

↓

Return Java Object

↓

HttpMessageConverter

↓

JSON Response

↓

Browser
```

---

# Request Mapping Registration

During startup Spring builds a routing registry.

Conceptually

```
GET /products/{id}

↓

ProductController#getProduct()

----------------------------

POST /products

↓

ProductController#createProduct()
```

Runtime becomes a direct lookup.

No annotation scanning occurs after startup.

---

# Parameter Resolution

Incoming Request

```
GET /products/10
```

Controller

```java
public Product getProduct(@PathVariable Long id)
```

Spring performs

```
Extract Path Variable

↓

Convert String → Long

↓

Invoke Controller
```

Controllers never parse HTTP requests.

---

# Request Body Conversion

Incoming Request

```json
{
    "id":10,
    "name":"Toothbrush"
}
```

Spring performs

```
JSON

↓

HttpMessageConverter

↓

Product Object

↓

Controller
```

---

# Response Generation

Controller returns

```java
return product;
```

Spring performs

```
Product

↓

HttpMessageConverter

↓

JSON

↓

HTTP Response
```

Controllers never serialize JSON.

---

# Exception Handling

Controller

```
throw new ProductNotFoundException();
```

Flow

```
Controller

↓

Exception Resolver

↓

@ControllerAdvice

↓

HTTP Response
```

Exception handling is centralized.

---

# Filter vs Interceptor vs AOP

## Filter

Layer

Servlet Container

Position

```
Browser

↓

Tomcat

↓

Filter
```

Responsibilities

- CORS
- Compression
- Character Encoding
- Logging
- Security Filter Chain
- Rate Limiting

Filter does not know controller information.

---

## Interceptor

Layer

Spring MVC

Position

```
DispatcherServlet

↓

HandlerMapping

↓

Interceptor

↓

Controller
```

Responsibilities

- Authorization
- Controller Logging
- Execution Timing
- Audit Logging
- Locale Resolution

Interceptor knows controller metadata.

---

## AOP

Layer

Business Layer

Position

```
Controller

↓

Service

↓

Proxy

↓

Business Method
```

Responsibilities

- Transactions
- Caching
- Async
- Logging
- Cross Cutting Concerns

AOP is independent of HTTP.

---

# Startup vs Runtime

Startup

- Infrastructure Registration
- Controller Discovery
- Mapping Registration
- Metadata Extraction
- Bean Creation

Runtime

- Mapping Lookup
- Parameter Resolution
- Controller Invocation
- Response Serialization

Spring always performs expensive work during startup.

---

# Performance Optimizations

Spring performs

- Annotation Scanning
- Reflection Discovery
- Metadata Extraction
- Route Registration

during startup.

Runtime performs only

```
Lookup

↓

Invoke

↓

Return
```

---

# Complete Spring MVC Lifecycle

```
Application Starts

↓

SpringApplication

↓

@EnableAutoConfiguration

↓

Register MVC Infrastructure

↓

Component Scan

↓

Bean Definition Creation

↓

Bean Creation

↓

@PostConstruct

↓

BeanPostProcessor

↓

Discover Controllers

↓

Build Mapping Registry

↓

Application Ready

──────────────────────────────

Browser

↓

Tomcat

↓

Filter

↓

DispatcherServlet

↓

HandlerMapping

↓

HandlerAdapter

↓

Argument Resolver

↓

HttpMessageConverter
(Request → Java Object)

↓

Controller

↓

Service

↓

Repository

↓

Database

↓

Return Java Object

↓

HttpMessageConverter
(Java Object → JSON)

↓

Exception Resolver

↓

Browser
```

---

# Common Interview Questions

- Why is DispatcherServlet called the Front Controller?
- What is the responsibility of HandlerMapping?
- Why does Spring use HandlerAdapter?
- How does @PathVariable work internally?
- How is @RequestBody converted into a Java object?
- What is HttpMessageConverter?
- How does Spring choose a MessageConverter?
- Difference between Filter, Interceptor and AOP?
- Why are request mappings built during startup?
- How does @ControllerAdvice work?
- Explain the Spring MVC request lifecycle.

---

# Engineering Principles

## Principle 1

Separate HTTP infrastructure from business logic.

---

## Principle 2

Infrastructure should exist before metadata registration.

---

## Principle 3

Perform expensive discovery during startup.

---

## Principle 4

Runtime should rely on precomputed metadata.

---

## Principle 5

Delegate specialized responsibilities to dedicated components.

---

## Principle 6

Controllers should operate on domain objects, not HTTP details.

---

## Principle 7

Cross-cutting concerns belong to the appropriate layer:

- Filter → Servlet Layer
- Interceptor → MVC Layer
- AOP → Business Layer

---

## Principle 8

Frameworks should prepare execution context before invoking business code.

---

# Key Takeaways

- Spring MVC follows the Front Controller pattern using DispatcherServlet.
- Spring registers MVC infrastructure during application startup.
- Controller mappings are discovered once and stored in a routing registry.
- HandlerMapping performs fast runtime lookups instead of repeated annotation scanning.
- HandlerAdapter coordinates controller invocation.
- ArgumentResolvers prepare controller method parameters.
- HttpMessageConverter converts request bodies to Java objects and Java objects to HTTP responses.
- Exception handling is centralized using ExceptionResolver and `@ControllerAdvice`.
- Filters, Interceptors, and AOP operate at different architectural layers and solve different problems.
- Spring MVC follows the same architectural philosophy as Transactions, AOP, Events, Async, Scheduling, and Caching: **Infrastructure Registration → Metadata Discovery → Runtime Lookup → Business Execution.**