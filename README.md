# OpenTodo 懒猫微服打包仓库

把上游开源项目 [OpenTodoApp/OpenTodo](https://github.com/OpenTodoApp/OpenTodo) 打包成懒猫微服（LPK v2），
并通过 GitHub Action 自动跟进上游镜像的新版本、发布到喵喵商店。

## 打包方式：镜像模式

本应用**不包含上游源码**，也不在本地或 GitHub 上构建上游代码：

- 运行时使用上游官方镜像 `ghcr.io/opentodoapp/opentodo`（Go 静态二进制 + 内嵌网页，distroless 镜像）。
- GitHub Action 定期检查上游镜像的版本 tag（`1.5.0` 这种三段 SemVer），
  选中最新稳定版后把镜像地址写回 `lzc-manifest.yml`，构建 LPK 并发布。
- 仓库里只有打包配置与图标，没有任何上游源码。

## 目录结构

| 文件 | 作用 |
|------|------|
| `package.yml` | 应用元数据：包名、版本、名称、多语言描述、权限、最低系统版本 |
| `lzc-manifest.yml` | 运行配置：入口、服务、镜像、环境变量、健康检查、数据目录挂载 |
| `lzc-build.yml` | 构建配置：指向 manifest、产物目录与图标 |
| `icon.png` | 启动器图标（取自上游 `web/public/pwa-512.png`） |
| `.github/lazycat-action.yml` | Action 配置：镜像模式、版本来源、喵喵商店发布 |
| `.github/workflows/lazycat.yml` | 定时（每 3 小时）+ 手动触发的构建发布流程 |

## 关键配置说明

- **包名**：`community.lazycat.app.opentodo`（社区前缀）
- **最低系统版本**：`1.6.0`。上游镜像以非 root 用户 `65532` 运行，manifest 里用
  `run_as: "65532:65532"` 声明容器 UID/GID，同时把 `/lzcapp` 下持久目录的 owner
  映射成同一个 UID，数据目录才能正常写入；这个字段需要 lzcos v1.6.0 及以上。
- **对外地址**：`OPENTODO_BASE_URL=https://{{ .S.AppDomain }}`。
  设置 base URL 后，Cookie 会标记 Secure、启用 HSTS、通行密钥可用，
  也不会出现「以域名打开却要求输入初始化码」的情况。
- **数据目录**：`/lzcapp/var/data:/data`，容器里的 `/data` 存放数据库、附件与推送密钥，
  升级/重装都不丢。
- **健康检查**：沿用镜像自带的 `/opentodo healthcheck`（distroless 镜像里没有 shell，
  所以用 exec 形式而不是 `CMD-SHELL`）。
- **只发布喵喵商店**：官方商店要求免密登录与文件选择器拦截，本应用暂不提交，
  因此 `stores.official.enabled: false`。

## 更新上游版本

不需要人工操作：GitHub Action 每 3 小时跑一次，发现上游有新版本 tag 就自动构建并发布。
需要手动触发时，在 Actions 页面点 “Publish OpenTodo LPK” → “Run workflow”。

## 本地校验

```bash
lzc-cli project release -o dist/opentodo.lpk   # 需要 lzc-cli v2.0.0+
lzc-cli lpk info dist/opentodo.lpk
```

## 已知取舍

- 打包产物是单架构（amd64）。上游镜像同时提供 amd64 与 arm64，
  需要 arm64 时把 `.github/lazycat-action.yml` 与 workflow 里的 `target_arch` 改成 `arm64`。
- 应用自带账号体系（密码 / 通行密钥 / 邀请），没有接入懒猫微服的免密登录；
  同时保留了微服自身的账号密码作为外层保护。

## 上游

- 项目主页：https://open-todo.app
- 源码仓库：https://github.com/OpenTodoApp/OpenTodo
- 许可：AGPL-3.0
