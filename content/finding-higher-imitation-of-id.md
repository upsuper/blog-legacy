Title: 寻找更高仿的 ID
Date: 2010-09-09 18:22
Tags: Python, Tieba
Category: Script
Slug: finding-higher-imitation-of-id
Author: Xidorn Quan

今天大学军训完了，不想做什么正经事，就想到前一段时间想做的寻找相似汉字的程序，用以寻找更高仿的贴吧 ID。用程序来寻找相似汉字，从另一个角度，也是从 Matrix67 大牛的一篇日志里得到的启发。不过 Matrix67 大牛使用的是 Mathematica 来寻找，我不大会 Mathematica，就想用我熟悉的 Python 来解决，毕竟 Python 是一个很强大的东西~

其实寻找的思路很简单，就是把某个汉字当作图片弄出来，让后对比两个图片的相似程度。因此做这个程序的第一步就是研究如何用 Python 处理图片和文字。Python 有一个非常著名的第三方库，名叫 Python Imaging Library，简称 PIL，就是专门用来处理图片的。

## 文字 to 图像

PIL 可以很轻松的将文字转换为图像，并且提供了虽然不能说是强大，但暂时够用的图像处理函数。

处理文字生成的图像，显然和彩色没有太大关系，因此可以使用灰度图像节省计算需要的空间和时间。此外我们知道，文字到图像有一个中间媒介，就是字体。因此我们必须先确定我们需要的字体和大小。用 Firebug 考查贴吧用于显示 ID 的字体和大小，发现是宋体 12px。这样参考 PIL 的手册，就有了最基本的文字到图片的代码了：

    :::python
    import Image, ImageDraw, ImageFont
    image = Image.new('L', [14, 14], 255)
    draw = ImageDraw.Draw(image)
    font = ImageFont.truetype('simsun.ttc', 12)
    draw.text((1, 1), u'好', font=font)
    image.save('hw.png')

看过去这个代码是正确的，不过在查看 `hw.png` 后，我发现并没有出现预期的“好”字，而是一堆混乱的东西，不知何故。后经过不断实验，发现只有当初始的文字大小设定为19或以上时，文字才可以被正确地绘制出来，_-b

于是不得不修改这段代码，让它先绘制一个大的，再缩放成需要的大小：

    :::python
    import Image, ImageDraw, ImageFont
    image = Image.new('L', [28, 28], 255)
    draw = ImageDraw.Draw(image)
    font = ImageFont.truetype('simsun.ttc', 24)
    draw.text((2, 2), u'好', font=font)
    # 抗锯齿方式缩放
    image = image.resize((14, 14), Image.ANTIALIAS)
    image.save('hw.png')

效果还不错。

## 枚举汉字

然后来找一种方式来枚举汉字。参考维基百科的 GB2312 条目，确定了 GB2312 中汉字区的编码，为第一个字节 `0xB0-0xF7`，第二个字节 `0xA1-0xFE`，其中 `D7FA-D7FE` 是空的，这样共有6763个汉字拿来搞。

其实还是挺少的就是了……

于是枚举汉字的代码：

    :::python
    for i in range(0xB0, 0xF8):
        chr_i = chr(i)
        for j in range(0xA1, 0xFF):
            if i == 0xD7 and j >= 0xFA and j <= 0xFE:
                continue
            char = unicode(chr_i + chr(j), 'gb18030')
            print char,

## 初步成果

现在结合上面两段，得到了下面的初步代码：

    :::python
    #!/usr/bin/python
    # - * - coding: UTF-8 - * -
    
    import Image, ImageDraw, ImageFont, ImageFilter
    
    font = ImageFont.truetype('simsun.ttc', 24)
    
    def get_char_data(char):
        image = Image.new('L', [28, 28], 255)
        draw = ImageDraw.Draw(image)
        draw.text((2, 2), char, font=font)
        image = image.resize((14, 14), Image.ANTIALIAS)
        return list(image.getdata())
    
    def diff_of_data(data1, data2):
        ret = 0
        for i in zip(data1, data2):
            ret += abs(i[0] - i[1])
        return ret
    
    chars = {}
    for i in range(0xB0, 0xF8):
        chr_i = chr(i)
        for j in range(0xA1, 0xFF):
            if i == 0xD7 and j >= 0xFA and j <= 0xFE:
                continue
            char = unicode(chr_i + chr(j), 'gb18030')
            chars[char] = get_char_data(char)
    
    remember_number = 5
    input_string = unicode(raw_input('ID: '), 'utf8')
    for input_char in input_string:
        if input_char not in chars:
            print input_char
            continue
        input_data = chars[input_char]
        diff_list = []
        for char, data in chars.iteritems():
            if char == input_char:
                continue
            diff_list.append((diff_of_data(data, input_data), char))
        diff_list.sort()
        print input_char,
        for diff, char in diff_list[:5]:
            print u'({0}, {1})'.format(diff, char).encode('utf8').ljust(11),
        print

发现效果还有待提高，并且每次测试一个新 ID 都要做一次预处理，每次预处理大概需要6s左右。

## 目前代码

对上面的代码做了一些修改，包括对文字图像做一些处理，以使其查找出的相似文字能更接近人的感觉，并且让一次预处理可以做多次比对，得到了下面代码：

    :::python
    #!/usr/bin/python
    # - * - coding: UTF-8 - * -
    
    import Image, ImageDraw, ImageFont, ImageFilter
    
    font = ImageFont.truetype('simsun.ttc', 24)
    
    def get_char_data(char):
        image = Image.new('L', [28, 28], 255)
        draw = ImageDraw.Draw(image)
        draw.text((2, 2), char, font=font)
        image = image.resize((14, 14), Image.ANTIALIAS)
        # XXX 期待更好的处理方式
        image = image.filter(ImageFilter.SHARPEN).filter(ImageFilter.SMOOTH)
        return list(image.getdata())
    
    def diff_of_data(data1, data2):
        ret = 0
        for i in zip(data1, data2):
            ret += abs(i[0] - i[1])
        return ret
    
    chars = {}
    for i in range(0xB0, 0xF8):
        chr_i = chr(i)
        for j in range(0xA1, 0xFF):
            if i == 0xD7 and j >= 0xFA and j <= 0xFE:
                continue
            char = unicode(chr_i + chr(j), 'gb18030')
            chars[char] = get_char_data(char)
    
    remember_number = 5
    while True:
        input_string = unicode(raw_input('ID: '), 'utf8')
        if not input_string:
            break
        for input_char in input_string:
            if input_char not in chars:
                print input_char
                continue
            input_data = chars[input_char]
            diff_list = []
            for char, data in chars.iteritems():
                if char == input_char:
                    continue
                diff_list.append((diff_of_data(data, input_data), char))
            diff_list.sort()
            print input_char,
            for diff, char in diff_list[:5]:
                print u'({0}, {1})'.format(diff, char).encode('utf8').ljust(11),
            print
        print

## 继续改进

目前这个程序测试的效果还行，不过还有待进一步改进。可改进的地方包括上面标出 XXX 的部分，即对文字图像进行处理的部分，或许可以有更好的处理方式。除此之外，如果能直接绘制出 12px 的字，必然也会有更好的效果。如果增大字库，从 GB2312 增加到 GB18030 的字库，可以用来对比的文字也会更多，也就更可能找到相似的字了~

这个，还有继续改进的余地嗯~有时间就继续努力咯……

## 参考资料


* [用Mathematica寻找最相似的汉字 – Matrix67](http://www.matrix67.com/blog/archives/2907)
* [The Image Module – Python Imaging Library Handbook](http://www.pythonware.com/library/pil/handbook/image.htm)
* [The ImageFont Module – Python Imaging Library Handbook](http://www.pythonware.com/library/pil/handbook/imagefont.htm)
* [The ImageDraw Module – Python Imaging Library Handbook](http://www.pythonware.com/library/pil/handbook/imagedraw.htm)
* [The ImageFilter Module – Python Imaging Library Handbook](http://www.pythonware.com/library/pil/handbook/imagefilter.htm)
* [GB 2312 – 维基百科](http://zh.wikipedia.org/zh-cn/GB_2312)
