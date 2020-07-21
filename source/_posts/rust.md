---
title: Clion 调试 Rust
date: 2020-07-17 11:11:54
author: Rootkit
top: false
mathjax: false
categories: Rust
tags:
  - Rust
---







## 安装 mingw 

安装 msys2 (包含 mingw-64 )

下载地址  https://www.msys2.org/ 

开一个 mingw 的终端，安装编译工具：

```
 pacman -Syu
 pacman -S mingw-w64-x86_64-toolchain
```

假设安装在 `c:\msys64` 目录下，则在系统的环境变量中，增加一个：

```
 MSYS2_HOME  C:\msys64
 PATH        <原来的路>;%MSYS2_HOME%\bin;%MSYS2_HOME%\mingw64\bin
```



## 安装 rust gnu 工具链

在 windows 上使用 rustup 安装的 rust 编译环境默认使用了 msvc 编译链，需要安装 gnu 编译链

```
 rustup install stable-gnu
 rustup default stable-gnu
```



## 设置 clion 编译工具链

在 clion 的 `File -> Settings -> Build, Execution, Deployment -> Toolchains` ，加上一个 mingw 的工具链，设置目录为 msys2 中的 mingw64 目录。如 msys2 安装在 `c:\msys64` ，则目录为 `c:\msys64\mingw64` ，目录正确的情况下，make 、 c-compiler 、 c++ compiler 、 debugger 等自动找到。

完成这些设置后，就可以使用 clion 调试 rust 了



<img src ="/images/2020/rust01.jpg"/>



<img src ="/images/2020/rust02.png"/>

