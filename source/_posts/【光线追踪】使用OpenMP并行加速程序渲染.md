---
title: 【光线追踪】使用OpenMP并行加速程序渲染
date: 2023-06-13 11:32:27
tags:
    - 光线追踪
    - C++
    - 优化
    - 并行计算

categories: 光线追踪
---

**前言**： 此文章是第一篇关于光线追踪中细节的工具文章，不会涉及太多原理，更多的是在实际项目中的一个可参考方案及我所踩过的坑的记录（~~拿出来给大家开心一下~~）

+ 平台：Windows
+ 工具：cmake

# 如何在cmake中开启OpenMP

先看`CMakeLists.txt`中需要**增加**什么代码
```C++
# omp 并行
find_package(OpenMP REQUIRED)

# 设置项目最终需要链接的list
set(PROJ_INCLUDE_DIRS)
set(PROJ_LIBS)
set(PROJ_COMPILE_OPTIONS)

list(APPEND PROJ_LIBS ${OpenMP_CXX_LIBRARIES})
list(APPEND PROJ_INCLUDE_DIRS ${OpenMP_CXX_INCLUDE_DIRS})
list(APPEND PROJ_COMPILE_OPTIONS ${OpenMP_CXX_FLAGS})

# 进行链接
add_executable(test main.cpp)

target_include_directories(test PRIVATE ${PROJ_INCLUDE_DIRS})
target_compile_options(test PRIVATE ${PROJ_COMPILE_OPTIONS})
target_link_libraries(test PRIVATE ${PROJ_LIBS})
```

有了编译条件，我们来看一下在程序中如何使用

```C++
#include <iostream>
#include "omp.h"

int main()
{
    #pragma omp parallel 
    std::cout << "hello world" << std::endl;
    return 0;
}
```
1. 在头文件中include `omp.h`
2. 在需要并行运行的代码块前添加语句`#pragma omp parallel`

我们将代码编译并运行后的结果：
![](pic\2023_6_13_1.jpg)
可见，一行代码被运行了16次，omp并行在正常运行，接下来使用到光追中吧



# 在光追中使用omp
光线追踪算法中，从相机出发，逐像素发射一系列光线，跟踪每一条光线得到像素点的值，遍历像素一般是用两个嵌套的for循环来做，这个过程显然可以进行并行计算。
```C++
for(int i = 0; i < Height; ++i)
{
    for(int j = 0; j < Width; ++j){
        // Generate Rays

        // for ray in Rays:   trace(ray) ...
    }
}
```
然而，根据上文的情况看来，使用`#pragma omp parallel`，会让这个嵌套for循环执行10次，并不会加速算法
正确方法是在语句后加上`for`，即可以对（嵌套）for循环的代码块进行并行运算。
```C++
#pragma omp parallel for
for(int i = 0; i < Height; ++i)
{
    for(int j = 0; j < Width; ++j){
        // Generate Rays

        // for ray in Rays:   trace(ray) ...
    }
}
```

**注意：for循环中的代码片段应是独立得到结果，互不干扰才能正常运行**

# 错误示例
我在做GAMES101的作业7时，在实现并行运算的过程中得到了这样的结果：
![](pic\2023_6_16_1.png)
原因是GAMES提供的框架中，在得到光线结果后，对framebuffer的取值用的是m，且每次循环结束时m++，这就导致不同进程相互干扰，得到错误的结果
```C++
for(int i = 0; i < Height; ++i)
{
    for(int j = 0; j < Width; ++j){
        // Generate Rays

        // for ray in Rays:   trace(ray) ...
        framebuffer[m] = trace(ray);
        m++;  //错误示范
    }
}
```


