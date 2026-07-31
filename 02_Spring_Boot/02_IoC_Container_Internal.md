# IoC Container Internals

## Why does this exist?

Spring application can contain hundred or even thousands of objects. Instead of developers creating this manually using new keyword and wiring these object we give this job to Spring's IoC Container. This container is responsible for creating, managing and injecting dependencies between them.

The IoC container also ensures that bean lifecycle. scope and dependency resolution are handled consistently throughout application .

## Key Concepts

- `ApplicationContext`
- `BeanFactory`
- `BeanDefinition`
- `Singleton Bean Registry`
- `Bean Scope`
- `Dependency Resolution`
- `Dependency Injection`

## Bean Management Flow

1. Spring Scans the application packages.
2. Component Scanning discovers class annotated with @component, @Service, @Controller, @Repository....
3. Spring creates a bean definition for every discovers bean.
4. BeanDefinitions are registered inside the BeanFactory
5. Spring resolves the dependency graph
6. Beans are instantiated in an order that satisfies all dependencies.
7. Singleton Beans are stored inside the singleton bean registry.
8. Dependencies are injected.
9. The fully initialized bean is returned when requested.

## Bean Definition

A bean definition is the metadata about a bean.

It is **not** the actual object.

A bean definition contains information such as:

- Bean Class
- Scope (Singleton / Prototype)
- Constructor Information
- Dependencies
- Lazy initialization
- Primary Bean
- Qualifier

A bean definition acts as a blueprint that Spring uses whenever a bean needs to be created.

---

## Singleton Bean Registry

Singleton beans are created only once.

After creation, spring stores them in Singleton Bean Registry.

Whenever another bean requests the same dependency, Spring returns the existing instance instead of creating a new one.

---

## Prototype Beans

Prototype beans are handled differently.

Spring does not store prototype bean instance inside the Singleton Registry

Instead:

1. Read the Bean Definition
2. Create new Object
3. Inject Dependencies
4. Return the object

Once the Spring returns the object. It no longer manages the lifecycle. When objects become unreachable, JVM reclaims the memory.

---

## Dependency Resolution

Before creating a bean, Spring examines its constructor dependencies.

Example:

```text
UserController
  ↓
Depends on
  ↓
UserService
  ↓
Depends on
  ↓
UserRepository
```

Spring creates the bean in dependency order:

1. `UserRepository`
2. `UserService`
3. `UserController`

This ensures every dependency is available before injection.

---

## Multiple Implementations

If multiple beans implement the same interface:

```text
NotificationService
  │
  ├── EmailService
  ├── SMSService
  └── PushNotificationService
```

Spring follows this process:

1. Find all matching beans.
2. If only one exists, inject it.
3. If multiple exist, look for `@Primary`.
4. If `@Qualifier` is specified, use the matching bean.
5. If ambiguity still exists, throw `NoUniqueBeanDefinitionException`.

---

## Singleton vs Prototype

Singleton

* One instance per application
* Stores in singleton Registry
* Reuses throughout application

Prototype

* New instance created every request
* Not stored after creation
* Lifecycle after creation is managed by JVM

---

## Important Learnings

* BeanDefinition is metadata, not the bean itself
* Spring separates bean metadata from bean instance
* Singleton bean are cached
* Prototype beans are created on demand.
* Dependency resolution occurs before bean creation.
* JVM garbage collector manages the prototype bean memory after spring returns the object

---

## Common mistakes

* Thinking bean definition is the actual bean
* Assuming Spring stores every bean instance forever
* Confusing application context with Bean Factory.
* Believing spring manually deletes prototype beans.

---

## Interview Questions

* What is BeanDefinition?
* Why does Spring uses BeanDefinition?
* How are Singleton bean stored?
* How are prototype bean managed?
* What happens when multiple implementation of an interface exist?
* What is the difference between BeanFactory and ApplicationContext?

---

## Engineering Principle

Separate metadata from Runtime state

Framework should store bean description of objects independently from object themselves.

---

## My Mental Model

Component Scan

↓

Bean Definitions

↓

BeanFactory

↓

Dependency Resolution

↓

Bean Creation

↓

Singleton?

↓

YES → Store in Singleton Registry

↓

NO → Return New Instance

↓

Dependency Injection

↓

ApplicationContext Ready

---

## Things I Don't Yet Understand

* How does BeanFactory create beans internally?
* What data structures does Spring actually use?
* How does ApplicationContext extend BeanFactory?
* How does Spring instantiate beans using Reflection?
* How are lazy beans created?
* How does Spring manage bean lifecycle callbacks?
