---
title: "在 kubernetes 中找出使用 jdk9 及以上版本的应用"
description: "在 kubernetes 中找出使用 jdk9 及以上版本的应用"
summary: "近日, Spring Cloud (SPEL) 中发现 RCE 0-day 漏洞, 为了排查 kubernetes 中所有存在安全威胁的应用. 特地开发了一个小工具来寻找。该工具基于 golang&client-go 开发, 程序会找出当前集群中所有 Running 的 pods, 然后逐个进入容器，执行 `java -version` 命令，将命令输出打印到文件中，使用编辑器进行查找检索即可。"
date: "2022-03-30"
menu: "main"
tags:
- "golang"
- "kubernetes"
categories:
- "technology"
---

## 一个漏洞

近日，在 Spring Cloud (SPEL) 中发现 RCE 0-day 漏洞，发布在**Ots安全**中，公众号文章如下:

---
![spring-cloud-sce-0day](/posts/2203/spring-cloud-sce-0day.png)

为了排查 kubernetes 中所有存在安全威胁的应用，特地开发了一个小工具来寻找。该工具基于 golang&client-go 开发, 程序会找出当前集群中所有 Running 的 pods, 然后逐个进入容器，执行 `java -version` 命令，将命令输出打印到文件中，使用编辑器进行查找检索即可。

## 源代码

代码量不大，这里直接贴出`main.go`的代码：

```go
package main

import (
	"bytes"
	"context"
	"flag"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/tools/remotecommand"
	"strings"
)

func main() {
	kubeconfig := flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err.Error())
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	pods, err := clientset.CoreV1().Pods("").List(context.TODO(), metav1.ListOptions{
		FieldSelector: "status.phase=Running",
	})
	if err != nil {
		panic(err.Error())
	}
	fmt.Printf("There are %d  running pods in the cluster\n", len(pods.Items))

	for _, item := range pods.Items {
		stdout, stderr, err := execInPod(config, item.Namespace, item.Name, item.Spec.Containers[0].Name)
		if err != nil {
			continue
		}
		fmt.Printf("--> %v/%v/%v\n", item.Namespace, item.Name, item.Spec.Containers[0].Name)
		fmt.Println(stderr)
		fmt.Println(stdout)
	}
}

func execInPod(config *rest.Config, namespace, podName, containerName string) (string, string, error) {
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}
	req := clientset.CoreV1().RESTClient().Post().
		Resource("pods").Name(podName).Namespace(namespace).
		SubResource("exec").
		Param("container", containerName)
	req.VersionedParams(
		&corev1.PodExecOptions{
			Command: []string{"java", "-version"},
			Stdin:   false,
			Stdout:  true,
			Stderr:  true,
			TTY:     false,
		},
		scheme.ParameterCodec,
	)

	var stdout, stderr bytes.Buffer
	exec, err := remotecommand.NewSPDYExecutor(config, "POST", req.URL())
	if err != nil {
		return "", "", err
	}
	err = exec.Stream(remotecommand.StreamOptions{
		Stdin:  nil,
		Stdout: &stdout,
		Stderr: &stderr,
	})
	if err != nil {
		return "", "", err
	}

	return strings.TrimSpace(stdout.String()), strings.TrimSpace(stderr.String()), err
}
```

## 使用方式

```shell
mkdir execjava-versioninpod
cd execjava-versioninpod
vi main.go
# 将上述代码粘贴进去
:wq
go mod init example.com/execjava-versioninpod
go mod tidy
go build .
kubectl config use-context <context-name> \
&& ./execjava-versioninpod \
  --kubeconfig=$HOME/.kube/config > java-version-outputs.txt
```

打印的文件`java-version-outputs.txt`的内容如下：

```text
There are 11  running pods in the cluster
--> default/tomcat-7cf47cf6b4-7bbqr/tomcat
openjdk version "1.8.0_292"
OpenJDK Runtime Environment (build 1.8.0_292-b10)
OpenJDK 64-Bit Server VM (build 25.292-b10, mixed mode)

--> default/tomcat-7cf47cf6b4-jcp7k/tomcat
openjdk version "1.8.0_292"
OpenJDK Runtime Environment (build 1.8.0_292-b10)
OpenJDK 64-Bit Server VM (build 25.292-b10, mixed mode)
...
```

使用常规的文本编辑器打开该文件，检索 jdk9 及以上版本的 pod 即可，至此完成～