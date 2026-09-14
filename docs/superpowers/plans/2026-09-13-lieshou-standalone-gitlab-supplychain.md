# lieshou-standalone GitLab 供应链化 + CI 迁移 实现计划（v2 · 按 runner 10 实况重写）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 `huntercat-digital/lieshou-standalone` 的 CI 从 GitHub Actions 迁到 GitLab CI（跑在用户已建好的 runner 10 上）并让它在流水线上真正跑绿；同时把本仓 10 条顶层 submodule URL 从 GitHub 迁到 GitLab 供应链仓，pin 不变。

**Architecture:** 两轨。**轨一（本计划）**：给 runner 10 加 `pre_clone_script` 让 docker 容器内的 git 能取到依赖（GitHub 走匿名 HTTPS），然后落地 `.gitlab-ci.yml` + `.gitmodules` 迁移，用冒烟门逐步验证。**轨二（推迟，见文末附录）**：把 38 条嵌套 submodule URL 也 GitLab 化 —— 经实测它需要 14 个跨历史 SHA 的分叉提交 + 三级 pin 级联，是独立工程。

**Tech Stack:** GitLab CE 19.3.1、gitlab-runner v19.3.2（**docker executor**，容器名 `gitlab-runner`）、`maven:3.9-eclipse-temurin-21`（含 git 2.43）、`node:22-bookworm`（含 git 2.39 + corepack 0.34.6）、GitLab Generic Packages API。

**Spec:** `docs/superpowers/specs/2026-09-13-lieshou-standalone-gitlab-supplychain-design.md`

---

## 执行前必读（全部已实测，不要再重新探索）

| 事实 | 值 |
| --- | --- |
| runner 宿主 | `wqc-server-dell` = `100.64.0.22`（>`ssh weilai@100.64.0.22`） |
| GitLab | 容器 `gitlab`；用 `docker exec gitlab gitlab-rails runner <file>` 操作 |
| **runner 10** | 容器名 `gitlab-runner`（`gitlab/gitlab-runner:latest`，v19.3.2），**已注册且 online** |
| runner 10 tags | **`java`, `wqc`**（`run_untagged=false`）→ job 必须写 `tags: [wqc]` |
| runner 10 默认镜像 | `maven:3.9-eclipse-temurin-21`（本机已有，含 git） |
| runner 10 已配 | `tls-ca-file=/etc/gitlab-runner/certs/caddy-root.crt`、`GIT_SSL_NO_VERIFY=true`、`extra_hosts` 映射 GitLab、`volumes=["/srv/gitlab-runner-cache/m2:/root/.m2","/cache"]`、`RestartPolicy=always`、`concurrent=1` |
| ⚠️ runner 10 **未配** | `pre_clone_script` → 容器内无 GitHub SSH 密钥，嵌套 GitHub submodule 会挂（这就是 Task 1 要补的） |
| config.toml 路径 | 宿主 `/srv/gitlab-runner/config/config.toml`（root:root 600，**weilai 不能直接改**）→ 走 `docker cp`（weilai 在 docker 组） |
| `sudo` | 🔴 不可用（需密码）。一切宿主侧操作只能走 docker |
| Docker Hub | 直连不可达，走 registry-mirrors；已缓存：`gitlab-runner`、`gitlab-runner-helper:x86_64-v19.3.2`、`maven:3.9-eclipse-temurin-21`、`node:22-bookworm` |
| 项目 id | `lieshou-standalone` = **47** |
| Auto DevOps | **已关**（`auto_devops_enabled? == false`）。pipeline 66/67 的 Auto DevOps job 是 09-06 一次性历史噪音，不会重现 |

**本仓 pin 基线（Task 3 前后必须逐一相等）：**

| path | pin SHA |
| --- | --- |
| `apps/admin` | `aa93a5b33e3144e790a81c390c1082e3beb3493d` |
| `apps/desktop` | `d25e50136d6967ba52e2a29dc524501a81cbd13e` |
| `apps/mobile` | `f5f72db16a6b0a5bc8f7e8d81605d4e5256798ba` |
| `apps/mini-program` | `11e2c92052e80a048ff0cc3d606e6e0709971da6` |
| `framework` | `899319579ce2e45493ee5258f56a77684ffd5545` |
| `open/contract-api` | `a8f8e611f24b51fc16c079e4ddc3a138a3e79f46` |
| `open/contract-config` | `425f91209874ec0f7ae67548650a1de4d1a83e5c` |
| `open/contract-types` | `93a2d49e2310fd064a81488985ad0510f3e22de5` |
| `open/core-web` | `d32dd57d82fbac106680ebcd26e2cf3d3629b5a9` |
| `open/ui` | `6da4b09316345578e0593fc1462254ffc634c4f7` |

---

## 文件结构

| 文件 | 动作 | 职责 |
| --- | --- | --- |
| `/srv/gitlab-runner/config/config.toml` | 在 dell 上修改（经 `docker cp`） | 加 `pre_clone_script`：容器内把依赖仓库的 SSH URL 改写成可取到的 HTTPS |
| `.gitlab-ci.yml` | 创建（本仓） | 三个 job：`backend` / `frontend` / `release` |
| `.gitmodules` | 修改（本仓） | 10 条顶层 URL：GitHub → GitLab 供应链仓 |
| `docs/superpowers/specs/...` | 修改（本仓） | 修正已作废章节 + 记录实施证据 |
| `README.md` | 修改（本仓） | 同步 CI 平台与发布资产描述 |

**提交划分（可独立回退）：** Task 2 的 `.gitlab-ci.yml`、Task 3 的 `.gitmodules`、Task 6 的文档各为独立提交。

---

## Task 0: 记录 pin 基线

- [ ] **Step 1: 记录当前 submodule 状态**

```bash
cd ~/dev/gitlab-mirror/lieshou-standalone
git submodule status | sed 's/^-//' | awk '{print $2, $1}' | sort > /tmp/lieshou-pins-before.txt
cat /tmp/lieshou-pins-before.txt
```

Expected：10 行，与本文档「本仓 pin 基线」表逐字相同。

- [ ] **Step 2: 确认工作区无脏改动**

Run: `git status --porcelain`
Expected: 只有 `?? TODO.md`。有其它输出先处理干净。

---

## Task 1: 给 runner 10 加 `pre_clone_script`

**Files:** Modify `/srv/gitlab-runner/config/config.toml`（dell，经 `docker cp`）

**为什么需要：** docker executor 的 job 容器里没有 GitHub SSH 密钥，`GIT_SUBMODULE_STRATEGY: recursive` 拉 `apps/*/open/*`（25 条 `git@github.com:`）会 permission denied。那 10 个 GitHub 仓**是公开仓**，改走匿名 HTTPS 即可。

- [ ] **Step 1: 导出当前 config 并确认 token 行存在（不要丢）**

```bash
ssh weilai@100.64.0.22 'docker exec gitlab-runner cat /etc/gitlab-runner/config.toml > /tmp/runner-config.toml && \
  grep -c "^  token = " /tmp/runner-config.toml && grep -c "executor = \"docker\"" /tmp/runner-config.toml'
```

Expected: `1` 与 `1`（各恰好一行）
⚠️ 后续输出**不要**打印 `token =` 那一行的值。

- [ ] **Step 2: 插入 `pre_clone_script`（只加一行，其余不动）**

```bash
ssh weilai@100.64.0.22 'cp /tmp/runner-config.toml /tmp/runner-config.toml.orig && python3 - <<'"'"'PY'"'"'
import pathlib
p = pathlib.Path("/tmp/runner-config.toml")
t = p.read_text()
if "pre_clone_script" in t:
    print("已存在，跳过"); raise SystemExit
needle = "executor = \"docker\""
assert t.count(needle) == 1, "executor 行不唯一，停止"
inject = needle + "\n  pre_clone_script = \"git config --global url.\\\"https://github.com/\\\".insteadOf \\\"git@github.com:\\\"\""
p.write_text(t.replace(needle, inject))
print("已插入 pre_clone_script")
PY'
```

Expected: `已插入 pre_clone_script`

- [ ] **Step 3: 校验新 config 是合法 TOML 且 token 未丢**

```bash
ssh weilai@100.64.0.22 'python3 -c "
import tomllib
d = tomllib.load(open(\"/tmp/runner-config.toml\",\"rb\"))
r = d[\"runners\"][0]
print(\"executor=\", r[\"executor\"])
print(\"pre_clone_script=\", r.get(\"pre_clone_script\"))
print(\"token_present=\", bool(r.get(\"token\")))
print(\"tag_keys=\", [k for k in r if \"tag\" in k])
"'
```

Expected: `executor= docker`；`pre_clone_script= git config --global url."https://github.com/".insteadOf "git@github.com:"`；`token_present= True`

- [ ] **Step 4: 写回容器并重启**

```bash
ssh weilai@100.64.0.22 'docker cp /tmp/runner-config.toml gitlab-runner:/etc/gitlab-runner/config.toml && \
  docker restart gitlab-runner >/dev/null && sleep 12 && \
  docker ps --format "{{.Names}} {{.Status}}" | grep gitlab-runner'
```

Expected: `gitlab-runner Up ...`（seconds 级）

- [ ] **Step 5: 确认 runner 重新上线（无人工干预）**

```bash
cat > /tmp/check_runner.rb <<'RUBY'
r = Ci::Runner.find(10)
puts "id=#{r.id} active=#{r.active} online=#{r.online?} contacted=#{r.contacted_at}"
RUBY
scp -q /tmp/check_runner.rb weilai@100.64.0.22:/tmp/check_runner.rb
ssh weilai@100.64.0.22 'docker cp /tmp/check_runner.rb gitlab:/tmp/check_runner.rb >/dev/null && docker exec gitlab gitlab-rails runner /tmp/check_runner.rb'
```

Expected: `online=true`，`contacted=` 为近 2 分钟内。

- [ ] **Step 6: 记录日志无报错**

Run: `ssh weilai@100.64.0.22 'docker logs --tail 20 gitlab-runner 2>&1 | grep -iE "error|panic|invalid" || echo "无错误"'`
Expected: `无错误`

---

## Task 2: `.gitlab-ci.yml` + 冒烟门（此时 `.gitmodules` 仍是 GitHub）

**目的：** 在**不改 URL** 的前提下先证明「runner 能接单 + 递归克隆能过 + 工具链就位」。这样 Task 4 若失败，可确定归因于 URL 迁移，而不是环境。

**Files:** Create `.gitlab-ci.yml`

- [ ] **Step 1: 写入流水线**

```bash
cd ~/dev/gitlab-mirror/lieshou-standalone
cat > .gitlab-ci.yml <<'EOF'
# GitLab CI —— 由 .github/workflows/ci.yml 平移（GitHub Actions → 内网 GitLab）
# runner：wqc-server-dell 的 docker executor（tag wqc）
# ⚠️ 不用 GitLab CI cache：runner 侧已有 /srv/gitlab-runner-cache/m2 挂到 /root/.m2，
#    且历史上 self-hosted 缓存上传触发过故障（对齐端仓结论 ae0feb0）
stages:
  - test
  - release

variables:
  GIT_SUBMODULE_STRATEGY: recursive
  GIT_DEPTH: 0

default:
  tags: [wqc]

# —— 后端：framework submodule + backend 聚合构建 ——
backend:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  timeout: 45m
  script:
    - java -version
    - mvn -B -ntp -f pom.xml verify
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'

# —— 前端：自动装配 → 依赖 → 类型检查 ——
# 用 bookworm 全量镜像而非 slim：slim 里没有 git，runner 需要它做克隆/子模块
frontend:
  stage: test
  image: node:22-bookworm
  timeout: 45m
  script:
    - node -v
    - corepack prepare pnpm@9.12.0 --activate
    - node scripts/prepare.mjs
    - pnpm install --no-frozen-lockfile --force
    - |
      for app in admin desktop mobile mini-program; do
        if [ -f "apps/$app/pnpm-workspace.yaml" ]; then
          (cd "apps/$app" && pnpm install --no-frozen-lockfile --force)
        fi
      done
    - (cd apps/admin && pnpm typecheck)
    - (cd apps/desktop && pnpm typecheck)
    - (cd apps/mini-program && pnpm typecheck)
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'

# —— 组合发布（tag v* 触发；在 main 上不触发即为惰性）——
# ⚠️ maven 镜像含 JDK21 + mvn + git，但**没有 Node**（也没有 xz，故只能取 .tar.gz），下面自装
release:
  stage: release
  image: maven:3.9-eclipse-temurin-21
  timeout: 90m
  before_script:
    - curl -fsSL -o /tmp/node.tar.gz https://registry.npmmirror.com/-/binary/node/v22.23.2/node-v22.23.2-linux-x64.tar.gz
    - tar -xzf /tmp/node.tar.gz -C /usr/local --strip-components=1
    - node -v
    - corepack enable && corepack prepare pnpm@9.12.0 --activate
    - pnpm -v
  script:
    - ./scripts/release.sh --with-mini --with-mobile
    - PKG_BASE="${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/lieshouboot/${CI_COMMIT_TAG}"
    - |
      for f in dist/lieshouboot-release/backend/*.jar dist/lieshouboot-release/README.md; do
        echo "上传 $(basename "$f")"
        curl -sS --fail --header "JOB-TOKEN: ${CI_JOB_TOKEN}" --upload-file "$f" "${PKG_BASE}/$(basename "$f")"
      done
    - tar czf "lieshouboot-${CI_COMMIT_TAG}.tar.gz" -C dist lieshouboot-release
    - curl -sS --fail --header "JOB-TOKEN: ${CI_JOB_TOKEN}" --upload-file "lieshouboot-${CI_COMMIT_TAG}.tar.gz" "${PKG_BASE}/lieshouboot-${CI_COMMIT_TAG}.tar.gz"
    - |
      curl -sS --fail --request POST \
        --header "JOB-TOKEN: ${CI_JOB_TOKEN}" \
        --header "Content-Type: application/json" \
        --data "{\"name\":\"LieShouBoot ${CI_COMMIT_TAG}\",\"tag_name\":\"${CI_COMMIT_TAG}\",\"assets\":{\"links\":[{\"name\":\"组合发布包\",\"url\":\"${PKG_BASE}/lieshouboot-${CI_COMMIT_TAG}.tar.gz\",\"link_type\":\"package\"}]}}" \
        "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/releases"
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v/'
EOF
```

- [ ] **Step 2: YAML 自检**

Run: `python3 -c "import yaml;d=yaml.safe_load(open('.gitlab-ci.yml'));print([k for k in d if k not in ('stages','variables','default')])"`
Expected: `['backend', 'frontend', 'release']`

- [ ] **Step 3: 提交并推送**

```bash
git add .gitlab-ci.yml
git commit -m "ci: GitHub Actions 平移为 GitLab CI（backend + frontend + release）

- runner: wqc-server-dell docker executor（tag wqc）
- backend: maven:3.9-eclipse-temurin-21 + mvn -B -ntp -f pom.xml verify
- frontend: node:22-bookworm（slim 无 git）+ prepare.mjs + 各端 pnpm install + 三端 typecheck
- release: tag v* → release.sh --with-mini --with-mobile → Generic Package + Release API
- 发布资产由 GitHub Release 改为 GitLab Generic Package"
git push origin main
```

- [ ] **Step 4: 轮询 pipeline 到结束**

```bash
TOKEN=$(ls ~/.gitlab-pat/*.token | head -1)
P() { curl -s --max-time 25 -H "PRIVATE-TOKEN: $(cat $TOKEN)" "https://gitlab.ts.huntercat.cn/api/v4/projects/47/pipelines?per_page=1" | python3 -c "import sys,json;print(json.load(sys.stdin)[0]['$1'])"; }
for i in $(seq 1 60); do S=$(P status); echo "[$i] $S"; case "$S" in success|failed|canceled) break;; esac; sleep 20; done
echo "pipeline $(P id) $(P web_url)"
```

Expected: 最终 `success`。**若卡在 `pending`** → runner 没接单（tag 不匹配），门失败。

- [ ] **Step 5: 读日志确认四件事（冒烟门判定）**

```bash
TOKEN=$(ls ~/.gitlab-pat/*.token | head -1)
PID=$(curl -s --max-time 25 -H "PRIVATE-TOKEN: $(cat $TOKEN)" "https://gitlab.ts.huntercat.cn/api/v4/projects/47/pipelines?per_page=1" | python3 -c "import sys,json;print(json.load(sys.stdin)[0]['id'])")
curl -s --max-time 25 -H "PRIVATE-TOKEN: $(cat $TOKEN)" "https://gitlab.ts.huntercat.cn/api/v4/projects/47/pipelines/$PID/jobs" \
  | python3 -c "import sys,json;[print(j['id'],j['name'],j['status']) for j in json.load(sys.stdin)]"
# 用上面的 job id 取日志
# curl -s -H "PRIVATE-TOKEN: $(cat $TOKEN)" ".../projects/47/jobs/<id>/trace" | head -60
```

Expected：
1. job 列表就是 `backend` / `frontend`，**无** `code_quality`/`semgrep-sast` 等 Auto DevOps job
2. `backend` 里 `java -version` 显示 21、`mvn` 走到 `BUILD SUCCESS`
3. `frontend` 里 `node -v` 显示 v22、三端 typecheck 全过
4. 两个 job 都真的在 `wqc-server-dell` 上跑（日志首行有 cloning/executor 信息）

- [ ] **Step 6: 门失败时的诊断顺序（只失败时执行）**

| 症状 | 首个要查的 |
| --- | --- |
| `stuck_pending` / `pending` 不变 | runner tag 是否 `wqc`；`run_untagged=false` 是否被别处改过 |
| 克隆阶段 permission denied (github) | Task 1 的 `pre_clone_script` 是否真的生效（`docker exec gitlab-runner cat /etc/gitlab-runner/config.toml`） |
| `frontend` 找不到 git / pnpm | 镜像是否被覆盖成 slim |
| `backend` OOM | runner `concurrent=1` 下内存；临时把 `mvn` 加 `-Dmaven.test.skip=false` 前的堆参数调整 |
| release job 里 `node: not found` / `pnpm: not found` | `before_script` 自装 Node 那段是否被执行（该 job 用的 maven 镜像**不含 Node**） |

修完重跑 Step 3–5。**门未通过不要进 Task 3。**

---

## Task 3: `.gitmodules` 顶层 10 条 URL 迁到 GitLab 供应链仓

**Files:** Modify `.gitmodules`

- [ ] **Step 1: 用规范工具改 URL**

```bash
cd ~/dev/gitlab-mirror/lieshou-standalone
node ~/dev/gitlab-mirror/huntercat-ai-kit/tools/pin.mjs urls --fix
```

Expected: 提示「`.gitmodules URL：修正 10 处`」

- [ ] **Step 2: 确认形态合规**

Run: `node ~/dev/gitlab-mirror/huntercat-ai-kit/tools/pin.mjs urls`
Expected: `10 条 URL · 非规范 0 条` + `✅ 全部为规范形态 ssh://git@gitlab.ts.huntercat.cn:22220/huntercat-digital/<repo>.git`

- [ ] **Step 3: 确认只改了 url 行**

Run: `git diff --stat && git diff .gitmodules | grep -c 'url = '`
Expected: `1 file changed, 10 insertions(+), 10 deletions(-)`；计数 `20`

- [ ] **Step 4: 同步并递归初始化（本机有 SSH 密钥，应通过）**

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

Expected: 退出码 0

- [ ] **Step 5: 核心验收 —— pin 逐一未变**

```bash
git submodule status | sed 's/^-//' | awk '{print $2, $1}' | sort > /tmp/lieshou-pins-after.txt
diff /tmp/lieshou-pins-before.txt /tmp/lieshou-pins-after.txt && echo "✅ PIN 未变"
```

Expected: `✅ PIN 未变`

- [ ] **Step 6: 顶层 remote 已全指 GitLab**

Run: `git submodule foreach 'git remote get-url origin' | grep -c 'git@github.com'`
Expected: `0`

- [ ] **Step 7: 提交并推送**

```bash
git add .gitmodules
git commit -m "chore(submodules): 顶层供应链 URL 由 GitHub 迁至 GitLab（pin 不变）

10 条顶层 submodule URL 规范化为 ssh://git@gitlab.ts.huntercat.cn:22220/huntercat-digital/<repo>.git。
迁移前后 pin 逐一相等（/tmp/lieshou-pins-{before,after}.txt diff 为空），可重现性不变。
嵌套 38 条仍指 GitHub —— 属有意保留的债务，见 docs/superpowers/plans 附录「轨二」。"
git push origin main
```

---

## Task 4: 验证 URL 迁移没有破坏 CI（关键风险点）

**为什么是关键：** runner 10 是 docker executor，job 容器内**没有 SSH 私钥**。顶层 10 条现在是 GitLab 的 `ssh://...:22220/...`。它能否被克隆，取决于 GitLab Runner 是否用 `CI_JOB_TOKEN` 改写同实例 submodule URL —— **`:22220` 这个非标准端口能否被识别，未经实测**。

- [ ] **Step 1: 轮询新 pipeline**

复用 Task 2 Step 4 的轮询。

- [ ] **Step 2: 判定**

| 结果 | 含义 | 动作 |
| --- | --- | --- |
| `success` | Runner 内置的 job-token 改写认了 `:22220` | 跳到 Task 5 |
| 克隆子模块时 `Permission denied (publickey)` | 未改写，容器无密钥 | 执行 Step 3 |

- [ ] **Step 3: 回退阶梯 L2 —— 用 `pre_clone_script` 显式改写 GitLab SSH → job token HTTPS**

把 Task 1 Step 2 的 `pre_clone_script` 换成两段拼接（GitHub 与 GitLab 各一条）：

```bash
ssh weilai@100.64.0.22 'docker exec gitlab-runner cat /etc/gitlab-runner/config.toml > /tmp/rc2.toml && python3 - <<'"'"'PY'"'"'
import pathlib, re
p = pathlib.Path("/tmp/rc2.toml"); t = p.read_text()
new = ("git config --global url.\\"https://github.com/\\".insteadOf \\"git@github.com:\\" && "
       "git config --global url.\\"https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.ts.huntercat.cn/\\".insteadOf \\"ssh://git@gitlab.ts.huntercat.cn:22220/\\"")
t2 = re.sub(r"^  pre_clone_script = .*$", "  pre_clone_script = \"" + new + "\"", t, flags=re.M)
assert t2 != t, "未替换到 pre_clone_script"
p.write_text(t2); print("已更新为 L2")
PY
docker cp /tmp/rc2.toml gitlab-runner:/etc/gitlab-runner/config.toml && docker restart gitlab-runner >/dev/null && echo restarted'
```

然后回 Step 1 重跑。**若仍失败** → 走 L3：给 job 容器挂只读 SSH 私钥（`volumes` 挂 `id_ed25519`，`environment` 设 `GIT_SSH_COMMAND`），并把该公钥作为 **deploy key** 加到涉及的 GitLab 项目；此时停下把实测日志反馈，不要连续试。

---

## Task 5: release job 端到端验证

- [ ] **Step 1: 打并推送 tag**

```bash
cd ~/dev/gitlab-mirror/lieshou-standalone
git tag -a v0.1.1 -m "LieShouBoot v0.1.1（GitLab CI 首版）"
git push origin v0.1.1
```

- [ ] **Step 2: 轮询 release pipeline（给足 90m）**

复用 Task 2 Step 4。Expected: `success`，job 列表含 `release`。

- [ ] **Step 3: 验证 Generic Package 存在**

```bash
TOKEN=$(ls ~/.gitlab-pat/*.token | head -1)
curl -s --max-time 25 -H "PRIVATE-TOKEN: $(cat $TOKEN)" "https://gitlab.ts.huntercat.cn/api/v4/projects/47/packages?per_page=20" \
  | python3 -c "import sys,json;[print(p['name'],p['version'],p['package_type']) for p in json.load(sys.stdin)]"
```

Expected: 含 `lieshouboot 0.1.1 generic`

- [ ] **Step 4: 验证 Release 记录**

```bash
TOKEN=$(ls ~/.gitlab-pat/*.token | head -1)
curl -s --max-time 25 -H "PRIVATE-TOKEN: $(cat $TOKEN)" "https://gitlab.ts.huntercat.cn/api/v4/projects/47/releases" \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print(d[0]['tag_name']);[print(' ',l['name'],l['url']) for l in d[0]['assets']['links']]"
```

Expected: `v0.1.1` + 资产链接

- [ ] **Step 5: 真实下载一次（不能只看链接存在）**

```bash
TOKEN=$(ls ~/.gitlab-pat/*.token | head -1)
curl -sS -o /tmp/lb-v0.1.1.tar.gz -w "http=%{http_code} bytes=%{size_download}\n" -L \
  --header "PRIVATE-TOKEN: $(cat $TOKEN)" \
  "https://gitlab.ts.huntercat.cn/api/v4/projects/47/packages/generic/lieshouboot/v0.1.1/lieshouboot-v0.1.1.tar.gz"
tar tzf /tmp/lb-v0.1.1.tar.gz | head -5
```

Expected: `http=200`、`bytes` 远大于 0、能列出 `lieshouboot-release/backend/...jar`

---

## Task 6: 文档收口 + 验收表

- [ ] **Step 1: 修正 spec 里已作废的章节**

`docs/superpowers/specs/2026-09-13-lieshou-standalone-gitlab-supplychain-design.md` 需要：
- §2.5、§3.2「shell executor」整段**删除**（其前提已被推翻）
- §3.2 改为描述**实际的 runner 10**（docker executor / tag `wqc` / maven 镜像 / CA / m2 缓存 / `RestartPolicy=always`）
- §3.4 镜像策略写实（`maven:3.9-eclipse-temurin-21` + `node:22-bookworm`，并写明**为何不用 slim**）
- §3.5 写实（容器无密钥 → `pre_clone_script` 改写 GitHub SSH→匿名 HTTPS；GitLab SSH 的认证结论按 Task 4 实测结果填写）
- §7 未决项改为「轨二：嵌套 38 条 GitLab 化」并链接附录
- 新增「## 9. 实施证据」：pipeline URL、job 状态、pin diff、release 下载输出

- [ ] **Step 2: 同步 README 里过期的事实**

Run: `grep -n -iE "github|actions|release 资产|CI" README.md`
把「打 tag v* 自动执行组合发布并上传 GitHub Release 资产」等描述改为 GitLab CI + Generic Package。

- [ ] **Step 3: 提交**

```bash
git add README.md docs/
git commit -m "docs: 修正已作废的设计章节 + README 同步（CI 平台/发布资产）+ 实施证据"
git push origin main
```

- [ ] **Step 4: 逐条对照验收（六条全部要有实测输出才可声明完成）**

| # | 标准 | 命令 |
| --- | --- | --- |
| 1 | 顶层 URL 全规范 | `node .../pin.mjs urls` |
| 2 | pin 未变 | `diff /tmp/lieshou-pins-{before,after}.txt` |
| 3 | 递归初始化成功 | `git submodule foreach --recursive 'git remote get-url origin'` |
| 4 | 流水线真跑 | pipeline API `status`（非 `stuck_pending`） |
| 5 | 无 Auto DevOps job | pipeline jobs 列表 |
| 6 | Release 可下载 | Task 5 Step 5 的 `http=200` |

---

## 已知取舍

1. **嵌套 38 条仍指 GitHub** —— Task 1 的 `pre_clone_script` 就是为它准备的。CI 因此仍依赖 GitHub 可用性（只读、匿名、公开仓）。
2. **`pnpm install --no-frozen-lockfile`** —— 沿用原 workflow 写法（各端子工作区与根 lockfile 不完全同步，用 frozen 会挂）。
3. **release 不含 desktop** —— 需 Rust toolchain，与原 workflow 一致。
4. **runner 10 的 `glrt-` token 已落进本次会话记录** —— 建议在 GitLab UI 对该 runner 做 rotate，并同步 `config.toml`。
5. **release job 的工具链是拼出来的** —— 因为该 job 同时需要 JDK21+mvn 与 Node+pnpm，而现有镜像没有一个同时具备：选 `maven:3.9-eclipse-temurin-21` 作为底（自带 JDK/mvn/git），再下载 Node 22 的 `.tar.gz` 自装（**不能用 `.tar.xz`：该镜像没有 `xz`**）。后续若要收敛，可自建一个镜像推 Harbor。

---

## 附录 · 轨二：嵌套 38 条 GitLab 化（本计划**不执行**，需独立立项）

实测数据（2026-09-13，GitLab API 全树遍历）：

- 38 条嵌套 URL，分布在 **14 个不同的 `(仓, SHA)` 对** 里（`.gitmodules` 是**父仓在那个 SHA 上的文件**）
- 涉及 7 个上游仓：`lieshou-ui`(4 个 SHA)、`lieshou-contract-api`(3)、`lieshou-contract-types`(3)、`lieshou-app-{admin,desktop,mobile,mini-program}`(各 1)
- 5 个嵌套目标仓（`contract-api` / `contract-config` / `contract-types` / `core-web` / `ui`）在 GitLab **全部存在**，且 **38 个 pin 全部存在** ✅ 技术上可行

代价与风险：
1. 需从每个历史 SHA 分叉造提交（只改 `.gitmodules`，内容逐字节不变）并打 tag，**共 14 个提交/tag**
2. 随后三级 pin 级联：ui/api/types → 4 端 → 本仓 10 个 pin，**「不动 pin」约束必然作废**
3. **URL 形态需先定**：绝对 SSH（`pin.mjs` 规范形态，13 个交付仓在用）vs 相对 URL（CI 自动用 job token，但与既有规范冲突）
4. 若不等价分叉而是直接升到 `main`，会把上游 09-12 的新内容带进 pin 位，构建风险从「零」变成「未知」

净收益：CI 结构上不再依赖 GitHub 可用性。**成本远大于「改 URL」，本质是一次跨 7 仓的版本升级工程**，故与轨一解耦。
