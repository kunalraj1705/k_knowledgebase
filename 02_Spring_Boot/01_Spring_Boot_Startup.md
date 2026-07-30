# Spring Boot Startup

## Purpose

Spring Boot reduces manual Spring configuration by using conventions and auto-configuration. Adding a starter dependency makes its libraries available on the runtime classpath; during startup, Boot evaluates conditions and configures compatible infrastructure. It does not read `pom.xml` at runtime.

## Key concepts

- `@SpringBootApplication`
- `SpringApplication`
- `ApplicationContext`
- component scanning
- bean definitions and beans
- auto-configuration
- dependency injection

## What `@SpringBootApplication` means

`@SpringBootApplication` combines:

1. `@SpringBootConfiguration`, a Boot-specific form of `@Configuration`.
2. `@EnableAutoConfiguration`, which imports conditionally selected configuration.
3. `@ComponentScan`, which discovers components in the application's package and its subpackages by default.

The expression `Application.class` is a Java `Class` object, not an application instance. Spring uses reflection and metadata reading to inspect that class and its annotations.

## Startup flow

```text
JVM starts main()
  |
  v
SpringApplication.run(Application.class, args)
  |
  v
Prepare environment and choose an ApplicationContext
  |
  v
Load configuration classes and component-scan candidates
  |
  v
Register BeanDefinitions
  |
  v
Apply conditional auto-configuration
  |
  v
Refresh ApplicationContext
  |
  v
Create and initialize required singleton beans
  |
  v
Start embedded web server, when this is a web application
  |
  v
Publish readiness events and return the context
```

During context refresh, Spring resolves dependencies, creates beans, invokes lifecycle callbacks, and applies `BeanPostProcessor`s. Constructor injection is usually preferred because a bean cannot be instantiated until its required dependencies are available.

## Important distinctions

- Component scanning discovers candidate classes and registers bean definitions; it does not by itself create every bean.
- Auto-configuration contributes configuration when its conditions match; it does not blindly configure every library on the classpath.
- `ApplicationContext` is the high-level IoC container API. It manages beans and adds features such as events, resources, environment support, and internationalization.
- Circular constructor dependencies typically fail with a circular-reference error; `BeanCurrentlyInCreationException` is one error Spring can report in such cases.

## Common mistakes

- Thinking Spring parses Maven's `pom.xml` at runtime.
- Treating annotations as code that executes on their own.
- Equating component scanning with bean creation.
- Assuming every dependency on the classpath is automatically configured without conditions.

## Mental model

Boot converts application code, configuration, and classpath libraries into bean definitions. Refreshing the `ApplicationContext` turns those definitions into a running application.
