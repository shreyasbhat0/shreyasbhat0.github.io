---
title: "Rust Fundamentals: Control Flow and the Branch Consistency Invariant"
date: 2025-04-05 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, control-flow, loops, conditionals, invariants]
description: "In Rust, every branch of a conditional must agree on its type. Every loop has explicit termination semantics. Control flow isn't just syntax — it's a set of invariants the compiler enforces."
---

Control flow is the order in which instructions are executed and evaluated. Most languages treat it as pure syntax — `if`, `else`, `for`, `while`. Rust treats it as a place where invariants are enforced. Conditions must be boolean. Branches must agree on types. Loops have explicit termination semantics. These rules catch entire categories of bugs at compile time.

## If Expressions

`if` in Rust allows you to branch based on a condition:

```rust
fn main() {
    let number = 20;
    if number != 20 {
        println!("Number not equal to 20");
    } else {
        println!("Number equal to 20");
    }
}
```

If the condition is true, the code in the `if` block executes. Otherwise, the `else` block runs. If there's no `else` and the condition is false, the program skips the block entirely.

### The Boolean Condition Invariant

Rust does not implicitly convert non-boolean types to booleans. This is a hard invariant:

```rust
fn main() {
    let number = 20;
    if number { // ERROR
        println!("Number not equal to 20");
    }
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:3:8
  |
3 |     if number {
  |        ^^^^^^ expected `bool`, found integer
```

In C, `if (number)` is valid — any non-zero value is truthy. This is a source of subtle bugs. Rust's invariant is explicit: **conditions must evaluate to `bool`**. There is no truthiness. There is no implicit conversion. `0` is not `false`. `""` is not `false`. Only `false` is `false`.

### Handling Multiple Conditions

Use `else if` to chain conditions:

```rust
fn main() {
    let number = 6;

    if number % 4 == 0 {
        println!("number is divisible by 4");
    } else if number % 3 == 0 {
        println!("number is divisible by 3");
    } else if number % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```

Output:

```
number is divisible by 3
```

Rust executes the first branch whose condition is true, then skips the rest. Even though 6 is divisible by both 3 and 2, only the first matching branch runs. The invariant: **exactly one branch executes**.

### Using if in a let Statement

Because `if` is an expression (not a statement), it evaluates to a value and can be used with `let`:

```rust
fn main() {
    let condition = false;
    let number = if condition { 5 } else { 6 };

    println!("The value of number is: {number}");
}
```

Output:

```
The value of number is: 6
```

Here's where Rust's type invariant intersects with control flow. Both branches must return the same type:

```rust
fn main() {
    let condition = false;
    let number = if condition { 5 } else { "six" }; // ERROR
}
```

```
error[E0308]: `if` and `else` have incompatible types
 --> src/main.rs:3:44
  |
3 |     let number = if condition { 5 } else { "six" };
  |                                 -          ^^^^^ expected integer, found `&str`
```

The invariant: **all branches of an expression must produce the same type**. Rust needs to know the type of `number` at compile time, and it cannot be "integer sometimes, string other times." This eliminates a class of bugs that dynamically typed languages discover only at runtime: the function that returns a number in one case and a string in another.

## Loops

### loop

The `loop` keyword creates an unconditional infinite loop:

```rust
fn main() {
    loop {
        println!("again!");
    }
}
```

This runs forever until explicitly stopped with `break`. The invariant is simple: **a `loop` always executes at least once** (unlike `while`, which might execute zero times).

You can also return values from loops:

```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    println!("The result is {result}");
}
```

Output:

```
The result is 20
```

The `break` expression returns a value from the loop. The invariant: **every `break` in a value-returning loop must produce the same type**.

### Loop Labels

When loops are nested, `break` and `continue` apply to the innermost loop by default. Loop labels let you target a specific loop:

```rust
fn main() {
    let mut count = 0;
    'counting_up: loop {
        println!("count = {count}");
        let mut remaining = 10;

        loop {
            println!("remaining = {remaining}");
            if remaining == 9 {
                break;
            }
            if count == 2 {
                break 'counting_up;
            }
            remaining -= 1;
        }

        count += 1;
    }
    println!("End count = {count}");
}
```

Output:

```
count = 0
remaining = 10
remaining = 9
count = 1
remaining = 10
remaining = 9
count = 2
remaining = 10
End count = 2
```

Labels must begin with a single quote (`'`). The invariant: **every `break` and `continue` targets a specific, named loop**. No ambiguity about which loop is being controlled.

### while

`while` loops execute as long as a condition is true:

```rust
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{number}!");
        number -= 1;
    }

    println!("LIFTOFF!!!");
}
```

Output:

```
3!
2!
1!
LIFTOFF!!!
```

The same boolean invariant applies — the condition must be a `bool` expression, never an implicit conversion.

### for

`for` loops iterate over a range or collection:

```rust
fn main() {
    for x in 1..11 { // 11 is not inclusive
        if x == 5 {
            continue;
        }
        println!("x is {}", x);
    }
}
```

Output:

```
x is 1
x is 2
x is 3
x is 4
x is 6
x is 7
x is 8
x is 9
x is 10
```

Iterating over a collection:

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a {
        println!("the value is: {element}");
    }
}
```

Output:

```
the value is: 10
the value is: 20
the value is: 30
the value is: 40
the value is: 50
```

The `for` loop in Rust uses the `Iterator` trait, which enforces its own invariant: **iteration proceeds element by element, in order, with no bounds violations**. You cannot overrun an array with a `for` loop — the iterator knows when to stop.

## The Control Flow Invariants, Summarized

| Construct | Invariant |
|:--|:--|
| `if` | Condition must be `bool` |
| `if`/`else` as expression | All branches must return the same type |
| `loop` | Executes at least once; `break` values must agree on type |
| `while` | Condition must be `bool`; may execute zero times |
| `for` | Iterator bounds are enforced; no overruns |

These aren't style preferences. They're compile-time guarantees. A branch that returns the wrong type is caught before your code ever runs.

---

*Control flow in Rust is where type invariants and boolean invariants intersect. The compiler doesn't just check syntax — it checks that every possible execution path is type-consistent. Next up: functions, and the invariants that govern their boundaries.*
