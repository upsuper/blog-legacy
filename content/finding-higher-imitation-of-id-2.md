Title: 寻找更高仿的 ID 第二季
Date: 2010-09-11 15:31
Tags: Python, Tieba
Category: Script
Slug: finding-higher-imitation-of-id-2
Author: Xidorn Quan

继上一篇文章之后，我又下大力气对这个程序做了许多修改，在精确度和速度方面似乎都有些许提高。在此推出第二季~

## 使用真正的 12px 宋体

在上一次的程序中使用的 PIL 似乎是因为不支持宋体 `ttc` 文件中对于小字体下优化的点阵形式，才在选择小于 `19px` 的字号时不能正确渲染汉字。考虑到这一点，我就想到把 `ttc` 文件里面 `12px` 的点阵字体单独提取出来使用，毕竟贴吧上面显示 ID 都是用这个字号显示的。

使用 FontForge 提取出来了 `simsun-12.bdf` 文件，就是宋体 `12px` 下的点阵。参考 PIL 的手册，发现 PIL 不能直接使用 `.bdf` 文件，需要使用一个叫做 `pilfont` 的脚本转换成专有的 `.pil` 文件才行。我想转换就转换呗。`simsun-12.bdf` 一个 2.4MB 的文件，转换完就剩不到 100KB，我就觉得肯定有问题，用 PIL 导入，发现还是不能渲染中文。后来知道，这个 `.pil` 文件根本不支持非拉丁字母的字符，它的储存空间限定了 256 个字符……

无奈了，这意味着 PIL 完全无法支持中文点阵了……

当然，办法总归是有的，那就是——抛弃 PIL！为什么我能有这样的想法呢，因为看到 .bdf 文件是 UNIX 标准的。UNIX 标准意味着什么呢？记不记得 UNIX 有一个非常好的传统叫做，尽量使用纯文本。是的，这虽然让有些文件会变得太大，不过同时也让这些东西更容易被其他程序读取，而 `.bdf` 恰好即使这么一种文件。

这样读取 `.bdf` 点阵字体文件的程序自己写不就好了，什么额外的库都不需要……当然，纠错性极弱就是了~

    :::python
    line_count = 0
    
    def read_split(f):
        global line_count
        line_count += 1
        line = f.readline()
        if not line:
            return False
        return line[:-1].split()
    
    with open('simsun-12.bdf', 'r') as f:
        chars = {}
        line = ['']
        try:
            # 获得字符总数
            while line[0] != 'CHARS':
                line = read_split(f)
            # 读取所有字符
            for i in range(int(line[1])):
                assert read_split(f)[0] == 'STARTCHAR'
                line = read_split(f)
                # 编码
                assert line[0] == 'ENCODING'
                char = unichr(int(line[1]))
                line = read_split(f)
                # 绘制参数
                assert line[0] == 'BBX'
                width, height, x1, y1 = [int(x) for x in line[1:]]
                x0, y0 = (1 + x1, 12 - y1 - height)
                # 准备绘制文字
                base_image = [[0 for x in range(14)] for y in range(14)]
                # 读取并绘制文字
                line = read_split(f)
                assert line[0] == 'BITMAP'
                for y in range(y0, y0 + height):
                    line = read_split(f)[0]
                    bits = int(line, 16)
                    bits >>= len(line) * 4 - width
                    for x in range(x0 + width - 1, x0 - 1, -1):
                        base_image[y][x] = bits & 1
                        bits >>= 1
                chars[char] = base_image
                # 结束这个文字
                line = read_split(f)
                assert line[0] == 'ENDCHAR'
            line = read_split(f)
            assert line[0] == 'ENDFONT'
        except AssertionError:
            print 'line', line_count
            raise

这样所有的字符就被读入，并变成一个单色像素二位数组了。

当然，这个性能很低，在我的机器上转换读取文件的 2W+ 字符大概需要 18s，这可能也是为什么 PIL 要选择进行转换。事实上，使用纯文本储存一直以来都给 UNIX 风格的这一类软件带来一定性能缺陷。不过其实，这很值得，因为方便。

不过这个时间确实是太长了，更何况我们到目前为止还什么都没处理。怎么办呢？两个想法：一、优化代码；二、保存处理的数据。

第一种，基本上是没什么希望了，而且即使能优化，估计效果也不会太好，可能省个几秒封顶了。第二个显然不是个坏想法。

Python 在数据的持续化方面还是有很多现成的东西的，比如 `pickle` 什么的。不过那个速度太慢，而且是纯文本！好吧，偶尔我也会不喜欢纯文本，因为在这里意义不大……因此选择了 `marshal`。`marshal` 也是一个用于数据持续化的库，不过仅能对 Python 的内部类型进行。我会看中它最重要的原因就是它的应用范围极其有限，只能持续化内部类型。如果一个 Python 标准库，它有很明显的限制，却没有标明不推荐或在新版中被剔除，说明它必然有一个其他库不可及的优势。对于 `marshal`，我猜它的优势就是效率。

使用 `marshal` 就很简单了……

    :::python
    import marshal
    marshal.dump(chars, open('base_image', 'wb'))

这样后面的处理不需要不断重复这个低效的步骤了~

## 处理文字图像

原来是用 PIL 处理文字图像，现在抛弃 PIL 了，就得自己写了……不过这样也很好，自由发挥的空间很大了~

我猜用的是 Matrix67 大牛的那种在附近留阴影的方法，不过似乎我写的不够好就是了，怎么测试效果都不大理想。除此之外，纯 Python 实现的算法效率和 PIL 这种包装还是没得比，很简单的算法却慢的不得了……

下面是目前的处理代码：

    :::python
    #!/usr/bin/python
    # - * - coding: utf8 - * -
    
    import marshal
    import copy
    
    base_chars = marshal.load(open('base_image', 'rb'))
    chars = {}
    for char, image in base_chars.iteritems():
        new_image = []
        for y in range(14):
            new_row = []
            for x in range(14):
                if image[y][x]:
                    value = 81
                else:
                    value = 0
                if y > 0 and image[y-1][x]:
                    value += 4
                if y < 13 and image[y+1][x]:
                    value += 4
                if x > 0 and image[y][x-1]:
                    value += 4
                if x < 13 and image[y][x+1]:
                    value += 4
                new_image.append(value)
        chars[char] = new_image
    
    marshal.dump(chars, open('advanced_data', 'wb'))

处理效果不是很理想就是了，耗时大概也是 30s+。

## 寻找相似字符

其实这个部分就是一样的了……直接贴代码好了……

    :::python
    #!/usr/bin/python
    # - * - coding: utf8 - * -
    
    import marshal
    
    chars = marshal.load(open('advanced_data', 'rb'))
    
    def image_diff(image1, image2):
        ret = 0
        for v1, v2 in zip(image1, image2):
            ret += (v1 - v2) ** 2
        return ret
    
    remember_number = 5
    try:
        searched = marshal.load(open('searched_chars', 'rb'))
    except IOError:
        searched = {}
    while True:
        input_string = unicode(raw_input('ID: '), 'utf8')
        if not input_string:
            break
        for char in input_string:
            if char not in chars:
                continue
            if char not in searched:
                diff_data = []
                image = chars[char]
                for c, v in chars.iteritems():
                    if c == char:
                        continue
                    diff_data.append((image_diff(v, image), c))
                diff_data.sort()
                searched[char] = diff_data[:remember_number]
            print char,
            for item in searched[char]:
                print u'({0}, {1})'.format(*item),
            print
        print
    
    marshal.dump(searched, open('searched_chars', 'wb'))

和上次不同的是查找过的字会被保存下来，效率可以高一些……

## 继续改进

现在的主要问题就是如何提高相似度的识别精度了……目前的想法是通过逐像素比对测试两个字的相似度，最多加一些模糊化什么的处理。doggy 提出一个想法是计算连通区域面积的比例，我个人认为不大可行……我的想法是识别文字的笔画，把文字的骨架弄出来，然后对比什么的，可能效果更好吧？

不知道各位还有没有其他什么想法？

## 参考资料

* [我想把simsun.ttf里的12号点阵字体单独提取出来做成一个ttf – 北大中文论坛](http://www.pkucn.com/viewthread.php?tid=202692)
* [Glyph Bitmap Distribution Format – Wikipedia](http://en.wikipedia.org/wiki/Glyph_Bitmap_Distribution_Format)
* [Glyph Bitmap Distribution Format (BDF) Specification – Adobe Developer Support](http://www.adobe.com/devnet/font/pdfs/5005.BDF_Spec.pdf)
* [marshal — Internal Python object serialization – Python documentation](http://docs.python.org/library/marshal.html)
