Title: 关于饭否的两个小脚本
Date: 2011-02-02 12:06
Tags: Fanfou, Python
Category: Script
Slug: two-scripts-for-fanfou
Author: Xidorn Quan

前几天因为某些原因，我把饭否上所有的好友和关注者全部清空了。当然，如果没有程序的帮忙，估计还不等我删完我也就后悔了。

我没有那么狠心的把饭否的消息给清空，因为消息是不可恢复的（而且也太多了），但好友和关注者是可以的。做事情都给自己留后路显然是我一贯的风格，不然的话，我大概早从我家阳台跳下去了……

两段脚本都不长，第一段是备份饭否的好友列表和关注者列表的，做的毫无泛用性，因为 bash 编程我并不很熟，只知道可以用 `wget` 来抓取。本来这种事情其实可以直接用 API 抓的，不过我想抓下我能直接看的东西，所以最终还是导出了 Cookie 抓网页。

第一段代码如下：

    :::bash
    #!/bin/bash
    
    mkdir friends
    for i in {1..11}
    do
        wget -k -e robots=off --load-cookies cookies.txt -P friends/ \
            http://fanfou.com/friends/upsuper/p.$i
        sed -i -e "s/http:\/\/fanfou\.com\/friends\/upsuper\///g" \
            "friends/p.$i"
        sed -i -e "s/http:\/\/fanfou\.com\/friends/p.1/g" \
            "friends/p.$i"
        sed -i -e "s/http:\/\/fanfou\.com\/followers/..\/followers\/p.1/g" \
            "friends/p.$i"
    done
    
    mkdir followers
    for i in {1..8}
    do
        wget -k -e robots=off --load-cookies cookies.txt -P followers/ \
            http://fanfou.com/followers/upsuper/p.$i
        sed -i -e "s/http:\/\/fanfou\.com\/followers\/upsuper\///g" \
            "followers/p.$i"
        sed -i -e "s/http:\/\/fanfou\.com\/friends/..\/friends\/p.1/g" \
            "followers/p.$i"
        sed -i -e "s/http:\/\/fanfou\.com\/followers/p.1/g" \
            "followers/p.$i"
    done
    
    for f in friends/* followers/*
    do
        sed -i -e "s/http:\/\/fanfou\.com\/friends/..\/friends/g" \
            "followers/p.$i"
        sed -i -e "s/http:\/\/fanfou\.com\/followers/..\/followers/g" \
            "friends/p.$i"
    done

想用的话，其中前两个 for 循环的数字必须改成自己的实际情况，里面的 `upsuper` 也要全数替换为自己的饭否ID，还有一个 `cookies.txt` 文件，可以用 Firefox 的 Cookie Exporter 插件从浏览器导出。

这个脚本非常值得赞美的一点是自动修复了所有的链接，使得在本地的页面全部可以通过链接进入，不在本地的页面也可以链接到饭否。这也是我调了很长世间的东西……

第二段脚本就是颇具破坏性的了：

    :::python
    #!/usr/bin/env python
    # - * - coding: utf8 - * -
    
    import sys
    import json
    
    from socket import timeout
    from urllib import urlencode
    from base64 import b64encode
    from getpass import getpass
    from httplib2 import Http
    
    username = raw_input('Username: ')
    password = getpass('Password: ')
    auth = 'Basic ' + b64encode('%s:%s' % (username, password))
    headers = { 'Authorization': auth }
    
    h = Http(timeout=1)
    sys.stdout = sys.stderr
    
    # Delete friends
    print 'Delete friends:'
    resp, content = h.request('http://api.fanfou.com/friends/ids.json', headers=headers)
    friends = json.loads(content)
    for friend in friends:
        query_str = urlencode({ 'id': friend.encode('utf8') })
        print friend.encode('utf8'),
        try:
            resp, content = h.request('http://api.fanfou.com/friendships/destroy.json?' + query_str, 'POST', headers=headers)
        except timeout:
            print 'timeout'
        else:
            print resp.status
    
    # Delete followers
    print 'Delete followers:'
    resp, content = h.request('http://api.fanfou.com/followers/ids.json', headers=headers)
    followers = json.loads(content)
    for follower in followers:
        query_str = urlencode({ 'id': follower.encode('utf8') })
        print follower.encode('utf8'),
        try:
            resp, content = h.request('http://api.fanfou.com/blocks/create.xml?' + query_str, 'POST', headers=headers)
        except timeout:
            print 'timeout',
        else:
            print resp.status,
        try:
            resp, content = h.request('http://api.fanfou.com/blocks/destroy.xml?' + query_str, 'POST', headers=headers)
        except timeout:
            print 'timeout'
        else:
            print resp.status

这段 Python 脚本其实颇具我写饭否应用的风格。这段脚本还是比较有泛用性的，进入的时候会提示输入饭否id和密码，然后他也会自动备份一个 JSON 格式的饭否好友列表和关注者列表。只是处于周全我才做的这一步，其实是完全没有必要的，因为我已经备份过了，而且我也不准备让程序帮我全部还原……

正好清理一下两个列表……

原理就不多做介绍了。第二段脚本在运行开始清理好友的时候，可能会连续出现 timeout 的情况不必在意，因为饭否似乎会在这个 API 执行的时候处理 timeline 的缓存。不过据我观察，脚本运行结束后，似乎会有部分好友留下，数量对我来说少于十个，可以快速手动清除……

出于观察输出的考虑，我没有用多线程。脚本删除关注者的原理是加入黑名单再取消黑名单，如果加入了黑名单在取消的时候没有成功，那就意味着误加了，这是找不到的，因为黑名单的列表似乎无法察看……

就这样了……我现在已经逐渐的开始恢复饭否活动了……不过还不准备完全回到过去……以上

突然觉得，第一段脚本可以用 make 实现？
