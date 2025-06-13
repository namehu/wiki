# 开发指南

本文档旨在为美萌开发者提供开发指南，帮助开发者快速上手。

## 快速上手

本文档采用的是 Vscode + Docker 的开发环境。是依据 [Wiki DEV](https://docs.requarks.io/dev) 指南基础上做的修改。所以你需要先读完这个文档

## 仓库

本项目采用了双远端(remote)仓库机制。在项目中 需要有两个远端分支 `origin` 与 `wmeimob`。其中

- `origin`是默认GitHub远端分支。可以同步开源项目上的新增/BUG等
- `wmeimob`是公司云效仓库。用于触发流水线与部署

你可以通过 `git remote -v` 命令查看。

> 如果没有 `wmeimob` 分支。请先执行 `git remote add wmeimob git@codeup.aliyun.com:{xxxxxx}/r_d_center/wikijs.git` 添加远端分支
> 推送代码 **git push wmeimob**

## 开发

在package.json文件中。有两个命令需要关注

```bash
// 启动开发环境。会同时启动服务端与客户端应用
yarn dev


// 启动客户端。并监听文件修改。 注意： 实际操作过程发现这个dev修改并不生效。具体原因未查询
yarn watch
```

```bash
启动后访问 127.0.0.1:3000

账号 admin@wiki.com  9A#5b8@!3Cf6

数据库访问地址 http://localhost:3001/
```
## 镜像构建

项目开发完毕后。需要构建镜像。本仓库提供了 GitHubAction的构建与 云效流水线的构建。

**在Docker HUB 被墙后。目前不建议采用GitHubAction构建镜像。**。而只是作为同步分支使用。不过这里依旧会讲述如何构建

### 云效FLow

当你修改完代码后。需要构建镜像。在云效流水线中。设置了推送代码即触发构建.即  `git push wmeimob`到远端后进行构建-部署

### GitHubAction (不推荐)

在GitHubAction中。也设置了触发构建。目前采用的是通过tag 触发构建。

构建完毕后会将镜像推送到Docker Hub。

```bash
git tag v2.5.301-{内部版本}

git push --tags // 推送标签 触发GitHub Action
```

### 本地构建镜像

当然如果上述的两种方式你都不可用时。你可以自己在本地进行镜像构建。具体操作为

- 1.修改package.json中的dev为false。 关闭开发提示

- 2.构建镜像. 时间会比较长（10-20分钟）

```bash
# 构建镜像
docker build -t {yourworkspace}/wiki -f dev/build/Dockerfile .
```

- 3. 发布到远端仓库

```bash
docker push {yourworkspace}/wiki
```

### 服务器部署

在服务器上 拉取最新仓库镜像

```bash
// 进入文件夹
cd /compose-file-path

docker-compose down // 停止现有容器服务

docker-compose up -d // 启动
```
