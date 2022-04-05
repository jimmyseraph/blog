---
title: Rust学习笔记（18）
author: 六道先生
date: 2022-04-04 11:21:39
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第十八天开始了，主要学习Rust的一些高级特性（一）。
---
# 非安全Rust
我们讨论至今的Rust都是有着安全的内存访问策略，任何非安全的操作都会被阻止。但是在Rust内部还存在第二种语言，它没有强制的安全访问要求，这种Rust内部的语言称为非安全Rust。它跟普通的Rust一样工作，但是给了我们更高的权限。
## 不安全的超级权限
要启用非安全Rust，需要添加`unsafe`关键字，然后在后面的语句块中，就可以使用下面五种超级权限：
1. 反引用原生指针
2. 调用一个非安全的函数或方法
3. 访问或修改一个可变静态变量
4. 实现一个非安全trait
5. 访问`union S`中的字段

## 反引用原生指针
所谓的原生指针跟普通的引用差不多，可以定义为可变更和不可变更，写作这样的:`*mut T`和`*const T`。这里的`*`不是反引用的意思，这是类型名称的一部分。原生指针和智能指针以及引用的区别有这么几点：
1. 原生指针可以忽略借用规则，允许同时存在可不变和可变指针指向目标值，或者多个可变指针指向目标值；
2. 原生指针不能保证指向的内存一定有效；
3. 原生指针允许为空；
4. 不用实现任何自动清理方法。

看看如何从引用来定义原生指针：
```Rust
    let mut num = 5;

    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;
```
这里用了`as`关键字来进行类型转换。<br>
还可以直接用一个内存地址来定义一个原生指针：
```Rust
    let address = 0x012345usize;
    let r = address as *const i32;
```
我们在安全Rust中创建了原生指针，但是不能直接反引用它，只能在非安全Rust中进行反引用：
```Rust
    let mut num = 5;

    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;

    unsafe {
        println!("r1 is: {}", *r1);
        println!("r2 is: {}", *r2);
    }
```
## 调用非安全函数或方法
看例子：
```Rust
    unsafe fn dangerous() {}

    unsafe {
        dangerous();
    }
```

## 局部使用非安全语句
通常来说，一个函数或者方法，不需要所有代码都是非安全的，只是其中部分需要使用非安全Rust，本身可以作为一个普通的安全函数或方法来使用：
```Rust
use std::slice;

fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();

    assert!(mid <= len);

    unsafe {
        (
            slice::from_raw_parts_mut(ptr, mid),
            slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut vector = vec![1, 2, 3, 4, 5, 6];
    let (left, right) = split_at_mut(&mut vector, 3);
}
```
在代码中也可以使用非安全赋值：
```Rust
    use std::slice;

    let address = 0x01234usize;
    let r = address as *mut i32;

    let slice: &[i32] = unsafe { slice::from_raw_parts_mut(r, 10000) };
```
> 这个slice变量只是一个内存引用，而from_raw_parts_mut并不确保一定可以被分配到10000个i32大小的内存空间。所以如果此时试图访问这个slice，则是空的未定义值。

## 外部调用
Rust可以通过`extern`关键字来调用其他语言的ABI，举个例子：
```Rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```
还可以用extern来定义一个可以被外部调用的函数：
```Rust
#[no_mangle]
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```
> 编译成共享库之后，可以被C语言链接进去使用。

## 访问或修改可变静态变量
看例子：
```Rust
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_to_count(3);

    unsafe {
        println!("COUNTER: {}", COUNTER);
    }
}
```

## 实现非安全的trait
看例子：
```Rust
unsafe trait Foo {
    // methods go here
}

unsafe impl Foo for i32 {
    // method implementations go here
}

fn main() {}
```

## 访问union结构
union结构跟struct类似，不过使用时只有一个类型被使用，这个来自于C语言的union。因为不能确定union到底是什么类型被使用，所以访问union是不安全的，需要放在unsafe中。

# 高级Traits
这里来学一下traits的高级特性。
## trait定义中的替换类型
在trait定义中可以使用type关键字定义一个替换类型，在被其他模块实现中可以替换为具体的类型。看例子：
```Rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```
实现时候，可以这样写：
```Rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

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
这里可能也有一个疑惑，为何不用泛型呢？像这样定义trait：
```Rust
pub trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```
泛型和替换类型很类似，但是这里如果用泛型，那么在实现时候，就可以这么写：
```Rust
impl Iterator<u32> for Counter {...}
```
但是，同样可以再定义`Iterator<String>`,`Iterator<i32>`等等实现在同一个Counter中。当我们使用Counter的一个实例，调用next方法时候，编译器就无法确认会使用哪个实现类型，需要特别去指定类型才行。而使用替换类型就不需要这样麻烦了。
## 泛型默认值和运算符重载
泛型默认值，定义方式很简单`<PlaceholderType=ConcreteType>`。<br>
至于运算符重载，有点类似C++，但是Rust不是允许你可以重载任意运算符，只允许重载`std::ops`中所列的trait，比如Add（`+`号），举个例子来重载`+`号，实现两个Point实例可以进行加法运算:
```Rust
use std::ops::Add;

#[derive(Debug, Copy, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    assert_eq!(
        Point { x: 1, y: 0 } + Point { x: 2, y: 3 },
        Point { x: 3, y: 3 }
    );
}
```
在标准库的Add这个trait定义中，使用了泛型默认值：
```Rust
trait Add<Rhs=Self> {
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}
```
这里的`<Rhs=Self>`就是指定了当实现Add的时候（rhs表示`right hand side`——右手边，也就是说，这个泛型用于指定`+`号右边的值的类型），如果没有指定泛型，那就使用Self这个类型，Self在标准库中表示当前这个类型。在上面Point的例子中，就是值Point类型。<br>
我也也可以指定这个泛型是什么，比如我们需要重载`+`号对两个不同类型进行加法操作，看例子：
```Rust
use std::ops::Add;

struct Millimeters(u32);
struct Meters(u32);

impl Add<Meters> for Millimeters {
    type Output = Millimeters;

    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + (other.0 * 1000))
    }
}
```

## 调用同名方法
假设有一个struct实现了多个trait的同名方法，并且自己也有一个同名方法：
```Rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

impl Human {
    fn fly(&self) {
        println!("*waving arms furiously*");
    }
}
```
Human一共实现了3个fly方法，当我们在Human实例上调用fly方法：
```Rust
fn main() {
    let person = Human;
    person.fly();
}
```
这会执行哪一个fly呢？实际上是Human本身那个fly，打印结果是：
```Shell
*waving arms furiously*
```
如果需要调用其他trait中的fly方法，需要这样写：
```Rust
fn main() {
    let person = Human;
    Pilot::fly(&person);
    Wizard::fly(&person);
    person.fly();
}
```
但如果是一个不带self参数的方法呢？像这样：
```Rust
trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Dog {
    fn baby_name() -> String {
        String::from("Spot")
    }
}

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn main() {
    println!("A baby dog is called a {}", Dog::baby_name());
}
```
这样调用`Dog::baby_name()`返回的显然是`Spot`，而我们在此处不能使用`Animal::baby_name()`这种方式，因为没有实现，而且没有self传参，可以理解为一个静态方法，Rust也无法判断实现方法。所以这里需要使用下面这种语法：
```Rust
fn main() {
    println!("A baby dog is called a {}", <Dog as Animal>::baby_name());
}
```
这种语法我们称为完整限定名（fully qualified），语法为：
```Rust
<Type as Trait>::function(receiver_if_method, next_arg, ...);
```
## 父类trait
类似Java中的接口继承，trait也可以定义父类trait，看例子：
```Rust
use std::fmt;

trait OutlinePrint: fmt::Display {
    fn outline_print(&self) {
        let output = self.to_string();
        let len = output.len();
        println!("{}", "*".repeat(len + 4));
        println!("*{}*", " ".repeat(len + 2));
        println!("* {} *", output);
        println!("*{}*", " ".repeat(len + 2));
        println!("{}", "*".repeat(len + 4));
    }
}
```
`fmt::Display`就是OutlinePrint的父类，如果某个struct要实现OutlinePrint，必须还要实现Display，看例子：
```Rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

impl OutlinePrint for Point {}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 1, y: 3 };
    p.outline_print();
}
```

## 新类型模式
比如我们想增强一下Rust内部结构`Vec<T>`，让它实现一个Display的trait，使其具备打印输出能力，这里我们就需要定义一个新类型模式，来包装`Vec<T>`，看例子：
```Rust
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    println!("w = {}", w);
}
```

# 高级类型
## 类型别名
看例子：
```Rust
fn main() {
    type Kilometers = i32;

    let x: i32 = 5;
    let y: Kilometers = 5;

    println!("x + y = {}", x + y);
}
```
当然这个例子并不好，其实类型别名主要是用于减少代码的重复率，比如这样的：
```Rust
    let f: Box<dyn Fn() + Send + 'static> = Box::new(|| println!("hi"));

    fn takes_long_type(f: Box<dyn Fn() + Send + 'static>) {
        // --snip--
    }

    fn returns_long_type() -> Box<dyn Fn() + Send + 'static> {
        // --snip--
    }
```
看着特别累赘，可以改成这样：
```Rust
    type Thunk = Box<dyn Fn() + Send + 'static>;

    let f: Thunk = Box::new(|| println!("hi"));

    fn takes_long_type(f: Thunk) {
        // --snip--
    }

    fn returns_long_type() -> Thunk {
        // --snip--
    }
```
还有在Result的处理中，也可以使用，比如：
```Rust
use std::fmt;

type Result<T> = std::result::Result<T, std::io::Error>;

pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;

    fn write_all(&mut self, buf: &[u8]) -> Result<()>;
    fn write_fmt(&mut self, fmt: fmt::Arguments) -> Result<()>;
}
```
## Never类型返回
当我们需要返回一个类型，但是本身又没有需要返回的值，可以用`!`来表示，这有点类似java中的null。Never类型返回的形式类似这样：
```Rust
fn bar() -> ! {
    // --snip--
}
```
那不是可以直接不做返回不就可以了吗？事实并非如此，比如下面的场景：
```Rust
impl<T> Option<T> {
    pub fn unwrap(self) -> T {
        match self {
            Some(val) => val,
            None => panic!("called `Option::unwrap()` on a `None` value"),
        }
    }
}
```
这里的unwrap方法，返回一个泛型T，那么在match分支中，`Some(val)`分支返回的是val本身，val属于T类型，这个返回匹配T没问题；那么None分支呢？如果`panic!`没有任何返回，其实这里会编译报错，事实上，`panic!`的返回类型就是`!`这个类型，也是可以匹配上T类型的，所以才不会出现编译错误。那么同理，`print!`也是具备这个返回类型的。
## 动态大小类型
动态大小类型（dynamically sized types）简称DST。首先，我面要清楚Rust本身不允许在定义一个变量时大小不固定，以`str`为例，还记得为何我面使用字符串的时候，都是使用`&str`而不是使用`str`呢？因为`str`是一个DST，大小不确定：
```Rust
    let s1: str = "Hello there!";
    let s2: str = "How's it going?";
```
这里肯定会出现编译错误。因为s1和s2都是str类型，然后大小确不一样，这是不可以的。而使用`&str`则没问题，因为是引用，大小是固定的。<br>
在Rust中存在一个名字叫Sized的trait，它用于告诉编译器在编译的时候，是否需要知道类型大小。我们平时写的泛型定义：
```Rust
fn generic<T>(t: T) {
    // --snip--
}
```
其实等价于：
```Rust
fn generic<T: Sized>(t: T) {
    // --snip--
}
```
所以在使用这个泛型时候，必须指定一个确认大小的类型才可以。如果需要不确定大小的类型（DST）时，可以这样定义：
```Rust
fn generic<T: ?Sized>(t: &T) {
    // --snip--
}
```
当然，也要注意把T换成`&T`。