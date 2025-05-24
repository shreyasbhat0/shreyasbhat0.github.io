---
title: "Structs: Building Custom Types with Custom Invariants"
date: 2025-05-24 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, structs, methods, data-types, invariants]
description: "Structs let you define your own types — and your own invariants. By controlling construction, field access, and method behavior, you decide what 'valid' means for your data."
---

Primitive types carry built-in invariants: a `bool` is always `true` or `false`, an `i32` is always a valid 32-bit integer. But real programs need domain-specific types — a `Student`, a `Point`, an `Address` — with domain-specific invariants. That's what structs are for.

A struct groups related data under a single type. More importantly, when combined with Rust's visibility rules and methods, a struct lets you **define and enforce your own invariants** — constraints on what constitutes valid data for your domain.

## Defining a Struct

```rust
struct Student {
    name: String,
    age: i32,
    email: String,
    semester: String,
    blood_group: String,
}
```

`Student` is now a type. It has five fields, each with a name and a type. The struct definition is a **template** — it declares what data a `Student` must contain but doesn't create any values.

The type invariant is immediate: a `Student` always has all five fields, always of the declared types. You cannot create a `Student` with a missing email or an age that's a string. The compiler enforces it.

## Creating Instances

Create an instance by providing values for every field:

```rust
fn main() {
    let student_1 = Student {
        name: String::from("Alice"),
        age: 30,
        email: String::from("alice@example.com"),
        semester: String::from("4th"),
        blood_group: String::from("AB+ve"),
    };
}
```

Field order doesn't matter — you can specify them in any sequence:

```rust
fn main() {
    let student_1 = Student {
        email: String::from("alice@example.com"),
        semester: String::from("4th"),
        blood_group: String::from("AB+ve"),
        name: String::from("Alice"),
        age: 30,
    };
}
```

The invariant: **every field must be initialized**. There is no "partial construction." A `Student` with four out of five fields is not a `Student` — it's a compile error.

### Field Init Shorthand

When a variable has the same name as a field, you can use the shorthand:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let x = 10;
    let y = 20;
    let point1 = Point { x, y }; // equivalent to Point { x: x, y: y }
}
```

### Creating Instances from Other Instances

The struct update syntax copies fields from an existing instance:

```rust
struct Point {
    x: i32,
    y: i32,
    z: i32,
}

fn main() {
    let x = 10;
    let y = 20;
    let z = 30;
    let p1 = Point { x, y, z };
    let p2 = Point { x: 40, ..p1 }; // y and z come from p1
}
```

The `..p1` syntax fills remaining fields from `p1`. The invariant holds: `p2` still has all three fields, all correctly typed.

## Tuple Structs

Structs without named fields — useful when field names would be redundant:

```rust
struct Address(String);

fn main() {
    let address = Address(String::from("INDIA"));
}
```

Tuple structs give the type a name without naming each field. `Address` is a distinct type from `String` — you cannot accidentally pass a `String` where an `Address` is expected. This is the **newtype pattern**, and it creates a type-level invariant: `Address` and `String` are different types with different semantics, even though they contain the same data.

## Unit-Like Structs

Structs with no fields at all:

```rust
struct UnitLike;

fn main() {
    let subject = UnitLike;
}
```

These are useful for implementing traits on a type that doesn't need to store data. The invariant is trivial — a `UnitLike` value always exists and is always valid.

## Accessing Fields

Use dot notation for named structs:

```rust
fn main() {
    let student_1 = Student {
        name: String::from("Alice"),
        age: 30,
        email: String::from("alice@example.com"),
        semester: String::from("4th"),
        blood_group: String::from("AB+ve"),
    };
    println!("{}", student_1.name);
}
```

Use index notation for tuple structs:

```rust
fn main() {
    let address = Address(String::from("INDIA"));
    println!("{}", address.0);
}
```

The invariant: **field access is always valid**. You cannot access a field that doesn't exist — the compiler rejects it. You cannot access a field at the wrong index. Type-checked, bounds-checked, at compile time.

## Methods

Methods are functions defined in the context of a struct. They are declared inside an `impl` block:

```rust
#[derive(Debug, Clone)]
struct Student {
    name: String,
    age: i32,
    sid: String,
    blood_group: String,
}

impl Student {
    fn new(name: String, age: i32, sid: String, blood_group: String) -> Self {
        Student {
            name,
            age,
            sid,
            blood_group,
        }
    }

    fn name(&self) -> &str {
        &self.name
    }

    fn sid(&self) -> &str {
        &self.sid
    }
}

fn main() {
    let s = Student::new(
        "ALICE".to_string(),
        32,
        "UNICAL12345".to_string(),
        "A+ve".to_string(),
    );

    println!("Student Name: {:?} and Student Id: {:?}", s.name(), s.sid());
}
```

### Associated Functions vs Methods

- **`new`** is an **associated function** — called with `Student::new(...)`, not on an instance. It's a constructor that ensures every `Student` is created with all required fields. This is where you enforce construction invariants.
- **`name` and `sid`** are **methods** — called on an instance with `s.name()`. Their first parameter is `&self`, borrowing the instance immutably.

### The Constructor Invariant

The `new` function is critical. By making struct fields private (the default for modules) and providing a `new` function, you control how instances are created. This is how you enforce invariants beyond what the type system gives you:

```rust
impl Student {
    fn new(name: String, age: i32, sid: String, blood_group: String) -> Self {
        assert!(age > 0, "Age must be positive");
        assert!(!sid.is_empty(), "Student ID cannot be empty");
        Student { name, age, sid, blood_group }
    }
}
```

Now the invariant "age is positive and SID is non-empty" is enforced at every construction site. You cannot create an invalid `Student`. The type itself guarantees validity.

### Method Self Parameters

The `self` parameter determines how the method borrows the instance:

| Parameter | Meaning |
|:--|:--|
| `&self` | Immutable borrow — reads but cannot modify |
| `&mut self` | Mutable borrow — can modify the instance |
| `self` | Takes ownership — consumes the instance |

These align directly with the borrowing invariants from the previous post. A method with `&self` guarantees it won't mutate your data. A method with `&mut self` guarantees exclusive access. The invariant is encoded in the signature.

## Structs as Invariant Containers

The pattern emerges: structs are not just data containers. They are **invariant containers**.

| Mechanism | Invariant Enforced |
|:--|:--|
| Field types | Every field holds the correct type |
| Required initialization | No partial construction |
| Private fields + constructor | Domain-specific validity rules |
| Methods with `&self` | Read-only access |
| Methods with `&mut self` | Exclusive mutable access |
| Newtype pattern | Type-level distinction between same-shaped data |

By combining struct definitions with visibility rules and `impl` blocks, you build types that are correct by construction. The compiler becomes your invariant checker, and invalid states become unrepresentable.

---

*Structs are where Rust's invariant system becomes your own. Primitive types enforce Rust's rules. Structs enforce yours. With ownership, borrowing, and custom types, you have the tools to make invalid states impossible — not just unlikely, but structurally unrepresentable.*
