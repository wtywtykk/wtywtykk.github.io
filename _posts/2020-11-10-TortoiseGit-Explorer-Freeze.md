---
layout: post_layout
title: 网络共享+TortoiseGit=资源管理器卡顿
time: 2020年11月10日 星期二
location: 成都
pulished: true
excerpt_separator: "11111"
---

最近折腾了下网络共享，顺手把共享pin到了资源管理器快速访问，之后右键就会经常卡顿，打开我的电脑一直空白。

VS调试了下Explorer，发现全都在TortoiseGit的模块里面等Lock。进一步查发现持有这个Lock的线程在查询网络共享的属性。但此时连不上共享，就一直卡住直到超时。

emmm不知道是技术问题还是怎么的，TortoiseGit用了这么一个全局锁。。。。。坑死

知道问题了解决就很好办了，把共享从快速访问去掉就行。但是直接右键去不掉，需要删除
```
C:\Users\用户名\AppData\Roaming\Microsoft\Windows\Recent\AutomaticDestinations
C:\Users\用户名\AppData\Roaming\Microsoft\Windows\Recent\CustomDestinations
```
里面的所有文件，清空掉整个快速访问栏。
