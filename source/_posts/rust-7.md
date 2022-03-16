---
title: Rust学习笔记（7）
author: 六道先生
date: 2022-03-14 07:37:47
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第七天开始了，主要学习Rust的Packages, Crates, 以及 Modules的概念。
---
# Packages和Crates
Crates本意是木箱，在Rust里面用于表示二进制仓库，意思就是编译后的一个可以被其他人使用的可执行的crate，或仓库crate。因为木箱嘛，就是装了东西可以搬来搬去的。<br>
Packages就是我自己创建的一个项目，它可以包含一个或多个Crates，用一个`Cargo.toml`文件进行管理，来描述那些crates该怎么构建。<br>
在Rust中，你创建的package可以有多个Crates，但不不能一个crate都没有。当使用`cargo new my-project`命令创建一个项目的时候，这就是你的一个package，其中的`src/main.rs`，就是一个和package同名的可执行crate（binary crate），在`Cargo.toml`中并没有提及，因为Rust默认`src/main.rs`就是入口，也就是所谓的这个crate的根所在。<br>
如果在src目录下的不是main.rs，而是lib.rs，那这就是一个库crate（library crate）。<br>
如果src目录下既有main.rs，又有lib.rs，那就会编译出一个可执行crate，以及一个仓库crate，这两个crates都和package同名。而且，一个package里面也可以有多个可执行crates。那就是在`src/bin`下面，建多个rs文件，每一个都会编译为和文件名同名的可执行crates。
# 模块（Modules）
模块用于组织我们写的代码，以及控制访问权限。现在举个例子，假设现在有一家餐馆，餐馆分成前台和后台，前台就是客人待着的地方，服务员给客人安排座位，以及点菜。后台就是厨师做菜，洗盘子，管理人员做餐馆管理事项的地方。我们安装前台来组织模块。<br>
首先我们创建仓库crate：
```Shell
cargo new --lib restaurant
```
然后在`src/lib.rs`中写如下代码：
```Rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```
这组织方式有点像目录，看一下结构：
```Tree
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```
# 模块的访问
模块不仅仅是组织代码结构，还有访问管理的作用，比如下面这段代码：
```Rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```
> 这段代码在eat_at_restaurant中，通过绝对路径和相对路径访问了模块中的add_to_waitlist方法，首先，这说明了模块中元素的方式，绝对路径是用crate开头，用`::`进行访问，相对路径就是直接从你当前所在的模块名称开始。

但是这段代码会出现编译错误：
```Shell
$ cargo build
   Compiling restaurant v0.1.0 (file:///projects/restaurant)
error[E0603]: module `hosting` is private
 --> src/lib.rs:9:28
  |
9 |     crate::front_of_house::hosting::add_to_waitlist();
  |                            ^^^^^^^ private module
  |
note: the module `hosting` is defined here
 --> src/lib.rs:2:5
  |
2 |     mod hosting {
  |     ^^^^^^^^^^^

error[E0603]: module `hosting` is private
  --> src/lib.rs:12:21
   |
12 |     front_of_house::hosting::add_to_waitlist();
   |                     ^^^^^^^ private module
   |
note: the module `hosting` is defined here
  --> src/lib.rs:2:5
   |
2  |     mod hosting {
   |     ^^^^^^^^^^^
```
这个提示明确指出了你访问的hosting模块，是一个私有模块，不能直接访问。也就是说，除非你是在front_of_house这个模块里面来调用hosting模块，否则是不能访问的。除非hosting模块是公有的，在前面加一个`pub`关键字。包括add_to_waitlist方法，也是私有的，需要对外开放，也是需要`pub`关键字的：
```Rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```
如果在某个mod里面的一个函数，需要访问所在mod外面的函数，可以使用super作为相对路径：
```Rust
fn serve_order() {}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```
下面看两个例子，说明下在mod中定义struct和枚举，并访问：
```Rust
mod back_of_house {
    pub struct Breakfast {
        pub toast: String,
        seasonal_fruit: String,
    }

    impl Breakfast {
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast {
                toast: String::from(toast),
                seasonal_fruit: String::from("peaches"),
            }
        }
    }
}

pub fn eat_at_restaurant() {
    // Order a breakfast in the summer with Rye toast
    let mut meal = back_of_house::Breakfast::summer("Rye");
    // Change our mind about what bread we'd like
    meal.toast = String::from("Wheat");
    println!("I'd like {} toast please", meal.toast);

    // The next line won't compile if we uncomment it; we're not allowed
    // to see or modify the seasonal fruit that comes with the meal
    // meal.seasonal_fruit = String::from("blueberries");
}
```
下面是枚举类型：
```Rust
mod back_of_house {
    pub enum Appetizer {
        Soup,
        Salad,
    }
}

pub fn eat_at_restaurant() {
    let order1 = back_of_house::Appetizer::Soup;
    let order2 = back_of_house::Appetizer::Salad;
}
```
# 使用`use`来访问模块
`use`有点类似python、java里面的import，根C#的use一样，看例子：
```Rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```
这个use是绝对路径的写法，也可以使用相对路径：
```Rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use self::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```
和其他语言一样，use进来的还能改名，使用`as`关键字：
```Rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```
另外，use前面还可以用pub进行修饰，这样外部代码也可以使用这个命名，类似这样：
```Rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;
```
如果有别的文件里面引用到了这个文件，也可以直接使用hosting。<br>
如果有如下的引用：
```Rust
// --snip--
use std::cmp::Ordering;
use std::io;
// --snip--
```
可以简写成这样：
```Rust
// --snip--
use std::{cmp::Ordering, io};
//
```
还有：
```Rust
use std::io;
use std::io::Write;
```
可以简写成：
```Rust
use std::io::{self, Write};
```
还有通配符可以使用：
```Rust
use std::collections::*;
```
# 用不同的文件来组织模块
随着模块的增加，都写在一个文件里也不好管理，比如上面那个front_of_house模块，可以在src下创建一个同名的文件，叫`front_of_house.rs`，里面内容就是：
```Rust
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```
然后在src下的lib.rs或者main.rs里面可以写：
```Rust
mod front_of_house;

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```