`Scope` &ndash; Maintaining State
=================================

{{#include ../links.md}}

By default, Rhai treats each [`Engine`] invocation as a fresh one, persisting only the functions
that have been registered but no global state.

This gives each evaluation a clean starting slate.

In order to continue using the same global state from one invocation to the next, such a state
(a `Scope`) must be manually created and passed in.

All `Scope` [variables] and [constants] have values that are [`Dynamic`], meaning they can store
values of any type.

Under [`sync`], however, only types that are `Send + Sync` are supported, and the entire `Scope`
itself will also be `Send + Sync`. This is extremely useful in multi-threaded applications.

```admonish info.small "Shadowing"

A newly-added [variable] or [constant] _[shadows][shadow]_ previous ones of the same name.

In other words, all versions are kept for [variables] and [constants], but only the latest ones can
be accessed via `get_value<T>`, `get_mut<T>` and `set_value<T>`.

Essentially, a `Scope` is always searched in _reverse order_.
```

```admonish tip.small "Tip: The lifetime parameter"

`Scope` has a _lifetime_ parameter, in the vast majority of cases it can be omitted and
automatically inferred to be `'static`.

Currently, that lifetime parameter is not used.  It is there to maintain backwards compatibility
as well as for possible future expansion when references can also be put into the `Scope`.

The lifetime parameter is not guaranteed to remain unused for future versions.

In order to put a `Scope` into a `struct`, use `Scope<'static>`.
```

~~~admonish tip.small "Tip: The `const` generic parameter"

`Scope` also has a _`const` generic_ parameter, which is a number that defaults to 8.
It indicates the number of entries that the `Scope` can keep _inline_ without allocations.

The larger this number, the larger the `Scope` type gets, but allocations will happen far
less frequently.

A smaller number makes `Scope` smaller, but allocation costs will be incurred when the
number of entries exceed the _inline_ capacity.
~~~


`Scope` API
-----------

| Method                                        | Description                                                                                                                                         |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `new` _instance method_                       | create a new empty `Scope`                                                                                                                          |
| `with_capacity` _instance method_             | create a new empty `Scope` with a specified initial capacity                                                                                        |
| `len`                                         | number of [variables]/[constants] currently within the `Scope`                                                                                      |
| `rewind`                                      | _rewind_ (i.e. reset) the `Scope` to a particular number of [variables]/[constants]                                                                 |
| `clear`                                       | remove all [variables]/[constants] from the `Scope`, making it empty                                                                                |
| `is_empty`                                    | is the `Scope` empty?                                                                                                                               |
| `is_constant`                                 | is the particular [variable]/[constant]  in the `Scope` a [constant]?                                                                               |
| `push`, `push_constant`                       | add a new [variable]/[constant] into the `Scope` with a specified value                                                                             |
| `push_dynamic`, `push_constant_dynamic`       | add a new [variable]/[constant] into the `Scope` with a [`Dynamic`] value                                                                           |
| `set_or_push<T>`                              | set the value of the last [variable] within the `Scope` by name if it exists and is not [constant]; add a new [variable] into the `Scope` otherwise |
| `contains`                                    | does the particular [variable] or [constant] exist in the `Scope`?                                                                                  |
| `get_value<T>`                                | get the value of the last [variable]/[constant] within the `Scope` by name                                                                          |
| `set_value<T>`                                | set the value of the last [variable] within the `Scope` by name, panics if it is [constant]                                                         |
| `remove<T>`                                   | remove the last [variable]/[constant] from the `Scope` by name, returning its value                                                                 |
| `get`                                         | get a reference to the value of the last [variable]/[constant] within the `Scope` by name                                                           |
| `get_mut`                                     | get a reference to the value of the last [variable] within the `Scope` by name, `None` if it is [constant]                                          |
| `set_alias`                                   | [exported][`export`] the last [variable]/[constant] within the `Scope` by name                                                                      |
| `iter`, `iter_raw`, `IntoIterator::into_iter` | get an iterator to the [variables]/[constants] within the `Scope`                                                                                   |
| `Extend::extend`                              | add [variables]/[constants] to the `Scope`                                                                                                          |

~~~admonish info.small "`Scope` public API"

For details on the `Scope` API, refer to the
[documentation](https://docs.rs/rhai/{{version}}/rhai/struct.Scope.html) online.
~~~


Serializing/Deserializing
-------------------------

With the [`serde`] feature, `Scope` is serializable and deserializable via
[`serde`](https://crates.io/crates/serde).

[Custom types] stored in the `Scope`, however, are serialized as full type-name strings.
Data in [custom types] are not serialized.


Example
-------

In the following example, a `Scope` is created with a few initialized variables, then it is threaded
through multiple evaluations.

```rust
use rhai::{Engine, Scope, EvalAltResult};

let engine = Engine::new();

// First create the state
let mut scope = Scope::new();

// Then push (i.e. add) some initialized variables into the state.
// Remember the system number types in Rhai are i64 (i32 if 'only_i32')
// and f64 (f32 if 'f32_float').
// Better stick to them or it gets hard working with the script.
scope.push("y", 42_i64)
     .push("z", 999_i64)
     .push_constant("MY_NUMBER", 123_i64)       // constants can also be added
     .set_value("s", "hello, world!");          // 'set_value' adds a new variable when one doesn't exist

// First invocation
engine.run_with_scope(&mut scope, 
"
    let x = 4 + 5 - y + z + MY_NUMBER + s.len;
    y = 1;
")?;

// Second invocation using the same state.
// Notice that the new variable 'x', defined previously, is still here.
let result = engine.eval_with_scope::<i64>(&mut scope, "x + y")?;

println!("result: {result}");                   // prints 1103

// Variable y is changed in the script - read it with 'get_value'
assert_eq!(scope.get_value::<i64>("y").expect("variable y should exist"), 1);

// We can modify scope variables directly with 'set_value'
scope.set_value("y", 42_i64);
assert_eq!(scope.get_value::<i64>("y").expect("variable y should exist"), 42);
```


`Engine` API Using `Scope`
--------------------------

[`Engine`] API methods that accept a `Scope` parameter all end in `_with_scope`, making that
`Scope` (and everything inside it) available to the script:

| `Engine` API                            | Not available under |
| --------------------------------------- | :-----------------: |
| `Engine::eval_with_scope`               |                     |
| `Engine::eval_ast_with_scope`           |                     |
| `Engine::eval_file_with_scope`          |     [`no_std`]      |
| `Engine::eval_expression_with_scope`    |                     |
| `Engine::run_with_scope`                |                     |
| `Engine::run_ast_with_scope`            |                     |
| `Engine::run_file_with_scope`           |     [`no_std`]      |
| `Engine::compile_file_with_scope`       |     [`no_std`]      |
| `Engine::compile_expression_with_scope` |                     |

~~~admonish danger "Don't forget to `rewind`"

[Variables] or [constants] defined at the global level of a script persist inside the custom `Scope`
even after the script ends.

```rust
let mut scope = Scope::new();

engine.run_with_scope(&mut scope, "let x = 42;")?;

// Variable 'x' stays inside the custom scope!
engine.run_with_scope(&mut scope, "print(x);")?;    //  prints 42
```

Due to [variable shadowing][shadowing], new [variables]/[constants] are simply added on top of
existing ones (even when they already exist), so care must be taken that new [variables]/[constants]
inside the custom `Scope` do not grow without bounds.

```rust
let mut scope = Scope::new();

// Don't do this - this creates 1 million variables named 'x'
//                 inside 'scope'!!!
for _ in 0..1_000_000 {
    engine.run_with_scope(&mut scope, "let x = 42;")?;
}

// The 'scope' contains a LOT of variables...
assert_eq!(scope.len(), 1_000_000);

// Variable 'x' stays inside the custom scope!
engine.run_with_scope(&mut scope, "print(x);")?;    //  prints 42
```

In order to remove [variables] or [constants] introduced by a script, use the `rewind` method.

```rust
// Run a million times
for _ in 0..1_000_000 {
    // Save the current size of the 'scope'
    let orig_scope_size = scope.len();

    engine.run_with_scope(&mut scope, "let x = 42;")?;

    // Rewind the 'scope' to the original size
    scope.rewind(orig_scope_size);
}

// The 'scope' is empty
assert_eq!(scope.len(), 0);

// Variable 'x' is no longer inside 'scope'!
engine.run_with_scope(&mut scope, "print(x);")?;    //  error: variable 'x' not found
```
~~~
