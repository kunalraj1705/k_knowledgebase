**## Learning Objective**

Understand how Spring creates proxies, executes interceptor chains, chooses between JDK Dynamic Proxy and CGLIB, and why self-invocation bypasses Spring AOP.



**# Module Overview**

Client

&#x20;  │

&#x20;  ▼

Proxy

&#x20;  │

&#x20;  ▼

Interceptor Chain

&#x20;  │

&#x20;  ▼

Target Method



This module answers four important questions:

1. How does Spring execute a proxied method?
2. How does Spring create proxy classes?
3. Why are there JDK Proxy and CGLIB?
4. Why does self-invocation fail?



Note:

CGLIB - Code Generation Library

AOP - Aspect-Oriented Programming



\---------------------------------------------------------------------

**# 1. Calling a Spring Bean**

Suppose we have

@Service

public class PaymentService {



&#x20;   @Transactional

&#x20;   @Cacheable

&#x20;   public void transfer() {

&#x20;       System.out.println("Business Logic");

&#x20;   }

}



Spring injects

@Autowired

private PaymentService paymentService;



Although it appears to be a normal object...

It is actually **Proxy**

so paymentService.transfer(); does not call PaymentService.transfer()

instead

Client

&#x20;  │

&#x20;  ▼

Proxy.transfer()



\----------------------------------------------------------------------

**# 2. Why a Proxy?**

Without Proxy

Client

&#x20;  │

&#x20;  ▼

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

&#x20;  │

&#x20;  ▼

Proxy

&#x20;  │

&#x20;  ▼

Interceptors

&#x20;  │

&#x20;  ▼

Business Method



The proxy becomes the execution gateway.



\--------------------------------------------------------------------------

**# 3. Interceptor Chain**

Suppose

@Transactional

@Cacheable



Spring creates

Proxy

&#x20;  │

&#x20;  ▼

CacheInterceptor

&#x20;  │

&#x20;  ▼

TransactionInterceptor

&#x20;  │

&#x20;  ▼

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



\-------------------------------------------------------------------------

**# 4. Chain of Responsibility Pattern**

Each interceptor never knows the complete chain.

it only knows invocation.proceed();



**Execution:**

Security

&#x20;   │

&#x20;   ▼

Cache

&#x20;   │

&#x20;   ▼

Transaction

&#x20;   │

&#x20;   ▼

Target



If an interceptor decides Stop then the chain terminates



Examples:

1. Cache Hit
2. Authorization Failure
3. Rate Limit
4. Circuit Breaker



\----------------------------------------------------------------------

**# 5. ReflectiveMethodInvocation**

Spring stores invocation state inside ReflectiveMethodInvocation



**Conceptually:**

class ReflectiveMethodInvocation {

&#x20;   Object target;

&#x20;   Method method;

&#x20;   Object\[] arguments;

&#x20;   List<MethodInterceptor> interceptors;

&#x20;   int currentInterceptorIndex;

}



This object represents One method invocation Not One bean



\------------------------------------------------------------------

**# 6. Why currentInterceptorIndex Starts at -1**

Spring initializes currentInterceptorIndex = -1; instead of 0



Reason:

Every invocation follows exactly the same algorithm

{

&#x20; index++;

&#x20; invoke(interceptor\[index]);

}



Execution:

\-1

&#x20;│

&#x20;▼

0 → Security

1 → Cache

2 → Transaction

3 → Target



\---------------------------------------------------------

**# 7. proceed()**

Conceptually:

Object proceed() {

&#x20;   currentInterceptorIndex++;

&#x20;   if(moreInterceptors){

&#x20;       invoke(nextInterceptor);

&#x20;   }else{

&#x20;       invokeTargetMethod();

&#x20;   }

}



Every interceptor simply executes

invocation.proceed();



It never knows

* previous interceptor
* next interceptor
* total interceptor count

Traversal is centralized.



\--------------------------------------------------------

**# 8. Exception Propagation**

Transaction interceptor 



beginTransaction();

try{

&#x20;   Object result = invocation.proceed();

&#x20;   commit();

}

catch(Exception e){

&#x20;   rollback();

&#x20;   throw e;

}



Why rethrow?

Because the caller must know The operation failed.



Otherwise

paymentService.transfer();

System.out.println("Success");



\---------------------------------------------------------------

**# 9. Runtime Execution**

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



\--------------------------------------------------------------------

**# 10. JDK Dynamic Proxy**

Suppose

public interface PaymentService {

&#x20;   void transfer();

}



Implementation:

public class PaymentServiceImpl implements PaymentService {

}



JDK creates

class $Proxy0 implements PaymentService {

&#x20;   InvocationHandler handler;

}



It implements the interface



\---------------------------------------------------------------------

**# 11. Why JDK Proxy Cannot Proxy Classes**

public class PaymentService {



}



JDK would need 

class Proxy implements PaymentService



This is not possible because Class ≠ Interface



Java allows

* extends Class
* implements Interface



Java doest allow

* implements Class



Therefore

JDK Proxy only works for interfaces.



\-------------------------------------------------------------

**# 12. CGLIB**

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

&#x20;   before();

&#x20;   super.transfer();

&#x20;   after();

}



\----------------------------------------------------------------

**# 13. Final Class Limitation**

Suppose: 

public final class PaymentService {



}



CGLIB tries:

class PaymentService$$Enhancer extends PaymentService



This fails because JAVA forbids extending final class

Result: No Proxy



\------------------------------------------------------------------

**# 14. Final Method Limitation**

Suppose:

public class PaymentService {

&#x20;   public final void transfer(){

&#x20;   }

}



CGLIB tries to override transfer() but it fails because JAVA doesn't allow overriding final method.

Result: No Proxy



\---------------------------------------------------------------------

**# 15. Spring's Proxy Selection**

Decision Tree

Does Bean Implement Interface?



&#x20;       │

&#x20;  ┌────┴─────┐

&#x20;  │          │

&#x20; Yes         No

&#x20;  │          │

&#x20;  ▼          ▼

JDK Proxy   CGLIB



Spring prefers the simplest mechanism that satisfies the requirement.



\---------------------------------------------------------------------------------

**# 16. Forcing CGLIB**

Spring allows

* EnableTransactionManagement(proxyTargetClass = true)
* @EnableAspectJAutoProxy(proxyTargetClass = true)



Even if interfaces exist Spring always generates CGLIB proxies.



\----------------------------------------------------------------------------

**# 17. Injection Differences**

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



\-------------------------------------------------------------------------

**# 18. Self Invocation**

Suppose:

public void pay(){

&#x20;   this.transfer();

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



\-------------------------------------------------------------------------------

**# 19. Why?**

Spring AOP intercepts Calls entering the proxy 

It does NOT monitor every JVM method invocation.

Once execution reaches the target object



this.transfer() is a normal Java call.



\-----------------------------------------------------------------------------

**# 20. Self Injection Workaround**

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



\---------------------------------------------------------------

**# 21. Runtime Class Types**

JDK:

paymentService.getClass()



return something similar to: jdk.proxy2.$Proxy45



CGLIB:

returns: PaymentService$$EnhancerBySpringCGLIB



The injected bean is not the original implementation class.

It is the generated proxy.



\------------------------------------------------------------

**# End-to-End Architecture**

&#x20;               Spring Startup



&#x20;                      │

&#x20;                      ▼

&#x20;             Scan Bean Definitions

&#x20;                      │

&#x20;                      ▼

&#x20;            Detect AOP Annotations

&#x20;                      │

&#x20;                      ▼

&#x20;             Build Advisor List

&#x20;                      │

&#x20;                      ▼

&#x20;        Bean Implements Interface?

&#x20;              │               │

&#x20;            Yes               No

&#x20;             │                │

&#x20;             ▼                ▼

&#x20;       JDK Dynamic        CGLIB Proxy

&#x20;           Proxy            Generation

&#x20;              │                │

&#x20;              └──────┬─────────┘

&#x20;                     ▼

&#x20;              Create Proxy Bean

&#x20;                     │

&#x20;                     ▼

&#x20;           Store Interceptor Chain

&#x20;                     │

&#x20;                     ▼



========================================



&#x20;            Runtime Execution



Client

&#x20;  │

&#x20;  ▼

Proxy

&#x20;  │

&#x20;  ▼

Security

&#x20;  │

&#x20;  ▼

Cache

&#x20;  │

&#x20;  ▼

Transaction

&#x20;  │

&#x20;  ▼

Business Method

&#x20;  │

&#x20;  ▼

Return Result



\--------------------------------------------------------------

CGLIB exists to overcome the limitation of JDK Dynamic Proxy. JDK proxies can only create proxy implementations for interfaces, whereas many Spring beans are concrete classes without interfaces. CGLIB generates a subclass of the target class at runtime and overrides its methods to insert interception logic. This allows Spring AOP to work even when no interface is present, while still keeping business logic separate from infrastructure concerns.

