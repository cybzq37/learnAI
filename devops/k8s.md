## Container

```yml
# hellok8s.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hellok8s
spec:
  containers:
    - name: hellok8s-container
      image: guangzhengli/hellok8s:v1
```

```bash
kubectl apply -f hellok8s.yaml

kubectl get pods

kubectl get pod -o wide

kubectl port-forward hellok8s 3000:3000

kubectl describe pod hellok8s-deployment-66799848c4-kpc6q
```

```bash
NAME       READY   STATUS             RESTARTS   AGE
hellok8s   0/1     ImagePullBackOff   0          22m
```

## Pod

## Deployment

如果我们在生产环境上，管理着多个副本的 hellok8s:v1 版本的 pod，我们需要更新到 v2 的版本，像上面那样的部署方式是可以的，但是也会带来一个问题，就是所有的副本在同一时间更新，这会导致我们 hellok8s 服务在短时间内是不可用的，因为所有 pod 都在升级到 v2 版本的过程中，需要等待某个 pod 升级完成后才能提供服务。

这个时候我们就需要滚动更新 (rolling update)，在保证新版本 v2 的 pod 还没有 ready 之前，先不删除 v1 版本的 pod。

在 deployment 的资源定义中, spec.strategy.type 有两种选择:

RollingUpdate: 逐渐增加新版本的 pod，逐渐减少旧版本的 pod。
Recreate: 在新版本的 pod 增加前，先将所有旧版本 pod 删除。
大多数情况下我们会采用滚动更新 (RollingUpdate) 的方式，滚动更新又可以通过 maxSurge 和 maxUnavailable 字段来控制升级 pod 的速率，具体可以详细看官网定义。：

maxSurge: 最大峰值，用来指定可以创建的超出期望 Pod 个数的 Pod 数量。
maxUnavailable: 最大不可用，用来指定更新过程中不可用的 Pod 的个数上限。

## Service

Service 为 pod 提供一个稳定的 Endpoint。Service 位于 pod 的前面，负责接收请求并将它们传递给它后面的所有pod。一旦服务中的 Pod 集合发生更改，Endpoints 就会被更新，请求的重定向自然也会导向最新的 pod。

## Ingress

## Namespace

## ConfigMap

## Secret

## Job



