# Boost 程序库完全开发指南

## 第零章 导读

本书的源码`https://github.com/chronolaw/boost_guide.git`

本书所用代码版本

- boost 1.72.0
- g++ 7.4.0
- Linux

## 第一章 Boost程序库总论



```c++
#include<boost/version.hpp>
#include<boost/config.hpp>
#include <iostream>
#include <string>
using namespace std;
int main()
{
    cout << BOOST_VERSION << endl;//Boost版本号
    cout << BOOST_LIB_VERSION << endl;//Boost版本号
    cout << BOOST_PLATFORM << endl;//操作系统
    cout << BOOST_COMPILER << endl;//编译器
    cout << BOOST_STDLIB << endl;//标注库
    return 0;
}
/*
108100
1_81
Win32
Microsoft Visual C++ version 1935
Dinkumware standard library version 650
*/
```

## 第二章 时间与日期

### 2.1 timer 库概述

### 2.2 timer

timer库提供简易的度量时间和进度显式功能，可以用于性能测试等需要计时的任务。

Boost 1.48版以后的timer库由两个组件组成：早期的timer （V1）和新的cpu_timer（V2），前者使用的是标准C/C++库函数，而后者则基于 chrono库使用操作系统的API，其计时精度更高。

```c++
#include<boost/timer.hpp>
#include<iostream>
using namespace std;
using namespace boost;
int main() {
	timer t;
	cout << "max timespan " << t.elapsed_max() / 3600 << "h" << endl;
	cout << "min timespan " << t.elapsed_min() << "s" << endl;
	cout << "now time elapsed " << t.elapsed() << "s" << endl;
}
/*
max timespan 596.523h
min timespan 0.001s
now time elapsed 0.002s
*/
```



timer源码

```c++
//  boost timer.hpp header file  ---------------------------------------------//

//  Copyright Beman Dawes 1994-99.  Distributed under the Boost
//  Software License, Version 1.0. (See accompanying file
//  LICENSE_1_0.txt or copy at http://www.boost.org/LICENSE_1_0.txt)

//  See http://www.boost.org/libs/timer for documentation.

//  Revision History
//  01 Apr 01  Modified to use new <boost/limits.hpp> header. (JMaddock)
//  12 Jan 01  Change to inline implementation to allow use without library
//             builds. See docs for more rationale. (Beman Dawes) 
//  25 Sep 99  elapsed_max() and elapsed_min() added (John Maddock)
//  16 Jul 99  Second beta
//   6 Jul 99  Initial boost version

#ifndef BOOST_TIMER_HPP
#define BOOST_TIMER_HPP

#include <boost/config/header_deprecated.hpp>
BOOST_HEADER_DEPRECATED( "the facilities in <boost/timer/timer.hpp>" )

#include <boost/config.hpp>
#include <ctime>
#include <boost/limits.hpp>

# ifdef BOOST_NO_STDC_NAMESPACE
    namespace std { using ::clock_t; using ::clock; }
# endif


namespace boost {

//  timer  -------------------------------------------------------------------//

//  A timer object measures elapsed time.

//  It is recommended that implementations measure wall clock rather than CPU
//  time since the intended use is performance measurement on systems where
//  total elapsed time is more important than just process or CPU time.

//  Warnings: The maximum measurable elapsed time may well be only 596.5+ hours
//  due to implementation limitations.  The accuracy of timings depends on the
//  accuracy of timing information provided by the underlying platform, and
//  this varies a great deal from platform to platform.

class timer
{
 public:
         timer() { _start_time = std::clock(); } // postcondition: elapsed()==0
//         timer( const timer& src );      // post: elapsed()==src.elapsed()
//        ~timer(){}
//  timer& operator=( const timer& src );  // post: elapsed()==src.elapsed()
  void   restart() { _start_time = std::clock(); } // post: elapsed()==0
  double elapsed() const                  // return elapsed time in seconds
    { return  double(std::clock() - _start_time) / CLOCKS_PER_SEC; }

  double elapsed_max() const   // return estimated maximum value for elapsed()
  // Portability warning: elapsed_max() may return too high a value on systems
  // where std::clock_t overflows or resets at surprising values.
  {
    return (double((std::numeric_limits<std::clock_t>::max)())
       - double(_start_time)) / double(CLOCKS_PER_SEC); 
  }

  double elapsed_min() const            // return minimum value for elapsed()
   { return double(1)/double(CLOCKS_PER_SEC); }

 private:
  std::clock_t _start_time;
}; // timer

} // namespace boost

#endif  // BOOST_TIMER_HPP

```

timer的计时使用了标准库头文件＜ctime＞里的std：：clock（）函数，它返回自进程启动以来的clock数，每秒的clock数则由宏CLOCKS_PER_SEC定义[2]。

函数 elapsed_min（）返回 timer 能够测量的最小时间单位，是CLOCKS_PER_SEC的倒数。函数 elapsed_max（）使用了标准库的数值极限类 numeric_limits，获得clock_t类型的最大值，该函数采用类似elapsed（）的方式计算能够测量的最大时间范围。

timer没有定义析构函数，这样做是正确且安全的。因为它仅有一个类型为clock_t的成员变量_start_time，故没有必要实现析构函数来特意释放资源（事实上，也无资源可供释放）。

### 2.3 progress_timer

### 2.4 data_time库概述

### 2.5 处理日期

### 2.6 处理时间

### 2.7 data_time库的高级议题

### 2.8 总结



## 第三章 内存管理

### 3.1 smart_ptr库概述

### 3.2 scoped_ptr

### 3.3 shared_ptr

### 3.4 weak_ptr

### 3.5 intrusive_ptr

### 3.6 pool库概述

### 3.7 pool

### 3.8 object_pool

### 3.9 singleton_pool

### 3.10 总结



## 第四章 实用工具

### 4.1 noncopyable

### 4.2 ignore_unused

### 4.3 optional

### 4.4 assign

### 4.5 tribool

### 4.6 operators

### 4.7 exception

### 4.8 uiid

### 4.9 config

### 4.10 utility

### 4.11 总结



## 第五章 字符串与文本处理

### 5.1 lexcal_cast

### 5.2 format

### 5.3 string_ref

### 5.4 string_algo

### 5.5 xpressive

### 5.6 总结



## 第六章 正确性与测试

### 6.1 assert

### 6.2 static_assert

### 6.3 lightweight_test

### 6.4 test

### 6.5 总结



## 第七章 容器和数据结构

### 7.1 array

### 7.2 dynamic_bitset

### 7.3 unordered

### 7.4 bimap

### 7.5 circular_buffer

### 7.6 tuple

### 7.7 any

### 7.8 variant

### 7.9 multi_array

### 7.10 property_tree

### 7.11 总结



## 第八章 算法

### 8.1 foreach

### 8.2 minmax

### 8.3 minmax_element

### 8.4 algorithm

### 8.5 总结



## 第九章 数学与数字

### 9.1 math.constants

### 9.2 integer

### 9.3 rational

### 9.4 ratio

### 9.5 crc

### 9.6 random

### 9.7 总结



## 第十章 操作系统相关

### 10.1 system

### 10.2 chrono

### 10.3 cpu_timer

### 10.4 filesystem

### 10.5 program_options

### 10.6 总结



## 第十一章 函数与回调

### 11.1 ref

### 11.2 bind

### 11.3 function

### 11.4 signals2

### 11.5 总结



## 第十二章 并发编程

### 12.1 atomic

### 12.2 thread

### 12.3 asio

### 12.4 总结



## 第十三章 组件速览

### 13.1 算法

### 13.2 字符串与文本处理

### 13.3 容器与数据结构

### 13.4 迭代器

### 13.5 函数对象与高级编程

### 13.6 泛型编程

### 13.7 模板元编程

### 13.8 预处理元编程

### 13.9 并发编程

### 13.10 数学与数学

### 13.11 输入输出

### 13.12 系统相关

### 13.13 语言特性模拟

### 13.14 杂项

### 13.15 总结

## 第十四章 设计模式

### 14.1 创建型模式

### 14.2 结构型模式

### 14.3 行为模式

### 14.4 其他模式

### 14.5 总结

## 第十五章 结束语

### 15.1 未臻完美

### 15.2 锦上添花

### 15.3 功夫在诗外

### 15.4 临别赠言



