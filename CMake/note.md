## CMake学习笔记

Cmake基本模板

```cmake
cmake_minimum_required(VERSION 3.10)
#set the project name 
project(hello)

add_executable(hello hello.c)
```

project命令会指定项目的名字，然后用add_executable命令加入可执行文件。构建方法是

```bash
mkdir build && cd build && cmake .. && make
```

会在当前项目下的build目录下生成一些中间文件，不会影响到代码的目录文件。



### 1、基础

CMake内置的一些变量:

1、${PROJECT_SOURCE_DIR}指的是当前工程的目录，即CMakeLIst.txt所在的目录。

2、 ${PROJECT_BINARY_DIR}指的是编译的主目录，即MakeFIle最终所在的目录。如果在build文件夹下构建，那么该变量指的就是build目录。



CMake常用命令如下:

```cmake
#设置一个变量
SET(MYDIR src)

#设置编译器的版本信息
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD 14)

#加入警告信息或者一些其他的编译参数
add_definitions(-Wall)
#还可以使用
set(CMAKE_CXX_FLAGS "-O0 -ggdb")

#加入包含头文件目录
include_directories(${MYDIR})

#设置生成的目标文件的路径
#设置可执行文件的输出目录为bin
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
#设置库文件的可执行目录为
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

```



### 2、生成库或者引用库

可以使用add_library命令生成库

```cmake
add_library(hello STATIC hello.c)
add_library(hello STATIC hello.c)
```

如果要生成名字相同的动态库和静态库需要用到SET_TARGET_PROPERTIES指令。基本语法如下所示

```cmake
SET_TARGET_PROPERTIES(target1 target2 ...
 	PROPERTIES prop1 value1
 	prop2 value2 ...)
 	
 #用法
 SET_TARGET_PROPERTIES(hello_static PROPERTIES OUTPUT_NAME "hello")
 #后两条指令也是必须的，用于不清理动态库
 SET_TARGET_PROPERTIES(hello PROPERTIES CLEAN_DIRECT_OUTPUT 1)
SET_TARGET_PROPERTIES(hello_static PROPERTIES CLEAN_DIRECT_OUTPUT
1)
```



关于引用库，直接使用LINK_DIRECTORIES 和 TARGET_LINK_LIBRARIES

```cmake
#LINK_DIRECTORIES指定搜索库的目录
TARGET_LINK_LIBRARIES(main hello)
TARGET_LINK_LIBRARIES(main libhello.so)
#上面两条语句等价
```



### 3 例子

在demo中有一个cmake的简单例子，先mkdir build 然后在build目录进行构建即可。





----



### 参考连接

1、 [Cmake官方的文档](https://cmake.org/cmake/help/v3.16/guide/tutorial/index.html)