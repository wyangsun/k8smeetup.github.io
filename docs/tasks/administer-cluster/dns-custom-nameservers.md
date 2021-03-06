---
approvers:
- bowei
- zihongz
title: 在 Kubernetes 中配置私有 DNS 和上游 nameserver
---

{% capture overview %}

本页展示了如何添加自定义私有 DNS 域（存根域）和上游 nameserver。

{% endcapture %}

{% capture prerequisites %}
* {% include task-tutorial-prereqs.md %}



* Kubernetes 1.6 及其以上版本。
* 集群必须使用 `kube-dns` 插件进行配置。

{% endcapture %}

{% capture steps %}



## 配置存根域和上游 DNS 服务器

通过为 kube-dns （`kube-system:kube-dns`）提供一个 ConfigMap，集群管理员能够指定自定义存根域和上游 nameserver。

例如，下面的 ConfigMap 建立了一个 DNS 配置，它具有一个单独的存根域和两个上游 nameserver：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-dns
  namespace: kube-system
data:
  stubDomains: |
    {“acme.local”: [“1.2.3.4”]}
  upstreamNameservers: |
    [“8.8.8.8”, “8.8.4.4”]
```



按如上说明，具有 “.acme.local” 后缀的 DNS 请求被转发到 DNS 1.2.3.4。Google 公共 DNS 为上游查询提供服务。



| 域名 | 响应查询的服务器 |
| ----------- | -------------------------- |
| kubernetes.default.svc.cluster.local| kube-dns |
| foo.acme.local| 自定义 DNS (1.2.3.4) |
| widget.com    | 上游 DNS (8.8.8.8, 8.8.4.4 中之一) |


查看 [ConfigMap 选项](#configmap-options) 获取更多关于配置选项格式的详细信息。

{% endcapture %}

{% capture discussion %}



## 理解 Kubernetes 中名字解析

可以为每个 Pod 设置 DNS 策略。
当前 Kubernetes 支持两种 Pod 特定的 DNS 策略：“Default” 和 “ClusterFirst”。
可以通过 `dnsPolicy` 标志来指定这些策略。
*注意：“Default” 不是默认的 DNS 策略。如果没有显式地指定 `dnsPolicy`，将会使用 “ClusterFirst”。*



### "Default" DNS 策略

如果 `dnsPolicy` 被设置为 “Default”，则名字解析配置会继承自 Pod 运行所在的节点。
自定义上游 nameserver 和 存根域不能够与这个策略一起使用。



### "ClusterFirst" DNS 策略

如果 `dnsPolicy` 被设置为 "ClusterFirst"，名字解析的处理有所不同，*依赖于是否对存根域和上游 DNS 服务器进行了配置*。

**未进行自定义配置**：没有匹配上配置的集群域名后缀的任何请求，例如 “www.kubernetes.io”，将会被转发到继承自节点的上游 nameserver。

**进行自定义配置**：如果配置了存根域和上游 DNS 服务器（和在 [前面例子](#configuring-stub-domain-and-upstream-dns-servers) 配置的一样），DNS 查询将根据下面的流程进行路由：



1. 查询首先被发送到 kube-dns 中的 DNS 缓存层。

1. 从缓存层，检查请求的后缀，并转发到合适的 DNS 上，基于如下的示例：
 
   * *具有集群后缀的名字*（例如 ".cluster.local"）：请求被发送到 kube-dns。

   * *具有存根域后缀的名字*（例如 ".acme.local"）：请求被发送到配置的自定义 DNS 解析器（例如：监听在 1.2.3.4）。

   * *不具有能匹配上后缀的名字*（例如 "widget.com"）：请求被转发到上游 DNS（例如：Google 公共 DNS 服务器，8.8.8.8 和 8.8.4.4）。

![DNS 查询流程](/docs/tasks/administer-cluster/dns-custom-nameservers/dns.png)



## ConfigMap 选项

kube-dns `kube-system:kube-dns` ConfigMap 的选项如下所示：

| 字段 | 格式 | 描述 |
| ----- | ------ | ----------- |
| `stubDomains`（可选）| 使用 DNS 后缀 key 的 JSON map（例如 “acme.local”），以及 DNS IP 的 JSON 数组作为 value。 | 目标 nameserver 可能是一个 Kubernetes Service。例如，可以运行自己的 dnsmasq 副本，将 DNS 名字暴露到 ClusterDNS namespace 中。|
| `upstreamNameservers`（可选）| DNS IP 的 JSON 数组。 | 注意：如果指定，则指定的值会替换掉被默认从节点的 `/etc/resolv.conf` 中获取到的 nameserver。限制：最多可以指定三个上游 nameserver。|



## 附加示例

### 示例：存根域

在这个例子中，用户有一个 Consul DNS 服务发现系统，他们希望能够与 kube-dns 集成起来。
Consul 域名服务器地址为 10.150.0.1，所有的 Consul 名字具有后缀 “.consul.local”。
要配置 Kubernetes，集群管理员只需要简单地创建一个 ConfigMap 对象，如下所示：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-dns
    namespace: kube-system
    data:
      stubDomains: |
          {“consul.local”: [“10.150.0.1”]}
```


注意，集群管理员不希望覆盖节点的上游 nameserver，所以他们不会指定可选的 `upstreamNameservers` 字段。



### 示例：上游 nameserver

在这个示例中，集群管理员不希望显式地强制所有非集群 DNS 查询进入到他们自己的 nameserver 172.16.0.1。
而且这很容易实现：他们只需要创建一个 ConfigMap，`upstreamNameservers` 字段指定期望的 nameserver 即可。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-dns
    namespace: kube-system
    data:
      upstreamNameservers: |
          [“172.16.0.1”]
```

{% endcapture %}

{% include templates/task.md %}
