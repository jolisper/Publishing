# Rust Key Concepts (I): Ownership System

![Rusty gears](./assets/Rusted_Gears.jpg)

Rust has specific design choices and feature selections that make the language unique. Let’s dive into the key concepts that lead us to a deeper understanding of the language.

## Ownership System

Ownership is hands down one of the first big ideas a newcomer to Rust has to wrap their head around. To really grasp Rust's ownership system, some background knowledge about variables, memory regions, and memory management is essential. This foundation will help you develop a solid mental model of what's going on. However, instead of diving into all that theory right away, let's start by building a practical sense of how the ownership system works. Let's set up a problem scenario and see how we can tackle it.

In Rust, when we write something like this:

```rust
let message = String::from("Hello Rust!");
```

We are doing more than simply assigning the “Hello Rust” string to the variable *message*. We’re binding the string — the value — to a scope.

Which scope? The one where the variable is defined.

```rust
{  
   let message = String::from("Hello Rust!");  
}
```

Statically, the syntax defines the scope as the block enclosed by the opening and closing braces.

```rust
t0 {  
t1   let message = String::from("Hello Rust!");  
t2 }  
t3
```

Dynamically speaking, the execution flow proceeds top-down, from the opening brace to the closing brace, spanning from *t0* to *t3*.

At *t0,* a new scope is defined.

At *t1,* a variable message is declared and initialized, allocating memory for the string. The lifetime of the string value is also bound to the scope of the variable message.

When execution reaches *t2,* the variable *message* goes out of scope, and the value associated with the variable is dropped from memory, deallocating the memory reserved at *t1*.

At *t3,* the program flow continues after the block, but the *message* variable is no longer accessible because it was dropped when the block exited.

In Rust parlance, we would say that *The message variable is the owner of the string ‘Hello Rust!’.* Technically, Rust couples the lifetime of a value in memory with the scope of the variable associated with that value.

If you come from a memory-managed language, this behavior might not impress you because it’s exactly what you expect — you don’t even need to think about it. However, the key difference is that this mechanism is determined at compile time. Rust shifts the memory management problem from run time to compile time, so you don’t incur extra run time overhead for automatic memory management.

Another important distinction compared to a typical garbage collection (GC) algorithm is that Rust’s memory management is deterministic — you know exactly when a value will be dropped. In contrast, with a GC, you can’t predict precisely when it will pause your application thread to perform garbage collection.

From the perspective of a C programmer, the benefits are more tangible because, in C, memory management is manual. With Rust, the compiler prevents many common problems associated with manual memory management, such as use-after-free errors, double-free errors, some memory leaks, dangling pointers, and out-of-bounds indexes.

As we can see, the semantics of assigning a value to a variable in Rust are denser than in other programming languages because they implicitly include an ownership declaration.

Rust has clear rules about ownership:

* Each value in Rust has an *owner*.
* There can only be one owner at a time.
* When the owner goes out of scope, the value will be dropped.

These rules are checked at compile time, which means that if any of them are violated, we’ll get a compilation error, preventing us from generating an executable with memory management deficiencies.

> Rust shifts the memory management problem from run time to compile time.

Let’s keep that in mind and explore how the ownership system affects Rust’s programming.

### Ownership Moving

Here is a slightly more complex code:

```rust
fn print_message(message: String) {  
  println!("{message}");  
}

fn main() {  
   let message = String::from("Hello Rust!");

   print_message(message);  
}
```

The *message* variable is assigned in the main function's scope and passed as an argument to another function that prints the message. This code compiles successfully, which means that all the ownership rules are upheld in this program.

The program not only compiles but also runs without any issues.

```none
$ cargo run  
   Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.00s  
    Running `target/debug/rust-kc-p1`  
Hello Rust!
```

Now, we want to print not only the message but also its size after printing the message itself.

```rust
fn print_message(printable_message: String) {  
  println!("{message}");  
}

fn print_message_size(sized_message: String) {  
   println!("{}", message.len());  
}

fn main() {  
   let message = String::from("Hello Rust!");

   print_message(message);

   print_message_size(message);  
}
```

To achieve this, we added another function to calculate and print the size, calling it after the first message was printed. However, there’s a problem: the compiler complains about our latest addition.

```
$ cargo run  
  Compiling rust-kc-p1 v0.1.0 (/home/jolisper/src/rust-key-concepts/rust-kc-p1)  
error[E0382]: use of moved value: `message`  
 --> src/main.rs:16:24  
   |  
12 |     let message = String::from("Hello Rust!");  
   |         ------- move occurs because `message` has type `String`, which does not implement the `Copy` trait  
13 |  
14 |     print_message(message);  
   |                   ------- value moved here  
15 |  
16 |     print_message_size(message);  
   |                        ^^^^^^^ value used here after move  
   |
```

Rust’s compiler error messages are usually helpful and often suggest ways to fix the problem. But to get the most out of them, you need a good grasp of how the ownership system works.

**Every time the Rust compiler stops you, it’s because a design decision must be taken.** Any design decision comes with its trade-offs. My advice here is not to rush and to solve the problem quickly or blindly follow the suggestions. Instead, invest the time to really understand why the error occurs; it is the best option in the long run.

I will say it again.

> Rust shifts the memory management problem from run time to compile time.

That means we have to solve more problems before our program can run. Spending more time on this part of the process is completely natural. Keep that in mind the next time the anxiety of getting your program running starts creeping up your fingertips.

Coming back to the code, the key thing here is that the compiler tells us we’re trying to use a *moved value*. What does this mean? Let’s revisit Rust’s ownership rules:

* Each value in Rust has an *owner*.
* **There can only be one owner at a time.**
* When the owner goes out of scope, the value will be dropped.

Let’s focus on the second rule. This clearly states that there can only be one owner for each value, but this doesn’t mean the owner must remain the same throughout the entire program’s execution. In fact, ownership can change hands during the program’s execution. For instance, when a value is assigned to another variable, added to a vector, or stored on the heap, ownership of the value moves from its original location to the new one. And that’s exactly what happens in this case.

When we first bind the *“Hello Rust!”* string to the variable *message*, that variable becomes its owner. However, when we pass the *message* variable as an argument for the *print_message()* function, the ownership of the value is transferred to the function’s parameter variable, *printable_message*. As a result, the original variable, *message*, loses its ownership of the value because it was moved. After this, the *message* variable in the main function is no longer valid, and attempting to use it again will result in a compilation error.

It's important to note that the scope of the string *“Hello Rust!”* changes when ownership is transferred. The new scope is defined by the *print_message* function. This means that when the *print_message* function returns, the *printable_message* parameter goes out of scope, resulting in the string being dropped. In cases like this, it is common to say that a function *consumes* the value.

The process described in the previous paragraph also applies to the first version of the code, where the *print_message()* function is the only additional function defined. However, since there is no attempt to reuse the *message* variable, no ownership rules are violated, and the code compiles successfully.

From here, things get interesting because Rust begins to reveal its full arsenal. There are several ways to resolve the issue, each with its trade-offs. It’s up to us, the programmers, to decide which path is best for our problem.

The ownership system is so prevalent in Rust that talking about it implies touching several other language features and concepts. To keep ownership at the center, in the rest of the article, we'll touch on them briefly, without deep dives, just to explain different aspects of the ownership system. You might feel tempted to take a bigger detour into those topics when that happens, but my humble advice is to take a deep breath, relax, and keep reading to build a practical sense of how the ownership system works before jumping to them. Consider these as glimpses into future articles.

### Use of Moved Value

Now that we've set up the problem, we’re ready to dive into the options for solving it. We’re not aiming to cover every possible solution here. Instead, we’ll explore a handful of approaches that let us dig into the nuances of the ownership system and put the basics we covered earlier to the test.

#### Return Ownership Back

One way to overcome the use of a moved value error is to make sure that the function returns the string passed as an argument and use that return value to call the other function.

```rust
fn print_message(printable_message: String) -> String {  
  println!("{printable_message}");  
  printable_message  
}

fn print_message_size(sized_message: String) {  
   println!("{}", sized_message.len());  
}

fn main() {  
   let mut message = String::from("Hello Rust!");

   message = print_message(message);

   print_message_size(message);  
}
```

Also, since the value is returned, we can avoid the intermediate assign of the message variable and chain the functions calls:

```rust
fn main() {  
   let message = String::from("Hello Rust!");

   print_message_size(print_message(message));  
}
```

Maybe this isn’t the first option in practice, but it illustrates how a function can take and return ownership of a value. It’s important to note that this process does not require any additional allocation.

In terms of ownership, we are adding an extra moving step to solve the issue: the returned ownership in the *print_message* function. Interestingly, the value itself isn’t what matters here. For example, we can return a completely different *String* value from the function, and the code still compiles:

```rust
fn print_message(printable_message: String) -> String {  
   println!("{printable_message}");  
   String::from("Another string value")  
}

fn print_message_size(sized_message: String) {  
   println!("{}", sized_message.len());  
}

fn main() {  
   let mut message = String::from("Hello Rust!");

   message = print_message(message);

   print_message_size(message);  
}
```

Of course, this breaks the functionality, as we are now printing the size of a new *String* rather than the original one.

```
Hello Rust!  
20
```

This reveals an important point: under Rust’s ownership rules, the specific values assigned to variables are irrelevant. What truly matters is the *ownership flow* through the variables and function calls.

##### Ownership and Mutability

Rust does not provide immutable values out of the box but provides immutable variables by default. This variable mutability condition is inherited by the value that the variable refers to, which means that a value behind an immutable variable is immutable, too[^1]. If we want to declare a variable as mutable, we must do it explicitly using the `mut` keyword in the variable declaration. This principle that transmit the mutability condition of a variable to the value behind it is called: *inherited mutability*.

In the example that serves as a problem scenario, we are using an immutable variable that refers to a String value holding the message to print out. But observe the following subtle change.

```rust
fn print_message(mut printable_message: String) {
    printable_message.push_str(" From a function");
    println!("{printable_message}");
}

fn main() {
    let message = String::from("Hello Rust!");

    print_message(message);
}
```

When ownership is moved, the function that takes ownership can decide to change the mutability condition of that value. In this case, although the variable passed as argument is immutable, the function decided to change that condition and made the value mutable.

```
Hello Rust! From a function.
```

Here is another version using shadowing.

```rust
fn print_message(printable_message: String) {
    println!("{printable_message}");
}

fn main() {
    let scrolling = String::from("A long time ago ");

    // scrolling.push_str("far far away"); // error: cannot mutate immutable variable value

    let mut scrolling = scrolling;

    scrolling.push_str("in a galaxy far, far away...");

    print_message(scrolling);
}
```

This shows us an important relation between mutability and ownership: when they are moved, owned values can be switched from immutable to mutable.

#### Cloning

Another way to overcome the issue is to simply avoid moving the value by passing a clone of the original instead.

```rust
fn print_message(printable_message: String) {  
   println!("{printable_message}");  
}

fn print_message_size(sized_message: String) {  
   println!("{}", sized_message.len());  
}

fn main() {  
   let message = String::from("Hello Rust!");

   print_message(message.clone());

   print_message_size(message);  
}
```

In this version, we create a copy of the original string and pass that copy as an argument to the *print_message* function. No ownership transfer happens in this case, but we are expending the double memory until the end of *print_message()* because we are allocating a copy of the string.

The need to explicitly call the *clone()* function for cloning a value is considered an advantage because it’s made explicit that the cloning occurs. To make this call, the type must implement the *Clone* trait. If we define our custom type, such as a struct, and all its fields implement *Clone*, the implementation is derivable.

```rust
#[derive(Clone)]  
struct Person {  
   name: String,  
   age: u8  
}
```

In this type of solution, evaluating the cost of the cloning is important. If the value is small, like in this example, the performance cost could be irrelevant, but in other cases where the value is big or if the cloning is inside a loop creating a lot of copies, the cost may represent too much.

Cloning everywhere is considered an anti-pattern in Rust because calling the *clone()* function introduces performance overhead, as every call creates a new memory allocation. Let’s explore alternative solutions to the problem scenario and return to this later.

#### Reference Counting

Let’s revisit the second rule of the ownership system:

* There can only be one owner at a time.

This rule is concise and clear, but, put differently, what’s truly restricted is shared ownership of a value. At times, this rule might feel overly restrictive and could hinder valid patterns. For example, if we need to share an immutable value, what’s the issue with sharing its ownership?

In Rust, the reference counting smart pointer, *Rc\<T>*, provides shared ownership of type *T*. Reference counting is a memory management technique where an allocated value keeps a count of how many references (or owners) point to it.

```rust
use std::rc::Rc;

fn print_message(printable_message: Rc<String>) {  
   println!("{printable_message}");  
}

fn print_message_size(sized_message: Rc<String>) {  
   println!("{}", sized_message.len());  
}

fn main() {  
   let message = Rc::new(String::from("Hello Rust!"));

   print_message(message.clone());

   print_message_size(message.clone());  
}
```

When you clone an *Rc\<T>*, another pointer to the same allocation is created[^2], and the reference count is incremented. When an *Rc\<T>* goes out of scope, its drop logic decrements the reference count. If the count reaches zero, the value is deallocated.

This means that *Rc<T>* does not create a copy of the value *T*; it merely increments or decrements the reference count. As a result, this approach introduces some overhead because *Rc<T>* needs to keep track of how many owners the value *T* has. In situations where we want or need to share the same instance of an object across different parts of a program, the *Rc<T>* smart pointer comes in handy.

#### Using References

In the previous solutions:

* We return ownership back, using a function that takes and gives ownership back to the caller.

* Avoid ownership move by passing a deep copy of the value to the function with `clone().`

* We relaxed the "only one owner" ownership system rule using shared ownership through *Rc\<T\>* smart pointer.

Those solutions have one important thing in common: all three are pass-by-value solutions[^3]. Another possibility allows us to use the value in another scope while keeping the original ownership intact. We achieve that by using references.

```rust
fn print_message(printable_message: &String) {
    println!("{printable_message}");
}

fn print_message_size(sized_message: String) {
    println!("{}", sized_message.len());
}

fn main() {
    let message = String::from("Hello Rust!");

    print_message(&message);

    print_message_size(message);
}
```

We can think of references as pointers that allow us to access data without taking ownership of it. The difference with a typical C pointer is that we can't make any pointer arithmetic on them, also they are restricted to specific rules. References add two more rules to the ownership system:

- At any given time, you can have *either* one mutable reference *or* any number of immutable references.
- References must always be valid.

The first rule tells us that references come in two flavors: *mutable*, and *immutable*. The first ones are denoted by `&` and the second ones are denoted by `&mut`. We can have any number of immutable references to a value or only one mutable reference to the same value at the same time. That's the reason why the immutable references are often called shared references, and the mutable ones are typically referred to as exclusive references. You can share or mutate, but not both.

Allowing access to a value in another scope without transferring ownership opens the door to new problems. The ownership system must ensure that the references point exclusively to non-dropped values. That is what the second rule is all about.

Notice that the *print_message()* function was changed to receive a reference of the message, but the *print_message_size()* is still taking ownership of the value. Because the *print_message()* function doesn't take ownership of the message value, the owner is still the message variable defined in the main function until the second function is called, and that does not break any rule of the ownership system. The code compiles. 

Now let's explore what happens if we invert the call function order:

```rust
fn main() {
    let message = String::from("Hello Rust!");

    print_message_size(message);

    print_message(&message); // error: borrow of moved value.
}
```

The compile error changes, telling us that we are trying to take a reference of a value that was moved. A reference is valid when its lifetime is equal to or less than the lifetime of the value that it points to.  

```rust
fn main() {
    let msg_ref;
    {
        let message = String::from("Hello Rust!");
        msg_ref = &message; // error: `message` does not live long enough
    } 

    print_message(msg_ref);
}
```

In this example, we define a reference variable in the main function scope and initialize it with a reference to a value bonded to an internal scope smaller than the full main function scope. 

When we use that reference after the inner scope has ended, the compiler tells us that the value behind the reference does not live long enough. In other words, the compiler tells us that the reference lifetime is greater than the lifetime of the value that's pointing to. Using a reference that points to a dropped value leads to undefined behavior, so the compiler inhibits that possibility.

In Rust parlance, when a function takes a reference as a parameter, we say that *the function borrows the value*. Besides the ownership moving semantics, *borrowing* is the other complementary concept that is prevalent is Rust. The *borrow checker* is the part of the compiler that enforces the Rust's ownership and borrowing rules at compile time.

A full discussion about borrowing must include an explanation of lifetimes, especially generic ones. That topic demands special attention, but since we need to understand the basics of the ownership system first, we’ll leave it for another article.

> Ownership is the path to the lifetimes. Ownership leads to borrowing. Borrowing leads to references. References lead to lifetimes.

#### Take, Replace (and Swap)

These two options have very specific use cases. In the context of this example, they are more hacks than real solutions, but it is good to know they exist.

```rust
fn print_message(printable_message: String) {  
   println!("{printable_message}");  
}

fn print_message_size(sized_message: String) {  
   println!("{}", sized_message.len());  
}

fn main() {  
   let mut message = String::from("Hello Rust!");

   print_message(std::mem::take(&mut message));  

   print_message_size(message);  
}
```

Here, we use the generic function *mem::take\<T\>(&mut T)*; this function returns the value behind the mutable reference and replaces its original value with the type's default value. If we run this code, the output is:

```
Hello Rust!  
0
```

That’s not the correct result because *print_message()* received the original value, but the value of the variable message was replaced with the default value, which, in the case of the *String* type, is an empty string.

Why does this work in terms of ownership? Because *mem::take(&mut T) -> T* receives a mutable reference of *T* from which extracts the current value, returns it as an owned value, and, in the middle, replaces the original value with the default value for *T*.

As a result, from the perspective of the ownership flow, nothing changes; the *message* variable never loses ownership of a value. It's just that the good old *mem::take()* swoops in, sneakily swipes the current value from *message*'s pocket, and replaces it with a new one of equal *weight*. The sleight of hand is so smooth that *message* remains blissfully unaware of what transpired, and the ownership rules remain perfectly intact.

Technically, `mem::take(&mut T) -> T` is a special case of `mem::replace(&mut T, T) -> T` with the default value of *T* as a second argument; in fact, *mem::take()* uses *mem::replace()* internally.

```rust
fn main() {  
   let mut message = String::from("Hello Rust!");

   let other_message = String::from("Another string value"); // len = 20

   print_message(std::mem::replace(&mut message, other_message));  

   print_message_size(message);  
}
```

That’s prints:

```
Hello Rust!  
20
```

Just to show the third friend of this group of functions, we’ll see an example of *mem::swap(&mut T, &mut T)*, which does exactly what we expect.

```rust
fn main() {  
   let mut message1 = String::from("Hello Rust!");  
   let mut message2 = String::from("Goodbye Rust!");

   std::mem::swap(&mut message1, &mut message2);  

   print_message(message1);

   print_message(message2);  
}  
```

Printing the result:

```
Goodbye Rust!  
Hello Rust!
```

#### Redesign to Avoid Ownership Transfer

Another valid option is to rethink how the code manages ownership of values throughout its structure.

```rust
fn print_message(printable_message: String) {  
   println!("{printable_message}");  
   println!("{}", printable_message.len());  
}

fn main() {  
   let message = String::from("Hello Rust!");

   print_message(message);  
}
```

Reconsidering the function call order while keeping the final result intact is valid. So, rethink the way that the code is handling ownership of the values through our code should serve us as a trigger to build a better solution, it forces us to find other way to get the same result, if we have the intention to simplify things, in general that lead us to better solution. 

#### Inlining

This can be thinking as a special case of the previous option. If we inline the *print_message_size()* function call, we end up with this:

```rust
fn main() {
   let message = String::from("Hello Rust!");

   println!("{printable_message}"); // manual inlined function

   print_message_size(message);  
}
```

Removing the function call also eliminates the ownership transfer. While this could resolve the ownership issue, it is not always a practical solution, as it may lead to code duplication and future maintenance challenges.

Just to mention, Rust has a function attribute called `#[inline]`:

```rust
#[inline(always)]
fn print_message(printable_message: String) {
    println!("{printable_message}");
}

fn print_message_size(sized_message: String) {
    println!("{}", sized_message.len());
}

fn main() {
    let message = String::from("Hello Rust!");

    print_message(message);

    print_message_size(message);
}
```

But serves as a compiler hint and not as an actual order that could solve ownership problems. The code is still not compiling.

```
  --> src/bin/inlining.rs:15:24
   |
11 |     let message = String::from("Hello Rust!");
   |         ------- move occurs because `message` has type `String`, which does not implement the `Copy` trait
12 |
13 |     print_message(message);
   |                   ------- value moved here
14 |
15 |     print_message_size(message);
   |                        ^^^^^^^ value used here after move
   |
```

## Related Concepts

After exploring ways to overcome the 'use of moved value' scenario, it's worth learning about some related concepts to complete the ownership picture.

### Avoid cloning

The *clone()* method creates a deep copy of a value, which in particular does not have anything bad at prior. However, overuse can produce unwanted performance problems and memory bloat, especially if we use it as an easy way to avoid ownership issues.

That configures the typical Rust tension between cloning and the ownership system, which is why many Rust programmers raise flags against using *clone()*. Even so, we must escape from any dogmatic crusade in software design.

What's really important here is to remember that:

> **Every time the Rust compiler stops you, it’s because a design decision must be taken.**

Cloning comes with a cost, and that cost should be carefully considered in the context of our design. It's worth taking the time to ask ourselves a few questions before simply writing `.clone()` after naming a value:

- Can ownership be taken instead?
- Would a reference suffice here?
- Could shared ownership through a smart pointer address the problem?

As a rule of thumb, use `.clone()` only when there is a specific reason for creating a new copy.

### Copy Trait

For a moment, let's focus on another simple example:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn print_x(p: Point) {
    println!("{}", p.x);
}

fn print_y(p: Point) {
    println!("{}", p.y);
}

fn main() {
    let p1 = Point { x: 10, y: 20 };

    print_x(p1);

    print_y(p1); // error: use of moved value: `p1`
}
```

This code is similar to the previous example in the sense that we are getting the same error from the compiler. But the kind of the types that our *Point* struct is based on allows us to use another way of solving the ownership issue.

```rust
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}
```

Because our struct uses only copy values, we can derive the Copy trait for Point. Copy values in Rust refer to values of types that implement the Copy trait. This trait allows for the duplication of values without transferring ownership. The Copy trait is automatically implemented for simple, fixed-size types, like integers, floating-point, booleans, and character types.

The use of the Copy trait in our custom types implies design decisions similar to those made with the *clone()* method use. The key difference lies in their purposes: `Copy` is a marker trait that signals to the compiler that a type's value can be duplicated by simply copying its bits in memory. In contrast, the `Clone` trait defines an explicit method for creating a copy of a value. For types implementing `Copy`, values are automatically duplicated instead of being moved when passed to a function or assigned to another variable. In this sense, the `Copy` trait represents a controlled exception to Rust's default move semantics in the ownership system.

### Interior Mutability

Some smart pointers (*Cell, RefCell, RwLock, Mutex*) allows the use of the pattern known as *interior mutability*. Interior mutability allows us to mutate a value through a shared reference. I can almost hear you saying:

> *Wait! What? We've seen that there are two types of references: mutable and shared. We can't mix them—we can either share or mutate through them, but not both!*

My answer is, *'Yes, you are right,'* but this pattern is an exception to the ownership rules we've seen before—in more ways than one.

First, it clearly allows something that is otherwise forbidden:

- At any given time, you can have *either* one mutable reference *or* any number of immutable references.

With this approach, we can have multiple shared (immutable) references while still allowing mutation of the value through any of them. In other words, it violates the principle of inherited mutability

Second, inherited mutability is enforced at compile time by (of course) the compiler. However, interior mutability relies on run time checks for safety. This means that the this family of containers enforces the necessary borrowing rules dynamically, allowing mutation through shared references while maintaining safety within their defined constraints. However, unlike inherited mutability, which ensures safety at compile time, this approach introduces a run time cost, as the rules are enforced programmatically during execution.

Third, because the rules of interior mutability are checked at run time, violations result in run time errors. This means that, for the interior mutability portion of a program, the occurrence of memory management issues depends on the program’s logic and the execution paths determined by run time conditions. These conditions are outside the compiler's scope, preventing it from catching such issues. This breaks Rust’s promise of catching these types of problems at compile time.

Why does interior mutability even exist if it comes with all those drawbacks? The Rust compiler is quite conservative about what constitutes a valid program. In terms of memory safety, it prefers to be strict, even at the cost of having false positive classifications of unsafe programs. Using group theory terminology, we could say that the set of memory-safe programs is a superset of the programs allowed by the Rust compiler. This leaves a whole subset of valid programs that cannot be represented without the interior mutability pattern.

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<u32>, // instead of: "count: u32"
}

impl Counter {
    fn new() -> Counter {
        Counter { count: Cell::new(0) }
    }

    fn increment(&self) {
        // self.count += 1; // cannot assign to `self.count`, which is behind a `&` reference
        self.count.set(self.count.get() + 1);
    }

    fn get(&self) -> u32 {
        self.count.get()
    }
}

fn main() {
    let counter = Counter::new();
    println!("Initial count: {}", counter.get());
    counter.increment();
    println!("Incremented count: {}", counter.get());
}
```

In the example above the interior mutability pattern allow us to expose an immutable interface of the counter but and the same time encapsulating the mutation of the counter in the increment method.

## Summary

Based on simple examples we learnt the rules of the ownership system, and we went through several ways to resolve ownership issues:

* We can take and return ownership of values without cutting the ownership flow.
* We can pass a copy of the value avoiding the ownership move.
* We can share ownership using reference counting.
* We can give access to values without renounce to ownership via borrowing using references.
* And we can refactor the code to remove the ownership transfer rethinking the function call order or manually inlining code.

Witch option we should pick? Well, like many interesting answers in life begins with the word: depends. There is no magic rule to follow any time, because the selected option would depends on the problem you have into hands. 

As general guidance immutable and mutable references are usually the most flexible option, because allow access to a value without transferring ownership, avoiding the need for cloning or moving data. One typical exception are the constructors where taking ownership of the values is usually the best option if we want to keep the ownership flows simple and understandable.

```rust
struct Person {
    name: String,
}

impl Person {
    fn new(name: String) -> Person { // instead of: fn new(name: &String) -> Person
        Person {
            name, // instead of: name: name.clone()
        }
    }

    fn get_name(&self) -> &String { // instead of: fn get_name(&self) -> String
        &self.name // instead of: self.name.clone()
    }
}
```

The main advantage to make an interface that returns references is we left the callers and the client code decide if the reference will suffice or if they need to clone the value instead.

```rust
fn main() {
    let person = Person::new(String::from("John"));

    let name = person.get_name();

    let other_john = Person::new(name.clone());

    println!("{} {}", person.name, other_john.name);
}
```

For other more complex interactions where we need to share some data or custom type across the program logic we should consider the *Rc\<T\>* approach. And if we are follow a more functional approach where immutability is central, make interfaces that returns  cloned owned value instead of references would help.

We must remember that, from a static analysis perspective, memory management in Rust is enforced syntactically[^4]. The compiler relies solely on the code’s structure and ownership rules (like moves, borrows, and lifetimes) to statically validate memory safety, even though memory operations are inherently dynamic at runtime.

This strict compile-time checking is a key contributor to Rust's learning curve. We're not just learning ideas like ownership and borrowing—we have to write code in a way that makes the borrow checker happy, which often requires adjusting code that seems correct or restructuring logic to meet the compiler’s expectations. While this can feel restrictive, it reflects Rust’s core trade-off: rejecting some valid patterns upfront to eliminate memory vulnerabilities entirely.

## Final Words

From a simple "use of moved value" scenario, we explored several concepts and techniques related to the ownership system. We also examined ways to relax or even defy the static rules with dynamic replacements. I'm confident that we've now built a practical understanding of how the ownership system works, which is essential for tackling more complex scenarios. Stay tuned!

[^1]: There are two exceptions to this: the use of unsafe code to bypass the mutability restrictions and the use of shareable mutable containers Cell, RefCell, and OnceCell, which allows us to mutate the data inside an immutable container.

[^2]: Internally, *Rc<T>* uses *PhantomData<T>*, a zero-cost abstraction, to represent ownership without storing an actual instance of T. Including *PhantomData<T>* allows the compiler to track ownership and enforce type relationships at compile time without spend more memory. 

[^3]: The *Rc<T>* solution is more subtle because it implies the pass-by-value of the smart pointer itself instead of the wrapped value.

[^4]: While Rust’s memory management is enforced through code structure (e.g., moves, borrows, and lifetimes), ownership itself is a semantic concept. Rust’s compile-time enforcement is rooted in memory safety semantics, even though it relies on syntactic patterns to express those rules.

___

References:

[YouTube: 5 Deadly Rust Anti-Patterns to Avoid.](https://www.youtube.com/watch?v=SWwTD2neodE)

[Book: Idiomatic Rust by Brenden Matthews - Chapters 1 and 2.](https://www.manning.com/books/idiomatic-rust)

[Book: Rust for Rustaceans by Jon Gjengset - Chapters 1 and 3.](https://rust-for-rustaceans.com/)
