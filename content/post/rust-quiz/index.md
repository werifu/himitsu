---
title: Rust Quiz 记录
description: 做题记录 & 扩展笔记
toc: true
authors: 
  - werifu
tags: 
  - Rust
categories: []
series: []
date: 2022-10-19T23:30:23+08:00
lastmod: 2022-10-19T23:30:23+08:00
featuredVideo:
featuredImage: 
draft: false
---
https://dtolnay.github.io/rust-quiz

## Quiz #1: 

## Quiz #2: () and {}

```rust
struct S(i32);

impl std::ops::BitAnd<S> for () {
    type Output = ();

    fn bitand(self, rhs: S) {
        print!("{}", rhs.0);
    }
}

fn main() {
    let f = || ( () & S(1) );
    let g = || { () & S(2) };
    let h = || ( {} & S(3) );
    let i = || { {} & S(4) };
    f();
    g();
    h();
    i();
}
```

### Answer

Output: 123

The closures f, g, and h are all of type impl Fn(). The closure bodies are parsed as an invocation of the user-defined bitwise-AND operator defined above by the BitAnd trait impl. When the closures are invoked, the bitwise-AND implementation prints the content of the S from the right-hand side and evaluates to ().

The closure i is different. Formatting the code with rustfmt makes it clearer how i is parsed.
```rust
let i = || {
    {}
    &S(4)
};
```
The closure body consists of an empty block-statement `{}` followed by a reference to `S(4)`, not a bitwise-AND. The type of `i` is `impl Fn() -> &'static S`.

The parsing of this case is governed by [this code](https://github.com/rust-lang/rust/blob/1.30.1/src/libsyntax/parse/classify.rs#L17-L37) in libsyntax.

## Quiz #3: semantics of const

```rust
struct S {
    x: i32,
}

const S: S = S { x: 2 };

fn main() {
    let v = &mut S;
    v.x += 1;
    S.x += 1;
    print!("{}{}", v.x, S.x);
}
```

### Answer
The semantics of const is that any mention of the const by name in expression position is substituted with the value of the const initializer. In this quiz code the behavior is equivalent to:


```rust
struct S {
    x: i32,
}

fn main() {
    let mut _tmp0 = S { x: 2 };
    let v = &mut _tmp0;
    v.x += 1;
    S { x: 2 }.x += 1;
    print!("{}{}", v.x, S { x: 2 }.x);
}
```



## Quiz #5

https://dtolnay.github.io/rust-quiz/5

What is the output of this Rust program?

```rust
trait Trait {
    fn p(self);
}

impl<T> Trait for fn(T) {
    fn p(self) {
        print!("1");
    }
}

impl<T> Trait for fn(&T) {
    fn p(self) {
        print!("2");
    }
}

fn f(_: u8) {}
fn g(_: &u8) {}

fn main() {
    let a: fn(_) = f;
    let b: fn(_) = g;
    let c: fn(&_) = g;
    a.p();
    b.p();
    c.p();
}
```

### 解答

输出 `112`

其实难点就是 b 到底属于哪种类型，需要把第二个实现展开，`impl<'a T> Trait for fn(&'a T)`，b 我们会推断 `_ = &'x u8`，b 的类型应该是 `fn(&'x u8)`，签名按照 fn(T) 来。



## Quiz #10: 方法的覆盖

https://dtolnay.github.io/rust-quiz/10

What is the output of this Rust program?
```rust
trait Trait {
    fn f(&self);
}

impl<'a> dyn Trait + 'a {
    fn f(&self) {
        print!("1");
    }
}

impl Trait for bool {
    fn f(&self) {
        print!("2");
    }
}

fn main() {
    Trait::f(&true);
    Trait::f(&true as &dyn Trait);
    <_ as Trait>::f(&true);
    <_ as Trait>::f(&true as &dyn Trait);
    <bool as Trait>::f(&true);
    <dyn Trait as Trait>::f(&true as &dyn Trait);
}
```

### 解答
输出 `222222`

为 bool 实现的方法会覆盖（shadow）掉内在的方法（inherent method），也就是 dyn Trait 那个。

而现在 Rust 还没有方法去调用那个 dyn Trait，如果按下面的方式调用会报错

```rust
error[E0034]: multiple applicable items in scope
  --> questions/010.rs:18:5
   |
18 |     <dyn Trait>::f(&true);
   |     ^^^^^^^^^^^^^^ multiple `f` found
   |
note: candidate #1 is defined in an impl for the type `dyn Trait`
  --> questions/010.rs:6:5
   |
6  |     fn f(&self) {
   |     ^^^^^^^^^^^
note: candidate #2 is defined in the trait `Trait`
  --> questions/010.rs:2:5
   |
2  |     fn f(&self);
   |     ^^^^^^^^^^^^
   = help: to disambiguate the method call, write `Trait::f(...)` instead
```

## Quiz #11: static lifetime function

```rust
fn f<'a>() {}
fn g<'a: 'a>() {}

fn main() {
    let pf = f::<'static> as fn();
    let pg = g::<'static> as fn();
    print!("{}", (pf == pg) as u8);
}
```

### Answer

Cannot compile.

Reason too complicated so I just copy the content: 
https://dtolnay.github.io/rust-quiz/11

Function pointer comparison is generally a Bad Idea. It is easily possible to get nonsensical behavior in optimized builds. For a jaw-dropping example of such behavior, check out [rust-lang/rust#54685](https://github.com/rust-lang/rust/issues/54685) in which `x == y` is both true and not true at the same time.

That said, the quiz code in this question fails to compile. Here is the compiler output:

error: cannot specify lifetime arguments explicitly if late bound lifetime parameters are present
```
 --> questions/011.rs:5:18
  |
5 |     let pf = f::<'static> as fn();
  |                  ^^^^^^^
  |
```
note: the late bound lifetime parameter is introduced here
```
 --> questions/011.rs:1:18
  |
1 | fn f<'a>() {}
  |      ^^
```
Generic parameters can be either early bound or late bound. Currently (and for the foreseeable future) type parameters are always early bound, but lifetime parameters can be either early or late bound.

Early bound parameters are determined by the compiler during monomorphization. Since type parameters are always early bound, you cannot have a value whose type has an unresolved type parameter. For example:
```rust
fn m<T>() {}

fn main() {
    let m1 = m::<u8>; // ok
    let m2 = m; // error: cannot infer type for `T`
}
However, this is often allowed for lifetime parameters:

fn m<'a>(_: &'a ()) {}

fn main() {
    let m1 = m; // ok even though 'a isn't provided
}
```
Since the actual choice of lifetime 'a depends on how it is called, we are allowed to omit the lifetime parameter and it will be determined at the call site. The lifetime can even be different for each time it gets called.

For this reason, we cannot specify the lifetime on this function until it is called:
```rust
// error: cannot specify lifetime arguments explicitly if late bound lifetime parameters are present
let m2 = m::<'static>;
```
We may not even ask the borrow checker to infer it too soon:
```rust
// error: cannot specify lifetime arguments explicitly if late bound lifetime parameters are present
let m3 = m::<'_>;
```
The idea of late bound parameters overlaps considerably with a feature of Rust called "higher ranked trait bounds" (HRTB). This is a mechanism for expressing that bounds on a trait's parameters are late bound. Currently this is limited to lifetime parameters, but the same idea exists in other languages (such as Haskell) for type parameters, which is where the term "higher ranked" comes from.

The syntax to express a HRTB for lifetimes uses the for keyword. To express the type of m1 above, we could have written:

```
let m1: impl for<'r> Fn(&'r ()) = m;
```
You can think of this as meaning: "There is a lifetime but we don't need to know what it is just yet".

Late bound lifetimes are always unbounded; there is no syntax for expressing a late bound lifetime that must outlive some other lifetime.
```
error: lifetime bounds cannot be used in this context
 --> src/main.rs:5:20
  |
5 |     let _: for<'b: 'a> fn(&'b ());
  |                    ^^
```
Lifetimes on data types are always early bound except when the developer has explicitly used the HRTB for syntax. On functions, lifetimes are late bound by default but can be early bound if:

The lifetime is declared outside the function signature, e.g. in an associated method of a struct it could be from the struct itself; or

The lifetime parameter is bounded below by some other lifetime that it must outlive. As we've seen, this constraint is not expressible in the HRTB that would be involved in late binding the lifetime.

By these rules, the signature fn f<'a>() has a late bound lifetime parameter while the signature `fn g<'a: 'a>()` has an early bound lifetime parameter — even though the constraint here is ineffectual.

Ordinarily these distinctions are compiler-internal terminology that Rust programmers are not intended to know about or think about in everyday code. There are only a few edge cases where this aspect of the type system becomes observable in the surface language, such as in the original quiz code.

## Quiz #12: deconstruct and drop

```rust
struct D(u8);

impl Drop for D {
    fn drop(&mut self) {
        print!("{}", self.0);
    }
}

struct S {
    d: D,
    x: u8,
}

fn main() {
    let S { x, .. } = S {
        d: D(1),
        x: 2,
    };
    print!("{}", x);

    let S { ref x, .. } = S {
        d: D(3),
        x: 4,
    };
    print!("{}", x);
}
```

### Answer

Output: 1243

Core question: Where does `D` get dropped?

The first S is dropped immediately since there is no any owner.

The second S remains in scope since its field is borrowed and is dropped after the scope of main.

## Quiz #16: --x

```rust
fn main() {
    let mut x = 4;
    --x;
    print!("{}{}", --x, --x);
}
```

### Answer

Output: 44

There is no unary increment or decrement operator in Rust.

> Why doesn't Rust have increment and decrement operators?
> 
> Preincrement and postincrement (and the decrement equivalents), while convenient, are also fairly complex. They require knowledge of evaluation order, and often lead to subtle bugs and undefined behavior in C and C++. x = x + 1 or x += 1 is only slightly longer, but unambiguous.

`--x` is parsed as `-(-x)`, so it's just an expression statement.



## Quiz #19: drop

https://dtolnay.github.io/rust-quiz/19

What is the output of this Rust program?

```rust
struct S;

impl Drop for S {
    fn drop(&mut self) {
        print!("1");
    }
}

fn main() {
    let s = S;
    let _ = s;
    print!("2");
}
```

### 解答

输出 `21`

比较简单，两个 let 只有一个 S 被创建了，第二个只是转移了所有权，在生命周期结束时调用 drop。





## Quiz #22: 宏的参数

https://dtolnay.github.io/rust-quiz/22

What is the output of this Rust program?

```rust
macro_rules! m {
    ($a:tt) => { print!("1") };
    ($a:tt $b:tt) => { print!("2") };
    ($a:tt $b:tt $c:tt) => { print!("3") };
    ($a:tt $b:tt $c:tt $d:tt) => { print!("4") };
    ($a:tt $b:tt $c:tt $d:tt $e:tt) => { print!("5") };
    ($a:tt $b:tt $c:tt $d:tt $e:tt $f:tt) => { print!("6") };
    ($a:tt $b:tt $c:tt $d:tt $e:tt $f:tt $g:tt) => { print!("7") };
}

fn main() {
    m!(-1);
    m!(-1.);
    m!(-1.0);
    m!(-1.0e1);
    m!(-1.0e-1);
}
```

### 解答

输出 `22222`

tt 表示 token，这个宏的功能是按传入的 token 数分类。

Rust 的编译器会把 `-` 单独看成一个负号，五个调用都是一个负号加一个数字。

`let x = -2.pow(2)` 会被解析成 `-(2.pow(2))` 而不是 `(-2).pow(2)`






## Quiz #24: 关于宏的 'hygiene'

https://dtolnay.github.io/rust-quiz/24

What is the output of this Rust program?
```rust
fn main() {
    let x: u8 = 1;
    const K: u8 = 2;

    macro_rules! m {
        () => {
            print!("{}{}", x, K);
        };
    }

    {
        let x: u8 = 3;
        const K: u8 = 4;

        m!();
    }
}
```

### 解答

输出 `14`

对于宏而言有一个概念叫 hygiene，中文是【卫生】的意思，由【宏如何处理外部变量】来区分是否 hygiene

[有一个 reddit 上对此的讨论](https://www.reddit.com/r/rust/comments/5v8r8f/so_what_are_hygienic_macros_anyway/)，摘抄一个回答：

> For example, if you declare a variable named x inside a macro and you happen to call that macro on an x from somewhere else, it won't suddenly and magically cause things to break because the compiler will know that they're two different things.
> 
> (The gist is that macros in languages like C have some very surprising misbehaviours and "hygienic" macros will behave more like functions when it comes to things like variable scopes and order of operations.)

如果编译器不处理出现在宏里的变量名，而是等着直接展开（如 C 语言），那么这个应该算作不卫生，因为可能会出现外部变量命名为 a，而宏内使用了 a 变量，使用结果会因为外部变量命名不同而有变化。

Rust 是门“部分卫生”的语言，会对一部分的外部变量进行处理。但是仅限本地变量，对 const 不会做处理（const 变量会被认为是个普通的单词而不是变量）。因此在这题里，Rust 会先把外边的 x 编进宏内，之后再进行展开，最终打印 `14`。

[hygiene 相关的笔记](https://danielkeep.github.io/tlborm/book/mbe-min-hygiene.html)

## Quiz #25: name resolution

```rust
use std::fmt::{self, Display};

struct S;

impl Display for S {
    fn fmt(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("1")
    }
}

impl Drop for S {
    fn drop(&mut self) {
        print!("2");
    }
}

fn f() -> S {
    S
}

fn main() {
    let S = f();    // <- this is destructuring instead of assignment
    print!("{}", S);
}

```

### Answer

Output: 212

On the first line of main, we call f() and perform an infallible match that binds no new variables. As no variables are declared on this line, there is no variable that could be the owner of the S returned by f() so that S is dropped at that point, printing 2. The S in let S = f() is a unit struct pattern (not a variable name) that matches a value of type S via destructuring but does not bind the value to any variable.

The second line of main conjures a new S, prints it, and drops it at the semicolon.

## Quiz #26: Lazy iter::map

```rust
fn main() {
    let input = vec![1, 2, 3];

    let parity = input
        .iter()
        .map(|x| {
            print!("{}", x);
            x % 2
        });

    for p in parity {
        print!("{}", p);
    }
}
```

### Answer

Output: 112031

`Iterator::map` is executed **lazily**.


## Quiz #28: empty struct drop

```rust
struct Guard;

impl Drop for Guard {
    fn drop(&mut self) {
        print!("1");
    }
}

fn main() {
    let _guard = Guard;
    print!("3");
    let _ = Guard;
    print!("2");
}
```

### Answer

Output: 3121

the Drop impl for `let _guard = Guard` runs at the end of main but the `Drop` impl for `let _ = Guard` runs right away.

In general, a value is dropped when it no longer has an owner.

### Something beyond itself
This distinction between the underscore pattern vs variables with a leading underscore is incredibly important to remember when working with lock guards in unsafe code.
```rust
use std::sync::Mutex;

static MUTEX: Mutex<()> = Mutex::new(());

/// MUTEX must be held when accessing this value.
static mut VALUE: usize = 0;

fn main() {
    let _guard = MUTEX.lock().unwrap();
    unsafe {
        VALUE += 1;
    }
}
```
If this code were to use `let _ = MUTEX.lock().unwrap()` then the mutex guard would be dropped immediately, releasing the mutex and failing to guard the access of `VALUE`.


## Quiz #30: zero-sized types for () and empty struct

```rust
use std::rc::Rc;

struct A;

fn p<X>(x: X) {
    match std::mem::size_of::<X>() {
        0 => print!("0"),
        _ => print!("1"),
    }
}

fn main() {
    let a = &A;
    p(a);
    p(a.clone());
    
    let b = &();
    p(b);
    p(b.clone());
    
    let c = Rc::new(());
    p(Rc::clone(&c));
    p(c.clone());
}
```

### Answer

Output: 111011

The function `p<X>` will print 0 if it is passed a value of type `X = ()` or `X = A`, and it will print `1` if passed a reference `X = &()` or `X = &A` regardless of exactly how big pointers happen to be.

If A implemented `Clone` then `a.clone()` would be a call to that impl. But since it doesn't, the compiler finds another applicable impl which is the implementation of `Clone` for references `&T` -- so concretely the clone call is calling the impl of Clone for `&A` which turns a `&&A` into a `&A` by simply duplicating the reference. We get another call to `p` with `X = &A` printing 1. The impl of Clone for references is useful in practice when a struct containing a reference wants to derive Clone, but as seen here it can sometimes kick in unexpectedly.

The type `()` does implement `Clone` so `b.clone()` invokes that impl and produces `()`. The implementation of `Clone` for `&()` would also be applicable as happened in the case of `A`, but the compiler prefers calling the trait impl for `()` which converts `&()` to `()` over the trait impl for `&()` which converts `&&()` to `&()` because the former is the one that requires fewer implicit references or dereferences inserted by the trait solver. In the call to `b.clone()`, b is of type `&()` which exactly matches the argument of the impl `Clone` for `()`, while in order to obtain a `&&()` to pass as argument to the impl `Clone` for `&()` the trait solver would need to insert an additional layer of referencing implicitly -- effectively computing `(&b).clone()`.

## Quiz #33: RangeFull

https://dtolnay.github.io/rust-quiz/33

What is the output of this Rust program?

```rust
use std::ops::RangeFull;

trait Trait {
    fn method(&self) -> fn();
}

impl Trait for RangeFull {
    fn method(&self) -> fn() {
        print!("1");
        || print!("3")
    }
}

impl<F: FnOnce() -> T, T> Trait for F {
    fn method(&self) -> fn() {
        print!("2");
        || print!("4")
    }
}

fn main() {
    (|| .. .method())();
}
```

### 解答

输出`24`

RangeFull 其实就是 `..`，是可以单独使用的，如下：

```rust
let arr = [0, 1, 2, 3, 4];
assert_eq!(arr[ ..  ], [0, 1, 2, 3, 4]); // This is the `RangeFull`
assert_eq!(arr[ .. 3], [0, 1, 2      ]);
assert_eq!(arr[ ..=3], [0, 1, 2, 3   ]);
assert_eq!(arr[1..  ], [   1, 2, 3, 4]);
assert_eq!(arr[1.. 3], [   1, 2      ]);
assert_eq!(arr[1..=3], [   1, 2, 3   ]);
```

但不能直接用于循环 `for i in ..`

这题只可能两种答案，`1` 或者 `24`，解析成 `|| ((..).method())`  就是 `1`，解析成 `(|| ..).method()` 就是 `24`

## Quiz #34: zero-sized type

```rust
fn d<T>(_f: T) {
    match std::mem::size_of::<T>() {
        0 => print!("0"),
        1 => print!("1"),
        _ => print!("2"),
    }
}

fn a<T>(f: fn(T)) {
    d(f);
}

fn main() {
    a(a::<u8>);
    d(a::<u8>);
}
```

### Answer

Output: 20

`a::<u8>`'s type is zero-sized.

In Rust, every distinct instantiation of a generic function has its own unique type. (two function with the same signature would have different types)

`iter::map` is one example: `iter::map(f)` and `iter::map(g)` would call `f(element)` and `g(element)` in compiled binary file instead of pass a function pointer as the argument like C or Go.

Currently in Rust there is no syntax to express the type of a specific function, so they are always passed as a generic type parameter with a FnOnce, Fn or FnMut bound. In error messages you might see function types appear in the form `fn(T) -> U {fn_name}`, but you can't use this syntax in code.

On the other hand, a function pointer, `fn(T) -> U`, is pointer-sized at runtime. Function types can be coerced into function pointers, which can be useful in case you need to defer the choice of function to call until runtime.

In the quiz code, the first call in main coerces `a::<u8>` from a function to a function pointer (`fn(fn(u8)) {a::<u8>}` to `fn(fn(u8))`) prior to calling d, so its size would be 8 on a system with 64-bit function pointers. The second call in main does not involve function pointers; d is directly called with T being the inexpressible type of `a::<u8>`, which is zero-sized.

As for `Rc`, it's obvious.



