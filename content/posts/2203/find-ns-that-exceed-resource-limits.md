---
title: "在 kubernetes 中找出过度使用资源的 namespaces"
description: "在 k8s 集群中找出过度使用资源的 namespaces"
summary: "我们知道, 在 kubernetes 中, namespace 的资源限制在 ResourceQuota 中定义, 比如我们控制 default 名称空间使用 1核1G 的资源. 通常来讲, 由于 kubernetes 的资源控制机制, `.status.used` 中资源的值会小于 `.status.hard` 中相应资源的值. 但是也有特例. 当我们开始定义了一个较大的资源限制, 待应用部署完毕, 资源占用了很多之后, 这时调低资源限制. 此时就会出现 `.status.used` 中的值超过 `.status.hard` 中相应值的情况, 尤其是内存的限制."
date: "2022-03-28"
menu: "main"
tags:
- "golang"
- "kubernetes"
categories:
- "technology"
---

我们知道, 在 kubernetes 中, namespace 的资源限制在 ResourceQuota 中定义, 比如我们控制 default 名称空间使用 64核80G 的资源:

```yaml
$ kubectl get resourcequota not-best-effort -oyaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: not-best-effort
spec:
  hard:
    limits.cpu: "64"
    limits.memory: 80G
status:
  hard:
    limits.cpu: "64"
    limits.memory: 80G
  used:
    limits.cpu: 30500m
    limits.memory: 59G
```

通常来讲, 由于 kubernetes 的资源控制机制, `.status.used` 中资源的值会小于 `.status.hard` 中相应资源的值. 但也有特例.

## 特殊情况

当我们开始定义了一个较大的资源限制, 待应用部署完毕, 资源占用了很多之后, 这时调低资源限制. 此时就会出现 `.status.used` 中的值超过 `.status.hard` 中相应值的情况, 尤其是内存的限制. 比如下面个:

```yaml
...
spec:
  hard:
    limits.cpu: "1"
    limits.memory: 1G
status:
  hard:
    limits.cpu: "1"
    limits.memory: 1G
  used:
    limits.cpu: 15600m
    limits.memory: 26044M
```

## 找出这些名称空间

在集群管理的过程中, 往往我们需要找出这些过度使用资源名称空间, 这可能是因为用户会为资源付费. 所以, 我尝试使用 golang 开发一个小工具, 找出这些名称空间. 现在代码已经写好了, 这个工具有很少的代码, 直接将全部代码贴出来:

```golang
package main

import (
    "bufio"
    "fmt"
    "io"
    "os"
    "strconv"
    "strings"
)

func main() {
    fi, err := os.Open("./resourcequotas.txt")
    if err != nil {
        fmt.Printf("Error: %s\n", err)
        return
    }
    defer fi.Close()

    br := bufio.NewReader(fi)
    for {
        l, _, c := br.ReadLine()
        if c == io.EOF {
            break
        }
        fmt.Print(string(l))

        s := strings.Split(string(l), "limits.memory:")
        mem := strings.Trim(s[1], " ")
        s = strings.Split(mem, "/")

        mused := formatUnits(s[0])
        mhard := formatUnits(s[1])
        fmt.Printf("(%v/%v)", mused, mhard)

        musedf, _ := strconv.ParseFloat(mused, 64)
        mhardf, _ := strconv.ParseFloat(mhard, 64)
        if musedf > mhardf {
            fmt.Println("    <== this line")
        } else {
            fmt.Println("")
        }
    }
}

func formatUnits(s string) string {
    if strings.Index(s, "G") >= 0 {
        s = strings.Trim(s, "G")
        s = strings.Trim(s, "Gi")

        return s
    }

    if strings.Index(s, "M") >= 0 {
        s = strings.Trim(s, "M")
        s = strings.Trim(s, "Mi")
        i, _ := strconv.Atoi(s)
        s = fmt.Sprintf("%.2f", float64(i)/1024)

        return s
    }

    if s != "0" {
        // byte
        i, _ := strconv.Atoi(s)
        s = fmt.Sprintf("%.2f", float64(i)/1024/1024/1024)

        return s
    }

    return s
}
```

> 该代码仅实现了查找内存占用超过限制的逻辑

首先, 我们通过命令, 将集群中所有的 ResourceQuota 资源导出到 `resourcequotas.txt` 文件中:

```sh
kubectl get resourcequota -A > resourcequotas.txt
# 文件内容如下:
NAMESPACE       NAME              AGE    REQUEST   LIMIT
namespace-001   not-best-effort   129d             limits.cpu: 30500m/64, limits.memory: 59G/80G
namespace-002   not-best-effort   125d             limits.cpu: 3300m/10, limits.memory: 3564M/10G
namespace-003   not-best-effort   4d5h             limits.cpu: 1/4, limits.memory: 1G/8G

```

将生成的 txt 文件和代码放在一个目录中, 接下来执行代码, 会看到下面的输出:

```sh
$ go run main.go
...
namespace-102   not-best-effort   349d      limits.cpu: 10100m/13, limits.memory: 20218M/26G(19.74/26)
namespace-103   not-best-effort   349d      limits.cpu: 15600m/1, limits.memory: 26044M/1G(25.43/1)    <== this line
namespace-104   not-best-effort   349d      limits.cpu: 5200m/8, limits.memory: 5460M/16G(5.33/16)
...
```

可以看到 `<== this line`, 名称空间 namespace-103 定义的内存限制是 1G, 但是实际使用了 25.43G, 很明显超过了资源限制. 我们的目的也就达到了.