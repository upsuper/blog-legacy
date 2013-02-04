Title: BMP to HTML 小程序
Date: 2010-03-10 17:46
Tags: Erlang, HTML
Category: Script
Slug: bmp-to-html-tool
Author: Xidorn Quan

什么叫 BMP 到 HTML 呢……？就是生成一个网页，里面用不同颜色的字符拼出那个图片……很无聊的功能嗯，而且原理上说，生成的 HTML 文件如果要表现整个 BMP 的所有细节，大小肯定大大超过原 BMP 文件……

为什么会做这个呢？主要是受到我们 C++ 老师的启发，尝试去做的。不过我没有用 C++ 写，而是选用了寒假学的 Erlang，这也是我写的第一个 Erlang 程序。

为什么会选用 Erlang 呢？主要是基于两点原因：1、寒假学了半天，一点都没有练过，就拿这个来练练；2、看中了 Erlang 强大的模式匹配和比特语法。比特语法在 Erlang 里面原来是用来解决网络传输协议中的二进制数据的，不过这里拿来处理二进制文件着实是一个很好的选择~不过其实 Erlang 真正最重要的特性：面向并发，我完全没有用到，而是继续使用了顺序编程。主要是，BMP to HTML 没什么可以并发化的，而且就算并发了，也是大传输小计算，并没有什么很大的优势。因为是第一次写 Erlang 程序，如果有 Erlang 高手路过，还请多多指点咯~

另外一点，为什么选用 BMP 这种几乎被人抛弃的格式呢？因为最容易呗……而且我这里还用了 BMP 中最简单的一种：真彩色无压缩格式。这是最直接的图形表示方式了，就是一个点一个点的，每个点三个字节表示一种颜色。所以还是很简单的。

先看看最后的效果：
![效果图](|filename|/images/bmp-to-html-tool.png)

下面说干就干。

## BMP 文件结构

首先要知道的就是 BMP 的格式。在维基百科里的相关条目有很详细的记载，甚至于文件头的 C 结构都已经直接给出来了。不过这个当然对我没有很大的用途，因为用的是 Erlang 嘛~识别文件头要比 C 方便多了~下面我先说说 BMP 的文件结构：

### BMP 文件头

一个 BMP 文件最开始是 BMP 文件头。这个文件头的大小是固定的14字节，包含 BMP 文件的 Magic Number (魔术数字，即文件识别码) 2字节、整个文件大小4字节、保留区4字节、图像数据在文件中的起始位置4字节。

其中 Magic Number 我们最常见的是 Windows BMP 文件的 `BM`，除此之外，还有 `BA`、`CI`、`CP`、`IC`、`PT` 这几种，应该是不太常见吧。

整个文件的大小其实我觉得在这里是没有什么用途的，而整整4字节的保留区也是令我很难理解的。整个14字节的头当中，最重要的应该就是图像数据的偏移。如果这个偏移量是相对于整个文件而言的，因此直接把读取指针指向偏移量的位置就是图像数据。

### DIB 头

这个信息其实是关于图像描述最有意义的部分。这里面将会描述这张图片几乎一切你需要的信息。

DIB 头最开始的32位，也就是4个字节，描述了 DIB 头的长度。根据 DIB 头的长度不同，共有4种 BMP 图片：

* 12字节 – OS/2 V1 格式，OS/2 和 Windows 3.0 以后所有版本支持
* 64字节 – OS/2 V2 格式
* 40字节 – Windows V3，Windows 3.0 以后所有版本支持
* 108字节 – Windows V4，Windows 95/NT4 以后所有版本支持
* 124字节 – Windows V5，Windows 98/2000 以后所有版本支持

鉴于 Windows V4、V5 都与 V3 的格式兼容，我在程序里面事实上仅仅实现了 V3 的头识别。

V3 头的40字节分布，我觉得已经很复杂了……分别是：

* DIB 头大小，4字节，这里为40
* 图像宽度，4字节
* 图像高度，4字节
* Color planes (不知道是什么东西)，2字节，必须为1
* 颜色位数，2字节，可以为1、4、8、16、24、32，我的程序中只识别24位色
* 压缩方式，4字节，我的程序里只能识别不压缩的……值为0
* 图像信息大小，4字节，与整个文件大小不一样，这里表示的是图像信息那块数据的大小
* 横向分辨率，4字节，单位像素/米
* 纵向分辨率，4字节，单位像素/米
* 调色板颜色数，4字节，如果为0则默认为2n
* 重要颜色数量，4字节，不知道干什么用的，直接忽略好了……

介绍完了这个 DIB 头，其实很很没意思就是了。真正有用的，也就这么几个：长度、宽度。如果要识别能力强一点，识别一下调色板相关的，也就差不多了。

### 调色板

因为我没有用到调色板，所以也没有细看这块内容，直接跳过……

### 图像数据

显然对于我的最基本选择要求：24位色、无压缩，图像信息是很简单的，就是每个像素占3个字节。

不过仍然有两点需要注意：1、颜色储存依次是是蓝、绿、红，而非红、绿、蓝；2、每行的字节数必须是4的整倍数。也就是 BMP 文件每行最后会有1-3个字节用于补足，这个补足用字节数正是宽度除以4的余数。

## Erlang 程序

BMP 图片的基本信息探究清楚了，下面就可以开始写咯~

### 导出函数

第一部分就是模块的声明和导出函数

    :::erlang
    -module(bmp).
    -export([bmp_to_html/2]).
    
    -define(ULI, unsigned-little-integer).
    
    bmp_to_html(Bmp, Html) ->
        case bmp_to_html(Bmp) of
            {ok, Bin} ->
                file:write_file(Html, Bin);
            Error ->
                {Bmp, Error}
        end.
    
    bmp_to_html(File) ->
        case file:open(File, [read, binary, raw]) of
            {ok, F} ->
                {ok, ["<span style=\"font-size: 1px; line-height: 1px;\">",
                        convert_bmp(F),"</span>"]};
            Error ->
                {File, Error}
        end.

我在 `bmp` 模块内导出了一个 `bmp_to_html` 的函数，他读取一个 `bmp` 文件，转换并写入一个 `html` 文件。中间定义了一个宏 `ULI`，很容易理解的~其实定义这个的主要原因是那个 little……由于 Erlang 最初是为网络协议做的这个比特语法，因此默认是 Big endian 的，而 x86 系的 CPU 却总是使用 Little endian 的……

### 读取 BMP 文件信息

    :::erlang
    convert_bmp(F) ->
        {Offset, {Size, Width, Height}} = read_header(F),
        {ok, Bin} = file:pread(F, Offset, Size),
        L = binary_to_list(Bin),
        each_pixel(Width, Height, L).
    
    read_header(F) ->
        {ok, Bin} = file:pread(F, 0, 14),
        case Bin of
            <<"BM", _FileSize:  32/?ULI,
                    _Creator:   32/?ULI,
                    Offset:     32/?ULI
            >> ->
                {Offset, read_dib_header(F)};
            _Other ->
                {error, unsupported, _Other}
        end.
    
    read_dib_header(F) ->
        {ok, Bin} = file:pread(F, 14, 4),
        case Bin of
            <<40:32/?ULI>> ->
                {ok, << Width:  32/?ULI,
                        Height: 32/?ULI,
                        _:      16/?ULI,
                        _:      16/?ULI,
                        0:      32/?ULI,
                        Size:   32/?ULI
                >>} = file:pread(F, 18, 20),
                {Size, Width, Height};
            _Other ->
                {error, unsupported, _Other}
        end.

很明显，为了简化程序，BMP 文件头和 DIB 头的识别都只写了一点点，够用就行~

### 转换图像数据

这是最后的部分了，将图像数据转换为 HTML 代码~

    :::erlang
    each_pixel(Width, Height, Bin) ->
        each_pixel(Width, Height, Width, Height, Bin, []).
    
    each_pixel(0, 0, _, _, _, Output) ->
        Output;
    each_pixel(0, Y, Width, Height, Bin, Output) ->
        NewBin = drop_begin(Bin, Width rem 4),
        NewOutput = [new_line() | Output],
        each_pixel(Width, Y-1, Width, Height, NewBin, NewOutput);
    each_pixel(_, 0, _, _, _, Output) ->
        Output;
    each_pixel(X, Y, Width, Height, Bin, Output) ->
        [B,G,R|NewBin] = Bin,
        NewOutput = [to_html(R, G, B) | Output],
        each_pixel(X-1, Y, Width, Height, NewBin, NewOutput).
    
    drop_begin(Bin, 0) ->
        Bin;
    drop_begin([_|Bin], Num) ->
        drop_begin(Bin, Num-1).
    
    new_line() ->
        ["<br>"].
    
    to_html(R, G, B) ->
        io_lib:format("<font color=\"#~.16B~.16B~.16B\">█</font>", [R, G, B]).

这是最后的部分了~其中字符选用了一个非常黑的字符“█”嗯~

至于为什么用 `font` 这个不推荐的标签，其实是我觉得这个文件本身就很冗长了，再变成样式就……所以……

### 使用

差不多了，整个程序全部拼在一起，就出来了~

最后，在 Erlang 的命令行中输入：

    :::erlang
    c(bmp).
    bmp:bmp_to_html("BMP 文件", "HTML 文件").

就好了~

我用了一个 2.6KB 的 BMP 文件，转换出了一个 26.3KB 的 HTML 文件……真够大……

## 参考资料

* [BMP file format – Wikipedia, the free encyclopedia](http://en.wikipedia.org/wiki/BMP_file_format)
* [Erlang Community – Converting Between Octal and Hex – Trapexit](http://www.trapexit.org/Converting_Between_Octal_and_Hex)
