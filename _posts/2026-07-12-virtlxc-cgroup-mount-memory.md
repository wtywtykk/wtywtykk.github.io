---
layout: post_layout
title: virtlxc报告Unable to find 'memory' cgroups controller mount
time: 2026年07月12日 星期日
location: 杭州
pulished: true
excerpt_separator: "首先"
---

在使用libvirt管理lxc时，出现如下错误：
```
Error starting domain: internal error: Unable to find 'memory' cgroups controller mount

Traceback (most recent call last):
  File "/usr/share/virt-manager/virtManager/asyncjob.py", line 67, in cb_wrapper
    callback(asyncjob, *args, **kwargs)
    ~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/share/virt-manager/virtManager/asyncjob.py", line 101, in tmpcb
    callback(*args, **kwargs)
    ~~~~~~~~^^^^^^^^^^^^^^^^^
  File "/usr/share/virt-manager/virtManager/object/libvirtobject.py", line 57, in newfn
    ret = fn(self, *args, **kwargs)
  File "/usr/share/virt-manager/virtManager/object/domain.py", line 1459, in startup
    self._backend.create()
    ~~~~~~~~~~~~~~~~~~~~^^
  File "/usr/lib64/python3.13/site-packages/libvirt.py", line 1416, in create
    raise libvirtError('virDomainCreate() failed')
libvirt.libvirtError: internal error: Unable to find 'memory' cgroups controller mount
```

原因为virtlxcd.service中没有设置CGroup Delegate

尝试访问支持的controller，结果如下：
```
# cat /sys/fs/cgroup/system.slice/virtlxcd.service/cgroup.controllers
cpu pids
```
列表中没有memory。解决方案：
```
# systemctl edit virtlxcd.service
```
在弹出的文件中增加如下：
```
[Service]
Delegate=pids memory cpu
```
无需重启服务即可正常使用。