# Java Reflection and Dynamic Bean Creation

## Purpose

Spring is built before it knows an application's classes. Java reflection lets the container inspect application types and invoke constructors or methods at runtime, while the container decides *which* objects to create and how to wire them.

## From source code to a bean

```text
UserService.java
  |
  v
Java compiler produces bytecode
  |
  v
Class loader loads UserService
  |
  v
JVM represents it as Class<UserService>
  |
  v
Spring registers a BeanDefinition
  |
  v
BeanFactory resolves dependencies and uses reflection
  |
  v
Initialized bean instance
```

`Class<?>` exposes metadata such as constructors, fields, methods, interfaces, and annotations. Spring may keep a class reference or class name in a bean definition, depending on how the definition was registered.

## Responsibilities

| Concern | Responsibility |
| --- | --- |
| JVM and class loaders | Load classes and make runtime type metadata available. |
| BeanFactory | Choose the bean, scope, constructor, dependency candidates, and lifecycle path. |
| Reflection | Inspect members and invoke constructors, methods, or fields selected by the container. |

Reflection is an execution mechanism, not a dependency-resolution engine.

## Dynamic creation flow

```java
public UserService(UserRepository repository, EmailService emailService) {
    // ...
}
```

To create `UserService`, Spring:

1. Finds its bean definition.
2. Selects the appropriate constructor.
3. Resolves `UserRepository` and `EmailService`, recursively if necessary.
4. Invokes the selected constructor with those arguments.
5. Populates any remaining properties and performs bean initialization.

Conceptually, the invocation resembles:

```java
constructor.newInstance(repository, emailService);
```

Spring has optimized paths and uses more than one metadata strategy internally, but this is the essential model.

## Key distinctions

- Loading a class does not create a bean.
- Instantiating an object does not complete the bean lifecycle.
- Reflection does not decide which implementation to inject; Spring's dependency-resolution logic does.
- A bean can also be produced by a `@Bean` factory method, a `FactoryBean`, or a supplier rather than directly by a component class constructor.

## Common mistakes

- Saying that Spring scans annotations only after every class is fully loaded. Spring can read class metadata without eagerly loading every candidate class.
- Treating reflection as the source of Spring's dependency-injection decisions.
- Assuming all beans are created directly from a no-argument constructor.

## Mental model

The container is the planner; reflection is one of the tools it uses to perform the plan.
