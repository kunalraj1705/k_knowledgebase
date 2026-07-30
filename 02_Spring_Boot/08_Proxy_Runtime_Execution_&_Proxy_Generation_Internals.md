## Learning Objective

Understand how Spring creates proxies, executes interceptor chains, chooses between JDK Dynamic Proxy and CGLIB, and why self-invocation bypasses Spring AOP.

# Module Overview

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

# 1. Calling a Spring Bean

Suppose we have

```java
@Service
public class PaymentService {

    @Transactional
    @Cacheable
    public void transfer() {
        System.out.println("Business Logic");
    }
}
```

Spring injects

```java
@Autowired
private PaymentService paymentService;
```

Although it appears to be a normal object...

It is actually a Proxy.

So `paymentService.transfer();` does not call `PaymentService.transfer()` directly.

Instead:

```text
Client
   │
   ▼
Proxy.transfer()
```

---

# 2. Why a Proxy?

Without Proxy

```text
Client
   │
   ▼
Business Method
```

Spring cannot execute

* Transaction
* Cache
* Security
* Retry
* Async
* Metrics

With a proxy

```text
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
```

The proxy becomes the execution gateway.

---

# 3. Interceptor Chain

Suppose

```java
@Transactional
@Cacheable
```

Spring creates

```text
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
```

Each interceptor performs:

```text
Before Logic
   ↓
Next
   ↓
After Logic
```

Conceptually:

```java
before();
Object result = invocation.proceed();
after();
return result;
```

Every interceptor follows exactly the same algorithm.

---

# 4. Chain of Responsibility Pattern

Each interceptor never knows the complete chain.

It only knows `invocation.proceed();`

Execution:

```text
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
```

If an interceptor decides to stop, the chain terminates.

Examples:

1. Cache Hit
2. Authorization Failure
3. Rate Limit
4. Circuit Breaker

---

# 5. ReflectiveMethodInvocation

Spring stores invocation state inside `ReflectiveMethodInvocation`.

Conceptually:

```java
class ReflectiveMethodInvocation {
    Object target;
    Method method;
    Object[] arguments;
    List<MethodInterceptor> interceptors;
    int currentInterceptorIndex;
}
```

This object represents one method invocation, not one bean.

---

# 6. Why `currentInterceptorIndex` Starts at -1

Spring initializes `currentInterceptorIndex = -1;` instead of `0`.

Reason:

Every invocation follows exactly the same algorithm:

```java
{
    index++;
    invoke(interceptor[index]);
}
```

Execution:

```text
-1
 │
 ▼
0 → Security
1 → Cache
2 → Transaction
3 → Target
```

---

# 7. `proceed()`

Conceptually:

```java
Object proceed() {
    currentInterceptorIndex++;

    if (moreInterceptors) {
        invoke(nextInterceptor);
    } else {
        invokeTargetMethod();
    }
}
```

Every interceptor simply executes:

```java
invocation.proceed();
```

It never knows:

* previous interceptor
* next interceptor
* total interceptor count

Traversal is centralized.

---

# 8. Exception Propagation

Transaction interceptor

```java
beginTransaction();

try {
    Object result = invocation.proceed();
    commit();
} catch (Exception e) {
    rollback();
    throw e;
}
```

Why rethrow?

Because the caller must know that the operation failed.

Otherwise:

```java
paymentService.transfer();
System.out.println("Success");
```

---

# 9. Runtime Execution

```text
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
Transaction Commit
   │
   ▼
Return
   │
   ▼
Client
```

---

# 10. JDK Dynamic Proxy

Suppose

```java
public interface PaymentService {
    void transfer();
}
```

Implementation:

```java
public class PaymentServiceImpl implements PaymentService {
}
```

JDK creates

```java
class $Proxy0 implements PaymentService {
    InvocationHandler handler;
}
```

It implements the interface.

---

# 11. Why JDK Proxy Cannot Proxy Classes

```java
public class PaymentService {
}
```

JDK would need

```java
class Proxy implements PaymentService
```

This is not possible because `Class ≠ Interface`.

Java allows:

* extends Class
* implements Interface

Java does not allow:

* implements Class

Therefore,

JDK Proxy only works for interfaces.

---

# 12. CGLIB

Spring solves this using CGLIB.

Instead of implementing interfaces, it generates subclasses.

Original:

```java
public class PaymentService {
}
```

Runtime:

```java
class PaymentService$$Enhancer extends PaymentService {
}
```

Now interception happens by

```java
@Override
transfer() {
    before();
    super.transfer();
    after();
}
```

---

# 13. Final Class Limitation

Suppose:

```java
public final class PaymentService {
}
```

CGLIB tries:

```java
class PaymentService$$Enhancer extends PaymentService
```

This fails because Java forbids extending a final class.

Result: No Proxy.

---

# 14. Final Method Limitation

Suppose:

```java
public class PaymentService {
    public final void transfer() {
    }
}
```

CGLIB tries to override `transfer()` but it fails because Java does not allow overriding a final method.

Result: No Proxy.

---

# 15. Spring's Proxy Selection

Decision Tree

```text
Does Bean Implement Interface?
        │
   ┌────┴─────┐
   │          │
  Yes         No
   │          │
   ▼          ▼
JDK Proxy   CGLIB
```

Spring prefers the simplest mechanism that satisfies the requirement.

---

# 16. Forcing CGLIB

Spring allows

* `EnableTransactionManagement(proxyTargetClass = true)`
* `@EnableAspectJAutoProxy(proxyTargetClass = true)`

Even if interfaces exist, Spring always generates CGLIB proxies.

---

# 17. Injection Differences

JDK Proxy:

```java
$Proxy0 implements PaymentService
```

Valid:

```java
@Autowired
PaymentService service;
```

Invalid:

```java
@Autowired
PaymentServiceImpl service;
```

Because:

```text
$Proxy0 IS NOT PaymentServiceImpl
```

CGLIB:

```java
PaymentService$$Enhancer extends PaymentService
```

Both work:

```java
@Autowired
PaymentService service;

@Autowired
PaymentServiceImpl service;
```

Because:

```text
Enhancer IS-A PaymentServiceImpl
```

---

# 18. Self Invocation

Suppose:

```java
public void pay() {
    this.transfer();
}
```

Execution:

```text
pay()
   ↓
this.transfer()
   ↓
Original Method
```

The proxy is bypassed. No interceptor executes.

Because the proxy works on the external call, that is, incoming calls from outside of the proxy class.

Therefore:

```java
@Transactional
@Cacheable
@Async
```

do not work.

---

# 19. Why?

Spring AOP intercepts calls entering the proxy.

It does not monitor every JVM method invocation.

Once execution reaches the target object,

```java
this.transfer();
```

is a normal Java call.

---

# 20. Self Injection Workaround

```java
@Autowired
private PaymentService self;
```

Execution:

```text
pay()
   ↓
self.transfer()
   ↓
Proxy
   ↓
Interceptors
   ↓
Target
```

Because:

```text
self = Proxy
```

---

# 21. Runtime Class Types

JDK:

```java
paymentService.getClass()
```

returns something similar to:

```text
jdk.proxy2.$Proxy45
```

CGLIB:

returns:

```text
PaymentService$$EnhancerBySpringCGLIB
```

The injected bean is not the original implementation class.

It is the generated proxy.

---

# End-to-End Architecture

```text
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
   │
   ├── Yes ──> JDK Dynamic Proxy
   │
   └── No ───> CGLIB Proxy Generation
   │
   ▼
Create Proxy Bean
   │
   ▼
Store Interceptor Chain
```

```text
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
```

CGLIB exists to overcome the limitation of JDK Dynamic Proxy. JDK proxies can only create proxy implementations for interfaces, whereas many Spring beans are concrete classes without interfaces. CGLIB generates a subclass of the target class at runtime and overrides its methods to insert interception logic. This allows Spring AOP to work even when no interface is present, while still keeping business logic separate from infrastructure concerns.