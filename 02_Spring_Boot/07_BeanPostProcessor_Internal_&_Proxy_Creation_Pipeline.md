# BeanPostProcessor Internals and the Proxy Creation Pipeline

## Purpose

`BeanPostProcessor` is Spring's extension point for participating in bean creation. It keeps features such as AOP, validation, and annotation-driven injection out of the core creation algorithm.

## Bean post-processors

A post-processor can run before and after initialization and can return the original bean or a replacement:

```java
Object postProcessAfterInitialization(Object bean, String beanName) {
    return bean;
}
```

Examples include processors for `@Autowired`, lifecycle annotations, and auto-proxying. Not every post-processor creates a proxy.

## AOP building blocks

| Building block | Meaning |
| --- | --- |
| Advice | What should run, such as transaction or cache behaviour. |
| Pointcut | Which classes or methods the advice applies to. |
| Advisor | A runtime pairing of advice and a pointcut. |
| Method interceptor | The common invocation form used to execute advice in a chain. |

For example, transaction management registers an advisor whose pointcut identifies transactional methods and whose advice is a transaction interceptor.

## Proxy creation pipeline

```text
Register application and infrastructure BeanDefinitions
  |
  v
Create target bean and perform initialization
  |
  v
Auto-proxy creator evaluates eligible advisors
  |
  +-- no matching advisor --> expose target
  |
  `-- matching advisor(s)
        |
        v
      order interceptors
        |
        v
      create one proxy for the bean
        |
        `-- expose proxy
```

`AnnotationAwareAspectJAutoProxyCreator` is a general auto-proxy creator used for annotation-style Spring AOP. It works with advisors; individual infrastructure modules contribute the advisors they need.

One proxy can hold an ordered chain of several interceptors. This is a useful model, but it is not an absolute promise that no additional proxy layer can ever exist: custom infrastructure or nested contexts can introduce one. Avoid depending on proxy layering as an application contract.

## Metadata timing

Spring parses and caches substantial metadata during configuration processing, then reuses it when creating beans. Some matching and introspection is still performed as part of bean creation; the safe mental model is “metadata is prepared and reused,” not “all annotations are scanned exactly once.”

## `@PostConstruct` versus post-processing

| `@PostConstruct` | `BeanPostProcessor` |
| --- | --- |
| Bean-specific initialization callback | Container extension mechanism |
| Runs after dependency injection | Can run before and after initialization |
| Does not replace the bean reference | Can wrap or replace the reference |

Because auto-proxying usually happens after initialization, calling an advised method from `@PostConstruct` does not normally traverse the resulting proxy.

## Key takeaway

Spring AOP works because infrastructure registers advisors and an auto-proxy creator. During bean creation, the auto-proxy creator turns matching advisors into an ordered interceptor chain on the bean's exposed proxy.
