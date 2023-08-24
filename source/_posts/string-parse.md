---
title: Hermes帮助你实现快速字符串解析
author: 六道先生
date: 2023-08-24 13:35:39
categories:
- IT
tags:
- Rust
- Parser
+ description: 解决自定义的字符串解析功能。
---
# 为何需要字符串解析

解析字符串是我们在开发一些需要交互的工具或者平台的时候，必须要解决的一个问题。举个例子来说吧，我想很多人都应该用过`JMeter`，一款`Apache`的开源负载测试工具。这个工具中，我们使用各种组件经常会在填写的文本框中输入带有变量的字符串，比如`token=${token}`，还有会使用它的一些内置函数`user-${__threadNum}`，这种方式，可以很方便的帮助我们实现特殊的请求参数、解决前后请求关联等实际问题。

如果你经常从事一些工具的开发，这类字符串的解析恐怕是不可避免会接触到的。

不过目前可用的类似这样的字符串解析的库并不多，这里给大家介绍一款Rust开发的解析库——`hermes_ru`。

# Hermes_ru的使用

直接在`Cargo.toml`中引入就可以：

```toml
hermes_ru = "0.1.0"
```

## 字符串解析语法

`hermes_ru`所支持的字符串语法和`JMeter`中的略有不同，`JMeter`中都按照字符串处理，而`hermes_ru`则支持函数调用、变量引用、数字、布尔值、字符串这五种类型，各类型用加号`+`连接，从这点看，要比`JMeter`的字符串解析通用性更强。

来看个例子：
```rust
let input = "${hostname()}+\"dd\"+true+${name}+${invalid}+${random_str(${len})}+${multiply(1,-2.5,3)}";
```

解读一下这个字符串，由`+`分成了7段，第一段是`${hostname()}`，这是一个函数调用，在`hermes_ru`中，使用`${函数名(参数1,参数2...)}`这样的形式来调用函数。

`\"dd\"`是代表字符串，使用双引号包住字符串，以便区分数字和布尔型。

`true`是代表布尔型，那么必然另一个布尔型就是`false`。

`${name}`、`${invalid}`都是变量引用，当然，前提是存有`name`和`invalid`这两个变量，如果没有，则当作空字符串处理。

`${random_str(${len})}`是调用`random_str`函数，这里注意函数内的参数是一个变量引用`${len}`，可见在`hermes_ru`中，函数的参数可以嵌套函数调用以及变量引用。这一点是很多解析库所做不到的，`hermes_ru`在这一点上很有优势。

`${multiply(1,-2.5,3)}`是调用`multiply`函数，其中三个参数，`1`、`-2.5`、`3`。

## 保存变量

在`hermes_ru`中，需要把关联的变量事先保存到`Cache`结构中才可以被引用到，直接看代码：

```rust
// new a cache with 10 size capacity
let mut cache = Cache::new(10);
// add two variables to cache
cache.add_variable(CacheVariable::new("name".to_string(), ExpType::STRING("liudao".to_string())));
cache.add_variable(CacheVariable::new("len".to_string(), ExpType::INT(10)));
```

> 代码中，我们创建了10元素大小的`Cache`结构，数字10代表可以存放10个变量，注意在Rust中必须要固定好大小，否则会因为无法计算使用的内存大小，而无法进行线程间的传递。 <br />
> 我们调用`add_variable`方法将变量保存到`Cache`中，接受两个参数，第一个是变量名字，第二个是变量的值，必须包在枚举类型`ExpType`中，`ExpType`有`INT`、`FLOAT`、`BOOL`、`STRING`四个类型的值。<br />

## 内置函数

在`hermes_ru`中已经定义了一些常用的函数，可以直接使用，主要是这些：

+ `random_str(len)` 生成随机字符串，需要一个i32参数来指定字符串的长度，字符串包含大小写字母和数字。
+ `random_bool()` 生成随机布尔值。
+ `random_num()` or `random_num(end)` or `random_num(start,end)` 生成随机数字，如果没有参数，则生成一个随机整数；如果传入一个整数作为参数，则生成从零开始到这个值为止（不含）的随机整数；如果传入一个浮点数，则生成从零开始到这个值为止（不含）的随机浮点数；如果输入两个数，则表示生成这两个数范围内的随机数（可以是整数也可以是浮点数）。
+ `hostname()` 返回主机名称。
+ `current_time(format)` 获取当前的时间，需要传入一个时间格式作为参数。

## 自定义函数

在`hermes_ru`中如果内置函数不足以满足要求，还可以自定义函数。注意自定义函数必须满足如下格式：
```rust
Fn(Vec<ExpType>) -> Result<ExpType, Error>
```
> 可以直接定义为`fn`，也可以写成闭包的形式。

比如上面的字符串中`${multiply(1,-2.5,3)}`就是属于调用自定义函数，需要用户自己实现它：
```rust
// add a custom function to cache
cache.add_function("multiply", Box::new(|params: Vec<ExpType>| -> Result<ExpType, Error>{
    Ok(params.into_iter().fold(ExpType::FLOAT(1.0f32), |acc, item| {
        let acc_num = if let ExpType::FLOAT(float) = acc {
            float
        } else {
            0.0f32
        };
        match item {
            ExpType::INT(i) => ExpType::FLOAT(acc_num * i as f32),
            ExpType::FLOAT(f) => ExpType::FLOAT(acc_num * f),
            _ => acc,
        }
    }))
        
}));
```
> 这里采用了闭包的写法，实现了一个乘法函数。

## 完整的例子

看一下完整代码：
```rust
use hermes_ru::{cache::{ ExpType, Cache, CacheVariable }, error::Error, lexer::parse_to_string};

fn main() {
    // new a cache with 10 size capacity
    let mut cache = Cache::new(10);
    // add two variables to cache
    cache.add_variable(CacheVariable::new("name".to_string(), ExpType::STRING("liudao".to_string())));
    cache.add_variable(CacheVariable::new("len".to_string(), ExpType::INT(10)));

    // add a custom function to cache
    cache.add_function("multiply", Box::new(|params: Vec<ExpType>| -> Result<ExpType, Error>{
        Ok(params.into_iter().fold(ExpType::FLOAT(1.0f32), |acc, item| {
            let acc_num = if let ExpType::FLOAT(float) = acc {
                float
            } else {
                0.0f32
            };
            match item {
                ExpType::INT(i) => ExpType::FLOAT(acc_num * i as f32),
                ExpType::FLOAT(f) => ExpType::FLOAT(acc_num * f),
                _ => acc,
            }
        }))
        
    }));

    // define a complex str
    let input = "${hostname()}+\"dd\"+true+${name}+${invalid}+${random_str(${len})}+${multiply(1,-2.5,3)}}";
    
    // parse the str
    println!("parse result is: {}", parse_to_string(input, &mut cache));

}
```
> 注意`${invalid}`是我们故意尝试的一个不存在的变量。

输出的最后结果：
```shell
parse result is: MacBook-Pro.localddtrueliudaojdTXX8f16d-7.5
```
> 最后连接为一个字符串，`MacBook-Pro.local`是主机名称，`dd`是字符串，`true`是布尔值，`liudao`是变量`name`的值，`jdTXX8f16d`是随机字符串，`-7.5`是自定义乘法函数的计算结果。

# 源代码

`hermes_ru`是开源库，源代码在[GitHub](https://github.com/jimmyseraph/hermes)上。