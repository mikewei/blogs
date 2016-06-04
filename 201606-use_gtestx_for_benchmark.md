### 使用gtestx写C++性能测试
[工具效率][2016-06-05 01:00:00 +0800]


在互联网后台开发领域，针对性能的单元测试常被忽视，实际上它非常重要，至少与普通的unit-test一样重要。特别是对一些基础库、关键部分代码，必要的性能单元测试，可帮助在模块粒度上发现并解决性能问题，这比在系统粒度上分析解决性能问题要容易地多。更为重要的，关注并熟悉模块级性能，是架构师掌控系统整体性能的基础，这在架构设计、线上问题分析等场景中会经常被使用。在过去几年工作中我写过很多的性能单测代码，它们帮助我更好地理解各种系统调用、数据结构、组件的性能；在我经历的几个海量系统研发过程中，严格的性能单测也对项目最终质量起到了显著的作用。

我在写性能测试方面（主要是C/C++）还算是有过不少经验，写这种代码虽不难，但是比较重复和枯燥，后来我尝试写了一些框架工具来简化这类工作，gtestx就是其中之一，开始只是个人工具，最近一年多在项目团队中推广使用大家觉得还不错。今天就系统性地介绍下这个工具。

gtestx是基于gtest的一个扩展，从取名上也很容易理解。为什么基于gtest呢？因为首先gtest基本上是现在C++的单元测试的主流选择，并且有广泛的群众基础，基于gtest来做测试测试可以更容易上手，也就是说，如果你已经在用gtest写单元测试了，基本不需要学习和额外工作就可以直接写gtestx的tests；另外gtest已经实现了一套比较先进好用的tests管理机制，性能单测本质上说也是一种单元测试，所以gtest很多优秀的特性gtestx可以直接继承复用。

下面我们就从几个实例出发，来介绍一下gtestx的用法。

#### 简单用例

首先一个简单的例子，测试系统调用getppid的性能：

```C++
#include <sys/types.h>
#include <unistd.h>
#include "gtestx/gtestx.h"

// Just like TEST macro, use PERF_TEST for performance testing
PERF_TEST(SimpleTest, SysCallPerf)
{
  getppid();
}
```

可以看出它跟gtest的TEST宏用法很像：

```C++
TEST(SimpleTest, SysCall)
{
  ASSERT_NE(0, getppid());
}
```

实际上PERF\_TEST跟TEST确实非常类似，它的两个参数的含意与TEST相同，都是表示gtest框架中的TestCaseName和TestName。所不同的是，PERF\_TEST的函数体并非只像TESt的函数体一样只运行一次，而是(默认)会死循环一段特定的时间并统计相关counter。在用例执行时，PERF\_TEST也会自动将统计的性能数据打印出来：

```
$ ./gtestx_examples --gtest_filter='SimpleTest.*'
[==========] Running 2 tests from 1 test case.
[----------] Global test environment set-up.
[----------] 2 tests from SimpleTest
[ RUN      ] SimpleTest.SysCall
[       OK ] SimpleTest.SysCall (0 ms)
[ RUN      ] SimpleTest.SysCallPerf
      count: 25,369,636
       time: 1.508939 s
         HZ: 16,812,897.009090
       1/HZ: 0.000000059 s
[       OK ] SimpleTest.SysCallPerf (1509 ms)
[----------] 2 tests from SimpleTest (1509 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test case ran. (1510 ms total)
[  PASSED  ] 2 tests.
```

从上面可以看出SimpleTest.SysCallPerf这个性能测试Test一共循环执行了25369636次，总共1.5秒时长，调用性能为每秒1681万次，平均每次调用耗时59纳秒。

#### Fixture用例

很多时候我们性能测试的过程是，先初始化需要的状态，然后性能测试，最后清理资源。这时我们适合使用gtest中的Fixture用例的gtestx版本。我们举一个具体一点的例子，比如要测试std::map的查找性能，需要先在map中插入一定数量的元素，然后执行性能单测，可以实现为下面的代码：

```C++
#include <map>
#include "gtestx/gtestx.h"

class StdMapTest : public testing::Test
{
protected:
  virtual ~StdMapTest() {}
  virtual void SetUp() override {
    for (int i = 0; i < 1000; i++) {
      map_.emplace(i, 1);
    }
  }
  virtual void TearDown() override {}
  
  std::map<int,int> map_;
};

PERF_TEST_F(StdMapTest, FindPerf)
{
  map_.find(999);
}
```

如上代码中，初始化部分在SetUp函数中完成，PERF\_TEST\_F宏是原有TEST\_F宏的一个gtestx版本，它只需一行代码，非常简单。用例执行结果如下：

```
$ ./gtestx_examples --gtest_filter='StdMapTest.FindPerf'
[==========] Running 1 test from 1 test case.
[----------] Global test environment set-up.
[----------] 1 test from StdMapTest
[ RUN      ] StdMapTest.FindPerf
      count: 3,187,592
       time: 1.508904 s
         HZ: 2,112,521.406266
       1/HZ: 0.000000473 s
[       OK ] StdMapTest.FindPerf (1512 ms)
[----------] 1 test from StdMapTest (1512 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test case ran. (1512 ms total)
[  PASSED  ] 1 test.
```


#### 带参数的Fixture用例

假设我们要测试std::unordered\_map的查找性能，但hash表的性能跟它的桶数量有很大的关系，我们希望测试在不同节点数/桶数量的配置下的查找性能的情况。若用普通方法我们需要根据不同的配置写多个用例，但使用带参数的Fixture用例一个就可以搞定了：

```C++
#include <unordered_map>
#include "gtestx/gtestx.h"

class HashMapTest : public testing::TestWithParam<float>
{
protected:
  virtual ~HashMapTest() {}
  virtual void SetUp() override {
    map_.max_load_factor(GetParam());
    for (int i = 0; i < 1000; i++) {
      map_.emplace(i, 1);
    }
  }
  virtual void TearDown() override {}

  std::unordered_map<int,int> map_;
  int search_key_ = 0;
};

INSTANTIATE_TEST_CASE_P(LoadFactor, HashMapTest, 
                        testing::Values(3.0, 1.0, 0.3));

PERF_TEST_P(HashMapTest, FindPerf)
{
  map_.find(++search_key_);
}
```

上面代码中我们使用INSTANTIATE\_TEST\_CASE\_P宏设定需要测试的不同参数值（这点与gtest中是完全相同），然后使用PERF\_TEST\_P定义性能测试的代码（跟gtest中的TEST\_P宏对应）。它的执行结果如下，可以看出性能随着节点/桶数的比例参数下降而升高：

```
$ ./gtestx_examples --gtest_filter='*HashMapTest.FindPerf*'
[==========] Running 3 tests from 1 test case.
[----------] Global test environment set-up.
[----------] 3 tests from LoadFactor/HashMapTest
[ RUN      ] LoadFactor/HashMapTest.FindPerf/0
      count: 4,629,181
       time: 1.509068 s
         HZ: 3,067,576.146337
       1/HZ: 0.000000326 s
[       OK ] LoadFactor/HashMapTest.FindPerf/0 (1511 ms)
[ RUN      ] LoadFactor/HashMapTest.FindPerf/1
      count: 11,975,484
       time: 1.508859 s
         HZ: 7,936,781.369233
       1/HZ: 0.000000126 s
[       OK ] LoadFactor/HashMapTest.FindPerf/1 (1510 ms)
[ RUN      ] LoadFactor/HashMapTest.FindPerf/2
      count: 16,320,267
       time: 1.508913 s
         HZ: 10,815,909.863591
       1/HZ: 0.000000092 s
[       OK ] LoadFactor/HashMapTest.FindPerf/2 (1509 ms)
[----------] 3 tests from LoadFactor/HashMapTest (4530 ms total)

[----------] Global test environment tear-down
[==========] 3 tests from 1 test case ran. (4530 ms total)
[  PASSED  ] 3 tests.
```

#### 模板Fixture用例

又假设我们要测试std::queue的进队出队的性能，并且需要考虑不同的元素类型（std::queue的模板参数类型）下的性能情况，我们可以使用基于模板Fixture的用例：

```C++
#include <queue>
#include "gtestx/gtestx.h"

template <class T>
class QueueTest : public testing::Test
{
protected:
  virtual ~QueueTest() {}
  virtual void SetUp() override {}
  virtual void TearDown() override {}

  std::queue<T> queue_;
};

typedef testing::Types<int, std::vector<char>> TestTypes;
TYPED_TEST_CASE(QueueTest, TestTypes);

TYPED_PERF_TEST(QueueTest, PushPopPerf)
{
  this->queue_.push(TypeParam());
  this->queue_.pop();
}
```

如上面代码，我们使用TYPED\_TEST\_CASE定义了int和std::vector<char>两个类参数分别来做性能测试，这时基于模板的用例使用TYPED\_PERF\_TEST定义（与gtest中的TYPED\_TEST对应）。执行结果如下：

```
$ ./gtestx_examples --gtest_filter='QueueTest*PushPopPerf*'
[==========] Running 2 tests from 2 test cases.
[----------] Global test environment set-up.
[----------] 1 test from QueueTest/0, where TypeParam = int
[ RUN      ] QueueTest/0.PushPopPerf
      count: 37,617,883
       time: 1.509303 s
         HZ: 24,924,009.956914
       1/HZ: 0.000000040 s
[       OK ] QueueTest/0.PushPopPerf (1509 ms)
[----------] 1 test from QueueTest/0 (1509 ms total)

[----------] 1 test from QueueTest/1, where TypeParam = std::vector<char, std::allocator<char> >
[ RUN      ] QueueTest/1.PushPopPerf
      count: 9,433,683
       time: 1.508871 s
         HZ: 6,252,146.803802
       1/HZ: 0.000000160 s
[       OK ] QueueTest/1.PushPopPerf (1509 ms)
[----------] 1 test from QueueTest/1 (1509 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 2 test cases ran. (3018 ms total)
[  PASSED  ] 2 tests.
```

#### PERF\_ABORT

如果你想在PERF函数体中使用ASSERT\_XXX等gtest宏，需要注意当断言fail时默认它只能跳出当前函数还不是整个性能测试的用例，所以gtestx中增加了一个辅助宏来帮助能正确地跳出整个用例。所以当使用ASSERT\_XXX等宏时，应当使用类似下面的代码：

```C++
PERF_TEST(MiscFeaturesTest, PerfAbort)
{
  ASSERT_NE(0, getppid()) << PERF_ABORT;
}
```

#### 指定频率和时长

默认gtestx对每个PERF用例的执行频率总是死循环压满，执行时长默认是固定的(1.5秒)。这两个参数实际都可以修改。什么场景需要修改呢？比如我们希望观察一定调用频率下的CPU或其它资源消耗，可以修改执行频率参数为某一固定值；再比如我们希望执行更长时间以观察性能的平均情况，则可以修改执行时长的参数。如何修改？gtestx考虑灵活性提供了2种方式。

方式一，使用带\_OPT后缀的参数扩展宏（实际所有的gtestx宏都有一个\_OPT版本），这种方式适用于某个特别的用例固定地需要一个特殊的频率或时长配置的场景。取一个例子说明：

```C++
PERF_TEST_OPT(MiscFeaturesTest, SysCallPerf, 10000, 5000)
{
  getppid();
}
```

以上代码以10000 calls/s的频率执行，同时一共执行5000 ms。如果频率和时长其中某个参数不想修改，只需填入-1（表示默认值）。

方式二，使用gtest命令来来修改（这种方式可以override方式一的修改），只需在命令行上使用-hz及-time两个选项来分别指定执行频率和时长参数。这种方式适用于在执行时临时修改不同的参数以观测比较执行结果的场景。以下为一例：

```
./gtestx_examples --gtest_filters='SimpleTest.*' -hz=10000 -time=10000
```

#### 实现和代码

gtestx的实现原理并不复杂，它其实是hack了gtest的用例宏，在用例层之上又包装一层性能测试逻辑。

如有兴趣，你可以在这里[https://github.com/mikewei/gtestx]()获取代码，其中也包括一些文档和示例代码。

#### 小结

本文简单介绍了gtestx以及其常用的做性能单元测试的方法，希望对你的工作有所帮助。



