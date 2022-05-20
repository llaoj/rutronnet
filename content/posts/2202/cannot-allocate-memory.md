---
title: "解决 kubelet cannot allocate memory 错误"
description: "mkdir /sys/fs/cgroup/memory/kubepods/burstable/podxxx: cannot allocate memory"
date: "2022-02-27"
menu: "main"
tags:
- "kubelet"
- "kubernetes"
categories:
- "technology"
---

## 问题描述

查看 pod 相关 events 如下：

```sh
Events:
  Type     Reason                    Age                   From               Message
  ----     ------                    ----                  ----               -------
  Normal   Scheduled                 18m                   default-scheduler  Successfully assigned container-186002196200947712/itms-5f6d7798-wrpjj to 10.206.65.144
  Warning  FailedCreatePodContainer  3m31s (x71 over 18m)  kubelet            unable to ensure pod container exists: failed to create container for [kubepods burstable pod31f4c93c-c3a1-49ad-b091-0802c5f1d396] : mkdir /sys/fs/cgroup/memory/kubepods/burstable/pod31f4c93c-c3a1-49ad-b091-0802c5f1d396: cannot allocate memory
```

这是内核bug，建议升级内核