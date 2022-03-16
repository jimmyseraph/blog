---
title: Rust学习笔记（8）
author: 六道先生
date: 2022-03-14 14:00:26
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第八天开始了，主要学习Rust的通用集合，主要是三个类型，一个叫vector，一个是String，另一个是map。
---
# Vector
## 定义Vector
先看如何创建一个空的Vector：
```Rust
    let v: Vec<i32> = Vec::new();
```
如果创建带有值的Vector，那就不需要给v说明类型，因为可以推测：
```Rust
    let v = vec![1, 2, 3];
```
> vec很明显，是个宏。

## 更新Vector
如果需要把值放入Vector，可以使用push方法：
```Rust
    let mut v = Vec::new();

    v.push(5);
    v.push(6);
    v.push(7);
    v.push(8);
```
> 这里的v不需要指定类型，因为后面push的值说明了v的类型。

## 读取Vector的值
有两种取值方式，看下面的例子：
```Rust
    let v = vec![1, 2, 3, 4, 5];

    let third: &i32 = &v[2];
    println!("The third element is {}", third);

    match v.get(2) {
        Some(third) => println!("The third element is {}", third),
        None => println!("There is no third element."),
    }
```
> 注意这里面的两个细节：<br>
> 1. vector中的元素序号从0开始，这一点跟大部分语言一样；
> 2. vector中的元素可以用序号访问（&+[]），也可以用get方法访问，前者返回的是一个引用，后者返回的是`core::option::Option<T>`这个枚举类型。

再来看访问越界的情况：
```Rust
    let v = vec![1, 2, 3, 4, 5];

    let does_not_exist = &v[100];
    let does_not_exist = v.get(100);
```
> 用`&v[100]`来访问的时候，100明显已经越界了，会在运行时候报一个panic；用get来访问时发生越界，不会报panic，会返回`core::option::Option<T>::None`。

看下关于ownership的规则：
```Rust
    let mut v = vec![1, 2, 3, 4, 5];

    let first = &v[0];

    v.push(6);

    println!("The first element is: {}", first);
```
这段代码看似没问题，但实际会编译报错：
```Shell
$ cargo run
   Compiling collections v0.1.0 (file:///projects/collections)
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:5
  |
4 |     let first = &v[0];
  |                  - immutable borrow occurs here
5 | 
6 |     v.push(6);
  |     ^^^^^^^^^ mutable borrow occurs here
7 | 
8 |     println!("The first element is: {}", first);
  |                                          ----- immutable borrow later used here
```
> 根据前面我们提过的Rust的规定，一个值不能在一个作用范围内同时发生不可变借用和可变借用。那么在这里发生了什么呢？<br>
首先v的值，存放在heap的一块连续空间中（地址连续的5块i32大小的空间），v本身在stack中存着heap存的值的地址。<br>
当执行`let first = &v[0]`时，heap中存第一个值的空间地址作为引用传给了first变量。因为first没有使用mut关键字，所以属于不可变借用。<br>
然后执行`v.push(6)`，执行push需要在heap中连续5个i32空间后面再追加一个i32大小的空间。但是有可能在原来的地址上添加不了连续的新空间了，那就会在heap中新找一块连续6个i32空间的地方，把原来的5个值，外加新push进的第六个值，一起放进去，然后把新地址保存给v。从这个意义上来讲，这个push在Rust中，是被认作一个可变借用。那么，一旦地址发生变动，first就指向了一个已经失效的引用，会导致安全问题，所以这就是一个违背了同一作用域内即发生可变借用，又发生不变借用的问题。并且在后面的println宏中，又使用了first这个不可变借用，违反了ownership的原则，那就是编译报错的原因了。

## 遍历Vector的值
用循环来遍历Vector的值：
```Rust
    let v = vec![100, 32, 57];
    for i in &v {
        println!("{}", i);
    }
```
也可以做可变的遍历：
```Rust
    let mut v = vec![100, 32, 57];
    for i in &mut v {
        *i += 50;
    }
```
> 这里的`*`表示反引用操作（dereference），其实用C/C++来说，就是指针（pointer）。i拿到的引用，然后用`*i`取到值。

## 使用枚举类型搭配Vector来保存不同类型的值
看个例子：
```Rust
    enum SpreadsheetCell {
        Int(i32),
        Float(f64),
        Text(String),
    }

    let row = vec![
        SpreadsheetCell::Int(3),
        SpreadsheetCell::Text(String::from("blue")),
        SpreadsheetCell::Float(10.12),
    ];
```
# String
## 什么是String
首先，在Rust核心语言中只有一种字符串，那就是字符串切片（slice）：str，我们通常看到 的就是它的引用形式`&str`。<br>
String类型是Rust标准库里面提供的，根str不一样，它是可增长，可变，可拥有（owned）的UTF-8的字符串类型。<br>
在Rust标准库里面，还有其他一些字符串类型，像`OsString`, `OsStr`, `CString`, 以及`CStr`。

## 创建String
可以用new方法创建一个空的String：
```Rust
    let mut s = String::new();
```
另外，还有一个to_string方法，可以把任何类型转成string类型，当然这个类型必须实现Display方法：
```Rust
    let data = "initial contents";

    let s = data.to_string();

    // the method also works on a literal directly:
    let s = "initial contents".to_string();
```
还有from方法，前面用过：
```Rust
    let s = String::from("initial contents");
```
## 变更String
使用String类型可以用push_str方法添加字符串：
```Rust
    let mut s = String::from("foo");
    s.push_str("bar");
```
还可以用push来添加一个字符：
```Rust
    let mut s = String::from("lo");
    s.push('l');
```
还有`+`号，可以用于字符串拼接，但是需要注意引用和owner的关系：
```Rust
    let s1 = String::from("Hello, ");
    let s2 = String::from("world!");
    let s3 = s1 + &s2; // note s1 has been moved here and can no longer be used
```
> 注意，`+`号之后的参数要使用引用，否则将会编译报错。

也可以使用format宏来构建格式化字符串：
```Rust
    let s1 = String::from("tic");
    let s2 = String::from("tac");
    let s3 = String::from("toe");

    let s = format!("{}-{}-{}", s1, s2, s3);
```
> format宏也是使用的这些变量的引用。

## 字符串长度
在Rust中，一个字符串对象，有len方法可以计算所占字节数。这里需要注意，len统计的是字节数，而不是字符数，因为字符串采用UTF-8编码，属于可变长度，那么就有如下情况：
```Rust
    let s1 = String::from("hello");
    let s2 = "你好";
    println!("{}, {}", s1.len(), s2.len());
```
执行结果为：
```Shell
   Compiling demo1 v0.1.0 (/Users/luyiyi/rustProj/demo1)
    Finished dev [unoptimized + debuginfo] target(s) in 0.37s
     Running `target/debug/demo1`
5, 6
```
> `hello`字符串计算的字节数是5，而`你好`字符串，则返回为6个字节，因为在UTF-8编码中，英文字母占一个字节，而一个中文占3个字节。
如果对字符串进行切片，则需要注意是否为完整的字符：
```Rust
    let s1 = String::from("hello");
    let s = &s1[0..1];
```
> s的值就是h，这个没问题。

在看下面这个：
```Rust
    let s1 = String::from("你好");
    let s = &s1[0..1];
```
> 这在运行时会报错，因为根据切片`[0..1]`，取了一个字节，而`你`这个字符需要3个字节，所以这个切片无法取出完整的一个UTF-8字符，那就会出现错误。

## 遍历字符串里的元素
可以按照字符来遍历：
```Rust
    let s2 = "你好";
    for c in s2.chars() {
        println!("{}", c);
    }
```
打印结果是：
```Shell
   Compiling demo1 v0.1.0 (/Users/luyiyi/rustProj/demo1)
    Finished dev [unoptimized + debuginfo] target(s) in 0.29s
     Running `target/debug/demo1`
你
好
```
也可以按照字节来遍历：
```Rust
    let s2 = "你好";
    for b in s2.bytes() {
        println!("{}", b);
    }
```
打印结果是：
```Shell
   Compiling demo1 v0.1.0 (/Users/luyiyi/rustProj/demo1)
    Finished dev [unoptimized + debuginfo] target(s) in 0.29s
     Running `target/debug/demo1`
228
189
160
229
165
189
```
# Hash Map
## 创建HashMap
`HashMap<K, V>`类型可以用new来创建一个空的类型，然后可以用insert插入键值对：
```Rust
    use std::collections::HashMap;
    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);
```
也可以用两个vector拼成一个map：
```Rust
    use std::collections::HashMap;
    let teams = vec![String::from("Blue"), String::from("Yellow")];
    let initial_scores = vec![10, 50];

    let mut scores: HashMap<_, _> =
        teams.into_iter().zip(initial_scores.into_iter()).collect();
```
> 这里的scores一定要指定类型，否则会报错，因为zip返回的类型必须依赖scores的类型，如果不指定，那就无法推测了。至于HashMap里key和value的类型，则不需要指定，因为可以根据vector来推测。

## 关于HashMap的Ownership
看下面的例子：
```Rust
    use std::collections::HashMap;
    let field_name = String::from("Favorite color");
    let field_value = String::from("Blue");

    let mut map = HashMap::new();
    map.insert(field_name, field_value);
    // field_name and field_value are invalid at this point, try using them and
    // see what compiler error you get!
```
> 在insert之后，field_name和field_value就无法使用了，因为已经更换了owner。除非使用引用进行insert。

## HashMap的访问
看下面的例子：
```Rust
    use std::collections::HashMap;
    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    let team_name = String::from("Blue");
    let score = scores.get(&team_name);
```
> map中的元素，可以通过get方法获取，get方法返回的是Option枚举类型包装的值。比如这里，返回的就是`Some(&10)`。如果get的key并不存在于这个map中，那就返回`None`。

还可以用for循环来对Map进行遍历：
```Rust
    use std::collections::HashMap;
    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }
```
## 修改HashMap中的值
map中的key是唯一的，如果有另一组键值对插入map，则会覆盖key一样的值，看例子：
```Rust
    use std::collections::HashMap;
    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Blue"), 25);

    println!("{:?}", scores);
```
> Blue的值会替换为25。

map还有一个特殊的entry方法，用于判断是否有这个key，如果没有，则可以赋值，看例子：
```Rust
    use std::collections::HashMap;

    let mut scores = HashMap::new();
    scores.insert(String::from("Blue"), 10);

    scores.entry(String::from("Yellow")).or_insert(50);
    scores.entry(String::from("Blue")).or_insert(50);

    println!("{:?}", scores);
```
结果是：
```Shell
   Compiling demo1 v0.1.0 (/Users/luyiyi/rustProj/demo1)
    Finished dev [unoptimized + debuginfo] target(s) in 0.48s
     Running `target/debug/demo1`
{"Yellow": 50, "Blue": 10}
```
or_insert会返回map中这个key对应的value的引用。可以用这个引用来对值进行修改，比如下面的这个例子：
```Rust
    use std::collections::HashMap;

    let text = "hello world wonderful world";

    let mut map = HashMap::new();

    for word in text.split_whitespace() {
        let count = map.entry(word).or_insert(0);
        *count += 1;
    }

    println!("{:?}", map);
```
将会打印：`{"world": 2, "hello": 1, "wonderful": 1}`。