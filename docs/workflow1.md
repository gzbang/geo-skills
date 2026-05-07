# openEuler GEO 优化工作流闭环建设

## Context

当前两个仓库各司其职但没接起来:

- **geo-workflow** (`/Users/gzbang/Desktop/Project/10_Common/Code/geo-workflow`) 已经把 "采样→评分→建 Issue→报告" 跑通,openEuler 已积累 5 次评估周期、104 道问题集
- **openEuler-portal** (`/Users/gzbang/Desktop/Project/01_openEuler/Code/openEuler-portal`) 是 VitePress SSG,sitemap/robots/TDK/Schema/llms.txt 这五项已落地,优化注入点是 `transformPageData`、`buildEnd`、sitemap 插件

缺的不是工具,而是**闭环**:Issue 提出后没有"修复→回归→再优化"的状态机;修复行为是否落地、AI 是否重新识别,目前没有任何机制区分这两件事;每轮迭代靠人记忆,没有趋势可见。

本计划目标:建立 **问题验证 → 修复 → 双层效果验证 → 再优化** 的 CI 化闭环,以 GitCode Issue 作为唯一状态源,以 label 流转表达生命周期,以 portal CI 做静态验证、geo-workflow CI 做动态回归。

---

## 总体架构

```
┌──────────────── geo-workflow (评估侧) ────────────────┐
│ ① 月度全量 assessment (cron)                          │
│   get-question → platform-chat → scoring-engine       │
│   → issue-creator(打 p0/p1/p2 + type + owner label)   │
│                                                        │
│ ② 周度子集回归 (cron, 只跑待验证 Issue 关联的问题)    │
│   platform-chat --scope-issue=NNN,NNN                 │
│   → scoring-engine → 比对历史分 → 切 label            │
│                       │                                │
└───────────────────────┼────────────────────────────────┘
                        │ GitCode Issue (单点状态)
                        ▼
            ┌──────────────────────────┐
            │   Issue label 状态机     │
            │  triaged → fixing →      │
            │  merged-pending-recheck  │
            │  → verified-improved /   │
            │    verified-no-effect    │
            └──────────────────────────┘
                        ▲
                        │ PR commit "Fixes openeuler-geo#NNN"
┌───────────────────────┼────────────────────────────────┐
│ openEuler-portal (修复侧)                             │
│ ③ PR/push CI: SEO precheck                            │
│   pnpm build → analyze.js → 校验 sitemap/llms.txt     │
│   → 失败 fail PR                                       │
│ ④ 工程类修复:/optimize-meta /generate-schema 直改文件 │
│ ⑤ 内容类修复:开内容工单 → SIG 审稿 → PR              │
└────────────────────────────────────────────────────────┘
```

---

## Phase 0 — Issue Label 状态机(全局规约)

在 GitCode 项目内统一以下 label,所有自动化都基于这套约定:

| 维度 | Label | 含义 | 谁打 |
|---|---|---|---|
| 严重度 | `geo:p0/p1/p2` | scoring-engine 已有 | 自动 |
| 类型 | `geo:type/tdk` | TDK 缺失/不优 | triage 自动 |
| 类型 | `geo:type/schema` | Schema 缺失 | triage 自动 |
| 类型 | `geo:type/llms-txt` | llms.txt 未覆盖 | triage 自动 |
| 类型 | `geo:type/content-missing` | 官网缺该主题内容 | triage 自动 |
| 类型 | `geo:type/no-cite` | SEO 完好但 AI 不引用(权重/外链问题) | triage 自动 |
| 责任方 | `geo:owner/frontend` | 工程层可自闭环 | triage 自动 |
| 责任方 | `geo:owner/sig-<name>` | 需协作内容 | triage 自动+人工修正 |
| 状态 | `geo:status/triaged` | 已分诊,可被认领 | triage |
| 状态 | `geo:status/fixing` | 有 PR 在开发 | PR open hook |
| 状态 | `geo:status/merged-pending-recheck` | PR 合入待回归 | PR merge hook |
| 状态 | `geo:status/verified-improved` | 周回归得分提升,自动 close | 回归 CI |
| 状态 | `geo:status/verified-no-effect` | 回归无变化/下降 | 回归 CI |
| 重试 | `geo:retry/<N>` | 第 N 次再优化 | 回归 CI |

**关键约定**:不再单独建看板,Issue + label 即看板。GitCode 默认看板按 label 过滤即可。

---

## Phase 1 — 问题验证(triage)

**目标**:scoring-engine 输出的 Issue 是"被引用率"信号,但不一定是"工程问题",需要二次分诊确定修复路径。

**新增能力**:在 geo-workflow 内新增 skill `triage-issue`(放在 `.claude/skills/triage-issue/`)。

输入:`assessments/<community>/<date>/scoring-results.json` + `issue-map.json`

执行步骤:
1. 对每个 Issue,从 issue-map 拿到 `official_urls`(若没有则跳过 owner 判断,标 `content-missing`)
2. 对每个 URL 调用 portal 已部署版本,跑 `geo-workflow/scripts/analyze.js`(已存在)做静态分析
3. 决策树:
   - URL 404 / 内容明显单薄 → `type/content-missing` + `owner/sig-<推断>`
   - TDK 字段缺/超长 → `type/tdk` + `owner/frontend`
   - 无 JSON-LD 或 schema 不匹配主题 → `type/schema` + `owner/frontend`
   - 该 URL 不在 llms.txt 里 → `type/llms-txt` + `owner/frontend`
   - 静态 SEO 全部 OK 但仍未被引用 → `type/no-cite` + `owner/frontend`(进再优化区)
4. 通过 GitCode API 给 Issue 追加 label,并附 `triage-report.md` 评论

**SIG 推断**:复用 portal 的 `app/.vitepress/jsonld/sigs/*.json` 做映射(URL 路径里出现的 SIG 名 → owner)。

---

## Phase 2 — 修复

### 2.1 工程类修复(`owner/frontend`)

完全在 portal 仓库内完成,按 type 走对应 skill:

| Type | 修复入口 | Skill |
|---|---|---|
| `tdk` | 改 `app/.vitepress/sitemap/sitemap-zh.ts` 或 `app/.vitepress/tdks.ts` | `/optimize-meta` |
| `schema` | 改 `app/.vitepress/jsonld/general.ts` 或 `jsonld/sigs/*.json` | `/generate-schema` |
| `llms-txt` | 改 `app/.vitepress/LLMsTxtSections.ts` | 直接编辑 |
| `no-cite` | 进入 Phase 4(再优化),不直接修代码 | — |

**PR 提交规约**:
- branch:`geo/<issue-id>-<short-slug>`
- commit:`fix(geo): #<issue-id> <一句话>`
- PR 描述模板必须含 `Fixes openeuler-geo#<id>`,GitCode 自动 link

新增文件:`openEuler-portal/.github/PULL_REQUEST_TEMPLATE.md`(强制 GEO Issue 关联字段)。

### 2.2 内容类修复(`owner/sig-<name>`)

需要协作,设独立工单流:

- portal 仓库新增 `.github/ISSUE_TEMPLATE/content-request.yml`,前端发起内容工单(包含原 GEO Issue 链接、目标 URL、关键词、首段定义句草稿)
- SIG 维护者在内容工单里 review,/write-content 出初稿,合并到 portal Markdown
- 内容 PR 合入后,反向把 GEO Issue label 推进到 `merged-pending-recheck`

---

## Phase 3 — 双层效果验证

### 3.1 L1 静态验证(portal CI,PR 即时)

**新增**:`openEuler-portal/.github/workflows/seo-precheck.yml`

```yaml
on: [pull_request, push]
jobs:
  precheck:
    steps:
      - checkout portal
      - checkout geo-workflow (sibling, 取 scripts/)
      - pnpm install && pnpm build
      - node ../geo-workflow/scripts/analyze.js dist/zh/<受影响页面>
      - 校验 dist/sitemap.xml 合法、URL 数无非预期下降
      - 校验 dist/llms.txt 大小在合理区间
      - 校验 dist/llms-full.txt 包含本次 PR 改动的 URL(如有)
      - 任一失败 fail PR
```

**作用**:确保"修复行为已落地",防止 TDK 改回去、Schema 写错 JSON、llms.txt 漏注册。**不验证 AI 是否引用**。

同时把 portal 现有 `husky/pre-commit` 里被注释掉的 `lint-staged` 解开,本地提交即跑 ESLint/Stylelint。

### 3.2 L2 动态回归(geo-workflow CI,周度)

**核心扩展**:在 `platform-chat` 和 `scoring-engine` 上加 `--scope-issue=<id>[,<id>...]` 参数,只对这些 Issue 关联的 question 子集做采样和打分。这是回归能跑得起的前提 —— 全量采样太重。

**新增**:`geo-workflow/.github/workflows/weekly-recheck.yml`

```yaml
on:
  schedule: ['0 1 * * 1']   # 每周一 09:00 北京时间
jobs:
  recheck:
    steps:
      - 通过 GitCode API 拉所有 label=geo:status/merged-pending-recheck 的 Issue
        + 关联 community 维度分组
      - 对每个 community: platform-chat --scope-issue=N,N,N
      - scoring-engine --scope-issue=N,N,N --compare-with=<上次该 Issue 评分>
      - 对每个 Issue:
          得分提升 → label 切 verified-improved + 自动 close + 评论附比对数据
          得分无变化/下降 → label 切 verified-no-effect + 累加 geo:retry/<N+1>
      - 推 weekly-summary.md 到 assessments/<community>/<date>/
```

**月度全量**:`monthly-full.yml`(每月 1 号),跑完整 assessment、生成 trend-report 跨月对比引用率。

### 3.3 双层是否冲突

L1 验证"代码合规",L2 验证"AI 认可",分别对应 Issue 的 `merged-pending-recheck` 这个状态切片。代码合并瞬间 → L1 通过 → label 进 `merged-pending-recheck` 等 L2;L2 周一跑完 → label 切 `verified-improved` 或 `verified-no-effect`。两层之间是"代码已交付,等市场反馈"的等待区。

---

## Phase 4 — 再优化

由 L2 标记为 `verified-no-effect` 的 Issue 进入此阶段。

**自动**:
- `geo:retry/<N>` 计数 +1
- 第 1 次失败:自动给 Issue 评论 "建议尝试 X(如:补充 llms-full.txt 该主题段落 / 在首页加入口 / 加 FAQ Schema)"
- 第 2 次失败后:自动把 Issue 升级 label `geo:p0` 提请人工归因

**人工**:
- 归因维度:页面在 sitemap 优先级、是否被首页/导航引用、是否有外链、内容深度、首段定义句、FAQ 覆盖
- 沉淀经验到 `geo-workflow/docs/patterns.md`(共性模式)和 `geo-workflow/assessments/<community>/learnings.md`(社区私有经验)

**月度趋势**:`monthly-full.yml` 输出的 trend-report 提供"修复有效率"指标(verified-improved / 已合并 PR 数),作为工作流自身的健康度信号。

---

## 落地清单与文件变更

### geo-workflow 侧

| 操作 | 路径 | 说明 |
|---|---|---|
| 新增 skill | `.claude/skills/triage-issue/SKILL.md` | Phase 1 自动分诊,用 `skill-creator` 脚手架 |
| 扩展 skill | `.claude/skills/platform-chat/SKILL.md` | 加 `--scope-issue` 参数支持子集 |
| 扩展 skill | `.claude/skills/scoring-engine/SKILL.md` | 加 `--scope-issue` + `--compare-with` |
| 扩展 skill | `.claude/skills/issue-creator/SKILL.md` | 创建 Issue 时自动打 type/owner label |
| 新增 CI | `.github/workflows/weekly-recheck.yml` | 周一定时子集回归 |
| 新增 CI | `.github/workflows/monthly-full.yml` | 月度全量 + trend report |
| 新增 doc | `docs/label-spec.md` | Phase 0 label 规约 |
| 新增 doc | `docs/patterns.md` | Phase 4 经验沉淀 |
| 复用 | `scripts/analyze.js`, `scripts/crawl.js`, `scripts/tdk.js` | triage 阶段调用,无需改 |

### openEuler-portal 侧

| 操作 | 路径 | 说明 |
|---|---|---|
| 新增 CI | `.github/workflows/seo-precheck.yml` | Phase 3 L1 静态验证 |
| 新增模板 | `.github/PULL_REQUEST_TEMPLATE.md` | 强制 `Fixes openeuler-geo#NNN` |
| 新增模板 | `.github/ISSUE_TEMPLATE/content-request.yml` | Phase 2.2 内容工单 |
| 改动 | `husky/pre-commit` | 解开 `lint-staged` 注释 |
| 复用 | `app/.vitepress/sitemap/sitemap-zh.ts`、`tdks.ts`、`jsonld/`、`LLMsTxtSections.ts` | 工程类修复直改这几个文件,无需新建机制 |

### 桥接

- geo-workflow 的 weekly-recheck CI 通过 `GITCODE_TOKEN`(已在 .env.example)读 portal 的 commit/PR 信息做关联
- 不引入额外 DB,Issue label + commit 信息即关联表

---

## 验证(交付后怎么知道工作流跑得通)

按以下顺序在真实环境跑一次完整闭环,每步可见预期结果:

1. **Phase 1 验证**:在 geo-workflow 跑 `triage-issue` 对最近一次(2026-04-13)openEuler 评估的 P0 Issue 子集分诊 → 检查 GitCode Issue 上 label 是否正确(随机抽 3 个 Issue 人工核对 type 和 owner)
2. **Phase 2.1 验证**:在 portal 挑一个 `type/tdk` 的工程类 Issue,走完 `/optimize-meta` → PR(commit 含 Fixes openeuler-geo#NNN)→ seo-precheck CI 绿 → 合并 → Issue label 自动切到 `merged-pending-recheck`
3. **Phase 3.1 验证**:故意在 PR 里把某个页面 title 删掉 → seo-precheck 必须 fail
4. **Phase 3.2 验证**:手动触发 weekly-recheck CI(workflow_dispatch),只对上一步合并的 Issue 跑回归 → 预期 `verified-improved` 或 `verified-no-effect` label 之一被打上,Issue 上有比对数据评论
5. **Phase 4 验证**:对一个被标 `verified-no-effect` 的 Issue,确认 `geo:retry/1` 已加,且自动评论给出再优化建议
6. **闭环 SLA 检查**:从 Issue 创建到 verified 的中位时间应该 ≤ 14 天(7 天开发 + 7 天等下次回归)

---

## 不在范围内的事

- 不引入 SaaS 看板(Notion / Linear),Issue + label 已够用
- 不做 Lighthouse / Search Console 集成,这是传统 SEO 范畴,与 GEO 评估正交,可后续单独立项
- 不做 PR 合并即时回归(即"PR merge 触发"方案),原因是 AI 索引时延会让首次回归大概率失败,徒增噪音
- 不为 backlink 建设设流程(`type/no-cite` 的 Issue 进入 Phase 4 人工区,后续视积累情况再独立设计)
