## Learning Objective


By the end of this module, you should understand:


* Why Spring caches metadata instead of repeatedly scanning annotations.
* The purpose of BeanPostProcessor.
* Why Spring creates one proxy instead of multiple nested proxies.
* The difference between Advice, Pointcut, and Advisor.
* How AnnotationAwareAspectJAutoProxyCreator works.
* The complete proxy creation pipeline.


---
# Problem Statement


Consider the following bean:

@Service

public class PaymentService {


   @Transactional

   @Cacheable

   public void transfer() {


   }


}


Questions:


* Who discovers @Transactional?
* Who decides a proxy is required?
* How does Spring know which interceptor to execute?
* Why doesn't Spring create one proxy per annotation?


This module answers these questions.


---
**1. Metadata is Parsed Once**

Spring separates startup into two phases.


**Phase 1 Ã¢â‚¬â€ Metadata Discovery**

.class

Ã¢â€ â€œ

ClassLoader

Ã¢â€ â€œ

Component Scan

Ã¢â€ â€œ

Read Annotations

Ã¢â€ â€œ

BeanDefinition (Metadata)


During component scanning Spring discovers:

* @Service
* @Repository
* @Controller
* @Transactional
* @Cacheable
* @Async
* etc.


No Java object exists yet.

Only metadata is collected.


**Phase 2 Ã¢â‚¬â€ Bean Creation**

Reflection

Ã¢â€ â€œ

Java Object

Ã¢â€ â€œ

Dependency Injection

Ã¢â€ â€œ

@PostConstruct

Ã¢â€ â€œ

BeanPostProcessor

Ã¢â€ â€œ

Proxy (if required)

Ã¢â€ â€œ

Return Bean


Instead of scanning annotations again, Spring reuses the metadata collected earlier.


---
**2. BeanPostProcessor**

A BeanPostProcessor is a Spring extension point.

It receives every fully initialized bean before the bean is exposed from the container.


Conceptually:

Bean

Ã¢â€ â€œ

BeanPostProcessor

Ã¢â€ â€œ

Inspect

Ã¢â€ â€œ

Modify / Enhance / Replace

Ã¢â€ â€œ

Return Bean


Not every BeanPostProcessor creates proxies.

Examples:

* Validation
* Dependency enhancement
* Metadata registration
* Proxy creation


---
**3. @PostConstruct vs BeanPostProcessor**


**@PostConstruct**

Purpose: Initialize the bean after dependency injection.


Reflection

Ã¢â€ â€œ

Object Created

Ã¢â€ â€œ

Dependencies Injected

Ã¢â€ â€œ

@PostConstruct


Runs inside the bean.

Example:


@PostConstruct

public void init() {


}


**BeanPostProcessor**

Purpose: Allow Spring to enhance or replace the bean.

@PostConstruct

Ã¢â€ â€œ

BeanPostProcessor

Ã¢â€ â€œ

Proxy Created (if required)


Runs outside the bean.


**Comparison:**


| @PostConstruct             | BeanPostProcessor           |

| -------------------------- | --------------------------- |

| Bean initializes itself    | Framework enhances bean     |

| Runs inside bean           | Runs outside bean           |

| After dependency injection | After initialization        |

| Cannot replace bean        | Can replace bean with proxy |


---
**4. Why Proxies are Created**

Suppose a bean contains:@Transactional


Should Spring create: Transaction Proxy for every annotation?

Ans: NO


Imagine:

@Transactional

@Cacheable

@Async

@Retryable


Multiple nested proxies would produce:

Async Proxy

Ã¢â€ â€œ

Retry Proxy

Ã¢â€ â€œ

Cache Proxy

Ã¢â€ â€œ

Transaction Proxy

Ã¢â€ â€œ

Target Object


This creates unnecessary complexity.


---
**5. One Proxy with Multiple Interceptors**

Spring creates one proxy.

Inside that proxy lives an interceptor chain.

Proxy

Ã¢â€ â€œ

SecurityInterceptor

Ã¢â€ â€œ

CacheInterceptor

Ã¢â€ â€œ

TransactionInterceptor

Ã¢â€ â€œ

Target Method


Each interceptor performs one responsibility.

Typical interceptor structure:

Before

Ã¢â€ â€œ

Proceed

Ã¢â€ â€œ

After


---
**6. Execution Order Matters**

Example:

@Transactional

@Cacheable

@PreAuthorize


A sensible execution order is:

Security

Ã¢â€ â€œ

Cache

Ã¢â€ â€œ

Transaction

Ã¢â€ â€œ

Business Logic


Reason:

* Unauthorized users are rejected immediately.
* Cache avoids unnecessary database work.
* Transactions begin only when required.


---
**7. Why @Transactional Doesn't Work Inside @PostConstruct**

Example:

@Service

@Transactional

public class PaymentService {


   @PostConstruct

   public void init() {

       transfer();

   }


   @Transactional

   public void transfer() {

   }

}


Lifecycle:

Reflection

Ã¢â€ â€œ

PaymentService

Ã¢â€ â€œ

@PostConstruct

Ã¢â€ â€œ

BeanPostProcessor

Ã¢â€ â€œ

Proxy Created


During @PostConstruct:

* Proxy does not exist.
* this refers to the original object.


Therefore: No transaction starts.

transfer()

Ã¢â€ â€œ

Original Object

Ã¢â€ â€œ

No TransactionInterceptor


**Same reason for Self Invocation**

public void save() {

   transfer();

}


Compiles to: this.transfer();

Again, proxy is bypassed.


**External Bean**

@Autowired

private PaymentService paymentService;


Spring injects:

Transaction Proxy

Ã¢â€ â€œ

PaymentService


Now @Transactional works.


---
**8. Advice**

Advice answers: What should execute?

Examples:

* TransactionInterceptor
* CacheInterceptor
* SecurityInterceptor
* AsyncInterceptor

One interceptor instance is reused across many methods.


---
**9. Pointcut**

Pointcut answers: Where should the behavior execute?

Examples:

\-All methods annotated with @Transactional

or

\-All methods inside com.krb.service.*

A Pointcut contains matching rules only.


---
**10. Advisor**

An Advisor combines: Pointcut + Advice


Conceptually:

Advisor

Ã¢â€Å“Ã¢â€â‚¬Ã¢â€â‚¬ Pointcut

Ã¢â€â€š      Ã¢â€ â€œ

Ã¢â€â€š  Which methods?

Ã¢â€â€š

Ã¢â€â€Ã¢â€â‚¬Ã¢â€â‚¬ Advice

      Ã¢â€ â€œ

What should execute?


Example:

Advisor


Pointcut

Ã¢â€ â€œ

@Transactional


Advice

Ã¢â€ â€œ

TransactionInterceptor


Another advisor:

Advisor


Pointcut

Ã¢â€ â€œ

@Cacheable


Advice

Ã¢â€ â€œ

CacheInterceptor


---
**11. AnnotationAwareAspectJAutoProxyCreator**

This is Spring's generic proxy creator.

Its responsibility is not to understand transactions or caching.

Instead it understands Advisors.


**Algorithm**

Receive bean.

Bean

Ã¢â€ â€œ

Load All Advisors

Ã¢â€ â€œ

For Every Advisor

Ã¢â€ â€œ

Does Pointcut Match?

Ã¢â€ â€œ

YES

Ã¢â€ â€œ

Collect Advice (Interceptor)

Ã¢â€ â€œ

Finished?

Ã¢â€ â€œ

Any Interceptors Collected?

Ã¢â€ â€œ

No Ã¢â€ â€™ Return Original Bean

Yes

Ã¢â€ â€œ

Create ONE Proxy

Ã¢â€ â€œ

Attach Interceptor Chain

Ã¢â€ â€œ

Return Proxy


Notice:

It never asks:

* Is this transactional?
* Is this cacheable?


Instead it asks:

* Which Advisors match?


---
**12. Complete Startup Flow**


Application Starts

       Ã¢â€â€š

       Ã¢â€“Â¼

Component Scan

       Ã¢â€â€š

       Ã¢â€“Â¼

Read Annotations

       Ã¢â€â€š

       Ã¢â€“Â¼

Create BeanDefinitions (Metadata)

       Ã¢â€â€š

       Ã¢â€“Â¼

Read Configuration

       Ã¢â€â€š

       Ã¢â€“Â¼

Register Infrastructure Beans

       Ã¢â€â€š

       Ã¢â€Å“Ã¢â€â‚¬Ã¢â€â‚¬ TransactionAdvisor

       Ã¢â€Å“Ã¢â€â‚¬Ã¢â€â‚¬ CacheAdvisor

       Ã¢â€Å“Ã¢â€â‚¬Ã¢â€â‚¬ AsyncAdvisor

       Ã¢â€â€Ã¢â€â‚¬Ã¢â€â‚¬ SecurityAdvisor

       Ã¢â€â€š

       Ã¢â€“Â¼

Create Application Bean

       Ã¢â€â€š

       Ã¢â€“Â¼

Reflection

       Ã¢â€â€š

       Ã¢â€“Â¼

Dependency Injection

       Ã¢â€â€š

       Ã¢â€“Â¼

@PostConstruct

       Ã¢â€â€š

       Ã¢â€“Â¼

AnnotationAwareAspectJAutoProxyCreator

       Ã¢â€â€š

       Ã¢â€“Â¼

Evaluate All Advisors

       Ã¢â€â€š

       Ã¢â€“Â¼

Collect Matching Interceptors

       Ã¢â€â€š

       Ã¢â€“Â¼

No Match?

       Ã¢â€â€š

       Ã¢â€Å“Ã¢â€â‚¬Ã¢â€â‚¬ Yes Ã¢â€ â€™ Return Original Bean

       Ã¢â€â€š

       Ã¢â€â€Ã¢â€â‚¬Ã¢â€â‚¬ No

             Ã¢â€â€š

             Ã¢â€“Â¼

Create ONE Proxy

             Ã¢â€â€š

             Ã¢â€“Â¼

Attach Interceptor Chain

             Ã¢â€â€š

             Ã¢â€“Â¼

Return Bean


---
**Internal Working Summary**


@Component Scan

       Ã¢â€â€š

       Ã¢â€“Â¼

Metadata Stored

       Ã¢â€â€š

       Ã¢â€“Â¼

Infrastructure Beans Registered

       Ã¢â€â€š

       Ã¢â€“Â¼

Application Bean Created

       Ã¢â€â€š

       Ã¢â€“Â¼

@PostConstruct

       Ã¢â€â€š

       Ã¢â€“Â¼

BeanPostProcessor

       Ã¢â€â€š

       Ã¢â€“Â¼

AnnotationAwareAspectJAutoProxyCreator

       Ã¢â€â€š

       Ã¢â€“Â¼

Advisor Matching

       Ã¢â€â€š

       Ã¢â€“Â¼

Collect Interceptors

       Ã¢â€â€š

       Ã¢â€“Â¼

Proxy Created

       Ã¢â€â€š

       Ã¢â€“Â¼

Spring Container Stores Proxy


---
**Interview Explanation (2Ã¢â‚¬â€œ3 Minutes)**


Spring parses annotations during component scanning and stores them as metadata in BeanDefinitions. During bean creation, Reflection instantiates the object, dependencies are injected, and @PostConstruct performs bean-specific initialization. Next, BeanPostProcessors receive the fully initialized bean. The AnnotationAwareAspectJAutoProxyCreator evaluates all registered Advisors against the bean. Each Advisor combines a Pointcut (which methods match) with Advice (the interceptor to execute). If no Advisors match, Spring returns the original bean. If one or more Advisors match, Spring creates a single proxy and attaches the matching interceptors as an ordered chain. At runtime, method invocations flow through this interceptor chain before reaching the target object. This architecture keeps Spring extensible, efficient, and compliant with the Open/Closed Principle.


**Final Takeaway**

Spring's AOP infrastructure is built on reusable abstractions. It discovers metadata once, registers reusable Advisors, creates a single proxy per bean, and composes an ordered interceptor chain based on matching Pointcuts. This separation of concerns keeps the framework performant, extensible, and maintainable.
