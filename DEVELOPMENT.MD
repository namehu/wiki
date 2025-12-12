# 开发指南

本文档旨在帮助开发者快速了解并上手美萌（Meimob）Wiki 项目的开发流程。

## 📋 环境准备

本项目基于 VS Code + Docker 进行开发，基础架构参考了 [Wiki.js 官方开发指南](https://docs.requarks.io/dev)。在开始之前，建议先阅读官方文档以了解基本概念。

### 推荐工具
- **编辑器**: VS Code
- **环境**: Docker & Docker Compose
- **包管理器**: Yarn

---

## 📦 仓库管理

本项目采用**双远端（Double Remote）**仓库机制，以兼顾开源同步与内部部署需求。

### 分支说明
- **`origin`**: 默认的 GitHub 远端仓库。用于同步开源项目的更新、Bug 修复等。
- **`wmeimob`**: 公司内部云效仓库。用于触发 CI/CD 流水线与部署。

### 配置指南

1. **查看当前远端配置**
   ```bash
   git remote -v
   ```

2. **添加内部仓库（如果缺失）**
   如果你的配置中没有 `wmeimob`，请执行以下命令添加：
   ```bash
   git remote add wmeimob git@codeup.aliyun.com:{xxxxxx}/r_d_center/wikijs.git
   ```

3. **日常推送**
   开发完成后，请推送到内部仓库以触发流水线：
   ```bash
   git push wmeimob
   ```

---

## 💻 本地开发

### 启动项目

在 `package.json` 中提供了以下核心开发命令：

```bash
# 1. 启动服务端
yarn dev

# 2. 启动客户端（支持热更新/监听文件修改）
yarn watch
```

### 访问服务

启动成功后，可通过以下地址访问服务：

| 服务 | 地址 | 默认账号 | 默认密码 |
| :--- | :--- | :--- | :--- |
| **Wiki 站点** | http://127.0.0.1:3000 | `admin@wiki.com` | `9A#5b8@!3Cf6` |
| **数据库管理** | http://localhost:3001/ | - | - |

---

## 🚀 镜像构建与发布

项目开发完成后需要构建 Docker 镜像。我们支持多种构建方式，推荐使用云效流水线。

### 1. 云效 Flow (推荐) ✅

这是公司内部标准的发布流程：

1. 提交代码并推送到内部远端：
   ```bash
   git push wmeimob
   ```
2. 前往云效流水线页面，手动或自动触发构建与部署任务。

### 2. GitHub Action (不推荐) ⚠️

> **注意**: 由于 Docker Hub 网络连接问题，目前仅建议将 GitHub Action 作为代码同步手段，不建议依赖其进行镜像构建。

GitHub Action 通过 Git Tag 触发：

```bash
# 打标签
git tag v2.5.301-{内部版本号}

# 推送标签触发 Action
git push --tags
```

### 3. 本地构建 (备选方案) 🛠️

当上述 CI/CD 工具不可用时，可手动在本地构建镜像。

**步骤：**

1. **配置修改**: 将 `package.json` 中的 `dev` 字段设为 `false`，关闭开发提示。
2. **构建镜像**: (耗时约 10-20 分钟)
   ```bash
   # 请替换 {yourworkspace} 为实际的命名空间
   docker build -t {yourworkspace}/wiki -f dev/build/Dockerfile .
   ```
3. **推送到远端**:
   ```bash
   docker push {yourworkspace}/wiki
   ```

---

## 🌐 服务器部署

在目标服务器上更新服务：

```bash
# 1. 进入 Compose 文件目录
cd /compose-file-path

# 2. 停止现有服务
docker-compose down

# 3. 拉取最新镜像并启动
docker-compose up -d
```
