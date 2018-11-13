# Intro
If you are a programmer and you like Rust you might be aware of the future movement that is buzzing around the Rust community. Great crates have embraced it fully (for example Hyper) so we must be able to use it. But if you are an average programmer like me you may have an hard time understanding them. Sure there is the official Crichton's tutorial but, while being very thorough, I've found it very hard to put in practice.

I'm guessing I'm not the only one so I will share my findings hoping they will help you to become more proficient with the topic.

# Futures in a nutshell
Futures are peculiar functions that do not execute immediately. They will execute in the future (hence the name). There are many reasons why we might want to use a future instead of a standard function: performance, elegance, composability and so on. The downside is they are hard to code. Very hard. How can you understand causality when you don't know when a function is going to be executed?

For that reason programming languages have always tried to help us poor, lost programmers with targeted features.

# Rust's futures
Rust's future implementation, true to its meme, is a work in progress. So, as always, what are you reading can already be obsolete. Handle with care.

Rust's futures are always Results: that means you have to specify both the expected return type and the alternative error one.

Let's pick a function and convert it to future. Our sample function returns either u32 or a Boxed Error trait. Here it is:
```rust
fn my_fn() -> Result<u32, Box<Error>> {
    Ok(100)
}
```
Very easy. Not look at the relative future implementation:
```rust
fn my_fut() -> impl Future<Item = u32, Error = Box<Error>> {
    ok(100)
}
```
Notice the differences. First the return type is no longer a Result but a impl Future. That syntax (still nightly-only at the time of writing) allows us to return a future. The `<Item = u32, Error = Box<Error>>` part is just a more verbose way of detailing the returned type and the error type as before.

* You need the `conservative_impl_trait` nightly feature for this to work. You can return a boxed trait instead but it's more cumbersome IMHO.

Notice also the case of the `Ok(100)` returned value. In the Result function we use the capitalized Ok enum, in the future one we use the lowercase ok method.

* Rule of the thumb: use lowercase return methods in futures.

All in all is not that bad. But how can we execute it? The standard function can be called directly. Note that since we are returning a Result we must `unwrap()` to get the inner value.
```rust
let retval = my_fn().unwrap();
println!("{:?}", retval);
```
As futures return before actually executing (or, more correctly, we are returning the code to be executed in the future) we need a way to run it. For that we use a Reactor. Just create it and call its run method to execute a future. In our case:
```rust
let mut reactor = Core::new().unwrap();

let retval = reactor.run(my_fut()).unwrap();
println!("{:?}", retval);
```
Notice here we are unwrapping the run(...) return value, not the `my_fut()` one.

Really easy.

# Chaining
One of the most powerful features of futures is the ability to chain futures together to be executed in sequence. Think about inviting your parents for dinner via snake mail. You send them a mail and then wait for their answer. When you get the answer you proceed preparing the dinner for them (or not, claiming sudden crippling illness). Chaining is like that. Let's see an example.

First we add another function, called squared both as standard and future one:
```rust
fn my_fn_squared(i: u32) -> Result<u32, Box<Error>> {
    Ok(i * i)
}
```
```rust
fn my_fut_squared(i: u32) -> impl Future<Item = u32, Error = Box<Error>> {
    ok(i * i)
}
```
Now we might call the plain function this way:
```rust
let retval = my_fn().unwrap();
println!("{:?}", retval);

let retval2 = my_fn_squared(retval).unwrap();
println!("{:?}", retval2);
```
We could simulate the same flow with multiple reactor runs like this:
```rust
let mut reactor = Core::new().unwrap();

let retval = reactor.run(my_fut()).unwrap();
println!("{:?}", retval);

let retval2 = reactor.run(my_fut_squared(retval)).unwrap();
println!("{:?}", retval2);
```
But there is a better way. A future, being a trait, has many methods (we will cover some here). One, called and_then, does exactly what we coded in the last snippet but without the explicit double reactor run. Let's see it:
```rust
let chained_future = my_fut().and_then(|retval| my_fut_squared(retval));
let retval2 = reactor.run(chained_future).unwrap();
println!("{:?}", retval2);
```
Look at the first line. We have created a new future, called `chained_future` composed by `my_fut` and `my_fut_squared`.

The tricky part is how to pass the result from one future to the next. Rust does it via closure captures (the stuff between pipes, in our case `|retval|`). Think about it this way:

1. Schedule the execution of `my_fut()`.
2. When `my_fut()` ends create a variable called retval and store into it the result of `my_fut()` execution.
Now schedule the execution of `my_fn_squared(i: u32)` passing retval as parameter.
3. Package this instruction list in a future called chained_future.
4. The next line is similar as before: we ask the reactor to run chained_future instruction list and give us the result.

Of course we can chain an endless number of method calls. Don't worry about performance either: your future chain will be zero cost (unless you Box for whatever reason).

* The borrow checked may give you an hard time in the capture chains. In that case try moving the captured variable first with move.

# Mixing futures and plain functions
You can chain futures with plain functions as well. This is useful because not every function needs to be a future. Also you might want to call external function over which you have no control.

If the function does not return a Result you can simply add the function call in the closure. For example, if we have this plain function
```rust
fn fn_plain(i: u32) -> u32 {
    i - 50
}
```
Our chained future could be:
```rust
let chained_future = my_fut().and_then(|retval| {
    let retval2 = fn_plain(retval);
    my_fut_squared(retval2)
});
let retval3 = reactor.run(chained_future).unwrap();
println!("{:?}", retval3);
```
If your plain function returns a Result there is a better way. Let's chain our `my_fn_squared(i: u32) -> Result<u32, Box<Error>>` function.

You cannot call and_then to a Result but the futures create have you covered! There is a method called done that converts a Result into a impl Future.
What does that means to us? We can wrap our plain function in the done method and than we can treat it as if it were a future to begin with!

Let's try it:
```rust
let chained_future = my_fut().and_then(|retval| {
    done(my_fn_squared(retval)).and_then(|retval2| my_fut_squared(retval2))
});
let retval3 = reactor.run(chained_future).unwrap();
println!("{:?}", retval3);
```
Notice the second line: `done(my_fn_squared(retval))` allows us to chain a plain function as it were a impl Future returning function. Now try without the done function:
```rust
let chained_future = my_fut().and_then(|retval| {
    my_fn_squared(retval).and_then(|retval2| my_fut_squared(retval2))
});
let retval3 = reactor.run(chained_future).unwrap();
println!("{:?}", retval3);
```
The error is:
```
   Compiling tst_fut2 v0.1.0 (file:///home/MINDFLAVOR/mindflavor/src/rust/tst_future_2)
error[E0308]: mismatched types
   --> src/main.rs:136:50
    |
136 |         my_fn_squared(retval).and_then(|retval2| my_fut_squared(retval2))
    |                                                  ^^^^^^^^^^^^^^^^^^^^^^^ expected enum `std::result::Result`, found anonymized type
    |
    = note: expected type `std::result::Result<_, std::boxed::Box<std::error::Error>>`
               found type `impl futures::Future`

error: aborting due to previous error

error: Could not compile `tst_fut2`.
```
The `expected type std::result::Result<_, std::boxed::Box<std::error::Error>> found type impl futures::Future` is somewhat misleading. In fact we were supposed to pass an impl Future and we used a Result instead (not the other way around). This kind of confusion is common using futures (see https://github.com/alexcrichton/futures-rs/issues/402). We'll talk about it in the second part of the post.

# Generics
Last but not least, futures work with generics without any magic.

Let's see an example:
```rust
fn fut_generic_own<A>(a1: A, a2: A) -> impl Future<Item = A, Error = Box<Error>>
where
    A: std::cmp::PartialOrd,
{
    if a1 < a2 {
        ok(a1)
    } else {
        ok(a2)
    }
}
```
This function returns the smaller parameter passed. Remember, we need to give and Error even if no error is expected in order to implement the Future trait. Also, the return values (`ok` in this case) are lowercase (the reason is `ok` being a function, not an enum).

Now we can run the future as always:
```rust
let future = fut_generic_own("Sampdoria", "Juventus");
let retval = reactor.run(future).unwrap();
println!("fut_generic_own == {}", retval);
```
I hope you got the gist by now :). Until now it should make sense. You may ave noticed I avoided using references and used owned values only. That's because lifetimes do not behave the same using impl Future. I will explain how to use address them in the next post. In the next post we will also talk about error handling in chains and about the await! macro.

# Notes
If you want to play with these snippets you just need to add in your Cargo.toml file:
```
[dependencies]
futures="*"
tokio-core="*"
futures-await = { git = 'https://github.com/alexcrichton/futures-await' }
```
And put these lines on top of src/main.rs:
```rust
#![feature(conservative_impl_trait, proc_macro, generators)]
extern crate futures_await as futures;
extern crate tokio_core;

use futures::done;
use futures::prelude::*;
use futures::future::{err, ok};
use tokio_core::reactor::Core;
use std::error::Error;
```
The futures-await crate is not really needed for this post but we will use it in the next one.

Original article from [here](https://dev.to/mindflavor/rust-futures-an-uneducated-short-and-hopefully-not-boring-tutorial---part-1-3k3)
