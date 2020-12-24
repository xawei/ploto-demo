# Ploto性能压测方案

[TOC]

## 1. 前期准备

- 可用的kubernetes集群，建议版本>=1.16.3
- 在集群上部署好Ploto（参考步骤1~4：https://github.com/xawei/ploto-demo）



## 2. 部署Prometheus和Grafana

（1）使用kube-prometheus快速部署

https://github.com/prometheus-operator/kube-prometheus

```shell 
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus

# Create the namespace and CRDs, and then wait for them to be availble before creating the remaining resources
kubectl create -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl create -f manifests/
```

（2）访问grafana

```
kubectl --namespace monitoring port-forward svc/grafana 3000 --address 0.0.0.0 &
```

初始用户名/密码是admin/admin

（如果是使用TKE独立集群，请注意master/worker节点的安全组设置。或建议使用公网service）



可以查看prometheus的配置：

```shell
kubectl -n monitoring get secret prometheus-k8s -ojson | jq -r '.data["prometheus.yaml.gz"]' | base64 -d | gunzip
```



## 3. 配置ploto-vegeta为prometheus的target

下载ploto-demo的manifest文件

```
git clone https://github.com/xawei/ploto-demo.git
```

配置ploto-vegeta为prometheus的target

```
# 创建servicemonitor，使prometheus采集ploto-vegeta的metrics数据
kubectl apply -f manifest/loadtest/ploto-vegeta-servicemonitor.yaml
```

修改serviceAccount prometheus-k8s 对应的clusterRole prometheus-k8s权限，使其可以操作ploto-system下的资源：

原ClusterRole prometheus-k8s的配置如下：

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201214233227.png)

修改为：

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201215101621.png)



启动ploto-vegeta

```
kubectl apply -f manifest/loadtest/ploto-vegeta-pod-and-svc.yaml
```

相关重要启动参数解析如下，可以修改yaml自定义启动参数：

```
...
spec:
  containers:
  - args:
    - --apiserver-addr=https://kubernetes.default
    - --ca-file=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    - --token-file=/var/run/secrets/kubernetes.io/serviceaccount/token
    - --duration=1200 # 压测持续的时间。
    - --task-para-len=1 # 压测创建的task中para的字符串长度
    - --mul-freq=false # 是否测试多种不同客户端的并发请求
    - --min-freq=1 # 如果非多种不同的freq，则默认取这个min-freq的值
...
```

查看采集的数据，例子如下：

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201215110416.png)

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201215110436.png)

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201215110522.png)

例子的snapshop可参见：

https://snapshot.raintank.io/dashboard/snapshot/ZwWu3XiwWhQwiwkBygH9eAeav1dREWKa



## 4. 查看task消费端统计数据

//todo: 在controller中暴露metrics，统计系统中task的相关统计数据，包括task从创建到开始处理的时间百分位、平均值等。

## 清理kube-prometheus

```
cd kube-prometheus
kubectl delete -f manifests/setup
kubectl delete -f manifests/
```


