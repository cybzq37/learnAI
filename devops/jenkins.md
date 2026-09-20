可以。下面给你一套从 **Git 提交 → Jenkins → 测试 → 构建 Docker 镜像 → 推送 Harbor → Kubernetes 滚动发布 → 发布验证** 的完整方案。

我先给你一套“容易落地、后续也能演进到生产”的基线架构。这里不依赖某个具体业务技术栈，Java / Node.js / Go 都能套。

> 说明：下面的镜像仓库、域名、命名空间都是示例，你替换成自己的即可。Jenkins 官方当前文档仍支持 Docker Pipeline、Kubernetes Agent、Credentials 等方案；Kubernetes 官方也建议通过专用 ServiceAccount/RBAC 做细粒度授权。([Jenkins][1])

# 一、最终架构

```text
                    ┌──────────────┐
                    │ GitLab/GitHub│
                    │      Git     │
                    └──────┬───────┘
                           │ push
                           │ Webhook
                           ▼
                  ┌───────────────────┐
                  │      Jenkins      │
                  │   CI/CD Pipeline  │
                  └────────┬──────────┘
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
          单元测试       Docker Build   kubectl
                           │             │
                           ▼             │
                    ┌──────────────┐     │
                    │    Harbor    │     │
                    │ Docker Image │     │
                    └──────┬───────┘     │
                           │ pull         │ deploy
                           ▼              ▼
                    ┌────────────────────────┐
                    │      Kubernetes        │
                    │                        │
                    │ Deployment              │
                    │   └── Pod               │
                    │   └── Pod               │
                    │   └── Pod               │
                    └────────────────────────┘
```

生产环境我建议把权限拆成：

```text
Jenkins
 ├─ Git：只读
 ├─ Harbor：只允许 push
 └─ Kubernetes：只允许指定 namespace 的 deployment 更新
```

不要给 Jenkins 一个 `cluster-admin` 的 kubeconfig。Kubernetes 官方明确建议给特定 ServiceAccount 授予所需范围的 Role，而不是宽泛的集群权限。([Kubernetes][2])

---

# 二、准备环境

假设：

```text
Git:
git.example.com

Jenkins:
jenkins.example.com

Harbor:
harbor.example.com

Kubernetes:
k8s-prod

K8s namespace:
demo-prod

Harbor project:
demo

镜像：
harbor.example.com/demo/demo-api
```

最终镜像类似：

```text
harbor.example.com/demo/demo-api:8b7e6c2
```

这里强烈建议用 **Git Commit SHA / Git Tag 作为镜像 tag**，不要只用 `latest`。

---

# 三、Harbor 配置

## 1. 创建 Project

进入 Harbor：

```text
Projects
  └── New Project
```

创建：

```text
demo
```

生产环境建议：

```text
Project: demo
Access Level: Private
```

Harbor 镜像标准格式就是：

```text
<harbor>/<project>/<repository>:<tag>
```

例如：

```text
harbor.example.com/demo/demo-api:8b7e6c2
```

Harbor 官方文档也是这个结构。([Harbor][3])

---

# 四、Harbor 创建 Robot Account

不要把 Harbor 管理员账号直接放进 Jenkins。

建议：

```text
Project
  demo
    └── Robot Accounts
```

创建：

```text
robot-ci
```

权限至少：

```text
Repository:
  Pull
  Push
```

Harbor 官方文档支持项目级 Robot Account 用于自动化流程，Docker 登录时使用 Robot 用户名和密码。([Harbor][4])

例如最终：

```text
Username:
robot$demo+ci

Password:
xxxxxxxxxxxxxxxx
```

---

# 五、Jenkins 安装

Jenkins 可以直接跑在 Linux VM 上。

如果是生产环境，我比较推荐：

```text
Jenkins Controller
        +
独立 Jenkins Agent
```

而不是让 Controller 自己执行 Docker Build。

Jenkins 官方 Docker 安装文档当前仍提供官方 `jenkins/jenkins` 镜像方案。([Jenkins][5])

例如：

```bash
docker volume create jenkins_home

docker run -d \
  --name jenkins \
  --restart=unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk21
```

实际部署时建议使用固定的 LTS 版本，而不是长期跟着 `latest` 漂移。

---

# 六、Jenkins 安装插件

至少安装：

```text
Pipeline
Git
GitLab / GitHub 相关插件
Credentials Binding
Docker Pipeline
Kubernetes
Pipeline: Stage View
```

其中：

### Docker Pipeline

让 Jenkins Pipeline 能调用 Docker。

Jenkins 官方支持：

```groovy
docker.build(...)
docker.withRegistry(...)
docker.image(...).push(...)
```

并支持使用 Jenkins Credentials 访问私有 Registry。([Jenkins][1])

### Kubernetes Plugin

以后你可以进一步把 Jenkins Agent 动态运行到 Kubernetes Pod 里。官方 Kubernetes Plugin 支持 `podTemplate` / Kubernetes Agent。([Jenkins][6])

---

# 七、Jenkins 凭据配置

进入：

```text
Jenkins
 → Manage Jenkins
 → Credentials
 → Global
```

创建几个凭据。

## 1. Git 凭据

例如：

```text
ID:
git-readonly

Kind:
Username with password
```

如果是 GitLab/GitHub，也可以使用 PAT。

---

## 2. Harbor 凭据

```text
ID:
harbor-robot

Kind:
Username with password
```

填写：

```text
Username:
robot$demo+ci

Password:
xxxxxxxx
```

Jenkins 官方建议通过 Credentials 存储外部系统认证信息，并在 Pipeline 中只通过 credential ID 引用，而不是把密码硬编码进 Jenkinsfile。([Jenkins][7])

---

## 3. Kubernetes 凭据

最简单的方式：

```text
ID:
k8s-prod-kubeconfig

Kind:
Secret file
```

上传：

```text
kubeconfig
```

这个 kubeconfig **必须是专门给 Jenkins CI/CD 用的账号**。

不要直接上传：

```text
~/.kube/config
```

尤其不能把管理员的 kubeconfig 放进去。

---

# 八、Jenkins Agent

你的 Jenkins Agent 至少应该有：

```text
git
docker
kubectl
bash
```

检查：

```bash
git --version
docker --version
kubectl version --client
```

例如 Ubuntu：

```bash
apt-get update

apt-get install -y \
    git \
    curl \
    ca-certificates
```

安装 Docker：

```bash
curl -fsSL https://get.docker.com | sh
```

安装 kubectl 时使用 Kubernetes 官方对应版本安装方式。

然后把 Jenkins Agent 加到：

```text
Manage Jenkins
  → Nodes
  → New Node
```

比如：

```text
Name:
docker-k8s-agent

Label:
docker-k8s
```

Jenkinsfile：

```groovy
agent {
    label 'docker-k8s'
}
```

---

# 九、Kubernetes 部署结构

Git 仓库建议这样组织：

```text
demo-api/
├── Dockerfile
├── Jenkinsfile
├── src/
├── pom.xml
└── k8s/
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── image-pull-secret.yaml
```

更进一步，可以再拆成：

```text
k8s/
├── base/
└── overlays/
    ├── dev/
    ├── test/
    └── prod/
```

规模起来以后可以进一步用 Helm 或 Kustomize。

---

# 十、Kubernetes Namespace

创建：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-prod
```

保存：

```text
k8s/namespace.yaml
```

执行：

```bash
kubectl apply -f k8s/namespace.yaml
```

---

# 十一、让 Kubernetes 从 Harbor 拉镜像

因为 Harbor 是私有仓库，Kubernetes Pod 需要 registry credentials。

Kubernetes 官方的做法就是：

```yaml
imagePullSecrets:
  - name: harbor-regcred
```

而这个 Secret 必须存在于 **Pod 所在的同一个 namespace**。([Kubernetes][8])

创建：

```bash
kubectl create secret docker-registry harbor-regcred \
  --docker-server=harbor.example.com \
  --docker-username='robot$demo+ci' \
  --docker-password='xxxxxxxx' \
  -n demo-prod
```

检查：

```bash
kubectl get secret harbor-regcred -n demo-prod
```

---

# 十二、Deployment

这是一个比较标准的：

```text
3 replicas
RollingUpdate
readinessProbe
livenessProbe
resources
imagePullSecrets
```

`k8s/deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: demo-api
  namespace: demo-prod

spec:
  replicas: 3

  revisionHistoryLimit: 5

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1

  selector:
    matchLabels:
      app: demo-api

  template:
    metadata:
      labels:
        app: demo-api

    spec:
      imagePullSecrets:
        - name: harbor-regcred

      containers:
        - name: demo-api

          image: harbor.example.com/demo/demo-api:latest

          imagePullPolicy: IfNotPresent

          ports:
            - containerPort: 8080

          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"

            limits:
              cpu: "1000m"
              memory: "1Gi"

          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080

            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 6

          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080

            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 5

          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - sleep 10
```

Kubernetes Deployment 默认支持 RollingUpdate，可以通过 `maxUnavailable` 和 `maxSurge` 控制滚动更新过程。([Kubernetes][9])

---

# 十三、Service

`k8s/service.yaml`：

```yaml
apiVersion: v1
kind: Service

metadata:
  name: demo-api
  namespace: demo-prod

spec:
  selector:
    app: demo-api

  ports:
    - port: 80
      targetPort: 8080

  type: ClusterIP
```

如果后面使用 Nginx Ingress / Gateway：

```text
Internet
   ↓
Ingress
   ↓
Service
   ↓
Pod
```

---

# 十四、Dockerfile

假设你的项目是 Spring Boot：

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY target/demo-api.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

完整流程就是：

```text
mvn package
    ↓
target/demo-api.jar
    ↓
docker build
    ↓
harbor.example.com/demo/demo-api:<git-sha>
```

---

# 十五、最关键的 Jenkinsfile

这是整套方案的核心。

```groovy
pipeline {

    agent {
        label 'docker-k8s'
    }

    environment {

        HARBOR_REGISTRY = 'harbor.example.com'
        HARBOR_PROJECT  = 'demo'
        IMAGE_NAME      = 'demo-api'

        K8S_NAMESPACE   = 'demo-prod'
        K8S_DEPLOYMENT  = 'demo-api'
        K8S_CONTAINER   = 'demo-api'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Get Version') {
            steps {
                script {
                    env.GIT_SHA = sh(
                        script: "git rev-parse --short=7 HEAD",
                        returnStdout: true
                    ).trim()

                    env.IMAGE = "${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${GIT_SHA}"

                    echo "IMAGE=${env.IMAGE}"
                }
            }
        }

        stage('Test') {
            steps {
                sh '''
                    chmod +x mvnw
                    ./mvnw test
                '''
            }
        }

        stage('Build Application') {
            steps {
                sh '''
                    ./mvnw clean package -DskipTests
                '''
            }
        }

        stage('Docker Build') {
            steps {

                sh """
                    docker build \
                      -t ${IMAGE} \
                      .
                """
            }
        }

        stage('Docker Push') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-robot',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASSWORD'
                    )
                ]) {

                    sh '''
                        set +x

                        echo "$HARBOR_PASSWORD" | \
                          docker login "$HARBOR_REGISTRY" \
                          -u "$HARBOR_USER" \
                          --password-stdin

                        docker push "$IMAGE"

                        docker logout "$HARBOR_REGISTRY"
                    '''
                }
            }
        }

        stage('Deploy Kubernetes') {
            steps {

                withCredentials([
                    file(
                        credentialsId: 'k8s-prod-kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl apply \
                          -f k8s/deployment.yaml \
                          -n "$K8S_NAMESPACE"

                        kubectl apply \
                          -f k8s/service.yaml \
                          -n "$K8S_NAMESPACE"

                        kubectl set image \
                          deployment/$K8S_DEPLOYMENT \
                          $K8S_CONTAINER=$IMAGE \
                          -n "$K8S_NAMESPACE"
                    '''
                }
            }
        }

        stage('Rollout Check') {
            steps {

                withCredentials([
                    file(
                        credentialsId: 'k8s-prod-kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl rollout status \
                          deployment/$K8S_DEPLOYMENT \
                          -n "$K8S_NAMESPACE" \
                          --timeout=5m
                    '''
                }
            }
        }
    }

    post {

        success {
            echo "Deploy success: ${IMAGE}"
        }

        failure {
            echo "Deploy failed"
        }

        always {
            cleanWs()
        }
    }
}
```

这个 Jenkinsfile 做了六件事：

```text
1. Checkout
2. Test
3. Maven Build
4. Docker Build
5. Push Harbor
6. kubectl 更新 Deployment
7. 等待 Rollout 完成
```

Jenkins 官方 Docker Pipeline 支持构建 Docker 镜像并推送到自定义 Registry。([Jenkins][1])

---

# 十六、一次完整 CI/CD 运行过程

比如开发：

```bash
git commit -m "fix api"
git push
```

Webhook 触发 Jenkins：

```text
Git Push
   ↓
Jenkins
   ↓
Checkout
   ↓
mvn test
   ↓
mvn package
   ↓
docker build
   ↓
docker tag
   ↓
docker push harbor
   ↓
kubectl set image
   ↓
Deployment RollingUpdate
   ↓
新 Pod 拉 Harbor 镜像
   ↓
readinessProbe 成功
   ↓
旧 Pod 逐渐退出
   ↓
发布完成
```

最终 Harbor：

```text
harbor.example.com
└── demo
    └── demo-api
        ├── 8b7e6c2
        ├── 91e8f10
        └── 2d3a411
```

Kubernetes：

```bash
kubectl get pods -n demo-prod
```

比如：

```text
NAME                         READY   STATUS
demo-api-7b9d8c6d8f-x1abc    1/1     Running
demo-api-7b9d8c6d8f-x2def    1/1     Running
demo-api-7b9d8c6d8f-x3ghi    1/1     Running
```

---

# 十七、为什么一定要用 Git SHA 做镜像 tag

不要这样：

```text
demo-api:latest
```

更建议：

```text
demo-api:8b7e6c2
```

甚至：

```text
demo-api:20260919-8b7e6c2
```

因为这样才能做到：

```text
Git Commit
     ↓
Docker Image
     ↓
Harbor
     ↓
K8s Deployment
```

一一对应。

例如：

```text
Commit:
8b7e6c2

Image:
harbor.example.com/demo/demo-api:8b7e6c2

Deployment:
demo-api:8b7e6c2
```

发生线上问题的时候可以快速判断：

```bash
kubectl describe deployment demo-api -n demo-prod
```

具体跑的是什么版本。

---

# 十八、回滚怎么做

这是 Jenkins + K8s 很大的优势。

```bash
kubectl rollout history deployment/demo-api \
  -n demo-prod
```

查看：

```bash
kubectl rollout history deployment/demo-api -n demo-prod
```

回滚：

```bash
kubectl rollout undo deployment/demo-api \
  -n demo-prod
```

然后：

```bash
kubectl rollout status deployment/demo-api \
  -n demo-prod
```

也可以更精确地直接指定上一版本镜像：

```bash
kubectl set image \
  deployment/demo-api \
  demo-api=harbor.example.com/demo/demo-api:91e8f10 \
  -n demo-prod
```

---

# 十九、Jenkins Kubernetes 权限

如果 Jenkins 在 Kubernetes 内跑 Agent，我更推荐：

```text
Jenkins Agent Pod
      │
      │ ServiceAccount
      ▼
Kubernetes API
      │
      ▼
Role
      │
      ▼
demo-prod namespace
```

例如：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-cd
  namespace: demo-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-cd
  namespace: demo-prod
rules:

  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs:
      - get
      - list
      - watch
      - update
      - patch

  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs:
      - get
      - list
      - watch

  - apiGroups: [""]
    resources: ["pods"]
    verbs:
      - get
      - list
      - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-cd
  namespace: demo-prod
subjects:
  - kind: ServiceAccount
    name: jenkins-cd
    namespace: demo-prod
roleRef:
  kind: Role
  name: jenkins-cd
  apiGroup: rbac.authorization.k8s.io
```

这种方式比：

```text
cluster-admin
```

安全得多。

Kubernetes 官方 RBAC 文档明确推荐针对具体 ServiceAccount 进行细粒度授权。([Kubernetes][2])

---

# 二十、一个容易踩坑的地方：Harbor HTTPS

生产环境一定建议：

```text
https://harbor.example.com
```

不要：

```text
http://harbor.example.com
```

如果 Harbor 使用自签名 CA，那么 Docker 客户端需要信任对应 CA；Harbor 官方文档也说明了 HTTPS/未知 CA 场景需要把 CA 配置到 Docker。([Harbor][3])

Kubernetes 节点/container runtime 同样要确保能正确访问 Harbor。

---

# 二十一、Docker Build 到底放在哪里

这里有三种常见模式。

## 方案 A：Jenkins Agent 直接 Docker Build

就是上面给你的：

```text
Jenkins Agent
    ↓
docker build
    ↓
docker push
```

优点：

```text
简单
容易理解
容易排查
```

缺点：

```text
Docker socket 权限比较敏感
```

---

## 方案 B：Docker-in-Docker

结构：

```text
Jenkins Agent
    ↓
Docker Client
    ↓
Docker-in-Docker
    ↓
Image
```

Jenkins 官方文档也给出了 Docker-in-Docker 的官方安装模式，但通常需要 `privileged`。([Jenkins][5])

适合：

```text
CI 环境
临时构建节点
测试环境
```

但要认真做隔离。

---

## 方案 C：BuildKit

生产环境如果你不想让 Jenkins Agent 直接操作宿主机 Docker Socket，可以考虑 BuildKit。

BuildKit 官方支持：

```text
Dockerfile
    ↓
buildctl
    ↓
Registry
```

并支持直接把构建结果推到 registry，例如：

```text
--output type=image,name=xxx,push=true
```

也支持 Kubernetes 和 rootless 模式。([GitHub][10])

---

# 二十二、关于 Kaniko

以前很多 Jenkins + Kubernetes 教程会用：

```text
kaniko
```

但现在需要注意：GoogleContainerTools 的原始 Kaniko 仓库已经于 **2025 年 6 月 3 日归档**，不再维护。([GitHub][11])

所以对于现在的新项目，我不会再把：

```text
gcr.io/kaniko-project/executor
```

作为首选生产方案。

如果你看到很多老教程：

```text
Jenkins
→ Kubernetes Agent
→ Kaniko
→ Harbor
```

它们技术上还能理解，但属于老方案。

现在更值得考虑：

```text
Jenkins
→ Kubernetes Agent
→ BuildKit / 其他受维护的镜像构建器
→ Harbor
```

BuildKit 官方也直接支持 Kubernetes 场景。([GitHub][10])

---

# 二十三、我建议你的生产目录这样设计

一个比较舒服的项目：

```text
demo-api/
│
├── src/
│
├── Dockerfile
├── .dockerignore
├── Jenkinsfile
│
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   │
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patch.yaml
│   │
│   └── prod/
│       ├── kustomization.yaml
│       └── patch.yaml
│
└── README.md
```

---

# 二十四、进一步升级成标准企业 CI/CD

你最终可以演进成：

```text
                   ┌─────────────┐
                   │ GitLab/GitHub│
                   └──────┬──────┘
                          │
                       Webhook
                          │
                          ▼
                   ┌─────────────┐
                   │   Jenkins   │
                   └──────┬──────┘
                          │
            ┌─────────────┼──────────────┐
            ▼             ▼              ▼
          Lint           Test          SAST
            │             │              │
            └─────────────┼──────────────┘
                          ▼
                    Build Image
                          │
                          ▼
                       Harbor
                          │
                  ┌───────┴────────┐
                  ▼                ▼
                 DEV              TEST
                  │                │
                  └───────┬────────┘
                          ▼
                        PROD
                          │
                          ▼
                     Kubernetes
                          │
                          ▼
                       Ingress
                          │
                          ▼
                        User
```

再进一步：

```text
Jenkins
   ↓
Build/Test/Scan
   ↓
Harbor
   ↓
GitOps Repo
   ↓
Argo CD
   ↓
Kubernetes
```

这种模式下 Jenkins 不需要直接执行：

```bash
kubectl apply
```

而是：

```text
Jenkins
  → 构建并推送镜像
  → 修改 GitOps 仓库 image tag

Argo CD
  → 检测 Git 变化
  → 自动同步 K8s
```

这是规模比较大以后很常见的演进方向。

---

# 二十五、整个系统里，哪些东西放哪里

| 内容             | 存放位置                                 |
| -------------- | ------------------------------------ |
| 源代码            | Git                                  |
| Dockerfile     | Git                                  |
| Jenkinsfile    | Git                                  |
| K8s Deployment | Git                                  |
| Docker 镜像      | Harbor                               |
| Harbor 推送密码    | Jenkins Credentials                  |
| K8s 访问凭据       | Jenkins Credentials / ServiceAccount |
| 线上 Deployment  | Kubernetes                           |
| Jenkins 数据     | Jenkins Persistent Volume            |
| 应用运行配置         | K8s ConfigMap / Secret               |
| 敏感配置           | K8s Secret / 外部 Secret Manager       |

---

# 二十六、最小可用版本

假设你现在就要搭，我建议第一阶段先做：

```text
GitLab
   ↓
Webhook
   ↓
Jenkins
   ↓
Maven Test
   ↓
Docker Build
   ↓
Harbor
   ↓
kubectl set image
   ↓
K8s
```

先不要一次把：

```text
Helm
Argo CD
SonarQube
Trivy
Vault
Prometheus
Grafana
```

全部塞进去。

先把这条链打通。

等它跑通，再加：

```text
        ┌── SonarQube
        ├── Trivy
        ├── Harbor Scan
        └── Unit Test
              ↓
            Harbor
              ↓
          K8s/ArgoCD
```

---

## 你最终需要准备的东西

```text
[1] GitLab/GitHub
    └── project

[2] Jenkins
    ├── Pipeline
    ├── Git
    ├── Credentials
    ├── Docker
    └── Kubernetes

[3] Harbor
    ├── Project: demo
    └── Robot: robot$demo+ci

[4] Kubernetes
    ├── namespace: demo-prod
    ├── harbor-regcred
    ├── deployment
    └── service

[5] Jenkins Credentials
    ├── git-readonly
    ├── harbor-robot
    └── k8s-prod-kubeconfig

[6] Git Repository
    ├── Dockerfile
    ├── Jenkinsfile
    └── k8s/
```

这样就已经形成一套完整的、可以实际使用的 CI/CD 闭环。

参考的 Jenkins / Harbor / Kubernetes 官方文档分别覆盖了 Docker Pipeline、Kubernetes Agent、Credentials、Harbor 镜像 push、私有 Registry 拉取，以及 Deployment RollingUpdate/RBAC 等机制。([Jenkins][1])

如果你是准备**真实搭一套环境**，我建议下一步直接按你现有环境定制：例如 **“Jenkins + Harbor + K8s 都是 Docker/VM 部署，GitLab 是 xxx，应用是 Spring Boot”**，我可以直接给你整理成一套可复制执行的项目，包括 `docker-compose.yml`、Jenkinsfile、Harbor 配置、K8s RBAC、Deployment/Service、Webhook 和从 0 到发布成功的命令。

[1]: https://www.jenkins.io/doc/book/pipeline/docker/?utm_source=chatgpt.com "Using Docker with Pipeline"
[2]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/?utm_source=chatgpt.com "Using RBAC Authorization | Kubernetes"
[3]: https://goharbor.io/docs/main/working-with-projects/working-with-images/pulling-pushing-images/?utm_source=chatgpt.com "Harbor docs | Pulling and Pushing Images in the Docker Client"
[4]: https://goharbor.io/docs/main/working-with-projects/project-configuration/create-robot-accounts/?utm_source=chatgpt.com "Harbor docs | Create Project Robot Accounts"
[5]: https://www.jenkins.io/doc/book/installing/docker/?utm_source=chatgpt.com "Docker"
[6]: https://www.jenkins.io/doc/pipeline/steps/kubernetes/?utm_source=chatgpt.com "Kubernetes plugin"
[7]: https://www.jenkins.io/doc/book/using/using-credentials/?utm_source=chatgpt.com "Using credentials"
[8]: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/?utm_source=chatgpt.com "Pull an Image from a Private Registry | Kubernetes"
[9]: https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com "Deployments | Kubernetes"
[10]: https://github.com/moby/buildkit?utm_source=chatgpt.com "GitHub - moby/buildkit: concurrent, cache-efficient, and Dockerfile-agnostic builder toolkit · GitHub"
[11]: https://github.com/GoogleContainerTools/kaniko?utm_source=chatgpt.com "GitHub - GoogleContainerTools/kaniko: Build Container Images In Kubernetes · GitHub"
