---
title: Rust学习笔记（13）
author: 六道先生
date: 2022-03-27 19:37:24
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第十三天开始了，主要学习Rust中的函数式语法特性——迭代器和闭包。
---
# 闭包
Rust的闭包，是指匿名函数，可以当作一个参数来传递，或者当作一个变量来保存。这个概念跟js中的闭包概念是一样的。<br>
要理解闭包，需要举个例子来说明。<br>
假设我们需要模拟一个算法，来给用户生成一个训练计划。先写一个模拟算法，这是一个代价比较大的算法，会花不少时间来运算。这里消耗时间我们用sleep来模拟：
```Rust
use std::thread;
use std::time::Duration;

fn simulated_expensive_calculation(intensity: u32) -> u32 {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    intensity
}
```
下面就是main函数：
```Rust
fn main() {
    let simulated_user_specified_value = 10;
    let simulated_random_number = 7;

    generate_workout(simulated_user_specified_value, simulated_random_number);
}
```
> `simulated_user_specified_value`模拟用户的输入，用户指定一个激烈系数，`simulated_random_number`是系统随机数，用于生成健身计划时候的一个运算参数。我们实现一下generate_workout：
```Rust
fn generate_workout(intensity: u32, random_number: u32) {
    if intensity < 25 {
        println!(
            "Today, do {} pushups!",
            simulated_expensive_calculation(intensity)
        );
        println!(
            "Next, do {} situps!",
            simulated_expensive_calculation(intensity)
        );
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                simulated_expensive_calculation(intensity)
            );
        }
    }
}
```
# 使用提取函数来重构
代码里面，`if`的第一个块中有两段代码调用了`simulated_expensive_calculation`，现在我们把这一段提取出来：
```Rust
fn generate_workout(intensity: u32, random_number: u32) {
    let expensive_result = simulated_expensive_calculation(intensity);

    if intensity < 25 {
        println!("Today, do {} pushups!", expensive_result);
        println!("Next, do {} situps!", expensive_result);
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!("Today, run for {} minutes!", expensive_result);
        }
    }
}
```
这样确实做到了只有一次调用，但是这样提取出来，导致if的两个分支都会被这个执行的消耗给block住，这并不是一个很好的选择。

# 使用闭包来重构
现在我们使用闭包来重构，我们写一个闭包：
```Rust
    let expensive_closure = |num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };
```
> `|num|`是`expensive_closure`这个闭包的参数，如果需要多个参数，可以用逗号分开：`|param1, param2|`。

这样我们的`generate_workout`可以这样写：
```Rust
fn generate_workout(intensity: u32, random_number: u32) {
    let expensive_closure = |num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };

    if intensity < 25 {
        println!("Today, do {} pushups!", expensive_closure(intensity));
        println!("Next, do {} situps!", expensive_closure(intensity));
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                expensive_closure(intensity)
            );
        }
    }
}
```

# 闭包类型推断和说明
闭包的参数和返回值并不需要指定类型，这一点和函数不同。因为函数是提供给其他人使用的，所以需要有明确的显式的类型定义，这样其他人才知道需要传什么参数，以及返回什么样的值。但是闭包本身是保存为一个变量使用的，并不需要暴露给外面，所以这个意义上来说，就不用刻意显示的指定参数类型和返回值类型了。因为闭包一般都比较短小和简单，所以编译器也可以很容易来推断参数以及返回值的类型。刻意增加类型说明，反而让代码显得冗余了。当然，如果要增加可读性，也是可以给闭包增加参数和返回值类型申明的：
```Rust
    let expensive_closure = |num: u32| -> u32 {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };
```
我们看一下闭包的各种写法，对比一下第一行的函数：
```Rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```
我们虽然可以省略闭包参数的类型，但是一旦编译器推断出类型后，我们是不可以乱用的，比如下面这个用法就会编译报错：
```Rust
    let example_closure = |x| x;

    let s = example_closure(String::from("hello"));
    let n = example_closure(5);
```
> 第一次使用的时候，传入闭包的参数是String类型，所以编译器会推断这个闭包的x是String类型的。那么当我们传入数字5时，就会报错，类型不匹配：
```Shell
$ cargo run
   Compiling closure-example v0.1.0 (file:///projects/closure-example)
error[E0308]: mismatched types
 --> src/main.rs:5:29
  |
5 |     let n = example_closure(5);
  |                             ^- help: try using a conversion method: `.to_string()`
  |                             |
  |                             expected struct `String`, found integer
```

# 使用特征Fn来将闭包作为泛型参数使用
Fn这个特征（trait）是标准库里面提供的，所有的闭包必须至少实现`Fn`、`FnMut`或者`FnOnce`这三个trait之一。这些trait让闭包可以作为一个泛型参数进行定义，这个做法有点类似TS中定义了一个Interface，其中一个变量是闭包函数。Rust中这样来写：
```Rust
struct Cacher<T>
where
    T: Fn(u32) -> u32,
{
    calculation: T,
    value: Option<u32>,
}
```
> 这里的calculation就是定义为一个闭包。<br>

实现方法类似下面这么写：
```Rust
impl<T> Cacher<T>
where
    T: Fn(u32) -> u32,
{
    fn new(calculation: T) -> Cacher<T> {
        Cacher {
            calculation,
            value: None,
        }
    }

    fn value(&mut self, arg: u32) -> u32 {
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.calculation)(arg);
                self.value = Some(v);
                v
            }
        }
    }
}
```
我们来使用一下这个Cacher，把之前的`generate_workout`函数改一下：
```Rust
fn generate_workout(intensity: u32, random_number: u32) {
    let mut expensive_result = Cacher::new(|num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    });

    if intensity < 25 {
        println!("Today, do {} pushups!", expensive_result.value(intensity));
        println!("Next, do {} situps!", expensive_result.value(intensity));
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                expensive_result.value(intensity)
            );
        }
    }
}
```
函数和闭包还有一个很大的区别，那就是闭包内是可以使用当前生命周期中的变量，比如下面这样的：
```Rust
fn main() {
    let x = 4;

    let equal_to_x = |z| z == x;

    let y = 4;

    assert!(equal_to_x(y));
}
```
> x虽然不是`equal_to_x`的参数，但是依然可以在闭包中访问，而函数是不可以这样访问的。

注意了，闭包的使用同样有ownership的问题，默认情况下，闭包中的变量是引用，而不是“move”，不过也可以用move关键字来进行说明，这样就会存在owner问题了，看例子：
```Rust
fn main() {
    let x = vec![1, 2, 3];

    let equal_to_x = move |z| z == x;

    println!("can't use x here: {:?}", x);

    let y = vec![1, 2, 3];

    assert!(equal_to_x(y));
}
```
> 这段代码会编译出错，因为闭包中的x不再是从main中引用来的，而是move进了闭包中，这样main中定义的x就失效了，导致println打印x会报错。<br>

# 迭代器
迭代器有点类似Java里面的iterator，通常用于一组数据的遍历。迭代器是懒加载的，如果不读取，那它什么都不会做。看个迭代器用于循环的例子：
```Rust
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();

    for val in v1_iter {
        println!("Got: {}", val);
    }
```
所有的迭代器都实现自Iterator特性，这是标准库的一个trait，样子大概这样：
```Rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // methods with default implementations elided
}
```
其中的next方法用于获取下一个元素，如果没有下一个，则返回None：
```Rust
    #[test]
    fn iterator_demonstration() {
        let v1 = vec![1, 2, 3];

        let mut v1_iter = v1.iter();

        assert_eq!(v1_iter.next(), Some(&1));
        assert_eq!(v1_iter.next(), Some(&2));
        assert_eq!(v1_iter.next(), Some(&3));
        assert_eq!(v1_iter.next(), None);
    }
```
我们把调用next的方法，都称为迭代消费适配器（consuming adaptors），比如像sum方法：
```Rust
    #[test]
    fn iterator_sum() {
        let v1 = vec![1, 2, 3];

        let v1_iter = v1.iter();

        let total: i32 = v1_iter.sum();

        assert_eq!(total, 6);
    }
```
> 注意，这里调用了sum之后，v1_iter的owner就是sum了，后面不可以再使用。

还有map方法，可以生成一个新的迭代器，这种方法，称为迭代适配器：
```Rust
    let v1: Vec<i32> = vec![1, 2, 3];

    let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();

    assert_eq!(v2, vec![2, 3, 4]);
```
再来看一个叫filter的迭代适配器的用法，这filter其实跟java中的stream流里面filter方法一样：
```Rust
#[derive(PartialEq, Debug)]
struct Shoe {
    size: u32,
    style: String,
}

fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn filters_by_size() {
        let shoes = vec![
            Shoe {
                size: 10,
                style: String::from("sneaker"),
            },
            Shoe {
                size: 13,
                style: String::from("sandal"),
            },
            Shoe {
                size: 10,
                style: String::from("boot"),
            },
        ];

        let in_my_size = shoes_in_size(shoes, 10);

        assert_eq!(
            in_my_size,
            vec![
                Shoe {
                    size: 10,
                    style: String::from("sneaker")
                },
                Shoe {
                    size: 10,
                    style: String::from("boot")
                },
            ]
        );
    }
}
```
# 定义自己的迭代器
假设我们自定义了一个结构体：
```Rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}
```
现在我们想让我们这个Counter也能使用迭代器特性，那就需要实现Iterator：
```Rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```
那现在我们的Counter就可以使用迭代器特性了：
```Rust
    #[test]
    fn calling_next_directly() {
        let mut counter = Counter::new();

        assert_eq!(counter.next(), Some(1));
        assert_eq!(counter.next(), Some(2));
        assert_eq!(counter.next(), Some(3));
        assert_eq!(counter.next(), Some(4));
        assert_eq!(counter.next(), Some(5));
        assert_eq!(counter.next(), None);
    }
    #[test]
    fn using_other_iterator_trait_methods() {
        let sum: u32 = Counter::new()
            .zip(Counter::new().skip(1))
            .map(|(a, b)| a * b)
            .filter(|x| x % 3 == 0)
            .sum();
        assert_eq!(18, sum);
    }
```
> zip方法拼接出了4对值：（1，1），（2，3），（3，4），（5，None），实际第4对值（5，None）不会真正出现，因为迭代遇到None就结束了。

# 改造minigrep的命令行参数
我们在12章写的minigrep中，`Config.new`传入的是一个vector，现在我们可以直接传入一个迭代器。首先修改main中的调用：
```Rust
fn main() {
    let config = Config::new(env::args()).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
}
```
然后修改`lib.rs`中的Config的new方法：
```Rust
impl Config {
    pub fn new(mut args: env::Args) -> Result<Config, &'static str> {
        args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };

        let filename = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file name"),
        };

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config {
            query,
            filename,
            case_sensitive,
        })
    }
}
```
还可以修改下search方法，让代码更简洁：
```Rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents
        .lines()
        .filter(|line| line.contains(query))
        .collect()
}
```
