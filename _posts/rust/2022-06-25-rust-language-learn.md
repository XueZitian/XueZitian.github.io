---
layout: post
title: Rust编程语言学习
category: rust
---

# 1 基本语法

## 1.1 变量和常量

### 1.1.1 变量的声明
```rust
// by default variables are immutable
let x = 5;
// the way that make variables mutable
let mut x = 6;
```

### 1.1.2 常量的声明
```rust
const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
```

### 1.1.3 常量与变量的异同
* 同：常量和不可变变量非常类似，只可以在定义时为变量（或常量）赋初始值，使用过程不可以改变数值；
* 异：常量在定义时，必须声明常量的类型，而变量则允许根据所赋予的数字，来推测出变量的类型；
* 异：常量可以在任何范围声明，包括全局范围；***变量可以在什么范围声明？***
* 异：常量的初始值必须是常量表达式，不能是一个运行时才能确定数值的变量；
* 异：***常量和变量在内存中存在的形式是否有差别，比如在不同的段？***

### 1.1.4 变量的作用域
每个变量都有自己的作用域，往往作用域和其生命周期是相同的。但是有一个例外，当在一个变量的声明周期内，又定义一个同名的变量，那么第一个变量的作用域会发生变化，其中两者生命周期重叠的部分会被第二个变量占有，即这部分区域第二个变量的作用域生效。

### 1.1.5 变量类型的声明
```rust
let x: u32 = 5;
```

## 1.2 数据类型
Rust中的每个值都有特定的数据类型，可以分为：标量和复合类型。Rust是一个静态类型语言，即在编译阶段它就必须知道每个变量的类型。编译器可以基于赋值和我们的用法来推理变量的类型，但是如果编译器无法推理出变量类型，就需要我们显示的声明变量的类型。

### 1.2.1 标量类型(Scalar Types)
* 整型(integer): i/u8, i/u16, i/u32, i/u64, i/u128, **i/usize**(arch)，默认类型是i32。debug模式，Rust会包含对整型数溢出的检查，而release模式则不会。另外，也可以显示的处理溢出情况，标准库提供了一些方法：wrapping_\*, checked_\*, overflowing_\*, saturating_\*。

    | Number literals    | Example     |
    |-----------------   |-------------|
    | 十进制(Decimal)     | 98_222      |
    | 十六进制(Hex)       | 0xff        |
    | 八进制(Octal)       | 0o77        |
    | 二进制(Binary)      | 0b1111_0000 |
    | 字节(Byte)(u8 only) | b'A'        |

* 浮点类型(floating-point)：f32, f64，默认类型是f64。
* 布尔类型(booleans)：bool，包含两个值：true和false，大小是一个字节。
* 字符类型(characters)：char，大小是四个字节，表示Unicode Scalar Value，从U+0000到U+D7FF，从U+E000到U+10FFFF。
    ```rust
    fn main() {
        let c = 'z';
        let z: char = 'ℤ'; // with explicit type annotation
        let heart_eyed_cat = '😻';
    }
    ```

### 1.2.2 算术运算
加减乘除以及求余，其中整数除法会向下舍入到最接近的整数。
```rust
fn main() {
    // addition
    let sum = 5 + 10;
    // subtraction
    let difference = 95.5 - 4.3;
    // multiplication
    let product = 4 * 30;
    // division
    let quotient = 56.7 / 32.2;
    let floored = 2 / 3; // Results in 0
    // remainder
    let remainder = 43 % 5;
}
```

### 1.2.3 复合类型(Compound Types)
复合类型可以将多个数值组合成一个类型，rust有两个复合类型的原语：元组(tuples)和数组(arrays)

* 元组类型(tuples): 将不同类型的一组数组合成一个复合类型，元组类型有固定的长度，不能动态伸缩。一个不包含任务数据的元组也称作单元(unit)。如果表达式不返回任何其它值，表达式会隐式的返回一个unit类型。
    ```rust
    // create a tuple with type annotations
    let tup: (i32, f64, u8) = (500, 6.4, 1);
    // To get the individual values out of tuple by pattern matching
    let (x, y, z) = tup;
    // also access a tuple element directly by using a . followed by the index
    let first = tup.0;
    let third = tup.1;
    // unit type, how to use it?
    let unit: () = ();
    ```

# Reference
1. [The Rust Programming Language][1]

[1]: https://doc.rust-lang.org/book/