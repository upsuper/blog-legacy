Title: 写了个小小的 vc-fix
Date: 2010-03-21 10:40
Tags: C, Microsoft
Category: Technique
Slug: little-vc6-fix.md
Author: Xidorn Quan

我们的 C++ 老师给我布置了 C++ 的大作业来替代其他同学交的无聊题目。大作业的第一题就是完整的高精度库，并且要求使用运算符重载。因为原来用 C 写过，这次写，思路上没有太大问题，不过全部程序被我 C++ 化了，代码看过去很诡异……呃……

我自己的机子上，自然使用 g++ 编译，不过我猜老师会要求 VC6 能够编译……我就用[上次安装的 VC6](|filename|/simple-script-for-using-vc6-compiler.md) 编译了一下，发现好几个错误和无数警告……其实也是我意料之中的。

其中我觉得最讨厌的莫过于 `for` 循环的循环变量不被视为 `for` 循环的局部变量这一点，导致大量变量被其认为是重复定义，这个是 VC6 和标准就语言上相去最远的问题了……不想每个都去改，麻烦死了。

在网上一找，还真找到一个简单的方法：

    :::c
    #define for if(0) ; else for

其实我也不知道这个是什么机理，不过真的很管用！

另外一个就是 VC6 的标准库中没有 `max` 和 `min` 函数，这个也很囧，于是也自己写了一个。

以前做网页的时候经常写 `ie-fix.css` 文件，今天我弄 VC6 遇到这些问题，于是我也写了个 `vc-fix.h` 文件。M$ 真是一个需要 fix 的公司，什么时候出一个 `m$-fix.com` 好了……

最后贴出我的 `vc-fix.h`，主要解决 VC6 下 `for` 循环变量的问题和 `max`、`min` 函数未定义的问题：

    :::c
    #ifndef _H_UPSUPER_VC_FIX_
    #define _H_UPSUPER_VC_FIX_
     
    #ifdef _MSC_VER
    #   if _MSC_VER <= 1200
    #       define for if (0); else for
     
    template <class T>
    inline const T& max(const T& a, const T& b) {
        return a > b ? a : b;
    }
     
    template <class T>
    inline const T& min(const T& a, const T& b) {
        return a < b ? a : b;
    }
     
    template <class T, class Compare>
    inline const T& max(const T& a, const T& b, Compare comp) {
        return comp(a, b) ? b : a;
    }
     
    template <class T, class Compare>
    inline const T& min(const T& a, const T& b, Compare comp) {
        return comp(a, b) ? a : b;
    }
     
    #   endif
    #endif
     
    #endif // _H_UPSUPER_VC_FIX_

最后要解决的就是警告的问题。其实我很无语的是，所有的警告都是在 VC6 自己的头文件里面的……VC6 自己提示可以添加 `/GX` 来消除那些警告。于是我不得不再次修改我的编译脚本：

    :::bash
    #!/bin/bash
    # - * - coding: UTF-8 - * -
     
    VC6_DIR="这里写上VC6的安装地址"
     
    BIN="$VC6_DIR/VC98/Bin"
    export INCLUDE="$VC6_DIR/VC98/Include"
    export LIB="$VC6_DIR/VC98/Lib" 
     
    ARGS=
     
    while getopts "o:cG:" optname
    do
        case "$optname" in
        "o")
            ARGS="$ARGS /o$OPTARG"
        ;;
        "c")
            ARGS="$ARGS /c"
        ;;
        "G")
            ARGS="$ARGS /G$OPTARG"
        ;;
        esac
    done
     
    wine "$BIN/CL.EXE" $ARGS ${@:$OPTIND}

到了最后，再补充一点点吧。VC6 发现了两个额外的错误，我觉得应该不是不兼容的问题。就是我重载的 `operator++` 和 `operator--` 不小心忘记写 `return *this;` 了，g++ 没有给我任何提示的编译通过了，而 VC6 则将这个视为错误。

在这个问题上，我同意 VC6 的看法，虽然我怀疑可能 g++ 自己加上了那句话，不过我觉得这个应该至少给出一个警告而非什么都不说。当然，可能一个 fatal error 太过了点……

参考资料：

* [VC6中FOR语句的变量声明问题](http://www.rugesy.cn/it/u20091012_22_34fe5127-ddfe-44fe-86f1-13afc360a794.html) 回复的7楼
* [如何在程序调试阶段，判断当前的编译器是vc6的编译器还是intel8.0的编译器？ – CSDN社区](http://topic.csdn.net/t/20041101/19/3511737.html)
* [max – C++ Reference](http://www.cplusplus.com/reference/algorithm/max/)
* [VC6.0不支持标准库函数max和min – C++技术 – 51CTO技术博客](http://panpan.blog.51cto.com/489034/103074/)
* [warning C4530:C++ exception handler used, but unwind semantics are not enabled. Specify -GX – CSDN社区](http://topic.csdn.net/t/20040909/19/3357414.html)
* [_MSC_VER_百度百科](http://baike.baidu.com/view/1276757.html)
