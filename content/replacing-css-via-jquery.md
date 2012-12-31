Title: 基于 jQuery 的 CSS 更换术
Date: 2009-10-18 20:57
Tag: jQuery, CSS
Category: Technique
Slug: replacing-css-via-jquery
Author: Xidorn Quan

最近开始写一中的新选歌系统，这次要大改，顺便练手。

想加入换肤功能（不然女生肯定觉得老是蓝色不好……），而且我想到的换肤，最简单的方式就是换 CSS，把界面颜色、图形相关的内容放入皮肤的 CSS 中就很容易了~不过问题是换肤呢？

正好新系统中因为客户端代码可能非常强大，准备引入 jQuery 框架来简化开发，便学了一些。于是我就想，能不能通过 jQuery 来解决呢？

首先，我给出了下面这个简单的页面：

    :::html
    <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
      "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
    <html xmlns="http://www.w3.org/1999/xhtml" xml:lang="zh-CN">
    <head profile="http://gmpg.org/xfn/11">
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <title>福州一中 学校音乐征集</title>
    <link rel="stylesheet" href="" id="theme" type="text/css" media="all" />
    <script type="text/javascript" src="jquery.js"></script>
    <script type="text/javascript" src="theme.js"></script>
    <style type="text/css">
    html, body { height: 100%; width: 100%; }
    </style>
    </head>
    <body>
    Hello world!
    </body>
    </html>

然后我开始用了一段 jQuery 手册里的某段示例代码：

    :::js
    $('<link rel="stylesheet" href="' + 
        (t++ & 1) + '.css" id="theme" type="text/css" media="all" />')
        .appendTo('head');

成功了，不过查看处理后的代码，发现大量冗余代码出现在 head 尾部……又查了查，发现了 jQuery 里面的另外一个好用的函数，于是上面代码就改为：

    :::js
    $('<link rel="stylesheet" href="' + 
        (t++ & 1) + '.css" id="theme" type="text/css" media="all" />')
        .replaceAll("#theme");

没有冗余代码出现，而且 IE6 都可以正常使用！jQuery 的兼容性果然超群……

然后我们想，这样每次都要重建标签，会不会很慢呢？如果能直接改属性或许不错~再查查，我们发现下面方法：

    :::js
    $("#theme").attr({ href: (t++ & 1) + '.css' });

又简洁看过去又高效~再试试 IE6，仍然没有问题哦~

最后给出实验用各完整代码：

    :::js
    var t = 0;
    $(function() {
      $(document.body).click(function() {
        $("#theme").attr({ href: (t++ & 1) + '.css' });
      });
    });

第一个 CSS 文件：

    :::css
    body {
      background: blue;
      color: yellow;
    }

第二个 CSS 文件：

    :::css
    body {
      background: #000;
      color: #fff;
    }
