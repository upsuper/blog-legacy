Title: 鼠标控制音乐播放的小程序
Date: 2009-11-03 20:06
Tags: Python, GTK
Category: Program
Slug: use-mouse-as-music-player-controller
Author: Xidorn Quan

看这个标题一定很奇怪：难道我以前控制音乐播放不用鼠标么？这个文章的标题看起来像鼠标刚刚发明的推广期的文章……不过，当然不是这样的！

其实只是想：如何把我的小本合上放书架上，当作一个音乐播放器+功放，并用我的无线6键鼠当遥控器遥控控制之。

想想其实还是蛮有意义的功能，这样我做作业的时候可以不用戴耳机，不用用MP3，直接把本当播放器；同时，我不会看到屏幕上的东西，可以安心做作业~再看看我的6键无限鼠，那额外的功能键平时根本不用，也想不出能有什么用……这么好的东西就这样被我浪费了……（话说，拿本当音乐播放器是不是更浪费？）

说干就干！

首先提出构想：左键用于暂停和播放，滚轮调节音量，侧边的两个功能键用来切换上一首和下一首。至于右键和中键……再说吧，说不定以后可以扩展更多功能？说不定以后高兴了弄个鼠标手势什么的~嘿嘿

接下来查找资料。印象中我的 Audacious 是可以用 D-Bus 控制的。简单地查阅了一下相关资料，发现了一个叫做 MPRIS 的播放器控制接口。为此，我还专门学习了一下 python-dbus 的使用。

插一句话：python-dbus 怎么没有中文教程啊！英文教程看得还是蛮吃力的……

连接播放器的代码很简单：

    :::python
    import dbus
    bus = dbus.SessionBus()
    player = bus.get_object('org.mpris.audacious', '/Player')

其中 `org.mpris.audacious` 指的是 audacious，支持 MPRIS 的其他播放器还可以有如下：

* `org.mpris.bmp`
* `org.mpris.vlc`
* `org.mpris.xmms2`

至于那个 `/Player` 指的是我们要获取的是一个叫 `Player` 的对象，用于控制播放器。在 MPRIS 中除了 `Player` 以外，还有根对象 (`/`) 和 `TrackList` 对象 (`/TrackList`) 两个，具体的用法其文档里详细说明。这里我只用到了 `Player` 对象……

看看 `Player` 对象的方法还挺多，这里我用了这几个：

* `Next()` - 切换到下一首歌曲
* `Prev()` - 切换到上一首歌曲
* `Pause()` - 暂停/继续
* `Play()` - 播放歌曲
* `VolumeGet()` - 获取当前音量
* `VolumeSet()` - 设置音量

根据前面写的需求，这些已经足够了~

怎么用这些？很简单嘛！就像平时在 Python 里面调用类的方法一样~类型转换？不，那完全不需要你关心！

Python 程序拥有控制播放器的能力了，可是鼠标呢？如何捕获鼠标的动作呢？

想想……屏幕都关了……你还能指望他为你显示什么呢？干脆建立一个窗口把整个屏幕盖住算了！然后让他截获鼠标事件。

快速学习了一下 pygtk 的用法（其实就是照着 Hello World 打了一遍），差不多就知道怎么用了。窗口的建立代码大约如下：

    :::python
    window = gtk.Window(gtk.WINDOW_TOPLEVEL)
    window.fullscreen()
    window.show()
    gtk.main()

这段代码很好理解了，第一行就是创建窗口对象，第二行令其全屏，第三行设置其显示，第四行开始 GTK 的主循环。

下面就是鼠标事件的问题了~添加事件用的是类似下面的代码：

    :::python
    window.connect('key-press-event', key_press)
    window.connect('destroy', lambda widget, data = None: gtk.main_quit())

第一个是注册了键盘按键的事件捕获，将 `key-press-event` 事件绑定上 `key_press` 这个函数，而把窗口的销毁事件 `destroy` 连接到退出 GTK 主循环。说一句废话，连接事件的语句要放在 `gtk.main()` 之前……`key_press` 主要用来实现当点击 escape 时退出：

    :::python
    def key_press(widget, event, data = None):
      if gdk.keyval_name(event.keyval) == 'Escape':
        window.destroy()

好了，下面是鼠标点击事件 `button-press-event`：

    :::python
    def button_press(widget, event, data = None):
      print event
    window.connect('button-press-event', button_press)

试试？不行的！差了一番后，发现 Window 这个东西默认是不打开鼠标点击的事件捕获的，逼近大家很少会点窗口本身，都是点里面的按钮什么的……下面的代码要求窗口打开鼠标点击事件的捕获：

    :::python
    window.add_events(gtk.gdk.BUTTON_PRESS_MASK)

接下来就可以了~`event.button` 表示的是点击的键是哪个。检查了一下，左键是1，右键是3，中键是2，功能键1是8，功能键2是9。滚轮呢？

原来滚轮有自己的事件 `scroll`。

    :::python
    def scroll(widget, event, data = None):
      print event
    window.connect('scroll-event', scroll)

直接可以用了。识别滚轮方向的是 `event.direction`，应该是 `gtk.gdk.SCROLL_UP`、`SCROLL_DOWN`、`SCROLL_LEFT`、`SCROLL_RIGHT` 其中的一个。

下面根据最初的设想，把他们都拼起来就形成了最初的版本：

    :::python
    #!/usr/bin/env python
     
    import pygtk
    pygtk.require('2.0')
    import gtk
    from gtk import gdk
     
    import dbus
     
    def button_press(widget, event, data = None):
      button = event.button
      if button == 1:
        if player.GetStatus() == 2:
          player.Play()
        else:
          player.Pause()
      elif button == 9:
        player.Prev()
      elif button == 8:
        player.Next()
     
    def scroll(widget, event, data = None):
      if event.direction == gdk.SCROLL_UP:
        player.VolumeSet(player.VolumeGet() + 3)
      elif event.direction == gdk.SCROLL_DOWN:
        player.VolumeSet(player.VolumeGet() - 3)
     
    def key_press(widget, event, data = None):
      if gdk.keyval_name(event.keyval) == 'Escape':
        window.destroy()
     
    bus = dbus.SessionBus()
    player = bus.get_object('org.mpris.audacious', '/Player')
     
    window = gtk.Window(gtk.WINDOW_TOPLEVEL)
    window.connect('destroy', lambda widget, data = None: gtk.main_quit())
    window.add_events(gdk.BUTTON_PRESS_MASK)
    window.connect('button-press-event', button_press)
    window.connect('key-press-event', key_press)
    window.connect('scroll-event', scroll)
    window.fullscreen()
    window.show()
    gtk.main()

OK 看起来不错了，也能控制了。可是，作为一个程序员，怎么能不考虑各种意外呢？试想，如果我们控制着控制着，突然，一个不明真相的窗口弹了出来怎么办呢？！这是一个问题……

最容易想到的方法：把这个窗口永久置顶！

事实上也确实找到了这么个方法，就是把窗体注册成 dock。我们知道 dock 是需要永久置顶的~

用如下方法可以让窗体变成 Dock：

    :::python
    window.set_type_hint(gtk.gdk.WINDOW_TYPE_HINT_DOCK)

但是问题又来了：dock 似乎没法接受键盘输入。因为 dock 只是形式上覆盖，并且可以拦截一切在他上面发生的鼠标事件，可键盘焦点就未必在他那里了~

那么我们就不要键盘事件了吧~于是我就把键盘退出给删了，改成了双击退出~查了下资料，双击事件也通过 `button-press-event` 发送的，必须根据 `event.type` 来判断是单击、双击还是三击。此外，双击和三击中的每一次点击都会引发一次单击事件。

OK，于是我们把 button_press 函数改成下面这样：

    :::python
    def button_press(widget, event, data = None):
      if event.type == gdk._2BUTTON_PRESS:
        window.destroy()
        return
      elif event.type == gdk._3BUTTON_PRESS:
        return
      button = event.button
      if button == 1:
        if player.GetStatus() == 2:
          player.Play()
        else:
          player.Pause()
      elif button == 9:
        player.Prev()
      elif button == 8:
        player.Next()

然后删掉键盘处理，最后就成了这样：

    :::python
    #!/usr/bin/env python
     
    import pygtk
    pygtk.require('2.0')
    import gtk
    from gtk import gdk
     
    import dbus
     
    def button_press(widget, event, data = None):
      if event.type == gdk._2BUTTON_PRESS:
        window.destroy()
        return
      elif event.type == gdk._3BUTTON_PRESS:
        return
      button = event.button
      if button == 1:
        if player.GetStatus() == 2:
          player.Play()
        else:
          player.Pause()
      elif button == 9:
        player.Prev()
      elif button == 8:
        player.Next()
     
    def scroll(widget, event, data = None):
      if event.direction == gdk.SCROLL_UP:
        player.VolumeSet(player.VolumeGet() + 3)
      elif event.direction == gdk.SCROLL_DOWN:
        player.VolumeSet(player.VolumeGet() - 3)
     
    bus = dbus.SessionBus()
    player = bus.get_object('org.mpris.audacious', '/Player')
     
    window = gtk.Window(gtk.WINDOW_TOPLEVEL)
    window.connect('destroy', lambda widget, data = None: gtk.main_quit())
    window.add_events(gdk.BUTTON_PRESS_MASK)
    window.connect('button-press-event', button_press)
    window.connect('scroll-event', scroll)
    window.set_type_hint(gdk.WINDOW_TYPE_HINT_DOCK)
    window.fullscreen()
    window.show()
    gtk.main()

完成了！

以后就可以拿我的小本来当音乐播放器+功放啦~

参考资料：

* [MPRIS – XMMS2](http://wiki.xmms2.xmms.se/wiki/MPRIS)
* [dbus-python tutorial](http://dbus.freedesktop.org/doc/dbus-python/doc/tutorial.html)
* [PyGTK 2.0 Tutorial](http://www.pygtk.org/pygtk2tutorial/index.html)
* [PyGTK 2.0 Reference Manual](http://library.gnome.org/devel/pygtk/stable/)
* [[pygtk] how to generate scroll wheel mouse event.](http://www.daa.com.au/pipermail/pygtk/2004-January/006832.html)
* [如何让窗口置顶？ Linux/Unix社区 / 程序开发区 – CSDN社区 community.csdn.net](http://topic.csdn.net/t/20030910/11/2243673.html)
