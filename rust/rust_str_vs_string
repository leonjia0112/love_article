# Rust: str vs String
Oct 12, 2017

As a Rust newbie, I was confused by the different ways used to represent strings. The “References and Borrowing” chapter of the Rust book uses three different types of string variables in the examples: ```String```, ```&String``` and ```&str```.

Let’s start with the difference between ```str``` and ```String```: ```String``` is a growable, heap-allocated data structure whereas ```str``` is an immutable fixed-length string somewhere in memory 1.

### String
If you’re a Java programmer, a Rust ```String``` is semantically equivalent to ```StringBuffer``` (this was probably a factor in my confusion, as I’m so used to equating ```String``` with immutable). As such, a ```String``` maintains a length and a capacity whereas a ```str``` only has a ```len()``` method. As an example:
```rust
let mut s = String::from("Hello, Rust!");
println!("{}", s.capacity()); // prints 12
s.push_str("Here I come!");
println!("{}", s.len()); // prints 24
```
```rust
let s = "Hello, Rust!";
println!("{}", s.capacity()); // compile error: no method named `capacity` found for type `&str`
println!("{}", s.len()); // prints 12
```
### &str
You can only ever interact with ```str``` as a borrowed type aka ```&str```. This is called a string slice, an immutable view of a string. This is the preferred way to pass strings around, as we shall see.

### &String
This is a reference to a ```String```, also called a borrowed type. This is nothing more than a pointer which you can pass around without giving up ownership. Turns out a ```&String``` can be coerced to a ```&str```:
```rust
fn main() {
    let s = String::from("Hello, Rust!");
    foo(&s);
}

fn foo(s: &str) {
    println!("{}", s);
}
```
In the above example, ```foo()``` can take either string slices or borrowed ```Strings```, which is super convenient. As such, you almost never need to deal with ```&String```s. The only real use case I can think of is if you want to pass a mutable reference to a function that needs to modify the string:
```rust
fn main() {
    let mut s = String::from("Hello, Rust!");
    foo(&mut s);
}

fn foo(s: &mut String) {
    s.push_str("appending foo..");
    println!("{}", s);
}

```
### Summary
Prefer ``&str`` as a function parameter or if you want a read-only view of a string; ```String`` when you want to own and mutate a string.
1. ```str``` data can live on the heap, stack or in the binary. [This](https://stackoverflow.com/questions/24158114/what-are-the-differences-between-rusts-string-and-str/24159933#24159933) excellent stackoverflow answer explains the scenarios for each. ↩


From: [Ameya's Blog](http://www.ameyalokare.com/rust/2017/10/12/rust-str-vs-String.html#fnref:1)
