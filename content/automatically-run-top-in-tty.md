Title: 在 tty 里添加一个开机自启动的任务管理器
Date: 2010-03-31 10:31
Tags: Linux
Category: Technique
Slug: automatically-run-top-in-tty
Author: Xidorn Quan

每当感觉到系统卡的时候，最好的方法无外乎进入 tty，登入，打开一个 `top` 监视。可是每次到了需要的时候才去开，打开的效率自然不敢恭维。于是便想，每次开机的时候，记起来就跑到 tty 下面去先开起来。不过这件事情总归是麻烦的，于是才有了现在的方案。

既然 tty 那么多，那我们就把他利用一下吧~让他开机自动在某个 tty 里面启动 `top` 无疑最方便了~

首先呢，我找到了 Ubuntu 里面 tty 配置存放的地方 `/etc/init/ttyX.conf`，其中的 X 便是 tty 的编号，我这里选择了 `tty6.conf`。打开这个文件，结构简单极了，看到里面

    :::bash
    exec /sbin/getty -8 38400 tty6

就知道，肯定和 `getty` 有关系。`man getty` 里面查到可以通过 `-l` 参数设置登入程序替代 `/bin/login`。查了一下 `man login`，发现可以通过 `-f username` 的方式不进行验证地登入。

于是我就在 `/bin` 下面新建了一个 `autologin` 文件（其实理论上放哪里都可以，不过最好要用 root 权限创建，不然可以乱改就不好了），里面写上

    :::bash
    #!/bin/sh
    /bin/login -f upsuper

给这个文件加上可执行属性，接着将 `/etc/init/tty6.conf` 里面刚才那一行改成

    :::bash
    exec /sbin/getty -8 -l '/bin/autologin' 38400 tty6

重启。

进入 tty6 发现没有效果，还是提示用户名，无语……于是输入了用户名 upsuper，结果发现没有要求密码，直接进入了。我退出登入，再输入 root，发现依然没有要求密码而直接进入了 upsuper 权限。

再查查 `man getty`，发现那个请求用户名是 `getty` 输出的，里面提到了 `-n` 参数，可以消除对用户名的请求，以及 `-i` 参数，不输出请求前的文字（在我的 Ubuntu 里面就是“Ubuntu 9.10”）。于是上面那行被改成了

    :::bash
    exec /sbin/getty -8in -l '/bin/autologin' 38400 tty6

重新启动，发现已经可以自动进入。

不过我要的不是这个效果~

其实简单地说，我那个要实现也不难，按照现在的情况，就是在 `~/.bashrc` 里面加上一行判断的事情了。不过我可不想这样。这样的话如果退出了 `top` 就会进入命令行。我的想法是，永远不让他进入命令行，这样看过去比较爽~

于是我就倒腾起了 `login` 程序的 `FAKE_SHELL`，如果在 `autologin` 脚本里改变环境变量，根本影响不了 `login` 程序，无论我改 `FAKE_SHELL`，还是 `SHELL`，都没有用，`login` 仍然义无反顾地进入了 `bash`……

最后我就想，唉，其实 `autologin` 脚本就是一正常脚本，只不过在登入的时候以 root 权限运行嘛，那我直接在里面运行 `top` 不久行了~考虑到权限因素，就是用 `su` 把权限改一下，不就解决问题了么？

于是最终版的 `autologin` 就出炉了：

    :::bash
    #!/bin/sh
    su -c '/usr/bin/top' upsuper

这个最后效果是什么样的呢？就是 `top` 以我的用户权限运行，然后点击 `q` 退出就会重新启动一个 `top`。这就是我要得效果了~很好很强大~算是合理的利用了一个 tty 了。现在只要点击 Ctrl-Alt-F6 就可以有现成的任务管理器了~

其实根据这个思路，tty 可以做的事情还很多。本来那个什么 `-l` 啦，`-n` 什么的，是拿来做自定义登入验证方式的，我觉得这个也大有文章可做~最后再感叹一下，Linux 实在太强大了~

补充：

这篇文章被我投递到了 LinuxTOY 上面，然后下面有人提到使用 `htop` 代替 `top`，我试了一下，貌似 `htop` 的资源占用要比 `top` 高出许多，因此我最后没有替换。

不过 `htop` 貌似确实好用很多，如果需要的话，只要安装 `htop` 后（Ubuntu 源里是有的），然后把 `autologin` 里面的

    :::bash
    su -c '/usr/bin/top' upsuper

改成

    :::bash
    su -c '/usr/bin/htop' upsuper

下面只要进入那个 tty 点 `q` 退出当前 `top`，马上就会自动替换为 `htop` 启动了~
