##

# 1. Why does Spring AOP exist?**

# Problem**

Enterprise applications require infrastructure logic across multiple business methods.

Examples:

* Transaction Management
* Security
* Logging
* Caching
* Metrics
* Retry
* Validation

**Without AOP:**

public void transfer() {

    authenticate();

    beginTransaction();

    log();

    businessLogic();

    commit();

}

Every service repeats the same infrastructure code.

Problems:

* Code duplication
* Tight coupling
* Business logic polluted with infrastructure
* Difficult maintenance
* Violates Single Responsibility Principle (SRP)

---

# Spring's Solution**

Separate business logic from infrastructure.

Client

    │

    ▼

Proxy

    │

    ├── Security

    ├── Transaction

    ├── Logging

    ├── Cache

    ├── Metrics

    └── Retry

    │

    ▼

Business Method remains clean

Infrastructure executes transparently.

---

# 2. Core Components**

* **Aspect**

  * Developer's logical grouping of cross-cutting behaviour.
  * @Aspect
  * class LoggingAspect {
  *
  * }
  * An Aspect is not executed directly.
* Advice

  * Defines what should execute.
  * Examples:

    * Before
    * After
    * Around
    * AfterReturning
    * AfterThrowing
* Pointcut:

  * Defines where Advice should execute.
  * Example:

    * Methods annotated with @Transactional
* Advisor:

  * Spring's runtime representation.
  * Advisor = Pointcut + Advice
  * Spring internally works with Advisors. Not Aspects.
* MethodInterceptor:

  * Common runtime abstraction.
  * Every Advice eventually becomes a MethodInterceptor.
  * Regardless of whether the developer writes:

    * @Before
    * @After
    * @Around
    * @Transactional
  * Spring executes:

    * interceptor.invoke(...)

---

# 3. Startup Flow**

**Step 1**

Developer writes

@Aspect

class LoggingAspect {

}

@Transactional

**Step 2**

Spring parses annotations.

**Step 3**

Spring creates Advisors.

Aspect

↓

Advisor

**Step 4**

AutoProxyCreator is registered.

AutoProxyCreator is a BeanPostProcessor.

**Step 5**

Application beans are created.

Example: PaymentService

**Step 6**

AutoProxyCreator executes.

For every bean:

Does any Advisor match?

YES

↓

Create Proxy

**Step 7**

ApplicationContext stores Proxy NOT Target Object

**Complete Startup Flow**

Application Starts

↓

ApplicationContext

↓

Register Infrastructure Beans

↓

TransactionInterceptor

↓

TransactionAdvisor

↓

AutoProxyCreator

↓

Create Application Bean

↓

BeanPostProcessor

↓

Advisor Match

↓

Create Proxy

↓

Store Proxy

---

# 4. Runtime Flow**

Client calls: paymentService.transfer();

**Actual Execution:**

Client

↓

Proxy

↓

Interceptor Chain

↓

Business Method

**Example:**

Client

↓

LoggingInterceptor

↓

SecurityInterceptor

↓

TransactionInterceptor

↓

Business Method

Each interceptor executes "proceed();" until the business method is reached.

---

# 5. Around Advice**

Example:

@Around

public Object log(ProceedingJoinPoint pjp){

    before();

    Object result = pjp.proceed();

    after();

    return result;

}

Important: pjp.proceed() ≠ Invoke Business Method

Instead:

pjp.proceed()

↓

ReflectiveMethodInvocation.proceed()

↓

Next Interceptor

Only the last interceptor invokes the target method.

---

# 6. Advisor Ordering**

Spring may discover

* Security Advisor
* Transaction Advisor
* Logging Advisor

Before proxy creation:

Collect Advisors

↓

Sort Advisors

↓

Create Interceptor Chain

↓

Create Proxy

Ordering can be controlled by:

@Order(1)

@Order(2)

@Order(3)

Spring internally uses: AnnotationAwareOrderComparator

---

# 7. Self Invocation**

Example:

@Transactional

public void transfer(){

    debit();

}

@Transactional

public void debit(){

}

java converts debit(); to this.debit();

**Execution:**

Proxy

↓

transfer()

↓

this.debit()

↓

Target

The second call never reaches the proxy.

Therefore

* TransactionInterceptor ❌
* CacheInterceptor ❌
* AsyncInterceptor ❌
* RetryInterceptor ❌

are skipped.

**Solution:**

PaymentService

↓

DebitService

Bean-to-bean communication naturally goes through proxies.

**Alternative:**

* Self Injection
* AopContext.currentProxy()
* AspectJ

---

# 8. Transaction Infrastructure**

Developer writes: @EnableTransactionManagement

Spring registers:

TransactionInterceptor

↓

TransactionAdvisor

↓

AutoProxyCreator

Infrastructure is created before application beans. Otherwise proxy creation is impossible.

---

# 9. Important Learnings**

* Spring AOP is proxy based.
* Spring works with Advisors internally.
* Every Advice becomes a MethodInterceptor.
* Interceptor Chain executes sequentially.
* Around Advice controls chain continuation.
* Startup prepares everything.
* Runtime simply executes the prepared chain.
* BeanPostProcessor creates proxies.
* ApplicationContext stores proxies.
* Self invocation bypasses AOP.

---

# 10. Common Mistakes**

**Mistake 1:** Thinking annotations execute logic.

Reality: Annotations are metadata. Spring interprets them.

**Mistake 2:** Thinking Aspects execute directly.

Reality: Spring converts Aspects into Advisors.

**Mistake 3:** Thinking pjp.proceed() calls the business method.

Reality: It continues the interceptor chain.

**Mistake 4:** Thinking this.method(); is intercepted.

Reality: Self invocation bypasses proxies.

**Mistake 5:** Thinking ApplicationContext stores target objects.

Reality: It usually stores proxies.

---

# 11. Interview Questions**

**Q:** Why does Spring use proxies?

**A:** To separate cross-cutting concerns from business logic while keeping application code clean.

**Q:** Why does Spring convert Advice into MethodInterceptors?

**A:** To provide one common execution model regardless of the advice type.

**Q:** Why doesn't @Transactional work during self invocation?

**A:** Because internal method calls (this.method()) bypass the Spring proxy, so the interceptor chain is never executed.

**Q:** What is the difference between Aspect and Advisor?

**A:** Aspect is the developer abstraction. Advisor is Spring's runtime abstraction.

**Q:** Why does Spring build proxies during startup?

**A:** To avoid expensive annotation scanning during runtime.

---

# 12. My Mental Model**

Developer

↓

Aspect

↓

Spring

↓

Advisor

↓

AutoProxyCreator

↓

Proxy

↓

Interceptor Chain

↓

Business Method

---

**Module Summary:**

Spring AOP is a proxy-based interception mechanism. During startup, Spring transforms developer-defined Aspects and annotations into Advisors and MethodInterceptors, creates proxies using AutoProxyCreator, and stores those proxies in the ApplicationContext. At runtime, every external method call flows through an ordered interceptor chain before reaching the target object. Understanding proxy boundaries, interceptor chaining, advisor ordering, and self-invocation explains the behaviour of @Transactional, @Cacheable, @Async, @Retryable, and most AOP-based features in Spring.
