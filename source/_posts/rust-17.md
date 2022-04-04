---
title: Rust学习笔记（17）
author: 六道先生
date: 2022-04-03 12:08:44
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第十七天开始了，主要学习Rust的匹配模式。
---
# 各种模式（pattern）使用场景
模式（pattern）在Rust中是一种比较特别的语法，汇总一下Rust中使用模式（pattern）的地方，有`match`，`if let`，`while let`，`for`，`let`，函数参数，这些场所都可以看到模式语法。一个一个来看。
## `match`匹配
在`match`的分支中，就是使用的模式语法：
```Rust
match VALUE {
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
}
```
具体的例子在前面已经看过很多了，不再例举。

## `if let`表达式
`if let`后面也是跟一个模式，看个例子：
```Rust
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        println!("Using your favorite color, {}, as the background", color);
    } else if is_tuesday {
        println!("Tuesday is green day!");
    } else if let Ok(age) = age {
        if age > 30 {
            println!("Using purple as the background color");
        } else {
            println!("Using orange as the background color");
        }
    } else {
        println!("Using blue as the background color");
    }
}
```
> 注意`if let Ok(age) = age`和`if age > 30`不能合并为`if let Ok(age) = age && age > 30`，因为age这个阴影变量要在下面的花括号内，也就是新的scope中才生效。

## `while let`条件循环
这个跟`if let`类似，只是变成了循环条件：
```Rust
    let mut stack = Vec::new();

    stack.push(1);
    stack.push(2);
    stack.push(3);

    while let Some(top) = stack.pop() {
        println!("{}", top);
    }
```
## `for`循环
`for x in y`这是我们经常使用的循环模式，其中的x就是模式（pattern），看个例子：
```Rust
fn main() {
    let v = vec!['a', 'b', 'c'];

    for (index, value) in v.iter().enumerate() {
        println!("{} is at index {}", value, index);
    }
}
```
> enumerate方法将会返回一个tuple，第一个元素是序号，第二个是value，所以我们在for语句中，使用`(index, value)`来解开这个tuple。

## `let`语句
`let`后面跟的也是模式语法：
```Rust
let PATTERN = EXPRESSION;
```
比如简单的：
```Rust
let x = 5;
```
还有稍微复杂点的，比如解开tuple：
```Rust
let (x, y, z) = (1, 2, 3);
```
## 函数的参数
函数的参数也是一种模式语法，比如这样：
```Rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({}, {})", x, y);
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```

# 可反驳与不可反驳模式
模式（pattern）有两种类型，一种叫可反驳（refutable）另一种叫不可反驳（irrefutable）。比如`let x = 5`就是一种不可反驳模式，其实也可以称为确定模式，5赋值给x，是一种确定的赋值，不会存在无法匹配的情况。另一种，比如`if let Some(x) = a_value`就是可反驳，或者叫不确定模式，因为x不一定能匹配到值，有可能a_value会是None。<br>
函数参数、`let`语句、`for`循环只能使用不可反驳模式，而`if let`、`while let`则使用可反驳模式。如果在函数参数、`let`语句、`for`循环中使用可反驳模式，则会出现编译错误：
```Rust
    let Some(x) = some_option_value;
```
看错误提示：
```Shell
$ cargo run
   Compiling patterns v0.1.0 (file:///projects/patterns)
error[E0005]: refutable pattern in local binding: `None` not covered
   --> src/main.rs:3:9
    |
3   |     let Some(x) = some_option_value;
    |         ^^^^^^^ pattern `None` not covered
    |
    = note: `let` bindings require an "irrefutable pattern", like a `struct` or an `enum` with only one variant
    = note: for more information, visit https://doc.rust-lang.org/book/ch18-02-refutability.html
    = note: the matched value is of type `Option<i32>`
help: you might want to use `if let` to ignore the variant that isn't matched
    |
3   |     if let Some(x) = some_option_value { /* */ }
    |
```
如果在`if let`、`while let`中使用不可反驳模式，像这样：
```Rust
    if let x = 5 {
        println!("{}", x);
    };
```
则会出现编译告警，因为这个if其实是无效的，永远为真——不可反驳：
```Shell
$ cargo run
   Compiling patterns v0.1.0 (file:///projects/patterns)
warning: irrefutable `if let` pattern
 --> src/main.rs:2:8
  |
2 |     if let x = 5 {
  |        ^^^^^^^^^
  |
  = note: `#[warn(irrefutable_let_patterns)]` on by default
  = note: this pattern will always match, so the `if let` is useless
  = help: consider replacing the `if let` with a `let`

warning: `patterns` (bin "patterns") generated 1 warning
    Finished dev [unoptimized + debuginfo] target(s) in 0.39s
     Running `target/debug/patterns`
5
```
# 模式的语法
这里汇总一下各类的模式语法。
## 匹配常量
之前就出现的例子：
```Rust
    match x {
        1 => println!("one"),
        2 => println!("two"),
        3 => println!("three"),
        _ => println!("anything"),
    }
```

## 匹配命名变量
看例子：
```Rust
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(y) => println!("Matched, y = {:?}", y),
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {:?}", x, y);
```
## 多模式
看例子：
```Rust
    let x = 1;

    match x {
        1 | 2 => println!("one or two"),
        3 => println!("three"),
        _ => println!("anything"),
    }
```

## 范围匹配
看例子：
```Rust
    let x = 5;

    match x {
        1..=5 => println!("one through five"),
        _ => println!("something else"),
    }
```
> `1..=5`表示从1到5（5包含在内）。

下面的例子也是ok的：
```Rust
    let x = 'c';

    match x {
        'a'..='j' => println!("early ASCII letter"),
        'k'..='z' => println!("late ASCII letter"),
        _ => println!("something else"),
    }
```
> 和C/C++一样，字符类型（char）其实也是数字，也可以用范围表示，只要是连续的。

## 解构赋值
### 解构struct
看例子：
```Rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x: a, y: b } = p;
    assert_eq!(0, a);
    assert_eq!(7, b);
}
```
还可以写的更简单一些：
```Rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x, y } = p;
    assert_eq!(0, x);
    assert_eq!(7, y);
}
```
这种解构还能在match中使用：
```Rust
fn main() {
    let p = Point { x: 0, y: 7 };

    match p {
        Point { x, y: 0 } => println!("On the x axis at {}", x),
        Point { x: 0, y } => println!("On the y axis at {}", y),
        Point { x, y } => println!("On neither axis: ({}, {})", x, y),
    }
}
```
因为匹配了x为0，所以会打印：
```Shell
On the y axis at 7
```
### 解构枚举类型
看例子：
```Rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let msg = Message::ChangeColor(0, 160, 255);

    match msg {
        Message::Quit => {
            println!("The Quit variant has no data to destructure.")
        }
        Message::Move { x, y } => {
            println!(
                "Move in the x direction {} and in the y direction {}",
                x, y
            );
        }
        Message::Write(text) => println!("Text message: {}", text),
        Message::ChangeColor(r, g, b) => println!(
            "Change the color to red {}, green {}, and blue {}",
            r, g, b
        ),
    }
}
```
### 解构struct和枚举的嵌套类型
看例子：
```Rust
enum Color {
    Rgb(i32, i32, i32),
    Hsv(i32, i32, i32),
}

enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(Color),
}

fn main() {
    let msg = Message::ChangeColor(Color::Hsv(0, 160, 255));

    match msg {
        Message::ChangeColor(Color::Rgb(r, g, b)) => println!(
            "Change the color to red {}, green {}, and blue {}",
            r, g, b
        ),
        Message::ChangeColor(Color::Hsv(h, s, v)) => println!(
            "Change the color to hue {}, saturation {}, and value {}",
            h, s, v
        ),
        _ => (),
    }
}
```
### 解构struct和tuple
看例子：
```Rust
fn main() {
    struct Point {
        x: i32,
        y: i32,
    }

    let ((feet, inches), Point { x, y }) = ((3, 10), Point { x: 3, y: -10 });
}
```
### 忽略对应位置的值
如果解构的一组值中有不需要的，可以用`_`忽略该值：
```Rust
fn foo(_: i32, y: i32) {
    println!("This code only uses the y parameter: {}", y);
}

fn main() {
    foo(3, 4);
}
```
也可以忽略一部分值：
```Rust
fn main() {
    let mut setting_value = Some(5);
    let new_setting_value = Some(10);

    match (setting_value, new_setting_value) {
        (Some(_), Some(_)) => {
            println!("Can't overwrite an existing customized value");
        }
        _ => {
            setting_value = new_setting_value;
        }
    }

    println!("setting is {:?}", setting_value);
}
```
还有这样的：
```Rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (first, _, third, _, fifth) => {
            println!("Some numbers: {}, {}, {}", first, third, fifth)
        }
    }
}
```
如果有个变量定义了之后不使用，但又想避免编译时候的警告，可以在变量名前用`_`：
```Rust
    let _x = 5;
```
### 忽略剩余的值
看例子：
```Rust
fn main() {
    struct Point {
        x: i32,
        y: i32,
        z: i32,
    }

    let origin = Point { x: 0, y: 0, z: 0 };

    match origin {
        Point { x, .. } => println!("x is {}", x),
    }
}
```
还可以省略中间一部分值：
```Rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (first, .., last) => {
            println!("Some numbers: {}, {}", first, last);
        }
    }
}
```
但是这种是<b style="color: red">错误</b>的：
```Rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (.., second, ..) => {
            println!("Some numbers: {}", second)
        },
    }
}
```
### 匹配的附加条件
可以在match的匹配模式后面，增加if来进行附加条件匹配：
```Rust
fn main() {
    let num = Some(4);

    match num {
        Some(x) if x < 5 => println!("less than five: {}", x),
        Some(x) => println!("{}", x),
        None => (),
    }
}
```
### 使用`@`进行模式绑定操作
看例子：
```Rust
fn main() {
    enum Message {
        Hello { id: i32 },
    }

    let msg = Message::Hello { id: 5 };

    match msg {
        Message::Hello {
            id: id_variable @ 3..=7,
        } => println!("Found an id in range: {}", id_variable),
        Message::Hello { id: 10..=12 } => {
            println!("Found an id in another range")
        }
        Message::Hello { id } => println!("Found some other id: {}", id),
    }
}
```
> `id: id_variable @ 3..=7`这个模式操作不仅判断id的值需要在3～7之间，并把满足条件的值赋值给id_variable变量。