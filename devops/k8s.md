## Container

Container (容器) 是一种沙盒技术。它是基于 Linux 中 Namespace / Cgroups / chroot 等技术结合而成。

自己实现一个容器 https://www.youtube.com/watch?v=8fi7uSYlOdc

## Pod

Pod 是我们将要创建的第一个 k8s 资源，也是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。与Container的区别： container (容器) 的本质是进程，而 pod 是管理这一组进程的资源， pod 可以管理多个 container。

编写一个可以创建 nginx 的 Pod，其中：kind 表示要创建的资源的类型， metadata.name 表示要创建的 pod 的名字（唯一）， spec.containers 表示要运行的容器和镜像名称。

```yml
# nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
```

```bash

kubectl apply -f nginx.yaml # 创建pod
kubectl get pods -o wide # 查看pod
kubectl port-forward nginx-pod 4000:80 # 端口映射
kubectl exec -it nginx-pod -- /bin/bash # 进入pod
kubectl describe pod hellok8s-deployment-66799848c4-kpc6q # 
kubectl logs --follow nginx # 查看日志
kubectl exec nginx -- ls # 调用容器命令
kubectl delete pod nginx
kubectl delete -f nginx.yaml
```

```bash
NAME       READY   STATUS             RESTARTS   AGE
hellok8s   0/1     ImagePullBackOff   0          22m
```

## Deployment

在生产环境中，我们不会直接管理 pod，借助 Deployment 管理 pod，完成自动扩容或者自动升级版本。

扩容：

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
        - image: guangzhengli/hellok8s:v1
          name: hellok8s-container
```

selector 表示的是 deployment 资源和 pod 资源关联的方式，这里表示 deployment 会管理 (selector) 所有 labels=hellok8s 的 pod。

template 是用来定义 pod 资源的, metadata.labels 和上面的 selector.matchLabels 对应, 表明 pod 是被 deployment 管理，不用在template 里面加上 metadata.name 是因为 deployment 会主动为我们创建 pod 的唯一 name。

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods             
kubectl delete pod hellok8s-deployment-77bffb88c5-qlxss 
kubectl describe pod hellok8s-deployment-66799848c4-kpc6q
kubectl rollout undo deployment hellok8s-deployment
kubectl rollout history deployment hellok8s-deployment
kubectl rollout undo deployment/hellok8s-deployment --to-revision=2
kubectl get pods
kubectl delete deployment,service --all
```

自动扩容：要想将 hellok8s:v1 扩容到 3 个副本，只需将 replicas 的值设为 3，重新执行 `kubectl apply -f deployment.yaml`。

升级版本：只需要更改 `spec.containers.image` 的镜像的版本号。

滚动更新：在 deployment 的资源定义中, spec.strategy.type 有两种选择:

- RollingUpdate: 逐渐增加新版本的 pod，逐渐减少旧版本的 pod。
- Recreate: 在新版本的 pod 增加前，先将所有旧版本 pod 删除。

可以通过 maxSurge 和 maxUnavailable 字段来控制升级 pod 的速率。

- maxSurge: 最大峰值，用来指定可以创建的超出期望 Pod 个数的 Pod 数量。
- maxUnavailable: 最大不可用，用来指定更新过程中不可用的 Pod 的个数上限。

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s-deployment
spec:
  strategy:
     rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  replicas: 3
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
        - image: guangzhengli/hellok8s:liveness
          name: hellok8s-container
          livenessProbe:
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 3
          readinessProbe:
            httpGet:
              path: /healthx
              port: 3000
            initialDelaySeconds: 1
            successThreshold: 5
```

存活探针 (livenessProb)：用来探测应用是否正常，确定是否重启。

就绪探针 (readiness)：kubelet 使用就绪探测器可以知道容器何时准备好接受请求流量，当一个 pod 升级后不能就绪，即不应该让流量进入该 pod，在配合 rollingUpate 的功能下，也不能允许升级版本继续下去，否则服务会出现全部升级完成，导致所有服务均不可用的情况。

## Service

Service 为 pod 提供一个稳定的 Endpoint。Service 位于 pod 的前面，负责接收请求并将它们传递给它后面的所有pod。一旦服务中的 Pod 集合发生更改，Endpoints 就会被更新，请求的重定向自然也会导向最新的 pod。

使用 ClusterIP 的方式定义 Service，通过 kubernetes 集群的内部 IP 暴露服务，当我们只需要让集群中运行的其他应用程序访问我们的 pod 时，就可以使用这种类型的Service。

ServiceTypes类型：

- ClusterIP：通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。  
- NodePort：通过每个节点上的 IP 和静态端口（NodePort）暴露服务。 NodePort 服务会路由到自动创建的 ClusterIP 服务。 通过请求 <节点 IP>:<节点端口>，你可以从集群的外部访问一个 NodePort 服务。
- LoadBalancer：使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 NodePort 服务和 ClusterIP 服务上。
- ExternalName：通过返回 CNAME 和对应值，可以将服务映射到 externalName 字段的内容（例如，foo.bar.example.com）。 无需创建任何类型代理。

```yml
apiVersion: v1
kind: Service
metadata:
  name: service-hellok8s-clusterip
spec:
  type: ClusterIP
  selector:
    app: hellok8s
  ports:
  - port: 3000
    targetPort: 3000
```

```bash
kubectl apply -f service-hellok8s-clusterip.yaml

kubectl get endpoints
# NAME                         ENDPOINTS                                          AGE
# service-hellok8s-clusterip   172.17.0.10:3000,172.17.0.2:3000,172.17.0.3:3000   10s

kubectl get pod -o wide
# NAME                                   READY   STATUS    RESTARTS   AGE    IP           NODE 
# hellok8s-deployment-5d5545b69c-24lw5   1/1     Running   0          112s   172.17.0.7   minikube 
# hellok8s-deployment-5d5545b69c-9g94t   1/1     Running   0          112s   172.17.0.3   minikube
# hellok8s-deployment-5d5545b69c-9gm8r   1/1     Running   0          112s   172.17.0.2   minikube
# nginx                                  1/1     Running   0          112s   172.17.0.9   minikube

kubectl get service
# NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# service-hellok8s-clusterip   ClusterIP   10.104.96.153   <none>        3000/TCP   10s
```

NodePort：k8s 集群并不是单机运行，它管理着多台节点（服务器）即 Node，可以通过每个节点上的 IP 和静态端口（NodePort）暴露服务。

```yml
apiVersion: v1
kind: Service
metadata:
  name: service-hellok8s-nodeport
spec:
  type: NodePort
  selector:
    app: hellok8s
  ports:
  - port: 3000
    nodePort: 30000
```

LoadBalancer 是使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 NodePort 服务和 ClusterIP 服务上，假如你在 AWS 的 EKS 集群上创建一个 Type 为 LoadBalancer 的 Service。它会自动创建一个 ELB (Elastic Load Balancer) ，并可以根据配置的 IP 池中自动分配一个独立的 IP 地址，可以供外部访问。

## Ingress

Ingress 可以“简单理解”为服务的网关 Gateway，它是所有流量的入口，经过配置的路由规则，将流量重定向到后端的服务。

Ingress 公开从集群外部到集群内服务的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。Ingress 可为 Service 提供外部可访问的 URL、负载均衡流量、 SSL/TLS，以及基于名称的虚拟托管。你必须拥有一个 Ingress 控制器 才能满足 Ingress 的要求。 仅创建 Ingress 资源本身没有任何效果。 Ingress 控制器 通常负责通过负载均衡器来实现 Ingress，例如 minikube 默认使用的是 nginx-ingress，目前 minikube 也支持 Kong-Ingress。

```yml
apiVersion: v1
kind: Service
metadata:
  name: service-hellok8s-clusterip
spec:
  type: ClusterIP
  selector:
    app: hellok8s
  ports:
  - port: 3000
    targetPort: 3000

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
        - image: guangzhengli/hellok8s:v3
          name: hellok8s-container
```

```yml
apiVersion: v1
kind: Service
metadata:
  name: service-nginx-clusterip
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 4000
    targetPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-container
```

```bash
kubectl apply -f hellok8s.yaml                 
# service/service-hellok8s-clusterip created
# deployment.apps/hellok8s-deployment created

kubectl apply -f nginx.yaml   
# service/service-nginx-clusterip created
# deployment.apps/nginx-deployment created

kubectl get pods            
# NAME                                   READY   STATUS    RESTARTS   AGE
# hellok8s-deployment-5d5545b69c-4wvmf   1/1     Running   0          55s
# hellok8s-deployment-5d5545b69c-qcszp   1/1     Running   0          55s
# hellok8s-deployment-5d5545b69c-sn7mn   1/1     Running   0          55s
# nginx-deployment-d47fd7f66-d9r7x       1/1     Running   0          34s
# nginx-deployment-d47fd7f66-hp5nf       1/1     Running   0          34s

kubectl get service
# NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
# service-hellok8s-clusterip   ClusterIP   10.97.88.18      <none>        3000/TCP   77s
# service-nginx-clusterip      ClusterIP   10.103.161.247   <none>        4000/TCP   56s
```

这样在 k8s 集群中，就有 3 个 hellok8s:v3 的 pod，2 个 nginx 的 pod。并且hellok8s:v3 的端口映射为 3000:3000，nginx 的端口映射为 4000:80。在这个基础上，接下来编写 Ingress 资源的定义，nginx.ingress.kubernetes.io/ssl-redirect: "false" 的意思是这里关闭 https 连接，只使用 http 连接。

匹配前缀为 /hello 的路由规则，重定向到 hellok8s:v3 服务，匹配前缀为 / 的跟路径重定向到 nginx。

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    # We are defining this annotation to prevent nginx
    # from redirecting requests to `https` for now
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
    - http:
        paths:
          - path: /hello
            pathType: Prefix
            backend:
              service:
                name: service-hellok8s-clusterip
                port:
                  number: 3000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-nginx-clusterip
                port:
                  number: 4000
```

```bash
kubectl apply -f ingress.yaml
# ingress.extensions/hello-ingress created

kubectl get ingress          
# NAME            CLASS   HOSTS   ADDRESS   PORTS   AGE
# hello-ingress   nginx   *                 80      16s

# replace 192.168.59.100 by your minikube ip
curl http://192.168.59.100/hello
# [v3] Hello, Kubernetes!, From host: hellok8s-deployment-5d5545b69c-sn7mn

curl http://192.168.59.100/
# (....Thank you for using nginx.....)
```

## Namespace

在实际的开发当中，有时候我们需要不同的环境来做开发和测试，例如 dev 环境给开发使用，test 环境给 QA 使用，那么 k8s 能不能在不同环境 dev test uat prod 中区分资源，让不同环境的资源独立互相不影响呢，答案是肯定的，k8s 提供了名为 Namespace 的资源来帮助隔离资源。

名字空间（Namespace） 提供一种机制，将同一集群中的资源划分为相互隔离的组。 同一名字空间内的资源名称要唯一，但跨名字空间时没有这个要求。 名字空间作用域仅针对带有名字空间的对象，例如 Deployment、Service 等。

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  
---

apiVersion: v1
kind: Namespace
metadata:
  name: test
```

可以通过kubectl apply -f namespaces.yaml 创建两个新的 namespace，分别是 dev 和 test。

```bash
kubectl apply -f namespaces.yaml    
# namespace/dev created
# namespace/test created


kubectl get namespaces          
# NAME              STATUS   AGE
# default           Active   215d
# dev               Active   2m44s
# ingress-nginx     Active   110d
# kube-node-lease   Active   215d
# kube-public       Active   215d
# kube-system       Active   215d
# test              Active   2m44s
```

如何在新的 namespace 下创建资源和获取资源呢？只需要在命令后面加上 -n namespace 即可。例如根据上面教程中，在名为 dev 的 namespace 下创建 hellok8s:v3 的 deployment 资源。

```yml
kubectl apply -f deployment.yaml -n dev
kubectl get pods -n dev
```

## ConfigMap

K8S 使用 ConfigMap 来将你的配置数据和应用程序代码分开，将非机密性的数据保存到键值对中。ConfigMap 在设计上不是用来保存大量数据的。在 ConfigMap 中保存的数据不可超过 1 MiB。如果你需要保存超出此尺寸限制的数据，你可能考虑挂载存储卷。

创建 hellok8s-config-dev.yaml 文件

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hellok8s-config
data:
  DB_URL: "http://DB_ADDRESS_DEV"
```

创建 hellok8s-config-test.yaml 文件

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hellok8s-config
data:
  DB_URL: "http://DB_ADDRESS_TEST"
```

分别在 dev test 两个 namespace 下创建相同的 ConfigMap，名字都叫 hellok8s-config，但是存放的 Pair 对中 Value 值不一样。

```bash
kubectl apply -f hellok8s-config-dev.yaml -n dev
# configmap/hellok8s-config created

kubectl apply -f hellok8s-config-test.yaml -n test 
# configmap/hellok8s-config created

kubectl get configmap --all-namespaces
NAMESPACE         NAME                                 DATA   AGE
dev               hellok8s-config                      1      3m12s
test              hellok8s-config                      1      2m1s
```

接着使用 POD 的方式来部署 hellok8s:v4，其中 env.name 表示的是将 configmap 中的值写进环境变量，这样代码从环境变量中获取 DB_URL，这个 KEY 名称必须保持一致。valueFrom 代表从哪里读取，configMapKeyRef 这里表示从名为 hellok8s-config 的 configMap 中读取 KEY=DB_URL 的 Value。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: hellok8s-pod
spec:
  containers:
    - name: hellok8s-container
      image: guangzhengli/hellok8s:v4
      env:
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: hellok8s-config
              key: DB_URL
```



## Secret

## Job



