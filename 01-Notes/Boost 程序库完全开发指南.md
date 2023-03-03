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

Boost 1.48版以后的timer库由两个组件组成:早期的timer (V1) 和新的cpu_timer(V2) ，前者使用的是标准C/C++库函数，而后者则基于 chrono库使用操作系统的API，其计时精度更高。

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

timer的计时使用了标准库头文件＜ctime＞里的std::clock() 函数，它返回自进程启动以来的clock数，每秒的clock数则由宏CLOCKS_PER_SEC定义[2]。

函数 elapsed_min() 返回 timer 能够测量的最小时间单位，是CLOCKS_PER_SEC的倒数。函数 elapsed_max() 使用了标准库的数值极限类 numeric_limits，获得clock_t类型的最大值，该函数采用类似elapsed() 的方式计算能够测量的最大时间范围。

timer没有定义析构函数，这样做是正确且安全的。因为它仅有一个类型为clock_t的成员变量_start_time，故没有必要实现析构函数来特意释放资源(事实上，也无资源可供释放) 。

### 2.3 progress_timer

progress_timer也是一个计时器，它派生自timer，会在析构时自动输出时间，省去了timer手动调用elapsed() 的工作，是一个相当方便的自动计时的小工具。

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

date_time库是一个非常全面且灵活的日期时间库，基于我们日常使用的公历(格里高利历) ，可以提供与时间相关的各种所需功能，如精确定义时间点、时间段和时间长度、加减若干天/月/年、日期迭代器等。date_time库还支持无限时间和无效时间这种在实际生活中有用的概念，而且它可以与C语言的传统时间结构tm相互转换，提供向下支持。

```c++
#include <boost/date_time/gregorian/gregorian.hpp>//处理日期的组件
using namespace boost::gregorian;
#include <boost/date_time/posix_time/posix_time.hpp>//处理时间的组件
using namespace boost::posix_time;
```



如果把时间想象成一个向前和向后都无限延伸的实数轴，那么时间点就是数轴上的一个点；时间段就是两个时间点之间一个确定的区间；时长(时间长度) 则是一个有正负号的标量，是两个时间点之差，不属于数轴。

date_time库支持无限时间和无效时间(Not Available Date Time，NADT) 这样特殊的时间概念，类似于数学中极限的含义。时间点和时长都有无限的值，它们的运算规则比较特别，如“+∞时间点+时长=+∞时间点”“时间点+∞时长=+∞时间点”。如果将正无限值与负无限值进行运算将有可能得到无效时间，如“+∞时长-∞时长=NADT”。
date_time库中用枚举special_values定义了这些特殊的时间概念，它位于名字空间boost::date_time，并被using语句引入其他子名字空间。

- pos_infin:表示正无限。
- neg_infin:表示负无限。
- not_a_date_time:无效时间。
- min_date_time:可表示的最小日期或时间。
- max_date_time:可表示的最大日期或时间。 

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

date_time库的日期基于格里高利历，支持从“1400-01-01”到“9999-12-31”之间的日期计算(很遗憾，它不能处理公元前的日期，不能用来研究古老的历史) 。它位于名字空间`boost::gregorian`。

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

date 也允许从一个字符串产生，这需要使用工厂函数 from_string() 或 from_undelimited_string() 。前者使用分隔符(斜杠或连字符) 分隔年、月、日格式的字符串，后者则是无分隔符的纯字符串。

day_clock是一个天级别的时钟，它也是一个工厂类，调用它的静态成员函数local_day() 或universal_day() 会返回一个当天的日期对象，分别是本地日期和UTC日期。day_clock内部使用了C标准库的函数`localtime() `和`gmtime() `，因此`local_day() `的行为依赖操作系统的时区设置。

#### 访问日期

date的对外接口很像C语言中的tm结构，可以获取它保存的年、月、日、星期等成分，但date还提供了更多的操作。

- 成员函数year() 、month() 和day() 分别返回日期的年、月、日:
- 成员函数year_month_day() 返回一个date::ymd_type结构，可以一次性地获取年、月、日数据:
- 成员函数day_of_week() 返回date的星期数，0表示星期天。day_of_year() 返回date是当年的第几天(最大值是366) 。
- end_of_month() 返回当月的最后一天的date对象:
- 成员函数week_number() 返回date所在的周是当年的第几周，其
  范围是0～53:

date还有5个is_xxx() 函数，用于检验日期是否是一个特殊日期，具体如下。

- `is_infinity() `:是否是一个无限日期。
- `is_neg_infinity() `:是否是一个负无限日期。
- `is_pos_infinity() `:是否是一个正无限日期。
- `is_not_a_date() `:是否是一个无效日期。
- `is_special() `:是否是任意一个特殊日期。

> //! check to see if date is a special value
>     BOOST_CONSTEXPR bool is_special()const
>     {
>       return(is_not_a_date() || is_infinity());
>     }

#### 日期的输出

可以将date对象很方便地转换成字符串，它提供了3个自由函数。

- `to_simple_string() `:转换为YYYY-mmm-DD 格式的字符串，其中，mmm为3字符的英文月份名。
- `to_iso_string() `:转换为YYYYMMDD格式的数字字符串。
- `to_iso_extended_string() `:转换为YYYY-MM-DD格式的数字字符串。
  date也支持流输入输出，默认使用YYYY-mmm-DD格式。

#### 转换C结构

date支持与C语言中的tm结构相互转换，转换的规则和函数如下。

- to_tm(date) :date转换到tm。将tm的时、分、秒成员(tm_
  hour/tm_min/tm_sec) 均置为0，将夏令时标志tm_isdst置为-1(表示
  未知) 。
- date_from_tm(datetm) :tm 转换到 date。只使用年、月、
  日3个成员(tm_year/tm_mon/tm_mday) ，其他成员均被忽略。



#### 日期长度 data_duration & days

日期长度是以天为单位的时长，是度量时间长度的一个标量。它与日期不同，其值可以是任意整数，可正可负。基本的日期长度类是date_duration

date_duration支持全序比较操作(==、!=、＜、＜=等) ，也支持完全的加减法和递增递减操作，用起来很像一个整数。此外date_duration还支持除法运算，可以除以一个整数，但不能除以另一个date_duration，它不支持其他的数学运算，如乘法、取模、取余等。
date_time库为date_duration定义了一个常用的typedef:days，这个新名字更好地说明了date_duration的含义——它可以用来计量天数。

#### 日期运算

date 支持加减运算，但两个date 对象的加法操作是无意义的(date_time库会以编译错误的方式通知我们) ，date主要用来与时长概念进行运算。

日期与特殊日期长度、特殊日期与日期长度进行运算的结果也是 特殊日期

在与months、years这两个时长类进行计算时要注意:如果日期是月末的最后一天，那么加减月或年会得到同样的月末时间，这是合乎生活常识的。但当天数是月末的28或29时，如果加减月份到2月份，那么随后的运算就总是月末操作，原来的天数信息就会丢失。例如:

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

- is_before() /is_after() :日期区间是否在日期前或后。
- contains() :日期区间是否包含另一个区间或日期。
- intersects() :两个日期区间是否存在交集。
- intersection() :返回两个区间的交集，如果无交集，则返回一个无效区间。
- is_adjacent() :两个日期区间是否相邻。

date_period提供了两种并集操作。

- merge() :返回两个日期区间的并集，如果日期区间无交集或不相邻，则返回无效区间。
- span() :合并两个日期区间及两者间的间隔，相当于广义的merge() 。



#### 日期迭代器

日期迭代器的用法基本类似，都需要在构造时传入一个起始日期和增减步长(可以是一天、两周或N个月等，默认是1个单位) ，然后就可以用operator++、operator--变化日期。

#### 其他功能

boost::gregorian::gregorian_calendar 类提供了格里高利历的一些操作函数，这些操作函数基本上在被date类内部使用，用户很少会用到。但它也提供了几个有用的静态函数:成员函数is_leap_year() 可以判断年份是否是闰年；end_of_month_day() 可以给定年份和月份，并返回该月的最后一天。

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

 C/C++只提供了有限的正确性验证/测试支持——assert 宏，这远远不够。C++标准中的`std::exception` 能够处理运行时的异常，但并不能检查代码的逻辑，缺乏足够的、语言级别的工具来保证软件的正确性，这使程序员很容易陷入与bug搏斗的泥沼中。Boost在这方面进行了改进:`boost.assert`库增强了原始的运行时的assert宏，`static_assert`库提供了静态断言(编译期诊断) ，而`lightweight_test`和`test`库则构建了完整的单元测试框架。

### 6.1 assert

boost.assert提供的主要工具是BOOST_ASSERT宏，它类似于C标准中的assert，提供运行时的断言，但其功能有所增强。为了使用boost.assert，需要包含的头文件如下:`#include ＜boost/assert.hpp>`

#### 基本用法

assert库定义了两个断言宏

```c++
# define BOOST_ASSERT(expr) ((void)0)
# define BOOST_ASSERT_MSG(expr, msg) ((void)0)

# define BOOST_ASSERT_IS_VOID
```



第一种形式的BOOST_ASSERT等同于assert宏，断言表达式为真。
第二种形式则允许断言失败时输出描述字符串，有助于排错。

宏的参数expr表达式可以是任意(合法的) C++表达式，从简单的关系比较到复杂的函数嵌套调用都可以。如果表达式值为true，那么断言成立，程序会继续向下执行，否则断言会引发一个异常，在终端上输出调试信息并终止程序执行

#### 禁用断言

BOOST_ASSERT是标准断言宏assert的增强版，因此它有更强的灵活性。如果在头文件＜boost/assert.hpp＞之前定义了宏BOOST_DISABLE_ASSERTS，那么BOOST_ASSERT将会定义为“((void) 0) ”，自动失效。但标准的assert宏并不会受影响，这可以让程序员有选择地关闭BOOST_ASSERT。

#### 拓展用法

如果在头文件＜boost/assert.hpp＞之前定义了宏BOOST_ENABLE_ASSERT_HANDLER，将导致BOOST_ASSERT的行为发生改变:它将不再等同于assert宏，断言的表达式无论是在debug 模式下还是在release模式下，都将被求值。如果断言失败，会发生一个断言失败的函数调用`boost::assertion_failed()`或 `assertion_failed_msg()`——这相当于提供了一个错误处理handler。
上述两个函数声明在boost名字空间里，但它们特意被设计为没有具体实现，其声明如下:当断言失败时，BOOST_ASSERT宏会把断言表达式字符串、调用函数名(使用BOOST_CURRENT_FUNCTION，参见4.10.2节) 、所在源文件名和行号都传递给 assertion_failed() 函数处理。用户需要自己实现assertion_failed() 函数，以恰当的方式处理错误——通常是记录日志或抛出异常。

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

BOOST_STATIC_ASSERT是一个编译期断言，使用了模板元编程技术实现，虽然在很多方面它都BOOST_ASSERT相似，但其用法还是有所不同的。最重要的区别在于使用范围:BOOST_ASSERT(expr) 必须是
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

- BOOST_TEST(e) :断言表达式成立。
- BOOST_TEST_NOT(e) :断言表达式不成立。
- BOOST_ERROR(s) :直接断言失败，输出错误消息s。
- BOOST_TEST_EQ(e1，e2) :断言两个表达式相等。
- BOOST_TEST_NE(e1，e2) :断言两个表达式不等。
- BOOST_TEST_CSTR_EQ(e1，e2) :断言两个C字符串相等。
- BOOST_TEST_CSTR_NE(e1，e2) :断言两个C字符串不相等。
- BOOST_TEST_THROWS(e，ex) :断言表达式e抛出异常ex。

如果以上断言失败，就会增加内部的错误计数，lightweight_test 提供函数boost::report_errors() 来输出测试结果，在测试结束时，我们必须调用report_errors() ，否则会发生BOOST_ASSERT断言错误。

#### 用法

lightweight_test不需要编译，也不需要特定的入口函数，测试断言可以用在程序里的任何地方，就像使用assert 一样，但不要忘记在main 函数的最后调用report_errors() 函数。

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
> C++中单元测试的基本要素包括:
>
> 1. 测试框架:选择一个适合你的项目的测试框架，如Google Test、Catch2等。
> 2. 测试用例:编写测试用例，测试代码的各种情况，如正常情况、边界情况、异常情况等。
> 3. 断言:使用断言来验证测试结果，确保代码运行符合预期。
> 4. 模拟对象:当测试代码依赖于其他模块或组件时，使用模拟对象模拟这些组件的行为。
>
> 下面是C++中单元测试的步骤:
>
> 1. 确定要测试的代码模块或函数。
> 2. 编写测试用例，覆盖代码的各种情况。
> 3. 使用断言验证测试结果是否正确。
> 4. 运行测试用例，检查测试是否通过。
> 5. 分析测试结果，找出测试失败的原因。
> 6. 修复代码中的问题，再次运行测试。
> 7. 重复步骤3至6，直到测试通过。
>
> 最终的测试结果应该注意以下因素:
>
> 1. 测试覆盖率:检查测试是否涵盖了代码的各种情况。
> 2. 测试结果准确性:确保测试结果正确反映了代码的实际行为。
> 3. 性能测试:在需要高性能的代码模块中，进行性能测试。
> 4. 稳定性测试:测试代码的稳定性，确保在不同的环境和负载下都能正常工作。
> 5. 安全性测试:测试代码的安全性，确保代码没有安全漏洞。

#### 测试元编程

lightweight_test库也提供了对元编程(可参考推荐书目[3]) 测试的有限支持，在头文件＜boost/core/lightweight_test_trait.hpp＞里定义了两个编译期的断言:

```c++
#define BOOST_TEST_TRAIT_TRUE((type));
#define BOOST_TEST_TRAIT_FALSE((type));
```

这两个宏的效果与BOOST_STATIC_ASSERT类似，但它们判断的是type::value，而不是type本身，所以宏的参数type应该是能够返回bool值的元函数。还需要特别注意:由于内部实现的原因，type必须要使用圆括号包围(宏展开为一个模板函数的参数) 。

### 6.4 test

test库提供了一个用于单元测试、基于命令行界面的测试套件:Unit Test Framework(简称UTF) ，它比其他的单元测试库更强大、更方便、更好用。

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
- BOOST_CHECK(e) :断言测试通过，如不通过也不影响程序执
行。
- BOOST_REQUIRE(e) :要求测试必须通过，否则程序停止执
行。
- BOOST_ERROR(s) :给出一个错误信息，程序继续执行。
- BOOST_FAIL(s) :给出一个错误信息，程序运行终止。
上述4个测试断言是最基本的测试断言，能够在任何地方使用，同时由于通用性，它们也不具有其他测试断言的好处，我们应当尽量少使用它们。

test库其他测试断言的形式是BOOST_LVL_XXX，如BOOST_CHECK_EQUAL、BOOST_WARN_GT，详细的命名规则如下。
- BOOST:遵循Boost库的命名规则，宏一律以大写的BOOST开头。
- LVL:断言的级别。WARN是警告级，不影响程序运行，也不增加错误数量；CHECK 是检查级别，如果断言失败则增加错误数量，但不影响程序运行；REQUIRE 是最高的级别，如果断言失败将增加错误数量并终止程序运行。最常用的断言级别是CHECK，WARN可以用于不涉及程序关键功能的测试，只有当断言失败会导致程序无法继续进行测试时才能使用REQUIRE。
- XXX:各种具体的测试断言，如断言相等/不等、抛出/不抛出异常、大于/小于等。

test库中常用的几个测试断言如下。
- BOOST_LVL_EQUAL(l，r) :检测l==r，当测试失败时会给出详细的信息。
它不能用于浮点数的比较，浮点数的相等比较应使用BOOST_LVL_CLOSE。

- BOOST_LVL_GE(l，r) :检测l＞=r，与GT(l＞r) 、LT(l＜r) 、LE(l＜=r) 和NE(l!=r) 相同，它们用于测试各种不等性。
- BOOST_LVL_THROW(e，ex) :检测表达式e，抛出指定的ex异常。
- BOOST_LVL_NO_THROW(e) :检测表达式e，不抛出任何异常。
- BOOST_LVL_MESSAGE(e，msg) :它与不带MESSAGE后缀的断言的功能相同，但测试失败时，它会给出指定的消息。
- BOOST_TEST_MESSAGE(msg) :它仅输出通知信息，不含有任何警告或错误信息，默认情况不会显示。在之后的几节中，将会看到这些测试断言的具体用法。



#### 测试主体

单元测试领域有很多专有概念，本节仅介绍test库中比较重要的几个。
test库将测试程序定义为一个测试模块，由测试安装、测试主体、测试清理和测试运行器四部分组成。测试主体是测试模块的实际运行部分，由测试用例和测试套件组织成测试树的形式。

### 6.5 总结

很多软件方法学(如敏捷开发、极限编程) 都非常强调测试的重要性，使用Boost测试库，我们就可以实践这些方法。
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

首先我们将学习一个小的工具类ref，它是本章其他库的基础，可以用它包装对象的引用，在传递参数时消除对象拷贝的代价，还可以利用它将不可拷贝的对象变为可以拷贝。
bind是C++标准库中函数适配器的增强，能够适配任意的可调用对象——包括函数指针、函数引用和函数对象，把它们变成一个新的函数对象，学习bind是迈向C++函数式编程的第一步。function库则是对C/C++中函数指针类型的增强，它能够容纳任意的可调用对象，可以配合bind使用。
最后我们讨论 signals2库，它实现了威力强大的观察者模式，如果读者曾经用过Java的Observable/Observer或C＃的event/delegate，那么就会知道signals2对于C++程序员的重要意义。

### 11.1 ref

C++标准库和 Boost 中的算法大量使用了函数对象作为判断式或谓词参数，而这些参数都是传值语义。一般情况下，传值语义都是可行的，但也有很多特殊情况:作为参数的函数对象拷贝代价过高(具有复杂的内部状态) ，不希望拷贝对象(内部状态不应该被改变) ，甚至拷贝是不可行的(noncopyable、singleton) 。

boost.ref 应用代理模式，引入对象引用的包装器概念解决了上述问题。它位于名字空间boost，需要包含的头文件如下。

```c++
#include<boost/ref.hpp>
using namespace boost;
```

##### 示例代码

```c++
// Copyright (c) 2015
// Author: Chrono Law
#include <cmath>
#include <functional>
#include "std.h"
//using namespace std;

#include <boost/ref.hpp>
using namespace boost;

struct square
{
    typedef void result_type;
    result_type operator()(int& x)
    {
        x = x * x;
    }
};

//////////////////////////////////////////

void case1()
{
    std::vector<int> v = { 1,2,3,4,5 };
    for_each(v.begin(), v.end(), square());
}

//////////////////////////////////////////

void case2()
{
    int x = 10;
    reference_wrapper<int> rw(x);
    assert(x == rw);
    (int&)rw = 100;
    assert(x == 100);

    reference_wrapper<int> rw2(rw);
    assert(rw2.get() == 100);

    std::string str;
    reference_wrapper<std::string> rws(str);
    *rws.get_pointer() = "test reference_wrapper";
    std::cout << rws.get().size() << std::endl;

}

//////////////////////////////////////////

void case3()
{
    double x = 2.71828;
    auto rw = cref(x);
    std::cout << typeid(rw).name() << std::endl;

    std::string str;
    auto rws = boost::ref(str);
    std::cout << typeid(rws).name() << std::endl;

    boost::cref(str);   //adl

    std::cout << std::sqrt(ref(x)) << std::endl;      //计算平方根
}

//////////////////////////////////////////

void case4()
{
    std::vector<int> v(10, 2);
    auto rw = boost::cref(v);

    assert(is_reference_wrapper<decltype(rw)>::value);
    assert(!is_reference_wrapper<decltype(v)>::value);

    std::string str;
    auto rws = boost::ref(str);
    std::cout << typeid(unwrap_reference<decltype(rws)>::type).name() << std::endl;
    std::cout << typeid(unwrap_reference<decltype(str)>::type).name() << std::endl;

}

//////////////////////////////////////////

void case5()
{
    std::set<int> s;
    auto rw = boost::ref(s);
    unwrap_ref(rw).insert(12);

    std::string str("test");
    auto rws = boost::cref(str);
    std::cout << unwrap_ref(rws) << std::endl;
    std::cout << unwrap_ref(str) << std::endl;

}

//////////////////////////////////////////

class big_class
{
private:
    int x;
public:
    big_class() :x(0) {}
    void print()
    {
        std::cout << "big_class " << ++x << std::endl;
    }
};
template<typename T>
void print(T a)
{
    for (int i = 0; i < 2; ++i)
        unwrap_ref(a).print();
}
void case6()
{
    big_class c;
    auto rw = ref(c);
    c.print();

    print(c);
    print(rw);
    print(c);
    c.print();
}

void case7()
{
    using namespace std;

    typedef double (*pfunc)(double);
    pfunc pf = sqrt;
    cout << std::ref(pf)(5.0) << endl;

    square sq;
    int x = 5;
    std::ref(sq)(x);
    cout << x << endl;

    vector<int> v = { 1,2,3,4,5 };
    for_each(v.begin(), v.end(), std::ref(sq));
}


int main()
{
    case1();
    case2();
    case3();
    case4();
    case5();
    case6();
    case7();
}
/*
22
class boost::reference_wrapper<double const >
class boost::reference_wrapper<class std::basic_string<char,struct std::char_traits<char>,class std::allocator<char> > >
1.64872
class std::basic_string<char,struct std::char_traits<char>,class std::allocator<char> >
class std::basic_string<char,struct std::char_traits<char>,class std::allocator<char> >
test
test
big_class 1
big_class 2
big_class 3
big_class 2
big_class 3
big_class 4
big_class 5
big_class 4
2.23607
25
*/
```



#### 引用包装器类摘要

```c++
template<class T>
class reference_wrapper
{
    public:
    explicit reference_wrapper(T &t):t_(&t){}
    operator T& () const {return *t_;}
    T& get() const {return *t_;}
    T* get_pointer() const {return t_;}
    private:
    T* t_;
}
```

reference_wrapper的构造函数接收类型 T的引用类型，内部使用指针存储指向t的引用，构造出一个reference_wrapper 对象，从而包装了引用。get() 和 get_pointer() 这两个函数分别返回存储的引
用和指针，相当于解开对t的包装。请注意:reference_wrapper 的构造函数被声明为explicit，因
此必须在创建reference_wrapper对象时就初始化赋值，如同使用一个引用类型的变量。
reference_wrapper还支持隐式类型转换，它可以在需要的语境下返回存储的引用，因此它很像引用类型，能够在任何需要T出现的地方使用reference_wrapper。

> 一个引用包装器(reference_wrapper) 的类模板。该类模板有以下几个特点:
>
> 1. 可以用于包装任意类型T的引用，并且包装后可以像原始类型一样使用。
> 2. 通过成员函数operator T&和get()可以获得包装器所包装的对象的引用。
> 3. 可以通过get_pointer()函数获得被包装对象的指针。
> 4. 可以通过构造函数来进行初始化，构造函数中的参数是被包装对象的引用。
> 5. 支持对引用包装器进行拷贝构造和赋值操作。
> 6. 友元类reference_wrapper可以直接访问其他reference_wrapper的私有成员。
> 7. 支持将不同类型的引用包装器进行转换，只要两者的类型之间可以进行隐式转换即可。

Boost库里的定义

```c++
template<class T> class reference_wrapper
{
public:
    /**
     Type `T`.
     模板参数类型T
    */
    typedef T type;

    /**
     Constructs a `reference_wrapper` object that stores a
     reference to `t`.

     @remark Does not throw.
     构造函数，用于将`t`引用包装为一个`reference_wrapper`对象
     不抛出异常
    */
    BOOST_FORCEINLINE explicit reference_wrapper(T& t): t_(boost::addressof(t)) {}

    // 下面的代码是特定编译器(Microsoft Visual C++ 2010) 的特定工作区间(Boost) 中的特定代码，具体作用请参考具体文档
#if defined( BOOST_MSVC ) && BOOST_WORKAROUND( BOOST_MSVC, == 1600 )
    BOOST_FORCEINLINE explicit reference_wrapper( T & t, ref_workaround_tag ): t_( boost::addressof( t ) ) {}
#endif

    // 下面的代码是通过判断是否支持C++11特性来决定是否编译的
#if !defined(BOOST_NO_CXX11_RVALUE_REFERENCES)
    /**
     @remark Construction from a temporary object is disabled.
     禁止从临时对象进行构造
    */
    BOOST_DELETED_FUNCTION(reference_wrapper(T&& t))
public:
#endif

    // 声明友元类reference_wrapper
    template<class Y> friend class reference_wrapper;

    /**
     Constructs a `reference_wrapper` object that stores the
     reference stored in the compatible `reference_wrapper` `r`.

     @remark Only enabled when `Y*` is convertible to `T*`.
     @remark Does not throw.
     构造函数，将一个兼容的`reference_wrapper`对象`r`中存储的引用存储到当前`reference_wrapper`对象中
     只有在`Y*`能够隐式转换为`T*`时才被启用
     不抛出异常
    */
    template<class Y> reference_wrapper( reference_wrapper<Y> r,
        typename enable_if_c<boost::detail::ref_convertible<Y, T>::value,
            boost::detail::ref_empty>::type = boost::detail::ref_empty() ): t_( r.t_ )
    {
    }

    /**
     @return The stored reference.
     @remark Does not throw.
     返回被包装对象的引用
     不抛出异常
    */
    BOOST_FORCEINLINE operator T& () const { return *t_; }

    /**
     @return The stored reference.
     @remark Does not throw.
     返回被包装对象的引用
     不抛出异常
    */
    BOOST_FORCEINLINE T& get() const { return *t_; }

    /**
     @return A pointer to the object referenced by the stored
       reference.
     @remark Does not throw.
     返回被包装对象的指针
     不抛出异常
    */
    BOOST_FORCEINLINE T* get_pointer() const { return t_; }

private:
    // 存储被包装对象的指针
    T* t_;
};

```

#### 基本用法

reference_wrapper的用法有些类似C++中的引用类型(T&) ，它就像被包装对象的一个别名。但它只有在使用T 的语境下才能够执行隐式转换，其他的情况下则需要调用类型转换函数或get() 函数才能真正地操作被包装对象。
此外，reference_wrapper支持拷贝构造和赋值，而引用类型是不可赋值的。

> `reference_wrapper`与真正的引用在使用上有以下几个异同点:
>
> 1. `reference_wrapper`是一个对象，而真正的引用是一种类型。这意味着，可以将`reference_wrapper`作为对象进行复制、移动、赋值等操作，而真正的引用则没有这些属性。
> 2. `reference_wrapper`需要使用函数调用运算符或者类型转换运算符才能访问所封装的引用，而真正的引用则可以直接访问。
> 3. `reference_wrapper`可以在不需要绑定到引用的地方使用，例如可以将其作为函数参数传递，而真正的引用则必须在声明时就被绑定到一个变量上。
> 4. `reference_wrapper`可以被赋值和复制，而真正的引用则不能被赋值或复制，只能被初始化。
> 5. `reference_wrapper`可以为指针类型的对象创建引用，而真正的引用只能为具体的变量或对象创建引用。
>
> 总的来说，`reference_wrapper`在某些方面比真正的引用更加灵活，可以用于更多的场景中。但是，需要注意的是，`reference_wrapper`并不是真正的引用，它仍然是一个对象，因此在使用时需要特别小心，避免出现一些不可预期的问题。

#### 工厂函数

reference_wrapper的名字过长，用它声明包装对象很不方便，因此ref库提供了两个便捷的工厂函数 ref() 和 cref() ，利用它们可以通过推导参数类型很容易地构造包装对象。

```c++
reference_wrapper<T> ref(T& t);
reference_wrapper<T> cref(T const &t);
```

#### 操作包装

ref库运用模板元编程技术提供两个特征类:is_reference_wrapp
er和unwrap_reference，用于检测reference_wrapper对象:

- is_reference_wrapper＜T＞::value可以判断T是否被包装。
- unwrap_reference＜T＞::type表明了T的真实类型(无论它是否经过包装) 。直接对一个未包装的对象使用unwrap_ref() 也是可以的，它将直接返回对象自身的引用

#### 与STD标准对比

ref将对象包装为引用语义，降低了复制的代价，使引用的行为更像对象(因为对象更有用、更强大) ，可以让容器安全地持有被包装的引用对象，可以称其为“智能引用”。因此它被收入了C++标准。

> - `boost::ref`是Boost库中的一个类模板，而`std::reference_wrapper`是C++标准库中的一个类模板。
> - `boost::ref`提供了更多的功能和运算符重载，如`make_ref()`、`reset()`、`operator->()`等，而`std::reference_wrapper`只提供了必要的功能和运算符重载。
> - `boost::ref`可以被隐式转换为原始引用类型，而`std::reference_wrapper`只能通过`get()`成员函数获取原始引用类型。
> - `boost::ref`可以包装对非const对象和函数的引用，而`std::reference_wrapper`只能包装对const对象和函数的引用(C++20中加入了对非const对象的支持) 。

### 11.2 bind

bind 是对 C++标准中函数适配器 bind1st/bind2nd 的泛化和增强，可以适配任意的可调用对象，包括函数指针、函数引用、成员函数指针、函数对象和lambda表达式。
bind远远地超越了STL中的函数绑定器bind1st/bind2nd，它最多可以绑定9个函数参数，而且bind对被绑定对象的要求很低，它可以在没有result_type内部类型定义的情况下完成对函数对象的绑定。

#### 工作原理

bind并不是一个单独的类或函数，而是非常庞大的家族，依据绑定的参数个数和要绑定的调用对象类型不同，bind总共有数十个不同的重载形式，但它们的名字都叫作bind，编译器会根据具体的绑定代码自动推导要使用的正确形式。

bind接收的第一个参数必须是一个可调用对象f，可以是函数、函数指针、函数对象或成员函数指针，之后 bind最多接收9个参数，参数的数量必须与f 的参数数量相等，这些参数将被传递给f作为入参。

绑定完成后，bind会返回一个函数对象，它内部保存了f的拷贝，具有operator() ，返回值类型被自动推导为f的返回值类型。在发生调用时，这个函数对象将把之前存储的参数转发给f完成调用。

bind的真正威力在于它的占位符，它们分别被定义为`_1`、`_2`、`_3`一直到`_9`，位于一个匿名名字空间。占位符可以取代bind中参数的位置，在发生函数调用时才接收真正的参数。

#### 操作普通函数

bind可以绑定普通函数，使用函数名或函数指针，假设我们有如下的函数定义:

占位符是bind 的精华

```c++
#include <iostream>
#include <boost/bind.hpp>

int f(int a, int b)
{
    std::cout << a + b << std::endl;
    return 0;
}

int g(int a, int b, int c)
{
    std::cout << a + b * c<< std::endl;
    return 0;
}
int main()
{
    int x = 1, y = 2, z = 3;
    bind(f, _1, 9)(x);
    bind(f, _1, _2)(x, y);
    bind(f, _2, _1)(x, y);
    bind(f, _1, _1)(x, y);
    bind(g, _1, 8, _2)(x, y);
    bind(g, _3, _2, _2)(x, y, z);
    return 0;
}

/*
10
3
3
2
17
7
*/
```

bind同样可以绑定函数指针，用法相同。

```c++
//typedef int (*f_type)(int, int);
//typedef int (*g_type)(int, int, int);
typedef decltype(&f) f_type;
typedef decltype(&g) g_type;//同样的做法，但是这种写法可读性更好，更优雅
f_type pf = f;
g_type pg = g;
std::cout << bind(pf, _1, 9)(x) << std::endl;
std::cout << bind(pg, _3, _2, _2)(x, y, z) << std::endl;
```

#### 操作成员函数

bind也可以绑定类的成员函数。
类的成员函数不同于普通函数，因为成员函数指针不能直接调用operator() ，它必须先被绑定到一个对象或指针上，然后才能得到this指针进而调用成员函数。因此bind需要“牺牲”一个占位符的位置，要求用户提供一个类的实例、引用或指针，通过对象作为第一个参数来调用成员函数。这意味着使用成员函数时最多只能绑定8个参数。

```c++
struct demo
{
    int f(int a, int b)
    {   return a + b;   }
};
struct point
{
    int x, y;
    point(int a = 0, int b = 0):x(a),y(b){}
    void print()
    {   std::cout << "(" << x << "," << y << ")\n";  }
};
void case2()
{
    demo a,&ra=a;
    demo *p = &a;

    std::cout << bind(&demo::f, a, _1, 20)(10) << std::endl;
    std::cout << bind(&demo::f, ra, _2, _1)(10, 20) << std::endl;
    std::cout << bind(&demo::f, p, _1, _2)(10, 20) << std::endl;

    std::vector<point> v(10);
    std::for_each(v.begin(), v.end(), bind(&point::print, _1));
}
```

> 在 C++ 中，类的成员函数和普通函数有所不同。每个成员函数都会隐式地接受一个指向该类对象的指针作为第一个参数，这个指针通常称为 "this 指针"。因此，在使用成员函数时，我们必须先创建一个对象，然后才能调用它的成员函数。
>
> 当我们使用 `std::bind` 函数时，它可以接受一个对象、指针或引用，它们都可以被视为一个指向该类对象的指针。在使用这些对象、指针或引用时，`std::bind` 函数会自动将它们转换为指向该类对象的指针，并将其作为第一个参数传递给成员函数。
>
> 因此，无论是对象、对象指针还是对象引用，都可以被用作 `std::bind` 函数的第一个参数，来绑定成员函数。这种灵活性可以帮助我们更方便地使用成员函数，并且不必担心指针和引用的语法细节。

bind 能够绑定成员函数，这是个非常有用的功能，它可以替代标准库中令人迷惑的mem_fun和mem_fun_ref绑定器，用来配合标准算法操作容器中的对象。下面的代码使用bind搭配标准算法for_each用来调用容器中所有对象的print() 函数:

bind同样支持绑定虚拟成员函数，其用法与非虚拟函数相同，虚函数的行为将由实际调用发生时的实例来决定。

#### 操作成员变量

bind的另一个对类的操作是它可以绑定public成员变量，它就像是一个选择器，其用法与绑定成员函数类似，只需要像使用一个成员函数一样去使用成员变量名。

```c++
void case3()
{
    std::vector<point> v(10);
    std::vector<int> v2(10);

    std::transform(v.begin(), v.end(), v2.begin(), bind(&point::x, _1));

    for(auto x : v2)                        //foreach循环输出值
        std::cout << x << ",";

    typedef std::pair<int, std::string> pair_t;
    pair_t p(123, "string");

    std::cout << bind(&pair_t::first , p)() << std::endl;
    std::cout << bind(&pair_t::second, p)() << std::endl;

}
```

#### 操作函数对象

bind不仅能够绑定函数和函数指针，也能够绑定任意的函数对象，包括标准库中的所有预定义的函数对象。

但如果函数对象没有定义result_type，则需要在绑定形式上进行改动，用模板参数指明返回类型

对于自定义函数对象，如果没有result_type类型定义，我们必须在模板参数里指明bind的返回值类型，像这样:

```c++
struct func
{
        int operator()(int a, int b)
            {   return a + b;   }
};

void case4()
{
    bind(std::greater<int>(), _1, 10);
    bind(std::plus<int>(), _1, _2);
    bind(std::modulus<int>(), _1, 3);

    std::cout << bind<int>(func(), _1, _2)(10,20) << std::endl;
}
```

#### 对比标准

C++标准使用可变参数模板和完美转发简化了bind 的定义(C++11.20.8.9) ，可以支持绑定任意数量的参数:

> 完美转发(perfect forwarding) 是 C++11 引入的一项特性，它允许将函数的参数在调用时精确地转发给另一个函数，保持参数类型和值的不变性，同时避免了不必要的拷贝和转换操作，提高了代码的效率和可维护性。
>
> 在使用完美转发时，需要使用两个 C++11 新特性:右值引用和可变参数模板。右值引用允许我们获取到右值对象的引用，并对其进行操作；而可变参数模板允许我们接受任意数量和类型的参数，并对其进行操作。
>
> 使用完美转发时，我们通常会定义一个转发函数，这个函数的参数可以接受任意数量和类型的参数，并将这些参数转发给另一个函数。为了避免参数的拷贝和转换，我们需要使用引用折叠(reference collapsing) 和 std::forward 函数模板来实现完美转发。
>
> 引用折叠是一个 C++11 的特性，它可以将两个引用折叠成一个新的引用。在使用完美转发时，我们通常会将参数定义为右值引用和万能引用(Universal Reference，也称为转发引用) ，然后通过引用折叠将它们折叠成一个新的引用，从而实现参数类型和值的不变性。
>
> std::forward 函数模板是一个用于完美转发的工具，它可以根据参数类型来选择是将参数转发为左值引用还是右值引用。在使用完美转发时，我们通常会将转发函数中的参数通过 std::forward 函数模板转发给被调用函数，以实现完美转发。
>
> 总之，完美转发是 C++11 中一个非常重要的特性，它可以在函数调用时避免不必要的拷贝和转换，提高代码的效率和可维护性。使用完美转发时需要注意参数的类型和引用，以及使用 std::forward 函数模板来实现参数的转发。

C++标准还提供了语言级别的lambda表达式(C++11.5.1.2) ，它可以就地声明匿名函数对象，其用法非常灵活。lambda 表达式在某种程度上也可以代替 bind，捕获列表“[...]”相当于绑定的变量，函数参数列表“(...) ”则相当于bind的占位符。

#### 高级议题

- 为占位符更名

  ```c++
  boost::arg<1> &_x = _1;
  ```

- bind表达式生成的函数对象的类型声明非常复杂使用auto来辅助我们写出正确的类型。

- 使用ref库

  bind采用拷贝的方式存储绑定函数对象和参数，这意味着绑定表达式中的每个变量都会有一份拷贝，如果函数对象或参数很大、拷贝代价很高，或者无法拷贝，那么bind的使用就会受到限制。因此 bind库可以搭配 ref库使用，ref库包装了对象的引用，可以让 bind存储对象引用的拷贝，从而降低拷贝的代价。但这也带来了一个隐患，因为有时候bind的调用可能会延后很久，程序员必须保证bind被调用时引用是有效的。如果调用bind时引用的变量或函数对象被销毁了，那么就会发生未定义行为。

- 嵌套绑定

- 操作符重载

- 绑定重载函数

  直接使用函数名的绑定方式存在一点局限，如果程序里有若干个同名的重载函数，那么bind就无法确定要绑定的具体函数，导致编译错误。

  一个解决方法是用typedef定义函数指针类型，再使用函数指针变量明确要绑定的函数

  但这个方法对于模板函数无效，因为我们很难写出一个准确的模板函数指针类型，如5.4.4节介绍的判断式:这时我们只能使用lambda表达式来变通地“绑定”:

  ```c++
  bind(boost::contains,_1,"a");//失败
  [](const string& x){return contains(x,"a");}
  ```

- 绑定非标准函数

  bind库大大增强了C++标准库中的函数绑定器，可以适配任何 C++中的函数。但标准形式bind(f，...) 不是100%适用于所有情况，有些非标准函数是无法自动推导出返回值类型的，典型的例子就是C中的可变参数函数printf() ，必须用bind＜int＞(printf，...) (...) 的形式。例如:`bind＜int＞(printf,＂%d+%d=%d\n＂,_1,1,_2)(6,7);`
  bind的标准形式也不能支持使用了不同的调用方式(如`__stdcall`、`__fastcall`、`extern ＂C＂`) 的函数，通常bind把它们看作函数对象，需要显式地指定bind的返回值类型才能绑定。

  我们也可以在头文件＜boost/bind.hpp＞之前加上 BOOST_BIND_ENABLE_STDCALL、BOOST_BIND_ENABLE_FASTCALL等宏，明确地告诉bind支持这些调用方式。

> 使用 `std::bind` 函数获取成员变量的值通常是在需要将这个函数对象传递给其他函数或算法时使用，例如 `std::for_each` 算法、`std::sort` 算法等等。在这些情况下，需要将一个函数对象作为参数传递给另一个函数或算法，而不能直接传递一个成员变量值。因此，可以使用 `std::bind` 函数将成员变量函数指针绑定到一个对象上，创建一个函数对象，从而实现将成员变量作为函数对象传递的目的。

### 11.3 function

function是一个函数对象的“容器”，在概念上它像是C++中函数指针类型的泛化，是一种“智能函数指针”。它以对象的形式封装了原始的函数指针或函数对象，能够容纳任意符合函数签名的可调用对象。因此它可以被用于回调机制，暂时保管函数或函数对象，在之后需要的时机再调用这些函数或函数对象，使回调机制拥有更多的弹性。

function 可以配合 bind/lambda 使用，以存储 bind/lambda 表达式的结果，使bind/lambda能够被多次调用。

#### 类摘要

同bind一样，function也不是一个单独的类，而是一个大的类家族。function可以容纳0到10个参数的函数，所以它有多个类，其命名分别是function0到function10。但我们通常不直接使用它们，而是使用一个更通用的function类，它的类摘要如下:

 `boost::function` 类的主要接口:

- `function()`:默认构造函数，创建一个空的函数对象；
- `~function()`:析构函数，销毁函数对象；
- `function(function const& other)`:拷贝构造函数，创建一个新的函数对象，共享目标对象；
- `function(function&& other)`:移动构造函数，创建一个新的函数对象，转移目标对象的所有权；
- `template<typename F> function(F f)`:将函数指针转换为函数对象，创建一个新的函数对象；
- `function& operator=(function const& other)`:重载赋值操作符，销毁当前的函数对象，共享目标对象；
- `function& operator=(function&& other)`:重载赋值操作符，销毁当前的函数对象，转移目标对象的所有权；
- `R operator()(Args... args)`:调用函数，返回函数的返回值；
- `bool operator==(function const& other) const noexcept`:比较函数对象是否相等；
- `bool operator!=(function const& other) const noexcept`:比较函数对象是否不相等；
- `void swap(function& other) noexcept`:交换两个函数对象的内容；
- `bool empty() const noexcept`:测试函数对象是否为空；
- `bool operator bool() const noexcept`:测试函数对象是否有效；
- `std::type_info const& target_type() const noexcept`:获取函数对象的目标对象类型；
- `void const* target() const noexcept`:获取函数对象的目标对象；
- `template<typename T> T* target() noexcept`:获取函数对象的目标对象，如果类型不匹配则返回 `nullptr`。

#### 声明形式

function只需要一个模板参数，这个参数就是将要容纳的函数类型。例如:`function＜int()＞ func;`将声明一个可以容纳返回值为int、无参函数的function对象。

如果我们已经知道将要容纳的函数，那么也可以用关键字 decltype 来直接获取函数类型。例如:

```c++
int f(int a,int b){}
function<decltype(f)> func;
```

注意decltype的用法，不能写成decltype(&f),这回推断出指针类型而不是函数类型。

```c++
int f6(int a, int b) { return a + b; }
void case6()
{
    std::cout << "case 6" << std::endl;
    function<decltype(f6)> func;
    std::cout << typeid(decltype(f6)).name() << std::endl;
    std::cout << typeid(decltype(&f6)).name() << std::endl;
}
/*
case 6
int __cdecl(int,int)
int (__cdecl*)(int,int)
*/
```

> 常见的函数调用规则有以下几种:
>
> - `__cdecl`:表示使用 C 语言调用规则，参数从右向左依次入栈，由调用者负责清理栈上的参数，返回值通常使用寄存器传递；
> - `__stdcall`:表示使用标准调用规则，参数从右向左依次入栈，由被调用函数负责清理栈上的参数，返回值通常使用寄存器传递；
> - `__fastcall`:表示使用快速调用规则，参数传递采用寄存器传递，有些编译器也允许在寄存器和栈之间进行混合传递，返回值通常使用寄存器传递；
> - `__vectorcall`:表示使用向量调用规则，参数传递采用寄存器传递，返回值也采用寄存器传递，适用于处理向量和矩阵等数据类型。
>
> 除此之外，一些特殊的调用规则也可能由编译器或操作系统定义，例如 Windows 下的 `__thiscall` 和 Linux 下的 `__attribute__((regparm))` 等。需要注意的是，不同的编译器和操作系统可能会采用不同的调用规则，因此编写可移植的代码时应该避免依赖特定的调用规则。

#### 用法

function 就像一个函数的容器，也可以把 function 想象成一个泛化的函数指针，只要符合其声明中的函数类型，任何普通函数、成员函数、函数对象都可以存储在function对象中，然后在任何需要的时候被调用。

function 这种能够容纳任意可调用对象的能力是非常重要的，尤其是在编写泛型代码的时候，它使我们可以接收任意函数或函数对象，增加程序的灵活性。
与原始的函数指针相比，function 对象的体积要稍微大一点(3个指针的大小) ，速度要稍微慢一点(10%左右的性能差距) ，但这些缺点与它带给程序的巨大好处相比微不足道。

function的基本用法

```c++
int f(int a, int b)
{   return a + b;}
void case2()
{
    //function<int(int,int)> func;
    function<decltype(f)> func;
    assert(!func);

    func = f;
    assert(func.contains(&f));

    if (func)
    {
        std::cout << func(10, 20);
    }

    func = 0;
    assert(func.empty());
}
```

只要函数签名式一致，function也可以存储成员函数和函数对象，还可以存储bind/lambda表达式。假设我们有如下的一个类，它既有普通成员函数，又重载了operator() 存储成员函数时

我们可以直接在function 声明的函数签名式中指定类的类型，然后用bind绑定成员函数。

我们也可以在函数类型中仅写出成员函数的签名，在bind时直接绑定类的实例:

```c++
struct demo_class
{
    int add(int a, int b)
    {       return a + b;   }
    int operator()(int x) const
    {       return x*x; }
};

void case3()
{
    function<int(demo_class&, int,int)> func1;
    func1 = bind(&demo_class::add, _1, _2, _3);
    demo_class sc;
    std::cout << func1(sc, 10, 20);

    function<int(int,int)> func2;
    func2 = bind(&demo_class::add,&sc, _1, _2);
    std::cout << func2(10, 20);

}
```

#### 使用ref库

function使用拷贝语义保存参数，当参数很大时，拷贝的代价往往很高，甚至有时候不能拷贝参数。这时我们可以向 ref库求助，它允许以引用的方式传递参数，能够降低function拷贝的代价。

function 并不要求 ref库提供 operator() ，因为它能够自动识别包装类reference_wrapper，并调用get() 方法获得被包装的对象。function能够直接调用被ref库包装的函数对象，这个功能可以弥补boost.ref没有operator() 的遗憾。下面的代码定义了一个求总和的函数对象。

如果ref库不提供operator() ，那么它将无法用于标准库算法，因为标准库算法总使用拷贝语义，其算法内部的改变不能影响原对象，function可以提供一个略显麻烦但可用的解法:

```c++
template<typename T>
struct summary
{
    typedef void result_type;
    T sum;

    summary(T v = T()):sum(v){}

    void operator()(T const &x)
    {   sum += x;   }
};
void case5()
{
    std::vector<int> v = {1,3,5,7,9};

    summary<int> s;
    function<void(int const&)> func(ref(s));

    std::for_each(v.begin(), v.end(), func);
    std::cout << s.sum << std::endl;
}
```

#### 用于回调

function可以容纳任意符合函数签名式的可调用物，因此它非常适合代替函数指针，存储用于回调的函数，而且它的强大功能会使代码更灵活、更富有弹性。

```c++
class demo_class
{
private:
    typedef function<void(int)> func_t;
    func_t func;
    int n;
public:
    demo_class(int i):n(i){}

    template<typename CallBack>
    void accept(CallBack f)
    {   func = f;   }

    void run()
    {   func(n);    }
};

void call_back_func(int i)
{
    using namespace std;
    cout << "call_back_func:";
    cout << i * 2 << endl;
}

void case1()
{
    demo_class dc(10);
    dc.accept(call_back_func);
    dc.run();

}
```

使用普通函数进行回调并不能体现 function 的好处，我们来编写一个带状态的函数对象，并使用ref库传递引用:

```c++
class call_back_obj
{
private:
    int x;
public:
    call_back_obj(int i):x(i){}

    void operator()(int i)
    {
        using namespace std;
        cout << "call_back_obj:";
        cout << i * x++ << endl;
    }
};

void case2()
{
    demo_class dc(10);
    call_back_obj cbo(2);

    dc.accept(ref(cbo));

    dc.run();
    dc.run();
}
```

因为demo_class使用了function作为内部可调用物的存储，因此不用进行任何改变，即可接收函数指针和函数对象，给用户以最大的方便。



function 还可以搭配 bind库，把 bind 表达式作为回调函数，可以接收类成员函数，或者把不符合函数签名式的函数bind转为可接收的形式。下面我们定义一个回调函数工厂类，它有两个回调函数:

```c++
class call_back_factory
{
public:
    void call_back_func1(int i)
    {
        using namespace std;
        cout << "call_back_factory1:";
        cout << i * 2 << endl;
    }
    void call_back_func2(int i, int j)
    {
        using namespace std;
        cout << "call_back_factory2:";
        cout << i *j * 2 << endl;
    }
};

void case3()
{
    demo_class dc(10);
    call_back_factory cbf;

    dc.accept(bind(&call_back_factory::call_back_func1, cbf, _1));
    dc.run();

    dc.accept(bind(&call_back_factory::call_back_func2, cbf, _1, 5));
    dc.run();
}
```

通过以上示例代码我们可以看到 function 用于回调的好处，它无须改变回调的接口就可以解耦客户代码，使客户代码不必绑死在一种回调形式上，进而可以持续演化，而function始终能够保证与客户代码进行正确沟通。

#### 对比标准

function类似一个容器，可以容纳任意有operator() 的类型(函数指针、函数对象、lambda表达式) ，它是运行时的，可以任意拷贝、赋值、存储其他可调用物。

std::function与boost::function基本相同，它们只有少量的区别:

- 没有clear() 和empty() 成员函数。
- 提供assign() 成员函数。
- explicit显式bool转型。

所以，同shared_ptr一样，std::function在函数返回值或函数参数等语境里转型bool需要使用static_cast＜bool＞(f) 或() !!f的形式。

### 11.4 signals2

signals2（衍生自Boost中另一个已被废弃的库signals）实现了线程安全的观察者模式。在signals2库中，观察者模式被称为信号/插槽（signals/slots）机制，它是一种函数回调机制，一个信号关联了多个插槽，当信号发出时，所有关联它的插槽都会被调用。
许多成熟的软件系统都用到了这种信号/插槽机制（另一个常用的名称是事件处理机制：event/event handler），它可以很好地解耦一组互相协作的类，有的语言甚至直接内建了对它的支持，signals2以库的形式为C++增加了这个重要的功能。

#### 1 类摘要

signals2库的核心是是signal 类，相当于 C＃语言中的event+delegate，它的模板参数总共有7个，

- `Signature`：信号的签名，以函数类型的形式指定。
- `Combiner`：用于组合连接到信号的槽返回值的二元函数对象。
- `Group`：可选参数，用于将相关信号分组。默认使用int来标记组号，也可以改为std：：string等类型，但通常没有必要。
- `GroupCompare`：用于比较分组值的比较函数对象。默认是升序（std：：less＜Group＞），因此要求Gro
  up必须定义operator＜。
- `SlotFunction`：可以连接到信号的函数对象的类型。
- `ExtendedSlotFunction`：可选参数，用于支持扩展的槽函数。
- `Mutex`：用于保护信号和槽的互斥对象类型。

signal继承自signal_base，而signal_base又继承自noncopyable（4.1节），因此 signal 是不可拷贝的。如果把 signal 作为自定义类的成员变量，那么自定义类也将是不可拷贝的，除非使用shared_ptr/ref来间接持有它。

#### 2 操作函数

signal最重要的操作函数是插槽管理函数connect（），它把插槽连接到信号上，相当于为信号（事件）增加了一个处理的handler。
插槽可以是任意的可调用对象，包括函数指针、函数对象，以及它们的bind/lambda表达式和function对象，signal内部使用function作为容器来保存这些可调用对象。

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



