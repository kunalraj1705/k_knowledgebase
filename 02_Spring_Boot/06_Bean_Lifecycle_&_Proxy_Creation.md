## Learning Objective
# By the end of this module, you should understand:
* How a Java object becomes a fully initialized Spring Bean.
* Why Spring introduced BeanPostProcessor.
* How Spring creates proxies.
* Difference between JDK Dynamic Proxy and CGLIB.
* Why @Transactional, @Cacheable, @Async, etc. work.
* Why self-invocation bypasses Spring AOP.
* Why component boundaries are important in Spring architecture

-------------------------------------------------------------------
# Problem Statement
Reflection has created the object.
UserService service = new UserService(repository);

Question: Is this immediately a Spring Bean?
Ans: No.

It is only a plain Java object.
Spring still needs to:
* initialize it
* allow extensions
* create proxies (if necessary)
* return the final object to the application

-----------------------------------------------------------------------
# Architecture
Bean Request
     │
     ▼
Dependency Resolution
     │
     ▼
Reflection
     │
     ▼
Java Object Created
     │
     ▼
@PostConstruct
     │
     ▼
BeanPostProcessor
     │
     ▼
Proxy Created (if required)
     │
     ▼
Bean Returned
-----------------------------------------------------------------------
# Stage 1 — Reflection Creates a Java Object
Reflection simply invokes the constructor.
UserService service = constructor.newInstance(repository);

At this stage:
* Dependencies exist
* Constructor has executed
* Object exists

But Spring has not yet finished processing it.

Stage 2 — @PostConstruct
Before exposing the object, Spring executes initialization callbacks.
Eg: 
@PostConstruct
public void init() {
   cache.load();
}

Purpose:
* Load configuration
* Initialize cache
* Validate state
* Prepare expensive resources

Only after initialization is complete can the bean be safely used.

Why?
Returning a partially initialized object could lead to:
* NullPointerException
* Invalid state
* Missing configuration
* Race conditions

Spring guarantees that every bean is fully initialized before it becomes available.

Stage 3 — BeanPostProcessor
Now Spring asks:
Should this bean be modified before it is returned?
Instead of embedding every feature inside BeanFactory, Spring delegates to extension points.

for (BeanPostProcessor processor : processors) {
   bean = processor.postProcess(bean);
}

Every processor gets an opportunity to inspect or replace the bean.

Why BeanPostProcessor Exists ?
Without extension points:
BeanFactory
├── Logging
├── Transactions
├── Security
├── Caching
├── Metrics
├── Validation
├── Async
├── Retry
└── …

The BeanFactory would become enormous.

Instead:
BeanFactory
     │
     ▼
BeanPostProcessor Chain
     │
     ├── Transaction Processor
     ├── Security Processor
     ├── Cache Processor
     ├── Async Processor
     └── Custom Processors

This keeps the framework open for extension while keeping the core simple.
Proxy Creation
* Suppose:
@Service
public class PaymentService {
   @Transactional
   public void transfer() {}
}

Spring does not usually return the original object.
* Instead:
	Developer
         │
	 ▼
   Transaction Proxy
	  │
     	  ▼
   Original PaymentService

The proxy adds behavior before and after the business method.

* Why Use a Proxy?
Instead of modifying your source code:
startTransaction();
transfer();
commitTransaction();

Spring wraps your object.
Conceptually:
proxy.transfer() {
   startTransaction();
   target.transfer();
   commitTransaction();
}

Your business class remains focused on business logic.

* Two Types of Proxies
1. JDK Dynamic Proxy
  PaymentService
  	  ▲
          │
 ┌──────┴────────┐
 │                     │
Impl         TransactionProxy

2. CGLIB Proxy
Used when no interface exists.
PaymentService
       ▲
       │
TransactionProxy

The proxy subclasses the original class and overrides methods.

--------------------------------------------------------------

# Java Constraints
Spring operates within Java's rules.
* Final Class
  * Eg:
    * final class PaymentService
  * Cannot be subclassed.
  * No CGLIB proxy can be created.
* Final Method
  * public final void transfer()
  * Cannot be overridden.
  * Proxy cannot intercept it.
* Private Method
  * Eg:
    * private void helper()
  * Private methods are not inherited.
  * They cannot be overridden.
  * Therefore they cannot be intercepted.
----------------------------------------------------------------
# Self Invocation
Consider:
@Service
public class OrderService {
   @Transactional
   public void placeOrder() {
       saveOrder();
   }

   @Transactional
   public void saveOrder() {
   }
}

Looks harmless.
But internally Java executes:
this.saveOrder();
Execution flow:
Developer
     │
     ▼
Proxy
     │
     ▼
placeOrder()
     │
     ▼
this.saveOrder()

The second call never passes through the proxy.

Result: saveOrder() is not intercepted.

---------------------------------------------------------------------
# Why Another Bean Works
Instead:
@Service
public class OrderService {
   @Autowired
   private OrderPersistenceService persistence;
   public void placeOrder() {
       persistence.saveOrder();
   }
}

Flow:
OrderService
     │
     ▼
OrderPersistenceService Proxy
     │
     ▼
saveOrder()

The call crosses a bean boundary. Therefore the proxy intercepts it.

# Self Injection
Some developers write:
@Autowired
private OrderService self;

Then:
self.saveOrder();

This works because Spring injects the proxy.
Flow:
Target Object
     │
     ▼
self
     │
     ▼
Proxy
     │
     ▼
saveOrder()

Although functional, this is generally considered a workaround.
The preferred solution is to separate responsibilities into different beans.

-----------------------------------------------------------------
# Public vs Private vs Self Invocation
| Scenario                     | Can Override? | Goes Through Proxy? | Intercepted? |
| ---------------------------- | ------------- | ------------------- | ------------ |
| Another bean → public method | ✅             | ✅                   | ✅            |
| `this.publicMethod()`        | ✅             | ❌                   | ❌            |
| `privateMethod()`            | ❌             | ❌                   | ❌            |
| `finalMethod()`              | ❌             | Depends             | ❌            |

----------------------------------------------------------------------------
# Internal Working Summary
BeanFactory
↓
Resolve Dependencies
↓
Reflection
↓
Java Object
↓
@PostConstruct
↓
BeanPostProcessor Chain
↓
Proxy Created (Optional)
↓
Return Bean
↓
Singleton Cache
------------------------------------------------------------------------------
# Mental Models
1. Bean Lifecycle
Constructor
↓
Java Object
↓
Initialization
↓
Enhancement
↓
Ready Bean

2. Proxy Pattern
Developer
↓
Proxy
↓
Original Object

Developer never interacts with the original object directly.

3. Self Invocation
Proxy
↓
Original Object
↓
this.method()

The proxy has already delegated execution.
Further calls stay inside the original object.
-------------------------------------------------------------------------------
# Interview Answer
Spring first creates a Java object using Reflection. At this point, it is not yet a fully managed Spring Bean. The framework executes lifecycle callbacks like @PostConstruct to complete initialization. Next, Spring passes the bean through a chain of BeanPostProcessors, which can inspect or replace the bean. If annotations such as @Transactional are detected, Spring creates a proxy instead of returning the original object. The proxy intercepts method calls to add cross-cutting behavior such as transactions, security, caching, or logging. Spring uses JDK Dynamic Proxies for interface-based beans and CGLIB subclass proxies for concrete classes. Because proxies rely on Java inheritance or interfaces, they cannot intercept final or private methods. Additionally, self-invocation (this.method()) bypasses the proxy, which is why proxy-based features work best across bean boundaries rather than within the same class.

