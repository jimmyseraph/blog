---
title: Rust多平台交叉编译
author: 六道先生
date: 2022-06-04 09:43:31
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 介绍如何编译多平台可执行文件。
---
# Rust编译

Rust可以通过`--target`参数来指定希望编译的平台，如果不加这个参数，那就是根据当前系统进行编译。可以通过rustc命令来查看所支持的编译目标：
```Shell
$ rustc --print target-list 
```

但是目前来说，rust的跨平台编译跟Golang还是有很大差距的，从我测试下来的结果看，基本都失败了。因为使用跨平台编译，需要准备的依赖工具太多了，所以跨平台编译非常不方便。

# cross工具

有一个工具团队专门维护了一组docker，配置了不同编译环境，可以用于rust的跨平台编译。[GitHub传送门]

[GitHub传送门]: https://github.com/cross-rs/cross

使用比较简单，首先安装cross工具：
```Shell
$ cargo install -f cross
```

看看目前的docker列表：
```
Dockerfile.aarch64-linux-android
Dockerfile.aarch64-unknown-linux-gnu
Dockerfile.aarch64-unknown-linux-musl
Dockerfile.arm-linux-androideabi
Dockerfile.arm-unknown-linux-gnueabi
Dockerfile.arm-unknown-linux-gnueabihf
Dockerfile.arm-unknown-linux-musleabi
Dockerfile.arm-unknown-linux-musleabihf
Dockerfile.armv5te-unknown-linux-gnueabi
Dockerfile.armv5te-unknown-linux-musleabi
Dockerfile.armv7-linux-androideabi
Dockerfile.armv7-unknown-linux-gnueabihf
Dockerfile.armv7-unknown-linux-musleabihf
Dockerfile.asmjs-unknown-emscripten
Dockerfile.i586-unknown-linux-gnu
Dockerfile.i586-unknown-linux-musl
Dockerfile.i686-linux-android
Dockerfile.i686-pc-windows-gnu
Dockerfile.i686-unknown-freebsd
Dockerfile.i686-unknown-linux-gnu
Dockerfile.i686-unknown-linux-musl
Dockerfile.mips-unknown-linux-gnu
Dockerfile.mips-unknown-linux-musl
Dockerfile.mips64-unknown-linux-gnuabi64
Dockerfile.mips64el-unknown-linux-gnuabi64
Dockerfile.mipsel-unknown-linux-gnu
Dockerfile.mipsel-unknown-linux-musl
Dockerfile.powerpc-unknown-linux-gnu
Dockerfile.powerpc64-unknown-linux-gnu
Dockerfile.powerpc64le-unknown-linux-gnu
Dockerfile.riscv64gc-unknown-linux-gnu
Dockerfile.s390x-unknown-linux-gnu
Dockerfile.sparc64-unknown-linux-gnu
Dockerfile.sparcv9-sun-solaris
Dockerfile.thumbv6m-none-eabi
Dockerfile.thumbv7em-none-eabi
Dockerfile.thumbv7em-none-eabihf
Dockerfile.thumbv7m-none-eabi
Dockerfile.wasm32-unknown-emscripten
Dockerfile.x86_64-linux-android
Dockerfile.x86_64-pc-windows-gnu
Dockerfile.x86_64-sun-solaris
Dockerfile.x86_64-unknown-freebsd
Dockerfile.x86_64-unknown-linux-gnu
Dockerfile.x86_64-unknown-linux-musl
Dockerfile.x86_64-unknown-netbsd
```

举个例子，现在编译windows平台下的可执行文件：
```Shell
$ cross build --target x86_64-pc-windows-gnu
```
其他用法查看cross官方文档。