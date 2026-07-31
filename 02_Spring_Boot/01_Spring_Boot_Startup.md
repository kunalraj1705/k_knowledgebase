# Spring Boot Startup

## Why does this exist?

Spring Framework required extensive manual configuration (XML and later Java Configuration). Spring Boot reduces this boilerplate by providing sensible defaults and automatic configuration while still allowing customization when needed.

This will auto-add the configuration when it finds the dependencies in the classpath. For example, if you add Tomcat or a database dependency, Spring Boot detects it during startup and automatically configures the related infrastructure.

## Key Concepts

- `@SpringBootApplication`
- `@EnableAutoConfiguration`
- `@ComponentScan`
- `ApplicationContext`
- `Bean`
- `IoC Container`
- `Dependency Injection`

## Startup Flow

Just like a Java program, the application class contains the `main()` method. The `main()` method is the entry point of any Java application. Inside it, we call `SpringApplication.run(ApplicationClassName.class)`. Internally, `SpringApplication.run()` creates and configures a `SpringApplication` instance using the `ApplicationClass.class` as its primary source.

The argument is a `Class` object. Spring then inspects that `Class` object using Java Reflection to discover annotations such as `@SpringBootApplication`. Once found, it continues the workflow defined for that annotation. This annotation has three sub-annotations:

1: @SpringBootConfiguration: Marks this class as a Spring configuration class. It is a specialized form of @Configuration and serves as the primary source of bean definitions.

2: @EnableAutoConfiguration: This will add the required components for dependencies found in the class path.

3: @ComponentScan: This will find all the class with annotation like: @Controller @Service @Repository @Component and stores the Bean Definition inside the BeanFactory.

After the bean definition is created it will start creating the bean. Spring resolves the dependency graph and creates beans in an order that satisfies all required dependencies. The ApplicationContext stores, manages, initializes, and provides beans throughout the application's lifecycle. IoC container creates bean and injects their required dependencies during bean initialization. There are two type of Injection Via Constructor, via Field. DI via constructor is preferred because before starting the Application it resolves all the dependency. If everything is resolved then it will start the application else there the application will not start and fail with exception BeanCurrentlyInCreationException.

## Important Learning

- Spring uses Java Reflection to inspect annotations.
- Component scanning discovers bean definitions.
- Auto-configuration creates the infrastructure beans.
- `ApplicationContext` manages the bean lifecycle.
- Constructor injection resolves dependencies before bean creation.

## Common Mistakes

- Thinking Spring reads `pom.xml` at runtime.
- Confusing component scanning with bean creation.
- Assuming annotations execute themselves.

## Interview Questions

- What happens inside `SpringApplication.run()`?
- What is the difference between IoC and DI?
- How does Spring discover beans?
- Why is constructor injection recommended?

## Engineering Principle

Understand the start-up sequence before using the framework.

## My Mental Model

```text
JVM
  ↓
main()
  ↓
SpringApplication.run()
  ↓
Read @SpringBootApplication
  ↓
Component Scan
  ↓
Bean Definitions
  ↓
Dependency Resolution
  ↓
Bean Creation
  ↓
ApplicationContext
  ↓
Embedded Tomcat Starts
  ↓
Application Ready
```

## Things I Don't Yet Understand

- How does `ApplicationContext` store beans?
- What is `BeanFactory`?
- What is `BeanDefinition`?
- How does auto-configuration actually work?
- How does embedded Tomcat start?
- How does `DispatcherServlet` get created?
