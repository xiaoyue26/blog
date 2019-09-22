---
title: rust入门笔记_1
date: 2019-09-22 19:20:21
tags: rust
categories: 
- rust

---

周末无事看看tidb开发者说很好用的rust，觉得很有趣，原来现代编程语言就是这种感觉，有一些细节上的简化。

文档参见: https://rustlang-cn.org/office/rust/book/getting-started/ch01-03-hello-cargo.html

源码:
```rust
fn main() {
    println!("Hello, world!");
}
```
编译和运行:
```shell
rustc main.rs
./main
```

用`cargo`(类似于`maven`)
```shell
cargo new hello_cargo # 创建项目
cargo build # 生成debug程序
cargo run # 运行debug程序, 会自动build
cargo check # 仅检查语法

cargo build --release # 生成优化后的程序(release)
# 类似的可以猜到run release程序的命令:
cargo run --rebase # 运行relase，会自动检测改动重新编译
```
可以看到专门提供了一个`cargo check`命令来避免编译、只是检查语法，看来网上大家说rust编译慢很可能是真的。

# 变量

```rust
let foo = 5; // 不可变
let mut bar = 5; // 可变
```

类方法/静态函数: 在rust中叫`关联函数`（associated function）;

# crate: 库
类似于mvn的中央仓库: https://crates.io/
`crate`不是创建的意思，差了一个字母，是rust的库的意思。

# 数据类型
`i32`: 32位数字；
`u32`: 32位无符号数字；
`i64`: 64位数字等等。

`Rust`默认使用`i32`.
但是如果你用`u32`类型和一个变量`a`比较,Rust会推断出a也是`u32`类型.

# let关键字
`let`类似于js里的`let`，用来定义一个变量，而且支持`shadowing`。
比如一开始定义了一个string类型的a;
后来转换成数字以后，可以直接
```rust
let a:u32 = a.guess.trim().parse()
        .expect("Please type a number!");
```
新的定义会覆盖以前的，这样就不用定义两个变量了（一个string_a,一个u32_a）。
原理上其实底层是生成了两个变量，因此可以把`let mut`覆盖成`let`，或者把不可变的覆盖成可变的。实测了一下确实也是可以的。

# match关键字
match和scala里的一样
```rust
match xxx {
        Ordering::Less => println!("Too small!"),
        Ordering::Greater => println!("Too big!"),
        Ordering::Equal => println!("You win!"),
}
```

# 通用概念(与其他编程语言核心对应的)

## 关键字转义
比如`match`在rust中是一个关键字,所以如果恰好有一个函数叫这个名字，需要转义以后才能调用：(用`r#`前缀)
```rust
r#match(); // 调用名为 'match' 的函数
```

## 不可变
```rust
const MAX_POINTS: u32 = 100_000; // 不可变,直接赋值;
let a = get_val(); // 不可变,可以运行时赋值;
let mut a = xxx;   // 可变。
```

## 数据类型
rust会尝试推断数据类型，推断不出来则会报错；

标量: 整型、浮点型、布尔类型和字符类型;

| 长度   | 有符号 | 无符号 |
|--------|--------|--------|
| 8-bit  | i8     | u8     |
| 16-bit | i16    | u16    |
| 32-bit | i32    | u32    |
| 64-bit | i64    | u64    |
| arch   | isize  | usize  |

这里的`arch`: 64 位架构上它们是 64 位的， 32 位架构上它们是 32 位的。
`isize`和`usize`主要作为索引类型。

赋值的时候: (还能在中间随意加横杠`_`):

| 数字字面值     | 例子        |
|----------------|-------------|
| Decimal        | 98_222      |
| Hex            | 0xff        |
| Octal          | 0o77        |
| Binary         | 0b1111_0000 |
| Byte (u8 only) | b'A'        |

`57u8`表示57，数据类型是`u8`；

数字溢出: debug版检查溢出并报错；
release版会进行溢出。
可以用`Wrapping`类型来使用溢出特性，以免被debug版本报错。

`f32`: 32位浮点数; 
`f64`: 64位浮点数. (默认类型,现代cpu下性能与f32几乎一样)
`bool`: 布尔值。
`char`: Unicode字符。

## 元组和数组
元组下标从0开始(和scala不同,scala从1开始)
```rust
let tup: (i32, f64, u8) = (500, 6.4, 1);
let five_hundred = x.0;
```
数组:
```rust
let a = [1, 2, 3, 4, 5];
// 或:
let a: [i32; 5] = [1, 2, 3, 4, 5];
```

