# ACR Image Sync

通过 GitHub Actions 自动将 Docker Hub、ghcr.io、quay.io 等国外镜像仓库的镜像同步到阿里云容器镜像服务（ACR），解决国内拉取海外镜像慢或超时的问题。

## 原理

```
海外镜像源（ghcr.io / Docker Hub / quay.io 等）
      │ GitHub Actions Runner（海外服务器）
      │ skopeo copy --all：整体复制多架构 index（含 attestation）
      ▼
GitHub Actions（prepare 生成矩阵 → 每镜像一个并行 job）
      │ 顶层 digest 与源一致 → 跳过；不一致 → 同步并校验
      ▼
阿里云 ACR（国内节点）
      │ 你从 ACR 拉取，速度快
      ▼
你的服务器 / NAS / 本地机器
```

核心特性：

- **保留原始 digest**：用 `skopeo copy --all` 整体复制多架构 index，推送后 ACR 中的顶层 digest 与上游逐字节一致（可直接按 digest 去重）。
- **digest 去重**：每次运行先比较源与 ACR 的顶层 manifest digest，未变化的镜像直接跳过，不产生重复传输。
- **预检分流（自动降级）**：推送前预检上游格式，若含 ACR 个人版不兼容的内容（`tar+zstd` 压缩层、`oci.empty` config 的 attestation），自动转入降级路径——把镜像拉到本地 OCI layout、手术过滤掉 attestation 条目、按需把 zstd 层转换为 gzip 后再推送，保证内容持续更新。此类镜像的顶层 digest 与源不同属预期，去重改用「真实平台 config digest 集合」指纹。上游改回合规格式后自动恢复主路径，无需改配置。
- **并行执行**：`prepare` job 解析镜像列表生成 JSON 矩阵，每个镜像由独立 runner 并行同步，单镜像失败不影响其他镜像（`fail-fast: false`），最后由 `report` job 汇总——任一镜像失败则整个运行标记失败以触发告警。

## 快速开始

### 1.  Fork 本仓库

Fork 一份本仓库到自己的账号下

### 2. 配置 Secrets（敏感信息）

前往 **Settings → Secrets and variables → Actions → Secrets**，添加以下两项：

| Secret 名称 | 说明 | 获取方式 |
|:---|:---|:---|
| `ACR_USERNAME` | ACR 镜像仓库登录名 | 阿里云的账号名称（不是账号ID） |
| `ACR_PASSWORD` | ACR 独立登录密码 | ACR 控制台 → 访问凭证 → 设置固定密码 |

> 登陆名
> 密码是 ACR 专用的 Registry 密码，不是阿里云登录密码。

### 3. 配置 Variables（普通配置）

前往 **Settings → Secrets and variables → Actions → Variables**，添加以下三项：

| 变量名 | 示例值 | 说明 |
|:---|:---|:---|
| `ACR_REGISTRY` | `registry.cn-guangzhou.aliyuncs.com` | 你的 ACR 实例地址 |
| `ACR_NAMESPACE` | `my-mirror` | 你的 ACR 命名空间 |
| `SYNC_IMAGES` | 见下方格式 | 需要同步的镜像列表 |
| `MAX_PARALLEL` | `20`（可选） | 最大并行 job 数，默认 20，需为正整数 |

`SYNC_IMAGES` 的填写格式（每行一个镜像，用 `|` 分隔三列）：

```
# 源镜像完整地址|目标镜像名|标签1,标签2,...
ghcr.io/home-assistant/home-assistant|home-assistant|stable
nginx|nginx|latest,1.26
redis|redis|latest,7.2
```

- 第一列：源镜像的完整地址
- 第二列：推送到 ACR 时的镜像名称
- 第三列：需要同步的标签，多个标签用逗号分隔
- 以 `#` 开头的行会被忽略

### 4. 手动触发

配置完成后，前往 **Actions** 页面，选择 **Sync images to ACR** 工作流，点击 **Run workflow** → **Run**。

等待执行完成，镜像就会出现在你的 ACR 仓库中。每个镜像对应一个并行 job，可在运行页面查看各自的同步/跳过/失败统计。

### 5. 在本地使用

同步完成后，将 `docker-compose.yml` 或其他配置文件中的镜像地址改为 ACR 地址：

```yaml
# 之前
image: "ghcr.io/home-assistant/home-assistant:stable"

# 之后
image: "registry.cn-guangzhou.aliyuncs.com/my-mirror/home-assistant:stable"
```

拉取速度会明显提升。

## 自动更新

工作流默认每 8 小时自动运行一次（`0 */8 * * *`，UTC）。如果要修改时间，编辑 `.github/workflows/sync.yml` 中的 cron 表达式：

```yaml
on:
  schedule:
    - cron: '0 */8 * * *'   # 每 8 小时
```

常见的 cron 写法：

| 执行频率 | cron 表达式 |
|:---|:---:|
| 每天 6:00 | `0 6 * * *` |
| 每 8 小时 | `0 */8 * * *` |
| 每 6 小时 | `0 */6 * * *` |
| 每周一 6:00 | `0 6 * * 1` |

## 添加 / 删除镜像

不需要修改 YAML 文件，只需编辑 Variables 中的 `SYNC_IMAGES`：

- **添加镜像**：在末尾追加一行
- **删除镜像**：删除对应行
- **无变化的镜像**：自动跳过——顶层 digest 与上次一致时不重复同步

之后手动触发一次工作流即可。

## GitHub Actions 免费额度

| 项目 | 额度 |
|:---|:---:|
| 每月运行分钟数 | 2,000 分钟（公开仓库不限） |
| 单次运行耗时 | 并行执行，约 1~3 分钟（取决于最大单镜像） |
| 每月可运行次数 | 约 400~1000 次 |

并发上限受 GitHub 账户计划约束（Free 20 / Pro 40 个并发 job），`MAX_PARALLEL` 不应超过该值。

## 常见问题

### Q：为什么拉取 ghcr.io 镜像还是超时？

只有配置了 GitHub Actions 同步到 ACR 后，从 ACR 拉取才快。如果在本地直接 `docker pull ghcr.io/xxx`，不走任何加速。

### Q：为什么有些镜像的 digest 与上游不一样？

ACR 个人版对以下两类内容有服务端限制，无法按原字节接收：

| 上游特征 | ACR 的反应 |
|:---|:---|
| 层使用 `tar+zstd` 压缩 | `blob type invalid` |
| attestation 使用 `oci.empty` 作 config 类型 | `unknown manifest class` |

工作流会在推送前预检识别这些格式，自动转入**降级路径**（过滤 attestation 条目 + zstd 转 gzip）后推送——镜像内容完整可用、持续更新，但顶层 digest 与上游不同，且不携带 attestation 签名元数据。这类镜像的去重改用「真实平台 config digest 集合」指纹，同样不会重复推送。日志中的 `降级路径: 检测到 ...` 与 `手术完成：移除 N 个 attestation 条目` 即为该流程。

### Q：镜像大小有限制吗？

GitHub Actions 免费 runner 磁盘空间约 14GB，单个 job 超时上限 6 小时（工作流内设为 60 分钟）。超大镜像（如 5GB+）建议单独同步。
