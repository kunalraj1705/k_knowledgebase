## Learning Objective**

Understand how Spring creates proxies, executes interceptor chains, chooses between JDK Dynamic Proxy and CGLIB, and why self-invocation bypasses Spring AOP.

# Module Overview**

Client

   │

   ▼

Proxy

   │

   ▼

Interceptor Chain

   │

   ▼

Target Method

This module answers four important questions:

1. How does Spring execute a proxied method?
2. How does Spring create proxy classes?
3. Why are there JDK Proxy and CGLIB?
4. Why does self-invocation fail?

Note:

CGLIB - Code Generation Library

AOP - Aspect-Oriented Programming

---

# 1. Calling a Spring Bean**

Suppose we have

@Service

public class PaymentService {

    @Transactional

    @Cacheable

    public void transfer() {

        System.out.println("Business Logic");

    }

}

Spring injects

@Autowired

private PaymentService paymentService;

Although it appears to be a normal object...

It is actually **Proxy**

so paymentService.transfer(); does not call PaymentService.transfer()

instead

Client

   │

   ▼

Proxy.transfer()

---

# 2. Why a Proxy?**

Without Proxy

Client

   │

   ▼

Business Method

Spring cannot execute

* Transaction
* Cache
* Security
* Retry
* Async
* Metrics

**With a proxy**

Client

   │

   ▼

Proxy

   │

   ▼

Interceptors

   │

   ▼

Business Method

The proxy becomes the execution gateway.

---

# 3. Interceptor Chain**

Suppose

@Transactional

@Cacheable

Spring creates

Proxy

   │

   ▼

CacheInterceptor

   │

   ▼

TransactionInterceptor

   │

   ▼

PaymentService

**Each interceptor performs**

Before Logic

↓

Next

↓

After Logic

**Conceptually:**

before();

Object result = invocation.proceed();

after();

return result;

Every interceptor follows exactly the same algorithm.

---

# 4. Chain of Responsibility Pattern**

Each interceptor never knows the complete chain.

it only knows invocation.proceed();

**Execution:**

Security

    │

    ▼

Cache

    │

    ▼

Transaction

    │

    ▼

Target

If an interceptor decides Stop then the chain terminates

Examples:

1. Cache Hit
2. Authorization Failure
3. Rate Limit
4. Circuit Breaker

---

# 5. ReflectiveMethodInvocation**

Spring stores invocation state inside ReflectiveMethodInvocation

**Conceptually:**

class ReflectiveMethodInvocation {

    Object target;

    Method method;

    Object\[] arguments;

    List<MethodInterceptor> interceptors;

    int currentInterceptorIndex;

}

This object represents One method invocation Not One bean

---

# 6. Why currentInterceptorIndex Starts at -1**

Spring initializes currentInterceptorIndex = -1; instead of 0

Reason:

Every invocation follows exactly the same algorithm

{

  index++;

  invoke(interceptor\[index]);

}

Execution:

\-1

 │

 ▼

0 → Security

1 → Cache

2 → Transaction

3 → Target

---

# 7. proceed()**

Conceptually:

Object proceed() {

    currentInterceptorIndex++;

    if(moreInterceptors){

        invoke(nextInterceptor);

    }else{

        invokeTargetMethod();

    }

}

Every interceptor simply executes

invocation.proceed();

It never knows

* previous interceptor
* next interceptor
* total interceptor count

Traversal is centralized.

---

# 8. Exception Propagation**

Transaction interceptor

beginTransaction();

try{

    Object result = invocation.proceed();

    commit();

}

catch(Exception e){

    rollback();

    throw e;

}

Why rethrow?

Because the caller must know The operation failed.

Otherwise

paymentService.transfer();

System.out.println("Success");

---

# 9. Runtime Execution**

Client

↓

Proxy

↓

Security

↓

Cache

↓

Transaction

↓

Business Method

↓

Transaction Commit

↓

Return

↓

Client

---

# 10. JDK Dynamic Proxy**

Suppose

public interface PaymentService {

    void transfer();

}

Implementation:

public class PaymentServiceImpl implements PaymentService {

}

JDK creates

class $Proxy0 implements PaymentService {

    InvocationHandler handler;

}

It implements the interface

---

# 11. Why JDK Proxy Cannot Proxy Classes**

public class PaymentService {

}

JDK would need

class Proxy implements PaymentService

This is not possible because Class ≠ Interface

Java allows

* extends Class
* implements Interface

Java doesn't allow

* implements Class

Therefore

JDK Proxy only works for interfaces.

---

# 12. CGLIB**

Spring solves this using CGLIB

Instead of implementing interfaces It generates subclasses.

Original:

public class PaymentService {

}

Runtime:

class PaymentService$$Enhancer extends PaymentService {

}

Now interception happens by

@Override

transfer(){

    before();

    super.transfer();

    after();

}

---

# 13. Final Class Limitation**

Suppose:

public final class PaymentService {

}

CGLIB tries:

class PaymentService$$Enhancer extends PaymentService

This fails because JAVA forbids extending final class

Result: No Proxy

---

# 14. Final Method Limitation**

Suppose:

public class PaymentService {

    public final void transfer(){

    }

}

CGLIB tries to override transfer() but it fails because JAVA doesn't allow overriding final method.

Result: No Proxy

---

# 15. Spring's Proxy Selection**

Decision Tree

Does Bean Implement Interface?

        │

   ┌────┴─────┐

   │          │

  Yes         No

   │          │

   ▼          ▼

JDK Proxy   CGLIB

Spring prefers the simplest mechanism that satisfies the requirement.

---

# 16. Forcing CGLIB**

Spring allows

* EnableTransactionManagement(proxyTargetClass = true)
* @EnableAspectJAutoProxy(proxyTargetClass = true)

Even if interfaces exist Spring always generates CGLIB proxies.

---

# 17. Injection Differences**

**JDK Proxy:**

$Proxy0 implements PaymentService

Valid:

@Autowired

PaymentService service;

Invalid:

@Autowired

PaymentServiceImpl service;

Because:

$Proxy0 IS NOT PaymentServiceImpl

**CGLIB:**

PaymentService$$Enhancer extends PaymentService

Both Works:

@Autowired

PaymentService service;

@Autowired

PaymentServiceImpl service;

Because: Enhancer IS-A PaymentServiceImpl

---

# 18. Self Invocation**

Suppose:

public void pay(){

    this.transfer();

}

Execution:

pay()

↓

this.transfer()

↓

Original Method

The proxy is bypassed. No interceptor executes.

Because proxy will work on the external call (incoming calls from outside of the proxy class)

Therefore:

@Transactional

@Cacheable

@Async

do not work.

---

# 19. Why?**

Spring AOP intercepts Calls entering the proxy

It does NOT monitor every JVM method invocation.

Once execution reaches the target object

this.transfer() is a normal Java call.

---

# 20. Self Injection Workaround**

@Autowired

private PaymentService self;

Execution:

pay()

↓

self.transfer()

↓

Proxy

↓

Interceptors

↓

Target

Because:

self = Proxy

---

# 21. Runtime Class Types**

JDK:

paymentService.getClass()

return something similar to: jdk.proxy2.$Proxy45

CGLIB:

returns: PaymentService$$EnhancerBySpringCGLIB

The injected bean is not the original implementation class.

It is the generated proxy.

---

# End-to-End Architecture**

                Spring Startup

                       │

                       ▼

              Scan Bean Definitions

                       │

                       ▼

             Detect AOP Annotations

                       │

                       ▼

              Build Advisor List

                       │

                       ▼

         Bean Implements Interface?

               │               │

             Yes               No

              │                │

              ▼                ▼

        JDK Dynamic        CGLIB Proxy

            Proxy            Generation

               │                │

               └──────┬─────────┘

                      ▼

               Create Proxy Bean

                      │

                      ▼

            Store Interceptor Chain

                      │

                      ▼

========================================

             Runtime Execution

Client

   │

   ▼

Proxy

   │

   ▼

Security

   │

   ▼

Cache

   │

   ▼

Transaction

   │

   ▼

Business Method

   │

   ▼

Return Result

---

CGLIB exists to overcome the limitation of JDK Dynamic Proxy. JDK proxies can only create proxy implementations for interfaces, whereas many Spring beans are concrete classes without interfaces. CGLIB generates a subclass of the target class at runtime and overrides its methods to insert interception logic. This allows Spring AOP to work even when no interface is present, while still keeping business logic separate from infrastructure concerns.
