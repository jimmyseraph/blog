---
title: Rust学习笔记（15）
author: 六道先生
date: 2022-03-29 21:52:08
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第十五天开始了，主要学习Rust关于指针和引用的知识。
---
# 关于`Box<T>`
`Box<T>`是一种包装类型，保存的数据放在Heap中，而在Stack中保存了指向Heap中数据存放地址的指针。这种指针，为了区别C/C++的指针，在Rust中称为“智慧指针”（Smart Pointer）。<br>
通常我们会在以下三种场景下使用`Box<T>`：
1. 当某一数据明确大小的数据类型，在编译时候并不知道需要存多少；
2. 当一组较为庞大的数据，需要变更owner，我们需要确保不发生copy导致内存泄露时；
3. 当我获取一个值，但是我只关心它的类型是否实现了某个trait，而不关心类型本身的时候。

我们看看如何把一个值包装进Box：
```Rust
fn main() {
    let b = Box::new(5);
    println!("b = {}", b);
}
```

# 递归类型
我们来看什么是编译时不知道大小的类型，其中典型的就是嵌套类型，像这样的：
```Rust
enum List {
    Cons(i32, List),
    Nil,
}
```
Cons里面包含了自己本身，因为在编译时候不知道会包含多少层，看下面：
```Rust
use crate::List::{Cons, Nil};

fn main() {
    let list = Cons(1, Cons(2, Cons(3, Nil)));
}
```
这可以不断嵌套，所以对于这种未知大小的，直接这么定义，会出现编译异常：
```Shell
$ cargo run
   Compiling cons-list v0.1.0 (file:///projects/cons-list)
error[E0072]: recursive type `List` has infinite size
 --> src/main.rs:1:1
  |
1 | enum List {
  | ^^^^^^^^^ recursive type has infinite size
2 |     Cons(i32, List),
  |               ---- recursive without indirection
  |
help: insert some indirection (e.g., a `Box`, `Rc`, or `&`) to make `List` representable
  |
2 |     Cons(i32, Box<List>),
  |               ++++    +

error[E0391]: cycle detected when computing drop-check constraints for `List`
 --> src/main.rs:1:1
  |
1 | enum List {
  | ^^^^^^^^^
  |
  = note: ...which immediately requires computing drop-check constraints for `List` again
  = note: cycle used when computing dropck types for `Canonical { max_universe: U0, variables: [], value: ParamEnvAnd { param_env: ParamEnv { caller_bounds: [], reveal: UserFacing }, value: List } }`
```
它提示我们可以用Box来定义嵌套类型，因为Box是一个指针，所以它的大小是明确的，这样就不会在编译时报错：
```Rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
}
```
指针，其实我们也可以把它想象为引用的反操作，看下面的例子：
```Rust
fn main() {
    let x = 5;
    let y = &x;

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```
> y保存了x的引用，我们说过，这是一个borrow的过程，在下面的assert中，必须用`*y`来反引用到这个值本身，否则就会出现类型不匹配的问题：`no implementation for {integer} == &{integer}`。

用Box也是可以当作一种引用来操作的，本质一样：
```Rust
fn main() {
    let x = 5;
    let y = Box::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```
我们也可以自己来定义一个包装类型，类似Box这种的，但是必须要实现Deref这个trait：
```Rust
use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```
> 这里的`type Target = T; `语法，定义了一个关联类型，以供Deref使用，关联语法和之前介绍过的泛型有点区别，后面再说。<br>
在deref方法中，用返回`&self.0`的语法，返回了引用地址的内容。

当代码处理`*y`的时候，在背后其实是：
```Rust
*(y.deref())
```
再看看`String`和`&str`，`String`其实实现了Deref，通过Deref的强制多态特性，实现了`String`转为`&str`。我们看看关于Deref的强制多态特性，以我们实现的`MyBox<T>`为例来说明，现在我们有一个hello函数：
```Rust
fn hello(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&m);
}
```
> 我们传入的是`&m`，这个引用指向了MyBox，也就是`&MyBox<String>`，因为MyBox我们实现了Deref，所以会转为`&String`，而标准库中因为String实现了Deref特性，所以`&String`能被转为`&str`，满足hello函数入参的类型要求。

如果没有强制多态这个特性，那就需要自己在代码中写成这样了：
```Rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&(*m)[..]);
}
```
> `(*m)`反向引用了`MyBox<String>`，指向了String这个值，然后用`&`和`[]`将String转为了切片的引用，来满足hello函数的入参。这样的写法，明显麻烦了很多，可读性也差。

# Drop特性
Drop这个trait也是和指针强相关的，所有的指针背后都需要实现Drop，因为指针在离开了生命周期范围后，都需要调用Drop特性的方法来清理heap内存中保存的值。<br>
我们来主动实现一下Drop特性：
```Rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer {
        data: String::from("my stuff"),
    };
    let d = CustomSmartPointer {
        data: String::from("other stuff"),
    };
    println!("CustomSmartPointers created.");
}
```
程序在离开main的生命周期时，就会自动调用drop方法：
```Shell
$ cargo run
   Compiling drop-example v0.1.0 (file:///projects/drop-example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.60s
     Running `target/debug/drop-example`
CustomSmartPointers created.
Dropping CustomSmartPointer with data `other stuff`!
Dropping CustomSmartPointer with data `my stuff`!
```
> 注意为啥先打印了`other stuff`，想想指针本身保存在stack中，特点就是先进后出。

drop方法不能主动调用，也就是说，如果我们用`c.drop()`会出现编译错误，只能通过drop函数来主动使用：
```Rust
fn main() {
    let c = CustomSmartPointer {
        data: String::from("some data"),
    };
    println!("CustomSmartPointer created.");
    drop(c);
    println!("CustomSmartPointer dropped before the end of main.");
}
```
# 关于`Rc<T>`
我们已经知道，在Rust中，是不允许一个值被多个变量拥有的，比如下面这个情况：
```Rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let a = Cons(5, Box::new(Cons(10, Box::new(Nil))));
    let b = Cons(3, Box::new(a));
    let c = Cons(4, Box::new(a));
}
```
这段代码一定会编译报错，因为b中会把a的值move走，c再想拥有a的值，a已经失效了：
```Shell
$ cargo run
   Compiling cons-list v0.1.0 (file:///projects/cons-list)
error[E0382]: use of moved value: `a`
  --> src/main.rs:11:30
   |
9  |     let a = Cons(5, Box::new(Cons(10, Box::new(Nil))));
   |         - move occurs because `a` has type `List`, which does not implement the `Copy` trait
10 |     let b = Cons(3, Box::new(a));
   |                              - value moved here
11 |     let c = Cons(4, Box::new(a));
   |                              ^ value used here after move
```
如果我们把Cons改成保存引用行不行呢？这样不就不会发生move了吗？也不行，因为还有`Box::new(Nil)`这个赋值，`let a = Cons(10, &Nil)`这句话也是会报错的，因为Nil不可以被引用。<br>
这里我们需要使用`Rc<T>`来解决这个问题：
```Rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}
```
Rc的全称叫reference counting，引用计数，在这里我们使用了`Rc::clone`方法，这个clone和我们之前介绍ownership时候的clone不同，如果我们这里使用`a.clone`，那其实是发生了一次deep copy，而Rc的clone，只是在a上增加了一次引用计数。由于没有发生数据的复制，所以速度要比深复制快得多。现在的引用情况，就如下图所示：<br>
{% asset_img trpl15-01.svg trpl15-01 %}

我们现在用`Rc::strong_count`来返回此时的引用数量：
```Rust
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
执行结果是：
```Shell
$ cargo run
   Compiling cons-list v0.1.0 (file:///projects/cons-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.45s
     Running `target/debug/cons-list`
count after creating a = 1
count after creating b = 2
count after creating c = 3
count after c goes out of scope = 2
```
# `RefCell<T>`与内部可变
内部可变属性是Rust中的一种设计规格，用于对一个数据进行变更，即使这个数据变量为不可变引用。一般来说，Rust根据ownership规则，是不允许对不可变引用进行数据变更的。我们看一个例子：
```Rust
pub trait Messenger {
    fn send(&self, msg: &str);
}

pub struct LimitTracker<'a, T: Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T>
where
    T: Messenger,
{
    pub fn new(messenger: &T, max: usize) -> LimitTracker<T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }

    pub fn set_value(&mut self, value: usize) {
        self.value = value;

        let percentage_of_max = self.value as f64 / self.max as f64;

        if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        } else if percentage_of_max >= 0.9 {
            self.messenger
                .send("Urgent warning: You've used up over 90% of your quota!");
        } else if percentage_of_max >= 0.75 {
            self.messenger
                .send("Warning: You've used up over 75% of your quota!");
        }
    }
}
```
现在我们为此写一个测试：
```Rust
#[cfg(test)]
mod tests {
    use super::*;

    struct MockMessenger {
        sent_messages: Vec<String>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: vec![],
            }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);

        limit_tracker.set_value(80);

        assert_eq!(mock_messenger.sent_messages.len(), 1);
    }
}
```
此时的编译会出错：
```Shell
$ cargo test
   Compiling limit-tracker v0.1.0 (file:///projects/limit-tracker)
error[E0596]: cannot borrow `self.sent_messages` as mutable, as it is behind a `&` reference
  --> src/lib.rs:58:13
   |
2  |     fn send(&self, msg: &str);
   |             ----- help: consider changing that to be a mutable reference: `&mut self`
...
58 |             self.sent_messages.push(String::from(message));
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable
```
因为我们的测试中定义的MockMessenger实现了Messenger的send方法，但是因为用了不可变引用`&self`，所以我们`self.sent_messages`就不可被变更。编译器提示我们使用`&mut self`，但是我们不能这么写，因为如果写成这样，就不符合trait中send的定义样式了。<br>
这种情况，我们可以用`RefCell<T>`来解决这个问题：
```Rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: RefCell::new(vec![]),
            }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        // --snip--
        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```
现在我们把Rc和RefCel一起来用：
```Rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(3)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(4)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```
看输出：
```Shell
$ cargo run
   Compiling cons-list v0.1.0 (file:///projects/cons-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.63s
     Running `target/debug/cons-list`
a after = Cons(RefCell { value: 15 }, Nil)
b after = Cons(RefCell { value: 3 }, Cons(RefCell { value: 15 }, Nil))
c after = Cons(RefCell { value: 4 }, Cons(RefCell { value: 15 }, Nil))
```
# 内存泄漏问题
Rc和RefCell可能会引起内存泄漏问题，要小心使用。看下面的例子：
```Rust
use crate::List::{Cons, Nil};
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match self {
            Cons(_, item) => Some(item),
            Nil => None,
        }
    }
}

fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));

    println!("a initial rc count = {}", Rc::strong_count(&a));
    println!("a next item = {:?}", a.tail());

    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));

    println!("a rc count after b creation = {}", Rc::strong_count(&a));
    println!("b initial rc count = {}", Rc::strong_count(&b));
    println!("b next item = {:?}", b.tail());

    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    println!("b rc count after changing a = {}", Rc::strong_count(&b));
    println!("a rc count after changing a = {}", Rc::strong_count(&a));

    // Uncomment the next line to see that we have a cycle;
    // it will overflow the stack
    // println!("a next item = {:?}", a.tail());
}
```
如果将代码最后一行的注释去掉，再运行，会出现一个打印的死循环。原因是我们使用了`*link.borrow_mut() = Rc::clone(&b);`，人为制造了一个死循环：<br>
{% asset_img trpl15-02.svg trpl15-02 %}

如何防止这类的泄漏？Rust提供了一个`Weak<T>`特性：
```Rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());

    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
}
```
这样不会出现循环引用，而是可以正常打印出以下内容：
```Shell
leaf parent = None
leaf parent = Some(Node { value: 5, parent: RefCell { value: (Weak) }, children: RefCell { value: [Node { value: 3, parent: RefCell { value: (Weak) }, children: RefCell { value: [] } }] } })
```
我们可以用weak_count和strong_count来观察其中的引用过程：
```Rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );

    {
        let branch = Rc::new(Node {
            value: 5,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![Rc::clone(&leaf)]),
        });

        *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

        println!(
            "branch strong = {}, weak = {}",
            Rc::strong_count(&branch),
            Rc::weak_count(&branch),
        );

        println!(
            "leaf strong = {}, weak = {}",
            Rc::strong_count(&leaf),
            Rc::weak_count(&leaf),
        );
    }

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );
}
```
打印结果是：
```Shell
leaf strong = 1, weak = 0
branch strong = 1, weak = 1
leaf strong = 2, weak = 0
leaf parent = None
leaf strong = 1, weak = 0
```