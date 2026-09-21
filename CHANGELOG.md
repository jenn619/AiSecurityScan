# 变更日志

本项目采用语义化版本号。v0.4.0 为本次验收交付版本。

---

## v0.4.0

### 新增能力

- **黑盒靶机三件套**：黑盒域（域 4 红队 / 域 5 护栏）的靶机不再只填 API 地址，
  补齐 `model` / `api_key` / `endpoint` 三个字段。
  - 配置中心：`aiss config set redteam.model glm-5.3` / `redteam.api_key`（密钥写用户级配置，
    不落项目 `.aissrc`），`guard.*` 同理
  - 控制台：黑盒靶机表单、新建体检表单均可直接填写，与配置文件取并集、表单值优先
  - 兜底：未填写时回落到配置中心的 `redteam.model` / `guard.model` / `*.api_key`；
    OpenAI 兼容端点缺 `model` 会在本地直接报「未填写被测模型名称」，不再把 400 甩给服务端
- **Git 取源自愈**：`--repo` 不再依赖 GitPython，改为原生 git 子进程 + 归档回落
  （GitHub codeload / GitLab `-/archive` / Gitea archive），支持 `--ref` 指定分支或标签；
  失败时给出可执行的归类诊断（未授权 / 仓库不存在 / 未装 git / 超时 / 网络不可达）
- **MCP 域支持仓库扫描**：`aiss scan mcp --repo ... --ref ...`
- **三组单条规则 CRUD**（控制台 + API）
  - 规则库：查看 / 编辑 / 删除 / 停用单条规则，支持新建自定义规则
  - 越狱测评集：查看 / 编辑 / 删除 / 追加单条样本
  - 安全体检判定规则：三组（refusal / compliance / bypass）判定模式的增删改、停用与重置
- **判定规则持久化**：体检判定规则落到 `~/.aiss/judge_rules.yml`
  （可用 `AISS_JUDGE_RULES_FILE` 覆盖），改完立即生效，可一键恢复内置默认

### 变更

- **术语收敛**：原 "AIG" 命名统一改为 `external`（外部同步规则）。
  规则目录 `aiss/rules/external/`、来源标记 `custom | external`、规则 ID 前缀 `EXT-`；
  仅保留 Cloudflare 真实响应头 `cf-aig-event-id` 等必须照抄的上游字面量
- 依赖精简：去掉 GitPython（改原生 git 子进程后不再需要）

### 修复

- 测评集样本 `metadata` 反复套娃（改一次多一层 `metadata.metadata.metadata`）
- 控制台 DELETE 请求把 204 空响应当成失败（`res.json()` 抛 `SyntaxError`）
- 判定规则「恢复默认」在受限环境下删文件被拦导致 500（改为改写空存储）
- 配置兜底不再塞占位模型名 `gpt-4o-mini`，避免静默用错模型

### 清理（本次交付）

- 剔除缓存 / 构建产物：`.mypy_cache`、`.mypy_cache2`、`.pytest_cache`、全部 `__pycache__`、
  `aiss.egg-info`、`aiss/rules/.external_cache.json`（6 MB 运行时缓存，可重建）
- 剔除不该随交付带走的文件：本机私有配置 `.aissrc`、根目录一次性扫描产物 `sbom.json`、
  内部开发方案草稿 `方案.txt`
- 代码去冗余：清理 23 处未使用的 import（多为重构后的残留）
- 补齐 `pyproject.toml` 中漏声明的 `aiss.evals` 包
- 新增 `.gitignore`，避免缓存再次混入

---

## v0.3.0

五域扫描能力的基础版本（基础设施 / MCP / 技能供应链 / 红队 / 护栏），
含基线增量门禁、SBOM、Web 控制台与初始外部规则同步。详细用法见 `README.md`。
