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

### 2.5 `wqc-server-dell` 的额外硬约束（决定 runner 形态，全部实测）

> 🔴 **【本节结论已被推翻，见 §8 修订记录 2026-09-13 第二条】**
> 下面「helper 镜像拉不到 → docker executor 不可用」的判断**是错的**，源于我用错了 tag（写成
> `x86_64-19.3.2`，缺 `v` 前缀）。正确 tag `gitlab/gitlab-runner-helper:x86_64-v19.3.2` **可正常拉取**。
> 因此 docker executor **可行**，§3.2 的 shell executor 方案**作废**。
> 事实性内容（sudo 不可用、Docker Hub 直连不可达、CA 未进系统信任库、`Linger=yes`）仍然成立并有效。
> 待重写的章节：§3.2（runner 形态）、§3.4（镜像策略）、§3.5（嵌套克隆）、§4（实施顺序）。

| 约束 | 实测 | 后果 |
| --- | --- | --- |
| `sudo -n` 不可用（需密码） | `sudo -n true` 失败 | **不能 apt 安装 gitlab-runner、不能装 CA 到系统信任库、不能建 systemd 系统服务** |
| Docker Hub 不可达，仅镜像源 | `/etc/docker/daemon.json` 三个 registry-mirrors；`registry-1.docker.io` 超时 | — |
| `gitlab/gitlab-runner` 镜像拉不到 | 直连与 `docker.1ms.run/` 均 FAIL | **不能跑容器化 runner** |
| `gitlab/gitlab-runner-helper` 镜像拉不到 | 同上（两个源都 FAIL） | 🔴 **docker executor 根本不可用**（helper 镜像缺失） |
| Caddy 内部 CA 未进系统信任库 | `curl https://gitlab.ts.huntercat.cn` → `(60) unable to get local issuer certificate` | runner 与 job 都必须显式拿到 CA |
| `Linger=yes`、`systemctl --user` 正常 | `loginctl show-user weilai` → `Linger=yes`；`systemctl --user is-system-running` → `running` | ✅ **可以跑用户级 systemd 服务**（免 sudo 且开机自启） |
| 已有可复用镜像 | `maven:3.9-eclipse-temurin-21`、`node:22-slim` 已在本机 | 仅在选 docker executor 时有意义（已排除） |

**结论**：docker executor 在此宿主上不可行（无 helper 镜像 + 无 sudo + CA 不受信）。runner 形态必须改为 **shell executor + 非 root 用户级服务 + `$HOME` 内工具链**。

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

### 3.2 runner 供给（🟠 本节作废，待重写）

> ⚠️ **本节基于 §2.5 的错误结论，已作废。** 实际采用用户已建好的 group runner id=10
> `java-maven (wqc-server-dell · docker)`（docker executor、tag `java,wqc`）。
> 重写前不要按本节施工。

在 `wqc-server-dell` 上以 **非 root 用户 `weilai`** 运行 **shell executor** 的 **group runner**（group = `huntercat-digital`），**带 tag `wqc-shell`**：

- 二进制：从官方下载 `gitlab-runner-linux-amd64` **v19.3.1**（与本实例 GitLab 版本对齐；S3 源实测 200 可达）落到 `~/.local/bin/gitlab-runner`，无需 sudo
- 服务：**用户级 systemd**（`~/.config/systemd/user/gitlab-runner.service` + `systemctl --user enable --now`）—— `Linger=yes` 已实测，开机自启、崩溃自恢复。这是避开「nohup 进程静默死掉」这个刚发生过的故障模式的关键
- 工具链：全部装在 `$HOME`，不碰系统目录（无 sudo）
  - JDK 21：Adoptium `OpenJDK21U-jdk_x64_linux_hotspot_21.0.12.1_1.tar.gz` → `~/.ci/tools/jdk-21`
  - Maven：`apache-maven-3.9.16-bin.tar.gz` → `~/.ci/tools/maven`
  - pnpm：宿主机已有 node v22.23.2 + corepack，用 `corepack prepare pnpm@9.12.0`（无需装）
  - CA：把 Caddy 根证书放 `~/.ci/certs/caddy-root.crt`，在 runner 配置的 `environment` 里注入 `GIT_SSL_CAINFO`（shell executor 下 git 与 maven 都认它）
- 并发：`concurrent = 2`

**为什么 group runner + tag 而不是 project runner**：带 tag 的 runner **只接取显式声明该 tag 的 job**，因此本仓之外的 `admin-web` / `desktop`（job 无 tag）**不会被误接**，blast radius 为 0；将来给它们的 job 加 `tags: [wqc-shell]` 即可逐步接入 —— 一次注册解决三个仓的 CI 死局，且不强制改变别人的行为。

**为什么 shell executor 而不是 docker executor**：见 2.5 —— docker executor 在本宿主上因缺 helper 镜像、缺 sudo、CA 不受信而**不可用**。shell executor 反而更匹配本组既有形态（`dwjk-demo` / `legalmind-demo` 的 runner 都是 shell executor）。本仓三个 job 都不需要容器（`release.sh` 只产 jar + 拷产物，不构建镜像），shell 足够。

runner 创建凭据通过 Rails console 生成（不铸造 `api` 全权 PAT），认证 token 只落在 runner 宿主机的 `config.toml`（600），不进仓库、不回显。

### 3.3 Auto DevOps（已确认关闭，无需操作）

实测：`project.auto_devops_enabled?` → **false**；`ProjectAutoDevops(id:28, enabled: false)` 自 2026-09-06 04:00 起存在。实例级默认仍为 `true`，但项目已显式覆盖。

pipeline 66/67 里那些 `code_quality / build / test / secret_detection / semgrep-sast / container_scanning` job 产生于 09-06 02:01（项目创建时刻，早于该覆盖记录 04:00），因此是**一次性历史噪音，不会重现**。

→ 本节取消操作动作，只保留 §5 的验证约束（新 pipeline 不得出现 Auto DevOps job）。

### 3.4 `.gitlab-ci.yml`（由 `.github/workflows/ci.yml` 平移）

对齐 `lieshou-app-admin-web/.gitlab-ci.yml` 的既有形态（`GIT_SUBMODULE_STRATEGY: recursive`、不用 CI cache、`corepack prepare pnpm@9.12.0`），保留 backend / frontend / release 三段语义：

| stage | job | 执行环境 | 内容 |
| --- | --- | --- | --- |
| test | `backend` | shell（`wqc-shell`） | `mvn -B -ntp -f pom.xml verify` |
| test | `frontend` | shell（`wqc-shell`） | `node scripts/prepare.mjs` → 根 + 各端 `pnpm install --no-frozen-lockfile --force` → admin/desktop/mini-program `typecheck` |
| release | `release` | shell（`wqc-shell`） | `./scripts/release.sh --with-mini --with-mobile` → 传资产 → 建 GitLab Release |

- 全部 job `tags: [wqc-shell]`
- `variables` 注入 `JAVA_HOME` / `PATH` 指向 `$HOME` 工具链；`GIT_SSL_CAINFO` 指向 Caddy 根证书
- `rules`：`merge_request_event` / `main` / `$CI_COMMIT_TAG`（release job 仅 `$CI_COMMIT_TAG =~ /^v/`）
- `timeout`：test 45m，release 90m

### 3.5 嵌套 submodule 的过渡处理

shell executor **不是容器**，job 直接跑在宿主 `weilai` 用户下，因此克隆凭据来自该用户的 SSH 环境（`~/.ssh/id_ed25519_gitlab` 已实测可免密 clone `huntercat-digital/*`），context 与现状一致：

- **同实例（GitLab）嵌套 submodule**：走 SSH，用户在宿主已有密钥 → 直接可用（不需要 job-token 改写）
- **GitHub 嵌套 submodule**（25 条 `git@github.com:`）：那 10 个仓是公开仓，但仍需该宿主机对 GitHub 的 SSH 可达性；为免依赖，在 runner 配置的 `pre_clone_script` 里做一次匿名 HTTPS 改写：

  ```bash
  git config --global url."https://github.com/".insteadOf "git@github.com:"
  ```

  ⚠️ **这条不能写在 `before_script` 里** —— runner 的 submodule 克隆发生在 job 脚本之前的准备阶段。shell executor 支持 `pre_clone_script`（**首选**）；备选是 job 里 `GIT_SUBMODULE_STRATEGY: none` + 自己在 script 里先 `git config` 再手动 `git submodule update --init --recursive`。

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
| 1 | 在 `wqc-server-dell` 装 `$HOME` 工具链（JDK21 / Maven / pnpm）+ 放 CA | `java -version` 显示 21、`mvn -v`、`pnpm -v` 均可用；`GIT_SSL_CAINFO=... git ls-remote https://gitlab.ts.huntercat.cn/...` 成功 |
| 2 | 装 gitlab-runner v19.3.1 二进制 + 用户级 systemd 服务 | `systemctl --user is-active gitlab-runner` → `active`；`systemctl --user is-enabled` → `enabled` |
| 3 | 创建 group runner（tag `wqc-shell`, `run_untagged=false`）并注册 | runner 出现在 group runner 列表且 online；token 不以明文入库 |
| 4 | **冒烟 job**：递归 submodule + 打印 `java/mvn/node/pnpm` 版本 | 递归 submodule 全部拉通（含嵌套 GitLab + 嵌套 GitHub）；版本符合预期 |
| 5 | 提交 `.gitmodules` URL 迁移（pin 不变） | `git submodule status` 全部就位且 SHA 与迁移前逐一相同 |
| 6 | 提交 `.gitlab-ci.yml`（backend + frontend） | main 上两个 job 绿，且 pipeline 内**无 Auto DevOps job** |
| 7 | 加 release job | 打 `v0.1.1` 标签后 Release 资产可下载 |

冒烟门（步 4）不通过则停下修正 §3.5 的克隆策略，不带着已知失败的克隆策略往下走。

## 5. 验收标准

| # | 标准 | 证据 |
| --- | --- | --- |
| 1 | 10 条 URL 均为 `ssh://git@gitlab.ts.huntercat.cn:22220/huntercat-digital/*.git` | `node pin.mjs urls` 输出「全部为规范形态」 |
| 2 | pin 未变 | 迁移前后 `git submodule status` 的 SHA 逐一相等 |
| 3 | 递归初始化成功 | `git submodule update --init --recursive` 退出码 0；顶层 10 条 `git submodule foreach --recursive 'git remote get-url origin'` 全部为 `gitlab.ts.huntercat.cn` 主机；嵌套 GitHub 层按 3.5 走匿名 HTTPS（其 URL 形态保持现状，属本次有意保留的债务） |
| 4 | 流水线真跑 | GitLab pipeline 上 `backend`、`frontend` job 状态 `success`，非 `stuck_pending` |
| 5 | Auto DevOps 未混入 | 新 pipeline 的 job 列表只含 `.gitlab-ci.yml` 定义的 job，无 `code_quality` / `semgrep-sast` 等 |
| 6 | Release 可下载 | 打 tag 后 GitLab Release 页面有资产链接且 `curl` 可下载 |

「跑通」的声明以 GitLab pipeline 页面 URL + job 状态为准。

## 6. 风险与回滚

| 风险 | 影响 | 回滚 |
| --- | --- | --- |
| 非 root 用户级 runner 在重启后不自启 | CI 静默失效（= 刚复现过的故障模式） | 已用 `systemctl --user` + `Linger=yes` 规避；实施后用「重启宿主验证 runner 自动上线」作为独立验收 |
| GitHub 匿名 HTTPS 被限流/不可用 | frontend/backend job 失败 | 并入「嵌套 25 条迁 GitLab」批次，或自带 token |
| `wqc-server-dell` 跑构建抢资源 | 中枢上其他服务受影响 | 并发设 2 + job timeout；必要时迁 dwjk-demo（已有 systemd 级 runner） |
| `.gitmodules` 改动被误当作 pin 变更 | 可重现性受损 | 本次提交只含 URL 行；用「pin 逐一相等」证据自查 |
| `$HOME` 工具链被其他操作弄脏 | job 随机失败 | 全部路径收敛在 `~/.ci/` 单一目录，可整体删除重建 |
| 构建产物/工作目录占满 `/` | 中枢磁盘告警 | runner `builds_dir` 指到独立目录并纳入清理；当前 77G 可用 |

**整体回滚**：`git revert` 对应提交即可（URL 与 CI 是两个独立提交，可分别回退）；runner 在 GitLab 侧禁用/删除，不影响仓库内容。

## 7. 未决项

1. **嵌套 25 条**是否立项迁 GitLab（跨 10 个上游仓的 MR）—— 本次明确不做
2. 是否把 `admin-web` / `desktop` 的 job 加 `tags: [wqc-shell]` 接入新 runner —— 本次不主动改他人仓
3. `$HOME` 工具链的版本升级策略（本次硬编码 JDK 21.0.12.1+1 / Maven 3.9.16 / pnpm 9.12.0）—— 后续可收敛成一个 `~/.ci/setup.sh`

## 8. 修订记录

| 日期 | 变更 | 原因 |
| --- | --- | --- |
| 2026-09-13 | runner 形态由 **docker executor** 改为 **shell executor + 非 root 用户级 systemd** | 实测：`wqc-server-dell` 上 `gitlab/gitlab-runner` 与 helper 镜像均拉不到（Docker Hub 被墙且镜像源无此镜像）、`sudo -n` 不可用、Caddy CA 未进系统信任库 → docker executor 不可行 |
| 2026-09-13 | §3.3 由「关闭 Auto DevOps」改为「已确认关闭」 | 实测 `project.auto_devops_enabled? == false`，pipeline 66/67 的 Auto DevOps job 是 09-06 项目创建时刻的一次性历史噪音 |
| 2026-09-13 | §7 未决项 1（release job 镜像）关闭 | 改用 shell executor 后工具链统一由 `$HOME` 提供，不再需要镜像选型 |
| 2026-09-13 | **上述两条 runner 形态变更全部作废** | 实测发现上一条的「helper 镜像拉不到」是 **tag 写法错误**（漏 `v` 前缀）所致：`gitlab/gitlab-runner-helper:x86_64-v19.3.2` 可正常拉取。docker executor 可行。另：用户已自行建好 group runner id=10（docker executor，tag `java,wqc`，`maven:3.9-eclipse-temurin-21` 镜像，CA 走 `tls-ca-file`，`RestartPolicy=always`） |
| 2026-09-13 | §2.5 事实层修正 | Docker Hub **直连**仍不可达（`registry-1.docker.io` 超时）、需走 registry-mirrors，但镜像源能正常供给所需的 `gitlab-runner` / `gitlab-runner-helper` / `maven` / `node` 镜像 → 「拉不到镜像」不成立 |
