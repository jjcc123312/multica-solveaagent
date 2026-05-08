# Multica 发布操作文档

> Fork 仓库：[jjcc123312/multica-solveaagent](https://github.com/jjcc123312/multica-solveaagent)
> 生产环境：阿里云硅谷区 ECS
> 适用对象：维护这套自部署的同事

---

## 一、整体流程（30 秒看懂）

```
本地改代码                   GitHub                ECS
───────────                ──────────             ────────
git push  ─────────►  build-{backend,web}.yml
                           │
                           │ ✓ 构建并推送镜像到 GHCR
                           ▼
                      deploy-{backend,web}.yml
                           │
                           │ ⏸ 等待人工 Approve
                           ▼
                      ssh ECS → 改 .env tag → docker pull → up -d → /readyz 烟测
                           │                                                   │
                           ✓ 部署成功                                          ✗ 自动回滚
```

**关键事实**：
- 镜像仓库：`ghcr.io/jjcc123312/multica-solveaagent-{backend,web}`
- 镜像 tag：commit short sha（如 `bc4525a`），不可变
- 部署脚本只动 ECS 上 `/opt/multica/.env` 的 tag 行，再 `docker compose up -d`，container 重启 30 秒内完成
- 任何一次部署失败（健康检查不通过）会**自动回滚**到上一个版本，无需人工介入

---

## 二、必备权限 / 配置

要操作部署你需要：

| 项 | 说明 | 谁给你 |
|---|---|---|
| GitHub fork 仓库的 **Write 权限** | 才能 push 代码 | repo 管理员加 collaborator |
| `production` environment 的 **Reviewer 权限** | 才能 Approve deploy | repo 管理员在 Settings → Environments → production → Required reviewers 加你 |
| ECS ssh 登录权限（**可选**） | 排查问题用，正常发布不需要 | OPS |

不需要本地装 Docker、不需要 Go/Node 环境——**整个发布流程只在 GitHub 网页上点点点**。

---

## 三、最常用：发一次代码改动

### Step 1 · 推代码

```bash
# 本地仓库切到 main，确保是最新
git checkout main
git pull origin main

# 改你想改的代码（后端在 server/，前端在 apps/web 或 packages/）
# ...

# 提交并推送
git add <改动的文件>
git commit -m "fix(xxx): 描述"
git push origin main
```

> **⚠️ 不要用 `git push --force` 或 `git push -u origin main`**。直接 `git push origin main` 即可。

### Step 2 · 等 build（约 5 分钟）

打开 [Actions 页面](https://github.com/jjcc123312/multica-solveaagent/actions) 看：

- 改了 `server/**` 或 `Dockerfile` → `Build & Push Backend to GHCR` 开始跑
- 改了 `apps/web/**` 或 `packages/**` → `Build & Push Web to GHCR` 开始跑
- 都改了 → 两个并行跑

等到对应的那条 workflow 出现绿色 ✓。

### Step 3 · Approve 部署

build 成功后，**deploy workflow 会自动启动并暂停**。两种方式收到通知：

1. GitHub 网页右上角铃铛会有 "Waiting for review" 通知
2. 邮件订阅打开的话会收到邮件

点进 deploy workflow 的运行页面，**右上角有黄色 "Review deployments" 按钮**：

```
┌──────────────────────────────────────────┐
│  ⏸ Waiting for approval                  │
│                          [Review deployments] ← 点这个
└──────────────────────────────────────────┘
```

弹窗里：
- 勾选 `production`
- 可选写一句 comment（"修复 OSS 上传问题" 之类）
- 点 **Approve and deploy**

部署开始，约 1-2 分钟完成。日志里你会看到：

```
==> Previous: bc4525a, New: a1b2c3d
==> Pulling backend image
==> Recreating backend container
==> Smoke test /readyz (max 60s)
✓ Healthy after 5s — deploy succeeded (a1b2c3d)
```

### Step 4 · 验证

打开 https://你的域名/，做一次冒烟测试（登录、发个 issue、传个文件什么的）。

---

## 四、紧急回滚（部署后发现问题）

> 如果是 deploy workflow 自己烟测失败，**已经自动回滚了**，你只需要修代码再 push 一次。
>
> 如果是 deploy 成功但**线上行为不对**（比如逻辑 bug、数据问题），手动回滚：

### 方法 A · 用 GitHub Actions 部署历史版本（推荐）

1. 打开 [Deploy Backend to ECS](https://github.com/jjcc123312/multica-solveaagent/actions/workflows/deploy-backend.yml)（或 web）
2. 右侧 **Run workflow** 下拉
3. **Tag 输入框**填上一个已知好的 short sha（去 [Packages](https://github.com/jjcc123312?tab=packages) 找历史 tag，比如 `bc4525a`）
4. 点 **Run workflow**
5. 走正常审批流程 → Approve → 部署

GitHub Packages 里能看到所有历史镜像版本：[ghcr.io/jjcc123312/multica-solveaagent-backend](https://github.com/jjcc123312?tab=packages&repo_name=multica-solveaagent)。

### 方法 B · ssh 直接改（仅 OPS 用，应急）

```bash
ssh multica-ai-ecs-prod
cd /opt/multica

# 看现在的 tag
grep MULTICA_BACKEND_TAG .env

# 看历史备份（每次部署都会留 .env.bak.<时间戳>）
ls -lt .env.bak.* | head -5

# 用某个备份恢复
cp .env.bak.1778224299 .env
docker compose up -d backend
```

---

## 五、其他常见操作

### 部署一个不在 main 上的特定 commit

GitHub Actions UI 不支持这个——必须先把 commit 推到 main 才会触发 build。

**正常流程**：用 PR/分支测试好再合 main。**不要**为了部署某个版本而绕过 main。

### 部署没改代码（重放当前版本）

[Run workflow] → tag 填当前已部署的那个 sha → Approve → workflow 检测到 `Already at <tag>, nothing to do` → 0 副作用退出。这个用法主要是验证部署管道本身能不能跑。

### 跳过部署（只 build 不 deploy）

push 后 build 会照常跑。Approval 阶段不点 Approve、点 **Reject** 即可。镜像保留在 GHCR，下次想用再手动 dispatch 部署。

### 同时部署 backend 和 web

两个 deploy workflow 是独立的。如果你一次 push 同时改了 backend 和 web，两个 build 跑完后会**先后**触发两次 approval（每个服务各自一次）。两次都 Approve 才会全部上线。

如果你只想其中一个上线（比如 web 改了但只想先上 backend），第二个 deploy 点 Reject 即可。

---

## 六、查看当前生产版本

任何时候想知道生产在跑什么：

### 网页方式

打开 https://你的域名 → 浏览器 devtools → Network → 看任意 `/api/*` 请求的 `Server` 响应头，或者查首页 HTML 里的 `<meta name="version">`（如果上游有）。

### ssh 方式

```bash
ssh multica-ai-ecs-prod 'cd /opt/multica && grep -E "^MULTICA_(BACKEND|WEB)_TAG" .env'
```

输出形如：
```
MULTICA_BACKEND_TAG=bc4525a
MULTICA_WEB_TAG=c18b9c4
```

把 sha 拿到 GitHub 上对应到具体 commit：`https://github.com/jjcc123312/multica-solveaagent/commit/bc4525a`

---

## 七、故障排查

### Build 跑挂了

- 点开失败的 step 看红色错误日志
- 90% 的情况是代码语法错误 / 测试不过
- 修代码 push 即可

### Deploy approve 后失败

日志里关注这些字段：

| 错误信号 | 可能原因 | 解法 |
|---|---|---|
| `Permission denied (publickey)` | SSH key 配错 / ECS authorized_keys 被改 | 联系 OPS 检查 ECS_SSH_KEY secret 和 ECS 端公钥 |
| `Connection timed out` | ECS 公网 IP 变了 / 安全组挡了 GitHub Actions | 联系 OPS 检查 ECS_HOST secret 和阿里云安全组 |
| `Cannot connect to the Docker daemon` | ECS 上 Docker 挂了 | ssh 进去 `systemctl restart docker` |
| `manifest unknown` | 镜像没 build 出来或 tag 错了 | 去 GHCR 检查 packages tab，重跑 build |
| `✗ /readyz failed after 60s, rolling back` | 新版本启动失败、健康检查不过 | 已自动回滚，去 ECS 看 `docker compose logs backend` 找 panic / migration 错误 |

### Approve 按钮看不见

- 你不在 `production` environment 的 Required reviewers 里 → 找 repo 管理员加
- 你已经在评审了但页面缓存 → 刷新一下

### 部署成功但页面 502

ALB 健康检查可能还没标 healthy（容器才起来 30 秒内）。**等 1 分钟自动恢复**。如果持续 502，ssh 检查：

```bash
ssh multica-ai-ecs-prod 'cd /opt/multica && docker compose ps && docker compose logs --tail=50 backend frontend'
```

---

## 八、设计原则（为什么是这样的）

了解一下方便你以后改进：

- **审批门禁不可绕过**：production environment 配了 Required reviewers，连 repo 管理员都没法跳过 Approve（除非自己删掉规则，那需要单独的设置权限）。
- **Tag 用 commit sha**：不可变 + 可追溯。生产任何时候在跑的版本都能精确对应到一个 git commit。
- **自动回滚兜底**：60 秒 `/readyz` 探活失败 = 自动回到上一个版本。不依赖人 24×7 盯着。
- **每次部署留 .env 备份**：`/opt/multica/.env.bak.<timestamp>`，应急时永远能 cp 回去。
- **两个服务独立部署**：backend 改动不会重启 frontend，反之亦然。最小化用户感知。

---

## 九、常用链接

- [Actions](https://github.com/jjcc123312/multica-solveaagent/actions) — 看所有 workflow 运行
- [Deploy Backend](https://github.com/jjcc123312/multica-solveaagent/actions/workflows/deploy-backend.yml) — 手动触发后端部署
- [Deploy Web](https://github.com/jjcc123312/multica-solveaagent/actions/workflows/deploy-web.yml) — 手动触发前端部署
- [Packages](https://github.com/jjcc123312?tab=packages&repo_name=multica-solveaagent) — 镜像仓库（看历史版本）
- [Settings → Environments → production](https://github.com/jjcc123312/multica-solveaagent/settings/environments) — 改审批人 / 分支白名单
- [上游 multica-ai/multica](https://github.com/multica-ai/multica) — 同步上游用

---

## 十、联系方式

| 问题类型 | 找谁 |
|---|---|
| Code 改动 / 评审 | 团队群 |
| 部署失败 / 紧急回滚 | @jjcc123312 |
| ECS / OSS / RDS / ALB 故障 | OPS |
| GitHub Actions 配额耗尽 | repo 管理员 |

---

_最后更新：2026-05-08_
