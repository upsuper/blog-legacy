Title: 在 Git 中隐藏文件里的部分内容
Date: 2013-10-11 15:44
Tags: Git
Category: Technique
Slug: mask-part-of-file-in-git
Author: Xidorn Quan

我们都知道，在 Git 里面，可以通过 `.gitignore` 或者 `.git/info/exclude` 来指定一些不希望被 Git 跟踪的文件。但是有的时候，我们只希望一个文件中的一部分内容不被跟踪，这样的内容可能是密码这类不希望被其他人知道的信息，也可能是一些临时性的调试语句，只是自己看看，不想让它进入源中。

往常来说，因为无法指定让 Git 不跟踪这些内容，就不得不在添加修改的时候用 `git add -p` 仔细剔除它们，不仅十分麻烦，而且在没有其他修改的时候仍然会看到本地工作目录存在修改，很是不爽。我就想，既然 git 如此强大，这件事情应该也是可以做到的吧？答案当然是肯定的。

简单地来说，方法是利用 git 的 filter 功能，这个功能支持在 checkout 之后更新工作目录时对文件进行修改，也支持 checkin 经过过滤器修改的文件内容（而不是工作目录中的文件本身）。而后面那个，正可以实现我所需要的功能。

在这里举一个简单的例子来说明这个方法。如果我有一个 `config` 文件，其中有一行 `key = "123456"`，我希望其中的密钥不要被跟踪，但是保留这个文件的其他内容，那么执行下面的命令就可以了：

    :::bash
    git config filter.remove_pw.clean "sed 's/^key *= *\".*\"/key = \"\"/'"
    echo "config filter=remove_pw" >> .git/info/attributes

这里稍微解释一下，第一个命令向 `.git/config` 添加了一个名叫 `remove_pw` 的 filter，其中 `remove_pw` 这个名字可以任意指定。后面跟的 `sed` 命令的作用就是清理掉文件中的密钥部分。第二个命令则向 `.git/info/attributes` 添加了一条规则，即对 `config` 文件应用刚才添加的那个 filter。

知道了原理一切就很简单了，如果我们不希望被跟踪的不是一行密钥，而是一段调试代码，我们可以在这个代码的两端加上标识，比如说这样：

    :::c
    #include <stdio.h>
    #define N 10
    int main() {
        int i;
        int f[N];
        f[0] = f[1] = 1;
        for (i = 2; i < N; i++)
            f[i] = f[i - 1] + f[i - 2];
        /* IGNORE */
        for (i = 0; i < N; i++)
            printf("%d: %d\n", i, f[i]);
        /* END */
        scanf("%d", &i);
        printf("%d\n", f[i]);
        return 0;
    }

然后将上面的 filter 后面的命令修改为 `awk '/\/\* IGNORE \*\// {f=1} f!=1 {print} /\/\* END \*\// {f=0}'`，最后在 `attributes` 里面将它应用到 `*.c` 文件，这样就大功告成了~

可以看到，Git 的 filter 功能还是相当强大的。在 Git 文档的示例里面还介绍了一个例子使用 filter 功能来加密解密文件呢。具体可以参考 [gitattributes 的文档中 filter 小节](https://www.kernel.org/pub/software/scm/git/docs/gitattributes.html#_tt_filter_tt)。
