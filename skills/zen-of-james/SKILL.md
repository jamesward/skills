---
name: zen-of-james
description: James Ward writes good code. These are his guiding priciples.
---

# When You Write Code

MAKE ILLEGAL STATES UNREPRESENTABLE!!!
- Use compile time validatable types (e.g. if a list can't be empty use a `NonEmptyList` type)
- Closed branching (ADTs & exhastive pattern matching)
- Represent nullability in types
- Universal equality is a huge potential source of hidden bugs (e.g. `person == cat`) so in languages that support it, use multiversal equality (like with `CanEqual` type classes)
- Always use manage resources correctly (e.g. try with resources) to avoid leaks

NO MUTABILITY EVER!!!
- Mutable variables may seem like a good idea. They are not.
- Pure functions are the ideal.
- When side effects are required, they should be represented as values / Effects. However, if this isn't possible because an effect-oriented approach isn't possible, the side-effecting functions should take an iterface as a parameter enabling easy testability of the operations that perform side effects.

DATA TYPES & FUNCTIONS SHOULD BE DOMAIN ORIENTED
- In languages with type aliases, wrap primitive types into domain types. For example, don't take a `customerId: String` create a type alias `CustomerId = String` and take the domain type instead.
- In languages with opaque types, use those instead of type aliases.
- Parse, don't validate

TEST ONIONS ARE GOOD. TEST ASSERTION DSLS ARE BAD.
- The test onion enables testing iteration loops which start fast & lite and progress to slower & integrated.
- Types > Unit tests for pure functions > Integration tests for side effects which use fake implementations > Integration tests for side effects with Testcontainers
- Use `assertTrue` with regular code / boolean evaluations
