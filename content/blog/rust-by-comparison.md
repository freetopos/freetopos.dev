+++
title = "Rust By Comparison"
date = 2019-08-29

draft = false
in_search_index = true
+++

> Edit: this is a reupload of the original post from my old blog. Some formatting and typos have been fixed.

I really like learning new programming languages, especially the ones that make you reconsider some fundamental assumptions you make when writing code or designing a piece of software. I've been playing with [Rust](http://rust-lang.org) for some time now and I can definitely say that it's one of these languages.

In a couple of words, one can say Rust is a high-level low-level programming language. What I mean by that is: Rust is a low-level language, in the sense that its programming model is closer to the hardware's so that you have to consider stuff like memory allocation and deallocation (to a certain degree), procedure calling conventions (call-by-value, call-by-reference, ...) and so on.

Despite that, it provides many features that are considered high-level, some of which are of great help when dealing with the low-level aspects. In particular, Rust has a very powerful **type system** that help enforce correctness of a lot of programming constructs.

So let's get into some of the details of what makes Rust such a remarkable language. I won't dedicate much time to Rust's syntax unless it's absolutely necessary, since there are already very good sources for learning that. The focus here is on semantics.

# Memory management overview

We know that operating systems generally divide a process's memory into regions that have different purposes. Here we are mostly interested in the **stack** and the **heap**, since they're the ones providing storage to a running process.

## The stack

To see how these regions relate to memory management, consider the following C code:

```c
#include <stdio.h>

struct Person {
  unsigned int age;
};

struct Person create_person() {
  struct Person new_person;
  new_person.age = 25;
  return new_person;
}

void do_stuff(struct Person person) {
  printf("%d", person.age);
}

int main() {
  struct Person p = create_person();
  do_stuff(p);
  // almost finished
  return 0;
}
```

The execution steps for this code go roughly as follows: starting at the `main` function, a variable `p` of type `struct Person` is declared, so that we have a corresponding allocation for its storage on `main`'s stack frame on the stack (obviously), with enough space to store said struct.

Then `create_person` is called to initialize `p`, which creates a stack frame on top of `main`'s frame. Inside `create_person`, we have yet another declaration for a `struct Person` variable, `new_person`, so we get some space for it on `create_person`'s stack frame and then write `25` to its `age` field. After that, execution returns `new_person`'s value to the caller, in this case `main`, and deallocate all space used by the variables on `create_person`'s frame and pop it from the stack.

By standard C semantics for assignments, argument passing and return values, `new_person`'s value is **copied** from `create_person`'s stack frame space to `main`'s frame, specifically to the space corresponding to `p`. For structs, this means copying each field, which can be really expensive.

Next, we call `do_stuff` and pass `p` as argument, so there's another copy of its value, now from `main`'s stack frame to the `person` parameter on `do_stuff`'s frame which was just created. We then print the person's age to the console and return to the `main` function, and once more all the memory allocated to `do_stuff` is freed and its stack frame popped.

Now we're back to the `main` function, and reach the `almost finished` comment. Notice that, at this point,  `p` still holds the (old) copy it got from the `create_person` function. We then return from the `main` function and the execution halts.

All of this stack handling process is performed automatically because the compiler has enough information to generate the appropriate instructions to do so. This is sometimes called **automatic memory** and is very easy to reason about: each variable **and** its storage live only throughout its declared scope, which is syntactically distinguishable, that is, you can see its span just by quickly inspecting the code. Assignments, arguments and return values just copy data from one location to another, so that destroying the ones which are not in scope anymore, doesn't leave any corrupt data or inconsistent state.

Let's call this copying behavior **Copy Semantics** (not standard terminology). As I mentioned before, this can be really computationally expensive, for example, if we have a huge struct that is passed around a lot.

## The heap

C programmers know that all this copying can be avoided by using pointers and manual memory allocation on the heap. For instance, we can modify the code from before to look like this:

```c
#include <stdio.h>
#include <stdlib.h>

struct Person {
  unsigned int age;
};

struct Person* create_person() {
  // malloc may fail by returning NULL, but let's ignore it for simplicity
  struct Person* new_person = malloc(sizeof(struct Person));
  new_person->age = 25;
  return new_person;
}

void do_stuff(struct Person* person) {
  printf("%d", person->age);
}

int main() {
  struct Person* p = create_person();
  do_stuff(p);
  // almost finished
  free(p);
  return 0;
}
```

The high-level behavior of this program is the same as the previous one, but now the functions deal with pointers to `struct Person`, this basically means that instead of copying a whole struct instance while returning from `create_person` or passing an argument to `do_stuff`, what is really copied is the pointer value, an address, which is represented by an integer in virtually all compilers and systems, so that's a really cheap operation.

It seems now that the problem with expensive copying has been overcome, but this approach has some disadvantages:

1. Memory management is now fully manual: the compiler can't help us keep track of the memory consumed by the program so for every explicit allocation request with `malloc` there must be an explicit deallocation request with `free`. Also if we reach the situation where there are no pointers to some memory we previously allocated then we have no way to free it anymore and we have caused what is widely known as a **memory leak**.

2. We have to make sure that all of our pointers actually contain valid addresses: for example, after the call to `free`, `p` is a **dangling pointer**, i.e., it points to invalid memory and using it from that point on could lead to **undefined behavior** or a [segmentation fault](https://en.wikipedia.org/wiki/Segmentation_fault).

To mitigate these issues, many languages, like Java, Python and Go, use some form of **garbage collection**. They provide enriched runtime environments which include a **garbage collector**, that runs alongside our code in order to detect which regions of the heap contain objects *not* referenced by any variable so that it can proceed to free this memory automatically.

Garbage collectors are a great solution to memory management problems provided that you can afford such a runtime environment. For example, the reference implementation of the Java virtual machine uses a **stop-the-world** garbage collector, which basically means that it stops the running program to perform the garbage collection and this wait may be unacceptable for certain applications. Also, these runtime environments may be too resource-consuming or otherwise inappropriate for some types of embedded or real-time systems.

# The Rust(y) way

Rust provides no runtime environment apart from it's standard library which puts it closer to how C and C++ work. To address these issues without such an environment, Rust uses two powerful concepts that were introduced by the C++ community: **Move Semantics** and **Scope-Based Resource Management**. One of Rust's novelties comes from backing up these two with its type system. Let's look at some examples.

## Move Semantics

Before we begin, let's talk about the concept of ownership in Rust: all objects have one and only one owner. When we write something like

```rust
struct Person {
  age: u32
}

fn main() {
  let a = Person { age: 36 };
  let b = a;
  println!("{}", a.age);
}
```

we are constructing a `Person` object and assigning it to `a`, this means that `a` is the owner of said object. However when we assign `a` to `b`, we are moving the ownership of that same object to variable `b`. Because of that, we cannot access it from `a` anymore, so that `println!` line is actually invalid. Indeed, if we try compiling this code we get the following error:

```
error[E0382]: borrow of moved value: `a`
 --> src/main.rs:8:20
 
... snip ...

7 |     let b = a;
  |             - value moved here
8 |     println!("{}", a.age);
  |                    ^^^^^ value borrowed here after move

error: aborting due to previous error
```

The compiler is basically telling us that we are *borrowing* after we have moved the value from `a`. We'll see what borrowing means another time, but for now think of it as a way to access an object.

Now let's apply this to our original context. We can translate our first piece of code from C to Rust in a really straightforward manner:

```rust
struct Person {
  age: u32
}

fn create_person() -> Person {
  return Person { age: 25 }; // "return" and ";" are superfluous here
}

fn do_stuff(person: Person) {
  println!("{}", person.age);
}

fn main() {
  let p = create_person();
  do_stuff(p);
}
```

Again, this program has the same overall behavior the original in C, but let's take a closer look. Instead of copying arguments, Rust defaults to **moving** their ownership, and the same applies to assignments and return values. So the steps here are roughly as follows: we call `create_person` which constructs a `Person` object, so it has an implicit owner that is bound to `create_person`'s scope (and stack frame). But it's immediately returned, so its ownership is moved to the caller, `main`, which moves said ownership to variable `p`.

Now, there's a call to `do_stuff` and we're passing `p` as an argument, thus moving the ownership from `p` to the `person` parameter on `do_stuff`. There we access it's age field and print it.

Some points are worth noting:

1. Given that the ownership of the object originally in `p` was moved from it after the call to `do_stuff`, we can no longer access it through `p` on `main` and this is guaranteed by the compiler. **This is Move Semantics in action!**

2. Since this same object is now owned solely by a parameter that is on `do_stuff`'s scope, when execution reaches the end of that scope, Rust compiler knows that it can free it's memory, since no one else can use it. **This is Scope-Based Resource Management in action!**

3. The same stack pushing and popping we described on the C version applies here, and even though the ownership moves around, the values itself may need copying because they may be living in some stack frame that is getting popped.

Point 1 may not seem like a big deal with such a simple example, but besides it's computational properties, being enforced by the compiler provides very interesting mathematical properties about programs and these properties can be used by the compiler to perform optimizations and "prove" the correctness of the code, specially regarding general resource usage. For the mathematically inclined, Rust compiler actually implements what's called an **affine type system** which is a kind of **affine logic**.

Scope-Based Resource Management is traditionally called **Resource acquisition is initialization (RAII)**, which doesn't really convey much about the concept, in my opinion.

We still have a small problem though: point 3 indicates that there's some copying involved and this bears some of the issues we originally had in C. But that's because we haven't used the heap in Rust yet and luckily we can make use of it with a pretty simple change to our current code:

```rust
struct Person {
  age: u32
}

fn create_person() -> Person {
  return Person { age: 25 }; // "return" and ";" are superfluous here
}

fn do_stuff(person: Box<Person>) {
  println!("{}", person.age);
}

fn main() {
  let p = Box::new(create_person());
  do_stuff(p);
}
```

Boxes are the simplest way to allocate objects on the heap in Rust. Here we just used its `new` function to create a `Box<Person>` instance with the `Person` object returned by `create_person`, which at this point is on the heap and owned by said box (the box itself is on the stack since it's owned by the variable `p`). We also changed `do_stuff`'s signature to take a `Box<Person>` parameter.

When we call `do_stuff` passing the box `p` as an argument, its ownership is moved to the `person` parameter, but it still points to the same `Person` object on the heap, so there's no copying there. The call may cause a copy of the box itself, but it's a very lightweight object, analogous to copying a pointer in C in our second code listing.

The advantage here is that point 2 from above still applies to this case: when execution reaches the end of `do_stuff`, the compiler knows that `person` is the unique owner of the box, so it can deallocate it's memory since no one's using it anymore. However, before deallocating the box, there's an implicit call to the box's `drop` method (which works as a destructor in Rust) and in the case of the `Box` type, `drop` just deallocates the memory it points to on the heap. And there it is! We finally solved our problems.

This call to `drop` is generated by the compiler for any type that implements the `Drop` **trait** (think improved interface) as a way to enforce Scope-Based Resource Management. In particular, one could say that Rust implements a kind of **compile-time garbage collection**.

But it's much more than that! Despite our example, this process does not apply only to memory, but also to any kind of resource that requires some sort of acquisition and release, like files, sockets, database connections, mutexes, etc. Each of these has a constructor that knows how to allocate the respective resource, and also implements `drop` to specify how to deallocate that same resource. Therefore, whenever you need one of them, just call its constructor, then pass the object around to do some work and let the compiler figure out when to drop it. You can even implement your own resources by following the same *protocol*.

That's it for this post. I intend to write on other Rust features very soon.

# References and further reading

- <https://doc.rust-lang.org/book/>

- <https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)>

- <https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization>

- <https://stackoverflow.com/questions/2321511/what-is-meant-by-resource-acquisition-is-initialization-raii>

- <https://stackoverflow.com/questions/16695874/why-does-the-jvm-full-gc-need-to-stop-the-world>
