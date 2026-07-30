# Spring AOP Internals

## 1. Why Spring AOP exists

Enterprise applications often need the same infrastructure behaviour around many business methods:

- Transaction management
- Security
- Logging
- Caching
- Metrics
- Retries
- Validation

Without AOP, that behaviour is repeated in business code:

```java
public void transfer() {
    authenticate();
    beginTransaction();
    log();
    businessLogic();
    commit();
}
```

This duplicates code, couples business logic to infrastructure, and makes maintenance harder. Spring AOP separates those cross-cutting concerns from the business method.

```text
Client
  |
  v
Proxy
  |-- Security
  |-- Transaction
  |-- Logging
  |-- Cache
  |-- Metrics
  `-- Retry
  |
  v
Business method
```

The infrastructure runs transparently before, after, or around the business method.

## 2. Core components

- **Aspect**: a developer-facing grouping of cross-cutting behaviour, commonly declared with `@Aspect`.

  ```java
  @Aspect
  class LoggingAspect {
      // advice methods
  }
  ```

- **Advice**: defines *what* runs, for example `@Before`, `@After`, `@Around`, `@AfterReturning`, or `@AfterThrowing`.
- **Pointcut**: defines *where* advice applies, such as methods annotated with `@Transactional`.
- **Advisor**: Spring's runtime representation of a pointcut plus advice. Spring AOP ultimately applies advisors to beans.
- **MethodInterceptor**: Spring's common runtime abstraction. Different advice types are adapted into interceptors that participate in one invocation chain.

## 3. Startup flow

1. The developer declares aspects or AOP-enabled annotations such as `@Transactional`.
2. Spring registers the required infrastructure, including advisors and an auto-proxy creator (a `BeanPostProcessor`). For transaction management, `@EnableTransactionManagement` imports this infrastructure.
3. As each eligible application bean is created, the auto-proxy creator checks whether any advisor matches it.
4. If an advisor matches, Spring creates an AOP proxy and exposes that proxy as the bean reference used for dependency injection and lookups.

```text
Application starts
  |
  v
Register AOP infrastructure
  |
  v
Create application bean
  |
  v
Auto-proxy creator checks matching advisors
  |
  +-- no match --> expose the target bean
  |
  `-- match ----> create and expose a proxy
```

The proxy delegates to a target object. Saying that the `ApplicationContext` *always* stores only proxies is too broad: a proxy is exposed when AOP applies, while Spring still manages the underlying target as part of the bean's creation and lifecycle.

## 4. Runtime flow

When a caller invokes a proxied bean, the call first reaches the proxy:

```text
Client
  |
  v
Proxy
  |
  v
Ordered interceptor chain
  |
  v
Target business method
```

For example, a `transfer()` call may execute logging, security, and transaction interceptors before the target method. Each interceptor can run behaviour before and after continuing the chain.

## 5. Around advice and `proceed()`

```java
@Around("execution(* com.example..*(..))")
public Object log(ProceedingJoinPoint joinPoint) throws Throwable {
    before();
    Object result = joinPoint.proceed();
    after();
    return result;
}
```

`joinPoint.proceed()` continues the invocation chain. It does not necessarily invoke the business method immediately; the target method is reached only after the remaining interceptors have proceeded.

## 6. Advisor ordering

Spring collects applicable advisors and builds an ordered interceptor chain when it creates the proxy. Use `@Order` or the `Ordered` interface when the order matters:

```java
@Order(1)
class SecurityAspect { }
```

Spring uses ordering rules, including `AnnotationAwareOrderComparator`, to determine the resulting order.

## 7. Self-invocation

```java
@Transactional
public void transfer() {
    debit();
}

@Transactional
public void debit() {
    // ...
}
```

The call to `debit()` is a call on the target instance (`this.debit()`); it does not leave the object and re-enter through the proxy. Consequently, proxy-based advice on `debit()` is skipped for that internal call.

```text
External caller --> proxy --> transfer() on target --> this.debit()
                                              ^
                                              `-- bypasses the proxy
```

Common solutions are to move the advised method to another bean, inject the bean's own proxy where appropriate, use `AopContext.currentProxy()` sparingly, or use AspectJ weaving when proxy-based AOP is not sufficient.

## 8. Transaction infrastructure

`@EnableTransactionManagement` registers transaction-management infrastructure, including a transaction advisor and interceptor. The auto-proxy creator must be available before application beans are initialized so it can create proxies for matching beans.

## 9. Important learnings

- Spring AOP is proxy-based by default.
- Spring applies matching advisors to eligible beans.
- Advice is executed through an ordered interceptor chain.
- Around advice controls whether and when the chain continues.
- Proxy creation happens during bean creation; calls use the prepared chain at runtime.
- Self-invocation bypasses proxy-based AOP.

## 10. Common mistakes

1. **Thinking annotations execute logic.** Annotations are metadata that Spring interprets.
2. **Thinking an aspect executes directly.** Spring turns applicable aspect definitions into runtime advisors and interceptors.
3. **Thinking `proceed()` always calls the target method.** It normally continues to the next interceptor first.
4. **Thinking `this.method()` is intercepted.** It bypasses the proxy.
5. **Thinking every Spring bean is a proxy.** Spring only creates a proxy for a bean when applicable AOP infrastructure requires one.

## 11. Interview questions

- **Why does Spring use proxies?** To apply cross-cutting behaviour without mixing it into business code.
- **Why are advice types adapted to `MethodInterceptor`?** A common interceptor model lets Spring execute different advice types in one chain.
- **Why can `@Transactional` fail during self-invocation?** The internal call bypasses the Spring proxy, so transaction advice is not invoked.
- **What is the difference between an aspect and an advisor?** An aspect is a developer-facing abstraction; an advisor is Spring's runtime pairing of advice and a pointcut.
- **Why does Spring create proxies during bean initialization?** It prepares interception once and exposes a bean reference that can apply the required advice at runtime.

## 12. Mental model

```text
Developer declaration
  |
  v
Aspect / transaction metadata
  |
  v
Advisor and auto-proxy infrastructure
  |
  v
Proxy for matching bean
  |
  v
Ordered interceptor chain
  |
  v
Target business method
```

## Summary

Spring AOP applies cross-cutting concerns through proxies. During bean creation, Spring converts applicable AOP metadata into advisors, finds matching beans, and exposes proxies for them. At runtime, external calls pass through an ordered interceptor chain before reaching the target method. Understanding proxy boundaries, ordering, and self-invocation explains the behaviour of `@Transactional`, `@Cacheable`, `@Async`, `@Retryable`, and other AOP-based features.
