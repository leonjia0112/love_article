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
