---
title: rust入门笔记_2
date: 2019-10-06 18:58:26
tags: rust
categories: 
- rust

---

# 解引用强制多态（`deref coercions`）
`coercions`的意思就是强制多态。
解引用强制多态的意思通俗来说就是，编译器根据目标类型，自动调用`deref`方法解引用，搜索路径来达到目标类型，最终达到节省程序员编写成本，但最后运行时开销维持最小的目标。(语法糖成本为0)

## 背景知识: 智能指针的解引用
自定义的智能指针，实现`Deref trait`(解引用特性)后，就可以使用`*`操作符来解引用了。
例如:
```rust
use std::ops::Deref;

# struct MyBox<T>(T);
impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T { // 注意这里的返回值需要是引用,以便维持所有权
        &self.0
    }
}
```
编译器每次遇到`*`操作符，都会尝试调用复杂类型的`deref`方法来获得基本类型的引用，以便进行解引用，也就是代码等效于:`*(y.deref())`。

## 实例
```rust
let m = MyBox::new(String::from("Rust"));
hello(&m); // 这里hello的形参是&str,实参是&MyBox
// 强制解引用多态后，等效于调用了: hello(&(*m)[..]);
// &MyBox =通过deref=> String =通过slice=> &str 
```

## 更多解引用多态规则
```rust
1. 当 T: Deref<Target=U> 时从 &T 到 &U。 // 不可变=>不可变
2. 当 T: DerefMut<Target=U> 时从 &mut T 到 &mut U。 // 可变=>可变
3. 当 T: Deref<Target=U> 时从 &mut T 到 &U。 // 可变=>不可变
```
只会发生安全的转换，而`不可变=>可变`这种属于不安全，因此不支持自动转换。

(上面的实例属于第一种。)

# (析构函数) std::mem::drop
自定义Drop trait:
```rust
impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}
```
定义以后drop函数不能显式调用，也不能禁用。
只能在离开作用域的时候自动调用。
如果要显式调用，只能使用标准库的函数`std::mem::drop`.
(默认预引入，可以直接调用drop)

# 智能指针汇总

Box: 没有特殊功能,类似于java中的普通引用,让编译器能确定分配空间大小;
Rc: 多所有权、单线程、引用计数;
RefCell: 单所有权、单线程、引用计数、内部可变性;
Arc: 多所有权、多线程、引用计数;
Mutex: 单所有权、多线程、引用计数、内部可变性;
Week: 弱引用。


# 多所有权、单线程、不可变：引用计数=>Rc<T> 
多所有权示例: b,c都拥有a的所有权。
```rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}
```
每次调用`Rc::clone`，都将引用计数加1.// 这里是浅拷贝
b: 3->5->10->Nil
c: 4->5->10->Nil

打印引用数:
```rust
fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!("count after creating a = {}", Rc::strong_count(&a));
    let b = Cons(3, Rc::clone(&a));
    println!("count after creating b = {}", Rc::strong_count(&a));
    {
        let c = Cons(4, Rc::clone(&a));
        println!("count after creating c = {}", Rc::strong_count(&a));
    }
    println!("count after c goes out of scope = {}", Rc::strong_count(&a));
}
```

# 单一所有权、单线程、内部可变：引用计数=>RefCell<T> 

## 借用规则
1. 每时每刻，要么有一个可变引用;要么有n个不可变引用；(这两者互斥)
2. 引用必须总是有效的。

对于引用和`Box<T>`: `编译时`进行借用规则检查；(编译错误)
对于`RefCell<T>`:   `运行时`进行借用规则检查；(`panic`)

引用和`Box<T>`: 第一级指针不可变，则级联得每一级都不可变。
`RefCell`: 第一级指针不可变，第二级可以借用可变引用，然后修改；

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger { sent_messages: RefCell::new(vec![]) }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
        // 注意这里的borrow_mut，借出可变:
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        // --snip--
#         let mock_messenger = MockMessenger::new();
#         let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);
#         limit_tracker.set_value(75);
        // 注意这里的borrow，借出不可变:
        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```

`borrow` 方法返回 `Ref` 类型的智能指针；
`borrow_mut` 方法返回 `RefMut` 类型的智能指针。

这两个类型都实现了 `Deref` 所以可以当作常规引用对待。
借用规则在运行时检查，例如同时借出两个可变引用将会panic:
```rust
let mut one_borrow = self.sent_messages.borrow_mut();
let mut two_borrow = self.sent_messages.borrow_mut();
```
底层原理依然是引用计数。

## 结合Rc和RefCell
结合Rc和RefCell的话，就可以结合多所有权和单所有权，不可变和可变:
```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),// 注意这里
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```
混合使用`Rc`和`RefCell`可以构造循环引用，造成内存泄露。
可以用弱引用来消除这一隐患: 将 `Rc<T>` 变为 `Weak<T>`。

# 弱引用
https://rustlang-cn.org/office/rust/book/smart-pointers/ch15-06-reference-cycles.html

# 并发
## 线程模型
rust标准库提供1:1线程，有其他第三方库提供M:N的。
创建线程: `thread::spawn`+闭包：
```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
    handle.join().unwrap();// 阻塞等待子线程
}
```
子线程中可以捕获主线程的变量，获得所有权. 使用`move`，以免主线程把它drop了：
```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
    });

    handle.join().unwrap();
}
```

## 消息传递: channel

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    // tx: 发送者
    // rx: 接收者
    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();// 单所有权,发送后不能再使用val
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```
mpsc： 多生产者、单消费者 
recv： 阻塞接收
try_recv: 非阻塞接收，立即返回

这里之所以是多生产者，是因为可以把生产者无限克隆出去，然后发送消息：
```rust
let tx1 = mpsc::Sender::clone(&tx);
```

## 共享状态并发
## 互斥器(mutex)
rust中的锁是一种特殊的智能指针，通过重载drop trait来确保离开作用域的时候释放锁。
```rust
use std::sync::Mutex;
fn main() {
    let m = Mutex::new(5);
    {
        let mut num = m.lock().unwrap();
        *num = 6;
    }
    println!("m = {:?}", m);
}
```

由于多个线程都需要访问同一个锁，因此需要多所有权的智能指针，并且能够并发使用：
```rust
Rc<T>: 单线程，多所有权；
Arc<T>: 多线程，多所有权。
```
并发访问的例子:
```rust
use std::sync::{Mutex, Arc};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));// 智能指针包装
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);// 克隆来增加引用计数
        let handle = thread::spawn(move || { 
            let mut num = counter.lock().unwrap();

            *num += 1; // move+离开作用域,引用-1
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

这里可以看出Mutex具有内部可变性。同时可以用Mutex构造死锁(循环依赖)。

## Send与Sync的trait
rust中用两个trait来标记所有权在线程中的转移以及引用的多线程访问: 
> `Send trait`:  支持多线程`所有权`转移,所有权在线程之间转移; 除Rc<T>以外的大部分类型是Send trait;
`Sync trait`:  支持多线程访问, 线程之间可以共享值的`引用`；
除Rc<T>以外的大部分类型是Sync trait;

`unsafe rust`中的裸指针也没有实现这两个trait.

绝大部分情况不需要手动实现send与sync的trait。