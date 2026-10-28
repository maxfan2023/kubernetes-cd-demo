# Kubernetes Continuous Deployment Demo

Kubernetes 第二天作业。Task 1–4 已完成应用、Kubernetes 资源和 Jenkins 配置；Task 5 采用 GitHub SCM Job 发布 v1 并更新到 v2。当前源码页面为 v2，正式构建、部署与 HTTP 验证结果见 [Task 5 报告](docs/task5-deploy-update.md)。Task 6 回滚仍待执行。

## Task 1 文件

```text
app/
├── index.html
└── Dockerfile
```

- `app/index.html`：显示作业标题和 `Application version: v2`；Task 1 的初始版本为 v1，Task 5 将其更新为 v2。
- `app/Dockerfile`：使用作业指定的 `nginx:1.25-alpine`，将页面复制到 Nginx 默认网站目录，声明容器端口 80。
- 根目录已有的 `main.py` 是 PyCharm 示例，不参与镜像构建。

## 本地构建和验证

先启动 Docker Desktop，然后在仓库根目录执行：

```bash
cd /Users/max/PycharmProjects/PythonProject/kubernetes-cd-demo
docker build -t kubernetes-cd-demo:v2 ./app
docker run --rm -d --name kubernetes-cd-demo \
  -p 127.0.0.1:18080:80 kubernetes-cd-demo:v2
curl -fsS http://127.0.0.1:18080/
```

在浏览器打开 <http://127.0.0.1:18080/>，应看到：

```text
Kubernetes Continuous Deployment Demo
Application version: v2
```

`EXPOSE 80` 声明镜像提供服务的端口；`-p 127.0.0.1:18080:80` 将本机 18080 端口映射到容器 80 端口。

验证结束后停止演示容器：

```bash
docker stop kubernetes-cd-demo
```

初始部署页面为 v1；当前页面已在 Task 5 更新为 v2。本地镜像标签 `:v2` 仅用于本节手动构建示例，Jenkins 发布使用对应 Git 提交号前 12 位作为镜像标签。

## Task 2 Kubernetes 配置

本节记录 Task 2 配置和复现命令；初始检查见 [Task 2 验证记录](docs/task2-validation.md)，正式 Jenkins 构建及 v1、v2 部署证据见 [Task 5 报告](docs/task5-deploy-update.md)。

```text
k8s/
├── namespace.yaml
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

本次沿用第一天作业已安装 Nginx Ingress Controller 的 `kind-k8s-chapter1` 集群。下面的命令显式指定 context，不会切换 kubectl 的全局默认 context。

| 配置项 | 值 |
| --- | --- |
| Context | `kind-k8s-chapter1` |
| Namespace | `training` |
| Deployment | `cd-demo-deployment` |
| 容器 | `cd-demo` |
| Pod 标签及 Service selector | `app: cd-demo` |
| 副本数 | `3` |
| Service | `cd-demo-service`，ClusterIP，端口 80 |
| Ingress | `cd-demo-ingress` |
| IngressClass | `nginx` |
| Ingress host | `cd-demo.local` |
| 镜像拉取 Secret | `cd-demo-acr-pull` |

Deployment 保留第一天的滚动更新、命名端口 `http` 和 HTTP readiness/liveness probes。独立的 `app: cd-demo` 标签避免 Service 选中第一天的 Nginx 练习 Pods。Deployment 和容器名称与后续作业的 Jenkinsfile、更新及回滚命令保持一致。

### 阿里云镜像与私有仓库凭据

源 Deployment YAML 保留以下初始 v1 镜像，平台为 `linux/arm64`。Task 5 的实际运行镜像使用 Git 提交号标签，见文末版本对应表：

```text
crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com/maxfan/kubernetes-cd-demo:v1
```

这是私有仓库。Deployment 通过 `imagePullSecrets` 引用 `training` Namespace 下的 `cd-demo-acr-pull` Secret。部署到新的集群或 Namespace 前，必须单独、安全地配置这个镜像拉取凭据；Secret 内容、Registry 密码和 kubeconfig 不应提交到 Git。

检查 Secret 是否已存在时，只查看名称和类型：

```bash
kubectl --context=kind-k8s-chapter1 -n training get secret cd-demo-acr-pull
```

本机 Docker 登录、Jenkins 的镜像推送凭据和 Kubernetes 的镜像拉取凭据分别作用于不同客户端。后续 Jenkins 中的 `container-registry-credentials` 用于构建节点登录并推送镜像，不能代替上述 Kubernetes Secret。

Registry 域名和镜像仓库完整路径也需要区分：

```text
REGISTRY_HOST=crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com
IMAGE_REPOSITORY=crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com/maxfan/kubernetes-cd-demo
```

### 应用和检查资源

以下命令手动复现 Task 2 的初始 v1 部署。直接应用源 `k8s/deployment.yaml` 会将运行镜像改为 `:v1`；完成 Task 5 后如需重新发布当前源码，应运行正式 Jenkins Job。只检查当前 v2 运行状态时，请跳过两个 `apply -f k8s/` 命令。

在仓库根目录执行。先创建或更新 Namespace，并确认镜像拉取 Secret 已配置，再进行服务端预检查和实际部署：

```bash
cd /Users/max/PycharmProjects/PythonProject/kubernetes-cd-demo

kubectl --context=kind-k8s-chapter1 apply -f k8s/namespace.yaml
kubectl --context=kind-k8s-chapter1 -n training get secret cd-demo-acr-pull

kubectl --context=kind-k8s-chapter1 apply --dry-run=server -f k8s/
kubectl --context=kind-k8s-chapter1 apply -f k8s/

kubectl --context=kind-k8s-chapter1 -n training rollout status \
  deployment/cd-demo-deployment --timeout=180s

kubectl --context=kind-k8s-chapter1 -n training get \
  deployments,pods,services,ingresses -l app=cd-demo -o wide

kubectl --context=kind-k8s-chapter1 -n training get endpointslices \
  -l kubernetes.io/service-name=cd-demo-service
```

预检查不会部署 Pods，也不能证明镜像能够拉取。实际验证应确认 Deployment 达到 `3/3` Ready、Service 的 EndpointSlices 包含就绪的应用 Pods，并通过 HTTP 检查页面版本。

### 通过 Service 验证页面

以下命令启动临时端口转发，需要保持该终端运行：

```bash
kubectl --context=kind-k8s-chapter1 -n training port-forward \
  --address 127.0.0.1 service/cd-demo-service 18081:80
```

在另一个终端执行，或在浏览器打开 <http://127.0.0.1:18081/>：

```bash
curl -fsS http://127.0.0.1:18081/
```

Task 5 的 v2 发布完成后，预期页面包含 `Application version: v2`；如果刚手动应用了本节初始 YAML，则页面为 v1。验证结束后，在运行端口转发的终端按 `Ctrl+C` 停止转发。这个命令用于手动复现，不表示端口转发会一直运行。

### 通过 Ingress 验证路由

Ingress 需要目标集群中实际运行的 Controller，以及到该 Controller 的可达入口。仅创建 Ingress YAML 不会启动 Controller，`cd-demo.local` 也不会自动产生 DNS 记录。

可以临时转发已有 Controller 的 HTTP Service，并用 Host 请求头匹配本次域名，无需修改本机 hosts 文件：

```bash
kubectl --context=kind-k8s-chapter1 -n ingress-nginx port-forward \
  --address 127.0.0.1 service/ingress-nginx-controller 18082:80
```

保持端口转发运行，在另一个终端执行：

```bash
curl -fsS -H 'Host: cd-demo.local' http://127.0.0.1:18082/
```

Task 5 的 v2 发布完成后，预期仍返回 v2 页面；页面版本应与上面的 Service 请求一致。这条路径经过 Ingress Controller 和 `cd-demo-service`，区别于直接访问 Docker 演示容器的 `18080` 端口。验证完成后用 `Ctrl+C` 停止端口转发。

Task 3 的 Jenkinsfile 已处理镜像提交号标签与本节初始 `:v1` 标签的衔接。Task 5 的正式 SCM 发布和版本更新证据见文末；Task 6 回滚仍待执行。

## Task 3 Jenkinsfile

根目录的 `Jenkinsfile` 使用本机 Jenkins 的 `built-in` 节点，包含五个阶段：

| 阶段 | 作用 |
| --- | --- |
| Checkout | 从 SCM 检出代码，以 Git 提交号前 12 位作为镜像标签。 |
| Validate | 检查页面、Dockerfile、四份 YAML 和 Git、Docker、Buildx、kubectl。 |
| Build and Push Image | 构建 `linux/arm64` 镜像，使用 Jenkins 凭据登录阿里云并推送。 |
| Deploy to Kubernetes | 使用指定 kubeconfig 和 context 应用资源，等待 rollout 完成。 |
| Verify | 核对实际镜像，并输出应用资源、EndpointSlices 和 rollout history。 |

流水线补充 Homebrew 和 Docker 工具路径，启用时间戳，禁止同一 Job 并发构建，并设置 20 分钟超时。

构建时禁用 provenance/SBOM 附加证明，推送时显式指定 `linux/arm64`，以兼容前面验证过的阿里云仓库和本地 ARM64 集群。Registry 登录使用临时 Docker 配置，结束时自动清理。

源 YAML 保留初始 `:v1` 镜像。流水线先用 `kubectl set image --local` 生成包含最终提交号镜像的临时 Deployment，再应用一次，避免先部署 `:v1` 再更新镜像产生额外修订，影响 Task 6 的回滚目标。

## Task 4 Jenkins 配置

已创建本机 Job：[kubernetes-cd-demo](http://localhost:8080/job/kubernetes-cd-demo/)。

| 配置项 | 值 |
| --- | --- |
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | `https://github.com/maxfan2023/kubernetes-cd-demo.git` |
| Git 凭据 | `github-token` |
| Branch Specifier | `*/main` |
| Script Path | `Jenkinsfile` |
| Lightweight checkout | 开启 |

已配置作业要求的两个凭据 ID：

| Credential ID | 类型 | 用途 |
| --- | --- | --- |
| `container-registry-credentials` | Username with password | 登录并推送阿里云镜像。 |
| `kubeconfig-training` | Secret file | 访问 `kind-k8s-chapter1` 集群。 |

Registry 凭据在 Jenkins 内复用已有配置；kubeconfig 从已有凭据中仅保留目标 context，继承原身份的集群权限。原有凭据保留。密码、Token 和 kubeconfig 内容均不应提交到 Git。

### Task 3、Task 4 配置阶段的历史检查

- Jenkins Declarative 语法验证通过。
- 五段 shell 脚本的 POSIX 语法检查通过。
- 实际 Buildx 构建成功，镜像平台确认为 `linux/arm64`；临时验证标签已清理。
- 实际镜像替换渲染和 Kubernetes server dry-run 通过。
- Jenkins 进程使用新 Registry 凭据实际登录阿里云成功，临时 Docker 配置已清理。
- Jenkins 进程使用新 kubeconfig 实际读取 `training` Namespace 成功，并确认具有修改该 Namespace 中 Deployment 的权限。
- 已核对保存后的 Job 分支、脚本路径和轻量检出配置；当时正式 Job 的构建次数为 0，后续正式发布记录见 Task 5。
- 使用模拟命令验证了推送成功/失败时的凭据清理、Docker endpoint 保留、Deployment 只应用一次、缺少拉取 Secret 时提前失败，以及镜像不匹配时验证失败。

模拟检查使用假凭据；另行完成的真实凭据检查验证了登录和集群访问。这些配置阶段检查当时没有覆盖真实镜像推送或 Jenkins 完整发布，后续实跑见下文。英文过程与验证边界见 [Task 3–4 report](docs/task3-task4-report.md)。

### Task 5 前置条件的后续处理

Task 3、Task 4 配置阶段结束时，GitHub 仓库尚无提交，正式 SCM Job 尚未执行发布。随后先使用独立的 Pipeline script Job 验证完整流水线；Task 5 再按用户选择提交并推送项目，由正式 Job 从 `main` 加载 `Jenkinsfile`。

`training/cd-demo-acr-pull` 现已按用户后续的明确授权，使用 Jenkins 中的 Registry 凭据配置完成。Kubernetes 通过这个独立 Secret 拉取私有镜像；凭据内容没有进入 Git。

Task 5 的构建日志、v1/v2 页面与镜像、滚动更新结果单独保存在 [Task 5 报告](docs/task5-deploy-update.md) 中。Task 6 回滚结果尚未收集。

## Task 3 Pipeline script 历史实跑（当时不上传 GitHub）

独立测试 Job：[kubernetes-cd-demo-script-test](http://localhost:8080/job/kubernetes-cd-demo-script-test/)，构建 [#1](http://localhost:8080/job/kubernetes-cd-demo-script-test/1/) 结果为 **SUCCESS**。

- 使用根目录的 `Jenkinsfile.script-test`，Definition 为 **Pipeline script**。
- 仅替换 Checkout：复制本地应用文件，在测试 Job 工作区创建快照提交，不修改原仓库的 Git 历史，也不上传 GitHub。其余流水线阶段与正式 Jenkinsfile 相同。
- 实际完成检查、ARM64 镜像构建、阿里云推送、Kubernetes 部署和 Verify。
- 运行镜像标签为 `4490e5590699`，Deployment 达到 `3/3 Ready`，Service 有三个就绪端点。
- 通过 Service 和 Ingress 的 HTTP 请求均返回与本地文件完全一致的 v1 页面。

详见 [英文实跑报告](docs/task3-script-test.md)、[成功构建日志](docs/task3-script-test-build-1.log) 和 [运行验证记录](docs/task3-script-test-runtime.txt)。该历史运行只覆盖本地 Pipeline script 和 v1；当时未覆盖的 GitHub SCM 检出及 v2 更新由 Task 5 正式 Job 验证，回滚仍属于待执行的 Task 6。

## Task 5 发布与更新

正式 Job 使用 **Pipeline script from SCM**，从 [GitHub 仓库](https://github.com/maxfan2023/kubernetes-cd-demo) 的 `main` 分支检出代码。先发布 v1，再将 `app/index.html` 中的版本更新为 v2，提交并推送后运行第二次构建。

| 页面版本 | Git 提交 | 正式 Jenkins 构建 | Deployment revision | 验证结果 |
| --- | --- | --- | --- | --- |
| v1 | `1ff38fb17176` | [#1](http://localhost:8080/job/kubernetes-cd-demo/1/) | `4` | `SUCCESS；3/3 Ready；HTTP 200` |
| v2 | `64f2a5d31ea5` | [#2](http://localhost:8080/job/kubernetes-cd-demo/2/) | `5` | `SUCCESS；3/3 Ready；HTTP 200` |

镜像仓库沿用上面的阿里云地址，镜像标签分别为对应 Git 提交号的前 12 位。页面内容中的 v1/v2 与镜像标签分开记录。

详细构建日志、Kubernetes rollout、Pod/ReplicaSet 状态和 Service/Ingress HTTP 响应见 [Task 5 发布与更新报告](docs/task5-deploy-update.md)。当前保留 v2 供后续 Task 6 回滚演示。

检查当前部署：

```bash
kubectl --context=kind-k8s-chapter1 -n training rollout status \
  deployment/cd-demo-deployment --timeout=180s
kubectl --context=kind-k8s-chapter1 -n training get pods -l app=cd-demo
kubectl --context=kind-k8s-chapter1 -n training get replicasets -l app=cd-demo
kubectl --context=kind-k8s-chapter1 -n training rollout history \
  deployment/cd-demo-deployment
```

Task 6 尚未执行 `rollout undo`；不能将 Task 5 的 rollout history 当作回滚成功证据。
