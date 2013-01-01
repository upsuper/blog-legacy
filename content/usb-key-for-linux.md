Title: 为 Linux 做一把 USB “钥匙”
Date: 2011-01-14 20:25
Tags: Linux, Security
Category: Technique
Slug: usb-key-for-linux
Author: Xidorn Quan

我曾经很早以前就在想，能不能将U盘作为登入我系统的验证机制。当时的想法是，这样比较有趣~不过后来发现另外一个重要的用途就是，防止在众目睽睽之下输入密码……

这个[问题提出](https://groups.google.com/group/shlug/browse_thread/thread/d507a796d11df859/15b0bce269c51f7f)后，邮件列表里很快就有人告诉我，Linux 已经有一个现成的机制了，这就是 pam_usb。不过我在网上搜了半晌都没搜到相关的中文资料，前几天弄成了，就写出来供大家参考~

其实来说，是很简单的。首先，当然，要准备一个U盘~（废话），然后安装 pam_usb。在 Ubuntu 下的话，源里就有，可以输入命令

    :::sh
    sudo apt-get install pamusb-tools

直接安装。当然如果你连终端也懒得打开，可以直接点击这里：[安装 pamusb-tools](apt:pamusb-tools)。当然，在后面的步骤中你终归还是要打开终端的，所以还是先开了吧~这个东西目前暂时还没什么图形界面的样子（当然做一个相信也不难）。

另外，Fedora 源里有 pam_usb 包，Arch Linux 似乎在 AUR 里有，在 Gentoo 中似乎是被默认屏蔽的，可以通过下面指令安装：

    :::sh
    echo "sys-auth/pam_usb" >> /etc/portage/package.keywords
    emerge -av ">=sys-auth/pam_usb-0.4.1"

其他的发行版也可以直接从他们的[项目主页](http://pamusb.org/)下载源码包编译安装~

安装好了以后，首先插入你作为钥匙的U盘，然后在终端中运行

    :::sh
    sudo pamusb-conf --add-device MyUSBDevice

其中的 `MyUSBDevice` 可以任意修改，只是一个标识符而已。接下来根据提示操作即可。如果你的电脑此时连接着超过一个U盘、移动硬盘，或者某个U盘、移动硬盘包含超过一个分区（就像我给U盘分了2个区），就会提示选择安装到哪里。设置完确认保存到配置文件即可。

下面添加认证用户，下面的命令是添加我为认证用户的：

    :::sh
    sudo pamusb-conf --add-user upsuper

原教程里面写的是添加 root 我认为是没有必要的，添加 `sudoer` 应该是已经足够了的。这条命令几乎不问什么问题，直接就完成了……这样以后在使用这把钥匙的时候就可以不需要输入相应用户的密码了。

最后最重要的一步，是编辑认证系统的配置文件。打开 `/etc/pam.d/common-auth`（对于 Gentoo 来说是 `/etc/pam.d/system-auth`），将下面这行插入到所有条目的前面：

    auth    sufficient      pam_usb.so

现在你的 USB 钥匙已经可以用了！现在，另外再打开一个终端，随便 `sudo` 点什么，然后你应该不会再看到输入密码的画面，取而代之的是下面的东西：

    * pam_usb v0.4.2
    * Authentication request for user "upsuper" (sudo)
    * Device "MyUSBDevice" is connected (good).
    * Performing one time pad verification...
    * Access granted.

然后运行成功了！不仅 `sudo` 可以验证，包括 `gksu` 和登入框等等都已经可以使用这把钥匙直接略过不需要输入密码了。

现在你已经成功的制作了一个属于自己的 USB 钥匙！

现在我们看看还有什么地方可以继续改进的……

我们注意到，无论我们是否连接了我们的钥匙，以后 `sudo` 的时候都会出现那些讨厌的提示，怎么办呢……？其实这完全也是可以解决的：打开 `/etc/pamusb.conf` 文件，我们发现这其实根本就是一个 XML 文件……在里面的 `<defaults>` 标签中间添加

    :::xml
    <option name="quiet">true</option>

保存后就直接生效了~

除此之外，我们发现在这里，我们的钥匙和原来的密码之间是一个替代的关系，如果你希望利用这个钥匙附加上密码提高安全性的话，可以将上面在 `/etc/pam.d/common-auth` 里面加入的那行中的 `sufficient` 改成 `required`，如果你干脆就不想再用密码了，那就把密码的那些删掉，留下一个 `required` 的 `pam_usb`~

话说这还真是强大呐~不过用了这个以后，你这个USB钥匙也得要好好保管鸟~不过其实对我来说最爽的无外乎以后在众目睽睽之下不需要再手动输入密码啦~

## 参考资料


* [HOWTO: pam_usb login with USB memory stick – Ubuntu Forums](http://ubuntuforums.org/showthread.php?t=17571)
* [doc:quickstart [pam_usb]](http://pamusb.org/doc/quickstart)
* [doc:configuration [pam_usb]](http://pamusb.org/doc/configuration)
