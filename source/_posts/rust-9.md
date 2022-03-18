---
title: Rust学习笔记（9）
author: 六道先生
date: 2022-03-16 18:53:55
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第九天开始了，主要学习Rust的异常处理方式。
---
# 异常处理
Rust把错误分为两类：可恢复错误和不可恢复错误。可恢复错误，比如文件找不到，这类错误需要回报给用户，并有机会重试。不可恢复错误，通常是bug，比如数组元素访问越界。这有点像Java中的Exception和Error<br>
Rust使用`Result<T, E>`来处理可恢复错误，用`panic`处理不可恢复错误。
## 关于Panic
当程序出现一些无法处理的错误，可以使用`panic!`宏。当panic被执行时，程序将会打印错误信息，并开始清理stack空间，完成清理后退出。清理stack内存是一件比较大的工作，所以如果想自己的程序快速退出，编译后的二进制执行文件足够小，可以选择将这个退出stack的过程，改为终止（abort）。也就是直接结束程序，而不进行内存清理，将stack的清理工作交给操作系统。切换about，需要在Cargo.toml文件中增加如下内容：
```Toml
[profile.release]
panic = 'abort'
```
来试试panic：
```Rust
fn main() {
    panic!("crash and burn");
}
```
看输出：
```Shell
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.25s
     Running `target/debug/panic`
thread 'main' panicked at 'crash and burn', src/main.rs:2:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
看提示，可以使用`RUST_BACKTRACE=1`环境变量来显示backtrace，所谓的backtrace其实就是错误栈，类似java抛出错误时候的调用链stacktrace。看加了环境变量之后的输出：
```Shell
$ RUST_BACKTRACE=1 cargo run
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', src/main.rs:4:5
stack backtrace:
   0: rust_begin_unwind
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/std/src/panicking.rs:483
   1: core::panicking::panic_fmt
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/panicking.rs:85
   2: core::panicking::panic_bounds_check
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/panicking.rs:62
   3: <usize as core::slice::index::SliceIndex<[T]>>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/slice/index.rs:255
   4: core::slice::index::<impl core::ops::index::Index<I> for [T]>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/slice/index.rs:15
   5: <alloc::vec::Vec<T> as core::ops::index::Index<I>>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/alloc/src/vec.rs:1982
   6: panic::main
             at ./src/main.rs:4
   7: core::ops::function::FnOnce::call_once
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/ops/function.rs:227
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```
最后一行还提示了一些细节被省略了，可以用`RUST_BACKTRACE=full`环境变量来显示全部细节。
## 关于Result
Result枚举专用于处理可恢复错误，就等价于Java中的Exception，Result像下面这样：
```Rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```
那么T和E是什么呢？看一个抛出Result的错误，就能理解了：
```Rust
use std::fs::File;

fn main() {
    let f: u32 = File::open("hello.txt");
}
```
这里强行给f指定了一个u32的类型，但open返回的显然不是u32类型，那么在编译中就会出现以下错误：
```Shell
$ cargo run
   Compiling error-handling v0.1.0 (file:///projects/error-handling)
error[E0308]: mismatched types
 --> src/main.rs:4:18
  |
4 |     let f: u32 = File::open("hello.txt");
  |            ---   ^^^^^^^^^^^^^^^^^^^^^^^ expected `u32`, found enum `Result`
  |            |
  |            expected due to this
  |
  = note: expected type `u32`
             found enum `Result<File, std::io::Error>`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `error-handling` due to previous error
```
> 错误很明显指出了open返回的类型是`Result<File, std::io::Error>`，之前看到的T和E这两个泛型，在这里被明确成了File和Error类型，其实也就是说，当文件打开成功，返回的就是OK(File)，而失败的话，返回的就是ERR(std::io::Error)。

看看如何处理Result：
```Rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => panic!("Problem opening the file: {:?}", error),
    };
}
```
> 如果文件存在，则会返回file，这是一个操作文件的句柄。如果不存在，则会抛出一个panic。

如果我们需要做到文件不存在时，去创建一个文件，我们可以这样操作：
```Rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => {
                panic!("Problem opening the file: {:?}", other_error)
            }
        },
    };
}
```
三层match，哈哈，有点难看了，而且可读性差了点，可以改改：
```Rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound {
            File::create("hello.txt").unwrap_or_else(|error| {
                panic!("Problem creating the file: {:?}", error);
            })
        } else {
            panic!("Problem opening the file: {:?}", error);
        }
    });
}
```
这里用了标准库中Result枚举类型的unwrap_or_else方法，这个方法表示如果Result枚举是OK，那就把OK中的对象从枚举中取出来返回，否则就执行里面的表达式。<br>
除了unwrap_or_else以外，还有unwrap也可以处理，当然，因为没有else，所以相对于unwrap_or_else，unwrap一旦发现Result枚举是ERR，那就直接抛出panic：
```Rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").unwrap();
}
```
如果文件不存在，那就会报：
```Shell
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Error {
repr: Os { code: 2, message: "No such file or directory" } }',
src/libcore/result.rs:906:4
```
expect也和unwrap方法一样，只是expect可以自己定义错误信息：
```Rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
}
```
这种用法感觉比unwrap更好一些。<br>
处理可能发生的错误，还可以向外传递，就类似java中可以try-catch来捕获异常，也可以直接throw异常，看下面的例子：
```Rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");

    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut s = String::new();

    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}
```
Rust还可以用`?`操作符来进行错误传递，把上面的例子改成这样：
```Rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```
这个`?`操作符非常方便，它实际上是调用了一个`from`的内置方法，会讲ERR转换为你定义的返回类型。还可以写的更简便：
```Rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();

    File::open("hello.txt")?.read_to_string(&mut s)?;

    Ok(s)
}
```
读取文件内容是一个常规操作，所以Rust本身也提供了方便的方法：
```Rust
use std::fs;
use std::io;

fn read_username_from_file() -> Result<String, io::Error> {
    fs::read_to_string("hello.txt")
}
```
需要知道的是，`?`操作符如果用在main中，因为main本身无返回，或者说返回的是`()`，所以`?`不会起作用，编译时候就会报错。`?`也可以作用于Option枚举：
```Rust
fn last_char_of_first_line(text: &str) -> Option<char> {
    text.lines().next()?.chars().last()
}
```
在Rust中，main默认都是返回`()`，编译之后的可执行文件，则会跟C一样，正常退出则返回0，非正常退出返回非0。main也可以接受一个固定类型的返回，`Result<(), E>`，看下面的例子：
```Rust
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let f = File::open("hello.txt")?;

    Ok(())
}
```
> 这里的`Box<dyn Error>`表示任何Error类型