# The `getBean()` Lifecycle

## Purpose

`getBean()` is the core lookup workflow used when Spring needs a managed bean. At startup, an `ApplicationContext` eagerly creates most non-lazy singleton beans through the same underlying creation path.

## High-level flow

```text
context.getBean(UserService.class)
  |
  v
Check existing singleton cache
  |
  +-- found --> return it
  |
  `-- not found
        |
        v
      Obtain BeanDefinition
        |
        v
      Resolve dependencies
        |
        v
      Instantiate bean
        |
        v
      Populate properties
        |
        v
      Initialize and post-process bean
        |
        `-- cache if singleton, then return
```

## Creation phases

1. **Resolve the definition and scope.** Spring finds the bean definition and determines whether it is singleton, prototype, or another scope.
2. **Resolve dependencies.** It selects a constructor and obtains its arguments recursively.
3. **Instantiate.** Spring invokes the selected constructor, usually through reflection.
4. **Populate properties.** Field and setter injection, if used, occur after instantiation.
5. **Initialize.** Aware callbacks, `BeanPostProcessor`s, `@PostConstruct`, `InitializingBean`, and custom init methods run in the relevant lifecycle phases. A post-processor may wrap the bean in a proxy.
6. **Expose the result.** Spring caches a singleton or returns a fresh prototype instance.

## Recursive dependency resolution

```text
getBean(UserService)
  -> obtain UserRepository
  -> select NotificationService implementation
     -> apply @Qualifier / @Primary when needed
  -> construct and initialize UserService
```

Spring does not construct `UserService` until its required constructor arguments are available. Constructor cycles therefore cannot be resolved automatically; field or setter cycles have limited support in some configurations, but should generally be redesigned.

## Singleton caching and circular references

Spring uses several singleton caches while creating beans. These caches can expose an early reference for certain singleton setter/field circular dependencies, including cases involving proxies. They do not make circular dependencies safe or solve constructor injection cycles.

## Key takeaway

`getBean()` is not simply object construction: it combines definition lookup, dependency resolution, lifecycle callbacks, post-processing, scopes, and caching.
