Title: 动态切换 Linux 使用的 CPU 数量
Date: 2011-10-19 14:04
Tags: Linux
Category: Script
Slug: dynamic-modify-cpu-number
Author: Xidorn Quan

由于要测试一些代码，其运行结果会受到多核并行的影响，所以希望能够调整使用的 CPU 数量。网络上之前看到的方法是在内核的启动参数上添加一个 `maxcpus`，但是如果这样的话每切换一次都要重启一次，是在太麻烦了。想想 Linux 应该是很强大的，所以可以动态修改 CPU 数量才对。

无意中看到 Linux 代码的 `Documentation` 文件夹下有个文件叫做 `cpu-hotplug.txt`，于是就看了一下，发现可以在 `/sys/devices/system/cpu` 看到代表各 CPU 的文件夹按照 `cpuX` 的命名方式，如 `cpu0`、`cpu1`、`cpu2` 等。这些文件夹里面有一个 `online` 文件，如果其值为0则禁用该 CPU，如果为1则启用该 CPU。注意，这里需要 `root` 权限哦。

因为我只要在单核和多核之间切换，所以我写了两个脚本放在 `/usr/local/sbin` 里面：

singlecore

    :::bash
    #!/bin/bash
     
    cpus_dir="/sys/devices/system/cpu"
     
    for cpu in $(ls "$cpus_dir" | grep 'cpu[0-9]\+')
    do
        cpu_online="$cpus_dir/$cpu/online"
        if [[ -e "$cpu_online" && $(cat $cpu_online) = 1 ]]
        then
            echo 0 > "$cpu_online"
        fi
    done

multicore

    :::bash
    #!/bin/bash
     
    cpus_dir="/sys/devices/system/cpu"
     
    for cpu in $(ls "$cpus_dir" | grep 'cpu[0-9]\+')
    do
        cpu_online="$cpus_dir/$cpu/online"
        if [[ -e "$cpu_online" && $(cat $cpu_online) = 0 ]]
        then
            echo 1 > "$cpu_online"
        fi
    done

之后需要切换的时候，只要运行 `sudo singlecore` 或者 `sudo multicore` 就可以了~

顺便说一句，我当时在想，如果我禁用了所有的 CPU 会怎么样呢？结果发现 `cpu0` 是没有 `online` 文件的，也就是 Linux 至少保证一个 CPU 处于可用状态。
