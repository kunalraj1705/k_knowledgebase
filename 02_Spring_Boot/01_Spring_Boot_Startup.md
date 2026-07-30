# Spring Boot Startup



## Why does this exist?

Spring Framework required extensive manual configuration (XML and later Java Configuration). Spring Boot reduces this boilerplate by providing sensible defaults and automatic configuration while still allowing customization when needed.

This will auto add the configuration when it finds the dependencies in the classpath. Lets say I want to add tomcat or I want to add a db. In Spring boot all we do is add the dependency in pom.xml and Maven downloads the dependency. During application startup, Spring Boot detects the dependency on the classpath and automatically configures the related infrastructure.



## Key Concept

* @SpringBootApplication
* @EnableAutoConfiguration
* @CompenentScan
* ApplicationContext
* Bean
* IoC Container
* Dependency Injection



## Startup flow

Just like a java program the application class call the main method. The main method is the entry point of any java application. Inside the main method we write SpringApplication.run(ApplicationClassName.class). SpringApplication.run() internally creates and configures a SpringApplication instance using the ApplicationClass.class as its primary source. The argument will be an object then how does this ApplicationClassName.class creates object. It doesn't create object ApplicationClassName.class returns the Class object. Using Java Reflection, Spring inspects the Class object to discover annotations such as @SpringBootApplication. If it find this then it continues the working defied for this annotation. This annotation has 3 sub annotation.

1: @SpringBootConfiguration: Marks this class as a Spring configuration class. It is a specialized form of @Configuration and serves as the primary source of bean definitions.

2: @EnableAutoConfiguration: This will add the required components for dependencies found in the class path.

3: @ComponentScan: This will find all the class with annotation like: @Controller @Service @Repository @Component and stores the Bean Definition inside the BeanFactory.



After the bean definition is created it will start creating the bean. Spring resolves the dependency graph and creates beans in an order that satisfies all required dependencies. The ApplicationContext stores, manages, initializes, and provides beans throughout the application's lifecycle. IoC container creates bean and injects their required dependencies during bean initialization. There are two type of Injection: Via Constructor, via Field. DI via constructor is preferred because before starting the Application it resolves all the dependency. If everything is resolved then it will start the application else the application will not start and fail with exception BeanCurrentlyInCreationException.





## Important Learning

* Spring uses java reflection to inspect annotation.
* Component scanning discovers bean definitnions.
* Auto-configuration creates the infrastructure beans.
* ApplicationContext manages the bean lifecycle.
* Constructor injects resolves dependencies before bean creation



## Common Mistakes

* Thinking spring reads pom.xml at runtime
* confusing component scan with bean creation
* assuming annotations executes themselves.



## Interview Questions

* What happens inside SpringApplication.run()?
* What is the difference between IoC and DI?
* How does Spring discover beans?
* Why is constructor injection recommended?



## Engineering Principle

Understand the start-up sequence before using the framework.



## My Mental Model

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



## Things I Don't Yet Understand



* How does ApplicationContext store beans?
* What is BeanFactory?
* What is BeanDefinition?
* How does AutoConfiguration actually work?
* How does Embedded Tomcat start?
* How does DispatcherServlet get created?

