# lieshou-standalone · GitLab 供应链化 + CI 迁移 设计

- **日期**：2026-09-13
- **状态**：待评审
- **范围决策**：用户选定「本仓 submodule URL 换成 GitLab 供应链仓」+「CI 从 GitHub Actions 迁到 GitLab CI」（同批）
- **runner 宿主决策**：用户选定 `wqc-server-dell` (100.64.0.22)
- **过程记录**：`TODO.md`（本仓根，未跟踪的工作记录）

## 1. 目标

把 `huntercat-digital/lieshou-standalone` 的依赖来源与流水线从 **GitHub 供应链 + GitHub Actions** 切成 **GitLab 供应链 + GitLab CI**，并保证：

- **不动 pin**：现有 10 个 submodule pin 原样保留，可重现性不变
- **不手改 submodule 内容**：端仓/契约仓的改动不经本仓
- CI 在本仓真的能跑起来（不是「配了不跑」）

### 非目标

- 不复活时代一的业务形态，不追平上游 pin（framework 落后 61 commits、admin 落后 269）
- 不把 `apps/*`、`open/*` 各自的 25 条嵌套 submodule URL 迁移到 GitLab（独立批次）
- 不构建 desktop **安装包**（需 Rust toolchain）。desktop 的 **typecheck 保留**，与现 GitHub workflow 一致

## 2. 现状（全部实测，2026-09-13）

### 2.1 引用面

全仓引用 GitHub 的位置**仅 `.gitmodules` 10 行**，别处（脚本/CI/文档/代码）零引用。

10 个 submodule 的目标仓在 GitLab 均存在，且**现 pin 的 SHA 在 GitLab 均存在**（逐个 GitLab API 校验通过）→ 换 URL 不影响 pin。

### 2.2 CI 现状

- `.github/workflows/ci.yml` 有 3 个 job：`backend`（`mvn -f pom.xml verify`）、`frontend`（`prepare.mjs` → 根 + 各端 pnpm install → 三个端 typecheck）、`release`（tag `v*` → `release.sh --with-mini --with-mobile` → `softprops/action-gh-release` 上传 `dist/lieshouboot-release/**`）
- 原注释记录了关键结论：**不用 CI cache**（self-hosted runner 缓存上传目标 Azure Blob 不通，会 0 MB/s 挂死；runner 本地已有全局 pnpm store）
- GitLab 侧**无 `.gitlab-ci.yml`**，但本项目 **Auto DevOps 是开启的** → pipeline 66/67 是 Auto DevOps 的 `build / test / code_quality / secret_detection / semgrep-sast / container_scanning`

### 2.3 runner 供给（本次的真正阻塞）

- 本项目（id 47）的 pipeline 66/67 **全部** `stuck_pending_no_matching_runners`
- 全实例 6 个 runner：唯一的 group runner 是 `imacos (desktop-macos)`（tag `desktop-macos`，mac）；其余 4 个是 **project_type**，项目集合里不含本项目；`wxf-dev mobile-web shell` 自 2026-09-09 起失联（wxf 区域退役中）
- **旁证**：`lieshou-app-admin-web`、`lieshou-app-desktop` 的 GitLab CI 现在同为 `stuck_pending_no_matching_runners` —— 它们的 `.gitlab-ci.yml` 从未真正跑起来

候选宿主实测：

| 机器 | CPU | RAM 可用 | 磁盘可用 | gitlab-runner | JDK21 | docker |
| --- | --- | --- | --- | --- | --- | --- |
| **`wqc-server-dell` 100.64.0.22** | 20 | 35G | 77G | ❌ | ❌ | ✅ 29.8 |
| `dwjk-demo` 100.64.0.24 | 8 | 11G | 42G | ✅ 19.3.1 | ❌ | ✅ |
| `legalmind-demo` 100.64.0.27 | 4 | 5G | 78G | ✅ 19.3.2 | ❌ | ✅ |
| 本机 `ubt-ofc-weilai` 100.64.0.30 | 2 | ~2G | 19G | ❌ | ✅ | ❌ |

### 2.4 嵌套 submodule

`apps/*` 与 `open/*` 各自的 `.gitmodules` 共 **25 条**仍指 GitHub（端仓 5×4 + contract-api 1 + contract-types 1 + ui 3）。
这 10 个 GitHub 上游仓**是公开仓**（未认证调 GitHub API 返回 200 可证）。

## 3. 设计

### 3.1 submodule URL 迁移（低风险）

用既有规范工具（13 个交付仓同款），不手写 sed：

```bash
node ~/dev/gitlab-mirror/huntercat-ai-kit/tools/pin.mjs urls --fix   # 在本仓根目录执行
```

目标形态：`ssh://git@gitlab.ts.huntercat.cn:22220/huntercat-digital/<repo>.git`
（该工具已内置两个真实坑：`:2222 → :22220` 只换端口不补路径段；`huntercat-digital/huntercat-digital/` 去重）

改完执行 `git submodule sync --recursive`，使 `.git/config` 与 `.gitmodules` 一致。

### 3.2 runner 供给（新增）

在 `wqc-server-dell` 注册 **docker executor** 的 **group runner**（group = `huntercat-digital`），**带 tag**：

- tags：`wqc-docker`
- 并发：`concurrent = 2`，单 job 资源上限保守
- `pull_policy = if-not-present`

**为什么 group runner + tag 而不是 project runner**：带 tag 的 runner **只接取显式声明该 tag 的 job**，因此本仓之外的 `admin-web` / `desktop`（job 无 tag）**不会被误接**，blast radius 为 0；将来给它们的 job 加 `tags: [wqc-docker]` 即可逐步接入 —— 一次注册解决三个仓的 CI 死局，且不强制改变别人的行为。

**为什么 docker executor**：宿主无需安装 JDK/mvn/pnpm，工具链全部来自官方镜像；`wqc-server-dell` 是既定的 CI/CD 中枢且资源最足（20C/35G/77G）。

runner 注册凭据通过 GitLab API/rails console 生成（本机已授权自取 PAT / 走 Rails），token 只落在 runner 宿主的 `config.toml`（600），不进仓库、不回显。

### 3.3 关闭 Auto DevOps（前置）

对本项目设 `auto_devops_enabled = false`，否则新 `.gitlab-ci.yml` 会与 Auto DevOps 的 job 混跑出噪音。

### 3.4 `.gitlab-ci.yml`（由 `.github/workflows/ci.yml` 平移）

对齐 `lieshou-app-admin-web/.gitlab-ci.yml` 的既有形态（`GIT_SUBMODULE_STRATEGY: recursive`、不用 CI cache、`corepack prepare pnpm@9.12.0`），保留 backend / frontend / release 三段语义：

| stage | job | 镜像 | 内容 |
| --- | --- | --- | --- |
| test | `backend` | `maven:3.9-eclipse-temurin-21` | `mvn -B -ntp -f pom.xml verify` |
| test | `frontend` | `node:22-bookworm` | `node scripts/prepare.mjs` → 根 + 各端 `pnpm install --no-frozen-lockfile --force` → admin/desktop/mini-program `typecheck` |
| release | `release` | 待定（需 node+pnpm+mvn+JDK21，见 3.6） | `./scripts/release.sh --with-mini --with-mobile` → 传资产 → 建 GitLab Release |

- 全部 job `tags: [wqc-docker]`
- `rules`：`merge_request_event` / `main` / `$CI_COMMIT_TAG`（release job 仅 `$CI_COMMIT_TAG =~ /^v/`）
- `timeout`：test 45m，release 90m

### 3.5 嵌套 submodule 的过渡处理

docker executor 的容器内**没有宿主的 SSH 密钥**，因此：

- **同实例（GitLab）嵌套 submodule**：依赖 GitLab Runner 用 `CI_JOB_TOKEN` 改写认证。**该项必须实测确认**；若不成立，走下面的 pre_clone 注入。
- **GitHub 嵌套 submodule**（25 条 `git@github.com:`）：那 10 个仓是公开仓，故可用匿名 HTTPS 顶掉 SSH：

  ```bash
  git config --global url."https://github.com/".insteadOf "git@github.com:"
  ```

  ⚠️ **这条不能写在 `before_script` 里** —— runner 的 submodule 克隆发生在 job 脚本之前的准备阶段。落点只有两个：
  1. runner `config.toml` 的 `pre_clone_script`（docker executor 支持，**首选**）
  2. job 里 `GIT_SUBMODULE_STRATEGY: none` + 自己在 script 里先 `git config` 再手动 `git submodule update --init --recursive`（备选，确定性更强但丢失 runner 的同实例 token 改写）

  代价：CI 仍依赖 GitHub 可用性。彻底 GitLab 化必须做那 25 条（独立批次）。

### 3.6 release 资产

GitHub Releases → GitLab Release：

1. `release.sh --with-mini --with-mobile` 产出 `dist/lieshouboot-release/**`
2. 以 Generic Package 上传：`POST $CI_API_V4_URL/projects/$CI_PROJECT_ID/packages/generic/lieshouboot/$CI_COMMIT_TAG/<file>`，凭 `JOB-TOKEN: $CI_JOB_TOKEN`
3. 用 `release-cli`（或 `POST .../releases`）建 Release 并把 asset link 指向上面的 package URL

release job 的工具链（需 node+pnpm+mvn+JDK21 同容器）：**首选**自建镜像（推 `wqc-server-dell` 上的 Harbor，与中枢一致），**备选**用 `maven:3.9-eclipse-temurin-21` 在 `before_script` 里装 node+corepack。此二选一在实施阶段按实测结果定，不阻塞前两段。

## 4. 实施顺序（含冒烟门）

| 步 | 动作 | 门 |
| --- | --- | --- |
| 0 | 关 Auto DevOps | pipeline 不再出现 Auto DevOps job |
| 1 | 注册 group runner `wqc-docker` + `pre_clone_script` | runner 出现在 group 列表且 online |
| 2 | **冒烟 job**：`git submodule update --init --recursive` + `mvn -v` + `node -v` + `pnpm -v` | 递归 submodule 全部拉通（含嵌套 GitLab + 嵌套 GitHub）；三者版本符合预期 |
| 3 | 提交 `.gitmodules` URL 迁移（pin 不变） | `git submodule status` 全部就位且 SHA 与迁移前逐一相同 |
| 4 | 提交 `.gitlab-ci.yml`（backend + frontend） | main 上两个 job 绿 |
| 5 | 加 release job | 打 `v0.1.1` 标签后 Release 资产可下载 |

冒烟门（步 2）不通过则停下修正 3.5 的方案，不带着已知失败的克隆策略往下走。

## 5. 验收标准

| # | 标准 | 证据 |
| --- | --- | --- |
| 1 | 10 条 URL 均为 `ssh://git@gitlab.ts.huntercat.cn:22220/huntercat-digital/*.git` | `node pin.mjs urls` 输出「全部为规范形态」 |
| 2 | pin 未变 | 迁移前后 `git submodule status` 的 SHA 逐一相等 |
| 3 | 递归初始化成功 | `git submodule update --init --recursive` 退出码 0；顶层 10 条 `git submodule foreach --recursive 'git remote get-url origin'` 全部为 `gitlab.ts.huntercat.cn` 主机；嵌套 GitHub 层按 3.5 走匿名 HTTPS（其 URL 形态保持现状，属本次有意保留的债务） |
| 4 | 流水线真跑 | GitLab pipeline 上 `backend`、`frontend` job 状态 `success`，非 `stuck_pending` |
| 5 | Auto DevOps 已关 | 项目设置 `auto_devops_enabled=false`，pipeline 内无 Auto DevOps job |
| 6 | Release 可下载 | 打 tag 后 GitLab Release 页面有资产链接且 `curl` 可下载 |

「跑通」的声明以 GitLab pipeline 页面 URL + job 状态为准。

## 6. 风险与回滚

| 风险 | 影响 | 回滚 |
| --- | --- | --- |
| 同实例嵌套 submodule 的 job token 认证不成立 | 冒烟门拦住，不往下走 | 改用 `pre_clone_script` 注入 SSH 私钥（file 型 CI 变量） |
| GitHub 匿名 HTTPS 被限流/不可用 | frontend/backend job 失败 | 并入「嵌套 25 条迁 GitLab」批次，或自带 token |
| wqc-server-dell 跑 docker executor 抢资源 | 中枢上其他服务受影响 | 并发设 2 + job timeout；必要时迁 dwjk-demo |
| `.gitmodules` 改动被误当作 pin 变更 | 可重现性受损 | 本次提交只含 URL 行；用「pin 逐一相等」证据自查 |
| Auto DevOps 关闭影响本项目其他能力 | 低 | 单项目设置，可一键开回 |

**整体回滚**：`git revert` 对应提交即可（URL 与 CI 是两个独立提交，可分别回退）；runner 在 GitLab 侧禁用/删除，不影响仓库内容。

## 7. 未决项

1. **release job 镜像**：自建（Harbor）vs `maven` 镜像内装 node —— 实施阶段定，不阻塞步 0–4
2. **嵌套 25 条**是否立项迁 GitLab（跨 10 个上游仓的 MR）—— 本次明确不做
3. 是否把 `admin-web` / `desktop` 的 job 加 `tags: [wqc-docker]` 接入新 runner —— 本次不主动改他人仓
