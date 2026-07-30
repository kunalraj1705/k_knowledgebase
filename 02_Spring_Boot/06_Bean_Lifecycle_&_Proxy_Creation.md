# Bean Lifecycle and Proxy Creation

## Purpose

A Java object becomes a usable Spring bean only after the container has completed dependency injection, initialization, and post-processing. A post-processor can replace the original object with a proxy when an infrastructure feature, such as transactions, applies.

## Lifecycle overview

```text
Resolve dependencies
  |
  v
Instantiate target object
  |
  v
Populate properties
  |
  v
Initialization callbacks and post-processors
  |
  v
Create proxy when AOP applies
  |
  v
Expose singleton or return scoped instance
```

The exact ordering includes several callbacks; the important point is that a bean is not simply the object returned by its constructor.

## Initialization and `BeanPostProcessor`

After property population, Spring invokes initialization mechanisms such as `@PostConstruct`, `InitializingBean`, and custom init methods. `BeanPostProcessor`s run before and after initialization and can inspect, wrap, or replace a bean.

```java
@PostConstruct
void initialize() {
    // validate state or prepare resources
}
```

`@PostConstruct` belongs to the target bean's lifecycle. A `BeanPostProcessor` is a container extension point. Auto-proxy creators are bean post-processors that typically create their proxy after initialization.

## Proxy creation

For a matching bean, Spring exposes a proxy that delegates to a target:

```text
Caller --> proxy --> interceptors --> target method
```

The proxy applies cross-cutting behaviour such as transactions, caching, security, or asynchronous execution without adding that code to the business method. It is optional: beans with no applicable advisor are normally exposed directly.

## JDK proxies and class-based proxies

- **JDK dynamic proxies** implement one or more interfaces. They are often used when proxying interfaces.
- **Class-based proxies** subclass the target class (Spring Framework uses a repackaged CGLIB implementation). They are needed when an interface proxy cannot satisfy the required proxy type, or when class proxying is explicitly enabled.

Class-based proxies cannot intercept `final` methods and cannot subclass `final` classes. Neither proxy style intercepts private methods. The actual proxy strategy may be configured and should not be guessed solely from whether a class declares an interface.

## `@PostConstruct` and AOP

Calling an AOP-advised method from the same bean's `@PostConstruct` generally does not invoke the final proxy. The bean is being initialized and the call is made on the target object, so proxy advice such as `@Transactional` is not applied. Put transactional startup work in an appropriate event listener or invoke it through another managed bean when needed.

## Self-invocation

```text
External caller --> proxy --> placeOrder() --> this.saveOrder()
                                            ^
                                            `-- skips the proxy
```

An internal `this.saveOrder()` call bypasses proxy-based advice. Prefer extracting the advised operation into a separate bean; self-injection and `AopContext.currentProxy()` are narrower workarounds.

## Key takeaway

Spring lifecycle processing prepares the object, while proxying changes the reference the rest of the application uses for matching AOP-enabled beans.
