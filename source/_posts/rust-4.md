---
title: Rust学习笔记（4）
author: 六道先生
date: 2022-03-10 10:37:29
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第四天开始了，主要学习Rust特有的Ownership概念。
---
# Ownership
Ownership是Rust语言所特有的，用于运行时内存管理的一套规则。这是Rust语言的核心特点。
## 前置知识
要理解Ownership概念，首先需要理解堆内存（Heap）和栈内存（Stack）的特点，这个属于基础知识了，不懂的小伙伴自行补下课。
## Ownership规则
先看一下这几条规则：
1. 每个值都需要有一个变量来承载，这个变量叫做Owner；
2. 在同一时间内，一个值只能有一个owner；
3. 当owner离开了自己的作用域（Scope），那么值就会被丢掉。
## 关于作用域
其实作用域很容易理解，跟c/c++，java等语言一样，看例子：
```Rust
    {                      // s is not valid here, it’s not yet declared
        let s = "hello";   // s is valid from this point forward

        // do stuff with s
    }                      // this scope is now over, and s is no longer valid
```
## 内存与分配
跟Java其实很像，基本数据类型（整型，浮点型，布尔型，字符型，包括这些类型组成了tuple类型），因为固定长度，类型也明确，所以会直接被分配保存到栈（stack）内存中，其余的类型，都会在堆内存中分配空间保存值，而把分配到的堆内存地址返回回来，保存在栈内存中。<br>
```Rust
    {                      
        let mut s = String::from("hello");   // s is valid from this point forward

        // do stuff with s
    }                      // this scope is now over, and s is no longer valid
```
> 这个例子和前面的很像，只是把字符串常量换成String了，这里的差别，就在于字面常量的"hello"是不可变的，其内容固定长度（5个字符类型），类型也确定，所以会保存在stack中，所以它不可变更。而实际应用中，通常字符串长度都无法在编译时确定，只有在运行时才能确定，所以这里使用了一个String类型。那么因为这个类型不属于基础类型，所以会将hello这五个字符值保存在heap中，并将heap中分配的地址、长度、容量保存到stack中。

那么其实就带来一个细节问题了，上面那段代码中的例子，当`let s`开始定义时，根据前面的说明，s有效了，在离开了作用域之后，s就会无效，此时遗留在heap中的5个字符"hello"怎么办？heap内存并不会主动去释放这个字节的空间。<br>
一些语言使用了GC的方式，比如java，使用GC的方式扫描heap中是否存在没有引用的值，这些值所占的空间会被释放。另一些语言则需要程序员主动去释放，比如C/C++，在malloc/new内存了之后，要有匹配的delete/free来进行内存释放，否则就可能会出现内存泄露问题。<br>
Rust选择了一条比较困难的路，在判断一个作用域到结尾的时候（通常就是"}"），会自动调用一个drop方法，去释放heap中无用的值所占空间。这个逻辑虽然看起来简单，但是会有很多细节的问题，导致了Rust的特殊性。来看个例子：
```Rust
    let x = 5;
    let y = x;
```
> 这个在内存中做了什么？首先在stack内存中栈顶分配了一块32个bit（4字节）大小的空间，直接存放了5，然后继续在栈顶分配了32bit的空间，依然存放了5，也就是说，两块紧挨着的内存空间，分别代表着x和y，都存放着5，这个很容易理解。

再看下面的例子：
```Rust
    let s1 = String::from("hello");
    let s2 = s1;
```
> 这会和前面的例子一样吗？对不起，完全不同。

首先s1的值`hello`存放在heap中，而stack中存放的，是值在heap内存中的地址，以及大小和容量，如下图：<br>
{% asset_img trpl04-01.svg 04-01 %}

然后是s2的赋值，s1赋值给s2的，是保存在stack中的heap内存地址、大小和容量，而不是hello这个值本身。所以，其实就变成了这样的情况：<br>
{% asset_img trpl04-02.svg 04-02 %}

那么这里就有一个问题了，我们前面说过，当变量离开自己的作用域时，Rust会调用一个drop方法，将值所占的heap空间释放掉。而我们这里的例子，s1和s2显然属于同一个作用域，那么肯定会在离开作用域时，大家都会调用drop释放heap中的值。但是注意了，s1和s2指向的heap空间是同一个，那就会出现重复释放的问题，导致内存访问异常，这是典型的安全问题。<br>
为了解决这个问题，Rust在s1赋值给s2时，会认为s1已经无用了，将其直接标识为无效。所以后面的释放就不用考虑s1了。那么下面这个错误也就可以理解了：
```Rust
    let s1 = String::from("hello");
    let s2 = s1;

    println!("{}, world!", s1);
```
这将会出现编译报错：
```Shell
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:5:28
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = s1;
  |              -- value moved here
4 | 
5 |     println!("{}, world!", s1);
  |                            ^^ value borrowed here after move

For more information about this error, try `rustc --explain E0382`.
error: could not compile `ownership` due to previous error
```
提示s1已经被"move"了。可见这种赋值，造成的其实是"move"的操作，并不是"copy"的方式，赋值之后，原来的变量就失效了。如果要保留s1，真正做到"copy"，那可以用clone的方式：
```Rust
    let s1 = String::from("hello");
    let s2 = s1.clone();

    println!("s1 = {}, s2 = {}", s1, s2);
```
这种"move"的情况，也同样出现在函数调用传值上：
```Rust
fn main() {
    let s = String::from("hello");  // s comes into scope

    takes_ownership(s);   
}
fn takes_ownership(some_string: String) { // some_string comes into scope
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
  // memory is freed.
```
s在把值"move"给函数`takes_ownership`后，自己就失效了，如果在`takes_ownership`之后要调用s，就会出现编译报错！这点在Rust编程中一定要小心。

根据这个例子，也可以这么理解Rust的Ownership机制 —— 每一个在heap内存中保存的值，只能有一个“拥有者”（Owner），也就是保存了这个内存地址的变量，一旦换了其他变量来保存，也就是换了“拥有者”，原来的拥有者就失效了。

## 引用与借用
前面的那个例子中，s一旦传给了函数，本身就失效了，因为换了Owner。如果我们后面的代码还想使用s，那就要换一种方式来给函数传值：
```Rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```
> 这段代码`calculate_length`函数的参数换成了`&String`，这个`&`符号表示引用，这里的s的类型就是String的引用类型，这个概念和C/C++一摸一样。引用的作用，就是把传入的参数的地址传进去，但并不是值本身，这样就没有改变`hello`这个值的Owner，那么s1就不会失效。s和s1的关系，看下图：<br>
{% asset_img trpl04-05.svg 04-05 %}

这种引用，在Rust中称为“借用”（borrow），很有意思，直白的表达了只是“借”，不是拥有者，借完了之后还要“还”。另外，一个变量，一次只能“借”给一个变量，不能在同一作用域被借用两次：
```Rust
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s;

    println!("{}, {}", r1, r2);
```
这段代码编译会直接报错：
```Shell
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> src/main.rs:5:14
  |
4 |     let r1 = &mut s;
  |              ------ first mutable borrow occurs here
5 |     let r2 = &mut s;
  |              ^^^^^^ second mutable borrow occurs here
6 | 
7 |     println!("{}, {}", r1, r2);
  |                        -- first borrow later used here
```
但是，如果“借用”是不可变借用，那可以被多次借用，这是Rust为了防止出现“数据争用”（data race）做的规定：
```Rust
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    let r3 = &mut s; // BIG PROBLEM

    println!("{}, {}, and {}", r1, r2, r3);
```
上面这个例子还说明了一个规则，不可变借用和可变借用不可同时使用，因为不可变借用不希望借用所指向的数据被忽然变更。但是下面这种情况可以：
```Rust
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    println!("{} and {}", r1, r2);
    // variables r1 and r2 will not be used after this point

    let r3 = &mut s; // no problem
    println!("{}", r3);
```
> 只要在r3借用之后，不再出现使用r1、r2的语句，那就不会有编译问题。

## 空悬引用
其实就是指无效引用，被引用的内存空间已经被释放，那这个引用就无效了，Rust会直接在编译时进行报错提示，看下面这个例子：
```Rust
fn main() {
    let reference_to_nothing = dangle();
}
fn dangle() -> &String { // dangle returns a reference to a String

    let s = String::from("hello"); // s is a new String

    &s // we return a reference to the String, s
} // Here, s goes out of scope, and is dropped. Its memory goes away.
  // Danger!
```

总体来说，Ownership这个概念中的“引用”其实跟C/C++挺像的，但是C/C++不会报这样的编译错误，并且不会有任何限制，而Rust为了内存访问安全的考虑，则做了很多限制，从这一点上看，Rust在内存安全上花了很多功夫。
## 切片slice类型
切片类型也是一种引用，所以本身不会存储值。切片的用法跟很多语言一样，像python、golang。看下面的例子：
```Rust
    let s = String::from("hello world");

    let hello = &s[0..5];
    let world = &s[6..11];

    let len = s.len();

    let slice = &s[3..len];
    let slice = &s[3..];
    let slice = &s[..];
```
在使用slice时，要注意如果被引用的对象本身被另外操作了，那就会出现访问错误，比如下面这个例子：
```Rust
fn main() {
    let mut s = String::from("hello world");

    let word = &s[..5];

    s.push_str("!!"); // error!

    println!("the first word is: {}", word);

}
```
这段代码会导致编译错误：
```Shell
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
  --> src/main.rs:10:5
   |
8  |     let word = &s[..5];
   |                 - immutable borrow occurs here
9  | 
10 |     s.push_str("!!"); // error!
   |     ^^^^^^^^^^^^^^^^ mutable borrow occurs here
11 | 
12 |     println!("the first word is: {}", word);
   |                                       ---- immutable borrow later used here

For more information about this error, try `rustc --explain E0502`.
```
这个错误，其实就跟前面说的，之前的slice，是做了不可变借用，而后面的push_str则发生了可变借用，那么在可变借用发生后，不可以再次使用前面的不可变借用。<br>
再回到字符串字面常量：
```Rust
let s = "hello world";
```
现在可以理解s了，它其实也是一个切片类型，是指向字符串字面常量的一个不可变借用。这就解释了为何s不能变更了。看下面的例子来理解切片引用：
```Rust
fn main() {
    let my_string = String::from("hello world");

    // `first_word` works on slices of `String`s, whether partial or whole
    let word = first_word(&my_string[0..6]);
    let word = first_word(&my_string[..]);
    // `first_word` also works on references to `String`s, which are equivalent
    // to whole slices of `String`s
    let word = first_word(&my_string);

    let my_string_literal = "hello world";

    // `first_word` works on slices of string literals, whether partial or whole
    let word = first_word(&my_string_literal[0..6]);
    let word = first_word(&my_string_literal[..]);

    // Because string literals *are* string slices already,
    // this works too, without the slice syntax!
    let word = first_word(my_string_literal);
}
```