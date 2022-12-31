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

因为 Quiz 是乱序的，所以完成进度也是乱序的

## Quiz #5: 

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



