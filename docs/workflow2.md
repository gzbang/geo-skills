# workflow2.md — openEuler-portal GEO 开发工作流设计

## 一、工作流目标

建立「新页面开发 → 本地预览验证 → 上线后复检」的闭环工作流，确保每个新页面上线前完成 SEO/GEO 全量检查。

适用场景：
- 新页面开发
- 重大内容改版
- SEO 问题修复验证

---

## 二、工作流架构

```
┌────────────────────────────────────────────────────────────────────┐
│                    openEuler-portal GEO 开发工作流                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Phase 1: 开发阶段                                                 │
│  ───────────────────                                               │
│  1. /geo:new-page-init     → 关键词研究 + 内容规划                  │
│  2. /geo:write-content     → GEO 优化正文生成                      │
│  3. /geo:optimize-meta     → TDK + OG 标签配置                     │
│  4. /geo:generate-schema   → JSON-LD 结构化数据                    │
│                                                                    │
│  Phase 2: 本地验证阶段                                             │
│  ───────────────────                                               │
│  5. 本地开发服务器启动   → vitepress dev app                       │
│  6. /geo:local-check       → 本地页面 SEO 诊断                     │
│     └─ 调用 geo-skills crawl.js (browser模式)                      │
│     └─ 调用 geo-skills analyze.js                                  │
│     └─ 对比预期配置 vs 实际渲染                                    │
│                                                                    │
│  Phase 3: 修复迭代                                                 │
│  ───────────────────                                               │
│  7. 修复问题 → 重新验证 → 循环直到通过                             │
│                                                                    │
│  Phase 4: 上线后复检                                               │
│  ───────────────────                                               │
│  8. 等待索引更新 (7-14天)                                          │
│  9. /geo:post-deploy-check → 生产环境复检                          │
│     └─ 调用 geo-skills crawl.js (http模式)                         │
│     └─ 对比本地审计 vs 线上实际                                    │
│                                                                    │
│  Phase 5: GEO 效果追踪（可选）                                     │
│  ───────────────────                                               │
│  10. 关联 geo-workflow → AI 引用率追踪                             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 三、文件结构

在 openEuler-portal 项目中新增以下结构：

```
openEuler-portal/
├── .claude/
│   ├── AGENT.md                    # 主工作流编排（新增）
│   └── commands/
│       ├── new-page-init.md        # 关键词研究 + 内容规划（新增）
│       ├── write-content.md        # GEO 优化正文（复用 geo-skills）
│       ├── optimize-meta.md        # TDK + OG 标签（复用 geo-skills）
│       ├── generate-schema.md      # JSON-LD 生成（复用 geo-skills）
│       ├── local-check.md          # 本地 SEO 诊断（新增）
│       ├── post-deploy-check.md    # 上线后复检（新增）
│       └── quick-seo-check.md      # 快速单页检查（新增）
│
├── geo-audit/                      # 审计记录目录（新增）
│   ├── local/                      # 本地验证记录
│   │   └── .gitkeep
│   └── deployed/                   # 上线后复检记录
│       └── .gitkeep
│
├── app/.vitepress/
│   ├── tdks/                       # TDK 配置（已有）
│   │   ├── zh.ts                   # 中文页面 TDK
│   │   ├── en.ts                   # 英文页面 TDK
│   │   └── index.ts                # 导出汇总
│   ├── jsonld/                     # JSON-LD 配置（已有）
│   │   ├── general.ts              # 通用页面 schema
│   │   └── sigs/                   # SIG 专用 schema
│   └── plugins/                    # 构建插件（已有）
│       ├── generateLLMsTxtNew/     # llms.txt 生成
│       └── lastModifiedPlugin.ts   # lastmod 时间戳
```

---

## 四、检查项清单

### 4.1 基础 SEO 检查项

| 类别 | 检查项 | 通过标准 | 严重级别 |
|------|--------|----------|----------|
| **基础 Meta** | Title 长度 | 30-60字符 | Critical |
| | Description 长度 | 120-160字符 | Critical |
| | Keywords | 非空，包含目标关键词 | Important |
| | Title 关键词 | 包含在标题前半部分 | Important |
| **技术标签** | Canonical | 存在且正确指向自身 | Critical |
| | Hreflang | 多语言页面必须有 | Important |
| | Robots meta | 无 noindex/nofollow | Critical |
| **社交分享** | og:title | 存在，与 title 一致 | Important |
| | og:description | 存在，与 description 一致 | Important |
| | og:image | 存在（首页必须，其他可豁免） | Important |
| | og:url | 存在，与 canonical 一致 | Minor |
| | twitter:card | 存在（summary_large_image） | Minor |
| | twitter:title | 存在 | Minor |
| **结构化数据** | JSON-LD 有效性 | 至少1个有效 schema | Critical |
| | Schema 类型匹配 | 符合页面内容类型 | Important |
| | FAQPage | Q&A 类页面必须有 | Critical |
| **标题层次** | H1 数量 | 有且仅1个 | Critical |
| | H1 关键词 | 包含目标关键词 | Important |
| | H2 数量 | ≥1个，结构清晰 | Important |
| | H1-H3 层次 | 逻辑层级，无跳跃 | Minor |
| **图片** | alt 覆盖率 | ≥95% | Critical |
| | 空 alt 数量 | =0 | Important |
| | alt 关键词 | 重要图片包含关键词 | Minor |
| **链接** | 内部链接 | ≥3个相关页面链接 | Important |
| | 外部链接 | ≤10个，有 rel=nofollow | Minor |
| | 锚文本描述性 | 避免「点击这里」 | Minor |

### 4.2 GEO 特定检查项

| 检查项 | 通过标准 | 严重级别 |
|--------|----------|----------|
| 首段定义句 | 150字内有直接定义（含具体数字+来源） | Critical |
| 数据出处标注 | 每个统计数字注明来源机构和日期 | Critical |
| FAQ 区块 | 覆盖5-8个用户常见问题 | Critical |
| FAQPage JSON-LD | FAQ 问题有对应 schema | Critical |
| 内容结构化 | 对比数据用表格，规格用列表 | Important |
| 内链密度 | 相关页面链接≥5个 | Important |
| llms.txt 包含 | 页面路径在 llms.txt 中 | Important |
| llms-full.txt 内容 | 页面正文已包含 | Important |

### 4.3 按页面类型的额外检查

| 页面类型 | 额外检查项 |
|----------|-----------|
| **首页 (home)** | og:image 必须 + FAQPage schema + Organization schema |
| **博客/新闻 (blog/news)** | Article schema + 作者信息 + 发布日期 + 修改日期 |
| **SIG 详情页 (sig/detail)** | Organization + CollectionPage + FAQPage 三合一 |
| **下载页 (download)** | Product/SoftwareApplication schema + 版本信息 |
| **迁移专区 (migration)** | HowTo schema（如有步骤指南）+ FAQPage |
| **用户案例 (showcase)** | CaseStudy schema + 用户信息 + 产品信息 |
| **Q&A 页面** | FAQPage schema（必须） |

---

## 五、核心 Skill 设计

### 5.1 `/geo:local-check` — 本地 SEO 诊断

**位置**：`openEuler-portal/.claude/commands/local-check.md`

**功能**：抓取本地开发服务器页面，运行 SEO 全量检查，对比预期配置。

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| url | string | 是 | 本地页面 URL，如 `http://localhost:5173/zh/new-page/` |
| page_type | string | 是 | 页面类型：home/blog/sig/detail/download/migration/other |
| target_keyword | string | 推荐 | 目标关键词，用于评分 |
| geo_skills_path | string | 否 | geo-skills 路径，默认 `/Users/gzbang/Desktop/Project/10_Common/Code/geo-skills` |

**工作流程**：

1. **读取预期配置**
   - 从 `app/.vitepress/tdks/zh.ts`（或 `en.ts`）读取预期 TDK
   - 从 `app/.vitepress/jsonld/general.ts` 或 `jsonld/sigs/*.jsonld.json` 读取预期 schema

2. **抓取本地页面**
   ```bash
   node {geo_skills_path}/scripts/crawl.js {url} --mode=browser --out=/tmp/geo-local-check --format=html
   ```

3. **运行 SEO 诊断**
   ```bash
   node {geo_skills_path}/scripts/analyze.js /tmp/geo-local-check.html
   ```

4. **对比预期 vs 实际**
   - Title: 预期 vs 渲染结果
   - Description: 预期 vs 渲染结果
   - Keywords: 预期 vs 渲染结果
   - JSON-LD: 预期 schema vs 渲染结果
   - OG 标签: 是否存在 vs 渲染结果

5. **生成诊断报告**
   - 按严重级别分类（Critical / Important / Minor）
   - 每项包含：问题描述 + 预期值 + 实际值 + 修复建议
   - 计算总分（100分制）

6. **输出修复清单**
   - 优先级排序的修复清单
   - 可执行的修复步骤

**输出格式示例**：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
本地 SEO 诊断报告
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

页面: {url}
页面类型: {page_type}
目标关键词: {keyword}
诊断时间: {timestamp}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
总体评分: XX/100
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[████████████████████░░░░░░░░░░░░░░░░░░░░] XX%

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Critical 问题 (必须修复)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| 检查项 | 预期 | 实际 | 修复建议 |
|--------|------|------|----------|
| Title 长度 | 30-60字符 | XX字符 | 调整标题长度 |
| H1 数量 | 1个 | XX个 | 修正标题结构 |
| JSON-LD | 有效schema | 无/无效 | 添加 {schema_type} |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
修复清单（优先级排序）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. [Critical] 修复 H1 数量问题
   → 操作：检查 frontmatter.title 或正文标题结构

2. [Critical] 添加 FAQPage JSON-LD
   → 操作：在 jsonld/general.ts 中添加配置
```

---

### 5.2 `/geo:post-deploy-check` — 上线后复检

**位置**：`openEuler-portal/.claude/commands/post-deploy-check.md`

**功能**：验证生产环境页面 SEO 效果，对比本地审计结果。

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| url | string | 是 | 生产环境页面 URL |
| prev_audit | string | 推荐 | 上次本地审计报告路径 |
| geo_skills_path | string | 否 | geo-skills 路径 |

**工作流程**：

1. **抓取生产环境页面**（http 模式）
2. **运行 SEO 诊断**
3. **对比本地 vs 线上**
4. **验证额外项**：
   - HTTP 状态码 = 200
   - robots.txt 未阻止
   - sitemap.xml 包含
   - llms.txt 包含

---

### 5.3 `/geo:quick-seo-check` — 快速单页检查

**位置**：`openEuler-portal/.claude/commands/quick-seo-check.md`

**功能**：快速检查任意 URL 的 SEO 基础状态，不生成完整报告。

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| url | string | 是 | 任意 URL（本地或线上） |
| mode | string | 否 | browser（本地）/ http（线上），默认自动判断 |

---

### 5.4 `/geo:new-page-init` — 新页面规划

**位置**：`openEuler-portal/.claude/commands/new-page-init.md`

**功能**：整合关键词研究 + 内容结构规划 + SEO 配置建议。

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| page_path | string | 是 | 页面路径，如 `zh/migration/new-guide` |
| page_type | string | 是 | 页面类型 |
| target_keyword | string | 推荐 | 目标关键词 |

**输出**：
- 关键词分析
- H1/H2/H3 结构规划
- FAQ 问题列表（5-8个）
- TDK 配置建议
- JSON-LD schema 建议
- OG 标签建议
- 首段定义句模板

---

## 六、AGENT.md 主工作流编排

见 `openEuler-portal/.claude/AGENT.md` 文件内容。

---

## 七、实施步骤

### 7.1 创建目录结构

```bash
mkdir -p openEuler-portal/.claude/commands
mkdir -p openEuler-portal/geo-audit/local
mkdir -p openEuler-portal/geo-audit/deployed
```

### 7.2 创建文件清单

| 文件 | 位置 | 状态 |
|------|------|------|
| AGENT.md | .claude/ | 新增 |
| local-check.md | .claude/commands/ | 新增 |
| post-deploy-check.md | .claude/commands/ | 新增 |
| quick-seo-check.md | .claude/commands/ | 新增 |
| new-page-init.md | .claude/commands/ | 新增 |
| write-content.md | .claude/commands/ | 复用 geo-skills |
| optimize-meta.md | .claude/commands/ | 复用 geo-skills |
| generate-schema.md | .claude/commands/ | 复用 geo-skills |

### 7.3 OG 标签补充方案

当前 openEuler-portal 缺少 OG 标签配置，建议在 `config.ts` 中添加：

```typescript
const setOgTags = (pageData: PageData, pagePath: string) => {
  const locale = pageData.filePath.slice(0, 2) as 'zh' | 'en';
  const tdkInfo = tdks[locale]?.[pagePath];
  pageData.frontmatter.head ??= [];
  
  if (tdkInfo) {
    pageData.frontmatter.head.push(
      ['meta', { property: 'og:title', content: tdkInfo.title }],
      ['meta', { property: 'og:description', content: tdkInfo.description }],
      ['meta', { property: 'og:url', content: `${sitemapHostname}/${pagePath}/` }]
    );
  }
  
  if (pagePath === 'zh' || pagePath === 'en') {
    pageData.frontmatter.head.push(
      ['meta', { property: 'og:image', content: `${sitemapHostname}/images/og-home.png` }]
    );
  }
};
```

---

## 八、与 geo-workflow 的关联

### 8.1 数据流向

```
openEuler-portal (开发)
    │
    ├─ 新页面上线
    │
    └─→ geo-workflow (评估)
         │
         ├─ questions.json 添加问题
         ├─ 填写 official_urls
         ├─ 运行采样评分
         │
         └─→ 评估报告
              │
              ├─ 引用率 OK → 结束
              │
              └─→ 引用率低 → 反馈到 openEuler-portal
                    │
                    ├─ 优化内容可引用性
                    ├─ 加强 FAQ 区块
                    ├─ 添加数据出处
                    │
                    └─→ 重新上线 → geo-workflow 复检
```

### 8.2 问题映射

| geo-workflow 评估结果 | openEuler-portal 优化动作 |
|-----------------------|---------------------------|
| P0（有内容未引用） | 优化内容结构、加强 FAQ、添加定义句 |
| P1（官方内容缺失） | 创建新页面，按本工作流开发 |

---

## 九、版本记录

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0 | 2026-05-07 | 初版设计 |