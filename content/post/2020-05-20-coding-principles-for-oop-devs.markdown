---
title: "A note on software design principles and practices"
tags: [coding, principles and practices, solid]
date: 2020-05-20T14:24:10+10:00
lastmod: 2020-05-20T14:24:10+10:00
draft: false
---

### Timeless design principles
* YAGNI = You aint't gonna need it. Unused code or dead code is liability.
* KISS = Keep it simple, stupid. Simple design first.
* DRY = Don't repeat yourself. Use abstraction, policy, normalisation to avoid redundancy.
* S (SOLID) = Single responsibility. Encapsulate functionality, only one reason to change.
* O (SOLID) = Polymorphic Open/Closed principle. Design for extension, change behaviour without modifying current code.
* L (SOLID) = Liskov subsitution principle. Subtypes shouldn't strengthen preconditions (what need for code run) or weaken postconditions (what happen after code run). Subtypes should preserve invariants, and shouldn't allow state changes that the parent disallowed.
* I (SOLID) = Interface segregation principles. No class should be forced to depend on methods it does not use because this introduces unnecessary coupling.
* D (SOLID) = Dependency Inversion. High-level modules should not depend upon low-level modules. Both should depend upon abstractions. Abstractions should not depend upon details. Details should depend upon abstractions.

### Coding practices
* Distinguish between Immutable and Mutable.
* Think if the code you write is to be maintained by other devs in next 6 months.
* Avoid Gold classes and God interface.
* Define smaller interface, suit to purpose, minimise dependency operations or internals of a domain object.
* Maintain high cohesion at class and method level. An interface with single method also not good - no cohesion.
* Override equals/hashcode if true comparision is required.
* The Principle of Least Astonishment: the result of performing some operation should be obvious, consistent, and predictable, based upon the name of the operation and other clues.
* Return domain class to wrap primitive values.
* Use Checked exception for recoverable error, use Unchecked exception (runtime exc) for unrecoverable error.
* Avoid Checked exceptions if possible - it creates clutter in code, reduce productivity.
* Use Validator and Notification pattern when dealing with user-defined checked exceptions.
* Avoid Overly specific - too many exception type class or Overly apathetic - one exception class for everything.
* Avoid writing function return Void, it is hard to test
* Composition over Inheritance.
* Avoid utility class if you can, use domain object instead.
* Use builder pattern and Fluent API.
* Never return null for a function, it can cause other developers to fail to handle null pointer.
* Try to use Enums / Domain object for multiple status outcome, don't use null, number, string or exception.
* Keep technology-specific stuff living outside of core domain (Hexagonal arch).
* Use well established security library for security functions (i.e Bouncy Castle's). Avoid writing your own.
* Common pattern for persistence is Query Object or Repository.
* Avoid hardcode each query into method of Repository, use Query obj instead.
* UoW = Unit of Work design pattern to guarantee transactions.
* Where possible, use functional programming to handle stream, reactive, parallel processing, avoid handling large threading code.
* Know when to use Value object.

### Testing and TDD

* Code coverage only tell you what has not been tested, not the quality of the test.
* Think about public interfaces first hand, and write tests for them.
* Write code using Interface, Mock TDD first, then start the implementation.

(TBC)