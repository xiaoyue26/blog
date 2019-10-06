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


## 函数
用fn声明. 
```rust
fn main() {
    another_function(5, 6);
}

fn another_function(x: i32, y: i32) {
    println!("The value of x is: {}", x);
    println!("The value of y is: {}", y);
}
fn five() -> i32 { // 返回i32类型
    5
}
```
## 表达式
表达式的结尾没有分号
```rust
let y = {
        let x = 3;
        x + 1 // 没有分号
    };
// y=4
```
## 循环
loop,while,for
```rust
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2;
    }
};
assert_eq!(result, 20);
while index < 5 {
    println!("the value is: {}", a[index]);
    index = index + 1;
}
let a = [10, 20, 30, 40, 50];
for element in a.iter() {
    println!("the value is: {}", element);
}
for number in (1..4).rev() {
    println!("{}!", number);
}
```

# ownership 所有权
rust无需gc。
要学习的点包括: 借用、slice、内存布局。
> 所有权：管理堆数据

所有权的三大法则: 
> 1. Rust中的每一个值都有一个被称为其 所有者（owner）的变量。
2. 值有且只有一个所有者。
3. 当所有者（变量）离开作用域，这个值将被丢弃。

创建一个堆上的变量:
```rust
{
let s = String::from("hello");
s.push_str(", world!"); // 追加
}// rust自动调用s的drop,回收内存(类似于free\RAII模式)
```
s的大小运行时可变，因此它显然分配在堆上。(栈每个slot大小相同)

## 浅拷贝、深拷贝、移动
rust对复杂类型默认是移动;
基本类型直接深拷贝。
rust没有浅拷贝、只有移动。
```rust
// 移动:
let s1 = String::from("hello");
let s2 = s1;
println!("{}, world!", s1); // fail,s1已经无效,被移动为s2了。
// 深拷贝:
let s1 = String::from("hello");
let s2 = s1.clone();
println!("s1 = {}, s2 = {}", s1, s2);
// 基本类型:
let x = 5;
let y = x;
println!("x = {}, y = {}", x, y);
```
总结就是: 
浅拷贝: 无;
深拷贝: 显式调用`clone`、或者是基本类型;
移动:   复杂类型;

如果一个类型拥有`Copy trait`,
一个旧的变量在将其赋值给其他变量后仍然可用。
rust的逻辑是，如果发现一个类型没有实现`Copy`，它就进行`move`。

除了用等号，调用函数时也会发生移动或者深拷贝。
例如:
```rust
let s = String::from("hello");  // s 进入作用域
takes_ownership(s);             // s 的值移动到函数里 ...
                                // ... 所以到这里s不再有效
```
函数return的时候也类似于等号，也会发生移动或者深拷贝，因此可以用return再取回所有权。
```rust
let s = String::from("hello");
let s = takes_back_ownership(s);
```
可以用`&`号来简化这个过程:
```rust
let s1 = String::from("hello");
let len = calculate_length(&s1); // 多一个&来取回所有权
```
这里`calculate_length`函数没有所有权，因此是借用了s1变量。
## 借用
函数借用变量s,不拥有所有权。
```rust
fn main() {
    let mut s = String::from("hello");
    change(&mut s);
}
fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```
借用并且要修改的话，要显式写上`&mut`.
### 借用的竞态
rust默认禁止竞态,编译不予通过:
```rust
let mut s = String::from("hello");
let r1 = &mut s;
let r2 = &mut s;// 如果后续都用的话报错,两个变量都借用了s,而且都是可写，有竞态,在同一作用域内。
// 只读引用的话可以有多个:
let r1 = &s; // no problem
let r2 = &s; // no problem
let r3 = &mut s; // BIG PROBLEM 有只读引用的时候，也不能再有可写引用
```
借用结束的话可以消除竞态:
```rust
let mut a = String::from("hello world");
let b = &a[0..4];
println!("{}", b);
append_a(&mut a);// 这里没问题,因为b后面没有用到。
// println!("{}", b); // 这里会报错，因为b的作用域和a的有交叉。

```

### 悬挂指针
Rust 中编译器确保永远不会有悬挂指针。
构造悬挂指针: 
```rust
fn main() {
    let reference_to_nothing = dangle();
}
fn dangle() -> &String {
    let s = String::from("hello");
    &s // 编译失败
}
```

## slice
slice的类型多一个&,属于不可变引用。
比如string的slice类型: `&str`，
字符串的字面量的类型：`str`
slice语法很简单:
```rust
let s = String::from("hello");

let slice = &s[0..2];
let slice = &s[..2];
let slice = &s[0..len]; // 整个字符串
let slice = &s[..]; // 省略头尾
// 包含右端点:
let hello = &s[0..=4];
```
用slice的好处是可以预防错误，因为持有了不可变引用，其他试图修改s的操作就会被阻止，因为修改s的时候会申请可变引用，根据上一节中的竞态阻止，申请可变引用会失败。

### 数组的slice
类型是 `&[i32]`
```rust
let a = [1, 2, 3, 4, 5];
let slice = &a[1..3];
println!("{:?}",slice);
```

## 结构体struct
```rust
# struct User {
#     username: String,
#     email: String,
#     sign_in_count: u64,
#     active: bool,
# }
#
let mut user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};

user1.email = String::from("anotheremail@example.com");
```
以前写代码经常会有`this.email=email`这种机械重复的代码，rust提供了简写省略的方法。
构造函数的简写: (`new`)
```rust
fn build_user(email: String, username: String) -> User {
    User {
        email, // 这里省略了同名输入变量
        username,
        active: true,
        sign_in_count: 1,
    }
}
```
类似的，结构体的update也有相应的简写:
```rust
let user2 = User {
    email: String::from("another@example.com"),
    username: String::from("anotherusername567"),
    ..user1 // 这句话的意思是其他变量都按user1的值来赋值就好
};
```

结构体的实例方法和类方法区别在于有没有第一个`&self`参数,方法可以位于不同`impl`块中:
```rust
# #[derive(Debug)]
# struct Rectangle {
#     width: u32,
#     height: u32,
# }
#
impl Rectangle {
    fn area(&self) -> u32 { // 实例方法
        self.width * self.height
    }
}

impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}
impl Rectangle {
    fn square(size: u32) -> Rectangle {// 关联方法、类方法
        Rectangle { width: size, height: size }
    }
}
```

## 自动解引用功能
统一`obj.xxx()`操作和`obj->xxx()`.
rust自动解引用:
> 当使用 object.something() 调用方法时，Rust 会自动为 object 添加 &、&mut 或 * 以便使 object 与方法签名匹配。

## 枚举
枚举可以直接绑定数据类型：(类似于一种`typedef`)
```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}
let home = IpAddr::V4(127, 0, 0, 1);
let loopback = IpAddr::V6(String::from("::1"));
```
也可以作为朴素的数据(数字):
```rust
# enum IpAddrKind {
#     V4,
#     V6,
# }
#
let four = IpAddrKind::V4;
let six = IpAddrKind::V6;
```
`Option`也是一种枚举类型。

## match
match的时候可以自动unapply枚举型:
```rust
match x {
        None => None,
        Some(i) => Some(i + 1),
    }
// 更复杂:
# #[derive(Debug)] // 支持直接打印
# enum UsState {
#    Alabama,
#    Alaska,
# }
#
# enum Coin {
#    Penny,
#    Nickel,
#    Dime,
#    Quarter(UsState),
# }
#
match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state);
            25
        },
        _ => (), 
}
```
这里类似于最后default兜底的值是`_`,返回值是`()`也就是`unit`类型。
`if let`是match的语法糖:
```rust
let mut count = 0;
if let Coin::Quarter(state) = coin {
    println!("State quarter from {:?}!", state);
} else {
    count += 1;
}
```

# 包
package: 包。cargo的功能.`cargo new`生成,带有`Cargo.toml`文件;里面可以有多个库。
Crates： 库。很多模块构成的库；(或者程序)
Modules: 模块。


包默认生成的库：
1.  `src/main.rs`; (程序、数量任意)
2.  `src/lib.rs`;  (库、最多1个)

# 模块
```rust
mod sound {// 同包下可以访问
    pub mod instrument {// 公有
        pub fn clarinet() {// 公有
            // 函数体
        }
    }
}
fn main() {
    // 绝对路径
    crate::sound::instrument::clarinet();
    // 相对路径
    sound::instrument::clarinet();
}
```
## super相对路径

```rust
mod sound {
    mod instrument {
        fn clarinet() {
            super::breathe_in();
        }
    }
    fn breathe_in() {
        // 函数体
    }
}
```

## use关键字引用
类似于`import`
```rust
use crate::sound::instrument;
// 相对路径引入:
use self::sound::instrument; // 一般还是用绝对路径引入

// 同名冲突处理: 使用as重命名
use std::fmt::Result;
use std::io::Result as IoResult;

```
默认use引入的项变成了私有，可以再加上pub让引入的项维持公有:
```rust
pub use crate::sound::instrument;
```

## 访问修饰符pub
枚举`enum`: 一旦pub，则所有字段pub;
结构体`struct`: 必须显式设定每个字段为pub，默认是private;

## 模块化
每个mod放在自己的同名文件中，其他文件中要用的时候，声明一下即可:
```rust
mod sound;
```
末尾是分号。

