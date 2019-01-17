http://blog.honeypot.io/errors-and-exceptions-in-rust/
# Errors and Exceptions in Rust
May 27th 2016, Giovanni Gapuano

## General Definitions
**Error**: An unusual and unexpected situation in the running program that can be resolved only by fixing the program (i.e. out of memory).

**Exception**: An expected and irregular situation that happens at runtime (i.e. read-protected file).

The main difference here is that errors are **unexpected** (and usually lead to exploitable bugs) and are often hard to predict (did someone whisper overflow?), while exceptions are expected (you know that they could happen) and usually lead to a crash of the application when not handled (for example when you want to modify a file that actually does not exist but you didn’t think about this possibility).

## How popular languages deal with exceptions

We could split programming languages into four categories based on the way they handle exceptions:

* Strict exception handling: the language provides full and explicit coverage to exceptions. Exceptions are defined as classes (either built-in and defined by the developer) that represent the type of exception they handle. Methods are shipped with a list of exceptions they can rise (explicitely or just by invoking other functions) and require to be handled with a `Try...catch`-like construct. Not handled exceptions lead to a compile-time error.
* Weak exception handling: everything that’s been said for the strict exception handling languages with the addition that developers are not required to handle exceptions and methods are not shipped with any definition of the exceptions they can rise demanding this task to the documentation. Exceptions that are not handled lead to an abortion of the application at run time.
* Exceptions-agnostic: exception handling is not provided by the language but can be implemented (usually with conditional statements and `goto`s).
* By monadic types: exception handling is realized with generic structs in which values are encapsulated.

The most popular language that implements the features of the first category is Java. The second category is basically huge, and includes most of the languages over here: *C++, Python, Ruby, PHP, JavaScript* and so on.

Usually handling exceptions in this very OOPish way is quite expensive in terms of resources. The C programming language does not have any knowledge about exceptions and most projects (the Linux kernel included) handle them with `if`s and `goto`s to jump to the block where a task gets rescued. It’s obvious that this trait asks developers to be completely aware about all the edge cases of their code and to write especially clean code.

The last category is the most appreciated inside the functional programming world. The concept is that instead of returning `NULL`s or raising exceptions in case of failures (i.e. an hashtable that does not include a given key), an option type is used that can either encapsulates a value or being empty. Haskell uses an option type called `Maybe` that accepts `Nothing` or `Just x` while Rust, Scala, OCaml, Swift and others use an Option type that accepts `None` or `Some(T)`. Through decostruction via pattern matching, the developer can handle both of these cases.

Let’s see shortly what in practice this does mean exemplifying with a language that despite is not purely functional, implements and makes extensive use of monadic types.

## Exception handling in Rust

This section wants to go a little deeper on what it’s been said previously. Please look at [this article]() and the [Rust book](https://doc.rust-lang.org/book/) for a great learning about this!

As said, talking about exceptions, the only tools Rust gives to developers are `Option<T>` and `Result<T, E>`. They are everything but magic, being so easy that you can implement them by yourself in a couple of lines, as done in the article linked above. And they are terribly optimized in compile-time too, so that their usage is almost cost-free.

### `Option<T>`
I’m the kind of person who prefers to leave the code to talk for itself. Please have a look at this snippet:
```
use std::collections::HashMap;

fn print_inventory(book_name: &str, book: Option<&u32>) {
    match book {
        Some(number) => println!("There are {} copies of {}", number, book_name),
        None         => println!("{} is out of stock :(", book_name)
    }
}

fn main() {
    let mut books: HashMap<&str, u32> = HashMap::new();
    books.insert("A silent voice", 5);
    
    // HashMap<&str, u32>::get() returns an Option<u32> (32-bit unsigned integer).
    let a_silent_voce = books.get("A silent voice");
    print_inventory("A silent voice", a_silent_voce);
    
    let spice_and_wolf = books.get("Spice and Wolf");
    print_inventory("Spice and Wolf", spice_and_wolf);
}
```
We define a dictionary containing the name of the books as key and the count of volumes in the stock as value. The `get()` method returns an `option` type that we want to deconstruct through pattern matching, extracting so its value if it’s present.

Instead of using pattern matching we can also `unwrap()` the value directly in this way. This method basically extracts the value from an `Option<T>`, assuming it `is_some()`.

It is quite a double edged weapon though. If the `Option<T>` `is_none()`, meaning that it has not a value encapsulated, it will `panic!` as you can see here.

```
use std::collections::HashMap;

fn print_inventory(book_name: &str, book: Option<&u32>) {
    // Option<T> provides two methods, is_some() and is_none().
    // The first one returns `true` if it encapsulates a value
    // or `false` if not. The second one does the opposite.
    if book.is_none() {
        println!("{} is out of stock :(", book_name);
    }
    else {
        let number = book.unwrap();
        println!("There are {} copies of {}", number, book_name);
    }
}

fn main() {
    let mut books: HashMap<&str, u32> = HashMap::new();
    books.insert("A silent voice", 5);
    
    // HashMap<&str, u32>::get() returns an Option<u32> (32-bit unsigned integer).
    let a_silent_voce = books.get("A silent voice");
    print_inventory("A silent voice", a_silent_voce);
    
    let spice_and_wolf = books.get("Spice and Wolf");
    print_inventory("Spice and Wolf", spice_and_wolf);
}
```

`panic!` is a macro defined in the Rust’s standard library that aborts the current thread (this means that our multi-threaded applications can be still alive too :)), causing most of the times the crash of the application, in the same flavor of a Ruby or Python’s unhandled exception.

We don’t care that our returned value is an `Option<T>` instead of the plain value as long as we don’t need to use it. Pattern matching, `unwrap()`, `unwrap_or()` and `unwrap_or_else()` are usually everything we need to keep our code safe from unexpected values that can be returned.

We can live in a world without `NULL`s. It sounds kinda like a dream, doesn’t it? But it’s real.


