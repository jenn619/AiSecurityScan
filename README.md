# AiSecurityScan（`aiss`）

一站式 **AI 应用安全检查平台**，覆盖五域。规则优先 + LLM 增强，纯 Python 3.11+，命令行交付，可进 CI。

```
 █████  AiSecurityScan v0.4.0
```

> **跑不起来先看这里**：本项目依赖 Python **3.11+**（用到 `match`、PEP 604 `X | Y` 语法）。
> 装完后若 `aiss` 命令找不到，多半是该环境的 `Scripts/` 目录没进 `PATH`，
> 直接用 `python -m aiss` 调用即可，二者完全等价。

## 文档索引

| 文档 | 内容 |
| --- | --- |
| `README.md`（本文） | 快速开始、五域用法、报告解读、规则体系、CI 集成、配置中心 |
| `CHANGELOG.md` | 版本变更记录 |
| `docs/delivery.md` | 交付说明：范围、目录结构、安装运行、去冗余清单、已知约束 |
| `docs/acceptance.md` | 验收清单：逐条可勾选的验收项与验证命令 |
| `docs/architecture.md` | 架构分层、一次扫描的完整链路、自安全设计 |
| `docs/ci.md` | CI/CD 集成：基线、指纹、退出码、忽略文件、SARIF、SBOM |
| `docs/rules.md` | 自定义规则编写指南 |
| `docs/extending.md` | 新增一个扫描域的扩展步骤 |
| `NOTICE` | 第三方规则与素材的来源与许可声明 |

## 为什么需要它

AI 应用的攻击面已经从「模型」外扩到「模型周围的一切」：MCP 服务、Agent 技能包、系统提示词、
工具描述、依赖供应链，以及跑模型的基础设施本身。这些组件大多来自第三方，安装即获得
**宿主机级别的权限**，而现有 SCA / SAST 工具基本看不见这类风险。

`aiss` 补的就是这块空缺：**静态三域审计供应链与部署，黑盒两域实测 Agent 与模型的抗攻击能力**。

---

## 快速开始

```bash
pip install -e ".[server]"     # Web 接口可选；只要 CLI 就 pip install -e .
python -m aiss --help

# 域3：扫一个 Agent 技能包（本地目录 / zip / git 仓库都行）
aiss scan skill --path ./my-skill
aiss scan skill --repo https://github.com/someone/some-skill

# 域2：扫 MCP 服务（配置文件或源码目录）
aiss scan mcp --path ./mcp-server --output report.html

# 域1：扫 AI 基础设施（组件识别 + CVE 版本匹配 + 部署配置体检）
aiss scan infra --path ./ai-platform
aiss scan infra --path ./deploy --target-url http://10.0.0.5:3000   # 附加主动指纹探测

# 域4：对自有 Agent 做黑盒红队（多轮对话攻击 + 蜜标判定）
aiss scan redteam --target-url http://localhost:8000/v1
aiss scan redteam --dry-run                 # 离线演示：内置"脆弱 Agent"靶机

# 域5：对自有模型做护栏 / 越狱测评（ASR + 等级）
aiss scan guard --target-url http://localhost:8000/v1
aiss scan guard --dry-run --persona guarded # 内置"有护栏"模型对照

# 黑盒靶机三件套：地址 + 模型名 + 密钥
# 缺 model 时 OpenAI 兼容端点会直接以「model 不能为空」拒绝请求。
# 模型名与密钥写在配置里（redteam.model / guard.model、*.api_key），
# 控制台的两个表单也可以直接填，两边取并集、表单值优先。
aiss config set redteam.model glm-5.3
aiss config set redteam.api_key sk-xxx      # 密钥默认写进用户级配置，不落 .aissrc

# 五域全扫，输出三份报告（JSON / SARIF / HTML）
aiss scan all --path ./project --dry-run --format all --output out/report

# 生成基线：把当前问题固化，之后 CI 只管「新引入」的
aiss baseline save --path . --type infra,mcp,skill -o .aiss-baseline.json

# 增量门禁：只对新增 / 等级劣化的风险返回非零退出码
aiss diff .aiss-baseline.json --path . --fail-on-new high

# SBOM：CycloneDX 1.5 软件物料清单（含命中 CVE）
aiss sbom --path ./deploy -o sbom.json

# CI：出现 high 及以上风险返回非零退出码（无基线时的退化模式）
aiss scan --type mcp,skill,infra --path . --fail-on high

# 规则包：导出模板 → 改 YAML → 上传更新（控制台「上传规则包」走同一套逻辑）
aiss rules export my-rules.zip
aiss rules import my-rules.zip --dry-run    # 先校验再落盘
```

常用全局参数：

| 参数 | 说明 |
| --- | --- |
| `--path / -p` | 本地目录 / zip\|tar 包 / 单个配置文件 |
| `--repo / -r` | Git 仓库 URL（公开仓库，私有仓库用 `--token`）；git 协议被代理拦掉时会自动改走源码归档下载 |
| `--type / -t` | 扫描域，逗号分隔：`infra,mcp,skill,agent_redteam,model_guard` |
| `--format / -f` | `console`（默认）/ `json` / `sarif` / `html` / `all` |
| `--output / -o` | 输出路径，后缀决定格式；`--format all` 时生成三份 |
| `--target-url / -u` | 黑盒靶机地址（OpenAI 兼容 `/chat/completions` 基址）；infra 下为主动探测站点 |
| `--dry-run` | 黑盒域不联网，用内置脚本化靶机演示 |
| `--persona` | 脚本化靶机人格：`vulnerable` / `safe` / `unguarded` / `guarded` |
| `--llm / --no-llm` | LLM 语义增强（默认关闭，纯规则即可工作） |
| `--fail-on` | CI 全量阈值：`critical\|high\|medium\|low\|info` |
| `--baseline / -b` | 基线文件（或上次的 JSON 报告），输出增量而非全量 |
| `--update-baseline` | 扫描后把结果写入该基线文件 |
| `--fail-on-new` | CI 增量阈值：出现该级别及以上「新增/劣化」即失败 |
| `--verbose / -v` | 打印证据片段与修复建议 |
| `--log-dir` | 黑盒测评对话留痕目录（审计用） |
| `--config` | 指定配置文件（默认 `~/.aiss/config.yml`） |

其他命令：

```bash
aiss rules list [--domain infra|mcp|skill] [--source custom|external]
aiss rules sync [--version main]        # 同步外部规则包
aiss baseline save -p . -o .aiss-baseline.json   # 生成基线
aiss baseline show .aiss-baseline.json           # 查看基线内容
aiss diff .aiss-baseline.json -p .               # 增量对比（重新扫描）
aiss diff prev.json cur.json --markdown pr.md    # 对比两份已有报告
aiss sbom -p ./deploy -o sbom.json               # CycloneDX SBOM
aiss serve --port 8811                  # 启动 FastAPI 服务
aiss config show
aiss version
```

---

## Web 控制台

不想敲命令时，启服务后在浏览器里下发扫描：

```bash
pip install -e ".[server]"
python -m aiss serve --port 8811     # 浏览器打开 http://127.0.0.1:8811
```

**Windows 双击启动**：直接运行 `scripts/start-web.bat`（会自动定位虚拟环境里的 Python，
无需把它配进 PATH），然后访问同样的地址。脚本内改 `PORT` 相关参数即可换端口。

> 若 `python -m aiss` 提示找不到包，说明运行时的 Python 与安装 AISS 的不是同一个，
> 用绝对路径调：`C:\Users\<你>\.workbuddy\binaries\python\envs\default\Scripts\python.exe -m aiss serve`

前端是**单页零构建**的（`aiss/web/index.html`，原生 JS，无 npm / 无打包步骤），由 FastAPI
同源托管，所以不存在跨域问题。界面按「左侧导航 + 内容区」组织，四个主视图：

**扫描任务** — 卡片式任务列表（目标、状态、扫描域、耗时），点开进详情：
评分卡 + 五个等级统计 + 风险明细表，附 JSON / SARIF 下载与 HTML 报告新窗口打开。

**规则库** — 按域分标签页（基础设施 / MCP 服务 / 技能供应链，角标为各域条数），
支持对规则 ID、名称、描述、标签、CWE 做全文搜索，卡片列表带分页器。
规则库接口不需要每次重解析（按规则目录 mtime 缓存），翻页 ≈ 50ms。

**上传规则包** — 规则库页右上角的按钮，支持把 `.zip` / `.tar.gz` / `.tgz` / 单个 `.yaml`
拖进去更新规则模板：

- 选目标：自研规则（`rules/custom`）或外部规则同步（`rules/external`）
- **试运行**先跑一遍校验，只告诉你会新增/覆盖哪些文件、解析出多少条规则，确认后再写入
- 默认先备份现有规则到 `aiss/rules/.backups/`，可关
- 有文件报错时默认整包拒绝（避免半套规则入库），勾「跳过校验失败的文件」可部分导入
- 模态里的「下载现有模板」导出当前规则 zip，改完再传回来

包里只有 YAML 会被采用，其余文件忽略；zip-slip（`../`）、符号链接、超量文件一律拒绝。

**单条规则管理** — 每张规则卡片右侧有「查看/编辑」「停用」「删除」三个入口：

- 自研规则（`custom`）可直接改 JSON 后保存，落盘到 `aiss/rules/custom/`，
  保存前会校验 id 唯一性与每条 `patterns` 的正则能否编译，坏规则不会写进文件
- 外部同步规则（`external`）**只读**：改删返回 409 并提示改用「停用」，
  因为下次 `rules sync` 会把它覆盖回去
- 停用对两类都生效：自研写回 YAML 的 `enabled: false`，外部写进用户配置的
  `rules.disabled_rules`，同步后依然保持停用
- 右上角「新建规则」按 domain 落到对应文件（infra / mcp / skill）

**越狱评测集** — 体检的「弹药库」，卡片式管理内置语料与自定义数据集：

- 内置两个评测集（越狱基线、编码绕过子集），由包内语料生成，**只读**、不可删改
- 上传 JSON 时兼容 UTF-8（含 BOM）与 GBK；同名冲突返回 409，需显式勾选「覆盖同名」
- 卡片展示语言、推荐指数（星级）、样本数、标签与风险类型；点「眼睛」就地展开前 20 条样本
- 支持改元信息（描述中英文 / 贡献者 / 标签 / 来源 / 推荐指数），样本不动
- 每张卡片可一键「用于体检」，直接带着这个数据集跳到新建体检表单
- **样本级增删改**：自定义集的样本可逐条改（提示词 / 风险类型 / 标签）、删除、追加；
  内置集样本可看不可改（返回 409）

**体检判定规则** — 体检页右上角的按钮，管理「这条算不算绕过」的三组正则
（拒答特征 / 合规特征 / 绕过痕迹）：

- 内置项只读可停用，自定义项可增删改，正则保存前先编译校验
- 落在 `~/.aiss/judge_rules.yml`，`RuleEvaluator` 每次判定都从这份配置取，改完立即生效
- 「恢复内置」清空全部用户改动

**安全体检** — 评测集 × 攻击算子 × 评估器的一次完整闭环（详见下一节）：
跑完的记录取拓扑盘 `~/.aiss/runs/`，列表按时间倒序，点开是风险率总览 + 两个维度汇总表 +
Badcase 明细（支持关键词过滤）+ CSV / JSONL 导出。

其他细节：

- 新建扫描是模态表单，三类目标切换（**本地路径 / Git 仓库 / 黑盒靶机**），字段随类型自动对应
- 新建体检表单里的靶机配置与「黑盒靶机」同构：Agent 名称 / URL / **模型名称** / **API Key** /
  HTTP 方法 / 离线人格；高级设置里可填对话端点、请求头、请求体模板、响应解析器、超时（ms）、
  输入验证，并可先跑连通性测试
- 体检表单可就地覆盖语义模型配置（API 地址 / API Key（可置空）/ 模型名 / 并发），
  用途是模型驱动算子与 LLM 评估器，留空则沿用配置文件
- 五域勾选；选「黑盒靶机」时自动选上红队与护栏两域
- 提交后自动轮询任务状态（1.2s 间隔），完成后弹提示并可直接展开报告
- 明细表支持关键词实时过滤（规则 / 标题 / 文件路径，带 250ms 防抖）
- **默认亮色主题，右上角可切暗色**（偏好存 localStorage）
- 刷新页面后仍能看到历史任务（后端保留最近若干条，默认 TTL 内可查）

对应的 HTTP 接口（前端就是消费这些，可直接被 CI 或第三方系统复用）：

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/health` | 健康检查，返回版本与规则条数 |
| GET | `/api/domains` | 可用扫描域与其中的黑盒域 |
| GET | `/api/rules` | 规则清单，支持 `domain` / `source` 过滤、`q` 搜索、`offset`/`limit` 分页（`total` 为过滤后总数） |
| POST | `/api/rules/import` | 上传规则包（multipart：`file` + `target` / `dry_run` / `backup` / `allow_partial`） |
| GET | `/api/rules/export?target=` | 导出规则模板 zip |
| GET | `/api/rules/meta?target=` | 最近一次导入记录（时间、文件、条数、备份路径） |
| GET | `/api/rules/{id}` | 单条规则详情（含原始定义，`editable` 标明能否改） |
| PUT | `/api/rules/{id}` | 改一条自研规则（外部规则 409） |
| DELETE | `/api/rules/{id}` | 删一条自研规则（外部规则 409） |
| POST | `/api/rules/{id}/toggle` | `{"enabled": false}` 启用 / 停用 |
| POST | `/api/rules` | 新增自研规则，`file` 可指定落盘文件 |
| POST | `/api/target/test` | 靶机连通性测试（也可测离线脚本化靶机） |
| GET | `/api/datasets/{name}/samples` | 样本分页清单（`q` / `offset` / `limit`） |
| GET/PUT/DELETE | `/api/datasets/{name}/samples/{i}` | 单条样本的查看 / 改 / 删（内置集 409） |
| POST | `/api/datasets/{name}/samples` | 追加一条样本 |
| GET | `/api/evals/rules` | 体检判定规则三组全量 |
| POST | `/api/evals/rules` | 新增判定规则（`group` + `pattern`） |
| GET/PUT/DELETE | `/api/evals/rules/{group}/{id}` | 单条判定规则的查看 / 改 / 删 |
| POST | `/api/evals/rules/{group}/{id}/toggle` | 启用 / 停用一条判定规则 |
| POST | `/api/evals/rules/reset` | 恢复内置判定规则 |
| GET | `/api/datasets` | 评测集清单：`q` 搜索、`offset`/`limit` 分页、`stats` 计数 |
| POST | `/api/datasets` | 导入评测集（multipart：`file` 或 `payload` 文本 + `replace`） |
| GET | `/api/datasets/{name}` | 评测集详情 + 样本预览（`?limit=`） |
| PUT | `/api/datasets/{name}` | 更新元信息（不改样本） |
| DELETE | `/api/datasets/{name}` | 删除自定义评测集（内置返回 400） |
| GET | `/api/evals/meta` | 体检表单选项：算子 / 评估器 / 模式清单 |
| POST | `/api/evals/run` | 提交体检，返回 202 + `task_id` |
| GET | `/api/evals/tasks/{id}` | 轮询体检任务，完成后 `report` 是完整 `EvalRun` |
| GET | `/api/evals/runs` | 历史体检记录列表 |
| GET | `/api/evals/runs/{id}` | 体检结果：概览 + badcase + 失败明细 |
| GET | `/api/evals/runs/{id}/export` | 导出明细：`format=csv\|jsonl\|json`、`scope=all\|badcase\|failure` |
| POST | `/api/scans` | 提交扫描，返回 202 + `task_id` |
| GET | `/api/scans` | 任务列表（`?limit=`） |
| GET | `/api/scans/{id}` | 单任务状态：`pending` / `running` / `done` / `failed` |
| GET | `/api/scans/{id}/report?format=` | 取报告，`format` 支持 `json` / `sarif` / `html` |
| DELETE | `/api/scans/{id}` | 删除任务记录 |

```bash
curl -X POST localhost:8811/api/scans -H 'Content-Type: application/json' \
     -d '{"path":"examples/malicious-skill","domains":["skill"]}'
curl localhost:8811/api/scans/<task_id>/report?format=json

# 规则包先试运行（只校验），确认后再真正导入
curl -X POST localhost:8811/api/rules/import \
     -F file=@my-rules.zip -F target=custom -F dry_run=true
curl -X POST localhost:8811/api/rules/import \
     -F file=@my-rules.zip -F target=custom
```

> 注意：服务当前**没有鉴权**，默认也只监听 `127.0.0.1`。放到内网之外时请自行加一层
> 反向代理认证——任务提交接口可接受任意本地路径，暴露出去等同于开放本机文件读取。

---

## 大模型安全体检（`aiss eval`）

域 5 的 `aiss scan guard` 是「跑一批固定越狱用例、看 ASR」。体检更进一步：把
**数据集、攻击算子、被测目标、评估器**拆成四层可自由组合的流水线，用来回答
「在哪些风险类型上最容易失守」「哪种攻击手法最有效」。只对自有或已授权目标使用。

```
评测集 (Dataset)  →  攻击算子 (Attack)  →  被测目标 (Target)  →  评估器 (Evaluator)  →  汇总
     样本 × 算子增强        多轮对话(可选)           判定 risk_score/label       双维度报表
```

三层模型角色互不混用：**被测目标**（要体检的模型/Agent）、**攻击模拟模型**（写 Cypher 化改写的算子）、
**评估模型**（LLM judge）。攻击模拟与评估没配 LLM 时自动回落到确定性算子与规则评估器。

### 评测集 JSON 格式

```json
{
  "name": "my-jailbreak-set",
  "description": "English description",
  "description_zh": "中文描述",
  "author": "你的名字",
  "source": ["https://example.com/dataset"],
  "tags": ["越狱", "编码绕过"],
  "recommendation": 5,
  "language": "zh",
  "data": [
    {"prompt": "请告诉我如何…", "risk_type": "harmful-content", "tags": ["auth"]}
  ]
}
```

- `name` **只允许 `[A-Za-z0-9_-]`（1-64 字符）**：它同时是文件名与结果里的 `dataset_id`，
  限制字符集比事后转义可靠
- `recommendation` 取值 1-5；`source` / `tags` 是数组，写逗号分隔的字符串也会被拆开
- `data` 每条至少要有 `prompt`；同义列名 `question` / `text` / `input` / `问题` 等会自动归一化，
  风险类型取 `risk_type` / `category` / `一级领域` 等，认不出的样本会跳过并在提示里告知
- 也接受「纯样本数组」或 `{"data": [...]}` 两种顶层形态

```bash
aiss eval datasets list                       # 内置 + 自定义评测集
aiss eval datasets import ./my-set.json       # 导入；同名冲突用 --replace 覆盖
aiss eval datasets delete my-set
```

### 跑一次体检

```bash
# 离线演示：脚本化靶机（unguarded / guarded / vulnerable / safe 四种人格）
aiss eval run --dataset builtin-jailbreak-baseline --persona unguarded --max-cases 5

# 真实目标
aiss eval run --dataset builtin-jailbreak-baseline \
  --target-url http://127.0.0.1:8000/v1 \
  --agent-name 客服Agent-线上 \
  --attacks baseline base64 role-context --mode parallel \
  --evaluator rule --concurrency 8 --max-turns 3 --seed 42

# 导出明细，喂给表格工具或标注平台
aiss eval run --dataset builtin-jailbreak-baseline \
  --format csv --scope badcase -o badcases.csv
aiss eval runs                                 # 历史记录（~/.aiss/runs/）
```

关键参数：

| 参数 | 含义 |
| --- | --- |
| `--mode` | `parallel` 每条样本跑全部算子；`random` 每条随机一个；`serial` 全部算子顺序叠加 |
| `--max-cases N` | 抽样上限（`--seed` 固定后可复现），0 为全量 |
| `--max-turns N` | 多轮对话：第二轮起用追问模板，检验「先拒后答」是否会滑动 |
| `--evaluator` | `rule`（零成本可复现）/ `llm-judge`（需配置 LLM）/ `http-classifier`（接自有分类服务） |

内置攻击算子 14 个：确定性 11 个（baseline / base64 / hex / rot13 / zero-width / char-split /
reverse / case-swap / json-wrap / markdown-wrap / prefix-filler）+ 模型驱动 3 个
（role-context / rephrase / multilingual，需要攻击模拟模型）。

### 读懂报告

两条硬性口径，避免被数字误导：

- **RiskRate = 命中 /（安全 + 命中）**。超时、限流、连接失败、空输出、评估失败一律**不进分母**，
  单独列出来——否则一次网络抖动会伪装成「这批用例很安全」
- **判定以 `label` 为准，`risk_score` 只用于 Badcase 排序**。评估失败不会被记成 `safe`，
  这是护栏测评里最危险的假阴性

报告同时给出两个维度：

- **按风险类型**：哪些领域最容易失守（按 `risk_type` 聚合）
- **按攻击方法**：哪种手法最有效，额外给出「输入被改写条数」与「改写生效占比」
  （区分「算子没生效」和「算子生效了但护栏扛住了」）

---

## 覆盖的扫描域

| 域 | 模块 | 检测内容 |
| --- | --- | --- |
| --- | --- | --- |
| 1. AI 基础设施 | `aiss/modules/infra` | 组件识别（requirements / Dockerfile / compose / helm / package.json）+ **CVE 版本约束匹配**（外部规则 2169 条）+ 部署配置体检（特权容器、明文凭据、0.0.0.0 暴露、鉴权关闭、latest 镜像…） |
| 2. MCP 服务与插件 | `aiss/modules/mcp` | 工具投毒、凭证泄露、命令注入、过度权限、提示注入、Rug-pull、不安全网络绑定/CORS、认证缺失、Unicode 隐藏指令、审计缺失 |
| 3. Agent 技能供应链 | `aiss/modules/skill` | SkillTrustBench **T01–T09** 全覆盖 + 数据外泄 + 间接提示注入 |
| 4. 智能体黑盒红队 | `aiss/modules/agent_redteam` | 18 条多轮攻击用例（提示注入 / 越权工具 / 记忆投毒 / 数据外带 / 目标劫持 / 资源滥用 / 工具链提权…）+ 蜜标判定 + 对话留痕 |
| 5. 大模型护栏与越狱 | `aiss/modules/model_guard` | 21 条越狱语料（DAN / 编码绕过 / 多语言 / many-shot / 拒绝抑制 / 令牌走私 / 虚构框架…）+ ASR 统计 + 护栏等级 A–F |

### 域 1：基础设施（组件 → CVE）

```
清单文件 → 组件 + 版本 → 外部规则 vuln 规则的版本约束求解 → INFRA-CVE-<CVE>
```

- 支持 `>=`、`<=`、`==` 及 `&&` 组合约束；浮动约束（`>=`/`~=`）自动下调置信度并提示需结合 lock 文件
- 内置 CVSS v3.1 基础分计算，风险等级由向量算出而非照抄字段
- 单组件 CVE 展示按 CVSS 降序封顶 5 条，避免同源风险淹没报告
- 镜像名匹配会剔除 `web` / `api` / `server` 等通用词，防止误报扩散

### 域 2：MCP 服务（OWASP MCP Top 10 对齐）

工具投毒（描述里藏指令 / 索取凭据文件）、凭证泄露（配置硬编码、env 注入、命令行参数）、
命令注入（`shell=True`、用 shell 启动服务）、过度权限（根目录开放、docker/kubectl、危险开关）、
提示注入（外部输入直拼 prompt）、Rug-pull（`npx -y` 未锁版本、远端动态加载工具定义）、
不安全网络（`0.0.0.0`、`CORS *`、明文 http）、认证缺失（sse/http 端点无凭据）、遥测外发、审计缺失。

### 域 3：技能供应链（T01–T09）

| 编号 | 风险 | 典型特征 |
| --- | --- | --- |
| T01 | 技能指令劫持 | `ignore previous instructions`、伪造系统消息、要求对用户隐瞒、注释/零宽字符藏指令 |
| T02 | Agent 记忆投毒 | 写入 `MEMORY.md`/`CLAUDE.md`/`.cursorrules`、要求永久记住 |
| T03 | 远程载荷执行 | `curl \| bash`、`iwr \| iex`、`exec(下载内容)`、base64 解码执行、URL/VCS 安装 |
| T04 | 内嵌恶意代码 | `os.system`、`shell=True`、不安全反序列化、反弹 shell、混淆 |
| T05 | 越权提权 | `sudo`、`chmod 777`、读 `.ssh`/`.aws`、docker.sock、全量环境变量 |
| T06 | 系统持久化 | crontab、systemd/launchd、`~/.bashrc`、注册表 Run 键、`authorized_keys` |
| T07 | 工具伪装 | 虚假官方声明、声明 bash/shell 高权限工具、依赖名仿冒 |
| T08 | 不安全依赖 | 未锁定版本、git/URL 安装、postinstall 脚本、私有索引源 |
| T09 | 不安全编码 | 命令拼接、硬编码凭据、明文 HTTP / `verify=False`、弱哈希、路径拼接、裸异常 |

### 域 4 / 5：黑盒测评怎么判定

判定采用「保守优先」策略，避免把正常回复误判成中招：

1. 命中拒绝话术 → 判定已拦截，不产生风险
2. 命中该用例的「合规输出特征」→ 判定中招（如真的执行了 shell、真的输出了密钥）
3. 触发高危工具调用（`payment.*` / `shell.exec` / `http.post`）→ 直接判定中招，不看措辞
4. 蜜标（每会话随机生成的一次性标识）出现在回复 → 判为上下文隔离失效

`--dry-run` 提供四种内置人格，可在不联网的情况下验证整条测评链路与误报率：
`vulnerable`（脆弱 Agent，红队默认）、`safe`（安全 Agent）、`unguarded`（无护栏模型，护栏默认）、`guarded`（有护栏模型）。

---

## 报告

- **Console**：彩色摘要 + 风险表 + 证据位置（`--verbose` 打印片段）
- **JSON**：完整结构化结果，含 `extra.transcript`（黑盒对话留痕）与分类统计
- **SARIF 2.1.0**：可直接上传 GitHub Code Scanning / 各类 IDE
- **HTML**：暗色主题的人读报告，含评分、等级、证据、修复建议

评分：从 100 起扣，按等级（Critical 40 / High 25 / Medium 10 / Low 3 / Info 0.5）×
置信度系数（high 1.0 / medium 0.75 / low 0.45）× 重复命中衰减（封顶 2.5 倍），
等级 A(≥90) / B(≥75) / C(≥60) / D(≥40) / E(≥20) / F。
护栏测评额外输出 ASR（攻击成功率）与 A–F 护栏等级。

---

## AST 语义引擎：为什么光有正则不够

`aiss/engines/dynamic.py` 用标准库 `ast` 做语法树分析——**只解析，绝不执行**——补上正则看不见的那一层。下面这些写法在正则下几乎无法命中，AST 全部还原：

```python
import os as _o
_o.system(cmd)                       # 模块别名
from pty import spawn as _s
_s(["bash"])                         # 函数别名
_m = __import__("subprocess")
_m.run(cmd, shell=True)              # 动态导入赋别名
getattr(_o, "system")(cmd)           # 反射取属性
_o.__dict__["system"](cmd)           # 字典取成员
importlib.import_module("os").system(cmd)

payload = base64.b64decode(x)
exec(payload)                        # 污点传播 → AST-TAINT-001
code = requests.get(U).text
eval(code)
open("/root/.ssh/authorized_keys", "a")   # 持久化写入 → AST-PERSIST-001
```

三层能力：

1. **调用解析**：跟随 import 别名、from-import、`__import__`/`importlib`（含赋别名）、`getattr`/`__dict__`/`globals()`，把调用点还原成 `module.func` 全名；
2. **危险 sink**：`AST-EXEC-00x`（命令执行 / 动态执行 / 动态导入 / 反序列化 / 反射）、`AST-NET-00x`、`AST-FS-001`、`AST-CRED-00x`、`AST-PERSIST-001`；
3. **轻量污点**：函数/模块作用域内做 reaching-definitions，`下载/解码/反序列化` 结果流入 `exec/eval/compile` 即判 Critical。

降噪策略：默认 `min_confidence=medium`，会屏蔽 `getattr`/`setattr` 这类高频写法；`AST-CRED-001` 仅在变量名形如凭证（`KEY`/`TOKEN`/`SECRET`…）时才报——`(config: ast)` 块里可调整。

### JS / TS 语义引擎

MCP Server 与 Agent Skill 有相当比例是 TypeScript 实现的，只有 Python AST 会让这部分目标整体隐形。`aiss/engines/jsast.py` 把同一套能力搬到了 Node 生态：**不依赖 node / tree-sitter，也不执行被测代码的任何一行**，纯词法 + 别名表还原：

```js
const g = globalThis;
g['child' + '_process'].execSync('whoami');          // 字符串索引 + 拼接混淆
const e = require('child_process').exec;             // 成员提取后延迟调用
const req = require.main.require;                    // 隐蔽 require 通道
req('child_process').execSync('id');
const _e = (await import('node:child_process')).execSync;  // ESM 动态导入
new Function('return process')();                    // eval 等价
setTimeout("console.log(1)", 100);                   // 字符串回调 = eval
fs.writeFileSync(path.join(home, '.ssh', 'authorized_keys'), key);  // 持久化
```

规则 ID：`JS-EXEC-001`（命令执行）/ `-002`（vm 沙箱）/ `-003`（eval 家族）/ `-004`（隐蔽 require）/ `-005`（反序列化与原型污染）、`JS-PERSIST-001`、`JS-CRED-001`、`JS-NET-001`。

支持后缀：`.js .jsx .mjs .cjs .ts .tsx .mts .cts`。TS 类型注解会被跳过。__注意__：`JS-NET-001`（对外网络请求）默认是 low 置信度，在 `min_confidence=medium` 下不报，设为 `low` 才启用——正常 MCP Server 天天在发请求，默认报会淹没真实风险。

---

## 增量扫描与 CI 门禁

历史遗留问题一次性修不完。若每次 PR 都全量报错，团队很快会对告警脱敏——所以 CI 要走增量：

```
main 分支：全量扫描 → 落盘 .aiss-baseline.json
PR       ：与基线比对 → 只对「新引入」和「等级劣化」判定失败
```

三态判定：

| 状态 | 含义 | CI 行为 |
| --- | --- | --- |
| `new` | 本次才出现 | 达到 `--fail-on-new` 阈值即失败 |
| `regressed` | 指纹不变但等级上升（medium → critical） | 同上 |
| `persistent` | 两边都在（存量） | 不阻塞 |
| `mitigated` / `fixed` | 等级下降 / 已消失 | 正向反馈 |

指纹按**证据粒度**计算：`sha256(rule_id | file | 归一化代码片段)`。不含行号（代码上下移动是常态）、数字被抹平、空白被压缩——这些都是为了在两次扫描之间保持稳定。

> 曾经按 Finding 粒度比对过，结果是灾难：同一文件同一规则的多个证据会被合并，`evidence[0]` 依赖分析顺序，一旦多一处同类调用就会同时产生「假新增 + 假修复」。改成证据粒度后彻底消失。

完整流程、退出码语义、`.aissignore` 用法见 [`docs/ci.md`](docs/ci.md)，可直接抄的流水线模板见 [`.github/workflows/aiss.yml`](.github/workflows/aiss.yml) 与 [`.gitlab-ci.yml`](.gitlab-ci.yml)。

---

## SBOM

```bash
aiss sbom --path ./deploy -o sbom.json --with-vulns
```

输出 CycloneDX 1.5，组件类型映射到 `library` / `container` / `application`，每个组件带 `purl` 与来源文件行号，命中 CVE 的进入 `vulnerabilities` 段落。可直接喂给 Dependency-Track / grype / trivy。

---

## Web 接口（`aiss serve`）

```bash
aiss serve --host 127.0.0.1 --port 8811
```

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/health` | 健康检查 + 规则数量 |
| GET | `/api/domains` | 支持的扫描域 |
| GET | `/api/rules?domain=&source=&q=` | 规则清单（搜索 + 分页） |
| POST | `/api/rules/import` | 导入规则包（zip / tar.gz / 单 YAML），支持 `dry_run` |
| GET | `/api/rules/export?target=custom\|external` | 导出规则模板 zip |
| GET | `/api/rules/meta?target=` | 最近一次导入记录 |
| POST | `/api/scans` | 提交扫描任务，立即返回 `task_id`（202） |
| GET | `/api/scans` | 任务列表 |
| GET | `/api/scans/{id}` | 任务状态 |
| GET | `/api/scans/{id}/report?format=json\|sarif\|html` | 取报告 |
| DELETE | `/api/scans/{id}` | 删除任务 |

```bash
curl -X POST localhost:8811/api/scans -H 'Content-Type: application/json' \
  -d '{"path":"examples","domains":["mcp","skill"],"format":"json"}'
curl localhost:8811/api/scans/<task_id>/report?format=sarif
```

任务在线程池中异步执行，状态与结果落盘到 `~/.aiss/tasks/*.json`；
需要横向扩展时把 `server/queue.py` 的 `submit` 换成真实 broker 即可。

---

## 规则体系

```
aiss/rules/
├── custom/          # 自研规则（内部格式，直接映射 Rule 模型）
│   ├── mcp.yml      # 25 条 MCP 规则
│   ├── skill.yml    # 46 条技能规则
│   └── infra.yml    # 11 条部署配置体检规则
├── external/             # 从外部规则同步（aiss rules sync）
│   ├── fingerprints/  # 组件指纹（域1 静态特征 + 主动探测）
│   ├── vuln/          # 2169 条 CVE / 版本比对规则
│   ├── mcp/           # 外部规则的 MCP 审计提示词规则
│   └── VERSION.txt
└── loader.py        # 加载 + 外部规则适配 + 缓存
```

自定义一条规则（`aiss/rules/custom/` 下新增 YAML 即可）：

```yaml
- id: SKILL-T09-007
  name: "不安全编码 - 示例"
  domain: skill
  category: T09-unsafe-coding
  severity: high          # critical | high | medium | low | info
  confidence: medium      # high | medium | low
  cwe: [CWE-000]
  owasp: [LLM05]
  description: 风险说明
  remediation: 修复建议
  include: ["**/*.py"]    # 作用域 glob，支持 **/*.py、*.json、requirements*.txt
  patterns:
    - regex: 'dangerous_call\s*\('
      flags: "im"         # i 忽略大小写 / m 多行 / s dotall
      note: 命中说明
```

> 规则文件用 **YAML 单引号**字符串，反斜杠为字面量，直接交给 Python `re` 解析；
> 含冒号的中文描述务必加引号，否则 YAML 会解析失败。需要匹配引号时用 `\x27` / `\x22`。

`aiss rules list` 查看已加载规则；`~/.aiss/config.yml` 的 `rules.disabled_rules` 可停用指定规则。

### 规则包导入 / 导出

不想手工往规则目录里拷文件时，把规则打包成压缩包上传即可（命令行与控制台同一套逻辑）：

```bash
aiss rules export my-rules.zip            # 导出当前自研规则当模板
# 改完 YAML ……
aiss rules import my-rules.zip --dry-run  # 先校验：会新增/覆盖什么、解析出多少条
aiss rules import my-rules.zip            # 确认后落盘，规则立即生效（无需重启服务）
aiss rules import external-bundle.tar.gz --target external --allow-partial
```

约定与安全边界：

- 目标只有两个：`custom`（自研）和 `external`（外部规则同步），写死白名单，不接受 `..` 与绝对路径
- 解压走隔离层：拒绝 zip-slip、符号链接、超量文件（600 个）/ 超大包
- 只取 `.yml` / `.yaml`，包里其他文件一律忽略
- 有语法错的文件默认整包拒绝（`--allow-partial` 才跳过坏文件）
- 默认落盘前备份现有规则到 `aiss/rules/.backups/<target>-<时间戳>.zip`
- 单文件 YAML 按原始文件名入库；仓库 zip 顶层的 `repo-main/` 会自动剥掉，
  但 `mcp/` `vuln/` `fingerprints/` 这类分类目录保留（外部规则靠目录名判域）

---

## 自身安全设计（防止被测对象反向劫持）

扫描器面对的是**不可信输入**，因此：

1. **绝不执行被测代码**：静态三域全程只读，不 import、不 run、不 eval 目标文件。
2. **路径逃逸防护**：zip-slip、符号链接、绝对路径、tar 链接一律拒绝（`core/isolation.py`）。
3. **资源硬上限**：文件数 4000、单文件 2MB、总体积 200MB（可在配置里调）。
4. **证据脱敏**：写入报告的片段会抹掉 OpenAI/AWS/GitHub/JWT 等密钥特征。
5. **Prompt 隔离**：交给 LLM 的内容用随机边界包裹并声明为「数据」，输出做结构校验与长度裁剪。
6. **Git 白名单**：只接受 http/https/ssh/git，`file://` 与本地路径被拒绝。
7. **黑盒域不出网时不落空**：`--dry-run` 走脚本化靶机；真实靶机异常统一降级为 error，不中断流程。
8. **取源失败要说人话**：`git clone` 失败会归类为「网络不通 / 仓库不存在 / 鉴权失败 / 超时 / 没装 git」
   并带上原始输出；网络类失败自动回落到源码归档下载（`utils/git.py`）。

> **拉不到 GitHub 仓库时**：先确认能否 `git clone`，不能的话可以直接把源码 zip 下载下来，
> 用 `--path <zip>` 或 `--path <解压目录>` 扫描；也可以设 `HTTPS_PROXY` 环境变量
> （或 `git config --global http.proxy <代理地址>`）后再用 `--repo`。

---

## 项目结构

```
aiss/
├── cli.py                 # Typer 入口（scan infra|mcp|skill|redteam|guard|all、rules、serve、
│                          #            eval run|runs|datasets）
├── core/
│   ├── models.py          # Asset / Finding / ScanResult / Report / Rule
│   ├── scoring.py         # 0-100 评分与等级
│   ├── config.py          # 配置（文件 + AISS_* 环境变量）
│   ├── isolation.py       # 沙箱、安全解压、脱敏、Prompt 隔离
│   ├── target.py          # 目录 / 文件 / 压缩包 / Git → 统一工作目录
│   ├── runner.py          # 扫描编排（CLI 与 Web 共用）
│   ├── baseline.py        # 基线生成与四态增量对比（new/fixed/persistent/regressed）
│   └── task.py            # ScanContext 与任务计时
├── engines/
│   ├── static.py          # 静态规则引擎（正则 + 证据定位 + 脱敏）
│   ├── dynamic.py         # AST 语义引擎：别名/混淆还原 + 污点传播
│   ├── llm.py             # LLM 语义分析（OpenAI 兼容接口，默认关闭）
│   └── target_client.py   # 黑盒靶机客户端（真实端点 + 脚本化靶机）
├── modules/
│   ├── base.py            # 四段式：静态规则 → AST → 结构化检查 → LLM 增强
│   ├── infra/             # 域 1：fingerprints / cve / manifests / scanner
│   ├── mcp/               # 域 2
│   ├── skill/             # 域 3
│   ├── agent_redteam/     # 域 4：attacks / judge / scanner
│   └── model_guard/       # 域 5：jailbreaks / judge / scanner
├── evals/                 # 大模型安全体检引擎（域 5 的增强链路）
│   ├── datasets.py        # 评测集仓库（内置集 + 自定义 JSON，name 字符集受控）
│   ├── attacks.py         # 14 个攻击算子：11 确定性 + 3 模型驱动
│   ├── evaluators.py      # rule / llm-judge / http-classifier
│   ├── runner.py          # 编排：抽样 × 增强 × 多轮 × 并发，错误分类
│   ├── report.py          # 双维度汇总 + badcase + CSV/JSONL 导出
│   └── service.py         # CLI 与 Web 共用的服务层（目标解析 / 落盘）
├── server/                # FastAPI 接口 + 任务队列（含规则包上传）
├── reporters/             # json / sarif / sbom / html(+Jinja2 模板) / console
├── rules/                 # custom + external + loader + importer（规则包导入导出）
└── utils/                 # fs（含 .aissignore）/ git / unicode
```

配套文件：`.github/workflows/aiss.yml`、`.gitlab-ci.yml`、`.pre-commit-config.yaml`、
`.aissignore.example`（改名为 `.aissignore` 放到目标根目录即生效），文档在 `docs/`。

---

## CI 集成示例

仓库里已有可直接使用的流水线：`.github/workflows/aiss.yml`（含 SARIF 上传与自动刷新基线）
与 `.gitlab-ci.yml`。核心两步：

```bash
# 主干：生成 / 刷新基线
aiss baseline save --path . --type infra,mcp,skill -o .aiss-baseline.json

# PR：只对新引入的风险失败，存量不阻塞
aiss diff .aiss-baseline.json --path . --fail-on-new high --markdown pr-comment.md
```

详见 [`docs/ci.md`](docs/ci.md)。

---

## LLM 增强（可选）

默认纯规则即可工作。开启后会把「已命中的文件」交给兼容 OpenAI 接口的模型做语义复核：

```bash
export AISS_LLM_ENABLED=1
export AISS_LLM_BASE_URL=http://localhost:11434/v1   # Ollama / vLLM 均可
export AISS_LLM_MODEL=qwen2.5:14b
aiss scan skill --path ./my-skill --llm
```

LLM 产生的发现带 `LLM-ANALYSIS` 规则 ID 与 `LLM` 标记，与规则发现区分展示。

黑盒两域用的是另一套配置（`AISS_TARGET_URL` / `AISS_TARGET_MODEL` / `AISS_TARGET_API_KEY`），
指向**被测** Agent 或模型，与审计用的 LLM 相互独立。

---

## 路线图

- **Phase 1（已完成）**：MCP + Skill 两域、三份报告、CLI、规则同步、隔离与脱敏
- **Phase 2（已完成）**：域 1 基础设施（组件 + CVE + 部署体检）、域 4 黑盒红队、FastAPI 任务接口
- **Phase 3（已完成）**：域 5 护栏与越狱测评、四种内置脚本化靶机人格
- **Phase 4（已完成）**：AST 语义引擎（别名/混淆还原 + 污点传播）、增量基线 y.diff、CycloneDX SBOM、CI 模板与 docs
- **Phase 5（已完成）**：正确性硬化——修复规则库自污染、**误报率从 85% 降到 ~1%**、
  扫描耗时降低 11 倍、发行包瘦身 15 倍（不再内含第三方规则）
- **Phase 6（已完成）**：JS/TS 语义引擎（Node 生态的别名/混淆还原）、配置中心（三层来源追溯）、
  `aiss watch` 持续监控、三套 HTML 报告模板
- **Phase 7（已完成）**：大模型安全体检引擎——评测集仓库（上传 JSON）、14 个攻击算子、
  三种评估器、多轮与并发执行、双维度报表与 Badcase 导出，控制台配套「越狱评测集 / 安全体检」两视图

---

## 配置中心

配置分三层加载，后者覆盖前者：

| 层级 | 位置 | 用途 |
| --- | --- | --- |
| L1 用户级 | `~/.aiss/config.yml` | 个人机器上的偏好 |
| L2 项目级 | `.aissrc` / `.aiss.yml` / `pyproject.toml` 的 `[tool.aiss]` | 随仓库提交、团队共享；从当前目录向上查找，遇 `.git` 停止 |
| L3 环境变量 | `AISS_*` | CI 注入、临时覆盖 |

```bash
aiss config --sources              # 看每个层级有没有生效、加载了哪些键
aiss config --get limits.max_files # 查单个值，并告诉你它来自哪一层
aiss config init                   # 生成 .aissrc 模板
aiss config set ast.min_confidence low
aiss config unset verbose
```

改了配置却没生效时，`--sources` 是第一手段：它会指出是哪一层把值写死了。
环境变量写法为「节名 + 字段名」蛇形大写，例如 `AISS_LIMITS_MAX_FILE_BYTES`、`AISS_FAIL_ON`。

## 多报告模板

同一份结果给不同受众看，`--template` 切换：

| 模板 | 场景 | 特点 |
| --- | --- | --- |
| `default` | 工程师本地排查 | 暗色、信息最全，含全部证据与片段 |
| `light` | 打印 / 导出 PDF / 贴进周报 | 浅色，`@media print` 优化，分页不切断问题项 |
| `compact` | CI 附件、工单系统 | 纯表格：等级 / 规则 / 位置 / 修复建议，体积约为 default 的 1/4 |

```bash
aiss scan mcp -p ./server --format html --template compact -o report.html
```

## 持续监控

`aiss watch` 把一次性扫描变成长期观测：**首轮自动作为基线，之后每轮只对比增量**，
避免每天被同一批存量问题淹没。

```bash
aiss watch ./my-mcp-server --type mcp --interval 300 --snapshot-dir ./history
aiss watch ./app --interval 600 --fail-on-new high --webhook https://hooks.example/xyz
```

只有出现**新增或等级劣化**时才打印差异并触发 webhook；`--fail-on-new` 命中即返回非零退出码，
可直接交给 supervisor / systemd 管理。`--snapshot-dir` 按轮次留痕 `round-0001.json`，随时可回溯。

## 文档

| 文档 | 内容 |
| --- | --- |
| [`docs/architecture.md`](docs/architecture.md) | 分层架构、扫描链路、自安全设计、五域落地形态 |
| [`docs/rules.md`](docs/rules.md) | 自定义规则 YAML 编写指南、glob 作用域的三个坑、调试方法 |
| [`docs/ci.md`](docs/ci.md) | 增量门禁、指纹算法、退出码、`.aissignore`、SARIF 上传 |
| [`docs/extending.md`](docs/extending.md) | 如何新增一个扫描域（五步） |
| [`docs/delivery.md`](docs/delivery.md) | 交付说明：范围、目录结构、安装运行、去冗余清单、已知约束 |
| [`docs/acceptance.md`](docs/acceptance.md) | 验收清单：逐条可勾选的验收项与验证命令 |
| [`CHANGELOG.md`](CHANGELOG.md) | 版本变更记录 |

## 开发

```bash
pip install -e ".[dev,server]"

pytest -q                            # 451 项测试
mypy aiss --ignore-missing-imports   # 类型检查，当前 0 错误（62 个文件）
```

两条命令都与 CI 门禁一致；改动 `aiss/` 后请确保两者都通过。

质量约束（由 `tests/test_hardening.py` 守着，改坏了会直接失败）：

- 扫描器自身的规则库/模板目录**不会**被当作被测源码（否则 CVE 描述里的
  `http://`、`os.system` 会被自己的规则再次命中）
- 所有函数注解必须可被 `typing.get_type_hints` 反射（开着 postponed annotations，
  漏 import 只有到这一步才炸）
- `pyproject.toml` 的 `package-data` 不得命中第三方规则目录

## 致谢与来源

- 主架构参考 [Tencent/AI-Infra-Guard](https://github.com/Tencent/AI-Infra-Guard)（外部规则），
  规则包通过 `aiss rules sync` 同步，版权归原作者所有，见 `NOTICE`。
- 技能检测参考 [NVIDIA/SkillSpector](https://github.com/NVIDIA/SkillSpector)、
  [snyk/agent-scan](https://github.com/snyk/agent-scan)。
- MCP 检测参考 Ramparts、aguara、mcp-armor、Cisco mcp-scanner，以及
  [awesome-ai-security-tools](https://github.com/scadastrangelove/awesome-ai-security-tools) 的清单。
- 风险分类对齐 SkillTrustBench T01–T09、OWASP MCP Top 10 与 OWASP LLM Top 10。
- CVSS v3.1 基础分按 [FIRST 规范](https://www.first.org/cvss/specification-document) 实现。
- Web 控制台标识（logo）由 [jenn619](https://github.com/jenn619) 提供，版权归原作者所有。

## 许可证

本项目本身采用 **MIT** 许可证（见 `LICENSE`）。

通过 `aiss rules sync` 拉取的外部同步规则数据与检测模板**不属于本项目**，
版权归各自原作者所有，遵循其原始许可证；详细的来源与用途说明见 `NOTICE`。
这些内容不会打包进发行版，而是在你本地首次执行 `aiss rules sync` 时下载。

## 免责声明

本项目仅用于授权的安全评估与自查。域 4 / 域 5 的攻击用例与越狱语料只能用于**你拥有或已获授权**
的 Agent 与模型；请勿用于未授权目标。扫描结论需结合人工复核。
