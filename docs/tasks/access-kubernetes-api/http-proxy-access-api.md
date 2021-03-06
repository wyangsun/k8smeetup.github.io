---
title: 使用 HTTP 代理访问 Kubernetes API
cn-approvers:
- zhangqx2010
---



{% capture overview %}
本文描述如何使用 HTTP 代理访问 Kubernetes API。
{% endcapture %}

{% capture prerequisites %}

* {% include task-tutorial-prereqs.md %}


* 如果您的集群中还没有任何应用，使用如下命令启动一个 Hello World 应用：

      kubectl run node-hello --image=gcr.io/google-samples/node-hello:1.0 --port=8080

{% endcapture %}

{% capture steps %}


## 使用 kubectl 启动代理服务器


使用如下命令启动 Kubernetes API server 的代理：

    kubectl proxy --port=8080


## 访问 Kubernetes API


当代理服务器在运行时，你可以通过 `curl`、`wget` 或者浏览器访问 API。


获取 API 版本：

    curl http://localhost:8080/api/

    {
      "kind": "APIVersions",
      "versions": [
        "v1"
      ],
      "serverAddressByClientCIDRs": [
        {
          "clientCIDR": "0.0.0.0/0",
          "serverAddress": "10.0.2.15:8443"
        }
      ]
    }


获取 Pod 列表：

    curl http://localhost:8080/api/v1/namespaces/default/pods

    {
      "kind": "PodList",
      "apiVersion": "v1",
      "metadata": {
        "selfLink": "/api/v1/namespaces/default/pods",
        "resourceVersion": "33074"
      },
      "items": [
        {
          "metadata": {
            "name": "kubernetes-bootcamp-2321272333-ix8pt",
            "generateName": "kubernetes-bootcamp-2321272333-",
            "namespace": "default",
            "selfLink": "/api/v1/namespaces/default/pods/kubernetes-bootcamp-2321272333-ix8pt",
            "uid": "ba21457c-6b1d-11e6-85f7-1ef9f1dab92b",
            "resourceVersion": "33003",
            "creationTimestamp": "2016-08-25T23:43:30Z",
            "labels": {
              "pod-template-hash": "2321272333",
              "run": "kubernetes-bootcamp"
            },
            ...
    }

{% endcapture %}


{% capture whatsnext %}
想了解更多信息，请参阅 [kubectl proxy](/docs/user-guide/kubectl/{{page.version}}/#proxy)。
{% endcapture %}

{% include templates/task.md %}
