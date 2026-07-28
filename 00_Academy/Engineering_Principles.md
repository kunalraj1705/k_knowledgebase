\# Engineering Principles



\## EP-001

A strong foundation reduces complexity later

Understanding a technology is more valuable than memorizing its syntax.





\##EP-002: Separate Discovery from Creation

\##EP-003: Separate Metadata from Runtime State

\##EP-004: Store Behavior, Not Unnecessary State

\##EP-005: High-Level Components Coordinate, Low-Level Components Execute

\##EP-006: Hide complexity behind a simple interface.

\##EP-007: Solve complex object graphs by recursively solving smaller dependency problems.

\##EP-008: Build complex systems by layering responsibilities.

\##EP-009: A coordinator should orchestrate work, not perform specialized work.

\##EP-010: Frameworks Operate on Metadata

\##EP-011: Separate Decision-Making from Execution

\##EP-012: Never expose an object before it is fully initialized.

\##EP-013: Frameworks should provide extension points instead of modifying core components for every new feature.

\##EP-014: Enhance behavior by wrapping an object instead of modifying its source code (Proxy Pattern).

\##EP-015: Separate Business Logic from Cross-Cutting Concerns

* Business classes should focus only on domain logic.
* Common concerns like transactions, logging, security, caching, and monitoring should be implemented outside the business class.
* Spring achieves this by wrapping beans with proxies created through BeanPostProcessors, allowing additional behavior without modifying the original source code.

\##EP-016: Extension mechanisms are constrained by the capabilities of the underlying language.

\##EP-017: Interception only occurs at object boundaries. Internal method calls bypass external interceptors.

\##EP-018: Framework features are most effective when component boundaries align with framework interception points.

\##EP-019: Prefer clear component boundaries over framework-specific workarounds.

\##EP-020: A proxy can intercept only methods that the language allows it to override or dispatch through the proxy.

\##EP-021: Parse once. Cache metadata. Reuse repeatedly.

\##EP-022: Separate bean creation from bean enhancement.

\##EP-023: Represent cross-cutting concerns as an ordered chain of independent interceptors rather than nested proxy objects.

\##EP-024: Separate where logic applies from what logic executes.

\##EP-025: Share behavior; pass context.

\##EP-026: Initialization prepares the object itself; post-processing allows the framework to enhance or replace the object.

\##EP-027: Annotations are declarative metadata, not executable behavior.

\##EP-028: Separate method selection from behavior execution.

\##EP-029: Represent framework behavior as reusable rules rather than per-method configurations.

\##EP-030: Design framework infrastructure around abstractions, not concrete features

