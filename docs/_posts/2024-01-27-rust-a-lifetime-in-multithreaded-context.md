---
layout: post
title:  "Rust: A lifetime in multithreaded context"
date:   2024-01-31 16:05:43 +0300
categories: rust
---


#### Introduction

In one project ([dom_finder](https://github.com/niklak/dom_finder)), I had a structure called `Config` and another structure called `Finder`. `Finder` was fully dependent on the lifetime of Config because its fields stored references to `Config`'s field values. This arrangement worked fine for me until I realized that this library would be used in a multi-threaded environment, with different scenarios: *Owned* and *Borrowed*. When `thread::scope` is used, everything works just fine. However, in a normal scenario where `Arc` or *cloning* is required for accessing an object, a problem arose with the lifetime of `Config` - the compiler refused to build the program because it required a static lifetime for `Config`. As an option, a solution with `once_cell` (`Lazy`) emerged - making `Config` with a *static lifetime*. However, this solution did not seem acceptable. In this approach, the user had to take care of extending the *lifetime* of Config themselves. That's when I had to think deeper and search for a solution.

My `Config` structure was simple, it didn't contain any references to other objects. However, the fields of `Finder` referenced some string fields of `Config` (the rest of the fields were copies, as referencing them made no sense), and `Config` also contained a pipeline field - `Vec<Vec<String>>`.
The simplest solution was to replace `&str` with `String`. This was an acceptable option, but then potential optimizations could be forgotten.


To focus solely on the issue, I wrote new simplified code with the same problem.

#### Full example

<script src="https://gist.github.com/niklak/09852436ba94e436cf8629b50d3af146.js"></script>

Try on [play.rust-lang.org](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=cc326b7c9416bdf7536c5710ef6fa281).

Compiler hints that it needs `dep` to have a *static lifetime*.

Here you can leave everything as it is and make the object static, but this will have to be done every time `thread::spawn(move || {})` occurs. Therefore, let's consider two small options to prevent such situations from arising.


#### Cloning a string value

<script src="https://gist.github.com/niklak/6b5a7ec5ee9c064539da67f9d874011e.js"></script>

This is the simplest and most effective option. If a text field is not copied during the program's execution, then I believe there is nothing to optimize (it can also return a reference to its String field).

If a method of a structure accepts some string value (`String` or `&str`) and performs a chain of various string operations on the structure's field (*property*) and the method's *argument* and some of these operations also return different string types (`replace` returns `String`, `trim` returns `&str`), then it is also **not possible** to return `&str`. Because a function that returns a `String` inside the method will not allow returning `&str`, since the method cannot return a reference to a value whose ownership ends inside the method.
(**a value referencing data owned by the current function**).
The essential point is how often this field is cloned during the program's execution. If unnecessary cloning can be avoided, then `Cow<str>` should be used.

#### Using Copy-on-write (Cow)

<script src="https://gist.github.com/niklak/8ffe18c5b5ee2aacdb78246559af1a22.js"></script>

In this approach, we clone the value of `Dependency.name` into `LongLivingEntity.dep_name` using `Cow`. This way, the ownership of `Dependency.name` remains intact, while `LongLivingEntity` has a lifetime independent of `Dependency`. Using Cow allows us to optimize our code in the future.

#### a working example after changes

<iframe src="https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=b3c3ec25d3e55de85ac3f668a1ee3a78" style="width:100%; height:500px;"></iframe>  




Although this is not my case at all, let's consider a situation where it is still advisable to return references to strings in some form - either as `&str` or as a `struct` with a nested `&str` field.
So, we have a *long-lived structure* that generates many temporary objects during program execution, and these objects are only needed to perform operations on string (string slice -- `&str`) in the future, such as searching by regex. Here, we don't need the actual string; having a reference to the slice (&str) is sufficient. That is, nothing needs to be cloned, and as a result, we can return an object that contains a field with a reference (`&String` or `&str`) to the generating object.


#### String ref benchmark

```rust

use criterion::{criterion_group, criterion_main, Criterion};

pub struct SubordinateStr<'a> {
    s: &'a str,
}
pub struct StrRefGen<'a> {
    s: &'a str,
}

impl StrRefGen<'_> {
    pub fn new(s: &str) -> StrRefGen {
        StrRefGen { s }
    }

    pub fn gen_slice(&self) -> SubordinateStr {
        SubordinateStr { s: self.s }
    }
    
}

pub struct StringGen {
    s: String,
}

impl StringGen {
    pub fn new(s: String) -> StringGen {
        StringGen { s }
    }

    pub fn gen_slice(&self) -> SubordinateStr {
        SubordinateStr { s: &self.s }
    }
}


fn string_gen_benchmark(c: &mut Criterion) {
    let generator = StringGen::new("The quick brown fox jumps over the lazy dog".to_string());
    c.bench_function("string_gen", |b| b.iter(|| generator.gen_slice()));
}

fn str_gen_benchmark(c: &mut Criterion) {
    let generator = StrRefGen::new("The quick brown fox jumps over the lazy dog");
    c.bench_function("str_gen", |b| b.iter(|| generator.gen_slice()));
}

criterion_group!(benches, str_gen_benchmark, string_gen_benchmark);
criterion_main!(benches);

```


```
str_gen                 time:   [432.26 ps 434.29 ps 436.49 ps]
Found 5 outliers among 100 measurements (5.00%)
  4 (4.00%) high mild
  1 (1.00%) high severe

string_gen              time:   [435.59 ps 437.57 ps 439.67 ps]
Found 5 outliers among 100 measurements (5.00%)
  4 (4.00%) high mild
  1 (1.00%) high severe
```

In this case, the difference in the operation of the generating structures, in both variants (with string fields and fields of references to strings), will be minimal.

---

*What I like about Rust is that it makes you think deeper. I don't recall ever thinking about object lifetimes before Rust.*
