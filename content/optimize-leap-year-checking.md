Title: 闰年判断的优化及其他
Date: 2009-10-10 21:11
Tags: C, Optimize
Category: Research
Slug: optimize-leap-year-checking
Author: Xidorn Quan

今天 Javran 发来短信给了一个短小的论年判断代码，并且问我是否认为有更简单的表达。下面是他最初给的代码：

    :::c
    return ((y & 3 != 0) ^ (y % 100 == 0) ^ (y % 400 != 0));

一切的探究就从这个代码开始了。

当然，这个代码是错的，因为疏忽了运算符的优先级，为达到本来的目的，这段代码大概应该这样改（测试代码3）：

    :::c
    return (((y & 3) != 0) ^ (y % 100 == 0) ^ (y % 400 != 0));

接着，我将其中“!=0”和“==0”可以进一步缩短，现在代码现在变成这样（测试代码4）：

    :::c
    return !!(y & 3) ^ !(y % 100) ^ !!(y % 400);

OK，对于抑或得到的思路，精简到这里差不多了。Javran 随后又给我了一个代码：

    :::c
    return !(y & 3) & ((!(y % 25) & !(y & 4)) | !!(y % 25));

我说这比我的代码长，他解释说这段代码模的规模比较小，应该快一些。我说再快快不过 if 句。话说，经过实验，这条语句似乎是错误的……

不过，这让我突然想起了对于 && 和 || 这样逻辑运算符的优化，我便想看看这个判断最快能至何~

我们先看看最传统的判断代码（测试代码1）：

    :::c
    if (y % 4 != 0) return 0;
    else if (y % 400 == 0) return 1;
    else if (y % 100 == 0) return 0;
    else return 1;

然后我们做点小小的优化（测试代码2）：

    :::c
    if (y & 3) return 0;
    else if (!(y % 400)) return 1;
    else if (y % 100) return 1;
    else return 0;

接着我就按着这个的判断方式的思路进行一点缩减：

    :::c
    return (y & 3) != 0 && (y % 100 == 0 || y % 400 != 0);

然后根据前面的方式缩短行（测试代码5）：

    :::c
    return !(y & 3) && (y % 100 || !(y % 400));

可以看出，这是目前最短的一个判断方式。

下面我做了30轮，每轮每个代码执行2,000,000次，取最短时间，得到如下结果：

    1: 0.043275
    2: 0.043562
    3: 0.093261
    4: 0.093184
    5: 0.036421

首先我们观察到，我最后推出的那个最短的式子是最快的，为什么？这是源于逻辑运算符的运算规则：对于 &&，如果前面项为假则不计算后项；对于 ||，如果前项为真则不计算后项。这就像 if 语句的递推作用：不做无谓的计算。而很显然，3和4的速度很慢，因为使用了异或运算，由于异或运算本身无法预测结果，必须把每一项都计算出来才行，因此慢了很多（比传统算法慢了一倍）。事实上按位运算应该都是这样。

接着我们看到，我对 Javran 最初代码的优化版本效率有一定提高，可能是因为将减法（比较运算实质是做减法）化为了位操作吧。而对传统代码的优化却反而减慢了它，或许是修改规则导致的副作用吧……不过我的最终优化代码还是快了不少，原因不明，或许 if 并不快？

应该有人会觉得奇怪，为什么要取最短时间，而不是平均值呢？事实上，我想在进行效率测试的时候，应该看最短时间。我们考虑测量效率时引入的误差出现在什么地方：CPU 肯定不会因为某个函数突然超频加速。那么有什么问题呢？因为我们用的都是分时系统，系统会不断的调度不同的线程使用 CPU。误差就在这里：时间会因为任务的切换而变长！因此测量结果只可能比实际值长，不可能比实际值短。所以要取最短时间。

OK，对闰年的探索暂告一段落。我们突然想起前面 Javran 所给出的“缩小模的规模带来效率提升”的论断。

因此我的测试程序增加了3个测试代码（测试代码6-8）：

    :::c
    return y % 25;
    return y % 100;
    return y % 400;

得到的结果让人吃惊：

    6: 0.106022
    7: 0.098434
    8: 0.098487

模25最慢，400次之，而100最快，一样原因不明。看起来难道模的速度和模的数有关系？

嗯，讨论了这么多，其实最初的问题——闰年判断的简化和优化——比较无聊，因为这段代码本身就不长，也几乎不可能被用于热点处。不过从这个过程中，我们看到了一些有趣的优化方式，虽然不能如算法改进那样降低复杂度，但这里有的时候常数也很重要，不是么？此外，还有一些关于效率测试的讨论。最后，我们还看到了一点神奇的结果，有待进一步探究咯~

最后贴出测试代码：

    :::c
    #include <stdio.h>
    #include <sys/time.h>
     
    #define TRUE  1
    #define FALSE 0
     
    #define CHECK(NUM)  if (check##NUM(i) != a) printf(#NUM)
     
    #define YEAR_AMOUNT 2000000
    #define TEST(NUM) \
      gettimeofday(&tv_s, &tz); \
      for (j = 1; j <= YEAR_AMOUNT; ++j) \
        check##NUM(j); \
      gettimeofday(&tv_e, &tz); \
      timeval_subtract(&tv_d, &tv_e, &tv_s); \
      if (mtime[NUM].tv_sec == 0 && \
          mtime[NUM].tv_usec == 0 || \
          tv_d.tv_sec < mtime[NUM].tv_sec || \
          tv_d.tv_sec == mtime[NUM].tv_sec && \
          tv_d.tv_usec < mtime[NUM].tv_usec) \
        mtime[NUM] = tv_d; \
      printf(#NUM " ")
     
    int y;
    struct timeval mtime[10];
    struct timezone tz;
     
    int check1(int y) {
      if (y % 4 != 0) return FALSE;
      else if (y % 400 == 0) return TRUE;
      else if (y % 100 == 0) return FALSE;
      else return TRUE;
    }
     
    int check2(int y) {
      if (y & 3) return FALSE;
      else if (!(y % 400)) return TRUE;
      else if (y % 100) return TRUE;
      else return FALSE;
    }
     
    int check3(int y) {
      return ((y & 3) != 0) ^ (y % 100 == 0) ^ (y % 400 != 0);
    }
     
    int check4(int y) {
      return !!(y & 3) ^ !(y % 100) ^ !!(y % 400);
    }
     
    int check5(int y) {
      return !(y & 3) && (y % 100 || !(y % 400));
    }
     
    int check6(int y) {
      return y % 25;
    }
     
    int check7(int y) {
      return y % 100;
    }
     
    int check8(int y) {
      return y % 400;
    }
     
    // 时间减法
    int timeval_subtract(result, x, y)
           struct timeval *result, *x, *y;
    {
      if (x->tv_usec < y->tv_usec) {
        int nsec = (y->tv_usec - x->tv_usec) / 1000000 + 1;
        y->tv_usec -= 1000000 * nsec;
        y->tv_sec += nsec;
      }
      if (x->tv_usec - y->tv_usec > 1000000) {
        int nsec = (x->tv_usec - y->tv_usec) / 1000000;
        y->tv_usec += 1000000 * nsec;
        y->tv_sec -= nsec;
      }
     
      result->tv_sec = x->tv_sec - y->tv_sec;
      result->tv_usec = x->tv_usec - y->tv_usec;
     
      return x->tv_sec < y->tv_sec;
    }
     
    int main() {
      int i, j;
     
      // 验证代码正确性
      for (i = 1; i <= YEAR_AMOUNT; ++i) {
        int a;
        a = check1(i);
        CHECK(2); CHECK(3); CHECK(4); CHECK(5);
      }
      printf("\nCheck complete!\n");
     
      // 开始计时
      struct timeval tv_s, tv_e;
      struct timeval tv_d;
      for (i = 0; i < 100; ++i) {
        printf("%d: ", i);
        TEST(1);
        TEST(2);
        TEST(3);
        TEST(4);
        TEST(5);
        TEST(6);
        TEST(7);
        TEST(8);
        printf("\n");
      }
     
      // 输出结果
      for (i = 1; i <= 8; ++i)
        printf("%d: %ld.%06ld\n", i, mtime[i].tv_sec, mtime[i].tv_usec);
     
      return 0;
    }

很久没写这种东西了……唉……
