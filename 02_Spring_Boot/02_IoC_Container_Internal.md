# IoC Container Internals

## Purpose

An application can contain hundreds of collaborating objects. Instead of creating and wiring each one with `new`, Spring's Inversion of Control (IoC) container creates, configures, and manages them as beans.

## Core concepts

- **BeanDefinition**: metadata describing how to create a bean; it is not the bean instance.
- **BeanFactory**: the core bean-management contract.
- **ApplicationContext**: a BeanFactory with additional application-level features.
- **Singleton registry**: caches Spring singleton instances.
- **Scope**: controls how instances are created and shared.
- **Dependency resolution**: finds a bean that satisfies an injection point.

## Bean management flow

```text
Configuration and component scanning
  |
  v
Register BeanDefinitions
  |
  v
Resolve dependencies and candidates
  |
  v
Instantiate and populate beans
  |
  v
Initialize beans and apply post-processors
  |
  v
Cache singleton beans / return prototype beans
```

A `BeanDefinition` can contain the bean class, scope, constructor arguments, property values, autowiring metadata, qualifiers, primary status, and lazy-init setting.

## Scopes

### Singleton

`singleton` is Spring's default scope: one shared instance per bean definition per container. Spring caches the fully created singleton and returns it for later lookups.

### Prototype

For `prototype` scope, Spring creates a new instance for each request. Spring performs initialization but does not manage destruction callbacks after handing the instance to the caller. The caller owns cleanup; garbage collection only reclaims an object when it is unreachable.

## Dependency resolution

For constructor injection, Spring resolves dependencies before constructing the dependent bean:

```text
UserController
  -> UserService
       -> UserRepository
```

Spring creates `UserRepository` before `UserService`, then constructs `UserController` with the ready `UserService`.

When several beans implement the required type, Spring uses the injection point's type and then applies `@Qualifier`, `@Primary`, bean-name fallback where applicable, and other candidate rules. If no unique candidate remains, it throws `NoUniqueBeanDefinitionException`.

## Common mistakes

- Treating a bean definition as the object itself.
- Assuming Spring stores every bean forever; prototype instances are not cached as singletons.
- Saying “singleton means one instance in the JVM”; it means one instance per Spring container and bean definition.
- Assuming the JVM runs Spring destruction callbacks for prototype beans.

## Mental model

Spring separates *metadata* (bean definitions) from *runtime state* (bean instances). That separation lets it resolve dependencies, apply scopes, and control lifecycle consistently.
