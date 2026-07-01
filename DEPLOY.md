# 部署指南（Docker 快速发布）

本文档介绍如何把 subconverter **发布到服务器、构建 Docker 镜像并启动容器**。

> ⚠️ 重要区别
> - 官方 `scripts/Dockerfile` 是从 GitHub **克隆上游代码**构建的，**不包含你本地的改动**。
> - 若要部署你自己修改过的代码（例如本仓库对抓取 header 的改动），请使用**方式一**（从本地源码构建，配套文件 `scripts/Dockerfile.local`）。
> - 只想快速跑起来、不在乎本地改动，用**方式二**（官方镜像）。

---

## 0. 前置条件

服务器（Linux）需已安装 Docker：

```bash
# 检查是否已安装
docker version

# 未安装时（Debian/Ubuntu 快速安装）
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker
```

服务端口：subconverter 容器内监听 `25500`，下文默认映射到宿主机 `25500`。

---

## 方式一：从本地源码构建（包含你的改动，推荐）

### 1. 把代码发布到服务器

任选一种把当前工作目录同步到服务器：

```bash
# 方案 A：rsync（无需 git 远程，最直接）
rsync -avz --exclude='.git' --exclude='build' --exclude='subconverter' \
  ./ user@your-server:/opt/subconverter/

# 方案 B：git（已推到你自己的远程仓库时）
ssh user@your-server
git clone -b develop https://your-git-remote/subconverter.git /opt/subconverter
```

> 注：`.dockerignore` 已排除 `.git`、`build`、本地编译产物等，构建上下文会保持精简，无需手动清理。

### 2. 构建镜像

在服务器上进入**仓库根目录**，用本地源码 Dockerfile 构建：

```bash
cd /opt/subconverter

# -f 指定本地源码 Dockerfile；结尾的 . 是构建上下文（仓库根目录）
docker build -f scripts/Dockerfile.local -t subconverter:custom .

# 可选：把当前 commit 短 SHA 写进版本号，便于识别所部署的版本
docker build -f scripts/Dockerfile.local \
  --build-arg SHA=$(git rev-parse --short HEAD) \
  --build-arg THREADS=$(nproc) \
  -t subconverter:custom .
```

首次构建会编译 quickjspp / libcron / toml11 等依赖，耗时较长（数分钟起）；后续改动仅重编 subconverter，速度快很多。构建过程需联网（拉取依赖 + `update_rules.py` 下载规则集）。

### 3. 启动容器

```bash
docker run -d --name subconverter --restart=always \
  -p 25500:25500 \
  subconverter:custom
```

跳到 [验证](#验证) 一节确认服务正常。

---

## 方式二：直接用官方镜像（最快，不含本地改动）

```bash
docker run -d --name subconverter --restart=always \
  -p 25500:25500 \
  tindy2013/subconverter:latest
```

---

## docker-compose 方式（可长期维护，推荐生产使用）

在服务器仓库根目录放一个 `docker-compose.yml`：

```yaml
services:
  subconverter:
    # 二选一：
    # A. 从本地源码构建（含你的改动）
    build:
      context: .
      dockerfile: scripts/Dockerfile.local
      args:
        THREADS: "4"
    image: subconverter:custom
    # B. 直接用官方镜像时，删除上面的 build 段，改用：
    # image: tindy2013/subconverter:latest
    container_name: subconverter
    restart: always
    ports:
      - "25500:25500"
```

启动 / 更新：

```bash
# 构建并后台启动
docker compose up -d --build

# 查看日志
docker compose logs -f

# 停止并移除
docker compose down
```

---

## 验证

```bash
curl http://localhost:25500/version
# 期望输出： subconverter vx.x.x backend
# 若带了 SHA 构建参数，版本号会形如 v0.9.0-46e9af7

# 完整转换自测（把 <订阅链接> 换成实际 URL，记得 URLEncode）
curl "http://localhost:25500/sub?target=clash&url=<订阅链接>"
```

看到 `subconverter vx.x.x backend` 即表示容器已正常运行。

---

## 更新配置（无需重建镜像）

subconverter 支持在运行时热更新 `pref` 配置：

```bash
# 假设本地配置文件名为 newpref.ini，token 为配置里设置的值
curl -F "data=@newpref.ini" \
  "http://localhost:25500/updateconf?type=form&token=password"
```

若要长期使用自定义配置 / 规则 / 模板，把替换文件放进 `base/`（镜像内路径 `/base/`，结构与仓库 `base/` 一致）后重建镜像；或用挂载卷覆盖：

```bash
docker run -d --name subconverter --restart=always \
  -p 25500:25500 \
  -v /opt/subconverter/base/pref.ini:/base/pref.ini \
  subconverter:custom
```

---

## 代码更新后重新部署（方式一）

```bash
cd /opt/subconverter
git pull          # 或再次 rsync 同步

docker build -f scripts/Dockerfile.local -t subconverter:custom .
docker stop subconverter && docker rm subconverter
docker run -d --name subconverter --restart=always -p 25500:25500 subconverter:custom

# 用 compose 则一条命令搞定：
# docker compose up -d --build
```

---

## 常用运维命令

```bash
docker logs -f subconverter          # 跟踪日志
docker restart subconverter          # 重启
docker stats subconverter            # 资源占用
docker image prune -f                # 清理旧的悬空镜像
```

---

## 常见问题

- **`/version` 里看不到我的改动**：多半是用了方式二（官方镜像）或忘了在服务器上重新 `git pull` / rsync。用方式一并加 `--build-arg SHA=...` 可确认部署的版本。
- **构建时下载依赖失败**：构建过程需联网访问 GitHub 与规则集源站；服务器网络受限时，考虑在能联网的机器上构建后 `docker save` / `docker load` 传输镜像。
- **端口冲突**：把 `-p 25500:25500` 左侧改成宿主机空闲端口，例如 `-p 15051:25500`。
