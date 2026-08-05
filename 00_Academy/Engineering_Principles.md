# Engineering Principles



##EP-001: A strong foundation reduces complexity later Understanding a technology is more valuable than memorizing its syntax.

##EP-002: Separate Discovery from Creation

##EP-003: Separate Metadata from Runtime State

##EP-004: Store Behavior, Not Unnecessary State

##EP-005: High-Level Components Coordinate, Low-Level Components Execute

##EP-006: Hide complexity behind a simple interface.

##EP-007: Solve complex object graphs by recursively solving smaller dependency problems.

##EP-008: Build complex systems by layering responsibilities.

##EP-009: A coordinator should orchestrate work, not perform specialized work.

##EP-010: Frameworks Operate on Metadata

##EP-011: Separate Decision-Making from Execution

##EP-012: Never expose an object before it is fully initialized.

##EP-013: Frameworks should provide extension points instead of modifying core components for every new feature.

##EP-014: Enhance behavior by wrapping an object instead of modifying its source code (Proxy Pattern).

##EP-015: Separate Business Logic from Cross-Cutting Concerns

* Business classes should focus only on domain logic.
* Common concerns like transactions, logging, security, caching, and monitoring should be implemented outside the business class.
* Spring achieves this by wrapping beans with proxies created through BeanPostProcessors, allowing additional behavior without modifying the original source code.

##EP-016: Extension mechanisms are constrained by the capabilities of the underlying language.

##EP-017: Interception only occurs at object boundaries. Internal method calls bypass external interceptors.

##EP-018: Framework features are most effective when component boundaries align with framework interception points.

##EP-019: Prefer clear component boundaries over framework-specific workarounds.

##EP-020: A proxy can intercept only methods that the language allows it to override or dispatch through the proxy.

##EP-021: Parse once. Cache metadata. Reuse repeatedly.

##EP-022: Separate bean creation from bean enhancement.

##EP-023: Represent cross-cutting concerns as an ordered chain of independent interceptors rather than nested proxy objects.

##EP-024: Separate where logic applies from what logic executes.

##EP-025: Share behavior; pass context.

##EP-026: Initialization prepares the object itself; post-processing allows the framework to enhance or replace the object.

##EP-027: Annotations are declarative metadata, not executable behavior.

##EP-028: Separate method selection from behavior execution.

##EP-029: Represent framework behavior as reusable rules rather than per-method configurations.

##EP-030: Design framework infrastructure around abstractions, not concrete features

##EP-031: Move execution state into a context object rather than distributing it across participants.

##EP-032: Centralize traversal logic so individual interceptors remain simple.

##EP-033: Represent one method invocation with a dedicated context object (ReflectiveMethodInvocation).

##EP-034: Design algorithms so every iteration follows the same path; eliminate special cases.

##EP-035: Restore system consistency locally (rollback) while preserving failure information globally (rethrow).

##EP-036: Recursive invocation naturally propagates execution, return values, and exceptions through the interceptor chain.

##EP-037: Proxy compatibility follows Java's type system, not Spring's dependency injection rules.

##EP-038: Interception occurs at object boundaries, not method boundaries.

##EP-039: A proxy preserves the public contract of an object, but not necessarily its concrete implementation type.

##EP-040: Generate repetitive infrastructure code at runtime instead of requiring developers to maintain it manually.

##EP-041: When an abstraction is too restrictive, extend the framework with another mechanism rather than forcing developers to change their design.

##EP-042: AOP modularizes cross-cutting concerns.

##EP-043: Annotations describe behaviour; framework infrastructure implements it.

##EP-044: The JVM provides metadata; Spring gives it meaning.

##EP-045: Advisors decide whether an advice applies; Interceptors define how it executes.

##EP-046: Aspects are for developers. Advisors are for Spring.

##EP-047: Separate the developer programming model from the runtime execution model.

##EP-048: Spring first decides whether (Advisor/Pointcut), then how (Advice → MethodInterceptor), then composes interceptors into an Interceptor Chain.

##EP-049: Different programming models become one runtime execution model.

##EP-050: Around Advice controls continuation of the interceptor chain.

##EP-051: Order cross-cutting concerns from least expensive to most expensive so invalid requests are rejected before costly resources are allocated.

##EP-052: Interceptors that can short-circuit execution should execute before expensive operations like transactions and database access.

##EP-053: Proxy-based AOP intercepts only calls entering through the proxy.

##EP-054: Proxy-based AOP can only intercept calls that pass through the proxy. Designing services with clear boundaries naturally allows AOP to work without special workarounds.

##EP-055: In proxy-based frameworks, interception depends on the object receiving the call, not the location of the calling code.

##EP-056: Expensive discovery happens during startup, not during runtime.

##EP-057: Framework infrastructure must be established before application components are processed, because enhancement happens during object creation—not afterward.

##EP-058: An event represents something that has already happened, not a request that must be handled.

##EP-059: Frameworks preprocess metadata during startup and execute precomputed decisions during runtime.

##EP-060: Runtime execution should rely on precomputed metadata rather than repeated discovery.

##EP-061: Frameworks expose a simple public API while delegating execution to specialized internal components.

##EP-063: Reuse existing extension points instead of introducing new lifecycle phases.

##EP-064: Manage objects as units of lifecycle and methods as units of behavior.

##EP-065: Wrap reflection metadata inside richer abstractions.

##EP-068: Dispatchers coordinate execution; adapters encapsulate invocation logic.

##EP-069: Always retrieve managed objects through the IoC container.

##EP-070: Trade startup work for runtime efficiency.

##EP-071: Build upon the language's type system.

##EP-073: Compose independent subsystems through extension points.

##EP-074: Expose lifecycle events so independent components can react without coupling.

##EP-075: Core framework components should remain unaware of extension-specific behavior.

##EP-076: Frameworks centralize infrastructure concerns such as thread management so developers can focus on business logic.

##EP-077: Frameworks maximize consistency by reusing existing lifecycle extension points instead of inventing new processing pipelines.

##EP-078: Interceptors coordinate execution while delegating resource management to specialized infrastructure components.

##EP-079: Proxy-based features apply only to calls entering through the proxy.

##EP-080: The IoC container manages proxy objects as the public representation of beans.

##EP-081: Asynchronous execution separates caller and worker thread control flow.

##EP-082: A Future represents a promise for a result that is not yet available.

##EP-083: Thread-bound infrastructure must be initialized in the thread that performs the work.

##EP-084: Execution frameworks absorb short bursts through bounded queues and reject excess work to preserve stability.

##EP-085: Bounded systems fail gracefully by applying backpressure.

##EP-086: System throughput is constrained by the slowest downstream dependency.

##EP-087: Framework features should be opt-in.

##EP-088: Framework annotations declare intent while imported configuration builds infrastructure.

##EP-089: Frameworks should separate business logic from infrastructure.

##EP-090: Heavy computation belongs during startup.

##EP-091: Runtime should operate using precomputed metadata.

##EP-092: Shared infrastructure should manage execution resources.

##EP-093: Scheduling strategy should match business requirements.

##EP-094: Infrastructure failures should remain isolated from business failures.

##EP-095: Application-level scheduling works only within a single JVM. Distributed coordination requires external mechanisms.

##EP-096: Frameworks should separate business logic from infrastructure.

##EP-097: Heavy computation belongs during startup.

##EP-098: Runtime should operate using precomputed metadata.

##EP-099: Shared infrastructure should manage execution resources.

##EP-100: Scheduling strategy should match business requirements.

##EP-101: Infrastructure failures should remain isolated from business failures.

##EP-102: Application-level scheduling works only within a single JVM. Distributed coordination requires external mechanisms.

##EP-103: Caching is a trade-off between memory and computation.

##EP-104: Database remains the source of truth.

##EP-105: Frameworks should move expensive work to startup.

##EP-106: Runtime should operate using precomputed infrastructure.

##EP-107: Cache keys must uniquely identify cached results.

##EP-108: Cache consistency should be designed based on business requirements.

##EP-109: Shared infrastructure should manage cross-cutting concerns instead of business code.

##EP-110: Separate HTTP infrastructure from business logic.

##EP-111: Infrastructure should exist before metadata registration.

##EP-112: Perform expensive discovery during startup.

##EP-113: Runtime should rely on precomputed metadata.

##EP-114: Delegate specialized responsibilities to dedicated components.

##EP-115: Controllers should operate on domain objects, not HTTP details.

##EP-116: Cross-cutting concerns belong to the appropriate layer: Filter → Servlet Layer, Interceptor → MVC Layer, AOP → Business Layer.

##EP-117: Frameworks should prepare execution context before invoking business code.

##EP-118: Reject invalid requests as early as possible.

##EP-119: Separate security from business logic.

##EP-120: Authentication and authorization are different responsibilities.

##EP-121: One component should own one responsibility.

##EP-122: Use delegation instead of monolithic implementations.

##EP-123: Perform expensive work once and reuse the result.

##EP-124: Store request-scoped security information using ThreadLocal.

##EP-125: Infrastructure first. Metadata later. Runtime lookup finally.

##EP-126: Protect HTTP requests using Filters. Protect business methods using AOP.

