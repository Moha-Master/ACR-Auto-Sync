# ACR Image Sync

通过 GitHub Actions 自动将 Docker Hub、ghcr.io、quay.io 等国外镜像仓库的镜像同步到阿里云容器镜像服务（ACR），解决国内拉取海外镜像慢或超时的问题。

## 原理

```
海外镜像源（ghcr.io / Docker Hub / quay.io 等）
      │ GitHub Actions Runner（海外服务器）pull
      ▼
GitHub Actions（在海外机房执行）
      │ docker pull → docker tag → docker push
      ▼
阿里云 ACR（国内节点）
      │ 你从 ACR 拉取，速度快
      ▼
你的服务器 / NAS / 本地机器
```

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

等待执行完成，镜像就会出现在你的 ACR 仓库中。

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

工作流默认每天早上 6 点（UTC+8）自动运行一次。如果要修改时间，编辑 `.github/workflows/sync.yml` 中的 cron 表达式：

```yaml
on:
  schedule:
    - cron: '0 6 * * *'   # 每天早上6点
```

常见的 cron 写法：

| 执行频率 | cron 表达式 |
|:---|:---:|
| 每天 6:00 | `0 6 * * *` |
| 每 6 小时 | `0 */6 * * *` |
| 每天 0:00 和 12:00 | `0 0,12 * * *` |
| 每周一 6:00 | `0 6 * * 1` |

## 添加 / 删除镜像

不需要修改 YAML 文件，只需编辑 Variables 中的 `SYNC_IMAGES`：

- **添加镜像**：在末尾追加一行
- **删除镜像**：删除对应行
- **无变化的镜像**：脚本每次都会重新同步，即使镜像没有更新

之后手动触发一次工作流即可。

## GitHub Actions 免费额度

| 项目 | 额度 |
|:---|:---:|
| 每月运行分钟数 | 2,000 分钟（公开仓库不限） |
| 单次运行耗时 | 约 2~5 分钟（取决于镜像大小和数量） |
| 每月可运行次数 | 约 400~1000 次 |

## 常见问题

### Q：为什么拉取 ghcr.io 镜像还是超时？

只有配置了 GitHub Actions 同步到 ACR 后，从 ACR 拉取才快。如果在本地直接 `docker pull ghcr.io/xxx`，不走任何加速。


### Q：镜像大小有限制吗？

GitHub Actions 免费 runner 磁盘空间约 14GB，单次工作流总时长限 6 小时。超大镜像（如 5GB+）建议单独同步。
