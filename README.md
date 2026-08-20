# dsh-unraid-template

DeepSeek Harness (dsh) 的 Unraid 公共安装模板 —— 对应 Docker Hub 镜像 `crushleorey/dsh`。

## 这是什么

`my-dsh.xml` 是 Unraid Community Apps 格式的模板，一键部署 **dsh**（DeepSeek 官方 agent harness，"everything is a plugin"）为受管容器。

单容器架构（镜像内自带，用户无感）：
- nginx（80/443，basic auth + 自签名 TLS）反代 → dsh（绑定 127.0.0.1:3080，不暴露 RCE 到局域网）
- 插件 / 配置 / 凭证持久化在 `/data`（Harness Home 卷）

## 安装方法

**方式一：Community Apps（推荐）**

1. Unraid 的 Apps → 右上角齿轮 → **Template Repositories** → 添加：
   `https://github.com/Victzhang79/dsh-unraid-template`
2. 回到 Apps 页搜索 **dsh** → 点击安装。

**方式二：手动**

Docker → Add Container → 从本仓库的 `my-dsh.xml` 导入模板。

## 首次登录

- 用户名：`admin`（模板默认）
- 密码：**DSH_AUTH_PASS 留空则容器首启用 openssl 生成随机密码**，写入挂载卷里的 `initial-password.txt`（例如 `/mnt/user/appdata/dsh/home/initial-password.txt`）。
- 浏览器接受一次自签名证书警告即可。

修改 Web UI 密码：在 Unraid 编辑页直接改模板里的 `DSH_AUTH_PASS` 变量 → **Apply**(容器重建,entrypoint 用新密码重建 htpasswd),改完用新密码登录即可。

## 镜像

- 镜像仓库：`crushleorey/dsh`（Docker Hub，公开，**multi-arch：amd64 + arm64**）
- 当前版本：`0.1.0-rc.8`
- 发布策略：**带版本号的 tag + 漂浮 `latest` 双轨**（模板默认用 `latest`，Unraid Update 按钮可检测新推送一键更新）；全部公开 tag 均为 multi-arch manifest，x86 与 ARM 上同一命令自动拉取对应架构。
- 基础镜像：`node:24-slim`（node 24 "Krypton" LTS，Active LTS 支持到 2028-04；dsh 未声明 engines，node 22/24 均验证可用）
- 更新：改模板 `Repository` 为 `crushleorey/dsh:0.1.0-rc.8<新版本>` → Apply，或点 Unraid 的 Update 按钮（从 Docker Hub 拉最新 tag）。

## 常见问题：Settings / 提供方目录打不开？（上游 rc.8 起 loopback 限制，本镜像已内置 lanfix）

上游从 **rc.8** 起把「Settings / 提供方目录 / 模型 / 凭据」这类特权配置 UI 限制为只能从 **loopback 浏览器**打开（远程浏览器默认不发起特权读）。但我们这个镜像**已经内置 lanfix 补丁**，彻底绕开了这个限制：

- 用 LAN IP 打开 `https://<dsh-ip>/` → 登录 → **Settings / 提供方 / 模型 / 凭据 直接可用**，无需 SSH 隧道、不会出现 `settings are unavailable in this browser`。
- lanfix 只改 `dsh-client-connection` 的 `isLoopback` 判定一行，不影响 dsh 本体；锚点带 fail-closed 校验（每构建自动重打，锚点变了构建失败并回滚旧镜像，不会静默丢补丁）。

> 若你因任何原因跑的是**上游原版/未打补丁**镜像，才需要改走 loopback 入口：本地 `ssh -N -L <port>:<dsh-ip>:443 <跳板机>`，浏览器开 `https://localhost:<port>`（接受一次自签证书警告）。

## Env 说明

| 变量 | 默认 | 说明 |
|---|---|---|
| `DSH_AUTH_USER` | `admin` | basic auth 用户名 |
| `DSH_AUTH_PASS` | 空 | basic auth 密码；留空则首启生成随机密码 |
| `DSH_TRUSTED_HOST` | 空 | trust fence 放行 Host；通常留空即可（nginx 已把 Host 重写为 loopback） |
| `DSH_PERMISSION_MODE` | `danger-full-access` | 容器内无 bwrap/Landlock 时给 bash 工具免沙箱权限 |

> ⚠️ `danger-full-access`：容器内 agent 可执行任意命令（无沙箱）。请只挂载可信目录，勿用 `--privileged`。

## 构建（维护者用）

镜像由 `node:24-slim` + nginx/openssl + `@deepseek-ai/dsh@<ver>` 构建，Dockerfile/entrypoint/nginx.conf 见同源留档。本仓库只托管 Unraid 模板，镜像单独推 Docker Hub（amd64 由 x86 宿主构建、arm64 由 aarch64 宿主构建后合并 multi-arch manifest）。