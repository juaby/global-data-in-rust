# A Guide to Global Data in Rust

This guide explains how you can use "global data" in Rust. When I say "global data," I mean data that is loaded near the start of the program and is accessible in almost all of the program.

Possible use cases for global data:

- App configuration, e.g. weapon characteristics for a game 
- Making data available everywhere without needing to pass it as an argument through all functions (apply this carefully!)
- Generating Rust code from external data
- Database connections... or other network resources?
- A logger, maybe

# Tradeoffs

Below are questions to think about when you're choosing a global data solution for your program.

## Compile time or run time?

Loading the data at compile time provides the opportunity to detect invalid data sooner. Also, it might improve the program's startup time.

Loading the data at run time can be nice because changing the data won't trigger a recompile. In complex Rust projects, long compilation times can be a pain point. Another advantage of loading at run-time is that the data can be loaded lazily, which could improve the program's startup time if there is lots of data but not all of it is needed immediately.

It's also possible to implement a hybrid approach where the data is _validated_ at compile time but _loaded_ at run time. That combines the eager validation of compile-time loading with the not-needing-to-recompile of run-time loading.

## Mutable vs. immutable

When I say "immutable" and "mutable," I mean it in a general and hand-wavy sense that is not the equivalent of a Rust type system concept. For an example of this, `lazy_static` uses mutability under the hood but I'm categorizing it as "immutable" because it presents an immutable interface to the user.

Immutable global data can be safely shared between threads with minimal synchronization. It's simple, fast, and easy to understand.

Mutable global data can be a really powerful tool but sometimes can make a program hard to reason about. Before choosing mutable global data, first consider whether there's a way to refactor your code to reduce the scope of the mutable data.

Hot-reloading is an interesting kind of unidirectional immutability where the program can't change the data but external entities can.

## Lifetime of data

Data with the `'static` lifetime can make things easier because you can use it literally anywhere in your program. Statics are "are baked into the data segment of the final binary" ([TRPL 1st ed.](https://doc.rust-lang.org/1.29.2/book/first-edition/lifetimes.html)).

Not all global data will need the `'static` lifetime. Maybe you only need your data available in _most_ of your program, not all of it. This can open up more options for loading your data at run time.

## Is heap allocation supported?

Heap allocation is convenient because you don't need to know the size of your data at compile time. However, it means that you can't use this method without an allocator. Avoiding heap allocations is most important in embedded programming, real-time systems, and really high-performance applications.

# Potential Solutions

Here I'll explain a bit about how each solution works and how to use them, as well as the advantages and disadvantages of each. I will try to order the solutions in order of increasing power, inspired by the [Principle of Least Power](https://www.lihaoyi.com/post/StrategicScalaStylePrincipleofLeastPower.html), although it won't be a strict ordering because there are qualitative differences.

## The `let` keyword

The [`let` keyword](https://doc.rust-lang.org/std/keyword.let.html), which you're probably already familiar with, is used to declare all variables in Rust. Although it might not be the most obvious choice for global data, it offers a number of advantages.

```rust
struct Config {
    my_name: String
}

fn my_fn(config: Config) {
    assert_eq!(config.my_name, "paul");
}

fn main() {
    // This does heap allocation
    let config = Config { my_name: String::from("paul") };
    my_fn(config);
}
```

Advantages:

- Built into Rust
- You often don't need to specify the type of the data. This can be useful for functions and complex types.
- It's easier to provide dummy data for testing
- Allows mutable data
- Allows heap-allocated data

Disadvantages:

- You need to pass the config through each function that you use, which may be bothersome

## The `const` keyword

The [`const` keyword](https://doc.rust-lang.org/std/keyword.const.html) ([TRPL Chapter 3](https://doc.rust-lang.org/stable/book/ch03-01-variables-and-mutability.html#differences-between-variables-and-constants)) is Rust's built-in way to handle immutable constant data. An extremely simple approach.

```rust
const MY_NAME: &str = "paul";

fn main() {
    assert_eq!(MY_NAME, "paul");
}
```

Advantages:

- Built into Rust
- `'static` lifetime
- Data type is validated at compile time

Disadvantages:

- The data that can be created is restricted to simple operations like creating a new struct, as well as some `std` functions that have the [`#[rustc_const_stable]`](https://rustc-dev-guide.rust-lang.org/stability.html#rustc_const_stable) annotation.

## `std::include_str!` and `std::include_bytes!`

The [`std::include_str!`](https://doc.rust-lang.org/std/macro.include_str.html) and [`std::include_bytes!`](https://doc.rust-lang.org/std/macro.include_bytes.html) macros include a file as `&'static str` and `&'static [u8]`, respectively.

See `src/lib.rs` for the code that uses these macros.

```rust
fn main() {
    assert_eq!("Hello, World!", global_data_in_rust::SAMPLE_STR);
    assert_eq!(b"Hello, World!", global_data_in_rust::SAMPLE_BYTES);
}
```

Advantages:

- Built into Rust
- Lifetime of data is `'static`
- Checks for the presence of the file at compile time

## The `lazy_static` and `once_cell` crates

The [`lazy_static`](https://docs.rs/lazy_static) and [`once_cell`](https://docs.rs/once_cell) crates both provide safe interfaces for exactly-once initialization of global static data. They are similar enough that I've grouped them together for now. `lazy_static` is more focused on convenient features for end users, whereas `once_cell` provides more low-level flexibility and avoids macros.

Advantages:

- `'static` lifetime
- Data is loaded lazily at run time
- Allows heap-allocated data
- Allows interior-mutable data
- Can work w/o `std` using `spin_no_std`

Disadvantages:

- The data type needs to fulfill the `Sync` trait. So, if you want have mutable data, you probably need to use like a `Mutex` or `RwLock`. Beware deadlocks and confusing code?
- If the type has a destructor, then it will not run when the process exits. So you probably wouldn't want to do this with anything that has complicated resources that need to be cleaned up. Maybe temporary files, lock files or PID files?

Here's `lazy_static`:

```rust
#[macro_use]
extern crate lazy_static;

use std::collections::HashMap;

lazy_static! {
    static ref GLOBAL_MAP: HashMap<&'static str, &'static str> = {
        let mut m = HashMap::new();
        m.insert("key", "value");
        m
    };
}

fn main() {
    assert_eq!(GLOBAL_MAP.get(&"key"), Some(&"value"));
}
```

...and here's `once_cell`:

```rust
use std::{sync::Mutex, collections::HashMap};
use once_cell::sync::Lazy;

static GLOBAL_MAP: Lazy<Mutex<HashMap<&'static str, &'static str>>> = Lazy::new(|| {
    let mut m = HashMap::new();
    m.insert("key", "value");
    Mutex::new(m)
});

fn main() {
    assert_eq!(GLOBAL_MAP.lock().unwrap().get("key"), Some(&"value"));
}
```

## Immutable static items

If you want flexible mutable global data, one way to accomplish it is to put a synchronization primitive into an [immutable static item](https://doc.rust-lang.org/reference/items/static-items.html). There are two broad ways to do this. One is by lazily initializing the synchronization primitive at run time using something like `lazy_static` and `once_cell`. The other way to do it is to use a synchronization primitive that can be initialized statically, such as those provided by the [`parking_lot` crate](https://docs.rs/parking_lot).

Advantages:

- `'static` lifetime
- Choose your own synchronization primitive
- Choose between compile time and run time initialization
- Enables interior mutability

Disadvantages:

- More choices to make
- `Drop` doesn't run on static items

The example below isn't parallel, but the `parking_lot` mutex can be used in parallel.

```rust
static MY_DATA: parking_lot::Mutex<&str> = parking_lot::const_mutex("hello");

pub fn main() {
    *MY_DATA.lock() = "world";
    assert_eq!(*MY_DATA.lock(), "world");
}
```

## The `phf` crate

The [phf](https://github.com/sfackler/rust-phf) crate lets you generate maps at compile time.

Advantages:

- Compile-time of data validity
- `'static` lifetime
- I think that no heap allocation is required (data lives in binary)

Disadvantages:

- Kind of complex to get working
- Only supports maps

There are two ways to use `phf`. Probably the most normal way is with a custom build script, which would let you generate the map from, e.g., an ingested data file. See `src/main.rs` for an example of this (I couldn't get it to work with `skeptic`).
 The other, simpler way is to create the map inline with a macro:

```rust
use phf::phf_map;

#[derive(Clone, Debug, PartialEq)]
pub enum Keyword {
    Loop,
    Continue,
    Break,
    Fn,
    Extern,
}

static KEYWORDS: phf::Map<&'static str, Keyword> = phf_map! {
    "loop" => Keyword::Loop,
    "continue" => Keyword::Continue,
    "break" => Keyword::Break,
    "fn" => Keyword::Fn,
    "extern" => Keyword::Extern,
};

fn main() {
    assert_eq!(KEYWORDS.get("loop"), Some(&crate::Keyword::Loop))
}
```

## The `arc-swap` crate

When choosing a solution for hot-reloadable global configuration, it's challenging to allow writes without blocking reads. The [`arc-swap` crate](https://docs.rs/arc-swap) provides a thoughtful solution to this problem by taking advantage of atomics. The crate is optimized for managing data that is read frequently but written only occasionally.

Here's an example of how `arc-swap` could be used with `lazy_static` to implement reloadable global configuration. Although the example is not concurrent, the crate will be most helpful in concurrent programming.

```rust
#[macro_use]
extern crate lazy_static;
use arc_swap::{ArcSwap};
use std::sync::Arc;

lazy_static! {
    static ref GLOBAL_CONFIG: ArcSwap<&'static str> = {
        ArcSwap::from(Arc::new("hello"))
    };
}

fn main() {
    assert_eq!(**GLOBAL_CONFIG.load(), "hello");
    GLOBAL_CONFIG.swap(Arc::new("world"));
    assert_eq!(**GLOBAL_CONFIG.load(), "world");
}
```

## `std::include!`

The [`std::include` macro](https://doc.rust-lang.org/std/macro.include.html) is kind of like copy-pasting a snippet of Rust into your code. It can be used to generate complex Rust code at compile time (as in `phf`).

Advantages:

- Built into Rust
- More powerful code generation than with a macro
- Errors will be detected at compile time
- Create mutable or immutable data
- Can work with `'static` lifetime

Disadvantages

- Unhygienic (in the macro sense)

See `src/lib.rs` for the example `include` code.

```rust
fn main() {
    assert_eq!(6, global_data_in_rust::ALSO_SIX);
}
```

## Mutable static items

Items declared as [`static mut`](https://doc.rust-lang.org/reference/items/static-items.html#mutable-statics) are extremely powerful; all access to them is `unsafe` and they provide no guard rails whatsoever. In fact, they are so "powerful" that they are being [considered for deprecation](https://github.com/rust-lang/rust/issues/53639) as of May 2020. They are likely not the solution you're looking for; you can almost always replace them with an immutable static using some kind of synchronization primitive. If you think there's no other way to solve your problem, you're basically talking about building your own synchronization primitive, so be sure to read the [Rustonomicon](https://doc.rust-lang.org/nomicon/) and thoroughly understand the implications of unsafe behavior beforehand!

## Domain-specific solutions

- The Embedded Rust Book [suggests using a singleton pattern](https://rust-embedded.github.io/book/peripherals/singletons.html) instead of a `static mut` to "treat your hardware like data" without requiring as much `unsafe`.
- The Amethyst game engine has a [`Loader`](https://docs-src.amethyst.rs/stable/amethyst_assets/struct.Loader.html) struct that can be used to load data.

# TODO

- Show an example of raw pointers or FFI with static?
- What's a real-life use case of an immutable static item?
- Show an example of multi-threaded mutable static item?
- [`const fn`](https://doc.rust-lang.org/nightly/unstable-book/language-features/const-fn.html) (unstable)?
- [`maplit`](https://docs.rs/maplit)?