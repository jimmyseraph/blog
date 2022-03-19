---
title: Rust学习笔记（9）
author: 六道先生
date: 2022-03-18 11:00:43
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第十天开始了，主要学习Rust的泛型、特征和生命周期。
---
# 泛型（generic type）
泛型主要用于解决一些重复性的定义，假设我们需要写一个函数能从一个vector中取出最大值。但是有i32类型以及char类型的vector，看下面这个例子：
```Rust
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn largest_char(list: &[char]) -> char {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest_i32(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest_char(&char_list);
    println!("The largest char is {}", result);
}
```
为了能处理i32和char两种类型的vector，所以写了两个不同的largest_char方法。但是从代码来看，这两个方法其实实现都一样，这时候我们可以使用泛型来定义方法，从而减少了大量的重复代码：
```Rust
use std::cmp::PartialOrd;
fn largest<T: PartialOrd+Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```
> 这里的泛型写的比较特殊，因为必须要能让`item > largest`这个比较能进行运算的T才不会报错，所以对泛型T有特别的要求。

也可以用泛型来定义struct：
```Rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```
如果要配合实现方法：
```Rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };

    println!("p.x = {}", p.x());
}
```
# 特征
特征（trait）告诉Rust编译器一种能被其他类型共享的功能。这个trait的概念，有点类似其他语言中的接口（interface）概念，当然也不是完全一样。

## 定义特征
假设我们有一个库中定义了两个struct，分别是Article和Tweet，然后我们需要给Article和Tweet实例做简介，现在定义一个特征trait：
```Rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```
> 在这里只有定义，跟接口一样。

trait中方法的实现，需要在针对每一个struct：
```Rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```
在main中使用：
```Rust
fn main() {
    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    println!("1 new tweet: {}", tweet.summarize());
}
```
## 实现trait的规则
如果要实现一个trait，只有一个原则，那就是trait本身或者要实现trait的类型，这两者至少有一个是本地的。<br>
比如我们想给Tweet实现一个标准库中的特征Display，这是可以的，因为Tweet是我们本地写的；我们也可以给`Vec<T>`实现Summary这个特征，因为Summary这个trait是我们本地定义的。

## 默认trait实现
前面我们定义了一个空的summarize在trait中，具体的实现写在了每个struct中。现在我们写一个默认的trait实现：
```Rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```
那么这时候，如果我们没有为Article定义summarize，那么在使用Article的对象调用summarize的时候，就会使用这个默认方法。

## 将`impl trait`作为参数传递
trait还可以作为参数来传递，看下面的例子：
```Rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```
这个notify函数，参数item是`&impl Summary`这个trait的引用，这样在函数中，item就可以调用Summary中的所有方法。notify被实际调用时，传入的item就是实现了trait中方法的那些struct的实例。如果传入的struct实例是没有实现trait的，那就会编译失败。<br>
上面这个例子，Rust还有一种长格式的语法糖的写法：
```Rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```
我们把这种写法称为特征边界（trait bound）。<br>
至于是采用`&impl`这种写法，还是语法糖这种写法，要看具体情况了。`&impl`的语法更直接清晰，可读性强，trait bound则可以表达更复杂的情况。看下面这个例子：
```Rust
pub fn notify(item1: &impl Summary, item2: &impl Summary)
```
这时候，用`&impl`就显得有点冗余了，改用语法糖则更简便一些：
```Rust
pub fn notify<T: Summary>(item1: &T, item2: &T)
```
Rust还可以指定多重trait作为参数：
```Rust
pub fn notify(item: &(impl Summary + Display))
```
或者：
```Rust
pub fn notify<T: Summary + Display>(item: &T)
```
这个`+`号表示传入的实例同时要实现Summary和Display这两个trait。<br>
如果有trait bound表达式比较长，可读性变得比较差的时候，还可以用`where`来增加可读性，像下面的这个例子：
```Rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {
```
T和U都是多个trait，这样太长，不容易看明白了，可以改成下面这种的：
```Rust
fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{
```
用了where之后，T和U一眼就看清楚了。

## 将`impl trait`作为作为返回值
看例子：
```Rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    }
}
```
它所返回的，就是任何一个实现了Summary这个trait的struct实例。但是注意了，这个返回必须是一个明确的类型，如果返回不同的类型，用`impl Summary`这个语法是不行的，看下面的错误例子：
```Rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        NewsArticle {
            headline: String::from(
                "Penguins win the Stanley Cup Championship!",
            ),
            location: String::from("Pittsburgh, PA, USA"),
            author: String::from("Iceburgh"),
            content: String::from(
                "The Pittsburgh Penguins once again are the best \
                 hockey team in the NHL.",
            ),
        }
    } else {
        Tweet {
            username: String::from("horse_ebooks"),
            content: String::from(
                "of course, as you probably already know, people",
            ),
            reply: false,
            retweet: false,
        }
    }
}
```
> 这段函数可能会返回NewsArticle也可能返回Tweet，那么编译器就无法确定返回的类型，这会导致编译出错！

还有一种概念，叫覆盖式实现（blanket implementation），比如系统库中的ToString特征是这样的：
```Rust
impl<T: Display> ToString for T {
    // --snip--
}
```
所以只要是实现了Display这个trait的类型，都可以直接调用ToString中定义的to_string方法，比如像Integer类型：
```Rust
let s = 3.to_string();
```

# 关于生命周期的问题
Rust中对于borrow，也就是引用的变量，有明确的生命周期的管理要求，比如这样的：
```Rust
    {
        let r;                // ---------+-- 'a
                              //          |
        {                     //          |
            let x = 5;        // -+-- 'b  |
            r = &x;           //  |       |
        }                     // -+       |
                              //          |
        println!("r: {}", r); //          |
    }    
```
> r开始没有赋值，在`'b`的生命周期中，定义了x，并借用给了r，但是在b生命周期结束后，x也就被清理了，那么r借用的内容也就没有了，所以后面的println来打印r，肯定会编译报错。

上面这个例子还比较容易理解，但是看下面的例子：
```Rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}

fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
这段代码编译会报错：
```Shell
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0106]: missing lifetime specifier
 --> src/main.rs:9:33
  |
9 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
9 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++

```
这个错误，是因为返回的是个引用，所以Rust需要明确知道是谁的引用，以确定是否这个引用的对象在有效的生命周期中。而这个引用，在代码中可能是x的引用，也可能是y的引用，所以这个不确定性，导致了编译的失败，错误提示说明了这一点，需要你明确生命周期。<br>
如何显式的申明生命周期呢？看下面的代码：
```Rust
&i32        // a reference
&'a i32     // a reference with an explicit lifetime
&'a mut i32 // a mutable reference with an explicit lifetime
```
我们把上面的longest函数改成带有显式生命周期的就可以了：
```Rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
带有`'a`的表示同一生命周期，这样就不会出现生命周期不确定了。<br>
结构体也可以带显式生命周期，比如这样写：
```Rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}
```
还有一种叫显式静态生命周期的定义方式，用于指定这个对象在整个程序的生命周期中都有效，有点类似全局变量：
```Rust
let s: &'static str = "I have a static lifetime.";
```
最后来看个例子，把泛型、特征边界、生命周期三个概念混一起：
```Rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```