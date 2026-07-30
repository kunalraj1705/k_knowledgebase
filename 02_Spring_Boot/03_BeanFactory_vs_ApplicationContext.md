# BeanFactory vs ApplicationContext

## Purpose

`BeanFactory` is Spring's core IoC-container interface. `ApplicationContext` extends it with application-oriented services and is the normal entry point for Spring and Spring Boot applications.

```text
ApplicationContext
  |
  +-- BeanFactory operations: lookup, creation, scopes, lifecycle
  |
  `-- application services: events, resources, environment, i18n
```

## Responsibilities

| Type | Main responsibility |
| --- | --- |
| `BeanFactory` | Defines bean lookup and basic container operations such as `getBean()`, `containsBean()`, and `isSingleton()`. |
| `ApplicationContext` | Adds event publishing, resource loading, message resolution, environment/property integration, and context lifecycle support. |
| `DefaultListableBeanFactory` | A common concrete factory that holds bean definitions and performs candidate resolution. |

An `ApplicationContext` does not merely delegate every operation to an unrelated factory; typical contexts contain or inherit a `DefaultListableBeanFactory` as their internal bean factory.

## Bean retrieval

```text
getBean(name or type)
  |
  +-- existing singleton? --> return cached instance
  |
  `-- otherwise
        |
        v
      read BeanDefinition
        |
        v
      resolve constructor arguments and dependencies
        |
        v
      create, populate, and initialize bean
        |
        `-- cache it when the scope is singleton
```

Dependencies are resolved recursively. If `UserService` needs a `UserRepository` and `NotificationService`, Spring obtains or creates those dependencies before it can construct `UserService`.

## Internal factory layers

```text
BeanFactory (interface)
  |
  v
AbstractBeanFactory
  |
  v
AbstractAutowireCapableBeanFactory
  |
  v
DefaultListableBeanFactory
```

- `AbstractBeanFactory` coordinates bean retrieval, scopes, and singleton caching.
- `AbstractAutowireCapableBeanFactory` implements creation: constructor resolution, instantiation, property population, initialization, and `BeanPostProcessor` application.
- `DefaultListableBeanFactory` adds bean-definition registration and type-based candidate lookup.

## Constructor selection and candidates

Spring normally uses the sole constructor of a class. With multiple constructors, it uses one marked for autowiring or applies its constructor-resolution rules. Candidate selection begins with type, then considers qualifiers, primary status, names, and other autowire-candidate rules.

## Key takeaway

Use `ApplicationContext` in application code. Understand `BeanFactory` to understand the container mechanics underneath it.
