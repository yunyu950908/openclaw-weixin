# CI Workflows

本仓库使用三个 workflow 协作完成 npm 包的"同步 → 合并 → 发版"流水线，按事件驱动解耦。

## 工作流总览

```
┌─────────────────────────┐
│  sync-package.yml       │  ← 定时 / 手动触发
│  检测新版本 → 推 sync 分支 │
│       → 创建 PR          │
└──────────┬──────────────┘
           │ workflow_run (success)
           ▼
┌─────────────────────────┐
│  auto-merge.yml         │
│  查找 bot PR → 等待       │
│  mergeable → squash 合并 │
└──────────┬──────────────┘
           │ push to main
           ▼
┌─────────────────────────┐
│  release.yml            │
│  匹配 commit → 打 tag    │
│       → 创建 Release     │
└─────────────────────────┘
```

---

## 1. `sync-package.yml` — 同步上游 npm 包

### 做什么

每日定时从 npm registry 拉取 [`@tencent-weixin/openclaw-weixin`](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin) 的最新版本源码，同步到本仓库并创建 PR。

### 工作流程

```
npm view 获取最新版本
        │
        ▼
与 VERSION 文件比较 ──(相同)──▶ 跳过，结束
        │
      (不同)
        ▼
下载 tarball 并解压
        │
        ▼
清除旧文件，同步新源码
        │
        ▼
关闭旧的 sync/ PR（如有）
        │
        ▼
创建 PR (sync/v{version})
```

### 触发方式

| 方式 | 说明 |
|------|------|
| 定时 | 每天 UTC 00:00（北京时间 08:00） |
| 手动 | Actions 页面 → Run workflow |

### 关键文件

| 文件 | 用途 |
|------|------|
| `VERSION` | 记录当前已同步的版本号，用于跳过重复同步 |
| `sync/v*` 分支 | 临时分支，合并后自动删除 |

---

## 2. `auto-merge.yml` — 自动合并机器人 PR

### 做什么

监听 `sync-package.yml` 完成事件，查找其创建的 PR 并在无冲突时自动 squash merge。

### 触发方式

| 方式 | 说明 |
|------|------|
| 自动 | `workflow_run` 事件，sync-package.yml 成功完成后触发 |

### 关键设计

- **`workflow_run` 触发器** — 绕过 `GITHUB_TOKEN` 创建的 PR 不能触发 `pull_request` 事件的限制
- **双重过滤** — 同时校验分支前缀 (`sync/`) 和作者 (`app/github-actions`)，避免误合人工 PR
- **mergeable 轮询** — GitHub 后台计算 mergeability 需要几秒，最多重试 10 次（每次 sleep 3s）
- **冲突时静默跳过** — 不阻塞流水线，PR 留在仓库等人工处理

---

## 3. `release.yml` — 自动发版

### 做什么

监听 main 分支的 push，匹配 sync commit 后自动创建 git tag 和 GitHub Release。

### 触发方式

| 方式 | 说明 |
|------|------|
| 自动 | 任何 push 到 main 的事件 |
| 过滤 | commit message 必须以 `chore: sync @tencent-weixin/openclaw-weixin` 开头 |

### 关键设计

- **commit message 匹配** — 比 tag 触发更稳健，未来调整 tag 时机不影响此逻辑
- **幂等性检查** — 创建前用 `gh release view` 检测，已存在则跳过；避免 force push、重跑 workflow 等场景下重复创建

---

## 为什么需要这套 CI

本仓库用于镜像 `@tencent-weixin/openclaw-weixin` 的源码。该 npm 包不提供公开的源码仓库，通过这套 CI 可以：

- **追踪变更** — 每次上游发版自动生成 PR 和 Release，通过 git diff 清晰看到版本间差异
- **版本归档** — 每个版本对应一个 git tag，可随时切换到任意历史版本
- **变更通知** — Watch 仓库即可收到上游更新的 Release 通知

## 所需仓库设置

- **Settings → Actions → General → Workflow permissions**:
  - 选择 "Read and write permissions"
  - 勾选 "Allow GitHub Actions to create and approve pull requests"
