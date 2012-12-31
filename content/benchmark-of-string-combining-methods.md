Title: 对字符串加长和数组合并的效率比较
Date: 2009-02-13 12:59
Tags: Javascript, PHP, Benchmark
Category: Research
Slug: benchmark-of-string-combining-methods
Author: Xidorn Quan

对于字符串累加的处理，在 PHP 或 JavaScript 中似乎都可以通过类似 += (.= for PHP) 的方式实现，但有不少人抱怨道，这种方式效率很低。事实上，在我还在用 VB 的时候我就注意到这样的效率很低，当时的效率低是因为累加需要反复申请内存，而解决方法也很简单，就是用 Space$ 命令事先申请内存，然后用 Mid$ 来修改，这样效率大大提高！

然而在这里就不一样了，PHP 和 JavaScript 的内存机制我不是非常了解，同时我们似乎也不再使用预申请的方法来加速了（似乎也比较困难……），而是直接用上了 += 这样的符号。

下面就是问题了：这样的效率低吗？

很多人（包括我自己最初）根据自己的想象，认为使用添加数组内容，最后合并数组为字符串，这样效率比简单的 += 要快，但我经过反复实验认为并不是这样的。首先是 JavaScript，我使用的测试代码如下：

    :::js
    function getTime() {
        return (new Date()).getTime();
    }
    function testit() {
        var st, et, ta, tb;
        for (i = 0; i < 10; ++i) {
            st = getTime();
            t = '';
            for (j = 0; j < 100000; ++j) {
                t += 'test';
            }
            et = getTime();
            ta = et - st;
     
            st = getTime();
            a = [];
            for (j = 0; j < 100000; ++j) {
                a.push('test');
            }
            t = a.join();
            et = getTime();
            tb = et - st;
            console.log(ta, ' ', tb);
        }
    }

以下是我在 Firefox 3.0.6 下执行的结果：

    196	207
    195	215
    199	203
    196	203
    200	197
    110	111
    104	111
    107	111
    108	113
    105	107

可以看出，+= 的效率并不比数组合并低，甚至略快于数组合并。

下面我们看看 PHP 呢？

下面是测试用 PHP 代码：

    :::php
    <?php
    for ($i = 0; $i < 10; ++$i) {
        $st = microtime(true);
        $str = '';
        for ($j = 0; $j < 10000; ++$j) {
            $str .= 'test';
        }
        $et = microtime(true);
        printf("%.7F\t", $et - $st);
     
        $st = microtime(true);
        $arr = array();
        for ($j = 0; $j < 10000; ++$j) {
            $arr[] = 'test';
        }
        $str = implode($arr);
        $et = microtime(true);
        printf("%.7F\n", $et - $st);
    }
    ?>

结果则如下：

    0.0039599	0.0461071
    0.0070710	0.0408709
    0.0052779	0.0322330
    0.0061541	0.0294971
    0.0051920	0.0230181
    0.0039949	0.0204601
    0.0038750	0.0246589
    0.0043111	0.0245950
    0.0047810	0.0232542
    0.0038841	0.0230379

可以看出，数组合并的效率远低于字符串直接叠加！

上一次用 spidermonkey-bin 中的 js 命令做了一下实验，结果在这种条件下，字符串叠加的效率也是远优于数组合并，而不像在 Firefox 中这样。

我对于这些脚本语言对字符串和数组的具体实现机理不是很清楚，但可以肯定的是，个个常用的脚本引擎都对字符串处理做了许多优化，我猜测（因为 Ubuntu 下暂时没有 stable 的 Chrome）Chrome 的 V8 引擎中，字符串直接叠加的效率将继续远超数组。因此，我们不应该想当然地认为某某方式效率高，而应该用测试结果说话……
