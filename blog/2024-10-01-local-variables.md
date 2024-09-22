---
layout: post
tags: [post, internals]
title: "How JavaScript Engines Optimize Your Variables"
authors: boa-dev
---

TODO: Write the intro based on actual content.

<!--truncate-->

## Disclaimers

TODO:

- Optimizations are only about boa. Other engines might do different things.
- There are several omissions to make the post easier and focus on the purpose.

## Scopes and Variables

To start us off, here is a bit of a refresher on how variables and scopes work.

If you have written JavaScript you may already know the concept of a `Scope`.
Scopes describe the areas of code in which variables a visible.
In most cases you might associate scopes with curly braces.

Consider this example:

```js
const a = 1;
console.log(a); // 1

{ // <- start of a block scope
    const a = 2;
    console.log(a); // 2
} // <- end of a block scope
```

We declare and initialize two variables with the name `a`.
Even tough both have the same name, they are different variables.
They are only separated by the scope in witch they are declared.

To demonstrate that the variables are really different we can check them again after the end of a nested scope:

```js
const a = 1;
console.log(a); // 1

function f() { // <- start of a function scope
    var a = 2;
    console.log(a); // 2

    { // <- start of a block scope
        let a = 3;
        console.log(a); // 3
    } // <- end of a block scope

    console.log(a); // 2
} // <- end of a function scope

f();

console.log(a); // 1
```

As you can see, there is no interaction between variables of different scopes.
After a scope has ended, the variable with the same name from the previous scope are accessible again.
Notice that functions also have a scope.
In reality there is a little bit more to function scopes.
There might be multiple scopes per function, but right now it is just important that there is at least one.

If variables do not have the same name, we can of course access variables from outer scopes:

```js
const a = 1;
console.log(a); // 1

{
    const b = 2;
    console.log(a); // 1
    console.log(b); // 2
}
```

These principles are mostly sufficient if we just want to write JavaScript.
But when developing a JavaScript engine we have to think about how we actually store and access these scopes and variables.

## Storing Variables

How would you store scopes and variables in your JavaScript engine?
Since we need to map variable names to their values, the obvious solution is a hashmap:

```rust
struct Scope {
    variables: HashMap<Name, Value>,
}
```

Like this we have an easy mapping from variable names in our current scope to their values.
But how do we access variables from an outer scope?
To do that we can just reference the outer scope and access it, when we do not find the variable name in the current one:

```rust
struct Scope {
    variables: HashMap<Name, Value>,
    outer: Box<Scope>,
}
```

This solution works and is probably the easiest to reason about.
The data structures map very well on to our mental model of the problem.
Indeed, this was the structure that was used in `Boa` before we switched to a different implementation over two years ago (https://github.com/boa-dev/boa/pull/1829).

Maybe you already spotted some performance issues with this structure.
Since accessing variables for both reading and writing is one of the core things we do in programming, you might imagine that doing a lookup in a hashmap for every access could be very imperformant.
This problem gets worse when our variable is in an outer scope.
In the worst case, we have to do as many hashmap lookups as there are scopes.

How would you solve this performance issue?
Maybe stop here and think a bit about what could be optimized.

The first thing we did in `Boa` two years ago was to move from dynamic runtime hashmap lookups to calculating the memory locations of variables before we even run the code.
Maybe you already had the solution in mind: We can define a unique index for every variable by just looking at the code.

Consider this example:

```js
const a = 1; // scope index: 0; variable index: 0
{
    const b = 2; // scope index: 1; variable index: 0
    const c = 3; // scope index: 1; variable index: 1
}

function f() {
    const d = 2; // scope index: 1; variable index: 1
    {
        const e = 3; // scope index: 2; variable index: 0
    }
}
```

In this example you can see the two indicies assigned to each variable, that make them unique.
The index of the scope and the index of the variable in that specific scope.
Notice that both `c` and `d` have the scope index 1 and the variable index 1.
This is fine because the indicies only need to be unique in their branch of the scope tree.
The variable `c` cannot be accessed from the scope where `d` ios defined or vice versa.
You might wonder why `d` has the variable index 1.
This is not a typo.
The variable not visible here is the `arguments` object of the function `f` that has the scope index 1 and the variable index 0.

With all of this in mind, we can build a data structure that allows us to access variables just based on two indicies.
It might look something like this:

```rust
struct Scopes {
    scopes: Vec<Scope>,
}

struct Scope {
    variables: Vec<Value>,
}
```

In addition we of course have to add the information about the two indicies to every variable access in our execution code.
Before running the code we still have to construct our hashmaps to determine which variable indicies we need at each specific access.
But instead of doing a lookup on every access, we just do it once.

## Local Variables

- stack vm
- call frame per function

TODO: Why is it faster? It's only local register vs two array indicies. -> Runtime scopes have to be GCd, checks before


## Outline

- Explain Variables / Bindings
- Explain Naive Storage
- Into Boa
- Variable Storage In Runtime Environemnts / Contrast to Naive Storage
- Introduce Scope Analysis
- Local Variables
- Further Optimizations based on Scope Analysis
