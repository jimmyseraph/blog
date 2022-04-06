---
title: Rust学习笔记（19）
author: 六道先生
date: 2022-04-05 17:49:19
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第十八天开始了，主要学习Rust的一些高级特性（二）。
---
# 高级函数和闭包特性
## 函数指针
我们可以把一个函数作为参数传递：
```Rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

fn main() {
    let answer = do_twice(add_one, 5);

    println!("The answer is: {}", answer);
}
```
函数作为参数和闭包不太一样，它更像是一个类型，所以使用了`fn`作为关键字。函数指针本身实现了所有的三个traits——`(Fn, FnMut, and FnOnce)`，所以，所有把闭包作为参数的方法，也可以传函数指针来替代闭包，看map的例子：
```Rust
    let list_of_numbers = vec![1, 2, 3];
    let list_of_strings: Vec<String> =
        list_of_numbers.iter().map(|i| i.to_string()).collect();
```
可以改写成：
```Rust
    let list_of_numbers = vec![1, 2, 3];
    let list_of_strings: Vec<String> =
        list_of_numbers.iter().map(ToString::to_string).collect();
```
还有一种有用的表达式，下面的例子将初始化一个枚举类型的tuple，我们这里用`(..)`来初始化一个u32的tuple，通过map把它转化成枚举类型的：
```Rust
    enum Status {
        Value(u32),
        Stop,
    }

    let list_of_statuses: Vec<Status> = (0u32..20).map(Status::Value).collect();
```
## 返回闭包
闭包本身是trait的表现，所以不可以用于定义返回值，比如下面这样的是错误的：
```Rust
fn returns_closure() -> dyn Fn(i32) -> i32 {
    |x| x + 1
}
```
这样会报编译错误：
```Shell
$ cargo build
   Compiling functions-example v0.1.0 (file:///projects/functions-example)
error[E0746]: return type cannot have an unboxed trait object
 --> src/lib.rs:1:25
  |
1 | fn returns_closure() -> dyn Fn(i32) -> i32 {
  |                         ^^^^^^^^^^^^^^^^^^ doesn't have a size known at compile-time
  |
  = note: for information on `impl Trait`, see <https://doc.rust-lang.org/book/ch10-02-traits.html#returning-types-that-implement-traits>
help: use `impl Fn(i32) -> i32` as the return type, as all return paths are of type `[closure@src/lib.rs:2:5: 2:14]`, which implements `Fn(i32) -> i32`
  |
1 | fn returns_closure() -> impl Fn(i32) -> i32 {
  |                         ~~~~~~~~~~~~~~~~~~~
```
所以需要用包装类型：
```Rust
fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```
# 宏Macro
## 宏与函数的差别
宏，应该算是一种元编程（Metaprogramming），在C语言中，也有宏定义。宏本身和函数是有一定区别的。<br>
函数需要定义明确的参数数量和类型，而宏不需要。比如`println!`，可以放任意多个参数。宏要比函数复杂的多，因为写一个宏，相当于是写一个“写代码”的代码。另一点，就是宏必须在开头的时候定义，后面才能使用，而函数的定义可以在任何地方。

## 定义宏
我们看看如何用`macro_rules!`来定义宏。我们之前用过`vec!`宏来创建Vector：
```Rust
let v: Vec<u32> = vec![1, 2, 3];
```
看看`vec!`是怎么定义的，当然只是简化的：
```Rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```
`#[macro_export]`注解说明了这是一个宏，并且在这个crate被加载时就可以被调用。<br>
当使用`macro_rules!`来定义一个宏时，名称后面不需要加`!`号。<br>
宏的身体部分，有点类似match的语法结构，首先是模式语法`( $( $x:expr ),* )`，后面跟一个`=>`，之后就是一个语句块。如果模式被匹配了，就会执行后面的语句块。模式语法跟我们在match中使用的很不相同，因为这里不是匹配值，而是要匹配Rust语句。匹配模式的语法可以参考[《Rust参考手册》](https://doc.rust-lang.org/reference/macros-by-example.html)。<br>
首先是一对小括号，这一对括号里就是我们的匹配模式，然后是`$`，后面跟着一对括号，`$x:expr`表示需要截取出匹配里面模式的任何Rust表达式，匹配的表达式将会存放在x这个变量中。后面的`*`表示把前面的匹配模式重复0或n次。

## 程序性宏
程序性宏运行有点像函数，它会接受一些代码，然后进行处理，最后输出一些代码。<br>
程序性宏有三种类型：自定义派生、类属性、类函数。使用这种程序性宏，就类似下面这个样子，其中`#[some_attribute]`说明下面这个fn是一个程序性宏。
```Rust
use proc_macro;

#[some_attribute]
pub fn some_name(input: TokenStream) -> TokenStream {
}
```
下面我们来看看自定义派生宏是怎么使用的。<br>
首先，我们创建一个库：
```Shell
$ cargo new hello_macro --lib
```
然后我们在`lib.rs`中定义trait和关联方法：
```Rust
pub trait HelloMacro {
    fn hello_macro();
}
```
此时如果我们要使用，那就像下面这样：
```Rust
use hello_macro::HelloMacro;

struct Pancakes;

impl HelloMacro for Pancakes {
    fn hello_macro() {
        println!("Hello, Macro! My name is Pancakes!");
    }
}

fn main() {
    Pancakes::hello_macro();
}
```
然后，我们每个要使用HelloMacro这个trait的类型，都要去实现这个hello_macro函数。那我们能不能使用默认实现来减少代码呢？也不行，因为Rust没有反射机制，可以在运行时获取类型名称，实现这里的名称打印功能。所以我们使用程序性宏。<br>
我们在hello_macro项目下面新建一个lib：
```Shell
$ cargo new hello_macro_derive --lib
```
我们在`hello_macro_derive/Cargo.toml`中定义好依赖：
```Toml
[lib]
proc-macro = true

[dependencies]
syn = "1.0"
quote = "1.0"
```
然后在`hello_macro_derive/src/lib.rs`中定义hello_macro_derive函数：
```Rust
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // Construct a representation of Rust code as a syntax tree
    // that we can manipulate
    let ast = syn::parse(input).unwrap();

    // Build the trait implementation
    impl_hello_macro(&ast)
}
```
> 目前`impl_hello_macro`没有实现，所以还不能编译。<br>
`proc_macro`是Rust自带的crate，所以不用在Cargo文件中添加依赖，仅仅需要打开开关，设置为`true`，它是编译器API，可以用它允许编译器读取Rust代码进来并进行处理。<br>
`syn`是提供了parse方法，可以把字符串转换成Rust的语法树数据结构（Abstract Syntex Tree）。`quote`库则是可以把抽象语法树的数据转换成字符串。

parse函数转换之前的Pancakes类型，差不多会变成这样的数据结构：
```Rust
DeriveInput {
    // --snip--

    ident: Ident {
        ident: "Pancakes",
        span: #0 bytes(95..103)
    },
    data: Struct(
        DataStruct {
            struct_token: Struct,
            fields: Unit,
            semi_token: Some(
                Semi
            )
        }
    )
}
```
下面我们实现一下`impl_hello_macro`函数：
```Rust
fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    gen.into()
}
```
这里的`#name`是一个模版语法，将会使用name具体的值替换。<br>
然后在main中就可以这么使用了：
```Rust
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
```
类属性宏跟派生宏不太一样，派生宏只能用在结构体或者枚举类型上，而类属性宏还可以用于函数上。比如在web应用上使用的一个类属性宏route：
```Rust
#[route(GET, "/")]
fn index() {
```
而route属性在框架中的定义差不多类似这样：
```Rust
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
```
route的两个参数，attr是接收了`GET, "/"`这个属性，后面的item，则是接收了`fn index()`这个函数。<br>

类函数宏则是用法跟普通函数类似：
```Rust
let sql = sql!(SELECT * FROM posts WHERE id=1);
```
这个`sql!`就是一个类函数宏，它的定义类似这样：
```Rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
```