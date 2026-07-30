# Proxy Runtime Execution and Proxy Generation Internals

## Purpose

For an AOP-enabled bean, a caller invokes a proxy rather than calling the target method directly. The proxy runs an ordered interceptor chain and eventually invokes the target method.

## Runtime execution

```text
Client
  |
  v
Proxy
  |
  v
Security interceptor
  |
  v
Cache interceptor
  |
  v
Transaction interceptor
  |
  v
Target method
```

The actual order depends on the applicable advisors and their ordering. A cache hit, security failure, or other interceptor can return or throw without continuing to the target.

## `proceed()` and the invocation chain

Spring represents one method call with an invocation object containing the target, method, arguments, and ordered interceptors. In simplified form:

```java
Object proceed() throws Throwable {
    if (hasMoreInterceptors()) {
        return nextInterceptor().invoke(this);
    }
    return invokeTargetMethod();
}
```

An interceptor usually performs work before and after `proceed()`:

```java
beginTransaction();
try {
    Object result = invocation.proceed();
    commit();
    return result;
} catch (Throwable ex) {
    rollback();
    throw ex;
}
```

This is a chain-of-responsibility design. Each interceptor only needs to decide whether and when to continue; it does not need to know the full chain.

## Proxy strategies

### JDK dynamic proxy

A JDK proxy implements interfaces and delegates calls to an invocation handler. It can be injected by an interface type, but it is not an instance of the concrete implementation class.

### Class-based proxy

A class-based proxy subclasses the target class and overrides interceptable methods. It can be used when class proxying is selected, but cannot subclass `final` classes or override `final` methods. Private methods are not interceptable by either strategy.

Spring can be configured to force class-based proxies, for example with `proxyTargetClass = true`. In modern Spring Boot applications, check the actual AOP configuration rather than assuming a strategy from the target's interfaces alone.

## Self-invocation

```java
public void pay() {
    this.transfer();
}
```

Once `pay()` has entered the target object, `this.transfer()` is an ordinary Java call on that object. It does not return to the proxy, so proxy-based advice on `transfer()` is skipped. Extracting `transfer()` to another bean is usually the clearest solution.

## Inspecting a proxy

The runtime class can reveal that a bean is proxied, but application code should not rely on generated class names. Use Spring AOP utilities when diagnostic code needs to determine whether an object is an AOP proxy.

## Key takeaway

The proxy is the boundary where AOP starts. External calls cross that boundary, run the applicable interceptor chain, and then reach the target; internal calls on `this` do not.
