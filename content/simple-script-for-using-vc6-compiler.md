Title: 方便使用 VC6 编译器的小脚本
Date: 2009-12-30 18:46
Tags: C, sh, wine
Category: Script
Slug: simple-script-for-using-vc6-compiler
Author: Xidorn Quan

因为一些原因，有时候不得不在 Linux 下使用 VC6 编译器。（比如学校的作业要求能在 VC6 下编译通过之类的要求）之前的用法太麻烦了，要把待编译的文件复制到 VC6 的安装目录，还要写很长的一串东西。要是能像调用 GCC 那么方便就好了~

于是就有了下面这个小脚本：

    :::bash
    #!/bin/bash
    # - * - coding: UTF-8 - * -
     
    VC6_DIR="这里写上VC6的安装地址"
     
    BIN="$VC6_DIR/VC98/Bin"
    export INCLUDE="$VC6_DIR/VC98/Include"
    export LIB="$VC6_DIR/VC98/Lib" 
     
    ARGS=
     
    while getopts "o:c" optname
    do
      case "$optname" in
      "o")
        ARGS="$ARGS /o$OPTARG"
        ;;
      "c")
        ARGS="$ARGS /c"
        ;;
      esac
    done
     
    wine "$BIN/CL.EXE" $ARGS ${@:$OPTIND}

然后把他放在 PATH 里面的某个目录下 (我放在了用户级的 /home/upsuper/bin 里，这个似乎要自己添加就是了)，然后给这个文件加上可执行属性，最后只要在需要的地方执行：

    :::bash
    vc6 xxx.cpp

就解决了~

不过从这个脚本中也可以看出，它的功能还不太完善，不对，是很不完善。目前支持设置输出文件名和阻止执行连接。我很想加入很多其他的参数，不过不知道该怎么弄……

参考资料：

* [Linux 技巧: Bash 参数和参数扩展](http://www.ibm.com/developerworks/cn/linux/l-bash-parameters.html)
* [vc6.0的INCLUDE 和LIB路径如何修改? – CSDN社区](http://topic.csdn.net/t/20060903/21/4995578.html)
