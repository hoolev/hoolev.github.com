---
layout: post
title: 使用Eclipse开发STM32程序
categories: 物联网
tags:
    - STM32
---

Eclipse 作为一款开源的跨平台的集成开发环境，本身就通过CDT提供了自己的项目格式，.cproject，可以自己设定make方式，
但终究在开发者和源代码之间建立了一个不太透明的隔阂，对于更加细致的编译要求也很难进行配置，
因此笔者要放弃Eclipse的编译功能，使用cmake进行项目编译工作，Eclipse则从事除了编译以外的代码编写、项目管理、源码控制等功能。

> 如果希望使用Eclipse的编译功能来开发STM32，那么[GNU ARM Eclipse](http://gnuarmeclipse.github.io/)这个项目是个不错的选择。

## 创建项目
创建一个c++项目，类型选 Makefile project - Empty Project - Other Toolchain，这个目的主要是为了让Eclipse尽量少的给我们的项目加额外的配置参数。
现在得到一个完全空白的项目，我们给他添加 CMakeLists.txt，build.sh和源码。
CMakeLists.txt用来组织源码
> 为了方便编辑cmake文件，可以下载 CmakeED 插件，它可以提供代码高亮和提示功能。

{% highlight sh%}

PROJECT(test_project)

CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
ENABLE_LANGUAGE(ASM)

FIND_PACKAGE(CMSIS REQUIRED)
FIND_PACKAGE(StdPeriphLib REQUIRED)

# Select stm32 target device
ADD_DEFINITIONS(-DSTM32F40XX) 

ADD_EXECUTABLE(${CMAKE_PROJECT_NAME} ${PROJECT_SOURCES} ${CMSIS_STARTUP_SOURCE} ${CMSIS_SOURCES} ${StdPeriphLib_SOURcES})

{% endhighlight %}

build.sh用来编译，这里使用[stm32-cmake](https://github.com/ObKo/stm32-cmake/tree/old-stdperiph)来构建STM32工程。
Eclipse在build project时会传入all，在clean project时会传入clean参数，
我们可以在脚本中使用这两个参数来编译和清除项目。

{% highlight sh%}

make_all() {
cd $stm32_cmake_path/cmsis
cmake -DCMAKE_TOOLCHAIN_FILE=$cmake_toolchain_file -DSTM32_FAMILY=F4 -DTOOLCHAIN_PREFIX=$toolchain_prefix -DSTM32F4_StdPeriphLib_DIR=$stm32_stdperiph_lib -DCMAKE_INSTALL_PREFIX=$toolchain_prefix/arm-none-eabi/ -DCMAKE_BUILD_TYPE=Release
make && make install

cd $stm32_cmake_path/stdperiph
cmake -DCMAKE_TOOLCHAIN_FILE=$cmake_toolchain_file -DSTM32_FAMILY=F4 -DTOOLCHAIN_PREFIX=$toolchain_prefix -DSTM32F4_StdPeriphLib_DIR=$stm32_stdperiph_lib -DCMAKE_INSTALL_PREFIX=$toolchain_prefix/arm-none-eabi/ -DCMAKE_BUILD_TYPE=Release
make && make install

mkdir -p $project_dir/build
cd $project_dir/build

cmake -DSTM32_CHIP=$stm32_chip -DCMAKE_TOOLCHAIN_FILE=$cmake_toolchain_file -DCMAKE_BUILD_TYPE=$cmake_build_type -DCMAKE_MODULE_PATH=$cmake_module_path -DTOOLCHAIN_PREFIX=$toolchain_prefix -DSTM32_FLASH_ORIGIN=$stm32_flash_origin -DSTM32_MIN_STACK_SIZE=$stm32_min_stack_size $project_dir 
make
}


clean_all() {
cd $stm32_cmake_path/cmsis
make clean

cd $stm32_cmake_path/stdperiph
make clean

cd $project_dir/build
make clean
}

if [ "$@"x = "all"x ]; then
	make_all
fi

if [ "$@"x = "clean"x ]; then
	clean_all	
fi

{% endhighlight %}

## 编译配置
打开项目属性，在 C++ Build 选项页的 Builder Settings里，去掉 Use default build command，
在 Build command 里输入 `sh ${ProjDirPath}/build.sh`

## 调试
首先需要把程序编译成Debug版本，在CMakeLists 中加入SET( CMAKE_BUILD_TYPE Debug )选项。
然后配置好可执行程序，工作路径，源代码搜索路径等。

至此，我们就建立起了一个完善的自定义工具开发环境。
