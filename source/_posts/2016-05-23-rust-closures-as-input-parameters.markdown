---
layout: post
title: "Rust closures as input parameters"
date: 2016-05-23 21:47:00 +0200
comments: true
categories: 
---

_Edit:_ [_Update (May 25)_](#update-2016-05-25)

I am learning [Rust](https://www.rust-lang.org/) and, as a beginner, I have sometimes problems achieving some little tasks that would be so easy in other programming languages I know better.

But when I met some Rust developers and they ask me about my difficulties, I often forget about them.

I therefore decided to write about my difficulties in Rust to keep track of them.

So today about closures.

When reading a [blog post introducing Rust for Node.js Developers](http://fredrik.anderzon.se/2016/05/10/rust-for-node-developers-part-1-introduction/) I made the same Todo application.

At the end, the program contains such piece of code:

```rust
fn remove_todo(todos: &mut Vec<Todo>, todo_id: i16) {
	if let Some(todo) = todos.iter_mut().find(|todo| todo.id == todo_id) {
		todo.deleted = true;
	}
}

fn mark_done(todos: &mut Vec<Todo>, todo_id: i16) {
	if let Some(todo) = todos.iter_mut().find(|todo| todo.id == todo_id) {
		todo.completed = true;
	}
}
```

The first function marks a Todo as deleted if it can be found by its ID in the vector.
The second function marks a Todo as completed, also if it can be found in the vector of Todos.

Some code is duplicated and I decided to refactor the common code in a third function, that would do _something_ on a Todo if found in a vector.

This third function would take a closure as input parameter, like in pseudo-code:
```rust
fn with_todo_id(todos: &mut Vec<Todo>, todo_id: i16, f: <closure - do something on a Todo>) {
    if let Some(todo) = todos.iter_mut().find(|todo| todo.id == todo_id) {
        f(todo);
    }
}
```

so that the 2 initial functions are simplified like that:

```rust
fn remove_todo(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, |todo| todo.deleted = true);
}

fn mark_done(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, |todo| todo.completed = true);
}
```

This closure is a side-effect on a Todo. It should accept a mutable Todo as parameter and return nothing.

One [source of documentation for closures as input parameters](http://rustbyexample.com/fn/closures/input_parameters.html) mentions that there exist 3 kinds of closures:

- Fn: takes captures by reference (`&T`)
- FnMut: takes captures by mutable reference (`&mut T`)
- FnOnce: takes captures by value (`T`)

This is a lot of information for a new developer.

I tried different possibilities, like:

```rust
fn with_todo_id(todos: &mut Vec<Todo>, todo_id: i16, f: &Fn(&mut Todo)) {
    if let Some(todo) = todos.iter_mut().find(|todo| todo.id == todo_id) {
        f(todo);
    }
}

fn remove_todo(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, |todo| todo.deleted = true);
}

fn mark_done(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, |todo| todo.completed = true);
}
```

Without any success:
```bash
$ cargo run
   Compiling todo-list v0.1.0 (file:///Users/yannsimon/projects/rust/rust-playground/todo-list)
src/main.rs:27:38: 27:50 error: the type of this value must be known in this context
src/main.rs:27 	with_todo_id(todos, todo_id, |todo| todo.deleted = true);
               	                                    ^~~~~~~~~~~~
```

The [official documentation](https://doc.rust-lang.org/book/closures.html#taking-closures-as-arguments) was not so much help neither.

I asked for help on [#rust-beginners](https://client00.chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust-beginners).
People on this channel are **very** helpful and kind. The community of Rust is awesome!

I was proposed 2 solutions. Both work, and I choose that one:
```rust
fn with_todo_id<P>(todos: &mut Vec<Todo>, todo_id: i16, f: P) where P: Fn(&mut Todo) {
    if let Some(todo) = todos.iter_mut().find(|todo| todo.id == todo_id) {
        f(todo);
    }
}

fn remove_todo(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, |todo| todo.deleted = true);
}

fn mark_done(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, |todo| todo.completed = true);
}
```

Compared to my previous attempt, the `f: &Fn(&mut Todo)` is replaced by `f: P where P: Fn(&mut Todo)`.

I still do not completely understand why this works and not the previous version. I was explained Rust can use the reference to the closure... I will continue reading documentation about it.... ;)

If you have any good source for this, please [tell me](https://twitter.com/simon_yann).

In conclusion I still find closure as input parameters quite complex in Rust. I surely need to more understand the theory behind the language to fully understand them.

The Rust community is very helpful, but it may not scale if there are more and more beginners like me.



#### <a name="update-2016-05-25"></a> [_Update (May 25)_](#update-2016-05-25)

The following [tweet](https://twitter.com/rustlang/status/734946700536774656) from [@rustlang](https://twitter.com/rustlang) provided me the good keywords to search for:

{% blockquote @rustlang, https://twitter.com/rustlang/status/734946700536774656 %}
it's about trait objects vs type parameters, which can be tough when you're learning
{% endblockquote %}

[Trait objects](https://doc.rust-lang.org/book/trait-objects.html) are used for [dynamic dispatch](https://en.wikipedia.org/wiki/Dynamic_dispatch), feature found in most OO languages.

With that in mind, I could understand the [Rust book about closures](https://doc.rust-lang.org/book/closures.html#taking-closures-as-arguments).

If I use trait objects, this version works:
```
fn with_todo_id(todos: &mut Vec<Todo>, todo_id: i16, f: &Fn(&mut Todo)) {
    if let Some(todo) = todos.iter_mut().find(|todo| todo.id == todo_id) {
        f(todo);
    }
}

fn remove_todo(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, &|todo| todo.deleted = true);
}

fn mark_done(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, &|todo| todo.completed = true);
}
```

Trait objects force Rust to use dynamic dispatch. For that, Rust must allocate the closure in the heap.

If I use type parameter instead of a trait object:
```
fn with_todo_id<P>(todos: &mut Vec<Todo>, todo_id: i16, f: P) where P: Fn(&mut Todo) {
    if let Some(todo) = todos.iter_mut().find(|todo| todo.id == todo_id) {
        f(todo);
    }
}

fn remove_todo(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, |todo| todo.deleted = true);
}

fn mark_done(todos: &mut Vec<Todo>, todo_id: i16) {
    with_todo_id(todos, todo_id, |todo| todo.completed = true);
}
```

then Rust is able to monomorphize the closure and use static dispatch, without any heap allocation.

Another great example of the [zero-cost abstraction](http://blog.rust-lang.org/2015/05/11/traits.html) possible with Rust!
