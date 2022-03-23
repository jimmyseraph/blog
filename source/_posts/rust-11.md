---
title: Rust学习笔记（11）
author: 六道先生
date: 2022-03-22 21:30:02
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第十一天开始了，主要学习Rust的测试相关内容。
---
# 如何写测试
一个测试，一般有下面这些步骤组成：<br>
1. 准备好输入的数据和以及数据的状态；
2. 执行你要测试的代码；
3. 校验结果是否符合预期。

在Rust中，一个测试函数由一个注解——test来申明。就是在`fn`定义一个函数的前面一行，写上`#[test]`，Rust就会识别为一个测试函数。在执行`cargo test`命令的时候，就会构建测试执行器来执行所有的测试函数。<br>
当我们自己用cargo创建一个库的时候，其实就会自动创建好测试的模版，试试看：
```Shell
$ cargo new adder --lib
     Created library `adder` project
```
在`src/lib.rs`文件中，可以看到下面的模版：
```Rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        let result = 2 + 2;
        assert_eq!(result, 4);
    }
}
```
试试运行：
```Shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.57s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```
> 注意其中除了执行了`tests::it_works`外，还执行了`Doc-tests adder`，这个是对项目的API文档测试，我们目前没有任何文档测试函数，所以是0。

写一个错误的测试函数试试：
```Rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        assert_eq!(2 + 2, 4);
    }

    #[test]
    fn another() {
        panic!("Make this test fail");
    }
}
```
another函数我们强制报错，运行后会出现详细的错误信息：
```Shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.72s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 2 tests
test tests::another ... FAILED
test tests::exploration ... ok

failures:

---- tests::another stdout ----
thread 'main' panicked at 'Make this test fail', src/lib.rs:10:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::another

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```
来看个具体的例子，假设有这个struct：
```Rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}
```
然后我们可以写一个针对can_hold的测试：
```Rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };

        assert!(larger.can_hold(&smaller));
    }

    #[test]
    fn smaller_cannot_hold_larger() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };

        assert!(!smaller.can_hold(&larger));
    }
}
```
看看test提供了哪些校验的宏：<br>
+ assert
+ assert_eq
+ assert_ne
+ assert_matches
+ debug_assert
+ debug_assert_eq
+ debug_assert_ne
+ debug_assert_matches

assert宏还可以增加自定义的错误信息：
```Rust
    #[test]
    fn greeting_contains_name() {
        let result = greeting("Carol");
        assert!(
            result.contains("Carol"),
            "Greeting did not contain name, value was `{}`",
            result
        );
    }
```
还可以使用注解`#[should_panic]`来申明希望抛出异常，如果被测程序没有抛异常则认为测试不通过：
```Rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```
should_panic也可以指定抛出错误的信息：
```Rust
impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 {
            panic!(
                "Guess value must be greater than or equal to 1, got {}.",
                value
            );
        } else if value > 100 {
            panic!(
                "Guess value must be less than or equal to 100, got {}.",
                value
            );
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "Guess value must be less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```
当然，也可以不用assert宏，也可以自行做断言，然后抛出Result枚举：
```Rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() -> Result<(), String> {
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("two plus two does not equal four"))
        }
    }
}
```
# 控制测试运行
`cargo test`命令会将你的tests全部编译为二进制文件，并执行。在这个命令后面还可以加一些参数，进行运行控制。比如可以用`--test NAME`来运行指定名称的测试。<br>
需要注意的是，默认情况下，rust会并行运行所有的tests，并控制输出，而不是一个一个运行，所以test之间是没有先后顺序的。如果需要指定单个线程来运行，可以用下面的命令：
```Shell
$ cargo test -- --test-threads=1
```
这就指定了只用一个线程来运行测试，那么就是一个一个来运行了。<br>
## 测试执行的输出控制
默认情况下，如果你写的test中有输出，比如println宏，在这个test执行通过的情况下，你将不会看到你的输出，只有测试pass的信息。如果这个test失败了，才会有你调用println输出的信息。这是因为将pass的测试中的输出信息也打印到控制台，信息会打乱测试是否pass的信息，而fail的测试信息打印，则有助于你定位问题。看下面输出的例子：
```Rust
fn prints_and_returns_10(a: i32) -> i32 {
    println!("I got the value {}", a);
    10
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn this_test_will_pass() {
        let value = prints_and_returns_10(4);
        assert_eq!(10, value);
    }

    #[test]
    fn this_test_will_fail() {
        let value = prints_and_returns_10(8);
        assert_eq!(5, value);
    }
}
```
执行后在控制台pass的测试没有输出信息，只有fail的那个才有：
```Shell
$ cargo test
   Compiling silly-function v0.1.0 (file:///projects/silly-function)
    Finished test [unoptimized + debuginfo] target(s) in 0.58s
     Running unittests (target/debug/deps/silly_function-160869f38cff9166)

running 2 tests
test tests::this_test_will_fail ... FAILED
test tests::this_test_will_pass ... ok

failures:

---- tests::this_test_will_fail stdout ----
I got the value 8
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `5`,
 right: `10`', src/lib.rs:19:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::this_test_will_fail

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```
如果想都显示出来，可以加执行参数：
```Shell
$ cargo test -- --show-output
```
## 执行测试子集
可以通过名称匹配来执行多个带有关键字的测试：
```Shell
$ cargo test smoke
```
rust会根据test后面跟的这个名字，搜索所有tests，只要名字里面带有smoke的都会被执行。这给了我们一个类似标签的思路。

## 忽略执行
我们可以通过添加`#[ignore]`来忽略某些test:
```Rust
#[test]
fn it_works() {
    assert_eq!(2 + 2, 4);
}

#[test]
#[ignore]
fn expensive_test() {
    // code that takes an hour to run
}
```
这样在执行`cargo test`的时候，expensive_test就会被忽略。如果想要运行被忽略的测试，可以在命令行添加参数：
```Shell
$ cargo test -- --ignored
```
## 组织不同级别的测试
Rust中可以组织不同级别的测试，和被测struct写在一个文件里面，并用`#[cfg(test)]`申明一个mod的，称为单元测试，它可以对struct的impl方法进行测试，不管是否有pub申明，也就是说内部方法也可以访问。<br>
还有就是创建一个名为tests的目录，来写test，不需要`#[cfg(test)]`注解的。因为Rust认为tests目录中的都是test，我们称为集成测试，它不能访问内部方法哦，比如创建一个tests目录中，文件名叫`integration_test.rs`：
```Rust
use adder;

#[test]
fn it_adds_two() {
    assert_eq!(4, adder::add_two(2));
}
```
如果需要在tests目录下创建一个被很多test调用的公共模块，那需要创建一个目录，名字随意，但是这个目录下面的文件必须叫`mod.rs`，这个目录才会被tests识别为模块，并且可以通过`mod`来调用。比如创建`tests/common/mod.rs`，里面写了一个setup函数：
```Rust
pub fn setup() {
    // setup code specific to your library's tests would go here
}
```
那么在tests目录中的test就可以调用：
```Rust
use adder;

mod common;

#[test]
fn it_adds_two() {
    common::setup();
    assert_eq!(4, adder::add_two(2));
}
```
需要注意，只有main.rs的可执行项目，是不可以使用tests目录进行集成测试的，只有lib.rs的库项目才可以。所以写rust项目的时候，需要注意所有的逻辑、算法，都放在library crate中，main只做调用。