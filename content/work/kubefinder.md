---
title: 'KubeFinder'
date: 2018-11-18T12:33:46+10:00
draft: false
weight: 2
heroHeading: 'KubeFinder'
heroSubHeading: '云原生容器文件浏览器'
heroBackground: '/work/kube-finder-bg.jpeg'
thumbnail: '/work/kube-finder-header.jpg'
images: [
	'/work/fileserver1.png', 
	'/work/fileserver2.png'
]
---

项目介绍

[llaoj/kube-finder](https://github.com/llaoj/kube-finder)是一个独立的容器文件服务器, 可以通过该项目查看容器内(namespace/pod/container)的文件目录结构/列表, 下载文件或者上传文件到指定目录中. 使用golang开发, 直接操作主机 /proc/<pid>/root 目录, 速度很快.

项目名取自MacOS Finder, 希望它像Finder一样好用. 目前该项目处在alpha阶段.

这是一个后端项目, 仅提供API, 前端对接之后可以是这样的, 如下
