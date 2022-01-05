[TOC]

# 快速开始

## 目标

- 在kubernetes集群中部署Ploto任务调度与执行框架
- 创建Task, DynamicExecutorPool自定义资源定义（Custom Resource Definition）
- 创建一系列的Task实例，和一个示例DynamicExecutorPool，然后ploto-controller会把这些task分配给DynamicExecutorPool管理的裸pod，pod中的应用容器消费任务（本例为打印task中的para）

## 前提条件

- 可用的kubernetes集群，建议版本>=1.16.3

## 步骤

### 1. 下载ploto-demo项目

ploto-demo中保存了本示例所需的yaml文件

```shell
git clone https://github.com/xawei/ploto-demo.git
cd ploto-demo
```



### 2. 创建CRD和新建命名空间

```shell
kubectl apply -f manifest/crds/ploto.io_dynamicexecutorpools.yaml
kubectl apply -f manifest/crds/ploto.io_fixedexecutorpools.yaml
kubectl apply -f manifest/crds/ploto.io_tasks.yaml
kubectl create ns ploto-system
kubectl create ns ploto-demo
```

查看CRD：

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201130163857.png)



### 3. 创建默认资源池扩缩容算法的Webhook

```shell
kubectl apply -f manifest/webhook/autoscaler-webhook-service.yaml
```



### 4. 创建Ploto Controller

```shell
kubectl apply -f manifest/controllers/ploto-controller.yaml
```

查看ns ploto-system下的资源：

```shell
kubectl get all -n ploto-system
```

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201130164726.png)

至此，我们已经完成了在kubernetes至少上部署了Ploto框架所必须的预备资源。下面再继续创建与业务相关的Task和DynamicExecutorPool自定义资源。



### 5. 创建自定义的Dynamic Executor Pool

准备创建一个DynamicExecutorPool，先查看一下我们定义的dep1.yaml：

```shell
cat manifest/deps/dep-simple-logger.yaml
```

```yaml
# manifest/deps/dep-simple-logger.yaml
apiVersion: "ploto.io/v1"
kind: DynamicExecutorPool
metadata:
  name: dep-simple-logger
  namespce: ploto-demo
spec:
  initialExecutor: 2 # executor资源池的初始pod（执行器）数量
  maxExecutor: 5 # executor资源池的最大pod（执行器）数量
  minExecutor: 2 # executor资源池的最小pod（执行器）数量
  # 动态扩缩容的webhook，可选，默认预估下一周所需executor pod数量为
  # 本周期在执行任务的executor pod数量的1.3倍
  webhook: 
    service:
      namespace: ploto-system
      name: autoscaler-webhook-service
      path: /scale
      port: 8000
  container: # 应用容器，消费task中的para信息
    command:
      - /app/bin/simple-logger
    args:
      - "--simulate-exe-time=60" # 应用容器可指定模拟任务执行的耗时，再打印task信息
    env:
      - name: PATH
        value: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    image: ccr.ccs.tencentyun.com/ploto/simple-logger:1.0
    imagePullPolicy: Always
    name: simple-logger
    resources:
      limits:
        cpu: 500m
        memory: 1Gi
      requests:
        cpu: 250m
        memory: 256Mi
    securityContext:
      privileged: false
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    workingDir: /
```

其中定义的container是我们的应用容器，持续监听本地任务文件，一旦任务文件存在，则会去读Task.Spec.Para中的数据，执行任务。

创建这个dynamic executor pool（简称dep）：

```shell
kubectl apply -f manifest/deps/dep-simple-logger.yaml
```

可以查看目前dep的状态：

```shell
kubectl get dep -o wide -n ploto-demo
kubectl get pod -n ploto-demo
```

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201202220002.png)

dep-simple-logger刚创建时，还没有executor pod，当ploto-controller监听到dep-simple-logger时，则为这个dep创建了2个pod（ initialExecutor: 2）。



### 6. 创建task，并查看执行情况

manifest/tasks/中准备了6个task，我们查看其中一个task1.yaml，下面附上了注释：

```
cat manifest/tasks/task1.yaml
```

```yaml
apiVersion: "ploto.io/v1"
kind: Task
metadata:
  name: task1
spec:
  executorPool: dep-simple-logger # 指定消费这个task的executor pod所属的资源池
  executorPoolType: Dynamic # 指定消费这个task的executor pod所属的资源池类型
  para: https://ploto.io/tasks/1.html # task携带的任务信息
```

创建这些task：

```shell
kubectl apply -f manifest/tasks/
```

如上图所示，6个新创建的task，初始状态为Pending，等待调度关联到对应的executorPool管理的pod，由pod执行消费任务。

```
kubectl get task -n ploto-demo
kubectl get pod -n ploto-demo
```

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201202220221.png)



task状态为Running后，等待manifest/deps/dep1.yaml中定义的simulate-exe-time启动参数值（单位秒）的时间后，可以看到部分task执行完成了（status: completed）：

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201202220326.png)

查看对应的executor pod，可以看到输出（消费task）：

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201202220655.png)

本例executor pod为简单地监听分配到的task任务，并打印task中的para参数。

ploto-controller会根据task的数量、执行情况，周期性地对dep管理的executor pod资源数量，在[minExecutor, maxExecutor]区间内做动态的扩缩容。默认为每一个周期需要的pod数量为上一周期的1.3倍。你也可以实现自定义的扩缩容webhook，在dep中配置。

等待所有任务执行完后，我们查看dep的状态，以及它所关联的pod。发现pod数量已缩容至dep-simple-loger实例中设置的最小值minExexutor: 2。

![](https://weiblog.oss-cn-beijing.aliyuncs.com/img/20201202220809.png)



### 附： 清理ploto相关资源

```shell
kubectl delete -f manifest/crds/
kubectl delete -f manifest/sa/
kubectl delete -f manifest/controllers/

kubectl delete ns ploto-system
kubectl delete ns ploto-demo
```

