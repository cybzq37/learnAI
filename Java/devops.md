DevOps 开发并不是指某一种特定的编程语言或框架，而是一套**文化、实践和工具链**的集合，目的是打通开发（Dev）和运维（Ops）之间的壁垒，实现软件的快速、可靠、频繁交付。

如果从“一个 DevOps 开发者需要掌握和参与的东西”这个角度来拆解，它通常包含以下几个核心板块：

### 1. 持续集成与持续交付/部署（CI/CD）
这是 DevOps 最核心的工程实践，目的是让代码从提交到上线的过程自动化。

- **持续集成（CI）**：开发人员频繁合并代码到主干，每次合并都触发自动构建和自动化测试。
- **持续交付（CD）**：在 CI 基础上，确保代码随时可以发布到生产环境。
- **持续部署**：进一步自动化，通过测试后自动部署到生产环境。
- **涉及工具**：Jenkins、GitLab CI、GitHub Actions、CircleCI、ArgoCD、Spinnaker。

### 2. 基础设施即代码（IaC）
用代码来定义和管理服务器、网络、数据库等基础设施，而不是手动在控制台点选。这样环境可以版本化、可复现。

- **配置管理**：Ansible、Puppet、Chef、SaltStack。
- **基础设施编排**：Terraform、Pulumi、CloudFormation。
- **不可变基础设施**：通过 Packer 构建镜像，结合 Terraform 部署。

### 3. 容器化与编排
容器解决了“在我机器上能跑”的环境一致性问题，编排解决了大规模容器管理问题。

- **容器技术**：Docker、Podman、containerd。
- **编排平台**：Kubernetes（绝对主流）、Docker Swarm、Nomad。
- **相关生态**：Helm（包管理）、Istio/Linkerd（服务网格）、Kustomize。

### 4. 云平台与云原生
DevOps 通常深度依赖云服务，需要了解主流云厂商的计算、存储、网络、IAM 等核心服务。

- **主流云**：AWS、Azure、GCP、阿里云、腾讯云。
- **云原生理念**：微服务、声明式 API、弹性伸缩、可观测性。
- **Serverless**：AWS Lambda、阿里云函数计算等。

### 5. 监控、日志与可观测性
系统上线后，需要知道它是否健康、为什么出问题。可观测性通常分为三大支柱：

- **Metrics（指标）**：Prometheus、Grafana、CloudWatch。
- **Logging（日志）**：ELK/EFK Stack、Loki、Fluentd。
- **Tracing（链路追踪）**：Jaeger、Zipkin、OpenTelemetry。
- **告警与事件管理**：Alertmanager、PagerDuty、Opsgenie。

### 6. 安全（DevSecOps）
安全不再只是上线前的扫描，而是嵌入到整个 DevOps 流程中。

- **代码安全**：SAST（静态扫描）、依赖扫描（Snyk、Trivy）。
- **容器安全**：镜像扫描、运行时安全（Falco）。
- **密钥管理**：Vault、Sealed Secrets、云厂商 KMS。
- **合规与策略**：OPA/Gatekeeper、合规即代码。

### 7. 协作与文化
这是 DevOps 的“软”部分，但往往决定成败。

- **敏捷与精益**：Scrum、Kanban、价值流映射。
- **协作工具**：Jira、Confluence、Slack/Teams。
- **SRE 实践**：SLO/SLI、错误预算、事后复盘（Postmortem）。

### 8. 编程与脚本能力
DevOps 开发者通常需要写“胶水代码”来串联工具，或开发内部平台。

- **脚本语言**：Bash、Python（最常用）、Go。
- **配置语言**：YAML、HCL、JSON。
- **API 交互**：REST、gRPC。

---

### 一个典型的 DevOps 工作流示例
1. 开发用 **Git** 提交代码到 **GitLab**。
2. **GitLab CI** 触发流水线：跑单元测试、**SonarQube** 代码扫描、**Trivy** 镜像扫描。
3. 构建 **Docker** 镜像，推送到 **Harbor**。
4. **ArgoCD** 监听镜像更新，自动同步到 **Kubernetes** 集群。
5. **Terraform** 管理云上 RDS、VPC 等资源。
6. **Prometheus + Grafana** 监控应用指标，**Loki** 收集日志，异常时 **Alertmanager** 告警到 **钉钉/Slack**。

---

### 总结
DevOps 开发包含的东西可以概括为：**用自动化和代码化的方式，把软件从需求到上线再到运维的整个生命周期管起来，同时保证速度、稳定性和安全性。** 它横跨开发、测试、运维、安全多个领域，工具链非常庞大，实际工作中往往根据团队规模和业务需求选择其中的子集来落地。



把上一轮提到的每个板块落到“怎么做”的层面，可以按一条**从代码提交到生产运维的完整流水线**来理解。下面按实际执行顺序，拆解每一步的落地方法。

### 第一步：代码提交与分支策略

**怎么做：** 建立分支保护规则，确保代码变更必须经过审查。典型做法是：`main` 分支对应生产环境，要求至少一名维护者审批方可合并；功能开发在 `feature/*` 分支进行，修复在 `hotfix/*` 分支。

**关键配置示例（GitLab）：**
```yaml
protected_branches:
  - name: main
    push_access_level: maintainer
    merge_access_level: maintainer
    required_approvals: 1
```

### 第二步：持续集成（CI）——自动构建与测试

开发者推送到功能分支后，CI 工具自动触发。**核心是让每一次提交都经过“构建 + 测试 + 扫描”三道关。**

**怎么实现：** 在项目根目录放一个 CI 配置文件（如 `.gitlab-ci.yml` 或 `Jenkinsfile`），定义流水线阶段。

**以 GitLab CI 为例：**
```yaml
build-job:
  image: node:20-alpine
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
```
这里 `artifacts` 把构建产物保存下来，供后续阶段使用。

**质量门禁怎么加：** 在测试阶段之后嵌入 SonarQube 扫描，并设定质量阈值——例如“新增代码漏洞数必须为 0”“单元测试覆盖率不低于 80%”。不达标则流水线失败，阻止合并。

### 第三步：容器化与镜像构建

**怎么做：** CI 阶段中增加一个“构建镜像”的步骤。用 Dockerfile 把应用打包成容器镜像，打上 commit hash 标签，推送到镜像仓库。

**关键实践：** 在 CI 中构建 Docker 镜像时，**避免使用 Docker-in-Docker 的 `--privileged` 模式**（安全风险高）。推荐用 Kaniko 或 BuildKit，它们能在非特权容器中完成镜像构建。

**镜像扫描：** 推送镜像后，用 Trivy 等工具扫描镜像中的已知漏洞。这一步可以配置为“发现严重漏洞则阻断部署”。

### 第四步：持续交付/部署（CD）——把镜像送到目标环境

**两种主流模式：**

**推送模式（Push）：** CI 流水线直接连接 K8s 集群，执行 `helm upgrade` 或 `kubectl apply`。适合小型团队，控制直接。

**拉取模式（GitOps）：** 更推荐。用 ArgoCD 或 Flux 监听 Git 仓库中存储的 K8s 清单变更。当 CI 流水线更新了清单文件中的镜像版本，ArgoCD 自动将集群同步到期望状态。这种方式**消除了流水线对集群的直接访问需求**，减少了攻击面，还自带漂移检测和自修复能力。

### 第五步：基础设施即代码（IaC）

**怎么做：** 用 Terraform 或 Pulumi 定义云资源（VPC、数据库、K8s 集群等），代码存入 Git 仓库。

**分层管理：** 建议把 IaC 分成三层流水线：
- **基础层**：组织级资源（项目、日志、安全基线），通常只跑一次或极少变更
- **基础设施层**：业务部门级的网络、数据库等，由各团队独立管理
- **应用层**：每个工作负载的部署清单

**执行方式：** 在 CI 中运行 `terraform plan` 输出变更预览，人工审批后运行 `terraform apply`。状态文件（`terraform.tfstate`）存入远程后端（如 GCS/S3），启用版本控制，防止并发修改冲突。

### 第六步：可观测性——监控、日志与告警

**监控怎么做：** 部署 Node Exporter 采集服务器指标，在 Prometheus 配置文件中定义 `scrape_configs`，告诉 Prometheus 去哪里拉数据。指标存入 Prometheus 后，Grafana 配置 Prometheus 为数据源，用仪表盘可视化。

**日志怎么做：** 用 Fluentd/Fluent Bit 采集容器日志，发送到 Loki 或 Elasticsearch。Grafana 中配置 Loki 数据源后，可以和指标在同一个面板中关联查看。

**告警怎么做：** Prometheus 中定义告警规则（如“5 分钟内错误率 > 5%”），告警发送到 Alertmanager，由它去重、分组后路由到钉钉、Slack 或 PagerDuty。

### 第七步：安全嵌入（DevSecOps）

安全不是最后一步，而是嵌入上述每一步。**“向左移动”** 意味着在开发早期就把安全纳入。

**具体嵌入点：**
- **IDE 阶段**：开发者编码时，VS Code 的安全扩展插件实时提示漏洞
- **CI 阶段**：依赖扫描（Snyk）、静态代码分析（SAST）、镜像扫描
- **运行时**：K8s 网络策略限制 Pod 间流量；Key Vault 在运行时注入密钥，不暴露给开发者
- **密钥管理**：CI 变量中存储的密钥应使用专用密钥管理服务（Vault、云厂商 KMS），而非明文存在 CI 配置中

### 第八步：渐进式发布与流量管理

**金丝雀发布怎么做：** 在 Istio 中定义 `DestinationRule` 划分 v1/v2 子集，然后用 `VirtualService` 设置权重。先部署 v2 的 Deployment（1 个副本），通过请求头匹配（如 `canary: true`）让内部测试流量先打到 v2。验证通过后，逐步增加权重（10% → 30% → 50% → 100%）。

**配套配置：** 设置超时和重试策略（如 `timeout: 3s`、`retries.attempts: 2`），防止故障扩散。同时用 PodDisruptionBudget 保证滚动更新时至少保留一定数量的可用 Pod。

### 整体串联

把这些步骤串起来，一条典型的流水线是：

1. 开发者提交代码 → 触发 CI
2. CI 运行构建、测试、代码扫描 → 失败则阻断
3. 构建 Docker 镜像并推送 → 扫描镜像漏洞
4. 更新 K8s 清单中的镜像版本 → GitOps 自动同步到集群
5. 金丝雀发布验证 → 逐步放量
6. Prometheus + Grafana 持续监控 → 异常时 Alertmanager 告警