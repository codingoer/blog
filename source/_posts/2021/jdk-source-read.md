---
title: 阅读JDK源码技巧
date: "2021/8/31 20:26:25"
tags: [Java]
categories: Java
---

# 搭建环境

要想阅读JDK源码，首先要知道源码在哪，一般我们安装完JDK后，源码都会在安装目录里面。

**Mac系统**

IDEA中选择Project Structure

![20230205213855](https:image.codingoer.top/blog/20230205213855.jpg)

进入如上目录，src.zip就是JDK的源码。

![20230205213910](https:image.codingoer.top/blog/20230205213910.jpg)

在自己的github上创建一个仓库（有个记录），将src.zip解压，然后拷贝到项目中。修改项目源码路径。

![20230205213923](https:image.codingoer.top/blog/20230205213923.jpg)

修改编译内存，改大一点

![20230205213939](https:image.codingoer.top/blog/20230205213939.jpg)

修改断点调试设置

![20230205213953](https:image.codingoer.top/blog/20230205213953.jpg)

现在就可以编写代码调试了。。。。

注意点：

在写注释的时候只能在行尾添加注释，不能改变源码的行结构，否则会出现断点错乱的问题。

![20230205214017](https:image.codingoer.top/blog/20230205214017.jpg)

# 编译OpenJDK11（Mac系统）

上面这种 方法在写注释或者调试的时候有些局限性，比如多线程调试的时候效果不是很好，接下来就介绍下编译自己的JDK。

<!-- more --> 

## OpenJDK与OracleJDK的区别和联系

谷歌一下或者参考OpenJDK FAQ http://openjdk.java.net/faq/

深入理解Java虚拟机第2版：

OpenJDK是sun在2006年末把Java开源而形成的项目，Oracle JDK采用了商业实现，而OpenJDK使用的是开源的FreeType。

## 下载

首先要下载OpenJDK11的源码。

![20230205214356](https:image.codingoer.top/blog/20230205214356.jpg)

或者从Github上下载或Fork到自己的仓库。（网速不好要翻墙）https://github.com/openjdk/
解压缩后的目录

![20230205214426](https:image.codingoer.top/blog/20230205214426.jpg)

src目录是源码。。build目录是编译成功后的JDK成品。

## 准备工具

1. Xcode Command Line Tools

Xcode是一套来自苹果系统的大型软件开发工具和库。Xcode Command Line Tools是Xcode的一部分。

许多常见的基于unix的工具的安装需要GCC编译器，Xcode Command Line Tools包括GCC编辑器，Apple LLVM compiler, linker, and Make等等。所以，不需要安装全部的Xcode套件。

检测是否已安装
```bash
xcode-select -p
```

如果已经安装会提示：/Library/Developer/CommandLineTools

安装Xcode Command Line Tools
```bash
xcode-select --install
```

安装完成
```bash
clang --version
```

2. HomeBrew

HomeBrew是安装macOS中没有的UNIX工具的最简单、最灵活的方法。

安装完成后
```bash
brew --version
```

## GCC - GNU Compiler Collection

### What is GNU? 

[官网](https://www.gnu.org/)

GNU是一个操作系统项目, 名字是一个递归的GNU's Not Unix!的缩写.Linux只是一个系统内核，GNU/Linux一起打包。

如果一台用Linux的计算机是一台汽车

- 计算机硬件
轮胎，车架，发动机等等的部件

- Linux
传动杆，变速箱，各种传感器和行车电脑。负责让发动机的能量传到轮胎上，并能获取车辆运行状态的系统部件

- GNU
油门/刹车踏板、挂档杆、方向盘、仪表、车门什么的，让司机能够开和用的部件。

知乎传送：https://www.zhihu.com/question/319783573/answer/656033035

### What is GCC?

[官网](https://www.gnu.org/software/gcc/)

![20230205215215](https:image.codingoer.top/blog/20230205215215.jpg)

GNU编译器集合包括C、c++、Objective-C、Fortran、Ada、Go和D的前端，以及这些语言的库(libstdc++，…)。GCC最初是作为GNU操作系统的编译器编写的。

- gcc 是GCC中的 GUN C Compiler（C 编译器）
- g++ 是GCC中的 GUN C++ Compiler（C++编译器）

### Clang与LLVM

LLVM官网：https://llvm.org/

LLVM项目是模块化和可重用的编译器和工具链技术的集合。

Clang官网：https://clang.llvm.org/

Clang是一个C语言、C++、Objective-C语言的轻量级编译器,基于LLVM的C/C++/Objective-C编译器.

Clang项目为LLVM项目提供了C语言家族(C、c++、Objective C/ c++、OpenCL、CUDA和RenderScript)的语言前端和工具基础设施。

GCC是在linux下使用的编译器，Clang是在mac上使用的编译器。对于苹果iOS，使用的编译器是LLVM，相比于xcode 5版本前使用的GCC编译速度提高很多。

相关连接：

GCC，LLVM，Clang：https://www.jianshu.com/p/aa27c51c74fb

Clang比GCC好在哪里：https://www.zhihu.com/question/20235742

## Configure

进入到解压后的目录，找到doc目录，找到里面的building.md，仔细阅读。

执行下面命令
```bash
bash configure --with-toolchain-type=clang --with-debug-level=slowdebug --enable-dtrace --with-jvm-variants=server --with-target-bits=64 --enable-ccache --with-num-cores=8 --with-memory-size=8000 --disable-warnings-as-errors --with-sysroot=/Library/Developer/CommandLineTools/SDKs/MacOSX10.14.sdk --with-boot-jdk=/Users/lionel/Environment/openjdk10/jdk-10.jdk/Contents/Home
```

几个重要的点：
1. boot-jdk
2. toolchain选择clang
3. 使用homebrew安装缺少的工具，提示少什么就按照什么
4. with-sysroot指定SDKs地址

我再编译时安装到的工具：
```java
brew install autoconf
brew install ccache
```

### make

```bash
make all
```

Build all images(product, docs and test)
我的电脑CPU配置： 2.7 GHz Intel Core i7，使用八核编译，13分钟完成。

```bash
make images
```

Build the JDK image

## 项目配置

1. 选择编译完成的JDK
![20230205220130](https:image.codingoer.top/blog/20230205220130.jpg)

2. 修改源代码路径
![20230205220150](https:image.codingoer.top/blog/20230205220150.jpg)

3. 修改源码后重新编译
![20230205220207](https:image.codingoer.top/blog/20230205220207.jpg)