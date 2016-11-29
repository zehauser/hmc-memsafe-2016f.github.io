layout: true
class: center middle

---

## Functions & Closures in Rust: 
### fn, Fn, FnMut, FnOnce

### Zach Hauser

29 November 2016

---

layout: false
class: left

## First-Class Functions

In a modern language like Rust, we need first class functions.

---

## First-Class Functions

In a modern language like Rust, we need first class functions.

We're a long way away from Java 7:

```
public class List<E> {
    public void retain(Predicate<E> p) { ... }
    ...
}

public interface Predicate<E> {
    public boolean test(E e);
}
```

---

## First-Class Functions

In Rust, we can express this with first-class functions:

```rust
fn is_positive(x: &isize) -> bool {
    *x > 0 
}

fn main() {
    let mut v = vec![-1,0,1,2];
    println!("{:?}", v); // [-1, 0, 1, 2]
    v.retain(is_positive);
    println!("{:?}", v); // [1, 2]
}
```

---

## First-Class Functions

Toy example:

```rust
fn is_positive(x: isize) -> bool {
    x > 0
}

fn test(pred: fn(isize) -> bool, x: isize) -> bool {
    pred(x)
}

fn main() {
    println!("{}", test(is_positive, 3)); // true
}
``` 

---

## Anonymous Functions

That's pretty verbose, let's use an anonymous function:

```rust
fn test(pred: fn(isize) -> bool, x: isize) -> bool {
    pred(x)
}

fn main() {
    println!("{}", test(|x| x > 0, 3)); // doesn't compile
}
``` 

---

## Closures

Toy example to illustrate a closure's environment:

```rust
fn test(pred: fn(isize) -> bool, x: isize) -> bool {
    pred(x)
}

fn main() {
    let y = 3;
    let z = 4;
    println!("{}", test(|x| x > y + z, 3)); 
}
```

---

## Closures

What's the potential issue? The ownership system, of course.

```rust
fn test(pred: fn(isize) -> bool, x: isize) -> bool {
    pred(x)
}

fn main() {
    let verifier = get_verifier();
    println!("{}", test(|x| verifier.verify(x), 3)); 
}
```

---

## Closures

What is `verifier.verify`? Does it take `Self`, `&Self`, or `&mut Self`?

```rust
fn test(pred: fn(isize) -> bool, x: isize) -> bool {
    pred(x)
}

fn main() {
    let verifier = get_verifier();
    println!("{}", test(|x| verifier.verify(x), 3)); 
    verifier.do_something(); // will this compile?
}
```

---

## Closures 

Another potential issue:

```rust
fn main() {
    let tester = |x| verifier.verify(x);
    println!("{}", tester, 3)); 
    println!("{}", tester, 5)); // can we even call `tester` twice?
}
```

---

## Function Traits

Rust has operator overloading via special traits in `std::ops`. For example, you can overload `+` by implementing `Add` and its required method `add`. 

This model extends to function/closure types, via the function traits. If you implement `Fn` and its required method `call`, then you can use the call operator `()`.

---

## Function Traits

Closures are desugared to structs that implement function traits. We can fix our example from the beginning:

```rust
fn test<F>(pred: F, x: isize) -> bool
where F: Fn(isize) -> bool {
    pred(x)
}

fn main() {
    println!("{}", test(|x| x > 0, 3)); // true
}
``` 

---

## Function Traits

There are actually _three_ function traits, which vary in how their call operator captures their environment.

```rust
pub trait FnOnce<Args> {
    type Output;
    fn call_once(self, args: Args) -> Self::Output;
}

pub trait FnMut<Args>: FnOnce<Args> {
    fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait Fn<Args>: FnMut<Args> {
    fn call(&self, args: Args) -> Self::Output;
}
```

Rust infers the appropriate trait for your closure.

---

## The Move Keyword

```rust
fn make_adder(x: isize) -> Box<Fn(isize) -> isize> {
    let closure = |y| x + y;
    Box::new(closure)
}


fn main() {
    println!("{}", make_adder(4)(5)); // 9
}
```

This doesn't quite work.

---

## Desugaring Closures?

A simple example:

```rust
fn main() {
    let adder = |x| x + 3;
    println!("{}", adder(5));
}
```

Let's desugar the closure syntax. 

---

## Desugaring Closures?

```rust
#![feature(fn_traits, unboxed_closures)]

struct Adder {
    x: isize
}

impl FnOnce<(isize,)> for Adder {
    type Output = isize;
    extern "rust-call" fn call_once(self, y: (isize,)) -> isize {
        y.0 + self.x
    }
}

fn main() {
    let adder = Adder { x: 3 };
    println!("{}", adder(5));
}
```