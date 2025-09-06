# Argo Workflow CI/CD 项目说明

本项目基于 Argo Workflows 实现一套完整的 CI/CD 流水线，用于自动化构建和部署 hbe-charge 应用。项目使用 GitLab Webhook 触发流水线，包含代码拉取、构建、部署和通知等完整流程。

## 项目结构

```
.
├── event-source-webhook.yaml           # Webhook 事件源配置
├── event-sensor-webhook.yaml           # 事件传感器配置
├── hbe-charge-pipeline.yaml            # 主工作流模板
├── hbe-charge-pull-template.yaml       # 代码拉取模板
├── hbe-charge-build-template.yaml      # 构建模板
├── hbe-charge-deploy-template.yaml     # 部署模板
├── hbe-charge-dingtalk-notify-template.yaml  # 钉钉通知模板
├── dingtalk-webhook-secret.yaml        # 钉钉 Webhook 密钥
├── deployment-ssh-keys.yaml            # 部署 SSH 密钥
├── gitlab-pull-secret.yaml             # GitLab 拉取代碼密钥
├── gitlab-webhook-secret.yaml          # GitLab Webhook 密钥
├── event-source-webhook-service.yaml   # Webhook 服务配置
├── eventbus.yaml                       # EventBus配置
└── pvc.yaml                            # 持久卷声明
env/
├── argo-workflow-ingress.yaml          # Ingress 配置
├── deployment-ssh-keys.yaml            # 部署 SSH 密钥
├── dingtalk-webhook-secret.yaml        # 钉钉 Webhook 密钥
├── event-source-webhook-service.yaml   # Webhook 服务配置
├── gitlab-access-secret.yaml           # GitLab 访问令牌
├── gitlab-pull-secret.yaml             # GitLab 拉取密钥
├── gitlab-webhook-secret.yaml          # GitLab Webhook 密钥
├── rbac.yaml                           # RBAC 权限配置
├── serviceaccount.yaml                 # ServiceAccount 配置
├── shared-data-pvc.yaml                # 共享数据 PVC 配置
└── stan-pv-0-pvc.yaml                  # EventBus PVC 配置
```

## 项目架构与数据流

本项目采用事件驱动的微服务架构，核心组件包括：

1. **Event Source (事件源)**: 监听 GitLab Webhook 事件
2. **EventBus (事件总线)**: 负责事件传输
3. **Event Sensor (事件传感器)**: 处理事件并触发工作流
4. **Workflow Templates (工作流模板)**: 执行具体的 CI/CD 任务

完整的数据流如下：
```
GitLab Push Event 
    → Webhook (event-source-webhook.yaml) 
    → EventBus (eventbus.yaml) 
    → Event Sensor (event-sensor-webhook.yaml) 
    → Workflow (hbe-charge-pipeline.yaml)
        ├── Pull Template (hbe-charge-pull-template.yaml)
        ├── Build Template (hbe-charge-build-template.yaml)
        ├── Deploy Template (hbe-charge-deploy-template.yaml)
        └── Notify Template (hbe-charge-dingtalk-notify-template.yaml)
```

### 关键数据字段传递

在事件处理过程中，以下关键数据字段会被提取和传递：

- **build-time**: 来自 GitLab webhook payload 中的 `commits[0].timestamp` 字段
- **BUILD_TIME_FORMATTED**: 在部署阶段，将 build-time 格式化为易读形式的变量
  - 格式化方式: `BUILD_TIME_FORMATTED=$(echo "$BUILD_TIME" | sed 's/T/ /g' | sed 's/+/_/g')`
  - 用途: 用于生成部署元数据和版本号

## 核心组件

### 1. Event Source (事件源)
文件: `event-source-webhook.yaml`

定义了 GitLab Webhook 事件源，监听以下端点：
- `/hbe-charge`: 用于触发 hbe-charge 项目的 CI/CD 流水线

配置详情：
- 监听 GitLab 项目: `lins/hbe-charge`
- 监听事件类型: PushEvents（推送事件）
- GitLab 基础 URL: `http://192.168.31.195`
- Webhook 回调 URL: `http://192.168.30.149:30080`

### 2. Event Sensor (事件传感器)
文件: `event-sensor-webhook.yaml`

监听来自 Event Source 的事件，并触发相应的工作流。主要功能包括：
- 接收 GitLab Webhook 事件
- 解析事件数据并提取关键参数
- 触发 hbe-charge CI/CD 流水线
- 提取的参数包括：
  - branch: 分支名 (来自 body.ref)
  - git-repo-url: Git 仓库地址 (来自 body.project.git_http_url)
  - commit-message: 提交信息 (来自 body.commits.0.message)
  - commit-id: 提交 ID (来自 body.commits.0.id)
  - project-name: 项目名称 (来自 body.project.name)
  - target-branch: 目标分支 (来自 body.ref)
  - build-user: 构建用户 (来自 body.user_username)
  - git-url: Git URL (来自 body.project.git_http_url)
  - git-commit: Git 提交哈希 (来自 body.commits.0.id)
  - build-time: 构建时间 (来自 GitLab webhook 中 commits[0].timestamp 字段)
  - commit-author: 提交作者 (来自 body.commits.0.author.name)

### 3. EventBus (事件总线)
文件: `eventbus.yaml`

负责事件消息的传输和路由，是 EventSource 和 Sensor 之间通信的枢纽。

EventBus 使用 NATS 作为消息传输后端，配置说明：
- replicas: NATS 集群副本数，设置为1以节省资源
- auth: 认证策略，使用 token 认证方式
- persistence: 持久化配置，确保事件数据不会丢失
  - storageClassName: 使用 nfs-client 存储类，与项目中的 PVC 保持一致
  - accessMode: ReadWriteOnce 访问模式
  - volumeSize: 1Gi 存储空间

### 4. Workflow Templates (工作流模板)

#### 主工作流模板
文件: `hbe-charge-pipeline.yaml`

定义了完整的 CI/CD 流水线，包含以下步骤：
1. 代码拉取 (hbe-charge-pull)
2. 代码构建 (hbe-charge-build)
3. 应用部署 (hbe-charge-deploy)
4. 钉钉通知 (hbe-charge-dingtalk-notify)

#### 代码拉取模板
文件: `hbe-charge-pull-template.yaml`

负责从 GitLab 拉取代码，主要功能：
- 使用 GitLab 凭据进行身份验证
- 克隆指定分支的代码
- 检测变更文件并确定需要构建的服务模块
- 将代码存储在共享 PVC 中供后续步骤使用
- 输出需要构建的服务列表参数

#### 代码构建模板
文件: `hbe-charge-build-template.yaml`

负责构建 Go 应用，主要功能：
- 使用 Go 1.23.8 环境
- 并行构建多个服务模块
- 每个构建任务处理一个特定的服务
- 构建产物保存在共享 PVC 中供部署步骤使用

#### 应用部署模板
文件: `hbe-charge-deploy-template.yaml`

负责将构建产物部署到目标服务器，支持多环境部署：
- Dev 环境: 192.168.30.111
- Main 环境: 192.168.0.210
- Release 环境: 20.77.170.190

部署流程：
1. 通过 SSH 连接到目标服务器
2. 停止服务
3. 备份旧版本
4. 传输新版本可执行文件
5. 启动服务
6. 生成部署元数据

在部署过程中，系统会使用 `BUILD_TIME_FORMATTED` 变量，该变量取自 GitLab webhook payload 中的 `commits[0].timestamp` 字段，并在部署脚本中格式化为更易读的形式。

#### 钉钉通知模板
文件: `hbe-charge-dingtalk-notify-template.yaml`

负责发送构建结果通知到钉钉群，主要功能：
- 根据构建状态发送成功或失败通知
- 包含详细的构建信息和服务列表
- 支持@指定用户

## 配置说明

### Secret 配置

1. **钉钉 Webhook 密钥** (`dingtalk-webhook-secret.yaml`)
   - webhook-url: 钉钉机器人 Webhook URL
   - secret-token: 钉钉机器人安全密钥

2. **部署 SSH 密钥** (`deployment-ssh-keys.yaml`)
   - dev-ssh-key: Dev 环境部署私钥
   - main-ssh-key: Main 环境部署私钥
   - release-ssh-key: Release 环境部署私钥

3. **GitLab 认证信息** (`gitlab-pull-secret.yaml`)
   - username: GitLab 用户名
   - password: GitLab 密码

4. **GitLab 访问令牌** (`gitlab-access-secret.yaml`)
   - token: GitLab Personal Access Token（用于 API 调用）

5. **GitLab Webhook 密钥** (`gitlab-webhook-secret.yaml`)
   - secretToken: GitLab Webhook 验证令牌
   - 当前设置为: cicd (base64编码为: Y2ljZA==)
   - 注意: 此密钥用于 GitLab Webhook 认证，需要与 GitLab Webhook 配置中的 Secret Token 保持一致

## 部署步骤

1. 创建命名空间:
   ```bash
   kubectl create namespace argo-cicd
   ```

2. 部署 RBAC 和 ServiceAccount:
   ```bash
   kubectl apply -f env/rbac.yaml -n argo-cicd
   kubectl apply -f env/serviceaccount.yaml -n argo-cicd
   ```

3. 部署所有 Secret:
   ```bash
   kubectl apply -f dingtalk-webhook-secret.yaml -n argo-cicd
   kubectl apply -f deployment-ssh-keys.yaml -n argo-cicd
   kubectl apply -f gitlab-pull-secret.yaml -n argo-cicd
   kubectl apply -f gitlab-access-secret.yaml -n argo-cicd
   kubectl apply -f gitlab-webhook-secret.yaml -n argo-cicd
   ```

4. 部署 PVC:
   ```bash
   kubectl apply -f env/pvc.yaml -n argo-cicd
   ```

5. 部署 EventBus:
   ```bash
   kubectl apply -f eventbus.yaml -n argo-cicd
   ```

6. 部署 Event Source 和 Service:
   ```bash
   kubectl apply -f event-source-webhook.yaml -n argo-cicd
   kubectl apply -f env/event-source-webhook-service.yaml -n argo-cicd
   ```

7. 部署 Event Sensor:
   ```bash
   kubectl apply -f event-sensor-webhook.yaml -n argo-cicd
   ```

8. 部署 Workflow Templates:
   ```bash
   kubectl apply -f hbe-charge-pull-template.yaml -n argo-cicd
   kubectl apply -f hbe-charge-build-template.yaml -n argo-cicd
   kubectl apply -f hbe-charge-deploy-template.yaml -n argo-cicd
   kubectl apply -f hbe-charge-dingtalk-notify-template.yaml -n argo-cicd
   kubectl apply -f hbe-charge-pipeline.yaml -n argo-cicd
   ```

## 使用说明

部署完成后，可以通过以下 URL 触发 CI/CD 流水线：

- 触发 hbe-charge 项目: `http://<node-ip>:30080/hbe-charge`

### GitLab Webhook 配置

在 GitLab 中配置 Webhook 时，请确保以下配置正确：

1. URL: `http://192.168.30.149:30080/hbe-charge`
2. Secret Token: `cicd` (与 gitlab-webhook-secret.yaml 中配置的值一致)
3. Trigger: 选择 "Push events"

### 支持的分支和环境

| 分支名 | 部署环境 | 目标服务器 | 端口 |
|--------|----------|------------|------|
| dev | Dev 环境 | 192.168.30.111 | 22 |
| main | Main 环境 | 192.168.0.210 | 22 |
| release | Release 环境 | 20.77.170.190 | 22 |

## 故障排除

### 常见问题

1. **Webhook 无法触发流水线**
   - 检查 GitLab Webhook URL 和 Secret Token 是否正确配置
   - 确认 EventSource 服务是否正常运行
   - 检查网络连通性

2. **代码拉取失败**
   - 检查 GitLab 凭据是否正确
   - 确认 GitLab 项目访问权限
   - 检查网络连接

3. **构建失败**
   - 查看构建日志了解具体错误
   - 检查项目依赖是否正确安装
   - 确认 Go 版本兼容性

4. **部署失败**
   - 检查目标服务器 SSH 连接
   - 确认部署密钥是否正确
   - 检查目标服务器磁盘空间

### 日志查看

```bash
# 查看 EventSource 日志
kubectl logs -n argo-cicd -l eventsource-name=gitlab-eventsource

# 查看 Sensor 日志
kubectl logs -n argo-cicd -l sensor-name=hbe-charge-eventsensor

# 查看工作流日志
argo logs <workflow-name> -n argo-cicd
```
