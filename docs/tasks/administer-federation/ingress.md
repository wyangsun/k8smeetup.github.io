---
title: Federated Ingress
---

{% capture overview %}
<!--
This page explains how to use Kubernetes Federated Ingress to deploy
a common HTTP(S) virtual IP load balancer across a federated service running in
multiple Kubernetes clusters.  As of v1.4, clusters hosted in Google
Cloud (both GKE and GCE, or both) are supported. This makes it
easy to deploy a service that reliably serves HTTP(S) traffic
originating from web clients around the globe on a single, static IP
address.   Low network latency, high fault tolerance and easy administration are
ensured through intelligent request routing and automatic replica
relocation (using [Federated ReplicaSets](/docs/tasks/administer-federation/replicaset/).
Clients are automatically routed, via the shortest network path, to
the cluster closest to them with available capacity (despite the fact
that all clients use exactly the same static IP address).  The load balancer
automatically checks the health of the pods comprising the service,
and avoids sending requests to unresponsive or slow pods (or entire
unresponsive clusters).
-->
这篇阐述了如何使用 Kubernetes Federated Ingress，通过在多 Kubernetes 集群中运行的 federated 服务来部署不同 HTTP(S) 虚拟 IP 负载均衡。作为 v1.4，托管在 Google 云（GKE 或 GCE，或两者）都支持。这使它易于部署一个稳定的 HTTP(S) 服务，在一个单独的静态 IP 地址上服务来自全球的 web 客户。通过智能请求路由和自动复制迁移（使用 [Federated ReplicaSets](/docs/tasks/administer-federation/replicaset/)）确保了低网络延迟，高容错和易于管理。通过短网络路径，客户端被自动通过最短网络路径路由到集群最近可用的容器（尽管事实是所有客户端使用相同的静态 IP 地址）。负载均衡器自动检测由 pod 构成的服务，并避免发送请求到无响应或缓慢 pod 上（或整体无响应的集群）。

<!--
Federated Ingress is released as an alpha feature, and supports Google Cloud Platform (GKE,
GCE and hybrid scenarios involving both) in Kubernetes v1.4.  Work is under way to support other cloud
providers such as AWS, and other hybrid cloud scenarios (e.g. services
spanning private on-premises as well as public cloud Kubernetes
clusters).
-->
Federated Ingress 作为一个 alpha 版发行，并且在 Kubernetes v1.4版本支持 Google 云平台（GKE，GCE 和两者混杂方案）。工作正在开展来支持其他云供应商，如 AWS，和其他混合云方案（例如服务跨私有内部部署就像 Kubernetes 公有云集群一样）。

<!--
You create Federated Ingresses in much that same way as traditional
[Kubernetes Ingresses](/docs/concepts/services-networking/ingress/): by making an API
call which specifies the desired properties of your logical ingress point. In the
case of Federated Ingress, this API call is directed to the
Federation API endpoint, rather than a Kubernetes cluster API
endpoint. The API for Federated Ingress is 100% compatible with the
API for traditional Kubernetes Services.
-->
创建 Federated Ingresses 大多数方式像传统 [Kubernetes Ingress](/docs/concepts/services-networking/ingress/): 通过一个 API 调用特殊的逻辑入口点所需的属性。以 Federated Ingress 为例，这个 API 调用被定向到 Federation API 后端，而不是 Kubernetes 集群 API 后端。Federated Ingress 的 API 100% 兼容传统的 Kubernetes 服务的 API。

<!--
Once created, the Federated Ingress automatically:

*  Creates matching Kubernetes Ingress objects in every cluster underlying your Cluster Federation
*  Ensures that all of these in-cluster ingress objects share the same
   logical global L7 (that is, HTTP(S)) load balancer and IP address
*  Monitors the health and capacity of the service shards (that is, your pods) behind this ingress in each cluster
*  Ensures that all client connections are routed to an appropriate healthy backend service endpoint at all times, even in the event of pod, cluster, availability zone or regional outages
-->
一旦创建了，Federated Ingress 自动的：

* 在每个你的 Cluster Federation 下的集群创建匹配 Kubernetes Ingress 对象
* 确保所有这些内部集群入口对象共享同样的全局7层（那是 HTTP(S)）逻辑负载均衡和 IP 地址
* 监控每个集群这个入口后面的共享服务的健康和容量(那是你的 pods)
* 确保所有客户端链接始终被路由到一个适当的健康后端服务节点，即使在 pod，集群，可用区或域不可用的情况下。

<!--
Note that in the case of Google Cloud, the logical L7 load balancer is
not a single physical device (which would present both a single point
of failure, and a single global network routing choke point), but
rather a
[truly global, highly available load balancing managed service](https://cloud.google.com/load-balancing/),
globally reachable via a single, static IP address.
-->
请注意，以 Google Cloud 为例，7层逻辑负载均衡不是以一个单独的物理设备(可能出现两个单点故障，和一个全球网络路由阻塞点)，而是一个[真正全球，高可用负载均衡管理服务](https://cloud.google.com/load-balancing/)，通过单独静态 IP 全球可达。

<!--
Clients inside your federated Kubernetes clusters (Pods) will be
automatically  routed to the cluster-local shard of the Federated Service
backing the Ingress in their cluster if it exists and is healthy, or the closest healthy shard in a
different cluster if it does not.  Note that this involves a network
trip to the HTTP(s) load balancer, which resides outside your local
Kubernetes cluster but inside the same GCP region.
-->
客户端在联合 Kubernetes 集群(Pods)内部将被自动路由到集群本地共 Federated Service 分片，如果它存在并是健康的，则回到他们的集群 Ingress，或者如果它不是，则回到最近的不同集群的健康分片。
{% endcapture %}

{% capture prerequisites %}
<!--
This document assumes that you have a running Kubernetes Cluster
Federation installation. If not, then see the
[federation admin guide](/docs/admin/federation/) to learn how to
bring up a cluster federation (or have your cluster administrator do
this for you). Other tutorials, for example
[this one](https://github.com/kelseyhightower/kubernetes-cluster-federation)
by Kelsey Hightower, are also available to help you.

You must also have a basic
[working knowledge of Kubernetes](/docs/setup/) in
general, and [Ingress](/docs/concepts/services-networking/ingress/) in particular.
-->
这个文档假定了你已经运行了 Kuernetes Cluster Federation。如果不是的话就看看[联合管理向导](/docs/admin/federation/)学习如何建立一个集群联合(或由你的集群管理员给你做这个)。其他指导，例如 Kelsey Hightower 的[这个](https://github.com/kelseyhightower/kubernetes-cluster-federation)，也可以帮助你。

一般来说你必须也有基本的[ Kubernetes 的工作知识](/docs/setup/)，尤其是[Ingress](/docs/concepts/services-networking/ingress/)。
{% endcapture %}

{% capture steps %}
<!--
## Creating a federated ingress

You can create a federated ingress in any of the usual ways, for example, using kubectl:

``` shell
kubectl --context=federation-cluster create -f myingress.yaml
```
For example ingress YAML configurations, see the [Ingress User Guide](/docs/concepts/services-networking/ingress/).
The '--context=federation-cluster' flag tells kubectl to submit the
request to the Federation API endpoint, with the appropriate
credentials. If you have not yet configured such a context, see the
[federation admin guide](/docs/admin/federation/) or one of the
[administration tutorials](https://github.com/kelseyhightower/kubernetes-cluster-federation)
to find out how to do so.
-->
## 创建一个联合的 ingress

你可以用任何普通的方式创建一个联合的 ingress，例如，使用 kubectl:
``` shell
kubectl --context=federation-cluster create -f myingress.yaml
```
例如 ingress YAML 配置，请看 [Ingress 用户指南](/docs/concepts/services-networking/ingress/)。
'--context=federation-cluster' 标志告诉 kubectl 来提交一个请求到 Federation API 具有相应凭据的后端。如果你还没有配置相应的内容，请看[联合管理指南](/docs/admin/federation/)或[管理员辅导](https://github.com/kelseyhightower/kubernetes-cluster-federation)之一，来找出如何去做。

<!--
The Federated Ingress automatically creates
and maintains matching Kubernetes ingresses in all of the clusters
underlying your federation.  These cluster-specific ingresses (and
their associated ingress controllers) configure and manage the load
balancing and health checking infrastructure that ensures that traffic
is load balanced to each cluster appropriately.

You can verify this by checking in each of the underlying clusters. For example:
-->
Federated Ingress 自动创建和维护所有联合集群下匹配的 Kubernetes ingresses。这些特定的集群 ingresses （和他们相关的 ingress 控制器）配置管理负载均衡和健康检查架构，确保流量负载均衡到每个合适的集群。

你可以通过检查每个集群证实这个。例如：
``` shell
kubectl --context=gce-asia-east1a get ingress myingress
NAME        HOSTS     ADDRESS           PORTS     AGE
myingress   *         130.211.5.194     80, 443   1m
```

<!--
The above assumes that you have a context named 'gce-asia-east1a'
configured in your client for your cluster in that zone. The name and
namespace of the underlying ingress automatically matches those of
the Federated Ingress that you created above (and if you happen to
have had ingresses of the same name and namespace already existing in
any of those clusters, they will be automatically adopted by the
Federation and updated to conform with the specification of your
Federated Ingress. Either way, the end result will be the same).

The status of your Federated Ingress automatically reflects the
real-time status of the underlying Kubernetes ingresses. For example:
-->
上面假设在你的客户端有一个环境名叫 ‘gce-asia-east1a‘ 配置那个区域的集群。ingress 的名字和命名空间自动匹配这些你上面创建的 Federated Ingeress（如果你正好有相同名字的 ingress 和命名空间已存在其他的集群，他们将被 Federation 自动适配然后更新符合你的 Federated Ingress 的规范。无论哪种方式，最终结果是相同的）。

你的 Federated Ingress 状态自动反应下面 Kubernetes ingresses 的实时状态。例如：
``` shell
kubectl --context=federation-cluster describe ingress myingress

Name:           myingress
Namespace:      default
Address:        130.211.5.194
TLS:
  tls-secret terminates
Rules:
  Host  Path    Backends
  ----  ----    --------
  * *   echoheaders-https:80 (10.152.1.3:8080,10.152.2.4:8080)
Annotations:
  https-target-proxy:       k8s-tps-default-myingress--ff1107f83ed600c0
  target-proxy:         k8s-tp-default-myingress--ff1107f83ed600c0
  url-map:          k8s-um-default-myingress--ff1107f83ed600c0
  backends:         {"k8s-be-30301--ff1107f83ed600c0":"Unknown"}
  forwarding-rule:      k8s-fw-default-myingress--ff1107f83ed600c0
  https-forwarding-rule:    k8s-fws-default-myingress--ff1107f83ed600c0
Events:
  FirstSeen LastSeen    Count   From                SubobjectPath   Type        Reason  Message
  --------- --------    -----   ----                -------------   --------    ------  -------
  3m        3m      1   {loadbalancer-controller }          Normal      ADD default/myingress
  2m        2m      1   {loadbalancer-controller }          Normal      CREATE  ip: 130.211.5.194
```

<!--
Note that:

*  The address of your Federated Ingress
corresponds with the address of all of the
underlying Kubernetes ingresses (once these have been allocated - this
may take up to a few minutes).
*  You have not yet provisioned any backend Pods to receive
the network traffic directed to this ingress (that is, 'Service
Endpoints' behind the service backing the Ingress), so the Federated Ingress does not yet consider these to
be healthy shards and will not direct traffic to any of these clusters.
*  The federation control system
automatically reconfigures the load balancer controllers in all of the
clusters in your federation to make them consistent, and allows
them to share global load balancers.  But this reconfiguration can
only complete successfully if there are no pre-existing Ingresses in
those clusters (this is a safety feature to prevent accidental
breakage of existing ingresses).  So, to ensure that your federated
ingresses function correctly, either start with new, empty clusters, or make
sure that you delete (and recreate if necessary) all pre-existing
Ingresses in the clusters comprising your federation.
-->
注意：

* 你的 Federated Ingress 地址与所有下面 Kubernetes ingresses 一致（一旦这些被分配 - 这也许要花几分钟）。
* 你还没提供任何后端 Pods 来直接接受网络流量到这个入口（就是在服务后的入口后面的 ‘Service Endpoints’ ），所以 Federated Ingress 还不认为这些是健康的分片，也不会将流量导入这些集群。
* 联合控制系统自动重构所有联合下集群的负载均衡控制器使他们一致，并允许他们共享全局负载均衡器。但是如果这些集群中没有已经预制的 Ingresses 只能成功完成这个重构（这是一个安全特性来避免损坏存在的 ingresses）。所以，确保你的联合入口当前的功能，启动新的，空的集群，或确保删除所有组成你的 federation 集群的预制 Ingresses。

<!--
## Adding backend services and pods

To render the underlying ingress shards healthy, you need to add
backend Pods behind the service upon which the Ingress is based.  There are several ways to achieve this, but
the easiest is to create a Federated Service and
Federated ReplicaSet.  To
create appropriately labelled pods and services in the 13 underlying clusters of
your federation:
-->
## 增加后端服务和 pods

你需要在以 Ingress 为基础的服务后增加后端 Pods，使下层入口分片健康。有很多方法来实现这个，但是最简单是创建一个 Federated Service 和 Federated ReplicaSet。在联盟13个下层集群中创建合适标签的 pods 和服务:
``` shell
kubectl --context=federation-cluster create -f services/nginx.yaml
```

``` shell
kubectl --context=federation-cluster create -f myreplicaset.yaml
```

<!--
Note that in order for your federated ingress to work correctly on
Google Cloud, the node ports of all of the underlying cluster-local
services need to be identical.  If you're using a federated service
this is easy to do.  Simply pick a node port that is not already
being used in any of your clusters, and add that to the spec of your
federated service.  If you do not specify a node port for your
federated service, each cluster will choose its own node port for
its cluster-local shard of the service, and these will probably end
up being different, which is not what you want.

You can verify this by checking in each of the underlying clusters. For example:
-->
请注意，为使您的联合入口在 Google Cloud 上正常工作，所有下层本地集群服务的节点端口必须相同。如果你正使用一个联合服务，则很容易。随意在你的集群挑一个还没使用的节点端口，增加到你的联合服务空间。如果你没有给你的联合服务指定节点端口，每个集群将选择它自己的节点端口给他的本地集群服务分片，并切这些很可能最终不一样，这不是你想要的。

你可以通过检查每个下层集群来验证这个。例如：
``` shell
kubectl --context=gce-asia-east1a get services nginx
NAME      CLUSTER-IP     EXTERNAL-IP      PORT(S)   AGE
nginx     10.63.250.98   104.199.136.89   80/TCP    9m
```

<!--
## Hybrid cloud capabilities

Federations of Kubernetes Clusters can include clusters running in
different cloud providers (for example, Google Cloud, AWS), and on-premises
(for example, on OpenStack).  However, in Kubernetes v1.4, Federated Ingress is only
supported across Google Cloud clusters.
-->
## 混合云能力

Kubernets Clusters 的 Federations 可以包含在不同云提供商运行的集群（例如，Google Cloud,AWS）,和本地（像 OpenStack）。无论如何，在 Kubernetes 1.4，Federated Ingress 只支持 Google Cloud 集群。

<!--
## Discovering a federated ingress

Ingress objects (in both plain Kubernets clusters, and in federations
of clusters) expose one or more IP addresses (via
the Status.Loadbalancer.Ingress field) that remains static for the lifetime
of the Ingress object (in future, automatically managed DNS names
might also be added).  All clients (whether internal to your cluster,
or on the external network or internet) should connect to one of these IP
or DNS addresses.  All client requests are automatically
routed, via the shortest network path, to a healthy pod in the
closest cluster to the origin of the request. So for example, HTTP(S)
requests from internet
users in Europe will be routed directly to the closest cluster in
Europe that has available capacity.  If there are no such clusters in
Europe, the request will be routed to the next closest cluster
(typically in the U.S.).
-->
## 发现联合 ingress

入口对象（在简单的 Kubernets 集群，和联合的集群里）暴露一个或多个 IP 地址（通过 Status.Loadbalancer.Ingress 段）在 Ingress 对象生命周期保持静态（将来，自动管理 DNS 名可能本添加）。所有客户端（无论链接你集群内部或外部网络或互联网）应该链接链接这些 IP 或 DNS 地址之一。所有客户端请求被自动的通过最短网络路径转发，到一个离原始请求最近的集群的健康 pod。例如，从欧洲互联网用户来的 HTTP(S) 请求将被转发到欧洲最近的可用集群。如果在欧洲没有这样的集群，请求将被路由到下一个最近的集群（一般在美国）。

<!--
## Handling failures of backend pods and whole clusters

Ingresses are backed by Services, which are typically (but not always)
backed by one or more ReplicaSets.  For Federated Ingresses, it is
common practise to use the federated variants of Services and
ReplicaSets for this purpose.

In particular, Federated ReplicaSets ensure that the desired number of
pods are kept running in each cluster, even in the event of node
failures.  In the event of entire cluster or availability zone
failures, Federated ReplicaSets automatically place additional
replicas in the other available clusters in the federation to accommodate the
traffic which was previously being served by the now unavailable
cluster. While the Federated ReplicaSet ensures that sufficient replicas are
kept running, the Federated Ingress ensures that user traffic is
automatically redirected away from the failed cluster to other
available clusters.
-->
## 处理后端 pod 和整个集群错误

Ingress 由 Services 支持，他一般（但不总是）支持一个或多个 ReplicaSets。对 Federated Ingresses 来说,他一般使用服务和 ReplicaSets 的联合变体来实现此目的。

特别是，联合 ReplicaSets 确保了每个集群上运行了预期数量的 pods，甚至在节点抱错的情况。甚至整个集群或可用区出现故障的情况，联合 ReplicaSets 自动在联合内其他可用集群放置额外的复制用来提供现在不可用的集群以前服务的流量。同时联合 ReplicaSets 确保保持运行充足的复制，Federated Ingress 确保用户流量从失败的集群自动的重定向到其他可用集群。

<!--
## Troubleshooting

#### I cannot connect to my cluster federation API.

Check that your:

1. Client (typically `kubectl`) is correctly configured (including API endpoints and login credentials).
2. Cluster Federation API server is running and network-reachable.

See the [federation admin guide](/docs/admin/federation/) to learn
how to bring up a cluster federation correctly (or have your cluster administrator do this for you), and how to correctly configure your client.
-->
## 排错

#### 我不能连接到我的集群联合 API。

检查你的：

1. 客户端（通常是`kubectl`）当前配置（包括 API 终端和登陆证书）。
2. Cluster Federation API 服务器运行在可达网络。

查看 [联合管理手册](/docs/admin/federation/) 来学习如何正确启动一个集群联合（或有集群管理员帮你做这些），和如何正确的配置你的客户端。

<!--
#### I can create a Federated Ingress/service/replicaset successfully against the cluster federation API, but no matching ingresses/services/replicasets are created in my underlying clusters.

Check that:

1. Your clusters are correctly registered in the Cluster Federation API. (`kubectl describe clusters`)
2. Your clusters are all 'Active'.  This means that the cluster
   Federation system was able to connect and authenticate against the
   clusters' endpoints.  If not, consult the event logs of the federation-controller-manager pod to ascertain what the failure might be. (`kubectl --namespace=federation logs $(kubectl get pods --namespace=federation -l module=federation-controller-manager -o name`)
3. That the login credentials provided to the Cluster Federation API
   for the clusters have the correct authorization and quota to create
   ingresses/services/replicasets in the relevant namespace in the
   clusters.  Again you should see associated error messages providing
   more detail in the above event log file if this is not the case.
4. Whether any other error is preventing the service creation
   operation from succeeding (look for `ingress-controller`,
   `service-controller` or `replicaset-controller`,
   errors in the output of `kubectl logs federation-controller-manager --namespace federation`).
-->
#### 我可以靠联合 API 成功创建一个 Federated Ingress/service/replicaset，但是不能匹配我下层集群创建的 ingresses/services/replicasets。

检查：

1. 你的集群被正确注册到 Cluster Federation API。（`kubectl describe clusters`）
2. 你的集群所有是 ‘Active’。这意味着集群联合系统可以连接认证集群终端。如果不是，查看 federation-controller-manager pod 日志来查明可能是什么错误（`kubectl --namespace=federation logs $(kubectl get pods --namespace=federation -l module=federation-controller-manager -o name`）。
3. 提供给集群联合 API 的登录凭证具有正确的授权和配额，以在集群中的相关名称空间中创建 ingresses/services/replicasets。如果不是这种情况，你应该在上面的事件日志文件再一次查看有关错误信息更多细节。
4. 是否有其他错误阻止了服务成功创建（在 `kubectl logs federation-controller-manager --namespace federation` 输出看 `ingress-controller`，`service-controller` 或 `replicaset-controller`，错误）。

<!--
#### I can create a federated ingress successfully, but request load is not correctly distributed across the underlying clusters.

Check that:

1. The services underlying your federated ingress in each cluster have
    identical node ports.  See [above](#creating_a_federated_ingress) for further explanation.
2. The load balancer controllers in each of your clusters are of the
   correct type ("GLBC") and have been correctly reconfigured by the
   federation control plane to share a global GCE load balancer (this
   should happen automatically).  If they are of the correct type, and
   have been correctly reconfigured, the UID data item in the GLBC
   configmap in each cluster will be identical across all clusters.
   See
   [the GLBC docs](https://github.com/kubernetes/ingress/blob/7dcb4ae17d5def23d3e9c878f3146ac6df61b09d/controllers/gce/README.md)
   for further details.
   If this is not the case, check the logs of your federation
   controller manager to determine why this automated reconfiguration
   might be failing.
3. No ingresses have been manually created in any of your clusters before the above
    reconfiguration of the load balancer controller completed
    successfully.  Ingresses created before the reconfiguration of
    your GLBC will interfere with the behavior of your federated
    ingresses created after the reconfiguration (see
    [the GLBC docs](https://github.com/kubernetes/ingress/blob/7dcb4ae17d5def23d3e9c878f3146ac6df61b09d/controllers/gce/README.md)
    for further information). To remedy this,
    delete any ingresses created before the cluster joined the
    federation (and had its GLBC reconfigured), and recreate them if
    necessary.
-->
#### 我能成功创建一个联合入口，但是请求负载没有正确分布到下层集群。

检查：

1. 在每个集群联合入口下的服务有指定端口。更多解释查看[上面](#creating_a_federated_ingress)。
2. 在每个集群负载均衡控制器是正确的类型并通过联合控制面板正确的重置来共享一个全局 GCE 负载均衡器（这应该自动发生）。如果他们是正确的类型并有正确的重置，在每个集群 GLBC configmap 中 UID 数据项在所有集群都是相同的。更多细节查看[ GLBC 文档](https://github.com/kubernetes/ingress/blob/7dcb4ae17d5def23d3e9c878f3146ac6df61b09d/controllers/gce/README.md)。
如果不是这种情况，检查你联合控制管理器日志来确定为什么自动化重置失败。
3. 在上面负载均衡器成功重置前没有任何集群入口被手动创建。在 GLBC 重置前创建入口将干涉你重置后创建联合入口（更多信息查看[ GLBC 文档](https://github.com/kubernetes/ingress/blob/7dcb4ae17d5def23d3e9c878f3146ac6df61b09d/controllers/gce/README.md)）。为了弥补这点，在集群加入联合前删除任何创建的入口（并重新配置了 GLBC）,如果必要则重新创建它们。

{% endcapture %}

{% capture whatsnext %}
<!--
*  If you need assistance, use one of the [support channels](/docs/tasks/debug-application-cluster/troubleshooting/) to seek assistance.
 *  For details about use cases that motivated this work, see
 [Federation proposal](https://git.k8s.io/community/contributors/design-proposals/federation/federation.md).
-->
* 如果你需要帮助，使用[支持频道](/docs/tasks/debug-application-cluster/troubleshooting/)之一来寻找帮助。 * 更多关于促进这个工作的用例，请看[联合提议](https://git.k8s.io/community/contributors/design-proposals/federation/federation.md)。
{% endcapture %}
{% include templates/task.md %}
