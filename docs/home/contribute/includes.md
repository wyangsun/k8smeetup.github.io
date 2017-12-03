---
approvers:
- chenopis
title: 自定义 Jekyll Include 片段
cn-approvers:
- chentao1596
---


{% capture overview %}

本页解释自定义 Jekyll Include 片段，它可用于 Kubernetes 文档模型。


想要了解更多 Jekyll 中关于 includes 的内容，请参考 [Jekyll 文档](https://jekyllrb.com/docs/includes/)。
{% endcapture %}

{% capture body %}

## 特性状态


在站点文档的模型页（.md 文件）中，您能够增加一个用于显示版本以及文档特征的状态的标签。


### 特性状态演示


下面是一个特性状态片段的演示。它用于表示这是 Kuberentes 1.6 版本的稳定特性。

{% assign for_k8s_version = "1.6" %}
{% include feature-state-stable.md %}


### 特性状态代码


下面是每个可用的特性状态的模板代码。


模型页上显示的 Kubernetes 版本是默认。能够通过设置 <code>for_k8s_version</code> 变量来修改它。

````liquid
{{ "{% assign for_k8s_version = " }} "1.6" %}
{{ "{% include feature-state-stable.md " }}%}
````


#### α 特性

````liquid
{{ "{% include feature-state-alpha.md " }}%}
````


#### β 特性

````liquid
{{ "{% include feature-state-beta.md " }}%}
````


#### 稳定特性

````liquid
{{ "{% include feature-state-stable.md " }}%}
````


#### 不建议使用的特性

````liquid
{{ "{% include feature-state-deprecated.md " }}%}
````


## 选项卡


在站点文档的模型页（.md 文件）中，您可以添加一个选项卡集来显示给定解决方案的多种风格。


### 选项卡演示


下面是一个选项卡片段的演示。在这里，它是用来显示对各种 Kubernetes 网络解决方案的安装命令。

{% capture default_tab %}

选择其中一个选项卡。
{% endcapture %}

{% capture calico %}
```shell
kubectl apply -f "http://docs.projectcalico.org/v2.4/getting-started/kubernetes/installation/hosted/kubeadm/calico.yaml"
```
{% endcapture %}

{% capture flannel %}
```shell
kubectl apply -f "https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml?raw=true"
```
{% endcapture %}

{% capture romana %}
```shell
kubectl apply -f "https://raw.githubusercontent.com/romana/romana/master/containerize/specs/romana-kubeadm.yml"
```
{% endcapture %}

{% capture weave_net %}
```shell
kubectl apply -f "https://git.io/weave-kube"
```
{% endcapture %}

{% assign tab_names = "Default,Calico,Flannel,Romana,Weave Net" | split: ',' | compact %}
{% assign tab_contents = site.emptyArray | push: default_tab | push: calico | push: flannel | push: romana | push: weave_net %}

{% include tabs.md %}


### 选项卡的 Liquid 模板代码示例


下面是选项卡的 [Liquid](https://shopify.github.io/liquid/) 模板代码，它将演示如何指定每个选项卡的内容。在最后，[`/_includes/tabs.md`](https://git.k8s.io/kubernetes.github.io/_includes/tabs.md) 文件被包含进来，然后使用这些元素渲染实际的选项卡设置。


下面的部分将分解所使用的每个单独的特性。

````liquid
{{ "{% capture default_tab " }}%}

选择其中一个选项卡。
{{ "{% endcapture " }}%}

{{ "{% capture calico " }}%}
```shell
kubectl apply -f "http://docs.projectcalico.org/v2.4/getting-started/kubernetes/installation/hosted/kubeadm/calico.yaml"
```
{{ "{% endcapture " }}%}

{{ "{% capture flannel " }}%}
```shell
kubectl apply -f "https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml?raw=true"
```
{{ "{% endcapture " }}%}

{{ "{% capture romana " }}%}
```shell
kubectl apply -f "https://raw.githubusercontent.com/romana/romana/master/containerize/specs/romana-kubeadm.yml"
```
{{ "{% endcapture " }}%}

{{ "{% capture weave_net " }}%}
```shell
kubectl apply -f "https://git.io/weave-kube"
```
{{ "{% endcapture " }}%}

{{ "{% assign tab_names = 'Default,Calico,Flannel,Romana,Weave Net' | split: ',' | compact " }}%}
{{ "{% assign tab_contents = site.emptyArray | push: default_tab | push: calico | push: flannel | push: romana | push: weave_net " }}%}

{{ "{% include tabs.md " }}%}
````


#### 捕获标签内容

````liquid
{{ "{% capture calico " }}%}
```shell
kubectl apply -f "http://docs.projectcalico.org/v2.4/getting-started/kubernetes/installation/hosted/kubeadm/calico.yaml"
```
{{ "{% endcapture " }}%}
````


`capture [variable_name]` 标签存储文本或模型内容，并且将它们分配给指定的变量。


#### 指定选项卡名称

````liquid
{{ "{% assign tab_names = 'Default,Calico,Flannel,Romana,Weave Net' | split: ',' | compact " }}%}
````


`assign tab_names` 标签获取选项卡中使用的标签列表。标签文本可以包括空格。通过逗号分隔的字符串将会拆分为一个数组，然后赋值给 `tab_names` 。


#### 指定选项卡内容

````liquid
{{ "{% assign tab_contents = site.emptyArray | push: default_tab | push: calico | push: flannel | push: romana | push: weave_net " }}%}
````


`assign tab_contents` 标签为选项卡面板增加内容。它将捕获上面定义的内容，然后将它们做为元素添加到 `tab_contents` 数组中。


#### 引入 tabs.md 模板

````liquid
{{ "{% include tabs.md " }}%}
````


在选项卡模板代码中引入 `{{ "{% include tabs.md " }}%}`，它会使用 `tab_names` 和 `tab_contents` 变量来渲染选项卡设置。

{% endcapture %}

{% capture whatsnext %}

* 学习 [Jekyll](https://jekyllrb.com/docs)。
* 学习 [写一个新的主题](/docs/home/contribute/write-new-topic/)。
* 学习 [使用页模板](/docs/home/contribute/page-templates/)。
* 学习 [阶段化您的变更](/docs/home/contribute/stage-documentation-changes/)。
* 学习 [创建一个 PR](/docs/home/contribute/create-pull-request/)。
{% endcapture %}

{% include templates/concept.md %}
