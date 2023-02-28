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
// Copyright (c) 2015-2017
// Author: Chrono Law
#ifndef _BOOST_GUIDE_STD_HPP
#define _BOOST_GUIDE_STD_HPP

#include <cassert>
#include <iostream>
#include <string>
#include <vector>
#include <set>
#include <map>
#include <algorithm>
#include <numeric>
//using namespace std;

#endif  //_BOOST_GUIDE_STD_HPP

```



```c++
#include<boost/timer.hpp>
#include "stdc++.h"
using namespace std;
using namespace boost;
int main() {
	timer t;
	cout << "max timespan " << t.elapsed_max() / 3600 << "h" << endl;
	cout << "min timespan " << t.elapsed_min()  << "s" << endl;
	cout << "now time elapsed "<<t.elapsed()<< "s" << endl;
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

timer的计时使用了标准库头文件＜ctime＞里的std::clock（）函数，它返回自进程启动以来的clock数，每秒的clock数则由宏CLOCKS_PER_SEC定义[2]。

函数 elapsed_min（）返回 timer 能够测量的最小时间单位，是CLOCKS_PER_SEC的倒数。函数 elapsed_max（）使用了标准库的数值极限类 numeric_limits，获得clock_t类型的最大值，该函数采用类似elapsed（）的方式计算能够测量的最大时间范围。

timer没有定义析构函数，这样做是正确且安全的。因为它仅有一个类型为clock_t的成员变量_start_time，故没有必要实现析构函数来特意释放资源（事实上，也无资源可供释放）。

### 2.3 progress_timer

progress_timer也是一个计时器，它派生自timer，会在析构时自动输出时间，省去了timer手动调用elapsed（）的工作，是一个相当方便的自动计时的小工具。

```c++
#include <boost/progress.hpp>
using namespace boost;

//////////////////////////////////////////

int main()
{
    {
        boost::progress_timer t;

    }
    {
        boost::progress_timer t;
    }

    stringstream ss;
    {
        progress_timer t(ss);
    }
    cout << ss.str();
}
```



progress_timer的定义

```c++
class progress_timer : public timer, private noncopyable
{
  
 public:
  explicit progress_timer( std::ostream & os = std::cout )
     // os is hint; implementation may ignore, particularly in embedded systems
     : timer(), noncopyable(), m_os(os) {}
  ~progress_timer()
  {
  //  A) Throwing an exception from a destructor is a Bad Thing.
  //  B) The progress_timer destructor does output which may throw.
  //  C) A progress_timer is usually not critical to the application.
  //  Therefore, wrap the I/O in a try block, catch and ignore all exceptions.
    try
    {
      // use istream instead of ios_base to workaround GNU problem (Greg Chicares)
      std::istream::fmtflags old_flags = m_os.setf( std::istream::fixed,
                                                   std::istream::floatfield );
      std::streamsize old_prec = m_os.precision( 2 );
      m_os << elapsed() << " s\n" // "s" is System International d'Unites std
                        << std::endl;
      m_os.flags( old_flags );
      m_os.precision( old_prec );
    }

    catch (...) {} // eat any exceptions
  } // ~progress_timer

 private:
  std::ostream & m_os;
};


//  progress_display  --------------------------------------------------------//

//  progress_display displays an appropriate indication of 
//  progress at an appropriate place in an appropriate form.

// NOTE: (Jan 12, 2001) Tried to change unsigned long to boost::uintmax_t, but
// found some compilers couldn't handle the required conversion to double.
// Reverted to unsigned long until the compilers catch up. 
```

在这个例子中，`boost::progress_timer` 对象被创建，并将 `std::stringstream` 对象 `ss` 作为其构造函数的参数传递。计时器对象被销毁时，它的输出结果将被写入到 `ss` 中。

然后，通过 `ss.str()` 方法，将 `std::stringstream` 对象中的字符串结果提取出来，并存储到一个 `std::string` 对象 `timing_result` 中。最后，这个结果被打印到控制台上。

```c++
#include <iostream>
#include <sstream>
#include <boost/progress.hpp>

int main() {
    std::stringstream ss;
    {
        boost::progress_timer t(ss);
        // do some work...
    }
    std::string timing_result = ss.str();
    std::cout << "Timing result: " << timing_result << std::endl;
    return 0;
}

```



### 2.4 data_time库概述

date_time库是一个非常全面且灵活的日期时间库，基于我们日常使用的公历（格里高利历），可以提供与时间相关的各种所需功能，如精确定义时间点、时间段和时间长度、加减若干天/月/年、日期迭代器等。date_time库还支持无限时间和无效时间这种在实际生活中有用的概念，而且它可以与C语言的传统时间结构tm相互转换，提供向下支持。

```c++
#include <boost/date_time/gregorian/gregorian.hpp>//处理日期的组件
using namespace boost::gregorian;
#include <boost/date_time/posix_time/posix_time.hpp>//处理时间的组件
using namespace boost::posix_time;
```



如果把时间想象成一个向前和向后都无限延伸的实数轴，那么时间点就是数轴上的一个点；时间段就是两个时间点之间一个确定的区间；时长（时间长度）则是一个有正负号的标量，是两个时间点之差，不属于数轴。

date_time库支持无限时间和无效时间（Not Available Date Time，NADT）这样特殊的时间概念，类似于数学中极限的含义。时间点和时长都有无限的值，它们的运算规则比较特别，如“+∞时间点+时长=+∞时间点”“时间点+∞时长=+∞时间点”。如果将正无限值与负无限值进行运算将有可能得到无效时间，如“+∞时长-∞时长=NADT”。
date_time库中用枚举special_values定义了这些特殊的时间概念，它位于名字空间boost::date_time，并被using语句引入其他子名字空间。

- pos_infin：表示正无限。
- neg_infin：表示负无限。
- not_a_date_time：无效时间。
- min_date_time：可表示的最小日期或时间。
- max_date_time：可表示的最大日期或时间。 

```c++
// Copyright (c) 2015
// Author: Chrono Law
#include "stdc++.h"
using namespace std;

//#define BOOST_DATE_TIME_POSIX_TIME_STD_CONFIG
#include <boost/date_time/gregorian/gregorian.hpp>
using namespace boost::gregorian;
#include <boost/date_time/posix_time/posix_time.hpp>
using namespace boost::posix_time;

//////////////////////////////////////////

void case1()
{
    date d(2014, 11, 3);
    date_facet* dfacet = new date_facet("%Y年%m月%d日");
    cout.imbue(locale(cout.getloc(), dfacet));
    cout << d << endl;

    time_facet* tfacet = new time_facet("%Y年%m月%d日%H点%M分%S%F秒");
    cout.imbue(locale(cout.getloc(), tfacet));
    cout << ptime(d, hours(21) + minutes(50) + millisec(100)) << endl;

}

//////////////////////////////////////////
days operator"" _D(unsigned long long n)
{
    return days(n);
}

weeks operator"" _W(unsigned long long n)
{
    return weeks(n);
}

date operator"" _YMD(const char* s, std::size_t)
{
    return from_string(s);
}

void case2()
{
    auto d = 100_D;
    auto w = 1_W;

    assert(d.days() == 100);
    assert(w.days() == (7_D).days());

    auto today = "2014/11/3"_YMD;
    cout << today << endl;
}

//////////////////////////////////////////
#include <boost/date_time/local_time/local_time.hpp>
using namespace boost::local_time;

void case3()
{
    tz_database tz_db;
    tz_db.load_from_file("E:/00Learning/C++/Boost/boost_guide-master/date_time/date_time_zonespec.csv");
    //tz_db.load_from_file("date_time_zonespec.csv");

    time_zone_ptr shz = tz_db.time_zone_from_region("Asia/Shanghai");

    time_zone_ptr sfz = tz_db.time_zone_from_region("America/Los_Angeles");

    cout << shz->has_dst() << endl;
    cout << shz->std_zone_name() << endl;

    local_date_time dt_bj(date(2014, 3, 6),
        hours(16),
        shz,
        false);
    cout << dt_bj << endl;

    time_duration flight_time = hours(12);
    dt_bj += flight_time;
    cout << dt_bj << endl;
    local_date_time dt_sf = dt_bj.local_time_in(sfz);
    cout << dt_sf;
}


//////////////////////////////////////////

int main()
{
    case1();
    case2();
    case3();
}
/*
2014年11月03日
2014年11月03日21点50分00.100000秒
2014年11月03日
0
CST
2014-Mar-06 16:00:00 CST
2014-Mar-07 04:00:00 CST
2014-Mar-06 12:00:00 PST
*/
```





### 2.5 处理日期

date_time库的日期基于格里高利历，支持从“1400-01-01”到“9999-12-31”之间的日期计算（很遗憾，它不能处理公元前的日期，不能用来研究古老的历史）。它位于名字空间`boost::gregorian`。

date是date_time库处理日期的核心类，使用一个32位的整数作为内部存储，以天为单位表示时间点概念。

```c++
  template<class T, class calendar, class duration_type_>
  class BOOST_SYMBOL_VISIBLE date : private
       boost::less_than_comparable<T
     , boost::equality_comparable<T
    > >
  {
  public:
    typedef T date_type;
    typedef calendar calendar_type;
    typedef typename calendar::date_traits_type traits_type;
    typedef duration_type_ duration_type;
    typedef typename calendar::year_type year_type;
    typedef typename calendar::month_type month_type;
    typedef typename calendar::day_type day_type;
    typedef typename calendar::ymd_type ymd_type;
    typedef typename calendar::date_rep_type date_rep_type;
    typedef typename calendar::date_int_type date_int_type;
    typedef typename calendar::day_of_week_type day_of_week_type;
    BOOST_CXX14_CONSTEXPR date(year_type y, month_type m, day_type d)
      : days_(calendar::day_number(ymd_type(y, m, d)))
    {}
    BOOST_CXX14_CONSTEXPR date(const ymd_type& ymd)
      : days_(calendar::day_number(ymd))
    {}
    //let the compiler write copy, assignment, and destructor
    BOOST_CXX14_CONSTEXPR year_type        year() const
    {
      ymd_type ymd = calendar::from_day_number(days_);
      return ymd.year;
    }
    BOOST_CXX14_CONSTEXPR month_type       month() const
    {
      ymd_type ymd = calendar::from_day_number(days_);
      return ymd.month;
    }
    BOOST_CXX14_CONSTEXPR day_type         day() const
    {
      ymd_type ymd = calendar::from_day_number(days_);
      return ymd.day;
    }
    BOOST_CXX14_CONSTEXPR day_of_week_type day_of_week() const
    {
      ymd_type ymd = calendar::from_day_number(days_);
      return calendar::day_of_week(ymd);
    }
    BOOST_CXX14_CONSTEXPR ymd_type         year_month_day() const
    {
      return calendar::from_day_number(days_);
    }
    BOOST_CONSTEXPR bool operator<(const date_type& rhs)  const
    {
      return days_ < rhs.days_;
    }
    BOOST_CONSTEXPR bool operator==(const date_type& rhs) const
    {
      return days_ == rhs.days_;
    }
    //! check to see if date is a special value
    BOOST_CONSTEXPR bool is_special()const
    {
      return(is_not_a_date() || is_infinity());
    }
    //! check to see if date is not a value
    BOOST_CONSTEXPR bool is_not_a_date()  const
    {
      return traits_type::is_not_a_number(days_);
    }
    //! check to see if date is one of the infinity values
    BOOST_CONSTEXPR bool is_infinity()  const
    {
      return traits_type::is_inf(days_);
    }
    //! check to see if date is greater than all possible dates
    BOOST_CONSTEXPR bool is_pos_infinity()  const
    {
      return traits_type::is_pos_inf(days_);
    }
    //! check to see if date is greater than all possible dates
    BOOST_CONSTEXPR bool is_neg_infinity()  const
    {
      return traits_type::is_neg_inf(days_);
    }
    //! return as a special value or a not_special if a normal date
    BOOST_CXX14_CONSTEXPR special_values as_special()  const
    {
      return traits_type::to_special(days_);
    }
    BOOST_CXX14_CONSTEXPR duration_type operator-(const date_type& d) const
    {
      if (!this->is_special() && !d.is_special())
      {
        // The duration underlying type may be wider than the date underlying type.
        // Thus we calculate the difference in terms of two durations from some common fixed base date.
        typedef typename duration_type::duration_rep_type duration_rep_type;
        return duration_type(static_cast< duration_rep_type >(days_) - static_cast< duration_rep_type >(d.days_));
      }
      else
      {
        // In this case the difference will be a special value, too
        date_rep_type val = date_rep_type(days_) - date_rep_type(d.days_);
        return duration_type(val.as_special());
      }
    }

    BOOST_CXX14_CONSTEXPR date_type operator-(const duration_type& dd) const
    {
      if(dd.is_special())
      {
        return date_type(date_rep_type(days_) - dd.get_rep());
      }
      return date_type(date_rep_type(days_) - static_cast<date_int_type>(dd.days()));
    }
    BOOST_CXX14_CONSTEXPR date_type operator-=(const duration_type& dd)
    {
      *this = *this - dd;
      return date_type(days_);
    }
    BOOST_CONSTEXPR date_rep_type day_count() const
    {
      return days_;
    }
    //allow internal access from operators
    BOOST_CXX14_CONSTEXPR date_type operator+(const duration_type& dd) const
    {
      if(dd.is_special())
      {
        return date_type(date_rep_type(days_) + dd.get_rep());
      }
      return date_type(date_rep_type(days_) + static_cast<date_int_type>(dd.days()));
    }
    BOOST_CXX14_CONSTEXPR date_type operator+=(const duration_type& dd)
    {
      *this = *this + dd;
      return date_type(days_);
    }

    //see reference
  protected:
    /*! This is a private constructor which allows for the creation of new
      dates.  It is not exposed to users since that would require class
      users to understand the inner workings of the date class.
    */
    BOOST_CONSTEXPR explicit date(date_int_type days) : days_(days) {}
    BOOST_CXX14_CONSTEXPR explicit date(date_rep_type days) : days_(days.as_number()) {}
    date_int_type days_;

  };

```

#### date是一个轻量级的对象

date是一个轻量级的对象，处理效率很高，可以被拷贝传值。date也全面支持比较操作和流输入输出，因此我们完全可以把它当成一个类似于int、string的基本类型来使用。

#### 创建日期对象

空的构造函数会创建一个值为not_a_date_time的无效日期；顺序传入年、月、日值则创建一个对应日期的date对象。

date 也允许从一个字符串产生，这需要使用工厂函数 from_string（）或 from_undelimited_string（）。前者使用分隔符（斜杠或连字符）分隔年、月、日格式的字符串，后者则是无分隔符的纯字符串。

day_clock是一个天级别的时钟，它也是一个工厂类，调用它的静态成员函数local_day（）或universal_day（）会返回一个当天的日期对象，分别是本地日期和UTC日期。day_clock内部使用了C标准库的函数`localtime（）`和`gmtime（）`，因此`local_day（）`的行为依赖操作系统的时区设置。

#### 访问日期

date的对外接口很像C语言中的tm结构，可以获取它保存的年、月、日、星期等成分，但date还提供了更多的操作。

- 成员函数year（）、month（）和day（）分别返回日期的年、月、日：
- 成员函数year_month_day（）返回一个date::ymd_type结构，可以一次性地获取年、月、日数据：
- 成员函数day_of_week（）返回date的星期数，0表示星期天。day_of_year（）返回date是当年的第几天（最大值是366）。
- end_of_month（）返回当月的最后一天的date对象：
- 成员函数week_number（）返回date所在的周是当年的第几周，其
  范围是0～53：

date还有5个is_xxx（）函数，用于检验日期是否是一个特殊日期，具体如下。

- `is_infinity（）`：是否是一个无限日期。
- `is_neg_infinity（）`：是否是一个负无限日期。
- `is_pos_infinity（）`：是否是一个正无限日期。
- `is_not_a_date（）`：是否是一个无效日期。
- `is_special（）`：是否是任意一个特殊日期。

> //! check to see if date is a special value
>     BOOST_CONSTEXPR bool is_special()const
>     {
>       return(is_not_a_date() || is_infinity());
>     }

#### 日期的输出

可以将date对象很方便地转换成字符串，它提供了3个自由函数。

- `to_simple_string（）`：转换为YYYY-mmm-DD 格式的字符串，其中，mmm为3字符的英文月份名。
- `to_iso_string（）`：转换为YYYYMMDD格式的数字字符串。
- `to_iso_extended_string（）`：转换为YYYY-MM-DD格式的数字字符串。
  date也支持流输入输出，默认使用YYYY-mmm-DD格式。

#### 转换C结构

date支持与C语言中的tm结构相互转换，转换的规则和函数如下。

- to_tm（date）：date转换到tm。将tm的时、分、秒成员（tm_
  hour/tm_min/tm_sec）均置为0，将夏令时标志tm_isdst置为-1（表示
  未知）。
- date_from_tm（datetm）：tm 转换到 date。只使用年、月、
  日3个成员（tm_year/tm_mon/tm_mday），其他成员均被忽略。



#### 日期长度 data_duration & days

日期长度是以天为单位的时长，是度量时间长度的一个标量。它与日期不同，其值可以是任意整数，可正可负。基本的日期长度类是date_duration

date_duration支持全序比较操作（==、！=、＜、＜=等），也支持完全的加减法和递增递减操作，用起来很像一个整数。此外date_duration还支持除法运算，可以除以一个整数，但不能除以另一个date_duration，它不支持其他的数学运算，如乘法、取模、取余等。
date_time库为date_duration定义了一个常用的typedef：days，这个新名字更好地说明了date_duration的含义——它可以用来计量天数。

#### 日期运算

date 支持加减运算，但两个date 对象的加法操作是无意义的（date_time库会以编译错误的方式通知我们），date主要用来与时长概念进行运算。

日期与特殊日期长度、特殊日期与日期长度进行运算的结果也是 特殊日期

在与months、years这两个时长类进行计算时要注意：如果日期是月末的最后一天，那么加减月或年会得到同样的月末时间，这是合乎生活常识的。但当天数是月末的28或29时，如果加减月份到2月份，那么随后的运算就总是月末操作，原来的天数信息就会丢失。例如：

```c++
{
        date d(2017,3,30);
        d -= months(1);
        d -= months(1);
        d += months(2);
        assert(d.day() == 31);//不再等于20170330
}
```

使用days则不会出现这样的问题，如果担心weeks、months、years这些时长类被无意使用进而扰乱了代码，可以 undef 宏 `BOOST_DATE_TIME_OPTIONAL_GREGORIAN_TYPES`，这将使date_time库不包含它们的定义头文件`＜boost/date_time/gregorian/greg_duration_types.hpp＞`。



#### 日期区间

date_time库使用date_period来表示日期区间的概念，它是时间轴上的一个左闭右开的区间，其端点是两个date对象。日期区间的左边界必须小于右边界，否则date_period将表示一个无效的日期区间。

date_period 可以使用成员函数判断某个日期是否在区间内，还可以计算日期区间的交集。

- is_before（）/is_after（）：日期区间是否在日期前或后。
- contains（）：日期区间是否包含另一个区间或日期。
- intersects（）：两个日期区间是否存在交集。
- intersection（）：返回两个区间的交集，如果无交集，则返回一个无效区间。
- is_adjacent（）：两个日期区间是否相邻。

date_period提供了两种并集操作。

- merge（）：返回两个日期区间的并集，如果日期区间无交集或不相邻，则返回无效区间。
- span（）：合并两个日期区间及两者间的间隔，相当于广义的merge（）。



#### 日期迭代器

日期迭代器的用法基本类似，都需要在构造时传入一个起始日期和增减步长（可以是一天、两周或N个月等，默认是1个单位），然后就可以用operator++、operator--变化日期。

#### 其他功能

boost::gregorian::gregorian_calendar 类提供了格里高利历的一些操作函数，这些操作函数基本上在被date类内部使用，用户很少会用到。但它也提供了几个有用的静态函数：成员函数is_leap_year（）可以判断年份是否是闰年；end_of_month_day（）可以给定年份和月份，并返回该月的最后一天。

```c++
#include "stdc++.h"
using namespace std;

//#define DATE_TIME_NO_DEFAULT_CONSTRUCTOR
#include <boost/date_time/gregorian/gregorian.hpp>
using namespace boost::gregorian;

//////////////////////////////////////////

void case1()
{
    date d1;
    date d2(2010, 1, 1);
    date d3(2000, Jan, 1);
    date d4(d2);

    assert(d1 == date(not_a_date_time));
    assert(d2 == d4);
    assert(d3 < d4);
}

//////////////////////////////////////////

void case2()
{
    date d1 = from_string("1999-12-31");
    date d2(from_string("2015/1/1"));
    date d3 = from_undelimited_string("20011118");

    cout << d1 << d2 << d3 << endl;

    cout << day_clock::local_day() << endl;
    cout << day_clock::universal_day() << endl;

}

//////////////////////////////////////////
void case3()
{
    date d1(neg_infin);
    date d2(pos_infin);
    date d3(not_a_date_time);
    date d4(max_date_time);
    date d5(min_date_time);

    cout << d1 << d2 << d3 << d4 << d5 << endl;

    try
    {
        //date d1(1399,12,1);
        //date d2(10000,1,1);
        date d3(2017, 2, 29);
    }
    catch (std::exception& e)
    {
        cout << e.what() << endl;
    }
}

//////////////////////////////////////////
void case4()
{
    date d(2017, 6, 1);
    assert(d.year() == 2017);
    assert(d.month() == 6);
    assert(d.day() == 1);

    date::ymd_type ymd = d.year_month_day();
    assert(ymd.year == 2017);
    assert(ymd.month == 6);
    assert(ymd.day == 1);

    cout << d.day_of_week() << endl;
    cout << d.day_of_year() << endl;
    assert(d.end_of_month() == date(2017, 6, 30));

    cout << date(2015, 1, 10).week_number() << endl;
    cout << date(2016, 1, 10).week_number() << endl;
    cout << date(2017, 1, 10).week_number() << endl;

    assert(date(pos_infin).is_infinity());
    assert(date(pos_infin).is_pos_infinity());
    assert(date(neg_infin).is_neg_infinity());
    assert(date(not_a_date_time).is_not_a_date());
    assert(date(not_a_date_time).is_special());
    assert(!date(2017, 5, 31).is_special());


}

//////////////////////////////////////////
void case5()
{
    date d(2017, 1, 23);

    cout << to_simple_string(d) << endl;
    cout << to_iso_string(d) << endl;
    cout << to_iso_extended_string(d) << endl;
    cout << d << endl;

    //cout << "input date:";
    //cin >>d;
    //cout << d;

}

//////////////////////////////////////////
void case6()
{
    date d(2017, 5, 20);
    tm t = to_tm(d);
    assert(t.tm_hour == 0 && t.tm_min == 0);
    assert(t.tm_year == 117 && t.tm_mday == 20);

    date d2 = date_from_tm(t);
    assert(d == d2);

}

//////////////////////////////////////////
void case7()
{
    days dd1(10), dd2(-100), dd3(255);

    assert(dd1 > dd2 && dd1 < dd3);
    assert(dd1 + dd2 == days(-90));
    assert((dd1 + dd3).days() == 265);
    assert(dd3 / 5 == days(51));

    weeks w(3);
    assert(w.days() == 21);

    months m(5);
    years y(2);

    months m2 = y + m;
    assert(m2.number_of_months() == 29);
    assert((y * 2).number_of_years() == 4);

}

//////////////////////////////////////////
void case8()
{
    date d1(2000, 1, 1), d2(2017, 11, 18);
    cout << d2 - d1 << endl;
    assert(d1 + (d2 - d1) == d2);

    d1 += days(10);
    assert(d1.day() == 11);
    d1 += months(2);
    assert(d1.month() == 3 && d1.day() == 11);
    d1 -= weeks(1);
    assert(d1.day() == 4);

    d2 -= years(10);
    assert(d2.year() == d1.year() + 7);

    {
        date d1(2017, 1, 1);

        date d2 = d1 + days(pos_infin);
        assert(d2.is_pos_infinity());

        d2 = d1 + days(not_a_date_time);
        assert(d2.is_not_a_date());
        d2 = date(neg_infin);
        days dd = d1 - d2;
        assert(dd.is_special() && !dd.is_negative());
    }

    {
        date d(2017, 3, 30);
        d -= months(1);
        d -= months(1);
        d += months(2);
        assert(d.day() == 31);
    }
}

//////////////////////////////////////////
void case9()
{
    date_period dp1(date(2017, 1, 1), days(20));
    date_period dp2(date(2017, 1, 1), date(2016, 1, 1));
    date_period dp3(date(2017, 3, 1), days(-20));

    date_period dp(date(2017, 1, 1), days(20));

    assert(!dp.is_null());
    assert(dp.begin().day() == 1);
    assert(dp.last().day() == 20);
    assert(dp.end().day() == 21);
    assert(dp.length().days() == 20);

    {
        date_period dp1(date(2017, 1, 1), days(20));
        date_period dp2(date(2017, 2, 19), days(10));

        cout << dp1;                        //[2010-Jan-01/2010-Jan-20]
        assert(dp1 < dp2);
    }
}

//////////////////////////////////////////

int main()
{
    cout << "---1" << endl;
    case1();
    cout << "---2" << endl;
    case2();
    cout << "---3" << endl;
    case3();
    cout << "---4" << endl;
    case4();
    cout << "---5" << endl;
    case5();
    cout << "---6" << endl;
    case6();
    cout << "---7" << endl;
    case7();
    cout << "---8" << endl;
    case8();
    cout << "---9" << endl;
    case9();
}


/*
---1
---2
1999-Dec-312015-Jan-012001-Nov-18
2023-Feb-28
2023-Feb-28
---3
-infinity+infinitynot-a-date-time9999-Dec-311400-Jan-01
Day of month is not valid for year
---4
Thu
152
2
1
2
---5
2017-Jan-23
20170123
2017-01-23
2017-Jan-23
---6
---7
---8
6531
---9
[2017-Jan-01/2017-Jan-20]
*/
```



### 2.6 处理时间 date_time

date_time库在格里高利历的基础上提供了微秒级别的时间系统，如果需要，它最高可以达到纳秒级别的精确度。

### 2.7 data_time库的高级议题

#### 编译配置宏

#### 自定义字面值

#### 格式化时间

#### 本地时间

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

 C/C++只提供了有限的正确性验证/测试支持——assert 宏，这远远不够。C++标准中的`std::exception` 能够处理运行时的异常，但并不能检查代码的逻辑，缺乏足够的、语言级别的工具来保证软件的正确性，这使程序员很容易陷入与bug搏斗的泥沼中。Boost在这方面进行了改进：`boost.assert`库增强了原始的运行时的assert宏，`static_assert`库提供了静态断言（编译期诊断），而`lightweight_test`和`test`库则构建了完整的单元测试框架。

### 6.1 assert

boost.assert提供的主要工具是BOOST_ASSERT宏，它类似于C标准中的assert，提供运行时的断言，但其功能有所增强。为了使用boost.assert，需要包含的头文件如下：`#include ＜boost/assert.hpp>`

#### 基本用法

assert库定义了两个断言宏

```c++
# define BOOST_ASSERT(expr) ((void)0)
# define BOOST_ASSERT_MSG(expr, msg) ((void)0)

# define BOOST_ASSERT_IS_VOID
```



第一种形式的BOOST_ASSERT等同于assert宏，断言表达式为真。
第二种形式则允许断言失败时输出描述字符串，有助于排错。

宏的参数expr表达式可以是任意（合法的）C++表达式，从简单的关系比较到复杂的函数嵌套调用都可以。如果表达式值为true，那么断言成立，程序会继续向下执行，否则断言会引发一个异常，在终端上输出调试信息并终止程序执行

#### 禁用断言

BOOST_ASSERT是标准断言宏assert的增强版，因此它有更强的灵活性。如果在头文件＜boost/assert.hpp＞之前定义了宏BOOST_DISABLE_ASSERTS，那么BOOST_ASSERT将会定义为“（（void）0）”，自动失效。但标准的assert宏并不会受影响，这可以让程序员有选择地关闭BOOST_ASSERT。

#### 拓展用法

如果在头文件＜boost/assert.hpp＞之前定义了宏BOOST_ENABLE_ASSERT_HANDLER，将导致BOOST_ASSERT的行为发生改变：它将不再等同于assert宏，断言的表达式无论是在debug 模式下还是在release模式下，都将被求值。如果断言失败，会发生一个断言失败的函数调用`boost::assertion_failed()`或 `assertion_failed_msg()`——这相当于提供了一个错误处理handler。
上述两个函数声明在boost名字空间里，但它们特意被设计为没有具体实现，其声明如下：当断言失败时，BOOST_ASSERT宏会把断言表达式字符串、调用函数名（使用BOOST_CURRENT_FUNCTION，参见4.10.2节）、所在源文件名和行号都传递给 assertion_failed（）函数处理。用户需要自己实现assertion_failed（）函数，以恰当的方式处理错误——通常是记录日志或抛出异常。

```c++
// Copyright (c) 2015
// Author: Chrono Law
#include <cstring>
#include "stdc++.h"
using namespace std;

//#define BOOST_DISABLE_ASSERTS
#define BOOST_ENABLE_ASSERT_HANDLER
#include <boost/assert.hpp>

//////////////////////////////////////////

double func(int x)
{
    BOOST_ASSERT_MSG(x != 0, "divided by zero");
    return 1.0 / x;
}

double func2(int x)
{
    BOOST_ASSERT(x != 0 && "divided by zero");
    cout << "after BOOST_ASSERT" << endl;

    assert(x != 0 && "divided by zero");
    cout << "after" << endl;

    return 1.0 / x;
}

void case1()
{
    BOOST_ASSERT(16 == 0x10);
    //BOOST_ASSERT(string().size() == 1);

    func(0);
    //func2(0);

    int len;
    BOOST_VERIFY(len = strlen("123"));
    assert(len == 3);
}

//////////////////////////////////////////

#include <boost/format.hpp>
namespace boost {
    void    assertion_failed(char const*, char const*, char const*, long) {}
    void assertion_failed_msg(char const* expr, char const* msg,
        char const* function,
        char const* file, long line)
    {
        boost::format fmt("Assertion failed!\nExpression: %s\n"
            "Function: %s\nFile: %s\nLine: %ld\n"
            "Msg: %s\n\n");
        fmt% expr% function% file% line% msg;
        cout << fmt;
    }
}

int main()
{
    case1();
}

/*
Assertion failed!
Expression: x != 0
Function: double __cdecl func(int)
File: D:\Vstudio\source\repos\Boost-Learning\Boost-Learning\Boost-Learning.cpp
Line: 15
Msg: divided by zero
*/
```



### 6.2 static_assert

static_assert库把断言的诊断时刻由运行期提前到编译期，让编译器检查可能发生的错误，这样能够更好地增加程序的正确性，它模拟实现了C++标准里的static_assert关键字。

BOOST_STATIC_ASSERT 使用了较复杂的技术，但简单来说，它实际上最终是一个typedef，因此在编译时它同样不会产生任何代码和数据，对运行效率不会有任何影响——无论是debug模式还是release模式。

BOOST_STATIC_ASSERT是一个编译期断言，使用了模板元编程技术实现，虽然在很多方面它都BOOST_ASSERT相似，但其用法还是有所不同的。最重要的区别在于使用范围：BOOST_ASSERT（expr）必须是
一个能够执行的语句，它只能在函数域里出现；而BOOST_STATIC_ASSERT则可以出现在程序的任何位置，如名字空间域、类域或函数域。

BOOST_STATIC_ASSERT 在类域和名字空间域的使用方式与在函数域的使用方式相同。例

```c++
namespace my_space
{
    class empty_class
    {
        BOOST_STATIC_ASSERT_MSG(sizeof(int) >= 4, "for 32 bit");
    };

    BOOST_STATIC_ASSERT(sizeof(empty_class) == 1);
}
```



“空类”其实并不空，因为C++不允许大小为0的类或对象存在，通常情况下，“空类”会由编译器在里面安插一个类型为char 的成员变量，令它有一个确定的大小。

### 6.3 lightweight_test

#### 测试断言

lightweight_test只提供最基本的单元测试功能，不支持测试用
例、测试套件的概念，但因为它简单小巧，所以它适用于要求不高或
快速的测试工作。
lightweight_test定义了数个测试断言，以下列出较常用的几个。

- BOOST_TEST（e）：断言表达式成立。
- BOOST_TEST_NOT（e）：断言表达式不成立。
- BOOST_ERROR（s）：直接断言失败，输出错误消息s。
- BOOST_TEST_EQ（e1，e2）：断言两个表达式相等。
- BOOST_TEST_NE（e1，e2）：断言两个表达式不等。
- BOOST_TEST_CSTR_EQ（e1，e2）：断言两个C字符串相等。
- BOOST_TEST_CSTR_NE（e1，e2）：断言两个C字符串不相等。
- BOOST_TEST_THROWS（e，ex）：断言表达式e抛出异常ex。

如果以上断言失败，就会增加内部的错误计数，lightweight_test 提供函数boost::report_errors（）来输出测试结果，在测试结束时，我们必须调用report_errors（），否则会发生BOOST_ASSERT断言错误。

#### 用法

lightweight_test不需要编译，也不需要特定的入口函数，测试断言可以用在程序里的任何地方，就像使用assert 一样，但不要忘记在main 函数的最后调用report_errors（）函数。

```c++
// Copyright (c) 2015
// Author: Chrono Law
#include <type_traits>
#include "stdc++.h"
using namespace std;

#include <boost/smart_ptr.hpp>
#include <boost/core/lightweight_test.hpp>
#include <boost/core/lightweight_test_trait.hpp>
//using namespace boost;

int main()
{
       auto p = std::make_shared<int>(10);  // 创建一个值为10的shared_ptr

    BOOST_TEST(*p == 10);  // 检查shared_ptr中存储的值是否为10
    BOOST_TEST(p.unique());  // 检查shared_ptr的引用计数是否为1
    BOOST_TEST_NOT(!p);  // 检查shared_ptr是否非空

    BOOST_TEST_EQ(p.use_count(), 1);  // 检查shared_ptr的引用计数是否为1
    BOOST_TEST_NE(*p, 20);  // 检查shared_ptr中存储的值是否不等于20

    p.reset();  // 重置shared_ptr
    BOOST_TEST(!p);  // 检查shared_ptr是否为空

    //BOOST_TEST_THROWS(*p, std::runtime_error);  // 检查是否抛出异常
    //BOOST_ERROR("error accured!!");  // 抛出测试失败的异常

    BOOST_TEST_TRAIT_TRUE((std::is_integral<int>));  // 检查int是否是整数类型
    BOOST_TEST_TRAIT_FALSE((std::is_function<int>));  // 检查int类型不是函数类型

    return boost::report_errors();
}
/*
No errors detected.
*/
```



> ### 单元测试的基本要素和步骤
>
> C++中单元测试的基本要素包括：
>
> 1. 测试框架：选择一个适合你的项目的测试框架，如Google Test、Catch2等。
> 2. 测试用例：编写测试用例，测试代码的各种情况，如正常情况、边界情况、异常情况等。
> 3. 断言：使用断言来验证测试结果，确保代码运行符合预期。
> 4. 模拟对象：当测试代码依赖于其他模块或组件时，使用模拟对象模拟这些组件的行为。
>
> 下面是C++中单元测试的步骤：
>
> 1. 确定要测试的代码模块或函数。
> 2. 编写测试用例，覆盖代码的各种情况。
> 3. 使用断言验证测试结果是否正确。
> 4. 运行测试用例，检查测试是否通过。
> 5. 分析测试结果，找出测试失败的原因。
> 6. 修复代码中的问题，再次运行测试。
> 7. 重复步骤3至6，直到测试通过。
>
> 最终的测试结果应该注意以下因素：
>
> 1. 测试覆盖率：检查测试是否涵盖了代码的各种情况。
> 2. 测试结果准确性：确保测试结果正确反映了代码的实际行为。
> 3. 性能测试：在需要高性能的代码模块中，进行性能测试。
> 4. 稳定性测试：测试代码的稳定性，确保在不同的环境和负载下都能正常工作。
> 5. 安全性测试：测试代码的安全性，确保代码没有安全漏洞。

#### 测试元编程

lightweight_test库也提供了对元编程（可参考推荐书目[3]）测试的有限支持，在头文件＜boost/core/lightweight_test_trait.hpp＞里定义了两个编译期的断言：

```c++
#define BOOST_TEST_TRAIT_TRUE((type));
#define BOOST_TEST_TRAIT_FALSE((type));
```

这两个宏的效果与BOOST_STATIC_ASSERT类似，但它们判断的是type::value，而不是type本身，所以宏的参数type应该是能够返回bool值的元函数。还需要特别注意：由于内部实现的原因，type必须要使用圆括号包围（宏展开为一个模板函数的参数）。

### 6.4 test

test库提供了一个用于单元测试、基于命令行界面的测试套件：Unit Test Framework（简称UTF），它比其他的单元测试库更强大、更方便、更好用。

它不仅支持简单的测试，也支持全面的单元测试，并且还具有程序运行监控和检测内存泄漏的功能，是一个用于保证程序正确性的强大工具，为软件开发的基本领域——单元测试提供了简单而富有弹性的解决方案，可以满足开发人员从高到低的各种需求。其优点如下。
- 易于理解，任何人都可以轻易地构建单元测试模块。
- 提供测试用例、测试套件的概念，并能够以任意的复杂度组织
它们。
- 提供丰富的测试断言，能够处理各种情况，包括C++异常。
- 可以轻易地初始化测试用例、测试套件或整个测试程序。
- 可以显示测试进度，对于大型测试非常适用。
- 测试信息可以显示为多种格式，如普通文本或XML格式。
- 支持命令行，可以指定运行任意一个测试套件或测试用例。
- 还有许多更高级的用法。



#### 测试断言

在test库中，测试断言是一组命名清楚的宏，它们的用法类似 BO
OST_ASSERT，断言测试通过。如果测试失败，则会记录出错的文件
名、行号及错误信息。
test库提供4个最基本的测试断言。
- BOOST_CHECK（e）：断言测试通过，如不通过也不影响程序执
行。
- BOOST_REQUIRE（e）：要求测试必须通过，否则程序停止执
行。
- BOOST_ERROR（s）：给出一个错误信息，程序继续执行。
- BOOST_FAIL（s）：给出一个错误信息，程序运行终止。
上述4个测试断言是最基本的测试断言，能够在任何地方使用，同时由于通用性，它们也不具有其他测试断言的好处，我们应当尽量少使用它们。

test库其他测试断言的形式是BOOST_LVL_XXX，如BOOST_CHECK_EQUAL、BOOST_WARN_GT，详细的命名规则如下。
- BOOST：遵循Boost库的命名规则，宏一律以大写的BOOST开头。
- LVL：断言的级别。WARN是警告级，不影响程序运行，也不增加错误数量；CHECK 是检查级别，如果断言失败则增加错误数量，但不影响程序运行；REQUIRE 是最高的级别，如果断言失败将增加错误数量并终止程序运行。最常用的断言级别是CHECK，WARN可以用于不涉及程序关键功能的测试，只有当断言失败会导致程序无法继续进行测试时才能使用REQUIRE。
- XXX：各种具体的测试断言，如断言相等/不等、抛出/不抛出异常、大于/小于等。

test库中常用的几个测试断言如下。
- BOOST_LVL_EQUAL（l，r）：检测l==r，当测试失败时会给出详细的信息。
它不能用于浮点数的比较，浮点数的相等比较应使用BOOST_LVL_CLOSE。

- BOOST_LVL_GE（l，r）：检测l＞=r，与GT（l＞r）、LT（l＜r）、LE（l＜=r）和NE（l！=r）相同，它们用于测试各种不等性。
- BOOST_LVL_THROW（e，ex）：检测表达式e，抛出指定的ex异常。
- BOOST_LVL_NO_THROW（e）：检测表达式e，不抛出任何异常。
- BOOST_LVL_MESSAGE（e，msg）：它与不带MESSAGE后缀的断言的功能相同，但测试失败时，它会给出指定的消息。
- BOOST_TEST_MESSAGE（msg）：它仅输出通知信息，不含有任何警告或错误信息，默认情况不会显示。在之后的几节中，将会看到这些测试断言的具体用法。



#### 测试主体

单元测试领域有很多专有概念，本节仅介绍test库中比较重要的几个。
test库将测试程序定义为一个测试模块，由测试安装、测试主体、测试清理和测试运行器四部分组成。测试主体是测试模块的实际运行部分，由测试用例和测试套件组织成测试树的形式。

### 6.5 总结

很多软件方法学（如敏捷开发、极限编程）都非常强调测试的重要性，使用Boost测试库，我们就可以实践这些方法。
assert库提供一个assert宏的增强版本BOOST_ASSERT，它的默认行为与assert相同，但其规范的大写形式有助于进行代码维护。如果定义了配置宏 BOOST_ENABLE_ASSERT_HANDLER，BOOST_ASSERT 就给断言安装了一个处理 handler，能够以任意方式处理断言失败的情形。static_assert库模拟实现了C++标准中的static_assert关键字，是编译期的断言，计算编译期表达式的值，在泛型编程和模板元编程领域非常有用。如果读者正在使用模板技术编写泛型算法或泛型类，那么应该仔细地研究它的用法，它可以保证程序中的模板函数或模板类像预期的那样工作。
lightweight_test是一个轻量级的单元测试框架，它不需要编译就可以使用，提供了基本的测试功能，很容易使用，但不适合大多数实际的软件开发项目。
本章的重点内容是test库，它实现了一个完整的单元测试框架UTF，不仅可以测试普通的函数和类，也可以测试模板函数和模板类，这是其他类似功能的单元测试工具所不具备的。
UTF定义了完整的单元测试框架，提供各种测试概念，包括测试安装/清理、测试套件、测试用例、测试断言等，其用法非常自由，支持手动和自动两种向框架注册的方式，在通常情况下，自动方式是最佳的选择，可以简化测试代码的编写工作。
UTF在测试的输出方面也很灵活，输出日志有详细的级别设定，通过运行参数可以指定允许输出的级别，日志也可以选择多种格式。运行参数还能控制单元测试的其他方面，如用彩色显示、测试进度、输出系统消息等。

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



