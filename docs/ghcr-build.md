# 用 GitHub Actions 构建 amd64 镜像（GHCR）

> 目的：在 GitHub 的 ubuntu（x86_64）runner 上原生构建 **linux/amd64** 镜像并推送到
> GitHub Container Registry（GHCR），服务器直接 `docker pull` 部署，无需本地跨架构构建、
> 无需手动传输 tar、无需在服务器上编译。

## 安全说明（重要）

- 本方案**不要求**任何 Docker Hub / 云平台账号密码。
- 推送 GHCR 使用 GitHub 自动注入的 `GITHUB_TOKEN`（Actions 内自动授权，无需配置 secrets）。
- 本目录所有文档与 workflow 中**不含任何真实账号、密码、令牌**；文中出现的 `<GitHub用户名>` 等均为占位符，请替换为你自己的值。
- 服务器拉取镜像的两种方式（见下），推荐**公开包**方式，全程无需任何凭据。

## 工作流文件

`.github/workflows/build-image.yml`

触发方式：
| 触发 | 生成的 tag |
|---|---|
| 推送 `main` 分支 | `latest` |
| 推送 `v*` 标签（如 `v4.5.2-fix`） | 标签名 |
| 手动 Run workflow（任意分支） | 12 位短提交号 |

构建产物：`ghcr.io/<GitHub用户名>/antigravity-manager:<tag>`

## 使用步骤

1. 把本仓库推送到你自己的 GitHub 仓库（fork 即可）。
2. 在 GitHub 仓库页 **Actions** → 选择 **Build and Publish amd64 Image** → **Run workflow**（或直接推送 `main` / 标签自动触发）。
3. 等构建完成（约 20~30 分钟，第一次会久一些）。

## 在服务器上部署

### 方式 A（推荐）：把 GHCR 包设为 Public

1. GitHub 仓库页 → 右侧 **Packages** → 进入 `antigravity-manager` 包页。
2. **Package settings** → **Change visibility** → 选 **Public**。
3. 服务器直接拉取，**无需任何登录**：

```bash
docker pull ghcr.io/<GitHub用户名>/antigravity-manager:<tag>
```

然后修改 compose 的镜像字段并重启：

```bash
cd /opt/antigravity-manager
sed -i 's|image: .*|image: ghcr.io/<GitHub用户名>/antigravity-manager:<tag>|' docker-compose.yml
docker compose up -d --force-recreate
```

### 方式 B：私有包 + 服务器登录

如果包保持私有，服务器需要先登录（使用你自己的 GitHub Personal Access Token，`read:packages` 权限）：

```bash
echo '<你的PAT>' | docker login ghcr.io -u <GitHub用户名> --password-stdin
docker pull ghcr.io/<GitHub用户名>/antigravity-manager:<tag>
```

> PAT 属于你的机密信息，请妥善保管，不要提交到任何仓库或文档。

## 常见问题

- **本地 Apple Silicon 也能构建**：workflow 在 GitHub 的 x86_64 runner 上原生构建，与本地架构无关。
- **需要 arm64 镜像**：把 workflow 中 `platforms: linux/amd64` 改为 `linux/amd64,linux/arm64`（构建时间约翻倍）。
- **构建缓存**：已启用 `type=gha` 缓存，后续构建会显著加快。
