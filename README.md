# EKS Fluent bit 日志采集

[EKS](https://console.cloud.tencent.com/tke2/cluster?rid=1) 场景下，通过 `Fluent bit` 自定义采集配置来采集容器的标准输出日志。
当前已支持：

1、kafka 投递，支持用户密码认证

2、es 投递

## 关键注解介绍

```yaml
## 指定日志采集组件
eks.tke.cloud.tencent.com/log-collector: fluent-bit   

## 指定日志采集配置对应的configmap
eks.tke.cloud.tencent.com/log-collector-configmap: logging/fluent-bit-config

## 开启配置变更自动重启采集组件
eks.tke.cloud.tencent.com/hot-reload-config: "true"
```

## F-bit 权限

用于与 EKS 集群通信，获取 pod 的 metadata 等信息。

```shell
# 创建命名空间
$ kubectl create namespace logging

$ kubectl -n logging apply -f fbit-serviceaccount.yaml
```

## F-bit 配置文件 configmap 详细配置

备注：`<>` 是需要客户自行调整的

### 通用配置

1、`ca.crt` 和 `token`:  连接 EKS 集群，获取命令参考如下: 
```shell
# 获取 ca.crt
$ kubectl -n logging get ServiceAccount fluent-bit -o jsonpath='{.secrets[0].name}' | xargs kubectl -n logging get secret -o jsonpath='{.data.ca\.crt}' | base64 -d

# 获取 token
$ kubectl -n logging get ServiceAccount fluent-bit -o jsonpath='{.secrets[0].name}' | xargs kubectl -n logging get secret -o jsonpath='{.data.token}' | base64 -d
```

2、`kubernetes.endpoints`: EKS apiserver 连接地址

```shell
# 获取 ENDPOINTS 字段
$ kubectl get endpoints kubernetes
```

### kafka 相关配置

配置块位置在 `output-kafka.conf`(不需要kafka投递则可直接删除)

1、`Brokers` 和 `Topic`: kafka 投递配置

2、`username` 和 `password`: kafka 用户验证配置

### es 相关配置

配置块位置在 `output-es.conf`(不需要es投递则可直接删除)

1、`Host` 和 `Port`: es 的 Hostname 和 端口

2、`HTTP_User` 和 `HTTP_Passwd`: es 用户认证相关配置

完成上述配置字段替换之后，执行如下命令创建：

```shell
$ kubectl -n logging apply -f fbit-cm.yaml
```

## F-bit 采集注解的全局配置

避免每次都要修改 pod 的配置，插入注解。

EKS 支持注解全局配置，参考如下：

```yaml
apiVersion: v1
data:
  pod.annotations: |-
    eks.tke.cloud.tencent.com/log-collector: fluent-bit
    eks.tke.cloud.tencent.com/log-collector-configmap: logging/fluent-bit-config
    eks.tke.cloud.tencent.com/hot-reload-config: "true"
kind: ConfigMap
metadata:
  name: eks-config
  namespace: kube-system
```

补充：Fluent bit 的版本 1.8.9