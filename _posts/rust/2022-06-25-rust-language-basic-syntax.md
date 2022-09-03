---
layout: post
title: Rust基本语法
category: rust
---

## 1 变量和常量

### 1.1 变量的声明
```rust
// by default variables are immutable
let x = 5;
// the way that make variables mutable
let mut x = 6;
```

### 1.2 常量的声明
```rust
const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
```

### 1.3 常量与变量的异同
* 同：常量和不可变变量非常类似，只可以在定义时为变量（或常量）赋初始值，使用过程不可以改变数值；
* 异：常量在定义时，必须声明常量的类型，而变量则允许根据所赋予的数字，来推测出变量的类型；
* 异：常量可以在任何范围声明，包括全局范围；***变量可以在什么范围声明？***
* 异：常量的初始值必须是常量表达式，不能是一个运行时才能确定数值的变量；
* 异：***常量和变量在内存中存在的形式是否有差别，比如在不同的段？***

### 1.4 变量的作用域
每个变量都有自己的作用域，往往作用域和其生命周期是相同的。但是有一个例外，当在一个变量的声明周期内，又定义一个同名的变量，那么第一个变量的作用域会发生变化，其中两者生命周期重叠的部分会被第二个变量占有，即这部分区域第二个变量的作用域生效。

### 1.5 变量类型的声明
```rust
let x: u32 = 5;
```

## 2 数据类型
Rust中的每个值都有特定的数据类型，可以分为：标量和复合类型。Rust是一个静态类型语言，即在编译阶段它就必须知道每个变量的类型。编译器可以基于赋值和我们的用法来推理变量的类型，但是如果编译器无法推理出变量类型，就需要我们显示的声明变量的类型。

### 2.1 标量类型(Scalar Types)
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

### 2.2 算术运算
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

### 2.3 复合类型
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
* 数组类型(Array)：不同于元组，数据中的每一个元素都必须是相同的类型，并且数组必须是固定长度。数组存在于栈中，而不是堆中。当使用一个变量作为访问数组的index，如果在运行时index超过了数组的最大长度，程序会退出并打印一个错误信息，因此，不同于c语言，Rust会在运行时检查是否发生了数组越界，这就是体现Rust内存安全特性的一个具体的例子。
    ```rust
    // define a array variable
    let a = [1, 2, 3, 4, 5];
    // add type and size annotations
    let a: [i32; 5] = [1, 2, 3, 4, 5];
    // init an array to contain the same value for each elem
    let a = [3; 5]; // equivalent to let a = [3, 3, 3, 3, 3];
    // accessing array elem
    let first = a[0];
    let second = a[1];
    ```

## 3 函数

声明一个新的函数的关键字是fn，Rust函数和变量的常规风格是字母全是小写并使用下划线隔离单词。
```rust
fn main() {
    println!("Hello, World!");

    another_function();
}

fn another_function() {
    println!("Another function.");
}
```
对于上面的代码片段，可以看到函数‘another_function’的定义在引用之后。不同于c，Rust对被引用函数定义的位置要求比较宽松，只要是在调用者可以看到的范围内即可。

### 3.1 函数参数

函数允许定义入参，它是函数签名的一部分，这里需要注意的是入参必须显性指定参数的类型。定义多个入参时，用逗号分割开。
```rust
fn another_function(x: i32, unit_label: char) {
    println!("The value of x is: {x}{unit_label}");
}
```
### 3.2 陈述和表达式

函数主体由一系列的Statements和可选择的表达式结尾组成。Rust是一个表达式基的语言，那么表达式和陈述的区别是什么呢？先来看看两者的定义。
* Statements是执行某些操作且不返回值的指令。由于陈述不返回值，所以不可以使用一个陈述为变量赋值。
* Expressions会计算出一个结果值，组成了Rust代码的大部分。表达式不包含分号，为表达式添加分号将到时表达式变为陈述，表达式可以作为陈述的一部分。
```rust
// Statements:
// 1. Creating a variable and assigning a value to it with the let keyword
let y = 6;
// 2. Function definitions
// Expressions：
// 1. 算术运算，或者一个常数
5 + 6
// 2. Calling a function
// 3. Calling a macro
// 4. 用大括号创建的一个代码块，注意，x + 1不能添加分号，这将导致代码块变成一个Statement
{
    let x = 3;
    x + 1
}
```

### 3.3 函数返回值

Rust函数可以定义返回值，必须声明返回值的类型，但不用为返回值命名。函数的返回值是函数主体的最后一个表达式的值，也可以在函数主体中通过return关键字提前返回。
```rust
fn five -> i32 {
    5
}
```

## 4 控制流

### 4.1 if

if表达式允许我们基于条件跳转我们的代码。Rust的if语法和c语言很像，但是在Rust中，整数类型变量不能直接作为if的条件，必须使用bool类型，Rust不会自动将非布尔类型的变量转换成布尔类型。
```rust
fn main() {
    let number = 6;

    if number % 4 == 0 {
        println!("number is divisible by 4");
    } else if number % 3 == 0 {
        println!("number is divisible by 3");
    } else if number % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```

if是一个表达式，所有我们可以放它在let陈述的右边，将if的结构赋值给一个变量。
```rust
fn main() {
    let condition = true;
    let number = if condition { 5 } else { 6 };

    println!("The value of number is: {number}");
}
```

### 4.2 loop

loop关键字会无限次的执行一个代码块的循环，直到你显性的去让程序停止，比如键入Ctrl+c，或者在代码块中插入break关键字，另外，也可以在代码块中插入continue去调用代码块剩余部分，然后开始下一次的循环。
```rust
fn main() {
    loop {
        println!("again!");
    }
}
```

**loop的返回值**

loop的一种用法是，尝试一个可能失败的操作，然后在跳出loop的时候，你希望获取loop操作的结果，这个时候可以使用break，并在break后边加一个表达式。
```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    println!("The result is {result}");
}
```

**loop标签**

当使用多个loop嵌套的时候，可以使用break或者continue加标签直接从里面的loop跳到外层的loop。如，下面的标签：'counting_up
```rust
fn main() {
    let mut count = 0;
    'counting_up: loop {
        println!("count = {count}");
        let mut remaining = 10;

        loop {
            println!("remaining = {remaining}");
            if remaining == 9 {
                break;
            }
            if count == 2 {
                break 'counting_up;
            }
            remaining -= 1;
        }

        count += 1;
    }
    println!("End count = {count}");
}
```

### 4.3 while

while循环的如法如下：

```rust
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{number}!");

        number -= 1;
    }

    println!("LIFTOFF!!!");
}
```

### 4.4 for

Rust中的for循环与c语言有很大差别，Rust从安全性和易用性角度出发，使得for循环非常适合用在遍历某个集合(如，数组)中的每一个元素的场景。
```rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a {
        println!("the value is: {element}");
    }
}
```
上面基于for循环对数组遍历，不用担心发生数组访问越界的问题，语法也非常精炼。

另外，可以可以借助标准库提供的Range，来产生一个数字序列供for循环使用，甚至可以调用rev方法来反转数字序列。
```rust
fn main() {
    for number in (1..4).rev() {
        println!("{number}!");
    }
    println!("LIFTOFF!!!");
}
// output:
// 3!
// 2!
// 1!
// LIFTOFF!!!
```

# Reference
1. [The Rust Programming Language][1]

[1]: https://doc.rust-lang.org/book/