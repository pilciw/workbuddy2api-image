# 用 GitHub 为 workbuddy 构建 Docker 镜像 —— 操作说明

适用仓库：https://github.com/ithtelab/workbuddy-manager （默认分支 `main`）
本文所有结论都来自仓库实际内容与实机核验，核验结果附在文末「已核验 / 未核验」。

---

## 0. 先分清要构建的是哪个镜像

| 镜像 | 是什么 | 源码在哪 | GitHub 构建可行吗 |
|---|---|---|---|
| **A. workbuddy-manager** | 管理端 + OpenAI 兼容网关（Python/FastAPI + Next.js 静态导出） | 本仓库 | **可行且官方支持**：仓库自带 fork 专用工作流 |
| **B. workbuddy2api（上游）** | 账号池反代服务（Go），容器名 `workbuddy2api`，端口 7863 | **原仓库 `Sliverkiss/workbuddy2api` 已被作者删除（现 404）**；源码只随本仓库 Release 分发 | 可行，但要自己搭：仓库里没有它的构建工作流（本仓库的 Dockerfile 明确写了**不含**上游） |

两条关键事实先摆在这里，它们决定了后面所有步骤：

1. `Dockerfile:5-7`：「**本镜像不含上游 workbuddy2api**。上游是独立的 Go 服务……推荐做法：两边分别用 compose 起，管理端通过 `WB2API_BASE` 连接上游。」
   所以**构建镜像 A 不需要上游源码**，两者是两条独立的构建线。
2. `docs/release-process.md:3-5`：「上游 `workbuddy2api` 的公开地址自 2026-09-23 起不可用……**不要把上游代码提交进本仓库，也不要另开公开镜像**。」
   所以镜像 B 的源码要另外取，而且**建议只推私有仓库/私有包**。

---

## 1. 前置条件

### 1.1 账号与权限（构建走 GitHub 托管 runner，本地不需要 Docker/Node/Go）

- 一个可用的 GitHub 账号；构建产物推到 **GHCR**（`ghcr.io`），用的是运行内置的 `secrets.GITHUB_TOKEN`，**不需要额外创建 PAT**。
- 仓库要开 Actions。**fork 出来的仓库第一次要手动启用**（Actions 页有提示按钮），否则推代码后什么都不会发生。
- 仓库 Settings → Actions → General → Workflow permissions 建议设为 **Read and write permissions**（工作流里已显式声明 `packages: write`，但仓库策略若是只读会拦住）。
- 可选：要同时推 Docker Hub，需要 `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN`（Access Token，权限 Read & Write，**不是登录密码**）。

### 1.2 运行时环境（如果你还要在机器上验证镜像）

- Docker 24+ 与 `docker compose` v2。镜像 A 的 compose 里挂了 `docker.sock`，可选。
- 磁盘：镜像 A 约 580MB、镜像 B 约 140MB（已实测两者在运行中的体积）；构建过程另需数 GB 临时空间。
- **网络**：镜像基础层来自 `docker.io`。本机与目标服务器**直连 docker.io 都不通**，服务器靠 `/etc/docker/daemon.json` 里的 `registry-mirrors`（`https://docker.pilciw.cc.cd`）拉取。GitHub runner 不受此影响。

### 1.3 构建期可选的镜像源参数（中国大陆网络下强烈建议显式指定）

镜像 A 的 Dockerfile 自带「官方源失败 → 自动降级国内镜像」的一次重试，但会先白等一轮。可直接传：

```bash
docker compose build \
  --build-arg DEBIAN_MIRROR=mirrors.aliyun.com \
  --build-arg PIP_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple \
  --build-arg NPM_REGISTRY=https://registry.npmmirror.com \
  --build-arg DOCKER_CLI_BASE=https://mirrors.aliyun.com/docker-ce \
  --build-arg COMPOSE_URL_PREFIX=https://ghfast.top/
```

（GitHub Actions 上不需要这些，runner 在境外。）

---

## 2. 仓库里涉及的构建配置文件与目录

| 路径 | 作用 |
|---|---|
| `Dockerfile` | 镜像 A 的唯一定义。两阶段：`node:20-slim AS web-builder` 产出前端到 `/dist`，`python:3.12-slim` 为主镜像 |
| `docker-compose.yml` | 本地构建/运行编排：`build.context: .`、`image: workbuddy-manager:latest`、端口 `127.0.0.1:7864`、卷 `./data`、`../workbuddy2api`、`/var/run/docker.sock` |
| `.dockerignore` | 构建上下文排除清单（必须精确排除 `web/node_modules`，且**不得**排除 `web/` 整个目录） |
| `.github/workflows/release.yml` | **维护者发版用**。打 Release 包（tar.gz/zip + 签名）、并把镜像推到 `ghcr.io/<owner>/workbuddy-manager`（`platforms: linux/amd64,linux/arm64`、`push: true`） |
| `deploy/fork-image/build-image.yml` | **fork 专用工作流模板**（上游默认不启用）。零配置：镜像归谁、触发分支、来源标签都运行时推断 |
| `deploy/fork-image/README.md` | 上面那份工作流的用法说明 |
| `deploy/README.md` | 部署指南（含上游源码来源、构建失败排查） |
| `deploy/install.sh` | 一键安装（宿主机形态）：上游源码三种来源 + `docker compose up -d --build` |
| `deploy/update.py` | 容器内一键更新器（会重建上游容器，依赖镜像内的 compose 插件） |
| `deploy/check-upstream.sh` | 发版前查上游新提交；上游已 404，现在退化为 no-op 并 exit 0 |
| `deploy/verify-release.sh` | 发布包验签（`ssh-keygen -Y verify`），与镜像无关 |
| `dev/check_dockerfile.py` | Dockerfile 结构自检（指令拼写、`COPY --from` 阶段存在、关键契约） |
| `dev/check_dockerfile_build.py` | 真跑 web-builder 的 RUN 块，验三种分支（有产物 / 产物不完整 / 无产物） |
| `dev/pack_upstream_src.py` | 维护者用：把上游源码打成 `workbuddy2api-src.tar.gz` 上传到 `upstream-src` Release |
| `server/tests/test_docker_deploy.py` | Docker/compose/镜像名的硬约束测试（端口必须绑 `127.0.0.1`、卷必须含 `/app/data` 与 `/opt/workbuddy2api`、镜像名必须带 `-multiarch` 等） |
| `.env.example` | **运行期**环境变量清单（不是构建期参数） |

### 镜像 A 的 Dockerfile 关键点（决定构建行为）

- 前端是**构建期**产物：`web/out/index.html` 已存在 → 直接复用（发布包/CI 走这条）；不存在 → 容器内 `npm ci` + `NEXT_OUTPUT_EXPORT=1 npx next build`（`git clone` 走这条）。这就是 issue #38「`COPY web/out: not found`」的修法。
- 构建参数：`NPM_REGISTRY`、`BASE_PATH`（写进 `NEXT_PUBLIC_BASE_PATH`，子路径部署必须与运行时 `WB_BASE_PATH` 一致）、`DEBIAN_MIRROR`、`DOCKER_CLI_VERSION`(27.3.1)、`DOCKER_CLI_BASE`、`COMPOSE_VERSION`(**必须停在 v2**)、`COMPOSE_URL_PREFIX`、`PIP_INDEX_URL`。
- 运行形态：非 root（uid 10001）、`EXPOSE 7864`、`HEALTHCHECK` 打 `http://127.0.0.1:7864/api/healthz`、CMD 为 uvicorn。

---

## 3. 镜像 A：workbuddy-manager（官方支持的 fork 构建）

### 步骤

```bash
# 1) 在 GitHub 上 fork 本仓库（或用 gh）
gh repo fork ithtelab/workbuddy-manager --clone
cd workbuddy-manager

# 2) 把 fork 专用工作流复制到 Actions 认的位置
mkdir -p .github/workflows
cp deploy/fork-image/build-image.yml .github/workflows/

# 3) 提交并推送（会触发构建）
git add .github/workflows/build-image.yml
git commit -m "ci: 构建自己的镜像"
git push

# 4) 也可以手动触发
gh workflow run "Build Image"
gh run list --workflow "Build Image"
gh run watch
```

工作流**不用改任何内容**（`deploy/fork-image/README.md:28-30`）。它运行时推断：

| 项 | 来源 |
|---|---|
| 镜像归属 | 仓库所有者，自动转小写 |
| 触发分支 | 仓库的**默认分支**（`if: github.ref_name == github.event.repository.default_branch`） |
| 触发路径 | `Dockerfile`、`.dockerignore`、`server/**`、`web/**`、`deploy/**`、工作流自身 |
| 来源标签 | `org.opencontainers.image.source` / `revision` 指向你的 fork 与那次提交 |
| Docker Hub | 有那两个 Secret 就推，没有就跳过（不报错） |

流程内部：Node 20 → `npm ci` + `npm run build:export` → 校验 `web/out/index.html` → QEMU + buildx → 登录 GHCR → `docker/build-push-action` 双架构构建推送（`provenance: false`，`cache-from/to: type=gha`）→ 运行页摘要输出可直接粘贴的拉取命令。

### 产物

- `ghcr.io/<你的用户名>/workbuddy-manager-multiarch:latest`
- `ghcr.io/<你的用户名>/workbuddy-manager-multiarch:sha-<7位短哈希>`

镜像名里的 `-multiarch` **不是笔误**：上游 `workbuddy-manager` 这个包名可能已被一个未链接到仓库的同名包占用，那种包 fork 的令牌拿不到写权限，推送会报 `denied: permission_denied: write_package`。

### 换成你自己的镜像来跑

`docker-compose.yml` 默认是本地 `build:`。改用拉取的镜像：把 `build:` 整段删掉，`image:` 换成你的名字：

```yaml
services:
  workbuddy-manager:
    image: ghcr.io/<你的用户名>/workbuddy-manager-multiarch:latest
```

---

## 4. 镜像 B：上游 workbuddy2api（原仓库已删除，需自建）

### 源码从哪来

- 原仓库 `github.com/Sliverkiss/workbuddy2api` **已 404**。
- 源码只随本仓库的分发：**固定 tag 的载体 Release `upstream-src`**，资产 `workbuddy2api-src.tar.gz`（839 KB，解包 288 个条目）。
- 包内顶层目录是 `workbuddy2api/`，**自带 `Dockerfile`、`docker-compose.yml`、`go.mod`、`LICENSE`、`config.example.json`**。其 Dockerfile：`golang:1.26-alpine` 构建（`go mod download` → 编译 6 个二进制）→ `alpine:3.20` 运行，非 root uid 10001，`EXPOSE 7863`，`HEALTHCHECK` 打 `/healthz`，ENTRYPOINT `/app/wb2api -config /app/config.json`。

下载地址（本文已实测 HTTP 200）：

```
https://github.com/ithtelab/workbuddy-manager/releases/download/upstream-src/workbuddy2api-src.tar.gz
```

### 路子 1（推荐）：私有仓库 + 现成工作流，构建时下载源码包

这样**不把上游代码提交进任何仓库**，也符合维护者「不要另开公开镜像」的要求。文件我已写好：

`out/build-upstream-image.yml`（同目录，已在服务器上通过 YAML 解析校验）

```bash
# 1) 建一个自己的私有仓库（名字随意，例如 workbuddy2api-image）
gh repo create workbuddy2api-image --private --clone
cd workbuddy2api-image
mkdir -p .github/workflows
cp /path/to/build-upstream-image.yml .github/workflows/
git add .github/workflows/build-upstream-image.yml
git commit -m "ci: 构建上游 workbuddy2api 镜像"
git push

# 2) 触发（或到 Actions 页点 Run workflow）
gh workflow run "Build Upstream Image"
gh run watch
```

要点：`permissions: packages: write`；默认只出 `linux/amd64`（要 arm64 手动触发时填 `linux/amd64,linux/arm64`，QEMU 会明显变慢）；产物 `ghcr.io/<你的用户名>/workbuddy2api-multiarch:latest` 与 `:sha-<短哈希>`；镜像里带 `io.workbuddy.upstream.src.sha256` 标签，记录源码包指纹。

### 路子 2：本地/服务器直接构建（不经 GitHub）

```bash
curl -fsSL -o /tmp/upstream-src.tar.gz \
  https://github.com/ithtelab/workbuddy-manager/releases/download/upstream-src/workbuddy2api-src.tar.gz
tar -xzf /tmp/upstream-src.tar.gz -C /tmp
cd /tmp/workbuddy2api
docker compose up -d --build      # 或：docker build -t workbuddy2api:local .
```

注意：直接 `docker build` **不要**加 `--platform` 之外的参数；上下文必须是 `workbuddy2api/` 目录本身（Dockerfile 是 `COPY go.mod ./` → `COPY . .`）。

---

## 5. 构建产物在哪里获取

| 产物 | 位置 |
|---|---|
| 镜像 A | `ghcr.io/<你的用户名>/workbuddy-manager-multiarch:{latest,sha-<7>}`；运行页摘要里直接给可粘贴的 `docker pull` 命令 |
| 镜像 B | `ghcr.io/<你的用户名>/workbuddy2api-multiarch:{latest,sha-<7>}` |
| 包页面 | `https://github.com/<用户名>?tab=packages`，点进包可看标签、digest、可见性 |
| 构建日志 / digest | Actions → 那次运行 → Summary（含 digest 与源码包 sha256）；`gh run view <id>` |
| 发布包（tar.gz/zip + `.sig`） | 那是**维护者发版产物**（`releases/latest`），不是镜像；fork 的工作流**不创建 Release、不签名** |
| 本地 `docker build` 的镜像 | 只在构建那台机器上，不会推到任何仓库 |

---

## 6. 如何验证构建成功

### 第一层：CI 本身

```bash
gh run list --workflow "Build Image" --limit 5     # 或 "Build Upstream Image"
gh run view <run-id>                                # 看每个 step
gh run view <run-id> --log                          # 出问题时看完整日志
```

通过判据：job 全绿；Summary 里出现镜像名 + `digest`（我的上游工作流还会打印源码包 sha256）。

### 第二层：镜像存在且内容对

注意：**本机与目标服务器都直连不上 docker.io**，`docker manifest inspect` 在这台服务器上也不可用（连 `python:3.12-slim` 都解析不了）。要用就走镜像源：

```bash
# 服务器上（daemon.json 已配 registry-mirrors: https://docker.pilciw.cc.cd）
docker pull ghcr.io/<用户名>/workbuddy-manager-multiarch:latest
docker image inspect ghcr.io/<用户名>/workbuddy-manager-multiarch:latest \
  --format '{{.Id}} {{.Architecture}} {{index .Config.Labels "org.opencontainers.image.revision"}}'

# 多架构确认（走镜像源时 manifest 查询可能不可用，可改看包页面）
docker buildx imagetools inspect ghcr.io/<用户名>/workbuddy-manager-multiarch:latest
```

镜像 A 应看到：`Architecture: amd64`（或 arm64）、标签里有你那次提交的 revision、`ExposedPorts: 7864/tcp`、`User: app`。
镜像 B 应看到：`7863/tcp`、ENTRYPOINT `/app/wb2api -config /app/config.json`。

### 第三层：真跑起来

```bash
# 镜像 A（沿用仓库 compose，只把 image 换成你的）
docker compose up -d
docker compose ps                 # STATUS 应为 Up (healthy)
curl -fsS http://127.0.0.1:7864/api/healthz
docker compose logs workbuddy-manager | grep -A2 密码   # 未设 WB_ADMIN_PASSWORD 时首启会打印随机密码

# 镜像 B
docker compose ps                 # workbuddy2api 应为 healthy
curl -fsS http://127.0.0.1:7863/healthz
```

功能层：打开 `http://<服务器IP>:7864`（若按仓库默认只绑 `127.0.0.1`，需经反向代理或临时改端口映射）→ 登录 → 设置页确认 `WB2API_BASE` 能连上上游 → 账号页应能列出上游 `auths/` 里的账号（**显示 0 个账号通常是卷属主问题，不是镜像问题**）。

### 第四层：溯源

`sha-<7位>` 标签是内容寻址的固定指针：`docker pull ...:sha-<7>` 拿到的就是那次提交构建出的那一份；与运行页 Summary 里的 `digest` 对得上即证明一致。

---

## 7. 常见失败原因与排查方向

| 现象 | 原因 | 排查 / 解决 |
|---|---|---|
| 推了代码但 Actions 没有任何运行 | fork 仓库**未启用 Actions**；或推的不是**默认分支**；或改动不在 `paths` 白名单 | Actions 页看提示并点启用；`gh run list`；确认改的是默认分支、且动了 `Dockerfile`/`server/**`/`web/**`/`deploy/**`/工作流自身 |
| 工作流跑了但 job 被跳过 | job 上 `if: github.ref_name == github.event.repository.default_branch` 不成立 | 在默认分支上推，或手动 `workflow_dispatch` |
| `denied: permission_denied: write_package` | 用了 `workbuddy-manager` 这个已被占用的包名；或令牌没有写权限 | 用工作流默认的 `-multiarch` 名；Settings → Actions → General → Workflow permissions 设为 Read and write |
| `repository name must be lowercase` | 用户名/组织名含大写 | 保留工作流里的 `${IMAGE,,}` 小写转换，别删 |
| 前端 step 失败：`构建失败：缺少 web/out/index.html` | `npm ci` 或 `next build` 没产出 | 看上一段日志；`npm ci` 失败多为 lockfile/网络；`next build` 失败多为依赖或内存 |
| `apt-get update` 502 / 超时 | 中国大陆访问 `deb.debian.org` | 传 `DEBIAN_MIRROR=mirrors.aliyun.com`；Dockerfile 内置一次自动降级，但会白等 |
| pip `SSL: UNEXPECTED_EOF_WHILE_READING` | 官方 PyPI 抖动 | 传 `PIP_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple` |
| 卡在下载 docker CLI / compose | `download.docker.com`、GitHub Releases 境外 | 传 `DOCKER_CLI_BASE=https://mirrors.aliyun.com/docker-ce`、`COMPOSE_URL_PREFIX=https://ghfast.top/` |
| `manifest unknown` / 拉基础镜像 TLS 超时 | docker.io 不可达 | 配 `/etc/docker/daemon.json` 的 `registry-mirrors`（本服务器已配） |
| `no space left on device` | 磁盘不足 | `docker system prune -af`；目标服务器根分区只剩 6.0 GB，够但紧张 |
| arm64 那一份极慢或超时 | arm64 在 amd64 runner 上靠 QEMU 模拟 | 只出 `linux/amd64`，或改用 arm64 原生 runner |
| 拉取时报 `unknown/unknown` 平台 | manifest list 里有 attestation 条目 | 工作流已设 `provenance: false`，别去掉 |
| 容器起来就退：`unable to open database file` | `./data` 属主是 root，容器以 uid 10001 跑 | 宿主机 `chown -R 10001:10001 ./data` |
| 管理端显示 0 个账号 | `auths/` 跨 uid 读不到 | `chown -R 10001:10001 /opt/wb/workbuddy2api/auths` |
| 浏览器 `ERR_CONNECTION_REFUSED` | 端口只绑 `127.0.0.1:7864`（设计如此，管理端持有全部凭据） | 经反向代理访问，或临时改成 `7864:7864` 并确认有访问控制 |
| 一键更新上游报 `FileNotFoundError: 'docker-compose'`（issue #28） | 镜像内 compose 插件缺失或升到了 v5（v5 需要外部 buildx） | `COMPOSE_VERSION` 必须停在 v2；别把 compose 升到 v5 |
| 面板一键更新拒绝安装 | fork 构建的镜像**不参与签名信任链** | 这是预期行为；镜像完整性与发版签名无关 |
| 上游镜像工作流第一步就失败 | `upstream-src` 包结构变了 | 我的工作流会打印缺哪个文件；不要用别的来源的包 |

---

## 8. 需要你补充 / 确认的信息

1. **GitHub 用户名或组织名**（决定镜像名与包归属；含大写也没关系，工作流会自动转小写）。
2. **仓库可见性**：镜像 A 的 fork 可以公开；镜像 B 上游维护者明确要求**不要另开公开镜像**，我按「私有仓库 + 私有包」设计的。若你坚持公开，需要你自己确认合规（上游是 MIT，法律上允许再分发，但请保留 `LICENSE` 与版权声明）。
3. **要不要 arm64**（默认只出 amd64；目标机是 x86_64，一般不需要）。
4. **要不要同时推 Docker Hub**（需要两个 Secret；镜像 A 的工作流已支持，镜像 B 的我没加，需要可以补）。
5. **部署方式**：你服务器现在是 `ghcr.io/ithtelab/workbuddy-manager:latest` + `ghcr.io/sliverkiss/workbuddy2api:latest`，且经 `ghcr.pilciw.cc.cd` 代理拉取。要不要我把 compose 改成引用你自己构建的镜像（并说明如何走你的 GHCR 代理）。
6. **是否要我真的执行**：需要你在本机有 `gh` 登录态，或授权我在服务器上跑本地构建。

---

## 9. 本文的核验边界

**已实测核验**：仓库树与关键文件内容（API 读取）；`Sliverkiss/workbuddy2api` 返回 404；`upstream-src` Release 与资产存在，本地下载 858,827 字节、解包 288 条目、含 Dockerfile/go.mod/LICENSE；该资产 URL 从服务器 `curl -I` 返回 200；`golang:1.26-alpine`、`alpine:3.20`、`python:3.12-slim` 标签经镜像源 Registry API 返回 200，且不存在的对照标签返回 404；三份工作流 YAML 经 `yaml.safe_load` 解析通过并断言了 triggers/permissions/steps/context。

**未核验**：没有真正跑过 GitHub Actions（需你的账号）；没有在服务器上执行任何构建；基础镜像标签是经镜像源验证的，不是直连 docker.io。

**顺带说明**：为完成核验，我在服务器 `/tmp` 放了三份 YAML 与一个响应文件（只读校验用），核验结束后已清理。
