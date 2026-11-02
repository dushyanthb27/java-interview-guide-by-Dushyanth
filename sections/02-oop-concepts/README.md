# 02 — Object-Oriented Programming

[Home](../../README.md) > Object-Oriented Programming

---

## Overview

Object-Oriented Programming is the backbone of Java. This section covers the four pillars of OOP, interface evolution, and modern functional programming integration. Understanding these concepts deeply is what separates developers who write Java code from those who design Java systems.

## Prerequisites

- [Java Fundamentals](../01-java-fundamentals/README.md) — Data types, memory model, and JVM basics

## Question Index

| # | Question | Difficulty | File |
|---|----------|------------|------|
| Q4 | What are classes and objects in Java? | 🟢 | [q01-classes-and-objects.md](./q01-classes-and-objects.md) |
| Q5 | Explain constructors and their types | 🟢 | [q02-constructors.md](./q02-constructors.md) |
| Q6 | How does inheritance work in Java? | 🟢 | [q03-inheritance.md](./q03-inheritance.md) |
| Q7 | Explain polymorphism with examples | 🟡 | [q04-polymorphism.md](./q04-polymorphism.md) |
| Q8 | What is encapsulation and why is it important? | 🟢 | [q05-encapsulation.md](./q05-encapsulation.md) |
| Q9 | What is abstraction and how is it achieved? | 🟢 | [q06-abstraction.md](./q06-abstraction.md) |
| Q10 | Interfaces vs Abstract Classes — when to use each? | 🟡 | [q07-interfaces-vs-abstract-classes.md](./q07-interfaces-vs-abstract-classes.md) |
| Q11 | How does method overloading work? | 🟡 | [q08-method-overloading.md](./q08-method-overloading.md) |
| Q12 | How does method overriding work? | 🟡 | [q09-method-overriding.md](./q09-method-overriding.md) |
| Q13 | Explain the super keyword | 🟢 | [q10-super-keyword.md](./q10-super-keyword.md) |
| Q14 | Explain the this keyword | 🟢 | [q11-this-keyword.md](./q11-this-keyword.md) |
| Q15 | Composition vs Inheritance | 🟠 | [q12-object-composition.md](./q12-object-composition.md) |
| Q16 | What are inner classes and their types? | 🟡 | [q13-inner-classes.md](./q13-inner-classes.md) |
| Q17 | How do lambdas relate to OOP? | 🟡 | [q14-lambdas-and-oop.md](./q14-lambdas-and-oop.md) |
| Q18 | What are functional interfaces? | 🟡 | [q15-functional-interfaces.md](./q15-functional-interfaces.md) |
| Q19 | How do default methods work in interfaces? | 🟠 | [q16-default-methods-interfaces.md](./q16-default-methods-interfaces.md) |
| Q20 | Static and private methods in interfaces | 🟠 | [q17-static-private-methods-interfaces.md](./q17-static-private-methods-interfaces.md) |
| Q21 | How does Java solve the Diamond Problem? | 🟠 | [q18-diamond-problem.md](./q18-diamond-problem.md) |

## Key Takeaways

- **Single class inheritance** with **multiple interface implementation** — Java avoids the diamond problem while enabling polymorphism
- **Prefer composition over inheritance** for flexible designs — has-a relationships are more maintainable than is-a chains
- **Interfaces have evolved significantly** from Java 8+ — default methods, static methods, and private methods enable richer APIs without breaking compatibility
- **Polymorphism enables extensible, maintainable code** — design around abstractions, not concrete implementations
- **Encapsulation protects internal state** — use access modifiers judiciously and expose behavior, not data
- **Abstraction hides implementation details** — abstract classes and interfaces define contracts
- **Method overloading vs overriding** — overloading is compile-time polymorphism, overriding is runtime polymorphism
- **The `this` and `super` keywords** — `this` refers to the current instance, `super` refers to the parent class
- **Inner classes** — nested classes provide logical grouping and access to outer class state
- **Lambdas bridge OOP and functional programming** — functional interfaces enable concise, readable code
- **Default methods in interfaces** — enable interface evolution without breaking existing implementations
- **The Diamond Problem** — Java's single inheritance + multiple interfaces approach solves the ambiguity

## Related Sections

- [01 — Java Fundamentals](../01-java-fundamentals/README.md) — Core concepts
- [06 — Java 8+ Features](../06-java8-plus-features/README.md) — Lambdas, streams, and functional programming
- [03 — Collections Framework](../03-collections-framework/README.md) — Data structures that leverage OOP principles
