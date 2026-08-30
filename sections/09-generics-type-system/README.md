# Generics & Type System

[Home](../../README.md) > Generics & Type System

---

## Overview
Generics provide compile-time type safety and eliminate casting. This section covers type erasure, wildcards, PECS principle, and advanced generic patterns used in frameworks.

## Prerequisites
- [Java Fundamentals](../01-java-fundamentals/README.md)
- [OOP Concepts](../02-oop-concepts/README.md)
- [Collections Framework](../03-collections-framework/README.md)

## Question Index
| # | Question | Difficulty | File |
|---|----------|------------|------|
| 1 | How do generic classes and methods work? | 🟢 | [q01](./q01-generic-classes-methods.md) |
| 2 | What is type erasure? | 🟡 | [q02](./q02-type-erasure.md) |
| 3 | What are bounded type parameters? | 🟡 | [q03](./q03-bounded-type-parameters.md) |
| 4 | Explain upper-bounded wildcards (? extends) | 🟡 | [q04](./q04-wildcards-extends.md) |
| 5 | Explain lower-bounded wildcards (? super) | 🟡 | [q05](./q05-wildcards-super.md) |
| 6 | What is the PECS principle? | 🟠 | [q06](./q06-pecs-principle.md) |
| 7 | What are generic constraints and limitations? | 🟡 | [q07](./q07-generic-constraints.md) |
| 8 | How does type inference work? | 🟡 | [q08](./q08-type-inference.md) |
| 9 | What is the diamond operator? | 🟢 | [q09](./q09-diamond-operator.md) |
| 10 | What are bridge methods? | 🟠 | [q10](./q10-bridge-methods.md) |
| 11 | Why can't you create generic arrays? | 🟠 | [q11](./q11-generic-arrays.md) |
| 12 | How to mix generics with legacy code? | 🟡 | [q12](./q12-generics-legacy-code.md) |
| 13 | Explain covariance and contravariance | 🟠 | [q13](./q13-covariance-contravariance.md) |
| 14 | How to design custom generic types? | 🟠 | [q14](./q14-custom-generic-types.md) |
| 15 | Advanced generics patterns | 🔴 | [q15](./q15-advanced-generics-patterns.md) |

## Key Takeaways
- Type erasure removes generic type information at runtime
- PECS: Producer Extends, Consumer Super
- Generic arrays cannot be created due to type erasure
- Bridge methods maintain polymorphism after erasure
- Recursive type bounds enable fluent builder patterns

## Related Sections
- [Collections Framework](../03-collections-framework/README.md)
- [OOP Concepts](../02-oop-concepts/README.md)
- [Java 8+ Features](../06-java8-plus-features/README.md)
