# Module 15 - Spring Security Internals

---

# Overview

Spring Security is Spring Framework's security infrastructure responsible for authentication, authorization, request protection, and security context management.

Spring Security separates security concerns from business logic by intercepting HTTP requests before they reach Spring MVC. It provides a centralized security architecture that manages authentication, authorization, session handling, exception handling, and request protection.

Like every Spring module, Spring Security performs expensive initialization during application startup and keeps runtime request processing lightweight through precomputed metadata and specialized infrastructure.

---

# Learning Objectives

After completing this module you will understand:

- Why Spring Security exists
- Authentication vs Authorization
- Security Filter Chain
- DelegatingFilterProxy
- FilterChainProxy
- AuthenticationManager
- AuthenticationProvider
- UserDetailsService
- PasswordEncoder
- SecurityContext
- SecurityContextHolder
- ThreadLocal
- JWT Authentication Lifecycle
- Method Security
- Complete Security Request Lifecycle
- Spring Security Architecture

---

# Problem Statement

Without Spring Security every controller would need to implement security manually.

Example

```java
@GetMapping("/accounts/{id}")
public Account getAccount(Long id){

    validateJWT();

    authenticateUser();

    authorizeUser();

    checkRole();

    return accountService.getAccount(id);

}
```

Imagine

- 500 Controllers
- 3000 APIs

Every controller would repeat

- JWT Validation
- Authentication
- Authorization
- Role Verification
- Security Context Creation

Problems

- Duplicate Code
- Poor Maintainability
- Security Bugs
- Business Logic mixed with Security
- Difficult to Extend

Spring Security solves this by centralizing all security responsibilities.

---

# Declarative Programming

Developer writes

```java
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/users")
public List<User> getUsers(){

    return service.getUsers();

}
```

Spring manages

- Authentication
- Authorization
- JWT Validation
- Security Context
- Exception Handling
- Filter Execution

Business code remains security independent.

---

# Authentication vs Authorization

## Authentication

Question

Who are you?

Examples

- Username & Password
- JWT
- OAuth2
- LDAP

Output

Authenticated User

---

## Authorization

Question

What are you allowed to access?

Examples

- ADMIN
- USER
- MANAGER

Authorization happens only after successful authentication.

---

# Why Spring Security Uses Filters

Request Flow

```
Browser

↓

Tomcat

↓

Security Filter Chain

↓

DispatcherServlet

↓

Controller
```

Security executes before Spring MVC.

Reasons

- Reject invalid requests early
- Save CPU
- Avoid unnecessary routing
- Avoid controller execution
- Protect every incoming HTTP request

Authentication failure

```
401 Unauthorized
```

Authorization failure

```
403 Forbidden
```

---

# Security Architecture

```
Browser

↓

Tomcat

↓

DelegatingFilterProxy

↓

FilterChainProxy

↓

SecurityFilterChain

↓

DispatcherServlet

↓

Spring MVC
```

---

# DelegatingFilterProxy

Purpose

Bridge between

- Tomcat
- Spring IoC Container

Responsibilities

- Registered with Tomcat
- Obtain Spring managed security filter
- Delegate request

It never performs authentication.

---

# FilterChainProxy

Responsibilities

- Maintain SecurityFilterChains
- Select matching chain
- Execute filters sequentially

Acts as the orchestrator of Spring Security.

---

# Why Multiple Filters

Instead of

```
SecurityFilter
```

Spring creates

```
Authentication Filter

↓

Authorization Filter

↓

Exception Filter

↓

Anonymous Filter

↓

Logout Filter
```

Benefits

- Single Responsibility Principle
- Open Closed Principle
- Easy Extension
- Easy Testing
- Easy Maintenance

---

# Multiple SecurityFilterChains

Applications may expose

```
/public/**

/user/**

/admin/**
```

Different URLs require different security pipelines.

Conceptually

```
Incoming Request

↓

FilterChainProxy

↓

Select Matching SecurityFilterChain

↓

Execute Required Filters
```

Example

```
/public/**

↓

Permit All

------------------------

/user/**

↓

JWT Authentication

------------------------

/admin/**

↓

JWT Authentication

↓

Role Verification
```

Only the required chain executes.

---

# SecurityContext

After successful authentication Spring creates

```
Authentication

↓

Principal

↓

Authorities

↓

Authenticated = true
```

This Authentication object is stored inside

```
SecurityContext
```

---

# SecurityContextHolder

SecurityContext is stored inside

```
SecurityContextHolder
```

Uses

```
ThreadLocal
```

Reason

Every HTTP request is processed by one Tomcat worker thread.

All components executing on the same thread can access the authenticated user without passing user information through method parameters.

---

# Request Thread Lifecycle

```
Tomcat

↓

Thread T1

↓

Authenticate User

↓

Create SecurityContext

↓

Store in ThreadLocal

↓

Process Request

↓

Clear SecurityContext

↓

Return Thread to Pool
```

Spring clears the context before thread reuse to prevent security leaks.

---

# Authentication Pipeline

```
Authentication Filter

↓

AuthenticationManager

↓

AuthenticationProvider

↓

UserDetailsService

↓

PasswordEncoder

↓

Authentication

↓

SecurityContext
```

Each component has one responsibility.

---

# Authentication Filter

Responsibilities

- Read Credentials
- Extract JWT
- Create Authentication Token
- Delegate Authentication

Never queries database.

---

# AuthenticationManager

Responsibilities

- Receive authentication request
- Select AuthenticationProvider
- Delegate authentication

Acts as an orchestrator.

---

# AuthenticationProvider

Responsibilities

- Coordinate authentication
- Validate credentials
- Call UserDetailsService
- Call PasswordEncoder
- Create Authentication object

Never directly manages SecurityContext.

---

# UserDetailsService

Responsibilities

- Load user information
- Return UserDetails

Only responsible for retrieving user data.

---

# PasswordEncoder

Responsibilities

- Compare passwords
- Verify password hash

Supports

- BCrypt
- Argon2
- PBKDF2
- SCrypt

AuthenticationProvider delegates password verification.

---

# JWT Authentication Lifecycle

Incoming Request

```http
Authorization: Bearer eyJhbGc...
```

Flow

```
Tomcat

↓

Thread T1

↓

DelegatingFilterProxy

↓

FilterChainProxy

↓

BearerTokenAuthenticationFilter

↓

Extract JWT

↓

Validate Signature

↓

Validate Expiration

↓

AuthenticationManager

↓

AuthenticationProvider

↓

UserDetailsService

↓

Load User

↓

Create Authentication

↓

SecurityContext

↓

SecurityContextHolder

↓

DispatcherServlet
```

JWT is validated only once per request.

---

# Method Security

Spring also supports method level authorization.

Example

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(){

}
```

Unlike URL security, method security is implemented using Spring AOP.

Flow

```
Controller

↓

Service Proxy

↓

Authorization Check

↓

Business Method
```

---

# URL Security vs Method Security

## URL Security

Layer

Security Filter Chain

Protects

HTTP Requests

Runs

Before DispatcherServlet

---

## Method Security

Layer

Spring AOP

Protects

Business Methods

Runs

Around Method Invocation

---

# Complete Security Request Lifecycle

```
Browser

↓

Tomcat

↓

Assign Thread T1

↓

DelegatingFilterProxy

↓

FilterChainProxy

↓

Select SecurityFilterChain

↓

Authentication Filter

↓

Extract JWT

↓

Validate Token

↓

AuthenticationManager

↓

AuthenticationProvider

↓

UserDetailsService

↓

Create Authentication

↓

SecurityContextHolder (ThreadLocal)

↓

DispatcherServlet

↓

HandlerMapping

↓

HandlerAdapter

↓

Controller

↓

@Transactional Proxy (if present)

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

SecurityContextHolder.clearContext()

↓

Tomcat

↓

Browser
```

---

# Startup Lifecycle

## Step 1

Application starts.

```
SpringApplication.run(...)
```

---

## Step 2

Spring Boot imports Security AutoConfiguration.

Infrastructure beans are registered.

Examples

- DelegatingFilterProxy
- FilterChainProxy
- AuthenticationManager
- AuthenticationProviders
- PasswordEncoder
- SecurityContext Infrastructure

---

## Step 3

Component Scanning begins.

Security beans are discovered.

---

## Step 4

Bean Creation

Spring creates

- Filters
- Providers
- Managers
- Services

Dependency Injection occurs.

---

## Step 5

SecurityFilterChains are created from application configuration.

Request matchers

Roles

Authentication rules

Authorization rules

are registered.

---

## Step 6

Application Ready.

Security infrastructure is fully initialized.

Runtime only performs lookups.

---

# Runtime Flow

```
Incoming Request

↓

Select SecurityFilterChain

↓

Authenticate

↓

Authorize

↓

Create SecurityContext

↓

DispatcherServlet

↓

Business Execution

↓

Clear SecurityContext

↓

Return Response
```

---

# Startup vs Runtime

Startup

- Register Infrastructure
- Create Filters
- Build SecurityFilterChains
- Register Authentication Providers
- Configure AuthenticationManager

Runtime

- Match Filter Chain
- Authenticate
- Authorize
- Store SecurityContext
- Execute Business Logic
- Clear Context

---

# Performance Optimizations

Spring performs

- Filter Registration
- Authentication Configuration
- Security Chain Creation
- Infrastructure Initialization

during startup.

Runtime performs

```
Lookup

↓

Authenticate

↓

Execute

↓

Return
```

---

# Engineering Principles

## Principle 1

Reject invalid requests as early as possible.

---

## Principle 2

Separate security from business logic.

---

## Principle 3

Authentication and Authorization are different responsibilities.

---

## Principle 4

One component should own one responsibility.

---

## Principle 5

Use delegation instead of monolithic implementations.

---

## Principle 6

Perform expensive work once and reuse the result.

---

## Principle 7

Store request-scoped security information using ThreadLocal.

---

## Principle 8

Infrastructure first.

Metadata later.

Runtime lookup finally.

---

## Principle 9

Protect HTTP requests using Filters.

Protect business methods using AOP.

---

# Common Interview Questions

- Why does Spring Security use Filters instead of Interceptors?
- What is DelegatingFilterProxy?
- What is FilterChainProxy?
- Difference between Authentication and Authorization?
- What is AuthenticationManager?
- What is AuthenticationProvider?
- What is UserDetailsService?
- Why is PasswordEncoder separated?
- What is SecurityContextHolder?
- Why does Spring Security use ThreadLocal?
- Explain JWT Authentication Flow.
- Difference between URL Security and Method Security.
- Explain complete Spring Security request lifecycle.

---

# Key Takeaways

- Spring Security separates security concerns from business logic.
- Every request enters the Security Filter Chain before reaching Spring MVC.
- DelegatingFilterProxy bridges Tomcat and the Spring IoC Container.
- FilterChainProxy selects and executes the appropriate SecurityFilterChain.
- AuthenticationManager orchestrates authentication by delegating to AuthenticationProviders.
- UserDetailsService loads user information, while PasswordEncoder verifies credentials.
- SecurityContextHolder stores the authenticated user in ThreadLocal for the lifetime of the request.
- URL security is implemented using Filters, while method security is implemented using AOP.
- Spring Security follows the same architectural philosophy as the rest of Spring: **Infrastructure Registration → Metadata Configuration → Runtime Lookup → Business Execution.**