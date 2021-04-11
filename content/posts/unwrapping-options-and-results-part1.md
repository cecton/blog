---
title: "Unwrapping Options and Results: Part 1"
date: 2021-04-04T08:30:00+01:00
---
Synopsis
--------

In Rust you don't handle errors through `try...catch/except` but through a
special type called a `Result`. You also don't handle out of bound indexing
(like `my_list[3]` where `my_list` has only 2 items) through defensive
programming but through `Option`. This blog post is for people who want to
know how and why Rust is forcing you to use this pattern. After reading this
you should be able to appreciate and love it.

The Wrapper Type
----------------

`Result` and `Option` are types that wrap a value. For example, a value of
type `u64` can be wrapped in an `Option`:

```rust
//      variable name     type                value
//
let     value           : u64               = 42;
let     wrapped_value   : Option<u64>       = Some(value);
```

In the case of `Option`:
 -  There are 2 variants: `Option::Some` and `Option::None`. They are both in
    the "std prelude", this means you can just write `Some` or `None` instead.
 -  There is only one type argument: the type of the inner value. (`u64` in
    this example).

Only the variant `Some` is able to hold a value. The variant `None` means "no
value". Because of that, you can actually use a wrapper type that will never
hold any value but still knows what type of value it should have if it had one.
For example, this is an `Option` of `u64` with no value:

```rust
//      variable name     type                value
//
let     wrapped_value   : Option<u64>       = None;
```

Handling a Wrapper Type
-----------------------

The point of using a wrapper type is to force the developer to handle all
cases, including the failing ones. Since the value is wrapped in an `Option`,
the only way to get the value back is to handle the case were the `Option`'s
variant is `Some` and when it is `None`. This is usually done using `match` or
`if let`. Example with `match`:

```rust
//      variable name     type                value
//
let     value           : u64               = 42;
let     wrapped_value   : Option<u64>       = Some(value);

match wrapped_value {
    Some(value) => {

        // This `let` is just here for the reader to understand that the value
        // is actually an `u64` now.
        //
        //      variable name     type                value
        //
        let     my_value        : u64               = value;

        println!("The value is: {}", my_value);
    }
    None => {
        println!("No value")
    }
}
```

Another way to deal with a wrapper type is to actually not handle it at all.
Example with `unwrap`:

```rust
//      variable name     type                value
//
let     value           : u64               = 42;
let     wrapped_value   : Option<u64>       = Some(value);

// We can "unwrap" `Option<u64>` to `u64` using the method `.unwrap()`:
let     value_back      : u64               = wrapped_value.unwrap();
println!("The value is: {}", value_back);
```

Both codes do the same but the difference is that one actually handles the case
where the `Option` is `None`, while the other panics when the `Option` is
`None`. Panicking means that your program is going to stop abruptly. It's a
bit like a `try...catch/except` where the program does not handle the
exception.

By now you should notice that using the `Option` pattern actually forces the
developer to deal with the `None` case. For example, in Rust this code won't
compile:

```rust
//      variable name     type                value
//
let     value           : u64               = 42;
let     wrapped_value   : Option<u64>       = Some(value);

// Crashes with something like "`Display` is not implemented for
// `Option<u64>`".
//
// If you are curious to understand this error: this is because `{}` requires
// that the type implements the trait `Display`.
//
// https://doc.rust-lang.org/nightly/std/fmt/trait.Display.html
//
println!("The value is: {}", wrapped_value);
```

In programming languages that have a `NULL` this code would have compiled
successfully. This means that if you didn't handle the `NULL` case (because
you are only human and you forgot, oops), this will break at runtime. Maybe it
will break during a Q&A testing, maybe it will break in production... Or maybe
you wrote a test somewhere to make sure this case is handled properly. If you
did write a test to ensure this won't break at runtime, you have effectively
increased the number of line of codes and maintenance cost of your application.

Though it is important to note that some modern languages that have a `NULL`
handle the `NULL` case using a special marker on the type (usually `?`). This
is out of the scope of this article but if you are curious about it you could
check "null safety" in Swift and Kotlin.

The TL;DR here is: even though it feels cumbersome, forcing the developer to
handle exceptions (`None` and `Err` cases) is actually good for the quality of
life. In the next section we will see how we can make it handy instead of
cumbersome.

Manipulating Wrapper Types
--------------------------

`Option` and `Result` share a lot of common methods like: `unwrap`, `map`,
`and_then`, `or`, `or_else`, `unwrap_or_else`, etc... These methods can be use
to make transformations in and on the wrapper type itself. For example, you can
transform an `Option<u64>` to an `Option<String>` using `.map()`:

```rust
//      variable name     type                value
//
let     value           : u64               = 42;
let     wrapped_value   : Option<u64>       = Some(value);

// `.map()` takes a closure that has, in this case:
//  -  in input: `u64`
//  -  in output: `String`
let     transformed     : Option<String>    =
    wrapped_value.map(|value| value.to_string());
```

But you can also make transformations that will get rid of the wrapper type.
For example, using `unwrap_or_else` you can unwrap an `Option<String>` to a
`String` and provide a default value in case the `Option` is `None`:

```rust
//      variable name     type                value
//
let     wrapped_status  : Option<String>    = None;

// This will print "n/a" because the wrapped_status is `None`. Otherwise it
// would have print the status inside the wrapper type.
//
println!(
    "System current status: {}",
    wrapped_status.unwrap_or_else(|| "n/a".to_string()),
);
```

The real power comes in when all these methods can be combined and chained
together to express an idea:

```rust
fn is_running(wrapped_status: Option<String>) -> bool {
    wrapped_status
        // Returns true if the status is "connected"
        .map(|status| status == "connected")
        // Try to unwrap it but return false if it can't
        .unwrap_or_else(|| false)
}
```

Handling Result Like a Boss
---------------------------

`Result` is a different beast:
 -  Just like `Option` it has 2 variants: `Result::Ok` and `Result::Err`. They
    also are directly accessible because they are in the "std prelude". (So
    you can write directly `Ok` and `Err`.)
 -  But it has 2 type arguments: one for its value used in the `Result::Ok`
    variant (which is similar to `Option::Some`) and another one used for its
    error called `Result::Err`.

Because the error case has a type, `Result` are harder to handle than `Option`.
This is because the error type must be the same if you want to chain things
with `Result`. For example, the following code won't compile:

```rust
//      variable name     type                value
//
let     input           : Vec<u8>           = vec![52, 50];

// This code will try to read a sequence of bytes as a string and then parse
// it to an integer but it will fail.
//
// It fails because:
//  -  `String::from_utf8` will return an error of type `FromUtf8Error`
//  -  `u64::from_str_radix` will return an error of type `ParseIntError`
//
let     value           : Result<u64, _>    =
    String::from_utf8(input)
        .and_then(|string| u64::from_str_radix(&string, 10));
```

This is unfortunate and this makes Rust's `Result` a bit cumbersome. To solve
this you will need one of these 2 crates:

 -  if you are building a binary: [`anyhow`](https://crates.io/crates/anyhow);
 -  if you are building a library:
    [`thiserror`](https://crates.io/crates/thiserror).

Let's focus on using `anyhow` for now as it is the easiest to understand.
`anyhow` provides its own `Result`, a trait `Context` and a `bail!()` macro.

The `Result` of anyhow is capable to transform any kind (or almost) of error to
its own error type. This means you now have a common error type for any
error of any library.

The `Context` trait extends any error of any library by adding the methods
`.context()` and `.with_context()`. This allow the developer to give an error
message depending on its context (thus its name) and at the same time transform
the error to an `anyhow` error, thus making it compatible.

Finally the `bail()` macro is handy when you want to return immediately an
error. This is kinda like `return Err(...)`. We will talk about it in the next
part of this blog post serie.

Let's now fix our previous example using `anyhow`:

```rust
// The trait `Context` needs to be in scope for the method `.context()` to be
// available.
//
use anyhow::Context;

//      variable name     type                value
//
let     input           : Vec<u8>           = vec![52, 50];

// this code will try to read a sequence of bytes as a string and then parse
// it to an integer
let     value           : anyhow::Result<u64> =
    String::from_utf8(input)
        // Transform `FromUtf8Error` to `anyhow::Error` and add a message
        .context("Cannot read byte sequence as UTF-8")
        .and_then(|string| {
            u64::from_str_radix(&string, 10)
                // Transform `ParseIntError` to `anyhow::Error` and add a
                // custom message generated with a closure
                .with_context(|| {
                    format!("Cannot parse string as integer: {}", string)
                })
        });
```

Now that our errors are compatible, we can chain the wrapper methods together
(`.and_then`).

Another important point to note about the `.context()` method is that it also
also available on `Option`. Yes, that means you can transform an `Option` to a
`Result` easily and add a message of your choice:

```rust
use anyhow::Context;

//      variable name     type                        value
//
let     wrapped_status  : Option<String>            = None;
let     result_status   : anyhow::Result<String>    =
    wrapped_status
        .context("Could not get system status!");
```


To summarize what we have learned in this section, I would give you this real
life example
[shared on Reddit](https://www.reddit.com/r/rust/comments/mjaoq7/unwraps_everywhere/)
recently at the time I wrote this blog post (which has inspired it):

```rust
// Original code not handling the error/none cases
i64::from_str_radix(
    j.result
        .unwrap()
        .as_str()
        .unwrap()
        .trim_start_matches("0x"),
    16,
).unwrap()
```

If you do want to handle all the wrapper types and avoid panicking, you can
chain all the `Result` and `Option` together using `anyhow`:

```rust
// Updated code handling the error/none cases

use anyhow::Context;

//      variable name     type                    value
//
let     result          : anyhow::Result<i64>   =
    j.result
        // We need to avoid taking ownernship of the field `result`
        .as_ref()
        // Transform whatever this is to `anyhow::Result`
        .context("No valid input found")
        // We don't know what this is but `.as_str()` seems to return a
        // wrapper type... probably only if valid UTF-8 (it's just a guess)
        .and_then(|result| result.as_str().context("Not valid UTF-8"))
        // Oh but we can still transform our string reference using map!
        .map(|string| string.trim_start_matches("0x"))
        // Parse the hexadecimal but convert the result to `anyhow::Result`
        .and_then(|string| {
            i64::from_str_radix(string, 16).context("Could not parse integer")
        });

match result {
    Ok(value) => {
        println!("The value is: {}", value);
    }
    Err(err) => {
        eprintln!("Error: {}", err);
    }
}
```

In the next article we will see how to use the `?` operator and the crate
`thiserror`.
