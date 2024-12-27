---
title: 端侧AI引擎MNN在Android上的部署初探索
tags:
  - AI
  - Android
  - C++
key: blog-comments
---
本文讲解如何运行阿里巴巴的深度神经网络引擎 [MNN](https://github.com/alibaba/MNN) 的 Android 图像识别示例,可以从中了解到如何在 Mac OS 上配置 C++ 环境。

<!--more-->
## 背景

## 克隆示例项目
[MNN](https://github.com/alibaba/MNN) 的 Android 图像识别示例项目在 [github上的MNN项目project/android/demo文件夹](https://github.com/alibaba/MNN/tree/master/project/android/demo)。

克隆文件到本地后,根据 [readme](https://github.com/alibaba/MNN/tree/master/project/android/demo) 操作。

首先需要先编译 `MNNConvert` ,这是 MNN 的模型转换工具模块。编译该模块的代码如下

```bash
# 在项目根目录下执行
mkdir build && cd build
cmake -DMNN_BUILD_CONVERTER=ON ..
make -j8
```

我们看到使用了 cmake 命令与 make 命令,因此我们需要配置 c++ 编译环境,主要是 [CMake](https://cmake.org/) 和 [GCC(GNU 编译器集合)](https://gcc.gnu.org/).

## 环境配置


### CMake 与 GCC
[CMake](https://cmake.org/) 是一个跨平台的构建编译工具,常用于 C 和 C++ 项目。它可以通过编写简单的配置文件 `CMakeLists.txt` ,这个文件描述了编译方式和项目依赖, [CMake](https://cmake.org/) 来生成适用于不同平台的编译和链接命令文件 `makefile`,这个 `makefile` 文件定义了一系列的规则来指定,哪些文件需要先编译,哪些文件需要后编译,哪些文件需要重新编译,甚至于进行更复杂的功能操作。然后再使用 make 工具来编译构建项目,make 工具会自动加载当前目录下的 `makefile` 文件。

- Mac OS 配置

访问 [CMake](https://cmake.org/) ,然后在右上角的 `DOWNLOAD` 里找到最新版本的 MacOS 的 dmg 安装包，我这里使用的是 3.31.3 版本,然后正常下载安装。

安装完成后你点击应用里面的 CMake ,会打开图形界面,但我们实际需要的是命令行工具。事实上，我们打开的这个程序叫做 `cmake-gui`，而命令行程序叫做 `cmake`，他们都在 `/Applications/CMake.app/Contents/bin` 目录下。因此我们只需要在 `~/.bash_profile` 里增加配置就好。

```bash
PATH=$JAVA_HOME/bin:$PATH:.:/Applications/CMake.app/Contents/bin
```

然后再在终端执行 `source ~/.bash_profile` 就可以使用 cmake 命令行了。

而Xcode命令行工具包含了 GCC ,因此安装 Xcode 命令行工具就可以使用 `make` 命令。

- window10 配置



### Android NDK 配置
[Android NDK](https://developer.android.google.cn/ndk?hl=zh-cn) 是让 Android 的 Java 或者 Kotlin 代码调用 C 和 C++ 代码的工具集,我们需要在 Android 上调用 [MNN](https://github.com/alibaba/MNN) 的 C++ 函数,因此需要它。这里我使用的是 Android NDK r21e 版本,在 Android Studio 的 SDK manager 里面版本号是  21.4.7075529。


## 运行 Android 示例项目

我下载 Android NDK 后文件路径在 /Users/ben/Library/Android/sdk/ndk/21.4.7075529 ,因此需要在 MNN 的示例项目的 `local.properties` 中指定 NDK 目录。
```
...
ndk.dir=/Users/ben/Library/Android/sdk/ndk/21.4.7075529
```