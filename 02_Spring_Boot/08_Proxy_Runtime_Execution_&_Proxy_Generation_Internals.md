## Learning Objective
Understand how Spring creates proxies, executes interceptor chains, chooses between JDK Dynamic Proxy and CGLIB, and why self-invocation bypasses Spring AOP.


# Module Overview
Client

  Ã¢â€â€š

  Ã¢â€“Â¼

Proxy

  Ã¢â€â€š

  Ã¢â€“Â¼

Interceptor Chain

  Ã¢â€â€š

  Ã¢â€“Â¼

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

  Ã¢â€â€š

  Ã¢â€“Â¼

Proxy.transfer()


---
# 2. Why a Proxy?
Without Proxy

Client

  Ã¢â€â€š

  Ã¢â€“Â¼

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

  Ã¢â€â€š

  Ã¢â€“Â¼

Proxy

  Ã¢â€â€š

  Ã¢â€“Â¼

Interceptors

  Ã¢â€â€š

  Ã¢â€“Â¼

Business Method


The proxy becomes the execution gateway.


---
# 3. Interceptor Chain
Suppose

@Transactional

@Cacheable


Spring creates

Proxy

  Ã¢â€â€š

  Ã¢â€“Â¼

CacheInterceptor

  Ã¢â€â€š

  Ã¢â€“Â¼

TransactionInterceptor

  Ã¢â€â€š

  Ã¢â€“Â¼

PaymentService


**Each interceptor performs**

Before Logic

Ã¢â€ â€œ

Next

Ã¢â€ â€œ

After Logic


**Conceptually:**

before();

Object result = invocation.proceed();

after();

return result;


Every interceptor follows exactly the same algorithm.


---
# 4. Chain of Responsibility Pattern
Each interceptor never knows the complete chain.

it only knows invocation.proceed();


**Execution:**

Security

   Ã¢â€â€š

   Ã¢â€“Â¼

Cache

   Ã¢â€â€š

   Ã¢â€“Â¼

Transaction

   Ã¢â€â€š

   Ã¢â€“Â¼

Target


If an interceptor decides Stop then the chain terminates


Examples:

1. Cache Hit
2. Authorization Failure
3. Rate Limit
4. Circuit Breaker


---
# 5. ReflectiveMethodInvocation
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
# 6. Why currentInterceptorIndex Starts at -1
Spring initializes currentInterceptorIndex = -1; instead of 0


Reason:

Every invocation follows exactly the same algorithm

{

 index++;

 invoke(interceptor\[index]);

}


Execution:

\-1

Ã¢â€â€š

Ã¢â€“Â¼

0 Ã¢â€ â€™ Security

1 Ã¢â€ â€™ Cache

2 Ã¢â€ â€™ Transaction

3 Ã¢â€ â€™ Target


---
# 7. proceed()
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
# 8. Exception Propagation
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
# 9. Runtime Execution
Client

Ã¢â€ â€œ

Proxy

Ã¢â€ â€œ

Security

Ã¢â€ â€œ

Cache

Ã¢â€ â€œ

Transaction

Ã¢â€ â€œ

Business Method

Ã¢â€ â€œ

Transaction Commit

Ã¢â€ â€œ

Return

Ã¢â€ â€œ

Client


---
# 10. JDK Dynamic Proxy
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
# 11. Why JDK Proxy Cannot Proxy Classes
public class PaymentService {


}


JDK would need

class Proxy implements PaymentService


This is not possible because Class Ã¢â€°Â  Interface


Java allows

* extends Class
* implements Interface


Java doesn't allow

* implements Class


Therefore

JDK Proxy only works for interfaces.


---
# 12. CGLIB
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
# 13. Final Class Limitation
Suppose:

public final class PaymentService {


}


CGLIB tries:

class PaymentService$$Enhancer extends PaymentService


This fails because JAVA forbids extending final class

Result: No Proxy


---
# 14. Final Method Limitation
Suppose:

public class PaymentService {

   public final void transfer(){

   }

}


CGLIB tries to override transfer() but it fails because JAVA doesn't allow overriding final method.

Result: No Proxy


---
# 15. Spring's Proxy Selection
Decision Tree

Does Bean Implement Interface?


       Ã¢â€â€š

  Ã¢â€Å’Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€Â´Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€Â

  Ã¢â€â€š          Ã¢â€â€š

 Yes         No

  Ã¢â€â€š          Ã¢â€â€š

  Ã¢â€“Â¼          Ã¢â€“Â¼

JDK Proxy   CGLIB


Spring prefers the simplest mechanism that satisfies the requirement.


---
# 16. Forcing CGLIB
Spring allows

* EnableTransactionManagement(proxyTargetClass = true)
* @EnableAspectJAutoProxy(proxyTargetClass = true)


Even if interfaces exist Spring always generates CGLIB proxies.


---
# 17. Injection Differences
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
# 18. Self Invocation
Suppose:

public void pay(){

   this.transfer();

}


Execution:

pay()

Ã¢â€ â€œ

this.transfer()

Ã¢â€ â€œ

Original Method


The proxy is bypassed. No interceptor executes.

Because proxy will work on the external call (incoming calls from outside of the proxy class)


Therefore:

@Transactional

@Cacheable

@Async


do not work.


---
# 19. Why?
Spring AOP intercepts Calls entering the proxy

It does NOT monitor every JVM method invocation.

Once execution reaches the target object


this.transfer() is a normal Java call.


---
# 20. Self Injection Workaround
@Autowired

private PaymentService self;


Execution:

pay()

Ã¢â€ â€œ

self.transfer()

Ã¢â€ â€œ

Proxy

Ã¢â€ â€œ

Interceptors

Ã¢â€ â€œ

Target


Because:

self = Proxy


---
# 21. Runtime Class Types
JDK:

paymentService.getClass()


return something similar to: jdk.proxy2.$Proxy45


CGLIB:

returns: PaymentService$$EnhancerBySpringCGLIB


The injected bean is not the original implementation class.

It is the generated proxy.


---
# End-to-End Architecture
               Spring Startup


                      Ã¢â€â€š

                      Ã¢â€“Â¼

             Scan Bean Definitions

                      Ã¢â€â€š

                      Ã¢â€“Â¼

            Detect AOP Annotations

                      Ã¢â€â€š

                      Ã¢â€“Â¼

             Build Advisor List

                      Ã¢â€â€š

                      Ã¢â€“Â¼

        Bean Implements Interface?

              Ã¢â€â€š               Ã¢â€â€š

            Yes               No

             Ã¢â€â€š                Ã¢â€â€š

             Ã¢â€“Â¼                Ã¢â€“Â¼

       JDK Dynamic        CGLIB Proxy

           Proxy            Generation

              Ã¢â€â€š                Ã¢â€â€š

              Ã¢â€â€Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€Â¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€â‚¬Ã¢â€Ëœ

                     Ã¢â€“Â¼

              Create Proxy Bean

                     Ã¢â€â€š

                     Ã¢â€“Â¼

           Store Interceptor Chain

                     Ã¢â€â€š

                     Ã¢â€“Â¼


========================================


            Runtime Execution


Client

  Ã¢â€â€š

  Ã¢â€“Â¼

Proxy

  Ã¢â€â€š

  Ã¢â€“Â¼

Security

  Ã¢â€â€š

  Ã¢â€“Â¼

Cache

  Ã¢â€â€š

  Ã¢â€“Â¼

Transaction

  Ã¢â€â€š

  Ã¢â€“Â¼

Business Method

  Ã¢â€â€š

  Ã¢â€“Â¼

Return Result


---
CGLIB exists to overcome the limitation of JDK Dynamic Proxy. JDK proxies can only create proxy implementations for interfaces, whereas many Spring beans are concrete classes without interfaces. CGLIB generates a subclass of the target class at runtime and overrides its methods to insert interception logic. This allows Spring AOP to work even when no interface is present, while still keeping business logic separate from infrastructure concerns.
