---
layout: post
title: Rust引用
category: rust
---

引用类似一个指针，它是一个地址，指向了变量存储的位置。以String类型为例，之前我们说过String变量由两部分：一部分是存储在栈中的控制字段，另一部分是存储在堆中的字符串文本。其中控制字段中包含了堆中数据的描述信息，比如地址，长度等。String变量的引用就是一个指向控制地段的指针。当我们通过引用发起对变量的访问时，变量的所有权不会发生转移，因此当我们使用引用作为函数的入参时，在该函数运行结束后，父函数中对应的变量仍然有效。与指针不同，引用的对象必须是一个实际定义了的某种类型的变量，并且只有在该变量的声明周期内引用才是有效的。
```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

从上面的代码块可以看到，对变量的引用是通过‘&’符号完成的，相反的，当我们想基于引用访问变量时，可以通过‘*‘符号来完成，叫做解引用。

我们将变量创建引用的这一行为称作借用。就像在现实生活中一样，如果一个人拥有某物，你可以向他们借用，用完后，你必须归还。

那么，
* 借用的东西，我可以修改吗？可以修改，但有一些约束，下面部分会解释
* 我借走后，是只有我才可以修改吗，***原本的拥有者还可以修改吗？？？***
* 什么时候完成，什么时候归还，是否有这两个概念？有，下面部分会解释

## 1 可变引用

引用和变量一样，默认是不可修改的，当我们想基于引用修改变量时，需要为引用变量添加mutable修饰，如下：
```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

但对于可变引用，有以下几个约束：
* 当为一个变量创建了一个可变引用，该变量不能再有其它引用。这是为了在编译时就将数据的竞争拦下来，这也是rust语言安全性的一大特点。因为，数据竞争和如何有效规避数据竞争是编程中的一个老大难问题。一旦数据竞争的问题泄露到运行时，可能就非常难以发现和定位；
* 同样，对于一个变量，同时存在一个可变引用和一个不可变引用，也是不允许的。因为这种情况，同样可能会出现数据竞争；
* 只包含多个不变引用是被允许的。

数据竞争会在以下三个场景发生：
1. 两个或多个指针同时访问相同的数据；
2. 至少有一个指针用于写入数据；
3. 没有用于同步数据访问的机制。

上面这些场景的发生一般是因为多核或者cpu调度导致的，***那么rust中哪些数据(有全局变量的概念吗，多线程又是怎样的)在什么场景下会发生上述现象呢？？？***

与往常一样，我们可以使用大括号创建一个新范围，允许多个可变引用，但不能同时引用：
```rust
    let mut s = String::from("hello");

    {
        let r1 = &mut s;
    } // r1 goes out of scope here, so we can make a new reference with no problems.

    let r2 = &mut s;
```

对于借用的开始和结束，rust是这样定义的，引用从创建时开始，一直持续到最后一次使用该引用。下面的例子，r1是变量s的一个不可变引用，虽然r1的作用域可以持续到r3创建之后，但是在r3创建之后，已经没有对r1的使用，所以可变引用r3的创建并不会导致编译器报错，因为编译器可以识别出r1的借用在r3创建之前已经结束了。
```rust
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    println!("{} and {}", r1, r2);
    // variables r1 and r2 will not be used after this point

    let r3 = &mut s; // no problem
    println!("{}", r3);
```

## 2 悬空指针

TODO

# Reference
1. [The Rust Programming Language][1]

[1]: https://doc.rust-lang.org/book/