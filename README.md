# Argo Workflow CI/CD 项目说明

本项目基于 Argo Workflows 实现一套完整的 CI/CD 流水线，用于自动化构建和部署应用。

## 项目结构

```
.
├── event-source-webhook.yaml           # Webhook 事件源配置
├── event-sensor-webhook.yaml           # 事件传感器配置
├── snciot-backend2-0-pipeline.yaml     # 主工作流模板
├── snciot-backend2-0-pull-template.yaml     # 代码拉取模板
├── snciot-backend2-0-build-template.yaml    # 构建模板
├── snciot-backend2-0-deploy-template.yaml   # 部署模板
├── snciot-backend2-0-dingtalk-notify-template.yaml  # 钉钉通模板
├── dingtalk-webhook-secret.yaml        # 钉钉 Webhook 密钥
├── deployment-ssh-keys.yaml            # 部署 SSH 密钥
├── gitlab-pull-secret.yaml             # GitLab 拉取代碼密钥
├── gitlab-webhook-secret.yaml          # GitLab Webhook 密钥
├── event-source-webhook-service.yaml   # Webhook 服务配置
└── pvc.yaml                            # 持久卷声明
env
```

## 核心组件

### 1. Event Source (事件源)
文件: `event-source-webhook.yaml`

定义了 Webhook 事件源，监听两个端点：
- `/snciot_backend2-0`: 用于触发 snciot-backend2-0 项目的 CI/CD 流水线
- `/hbe-charge`: 用于触发 hbe-charge 项目的 CI/CD 流水线

### 2. Event Sensor (事件传感器)
文件: `event-sensor-webhook.yaml`

监听来自 Event Source 的事件，并触发相应的工作流。包含两个触发器：
- snciot-backend2-0-trigger: 处理 snciot-backend2-0 项目事件
- hbe-charge-trigger: 处理 hbe-charge 项目事件

### 3. Workflow Templates (工作流模板)

#### 主工作流模板
文件: `snciot-backend2-0-pipeline.yaml`

定义了完整的 CI/CD 流水线，包含四个步骤：
1. 代码拉取 (snciot-backend2-0-pull)
2. 代码构建 (snciot-backend2-0-build)
3. 应用部署 (snciot-backend2-0-deploy)
4. 钉钉通知 (snciot-backend2-0-dingtalk-notify)

#### 代码拉取模板
文件: `snciot-backend2-0-pull-template.yaml`

负责从 Git 仓库拉取代码，支持不同分支的代码获取。

#### 代码构建模板
文件: `snciot-backend2-0-build-template.yaml`

负责执行代码构建任务，使用 Node.js 环境进行前端项目构建。

#### 应用部署模板
文件: `snciot-backend2-0-deploy-template.yaml`

负责将构建好的应用部署到目标服务器，支持多环境部署（dev、main、release）。

#### 钉钉通知模板
文件: `snciot-backend2-0-dingtalk-notify-template.yaml`

在流水线执行完成后发送通知到钉钉群，包含构建状态、提交信息等。

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

4. **GitLab Webhook 密钥** (`gitlab-webhook-secret.yaml`)
   - secretToken: GitLab Webhook 验证令牌

## 部署步骤

1. 创建命名空间:
   ```bash
   kubectl create namespace argo-cicd
   ```

2. 部署所有 Secret:
   ```bash
   kubectl apply -f dingtalk-webhook-secret.yaml -n argo-cicd
   kubectl apply -f deployment-ssh-keys.yaml -n argo-cicd
   kubectl apply -f gitlab-pull-secret.yaml -n argo-cicd
   kubectl apply -f gitlab-webhook-secret.yaml -n argo-cicd
   ```

3. 部署 PVC:
   ```bash
   kubectl apply -f pvc.yaml -n argo-cicd
   ```

4. 部署 Event Source 和 Service:
   ```bash
   kubectl apply -f event-source-webhook.yaml -n argo-cicd
   kubectl apply -f event-source-webhook-service.yaml -n argo-cicd
   ```

5. 部署 Event Sensor:
   ```bash
   kubectl apply -f event-sensor-webhook.yaml -n argo-cicd
   ```

6. 部署 Workflow Templates:
   ```bash
   kubectl apply -f snciot-backend2-0-pull-template.yaml -n argo-cicd
   kubectl apply -f snciot-backend2-0-build-template.yaml -n argo-cicd
   kubectl apply -f snciot-backend2-0-deploy-template.yaml -n argo-cicd
   kubectl apply -f snciot-backend2-0-dingtalk-notify-template.yaml -n argo-cicd
   kubectl apply -f snciot-backend2-0-pipeline.yaml -n argo-cicd
   ```

## 使用说明

部署完成后，可以通过以下 URL 触发 CI/CD 流水线：

- 触发 snciot-backend2-0 项目: `http://<node-ip>:30080/snciot_backend2-0`
- 触发 hbe-charge 项目: `http://<node-ip>:30080/hbe-charge`

Webhook 请求应包含以下参数：
- ref: Git 分支引用 (如 refs/heads/master)
- git-repo-url: Git 仓库地址
- commit-author: 提交作者
- commit-message: 提交信息
- commit-id: 提交 ID
- project-name: 项目名称
- target-branch: 目标分支
- build-user: 构建用户
- git-commit: Git 提交哈希
- build-time: 构建时间