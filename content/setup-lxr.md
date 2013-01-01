Title: 安装配置 LXR
Date: 2011-10-03 15:07
Tags: Linux
Category: Technique
Slug: setup-lxr
Author: Xidorn Quan

专业课学习操作系统，满心欢喜地以为可以是 Linux 代码导读，结果选用了一本八十年代的教材，介绍 UNIX v6 的。于是自己从图书馆借来了内核开发的入门读物《Linux 内核设计与实现》。既然是介绍内核的书，自然少不了代码，但是书中又不可能将每个提到的代码的相关信息全部写出来，这时就得自己去查看代码。查看代码的话，虽然 Linux 的代码可以很容易地得到，但那来看终归有些麻烦，各种调用需要查找半天。于是想起了很有名的一个网站 [LXR](http://lxr.linux.no)，Linux 交叉引用。这个网站好是好，就是网络原因速度太慢，于是就想自己装一个。

先说一句，下面的安装环境都是64位 Gentoo。

最开始找到了 [LXR Cross Referencer](http://sourceforge.net/projects/lxr/) 这个项目，一看最后更新时间还挺新，看起来是一直都还在维护的。然后在 Gentoo 的网站上查到 LXR 是可以直接用 portage 安装的，于是安装，安装相关的包，最后放弃了。出于什么原因我也不记得了，最后一个原因肯定是不好看，肯定达不到上面那个网站的效果，所以就放弃了，到那个网站上去下载 LXR 分出来的版本 LXRng。（话说这个 ng 不会是表示 next generation 吧？）

## 安装支持库

首先从 LXR 的网站上用 `git` 把代码下载下来，

    :::sh
    git clone git://lxr.linux.no/git/lxrng.git

然后我打开了里面的 `INSTALL` 文件，里面写着好多好多库。先说结论吧，在 Gentoo 的官方源里面，有两个库是没有的，分别是 `Search-Xapian` 和 `CGI-Ajax`，这两个最后安装。

最首要的是先安装 PostgreSQL，由于 LXR 是用 Perl 写成的，所以在 `USE` 里面加入了 Perl，然后安装。安装完成以后，安装 PostgreSQL 的 Perl 库 `DBD-Pg`。接下去安装 `Cgi-Simple`、`HTML-Parser`、`HTML-Entities`、`Term-ProgressBar`、`Devel-Size`、`Template-Toolkit`，这些都很容易，直接安装就可以了。（虽然我确定这些包的名称还费了些时间）

然后是 Apache 和它的 `mod_perl`，因为之前安装了，并没有太大困难，这里也不详述了。

最麻烦的问题来了，对于源里没有的 `Search-Xapian` 和 `CGI-Ajax` 怎么办呢？先把 Xapian 的主要部分给安装了吧。

安装 `xapian` 和 `xapian-bindings` 这两个包。因为这两个包的最新版本对 amd64 平台都是 unstable 的，所以要在 `/etc/portage/package.keywords` 里面加入

    =dev-libs/xapian-1.2.7-r1 ~amd64
    =dev-libs/xapian-bindings-1.2.7-r2 ~amd64

（是的，在 Gentoo 的查询系统上显示，`xapian` 的 1.2.5 是稳定版本，我也曾经试图安装那个版本，然后仅安装非稳定版的 `-bindings`，但是之后安装的 `Search-Xapian` 还是会要求新的 1.2.7，所以就这样吧。另外一般状况下，最前面是写 `>=` 的，但是我出于个人喜好和完美主义，写了 `=`。）接下去直接安装这两个包即可。记得检查已经在 `USE` 里面加入了 `perl`。

接下去安装那两个包。

Gentoo 有个很神奇的工具，也是我这次才发现的，叫做 `g-cpan`，可以把 CPAN 上面的包自动打包安装为 `portage` 的包。不过如果是第一次使用必须要先配置一下，在 `/etc/make.conf` 最后加上

    ACCEPT_KEYWORDS="amd64"
    PORTDIR_OVERLAY="/usr/local/portage"

（虽然这个配置看过去很简单，不过因为一开始忽略了这件事情，所以纠结了很长时间……）

接下去用 `g-cpan` 安装就可以了

    :::sh
    sudo g-cpan -g CGI::Ajax Search::Xapian
    sudo emerge CGI-Ajax Search-Xapian

至此需要安装的东西就已经全部装完了，下面进入第二阶段~

## 配置数据库

这个很简单了，不过在配置之前要先把自己将会用到的用户加入到 `postgres` 组里以保证可以访问。最重要的是要把之后的 `apache` 用户加入到 `postgres` 组里，否则后面会出现一些状况。

然后 `su` 到 `postgres` 用户里，添加用户 `root`，并把 `root` 设置为管理员（因为之后生成的时候需要用到）

    :::sh
    createuser root

然后创建 LXR 的数据库

    :::sh
    createdb lxrng

大体上这样就没问题了。

## 调整配置文件并建立工作目录

我出于完美主义的原因，将 LXR 的工作目录放在了 `/var/lib/lxrng` 里面，如果你没有那些奇怪的癖好，完全可以直接在自己的文件夹下面放置这些东西。

首先要设置配置文件，将 LXR 根目录下的 `lxrng.conf-dist` 复制为 `lxrng.conf`，然后打开修改。里面大体上还是比较清晰的，如果只是要做一个 Linux 代码的交叉引用的话，大体上按照里面的配置，修改第10行

    :::perl
    my $gitrepo = LXRng::Repo::Git
        ->new('/var/lib/lxrng/repos/linux-2.6/.git',
          release_re => qr/^v[^-]*$/,
          author_timestamp => 0);

里面的那个路径，使其指向你放置代码的 git 源（一般是代码文件夹下的 `.git`）。

如果你没有使用 git 来抓取代码，而是直接下载某个版本的代码，如 v3.1，可以放置到比如 `/var/lib/lxrng/repos/linux/v3.1`，那么这个部分就修改为

    :::perl
    use LXRng::Repo::Plain;
    $plainrepo = LXRng::Repo::Plain
        ->new('/var/lib/lxrng/repos/linux');

即可。（注意上面的 `$gitrepo` 在下面还有使用过一次，如果修改的话需要一并修改）

接下去是第19行

    :::perl
    my $search  = LXRng::Search::Xapian->new('/var/lib/lxrng/text-db/linux-2.6');

需要在某个位置建立一个 `text-db` 文件夹，然后将上面的路径修改为你建立的那个文件夹的路径即可。同样的操作也发生在第29行

    :::perl
        # Must be writable by httpd user:
        'cache'	      => '/var/lib/lxrng/cache',

注意这个文件夹需要对 `apache:apache` 可写。我的做法是把这个文件夹的组设置为 `apache`，然后设置权限为`0775`，当然也可以直接把所有者设置为 `apache` 然后保留原来权限。

注释掉下面这行

    :::perl
        'ctags_flags' => ["-I\@$LXRng::ROOT/lxr-ctags-quirks"],

不要问我为什么，这个我真不知道，总之如果部注释掉一会儿会出错。

最后是要生成引用的版本和默认显示的版本：

    :::perl
    'ver_list' => [$gitrepo->allversions],
     
    'ver_default' => 'v2.6.20.3',

我强烈建议你将 `$gitrepo->allversions` 修改为你想看的几个版本，甚至于只有一个版本也是没有问题的即使你有完整的历史记录，因为每个版本都需要生成很长时间，而且似乎过程很不稳定，如果没有特别的原因，最好不要生成太多版本。如果是不用 git 源的话，只要把你放在那个文件夹里的对应版本号填进去就可以了，最后修改默认显示的版本。

如果还想添加其他的代码，只要把代码最后 `return` 的大括号里面的部分复制一遍，根据需要修改就可以了。

## 初始化数据库及生成交叉引用

首先要添加一个符号链接

    :::sh
    sudo ln -s /usr/bin/exuberants-ctags /usr/bin/ctags-exuberants

接下去没什么太大的差别，就是进入程序所在目录，然后

    :::sh
    ./lxr-db-admin linux --init
    ./lxr-genref linux

值得一说的是，这个过程非常非常非常漫长，在我的 i7 本上的虚拟机里，一跑至少三四个小时，而且看起来还很不稳定，不时会自动强制退出，而且退出以后可能会出现一些问题导致无法继续。这个问题比较严重，遇到这个问题如何解决放到之后的部分再来说吧。

## 配置 Apache

最后来配置 Apache。直接把文件夹下的 `apache2-site.conf-dist-mod_perl` 复制到 `/etc/apache/vhosts.d/10_lxrng.conf`，然后打开这个文件，将里面的所有 `@@LXRROOT@@` 和 `@@LXRURL@@` 根据自己的情况替换为相应的路径就可以了。然后重新启动 Apache

    :::sh
    sudo /etc/rc.d/apache restart

## 问题解决

由于原来的版本在我这里基本上没什么希望能生成结束，所以我对这个程序做了一些修改，这个修改后的版本可以直接在我的 GitHub 上面找到：[upsuper/lxrng](https://github.com/upsuper/lxrng)。如果需要的话，可以不使用原来官方的代码而直接使用我修改过的代码。主要的差别有几点：一是消除了生成交叉引用时过大量的输出信息；二是增加了交叉引用生成时刷写 Xapian 索引的频率，以减少退出重做时可能出现的错误；三是修正了一些最后浏览时可能遇到的问题。

当然生成的时候还是可能会出错，这我也没办法。如果生成时被意外中断，重新执行又出现错误，可以将我修改的那个程序里面的 `lxr-genref` 第336行

    :::perl
        #warn("here $docid\n");

的注释符去掉，重新运行 `lxr-genref`，然后查看当程序报错时停止的那个编号，比如 12345。然后执行 `psql lxrng` 进入 lxrng 数据库，执行

    :::sql
    DELETE FROM hashed_documents WHERE doc_id>=12345

然后再次执行 `lxr-genref`。这个过程可能反复一两次，直到不会报错位置。

如果用原始版本的话，最后在浏览的时候使用搜索，有可能会出现500错误以及无法显示出来的情况，如果出现，可以参考我做的修改。

## 后记

配置这个 LXR 真是折腾死我了，费了好大功夫，最后也总算是成功了。

另外真是很久很久没有在这里写东西了。也正因为这个过程实在太麻烦了，所以来写一写，权当一个记录。不过现在有 GitHub 这种东西，倒是好得多了。

## 参考文档


* [How to setup LXR – Step by Step guide « Mohamed Thalib's Blog](http://mohammadthalif.wordpress.com/2010/07/24/how-to-setup-lxr-%E2%80%93-step-by-step-guide-3/)
* [在自己的计算机上建立lxr源代码检索服务器 | kernelchina](http://www.kernelchina.org/node/241)
* [lxrng.install-gentoo_百度文库](http://wenku.baidu.com/view/8150646727d3240c8447ef2d.html)
* [Gentoo Linux Documentation — g-cpan Guide](http://www.gentoo.org/proj/en/perl/g-cpan.xml)

（貌似还有些别的参考资料，不记得是什么了……）

## 参考配置文件

最后最后贴一下自己的配置文件吧。配置文件里面声明了两个代码，一个是 Liunx 的，一个是 UNIX v6 的，Linux 是用 git，UNIX 是代码。程序全部放在 `/var/lib/lxrng` 里面，代码放在 `/var/lig/lxrng/repos` 里。

    :::perl
    # -*- mode: perl -*-
    # Configuration file
    #
    # 
     
    use LXRng::Index::PgBatch;
    use LXRng::Repo::Git;
    use LXRng::Repo::Plain;
    use LXRng::Search::Xapian;
     
    my $linuxrepo = LXRng::Repo::Git
        ->new('/var/lib/lxrng/repos/linux.git',
          release_re => qr/^v[^-]*$/,
          author_timestamp => 0);
    my $unixrepo = LXRng::Repo::Plain
        ->new('/var/lib/lxrng/repos/unix');
     
    my $index   = LXRng::Index::PgBatch->new(db_spec => 'dbname=lxrng;port=5432', 
                         db_user => "", db_pass => "",
                         # table_prefix => 'lxr'
                         );
    my $search  = LXRng::Search::Xapian->new('/var/lib/lxrng/text-db');
     
    return {
        'linux' => {
        'repository'  => $linuxrepo,
        'index'       => $index,
        'search'      => $search,
     
        'base_url'    => 'http://upsuper-gentoo/lxr',
        # Must be writable by httpd user:
        'cache'	      => '/var/lib/lxrng/cache/linux',
     
        'fs_charset'  => 'iso-8859-1',
        # Tried successively
        'content_charset' => ['utf-8', 'iso-8859-1'],
     
        'languages'   => ['C', 'GnuAsm', 'Kconfig'],
        #'ctags_flags' => ["-I\@$LXRng::ROOT/lxr-ctags-quirks"],
        #'ver_list'    => [$gitrepo->allversions],
        'ver_list'    => ['v2.6.34', 'v3.0'],
     
        'ver_default' => 'v2.6.34',
     
        'include_maps' => 
            [
             [qr|^arch/(.*?)/|, qr|^asm/(.*)|,
              sub { "include/asm-$_[0]/$_[1]" }],
             [qr|^include/asm-(.*?)/|, qr|^asm/(.*)|,
              sub { "include/asm-$_[0]/$_[1]" }],
             [qr|^|, qr|^asm/(.*)|,
              sub { map { "include/asm-$_/$_[0]" }
                qw(i386 x86_64) }],
             [qr|^|, qr|(.*)|,
              sub { "include/$_[0]" }],
             ],
        },
        'unix' => {
        'repository'  => $unixrepo,
        'index'       => $index,
        'search'      => $search,
     
        'base_url'    => 'http://upsuper-gentoo/lxr',
        # Must be writable by httpd user:
        'cache'	      => '/var/lib/lxrng/cache/unix',
     
        'fs_charset'  => 'iso-8859-1',
        # Tried successively
        'content_charset' => ['utf-8', 'iso-8859-1'],
     
        'languages'   => ['C', 'GnuAsm'],
        #'ctags_flags' => ["-I\@$LXRng::ROOT/lxr-ctags-quirks"],
        'ver_list'    => ['v6'],
     
        'ver_default' => 'v6',
     
        'include_maps' => 
            [
             [qr|^|, qr|(.*)|,
              sub { "sys/$_[0]" }],
             ],
        },
    };
