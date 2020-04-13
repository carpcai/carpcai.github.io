---
title: coredns 概念原理与使用
tags:
  - kubernetes/k8s
originContent: ''
categories:
  - kubernetes/k8s
toc: false
date: 2020-04-13 11:34:06
---

# WHY ？
> kuberntes 中的 pod 基于 service 域名解析后，再负载均衡分发到 service 后端的各个 pod 服务中，如果没有 DNS 解析，则无法查到各个服务对应的 service 服务

## 在 Kubernetes 中，服务发现有几种方式：
- 基于环境变量的方式
- 基于内部域名的方式
 

# WHAT ?
      从 K8S 1.11 开始，K8S 已经使用 CoreDNS，替换 KubeDNS 来充当其 DNS 解析


DNS 如何解析，依赖容器内 resolv 文件的配置

```sh
# cat /etc/resolv.conf 
nameserver 10.200.254.254
search default.svc.cluster.local. svc.cluster.local. cluster.local.
options ndots:5
```


     ndots:5：如果查询的域名包含的点 “.” 不到 5 个，那么进行 DNS 查找，将使用非完全限定名称（或者叫绝对域名），如果你查询的域名包含点数大于等于 5，那么 DNS 查询，默认会使用绝对域名进行查询。

 

Kubernetes 域名的全称，必须是 service-name.namespace.svc.cluster.local 这种模式，服务名

```sh
# nslookup kubernetes.default.svc.cluster.local
Server:        10.200.254.254
Address:    10.200.254.254:53

Name:    kubernetes.default.svc.cluster.local
Address: 10.200.0.1
```
 

## DNS策略，在Pod，Deployment RC等资源设置 dnsPolicy 
- None

> 用于想要自定义 DNS 配置的场景，而且需要和 dnsConfig 配合一起使用

- Default

> 让 kubelet 来决定使用何种 DNS 策略。而 kubelet 默认使用宿主机的 /etc/resolv.conf（使用宿主机的DNS策略）

> 但 kubelet 可以配置使用什么文件来进行 DNS 策略，使用 kubelet 的参数：–resolv-conf=/etc/resolv.conf 来决定 DNS 解析文件地址

- ClusterFirst

> 表示 POD 内的 DNS 使用集群中配置的 DNS 服务，使用 Kubernetes 中 kubedns 或 coredns 服务进行域名解析。如果解析不成功，才会使用宿主机的 DNS 进行解析

- ClusterFirstWithHostNet
> POD 是用 HOST 模式启动的（HOST模式），用 HOST 模式表示 POD 中的所有容器，都使用宿主机的 /etc/resolv.conf 进行 DNS 查询，但如果使用了 HOST 模式，还继续使用 Kubernetes 的 DNS 服务，那就将 dnsPolicy 设置为 ClusterFirstWithHostNet

 

## 配置文件使用 configmap 
- health：CoreDNS 健康检查为 http://$IP:8080/health，返回为 OK
- kubernetes：CoreDNS 将根据 Kubernetes 服务和 pod 的 IP 回复 DNS 查询
- prometheus：CoreDNS 度量 http://$IP:9153/metrics
- proxy：不在 Kubernetes 集群域内的查询都将转发到预定义的解析器（/etc/resolv.conf），可以配置多个upstream 域名服务器，也可以用于延迟查找 /etc/resolv.conf 中定义的域名服务器
- cache：启用缓存，30 秒 TTL
- loop：检测简单的转发循环，如果找到循环则停止 CoreDNS 进程
- reload：允许自动重新加载已更改的 Corefile
- loadbalance：DNS 负载均衡器，默认round_robin
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: namespace-test
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local  10.200.0.0/16 {
          pods insecure
          upstream 114.114.114.114
          fallthrough in-addr.arpa ip6.arpa
          namespaces namespace-test
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

 

# coredns 安装部署
下载：https://github.com/coredns/deployment/tree/master/kubernetes

     deploy.sh 用于生成用于 kube-dns 的集群上运行 CoreDNS 的 yaml 文件

     coredns.yaml.sed 文件作为模板，它创建一个 ConfigMap 和一个 CoreDNS  deployment 的yaml 文件 

 

    ./deploy.sh 172.18.0.0/24 cluster.local 生成 yaml 文件，在使用 kubectl apply 部署在 k8s 中

 

 
# 官方性能
计算表达式： MB required (default settings) = (Pods + Services) / 1000 + 54

- cache 需要 30 MB，大约缓存 10000 条记录
- 操作 buffer 需要 5 MB，用于处理查询，大约可以承受 30 K QPS 量

![UTOOLS1586748784214.png](http://yanxuan.nosdn.127.net/8732d7f0742b31bb0643ada645e680b4.png)
 
    容器内抓包
```sh
docker inspect --format "{{.State.Pid}}" container
# 进入 container namespace
nsenter -n -t ￥pid
# 对 53 端口进行抓包
# tcpdump -i eth0 -N udp dst port 53
```
