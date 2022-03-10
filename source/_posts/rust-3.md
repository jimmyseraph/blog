---
title: Rust学习笔记（3）
author: 六道先生
date: 2022-03-08 16:07:30
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第三天开始了，主要学习Rust基础语法和数据类型，加油。
---
# 变量和可变属性
## 变量定义
```Rust
let x = 5;
```
> 用`let`关键字定义变量，rust这一点和python、js很像，是弱数据类型的，通过赋值来推测变量类型。<br>

需要注意的是，这样直接定义的变量，不能再次赋值，比如：
```Rust
let x = 5;
println!("The value of x is: {}", x);
x = 6;
println!("The value of x is: {}", x);
```
就会直接报错，会提示不能第二次给不可变的变量赋值（cannot assign twice to immutable variable）。<br>
除非写成这样：
```Rust
let mut x = 5;
println!("The value of x is: {}", x);
x = 6;
println!("The value of x is: {}", x);
```
这样倒是没问题，因为mut关键字指定了x是可变的。Rust里面，默认居然是不可变的，变量还要主动指定mut，才是可变的！WTF！
## 常量
下面是常量的定义方式：
```Rust
const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
```
> 关键字const定义常量，常量必须指定数据类型，比如u32，很奇怪的是，前面介绍的`let x = 5`，同样是赋值一次后不可变，本身也有常量的意思了，这里还提供了const关键字作为常量定义，让我有点感觉多余。虽然根据官方的解释，这两者有区别：<br>
> 1. 不能使用mut关键字来指const常量可变；
> 2. 可定义的范围（scope）不同，const可以在任何位置定义，甚至是全局环境；
> 3. 可以给一个const常量赋值一个常量表达式（如上面的例子）

## 阴影
这个概念不好理解，其实就是定义和前面同名的变量，那么后面的会覆盖住前面的变量。看个例子就明白了：
```Rust
fn main() {
    let x = 5;

    let x = x + 1;

    {
        let x = x * 2;
        println!("The value of x in the inner scope is: {}", x);
    }

    println!("The value of x is: {}", x);
}
```
执行结果：
```Shell
$ cargo run
   Compiling variables v0.1.0 (file:///projects/variables)
    Finished dev [unoptimized + debuginfo] target(s) in 0.31s
     Running `target/debug/variables`
The value of x in the inner scope is: 12
The value of x is: 6
```
# 数据类型
Rust语言规定每一个值都要有明确的数据类型，虽然在变量定义时，是弱数据类型的（不需要在定义变量时说明变量的数据类型），但是变量的数据类型必须可以被推测，如果在赋值时不能被推测，那么就强制要求定义时必须要指定类型。比如下面这个例子：
```Rust
let guess: u32 = "42".parse().expect("Not a number!");
```
如果没有指定guess为u32类型，就会出现：
```Shell
$ cargo build
   Compiling no_type_annotations v0.1.0 (file:///projects/no_type_annotations)
error[E0282]: type annotations needed
 --> src/main.rs:2:9
  |
2 |     let guess = "42".parse().expect("Not a number!");
  |         ^^^^^ consider giving `guess` a type

For more information about this error, try `rustc --explain E0282`.
error: could not compile `no_type_annotations` due to previous error
```
> 显然是parse方法无法进行类型转换的推测，所以报错了。必须要给guess一个明确的类型，parse方法才知道要转成什么类型。

## 简单标量类型
Rust内置4种标量类型——整型，浮点型，布尔型，字符型。
### 整型
类型有6种：
| Length | Signed | Unsigned |
| :----: | :----:| :----: |
| 8-bit | i8 | u8 |
| 16-bit | i16 | u16 |
| 32-bit | i32 | u32 |
| 64-bit | i64 | u64 |
| 128-bit | i128 | u128 |
| arch | isize | usize |

整型的字面表达（literal）是这样的：
| 字面类型 | 例子 |
| :-: | :-: |
| 十进制 | 98_222 |
| 十六进制 | 0xff |
| 八进制 | 0o77 |
| 二进制 | 0b1111_0000 |
| 字节 (u8) | b'A' |

### 浮点类型
Rust的两个浮点类型是f32和f64，默认一个浮点数是f64的：
```Rust
fn main() {
    let x = 2.0; // f64

    let y: f32 = 3.0; // f32
}
```
### 布尔类型
很简单，就是`bool`，值就是`true`和`false`，在内存中，只占一个字节:
```Rust
fn main() {
    let t = true;

    let f: bool = false; // with explicit type annotation
}
```
### 字符类型
和java完全一样，用单引号表示一个unicode字符，在内存中占4个字节：
```Rust
fn main() {
    let c = 'z';
    let z = 'ℤ';
    let heart_eyed_cat = '😻';
}
```
## 复合类型
### Tuple类型
tuple就是简单的把多个值聚合在一个里面，类似这样：
```Rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
}
```
既能打包在一起，也可以单独取出来，看这个例子：
```Rust
fn main() {
    let tup = (500, 6.4, 1);

    let (x, y, z) = tup;

    println!("The value of y is: {}", y);
}
```
也可以通过`.`符号，直接访问tuple里面的元素：
```Rust
fn main() {
    let x: (i32, f64, u8) = (500, 6.4, 1);

    let five_hundred = x.0;

    let six_point_four = x.1;

    let one = x.2;
}
```
### 数组类型
数组和其他语言很像，没什么差别：
```Rust
fn main() {
    let a = [1, 2, 3, 4, 5];
}
```
数组的类型定义：
```Rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
```
下面是一种比较特别的初始化数组的方式：
```Rust
let a = [3; 5];
```
> 意思是数组5个元素，都是3，也就是说，第一个值是数组的初始化值，第二个值是重复多少遍。倒是很方便的表达式。

访问数组元素，也很简单：
```Rust
fn main() {
    let a = [1, 2, 3, 4, 5];

    let first = a[0];
    let second = a[1];
}
```
# 函数
直接看代码：
```Rust
fn plus(x: i32, y: i32) -> i32 {
    x + y
}
```
> 函数定义很简单，fn是关键字，plus是函数名，x和y是入参，注意入参要有类型，`->`这个箭头用于说明返回类型。
比较奇特的是，Rust默认把最后的一个表达式作为函数的返回，而不需要return关键字。当然，也可以使用return来指定返回。
这里有个重要的细节，默认是最后一个表达式，而不是语句，也就是说，结尾的返回值不要加分号，加了分号，就认为是语句而非表达式了！

看下面这个错误的例子：
```Rust
fn plus(x: i32, y: i32) -> i32 {
    x + y;
}
```
> 这段代码将会编译报错，会提示类型不匹配，希望返回的是i32类型，而实际因为`x + y;`是语句，所以没有找到最后的表达式，将会返回一个空的tuple——`()`，自然就是类型不匹配了。

# 注释
Rust使用双斜杠`//`作为注释符号来注释后面的一行，和很多语言一样。Rust还有一种特殊的注释方式，作为发布时的自动文档，后面再学了。

# 控制流
## if语句
`if`-`elseif`-`else`的语法没什么特别，跟大部分语言类似：
```Rust
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
if还可以用在赋值语句中：
```Rust
fn main() {
    let condition = true;
    let number = if condition { 5 } else { 6 };

    println!("The value of number is: {}", number);
}
```
> 这应该就是类似java等类似语言中的三元运算符`? :`吧。

## loop循环
类似shell中的loop循环，也是无条件的循环，单单用一个loop来做循环，那就是无限循环了，一般会搭配上`break`和`continue`关键字，来控制退出条件。另外也可以类似一些早期的编程语言，可以用标签来标识跳出点，直接看代码：
```Rust
fn main() {
    let mut count = 0;
    'counting_up: loop {
        println!("count = {}", count);
        let mut remaining = 10;

        loop {
            println!("remaining = {}", remaining);
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
    println!("End count = {}", count);
}
```
> 其中的`'counting_up`就是一个标签，break可以指定跳出标签所标记的loop。这段代码的输出：
```Shell
$ cargo run
   Compiling loops v0.1.0 (file:///projects/loops)
    Finished dev [unoptimized + debuginfo] target(s) in 0.58s
     Running `target/debug/loops`
count = 0
remaining = 10
remaining = 9
count = 1
remaining = 10
remaining = 9
count = 2
remaining = 10
End count = 2
```
loop也可用于赋值语句上，比如：
```Rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    println!("The result is {}", result);
}
```
> break后面的表达式，就是loop跳出后的返回值，这个例子当然就是返回了20。

## 条件循环while
用法没什么特别，跟其他语言一样：
```Rust
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{}!", number);

        number -= 1;
    }

    println!("LIFTOFF!!!");
}
```
## for循环
可以用于集合的遍历：
```Rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a {
        println!("the value is: {}", element);
    }
}
```
也有类似java那种`for(int i=0; i<10; i++)`这类的计数式的循环：
```Rust
fn main() {
    for number in (1..4).rev() {
        println!("{}!", number);
    }
    println!("LIFTOFF!!!");
}
```
> 这里的rev表示倒过来计数，也就是从4到1.