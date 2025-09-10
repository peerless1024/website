---
title: "Envoy 网格接入"
linkTitle: "Envoy 网格接入"
weight: 4
---
> 注意：当前此接入方案处于实验阶段，请勿在生产环境使用

在 `Polaris` 的服务网格方案中，`Polaris` 是您的控制平面，`Envoy Sidecar` 代理是您的数据平面。当前

![](../images/envoy/架构图.png)

- 服务数据同步：`polaris-controller` 安装在用户的Kubernetes集群中，可以同步集群上的 Namespace，Service，Endpoints 等资源到 `polaris` 中，同时 `polaris-controller` 提供了 `Envoy Sidecar` 注入器功能，可以轻松地将 `Envoy Sidecar` 注入到您的 Kubernetes Pod 中，Envoy Sidecar 会自动去 Polaris 同步服务信息。

- 规则数据下发：```polaris```控制面通过XDS v3标准协议与 envoy 进行交互，支持官方开源的 envoy 直接接入，当前支持的 envoy 版本为 1.26.2

## 环境准备

### 部署polaris

如果已经部署好了polaris，可忽略这一步。

polaris支持在kubernetes环境中进行部署，注意必须保证暴露HTTP端口为8090，gRPC端口为8091。具体部署方案请参考：

- [单机版部署指南](/docs/使用指南/服务端安装/单机版安装/)
- [集群版部署指南](/docs/使用指南/服务端安装/集群版安装/)

### 部署 polaris-controller 

- [controller 部署](/docs/使用指南/k8s和网格代理/安装polaris-controller/)

## 快速接入

### 服务调用关系说明

![](../images/envoy/bookinfo.png)

### 启用 sidecar 自动注入功能

- 创建命名空间 `bookinfo` ：```kubectl create namespace bookinfo```

- 为 `bookinfo` 命名空间启用注入：
  
```
kubectl label namespace bookinfo polaris-injection=enabled 
```

使用一下命令来验证 `bookinfo` 命名空间是否已经正确启用：

```
kubectl get namespace -L polaris-injection
```

此时应该返回：

```
NAME             STATUS   AGE    POLARIS-INJECTION
bookinfo          Active   3d2h   enabled
```

### 启用 Envoy 按需加载机制

{{< note >}}
- polaris-server 需要 >= 1.18.0-beta 版本
- polaris-controller 需要 >= 1.7.0-beta 版本
{{</ note >}}

在 POD 中添加以下 annonations **sidecar.polarismesh.cn/openOnDemand: true** 即可启用 Envoy 的按需加载

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotations-demo
  annotations:
    sidecar.polarismesh.cn/openOnDemand: "true"
```

### 部署样例

- 下载样例部署文件：[bookinfo](https://github.com/polarismesh/examples/blob/main/servicemesh/bookinfo/bookinfo.yaml)

- 执行部署：```kubectl create -f bookinfo.yaml```

- 查看容器注入是否注入成功

启动自动注入后，`polaris-controller` 会将 `Envoy Sidecar`、`Polaris Sidecar` 容器注入到在此命名空间下创建的 pod 中。

可以看到运行起来的 pod 均包含三个容器，其中第一个容器是用户的业务容器，另外两个容器是由 Polaris Controller 注入器注入的 Envoy Sidecar 容器和 Polaris Sidecar 容器。您可以通过下面的命令来获取有关 pod 的更多信息：

```
kubectl describe pods -l app=productpage --namespace=bookinfo
```

此时应返回：

```
... ...
Init Containers:
# polaris-bootstrap-writer 产生 Envoy 的 Bootstrap 配置
polaris-bootstrap-writer:
... ... 
# istio-init 为 envoy sidecar 设置流量拦截
istio-init:
... ... 
Containers:
# demo 的业务容器
productpage:
... ...
# Envoy 是代理流量的容器
envoy:
... ... 
```

- 打开productpage界面

通过productpage暴露的地址，可以访问productpage的主界面，进入Normal User或者TestUser后，可以看到（红、黑、无）三种形态的星星，代表demo已经部署成功。

## 使用服务治理能力

### 流量调度

北极星网格支持根据http请求的头部字段进行路由，支持通过path, header, query这3种类型的属性来进行路由。

1. 使用场景

demo 项目中，productpage 会访问 reviews 服务，reviews 服务共有三个实例，三个实例分别部署了三个版本（会显示红、黑、无三种颜色的星星），需要保证特定的灰度用户（用户名为jason），请求到特定版本的 reviews 服务。

2. 配置路由规则

为 reviews 服务创建路由规则。将请求中 header 包含字段 end-user=jason 的请求，路由到 version=v2 的服务实例中。同时再创建一条路由规则，指定标签键值为任意请求，路由到 version=v1 的服务实例中。

![](../images/envoy/路由规则.png)

路由规则的标签填写格式要求：

- 对于Path：标签KEY需要填写 **$path**
- 对于Header：标签KEY需要带上前缀 **$header**，如 **$header.end-user**
- 对于Query：标签KEY需要带上前缀 **$query**，如 **$query.end-user**

3. 验证路由是否生效

未登陆时，刷新 productpage 的页面，可以看到只返回没有颜色的星星（version=v1）。当使用 `jason` 登陆后，productpage 请求 reviews 时，会带上 header，end-user=jason，此时再刷新 productpage 页面，发现只会显示黑色的星星，即上面 version=v2 的实例。

### 访问限流

{{< note >}}
- polaris-server 需要 >= 1.17.3 版本
- polaris-controller 需要 >= 1.6.0 版本
- 在envoy接入的场景中，受XDS协议的限制，当前限流标签仅支持以下两个。
  - 请求头(HEADER)
  - 请求参数(QUERY)
{{< /note >}}

![](../images/envoy/分布式限流.png)

1. 使用场景: demo项目中，为 detail 服务设置流量限制，对于jason用户的请求，设置访问的频率为5/m，其余请求不做限制。

2. 设置限流规则: 指定请求中 header 包含字段 end-user=jason 的请求，设置限流规则为5/m，限流类型为分布式限流。

    ![](../images/envoy/限流规则.png)

    {{< note >}}
    详细的限流规则匹配及使用指南可参考：[访问限流](/docs/使用指南/控制台使用/服务网格/访问限流/)
    {{< /note >}}

3. 验证限流是否生效: 未登陆时，多次刷新界面，不会出现错误。以jason用户身份登陆，一分钟刷新超过5次，detail 界面出现限流的错误信息。


## mTLS

北极星网络支持服务间的mTLS认证及加密通讯，提供三种不同的服务粒度模式供用户选择：

| 模式       | 解释                                                                                                 |
|------------|----------------------------------------------------------------------------------------------------|
| Permissive | 宽容模式，服务接受纯文本/mTLS服务调用；发起服务调用时，根据对端接受状况自动选择发起mTLS或纯文本服务调用 |
| Strict     | 严格模式，服务仅接受/发起mTLS服务调用                                                                 |
| None       | 无加密模式（为默认选项），服务仅接受/发起纯文本服务调用                                                 |

### 启用方式

只需要在服务的`metadata`中加入键为`polarismesh.cn/tls-mode`的`label`即可开启该功能，可选的值为`strict`,`permissive`,`none`，无此`label`时或者值错误时，默认为无加密的`none`模式。

### 使用示例

#### 部署polaris-security

`polaris-security`是北极星的证书机构，负责签发证书以供服务使用，是开启双向TLS功能的必要组件。
- 下载[polaris-security](https://github.com/polarismesh/polaris-security)
- 将示例证书/密钥加载为k8s secret：```./deploy/load-certs-into-k8s.sh```
- 验证secret加载成功：```kubectl get secrets -n polaris-system``` 
- 使用Helm部署`polaris-security`, ```cd deploy/helm && helm install polaris-security .```
- 验证polaris-security部署成功：```kubectl get po -n polaris-system | grep polaris-security```

#### 部署mTLS版bookinfo示例
- 下载样例部署文件：[mTLS版bookinfo](https://github.com/polarismesh/examples/blob/main/servicemesh/bookinfo-mtls/bookinfo.yaml)
- 执行部署：```kubectl create -f bookinfo.yaml```

mTLS版bookinfo在配置文件中使用`polarismesh.cn/tls-mode`的`label`为不同的服务启用了各自的双向TLS模式，部署完成后服务调用图如下所示：

![](../images/envoy/mtls.png)

#### 效果验证
1. Strict模式验证
由于`Reviews V3`服务使用了None模式，它将向`Ratings`服务发起纯文本请求，而`Ratings`服务使用了Strict模式，仅接受mTLS服务调用，因此`Reviews V3`到`Ratings`之间的服务调用总会失败。  
因此，使用浏览器访问部署好的`ProductPage`，无论怎么刷新都无法看到红色的星星评级。

2. mTLS 验证
使用 Wireshark 抓包验证 mTLS 启用,如下图：

![](../images/envoy/wireshark.png)

可以看到 Server 向 Client 提供证书后，要求 Client 提供自身证书，验证通过后方可开始加密数据通信。


## 相关链接

[Polaris](https://github.com/polarismesh)

[Polaris Controller](https://github.com/PolarisMesh/polaris-controller)

[Polaris Demo](https://github.com/polarismesh/examples/tree/main/servicemesh/bookinfo)
