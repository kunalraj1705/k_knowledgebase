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

**Phase 1 — Metadata Discovery**

.class

↓

ClassLoader

↓

Component Scan

↓

Read Annotations

↓

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

**Phase 2 — Bean Creation**

Reflection

↓

Java Object

↓

Dependency Injection

↓

@PostConstruct

↓

BeanPostProcessor

↓

Proxy (if required)

↓

Return Bean

Instead of scanning annotations again, Spring reuses the metadata collected earlier.

---

**2. BeanPostProcessor**

A BeanPostProcessor is a Spring extension point.

It receives every fully initialized bean before the bean is exposed from the container.

Conceptually:

Bean

↓

BeanPostProcessor

↓

Inspect

↓

Modify / Enhance / Replace

↓

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

↓

Object Created

↓

Dependencies Injected

↓

@PostConstruct

Runs inside the bean.

Example:

@PostConstruct

public void init() {

}

**BeanPostProcessor**

Purpose: Allow Spring to enhance or replace the bean.

@PostConstruct

↓

BeanPostProcessor

↓

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

↓

Retry Proxy

↓

Cache Proxy

↓

Transaction Proxy

↓

Target Object

This creates unnecessary complexity.

---

**5. One Proxy with Multiple Interceptors**

Spring creates one proxy.

Inside that proxy lives an interceptor chain.

Proxy

↓

SecurityInterceptor

↓

CacheInterceptor

↓

TransactionInterceptor

↓

Target Method

Each interceptor performs one responsibility.

Typical interceptor structure:

Before

↓

Proceed

↓

After

---

**6. Execution Order Matters**

Example:

@Transactional

@Cacheable

@PreAuthorize

A sensible execution order is:

Security

↓

Cache

↓

Transaction

↓

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

↓

PaymentService

↓

@PostConstruct

↓

BeanPostProcessor

↓

Proxy Created

During @PostConstruct:

* Proxy does not exist.
* this refers to the original object.

Therefore: No transaction starts.

transfer()

↓

Original Object

↓

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

↓

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

├── Pointcut

│      ↓

│  Which methods?

│

└── Advice

       ↓

What should execute?

Example:

Advisor

Pointcut

↓

@Transactional

Advice

↓

TransactionInterceptor

Another advisor:

Advisor

Pointcut

↓

@Cacheable

Advice

↓

CacheInterceptor

---

**11. AnnotationAwareAspectJAutoProxyCreator**

This is Spring's generic proxy creator.

Its responsibility is not to understand transactions or caching.

Instead it understands Advisors.

**Algorithm**

Receive bean.

Bean

↓

Load All Advisors

↓

For Every Advisor

↓

Does Pointcut Match?

↓

YES

↓

Collect Advice (Interceptor)

↓

Finished?

↓

Any Interceptors Collected?

↓

No → Return Original Bean

Yes

↓

Create ONE Proxy

↓

Attach Interceptor Chain

↓

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

        │

        ▼

Component Scan

        │

        ▼

Read Annotations

        │

        ▼

Create BeanDefinitions (Metadata)

        │

        ▼

Read Configuration

        │

        ▼

Register Infrastructure Beans

        │

        ├── TransactionAdvisor

        ├── CacheAdvisor

        ├── AsyncAdvisor

        └── SecurityAdvisor

        │

        ▼

Create Application Bean

        │

        ▼

Reflection

        │

        ▼

Dependency Injection

        │

        ▼

@PostConstruct

        │

        ▼

AnnotationAwareAspectJAutoProxyCreator

        │

        ▼

Evaluate All Advisors

        │

        ▼

Collect Matching Interceptors

        │

        ▼

No Match?

        │

        ├── Yes → Return Original Bean

        │

        └── No

              │

              ▼

Create ONE Proxy

              │

              ▼

Attach Interceptor Chain

              │

              ▼

Return Bean

---

**Internal Working Summary**

@Component Scan

        │

        ▼

Metadata Stored

        │

        ▼

Infrastructure Beans Registered

        │

        ▼

Application Bean Created

        │

        ▼

@PostConstruct

        │

        ▼

BeanPostProcessor

        │

        ▼

AnnotationAwareAspectJAutoProxyCreator

        │

        ▼

Advisor Matching

        │

        ▼

Collect Interceptors

        │

        ▼

Proxy Created

        │

        ▼

Spring Container Stores Proxy

---

**Interview Explanation (2–3 Minutes)**

Spring parses annotations during component scanning and stores them as metadata in BeanDefinitions. During bean creation, Reflection instantiates the object, dependencies are injected, and @PostConstruct performs bean-specific initialization. Next, BeanPostProcessors receive the fully initialized bean. The AnnotationAwareAspectJAutoProxyCreator evaluates all registered Advisors against the bean. Each Advisor combines a Pointcut (which methods match) with Advice (the interceptor to execute). If no Advisors match, Spring returns the original bean. If one or more Advisors match, Spring creates a single proxy and attaches the matching interceptors as an ordered chain. At runtime, method invocations flow through this interceptor chain before reaching the target object. This architecture keeps Spring extensible, efficient, and compliant with the Open/Closed Principle.

**Final Takeaway**

Spring's AOP infrastructure is built on reusable abstractions. It discovers metadata once, registers reusable Advisors, creates a single proxy per bean, and composes an ordered interceptor chain based on matching Pointcuts. This separation of concerns keeps the framework performant, extensible, and maintainable.
