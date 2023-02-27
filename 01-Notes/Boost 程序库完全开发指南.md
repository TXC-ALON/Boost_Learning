# Boost 程序库完全开发指南

## 一、Boost程序库总论



```c++
#include<boost/version.hpp>
#include<boost/config.hpp>
#include <iostream>
#include <string>
using namespace std;
int main()
{
    cout << BOOST_VERSION << endl;
    cout << BOOST_LIB_VERSION << endl;
    cout << BOOST_PLATFORM << endl;
    cout << BOOST_COMPILER << endl;
    cout << BOOST_STDLIB << endl;
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

## 二、时间与日期

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



