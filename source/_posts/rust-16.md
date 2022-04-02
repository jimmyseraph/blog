---
title: Rust学习笔记（16）
author: 六道先生
date: 2022-04-01 18:10:20
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第十六天开始了，主要学习Rust关于并发的使用。
---
# 线程
系统来说，线程就是一个进程中同时运行的多项工作。表现在代码中，就是有独立的几段代码互不干涉各自运行。在不同的编程语言中，有些是1:1模型，也就是每个系统线程就是一个程序线程，像Java就是如此。有些则是M:N模型，就是M个系统线程运行着N个程序线程，类似Golang这样的，我们也有把这种程序线程称为协程的。<br>
协程这类模型，需要编程语言本身来控制这些线程调度，所以需要需要处理的运行态事件比较多，基于这种考虑，Rust的标准库默认提供了1:1线程模型的方式。当然，如果你确实需要使用协程，也可以用引入一些包（crates）。<br>
来看一个线程使用的例子：
```Rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```
spawn方法接受一个闭包参数，用于指定线程要运行的代码。看看输出：
```Shell
hi number 1 from the main thread!
hi number 1 from the spawned thread!
hi number 2 from the main thread!
hi number 2 from the spawned thread!
hi number 3 from the main thread!
hi number 3 from the spawned thread!
hi number 4 from the main thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
```
可以看到先输出的是`main thread`，然后因为sleep，main线程就暂停了，使得`thread::spawn`有机会输出。但是因为main循环到4之后，就结束了，所以thread中没有机会再输出，只打印到了5。<br>
如果要等到所有线程都运行结束才终止，可以使用join函数：
```Rust
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

    handle.join().unwrap();
}
```
> join所起的作用，就是将一个线程合并到当前线程上来，成为同一个线程。<br>

在使用spawn时，里面传入的闭包需要注意ownership的问题，看下面这个例子：
```Rust
fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("Here's a vector: {:?}", v);
    });

    handle.join().unwrap();
}
```
这段代码会出现编译异常：
```Shell
$ cargo run
   Compiling threads v0.1.0 (file:///projects/threads)
error[E0373]: closure may outlive the current function, but it borrows `v`, which is owned by the current function
 --> src/main.rs:6:32
  |
6 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `v`
7 |         println!("Here's a vector: {:?}", v);
  |                                           - `v` is borrowed here
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:6:18
  |
6 |       let handle = thread::spawn(|| {
  |  __________________^
7 | |         println!("Here's a vector: {:?}", v);
8 | |     });
  | |______^
help: to force the closure to take ownership of `v` (and any other referenced variables), use the `move` keyword
  |
6 |     let handle = thread::spawn(move || {
  |                                ++++
```
原因就在于，闭包代码会在一个新的线程中运行，而println使用变量v仅需要借用即可，所以v是被引用到闭包中的。而Rust不知道这个v在原代码中的生命周期有多久，所以就出现了这个编译错误，因为一旦v失效，这个闭包就无法引用到这个变量的值了。可以使用move关键字来强制转移owner：
```Rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
    });

    handle.join().unwrap();
}
```
# 线程间通讯
Rust中进行线程通讯使用channel，我们用`mpsc::channel()`来进行消息发送，mpsc是<i>multiple producer, single consumer</i>（多生产者，单消费者）的缩写，我面看看例子：
```Rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```
`mpsc::channel()`会返回一个tuple，第一个元素是一个发送端，第二个元素是接收端，所以我面使用`let (tx, rx)`这种形式来接收，将发送端赋值给tx，接收端赋值给rx。接收端有两个方法，一个是`recv`另一个是`try_recv`，`recv`是阻塞式的，如果没有收到数据，会一直阻塞住当前的线程；`try_recv`是非阻塞式的。<br>

这里也要注意了，send的行为或将ownership移走，也就是说send的内容是move行为：
```Rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
        println!("val is {}", val);
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```
这段代码会报错，因为send之后的println其实已经引用不到val变量了。<br>

再来看看如何处理多次数据发送：
```Rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```
这里用for循环，处理tx的多次发送，直到thread结束发送，这个for循环也就结束了，我面可以看到下面的打印：
```Shell
Got: hi
Got: from
Got: the
Got: thread
```

这里再看看多个生产者的情况：
```Rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {

    let (tx, rx) = mpsc::channel();

    let tx1 = tx.clone();
    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx1.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    thread::spawn(move || {
        let vals = vec![
            String::from("more"),
            String::from("messages"),
            String::from("for"),
            String::from("you"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```
看看打印结果：
```Shell
Got: hi
Got: more
Got: from
Got: messages
Got: for
Got: the
Got: thread
Got: you
```
# 锁机制
在Rust中可以使用`Mutex<T>`来进行锁操作，先看基本用法：
```Rust
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
但是如果直接在多线程中使用，会出现问题：
```Rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handles = vec![];

    for _ in 0..10 {
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```
代码编译直接报错：
```Shell
$ cargo run
   Compiling shared-state v0.1.0 (file:///projects/shared-state)
error[E0382]: use of moved value: `counter`
  --> src/main.rs:9:36
   |
5  |     let counter = Mutex::new(0);
   |         ------- move occurs because `counter` has type `Mutex<i32>`, which does not implement the `Copy` trait
...
9  |         let handle = thread::spawn(move || {
   |                                    ^^^^^^^ value moved into closure here, in previous iteration of loop
10 |             let mut num = counter.lock().unwrap();
   |                           ------- use occurs due to use in closure
```
很明确，`Mutex<i32>`在循环开始就已经move掉了，后面的就拿不到了。为了能让所有线程都能共享`Mutex<i32>`，我们可以想到`Rc<T>`，但是它没有实现Send特性，也就是说不能在线程间安全传输数据，所以也会报错。<br>
这里我们必须使用`Arc<T>`，看例子：
```Rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```
这段代码可以正常运行。