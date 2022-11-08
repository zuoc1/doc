github：https://github.com/boostorg/boost

官网：https://www.boost.org/

Boost程序库完全开发指南配套源码：https://github.com/chronolaw/boost_guide

# 1. 编译安装

头文件默认安装位置：/usr/local/include，库文件默认安装位置：/usr/local/lib。

```shell
$ git clone --recursive http://github.com/boostorg/boost.git
$ cd boost
# 查看必须编译才能使用的库
$ ./bootstrap.sh --show-libraries
# 指定安装的库和路径
$ ./bootstrap.sh --with-libraries=context,fiber --prefix="/home/zuoc/work/boost/boost_install"
# 默认使用release模式
$ ./b2 install
# 完整编译，安装所有调试版、发行版的静态库和动态库
$ ./b2 --buildtype=complete install
# 编译安装所有静态链接库
$ ./b2 link=static install
# 安装date_time，--without是不安装
$ ./b2 --with-date_time install

# 添加环境变量，安装默认位置则不需要
# gcc找到头文件的路径
export C_INCLUDE_PATH="$C_INCLUDE_PATH:/home/zuoc/work/boost/boost_install/include"
# g++找到头文件的路径
export CPLUS_INCLUDE_PATH="$CPLUS_INCLUDE_PATH:/home/zuoc/work/boost/boost_install/include"
# 动态链接库的路径
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/home/zuoc/work/boost/boost_install/lib"
# 静态库的路径
export LIBRARY_PATH="$LIBRARY_PATH:/home/zuoc/work/boost/boost_install/lib"
source ~/.bashrc
```

## 1.1 测试是否安装成功

std.hpp

```c++
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

test.cpp

```c++
#include <std.hpp>
using namespace std;

#include <boost/version.hpp>
#include <boost/config.hpp>

int main()
{
    cout << __cplusplus << endl;
    cout << BOOST_VERSION << endl;
    cout << BOOST_LIB_VERSION<< endl;

    cout << BOOST_PLATFORM << endl;
    cout << BOOST_COMPILER << endl;
    cout << BOOST_STDLIB << endl;
}
```

```shell
$ g++ test.cpp -o test -I.
$ ./test
201402
108000
1_80
linux
GNU C++ version 9.4.0
GNU libstdc++ version 20210601
```

## 1.2 b2构建工具

```shell
cd /home/zuoc/work/boost/boost_1_80_0/tools/build
./bootstrap.sh
./b2 install --prefix="/home/zuoc/work/boost/b2"

# 添加可执行文件路径
export PATH="$PATH:/home/zuoc/work/boost/b2/bin"
source ~/.bashrc
```

这样上面那个测试用例就可以用以下语句自动编译执行：

```shell
$ cd /home/zuoc/work/boost/boost_guide/common
$ b2 t
...found 21 targets...
...updating 8 targets...
gcc.compile.c++ bin/gcc-9/debug/link-static/threading-multi/test.o
gcc.link bin/gcc-9/debug/link-static/threading-multi/t
testing.unit-test bin/gcc-9/debug/link-static/threading-multi/t.passed
201103
108000
1_80
linux
GNU C++ version 9.4.0
GNU libstdc++ version 20210601
...updated 8 targets...
```

# 2. 时间和日期

## 2.1 timer

作为v1 timer已过时，v2为cpu_timer。

## 2.2 date_time

