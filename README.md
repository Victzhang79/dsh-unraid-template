# dsh-unraid-template

[![Docker Pulls](https://img.shields.io/docker/pulls/crushleorey/dsh?logo=docker)](https://hub.docker.com/r/crushleorey/dsh)
[![Docker Image Size](https://img.shields.io/docker/image-size/crushleorey/dsh/latest?label=image-size&logo=docker)](https://hub.docker.com/r/crushleorey/dsh/tags)
[![Docker Image Version](https://img.shields.io/docker/v/crushleorey/dsh/latest?label=version&logo=docker)](https://hub.docker.com/r/crushleorey/dsh/tags)
![arch](https://img.shields.io/badge/arch-amd64%20%2F%20arm64-green?logo=linux)
[![GitHub stars](https://img.shields.io/github/stars/Victzhang79/dsh-unraid-template?logo=github&style=flat)](https://github.com/Victzhang79/dsh-unraid-template/stargazers)

One-tag, dual-arch (**amd64 + arm64**) Unraid template for **DeepSeek Harness (dsh)** — DeepSeek's official "everything is a plugin" agent harness, delivered as a hardened single container (nginx basic-auth + self-signed TLS in front, dsh bound to loopback).

DeepSeek 官方 agent harness **dsh** 的 Unraid 公共安装模板：单 tag 双架构自适应，容器内 nginx（basic auth + 自签 TLS）反代、dsh 只绑 loopback，插件/配置/凭据持久化在 `/data`。

## Install（Unraid Community Apps 一键装）

1. Unraid 的 **Apps → 右上角齿轮 → Template Repositories** → 添加：
   `https://github.com/Victzhang79/dsh-unraid-template`
2. 回到 Apps 页搜索 **dsh** → 点击安装（任意 Docker 宿主 `docker run` 同样可用，详见 [Docker Hub Overview](https://hub.docker.com/r/crushleorey/dsh)）。

## First login（首次登录）

- 用户名：`admin`（模板默认）
- 密码：**`DSH_AUTH_PASS` 留空则容器首启用 openssl 生成随机密码**，写入挂载卷里的 `initial-password.txt`（如 `/mnt/user/appdata/dsh/home/initial-password.txt`）
- 浏览器接受一次自签证书警告即可；改密码 = 在 Unraid 模板页改 `DSH_AUTH_PASS` → **Apply**（entrypoint 用新密码重建 htpasswd）

## Update（更新）

模板固定拉取 `crushleorey/dsh` 镜像的 `latest` tag（multi-arch，x86/ARM 同一 tag 自动适配）——**点 Unraid 的 Update 按钮即可**，无需手改版本号；维护侧每日 3:00 自动构建 amd64（x86 宿主）+ arm64（DGX Spark）并合并 multi-arch manifest 推送。

## FAQ：Settings / 提供方目录打不开？（上游 loopback 限制，本镜像已内置 lanfix）

上游从 **rc.8** 起把「Settings / 提供方目录 / 模型 / 凭据」特权 UI 限制为只能从 **loopback 浏览器**打开。本镜像**已内置 lanfix 补丁**：LAN IP 直接打开 `https://<dsh-ip>/` → 登录 → Settings/提供方/模型/凭据 **直接可用**（无需 SSH 隧道、不会出现 `settings are unavailable in this browser`）。lanfix 只改 `dsh-client-connection` 的 `isLoopback` 判定一行，锚点带 fail-closed 校验（锚点变了构建失败并回滚旧镜像，不会静默丢补丁）。已验证至 **0.2.2-rc.5**。

> 若跑的是上游原版/未打补丁镜像，才需要 loopback 入口：`ssh -N -L <port>:<dsh-ip>:443 <跳板机>` 后浏览器开 `https://localhost:<port>`（接受一次自签证书警告）。

## Env 说明

| 变量 | 默认 | 说明 |
|---|---|---|
| `DSH_AUTH_USER` | `admin` | basic auth 用户名 |
| `DSH_AUTH_PASS` | 空 | basic auth 密码；留空则首启生成随机密码 |
| `DSH_TRUSTED_HOST` | 空 | trust fence 放行 Host；通常留空即可（nginx 已把 Host 重写为 loopback）；若经域名/反代访问遇 /api 403，把访问用的主机名填这里 |
| `DSH_PERMISSION_MODE` | `danger-full-access` | 容器内无 bwrap/Landlock 时给 bash 工具免沙箱权限 |

> ⚠️ `danger-full-access`：容器内 agent 可执行任意命令（无沙箱）。只挂载可信目录，勿用 `--privileged`。

## Image（镜像）

- 镜像仓库：`crushleorey/dsh`（Docker Hub 公开，**multi-arch：amd64 + arm64**，双架构均通过浏览器级 UI 端到端验证）
- 发布策略：**版本化 tag + 漂浮 `latest` 双轨**——版本化 tag 可回滚/pin，`latest` 供自动更新；单架构 `-amd64`/`-arm64` 为实现细节 tag
- 基础镜像：`node:24-slim`（Node 24 "Krypton" LTS，Active LTS 至 2028-04；dsh 未声明 engines，node 22/24 均验证可用）
- 详细说明（资源实测/安全设计/架构图）见 [Docker Hub Overview](https://hub.docker.com/r/crushleorey/dsh)

## Feedback & Star（反馈）

- **报障 / 需求**：[Issues](https://github.com/Victzhang79/dsh-unraid-template/issues) —— 是最有用的信号
- **如果这个镜像帮你省了时间，点个 Star** ⭐ —— 让其他 Unraid/Docker 用户也能找到它

## 构建（维护者用）

镜像由 `node:24-slim` + nginx/openssl + `@deepseek-ai/dsh@<ver>` 构建，Dockerfile/entrypoint/nginx.conf 见同源留档。本仓库只托管 Unraid 模板；镜像单独推 Docker Hub（amd64 由 x86 宿主构建、arm64 由 aarch64 宿主构建后合并 multi-arch manifest）。
