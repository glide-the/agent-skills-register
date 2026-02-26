# 能力兼容性矩阵与实现方案

> **文档类型**：解决方案设计  
> **日期**：2026-02-25  
> **输入来源**：  
> - [gap-analysis-clawhub-vs-design.md](gap-analysis-clawhub-vs-design.md)（ClawHub 实际代码 vs 设计文档差异）  
> - [gap-analysis-vercel-skills.md](gap-analysis-vercel-skills.md)（设计文档 vs vercel-skills 差异）  
> **目标**：横向对比 ClawHub 与 vercel-skills 在 **Registry**、**CLI 客户端**、**Skill Store** 三大维度的能力覆盖率，并给出私有化 Registry 的多套实现方案。

---

## 目录

1. [三维能力兼容性总览](#1-三维能力兼容性总览)
2. [Registry（仓库/注册表）能力矩阵](#2-registry仓库注册表能力矩阵)
3. [CLI 客户端能力矩阵](#3-cli-客户端能力矩阵)
4. [Skill Store（商店）能力矩阵](#4-skill-store商店能力矩阵)
5. [实现方案](#5-实现方案)
   - 5.1 [方案 A：ClawHub-First（最大兼容方案）](#51-方案-aclawhub-first最大兼容方案)
   - 5.2 [方案 B：Hybrid-Forge（混合创新方案）](#52-方案-bhybrid-forge混合创新方案)
   - 5.3 [方案 C：Clean-Slate（全新自建方案）](#53-方案-cclean-slate全新自建方案)
6. [方案对比决策矩阵](#6-方案对比决策矩阵)
7. [推荐路径与里程碑](#7-推荐路径与里程碑)

---

## 1. 三维能力兼容性总览

### 1.1 评估维度定义

| 维度 | 定义 | 示例场景 |
|------|------|---------|
| **Registry（仓库/注册表）** | 服务端能力：包存储、版本管理、API 端点、认证授权、安全扫描、数据模型 | 包上传/下载、版本管理、搜索索引、安全扫描 |
| **CLI 客户端** | 命令行工具：安装、发布、更新、配置管理、Agent 分发 | `skill install / publish / search / update` |
| **Skill Store（商店）** | 用户体验层：搜索发现、评级排名、安全展示、社区交互 | 搜索 UI、安全报告展示、Stars/评论、排行榜 |

### 1.2 总览矩阵

| 能力维度 | ClawHub 覆盖度 | vercel-skills 覆盖度 | 私有化需求满足度 | 
|---------|---------------|---------------------|-----------------|
| **Registry 服务端** | 🟡 65% | 🔴 5% | 需大量新建 |
| **CLI 客户端** | 🟡 70% | 🟢 85% | 可借鉴整合 |
| **Skill Store** | 🟡 55% | 🟡 30% | 需补齐安全+企业功能 |

---

## 2. Registry（仓库/注册表）能力矩阵

### 2.1 核心 API 端点

| 关键需求 | ClawHub | vercel-skills | 私有化 Gap | 优先级 |
|---------|---------|---------------|-----------|--------|
| **Skill CRUD** (`GET/POST/DELETE /skills`) | ✅ 完整（slug 路由，V1 API） | ❌ 无服务端 | ClawHub 可复用，需修正 multipart 上传协议 | P0 |
| **版本管理** (`/skills/:slug/versions`) | ✅ 支持版本列表/详情/文件查看 | ❌ 无版本概念（用 Git tree SHA） | ClawHub API 基本可用 | P0 |
| **包下载** (`GET /download`) | ✅ 动态生成 zip（`buildDeterministicZip`） | ❌ 直接 git clone | 需注意动态 zip 对缓存策略的影响 | P0 |
| **搜索** (`GET /search`) | ✅ 混合搜索（向量+lexical），无分页 | ⚠️ 依赖外部 `skills.sh` API | 需新建全文+向量搜索引擎，可参考 ClawHub 排序算法 | P0 |
| **认证授权** | ⚠️ Bearer Token 单一模式，无 scope | ❌ 仅 GitHub Token 限速 | 需新建 OAuth2 + API Key + scope 体系 | P0 |
| **文件上传** | ⚠️ 逐文件上传 Convex Storage（非 presigned URL 列表） | ❌ 无 | 需适配或重新设计两阶段上传协议 | P0 |
| **Tag 管理** | ⚠️ 无 HTTP 端点，仅 Convex mutation + 发布时隐式设置 | ❌ 无 | 需新建 REST 端点 | P1 |
| **Resolve（版本匹配）** | ✅ `GET /resolve?slug=X&hash=Y` 基于 fingerprint | ❌ 无 | 设计文档遗漏，需补充 | P1 |
| **Stars/收藏** | ✅ `POST/DELETE /stars/:slug` | ❌ 无 | 社区功能，可选 | P2 |
| **Souls（人格配置）** | ✅ 完整 CRUD + 版本 + 搜索 | ❌ 无 | 设计文档完全遗漏，需评估 | P2 |

### 2.2 安全与扫描

| 关键需求 | ClawHub | vercel-skills | 私有化 Gap | 优先级 |
|---------|---------|---------------|-----------|--------|
| **VirusTotal 集成** | ✅ 异步 cron（5 分钟 poll + 每日 rescan） | ❌ 无 | 可复用 ClawHub 扫描逻辑 | P0 |
| **LLM 安全分析** | ✅ OpenAI 异步评估（verdict + dimensions + findings） | ❌ 无 | 可复用 prompt 和评估框架 | P1 |
| **质量门控** | ✅ 同步质量评分（signals → score → pass/quarantine/reject） | ❌ 无 | **设计文档遗漏**，需补充 | P0 |
| **Moderation 状态机** | ✅ active/hidden/removed + moderationReason | ❌ 无 | 比设计文档的单字段 status 更丰富 | P0 |
| **cosign 包签名** | ❌ 不存在 | ❌ 不存在 | 需全新建设（设计文档 04 已规划） | P1 |
| **SBOM 生成** | ❌ 不存在 | ❌ 不存在 | 需全新建设 | P2 |
| **Quarantine 审批流** | ⚠️ 管理员手动 mutation，无专门 UI | ❌ 无 | 需新建审批端点和 UI | P1 |

### 2.3 数据模型

| 关键需求 | ClawHub | vercel-skills | 私有化 Gap | 优先级 |
|---------|---------|---------------|-----------|--------|
| **Skill 实体** | ✅ 丰富（quality, badges, forkOf, canonical, stats） | ⚠️ 简单（name, description, path 三字段） | ClawHub 模型可参考，需增加 namespace/visibility/risk_level | P0 |
| **SkillVersion 实体** | ✅ 含 fingerprint, parsed, vtAnalysis(内嵌), llmAnalysis(内嵌) | ❌ 无版本实体 | 需将内嵌分析字段改为独立表（便于查询扩展） | P0 |
| **Tags 结构** | ⚠️ 内嵌 Record（非独立表） | ❌ 无 | 可选择独立表（设计文档方案）或内嵌（ClawHub 方案） | P1 |
| **用户/组织/团队** | ⚠️ 仅用户（admin/moderator/user 三级） | ❌ 无用户系统 | 需全新建设多租户 RBAC（设计文档 08 已规划） | P0 |
| **API Token** | ⚠️ 有 token 但无 scope/expiry | ❌ 无 | 需扩展 scope + expiry + allowed_skills | P0 |
| **审计日志** | ⚠️ 简化版（无 IP/UserAgent） | ⚠️ 仅 telemetry fire-and-forget | 需全新建设 append-only 审计日志 | P1 |
| **Lock File** | ⚠️ 简单（version + installedAt，无 contentHash/tag/registry） | ✅ 两层设计（Global + Local） | 需融合：ClawHub 兼容 + vercel-skills 双层思路 | P0 |
| **Slug 保留** | ✅ `reservedSlugs` 表 | ❌ 无 | 防 squatting，推荐实现 | P2 |

### 2.4 部署与运维

| 关键需求 | ClawHub | vercel-skills | 私有化 Gap | 优先级 |
|---------|---------|---------------|-----------|--------|
| **部署方案** | Convex 云服务（SaaS） | 纯 CLI（无部署） | 需全新建设（设计文档 10 已规划三档部署） | P0 |
| **代理/镜像层** | ❌ 无 | ❌ 无 | 需全新建设（设计文档 05） | P1 |
| **离线支持** | ❌ 无 | ❌ 无 | 需全新建设 | P1 |
| **备份恢复** | ✅ GitHub 备份/恢复 | ❌ 无 | 可参考 ClawHub GitHub 备份方案 | P1 |
| **监控告警** | ❌ 无（Convex 内置） | ❌ 无 | 需建设 Prometheus + Grafana（设计文档 10） | P1 |

---

## 3. CLI 客户端能力矩阵

### 3.1 核心命令

| 关键需求 | ClawHub CLI | vercel-skills CLI | 私有化 Gap | 优先级 |
|---------|------------|-------------------|-----------|--------|
| **install** | ✅ slug 安装 + lockfile + `.clawhub/origin.json` | ✅ 多源安装（Git/URL/well-known/npm） | ClawHub 基础 + vercel-skills 多源扩展 | P0 |
| **search** | ✅ 基础搜索 | ✅ **亮点**：fzf 风格实时交互搜索 + debounce | 借鉴 vercel-skills 交互式 UX | P0 |
| **publish** | ✅ 逐文件上传 → publish | ❌ 无（推 Git 即发布） | 需基于 ClawHub 流程，扩展签名 | P0 |
| **update** | ✅ resolve API + fingerprint 版本匹配 | ✅ 基于 folder hash 检测变更 | 优先兼容 ClawHub resolve 机制 | P0 |
| **uninstall** | ✅ 基础 | ✅ 基础 | 两者均可 | P1 |
| **list** | ✅ 列出已安装 | ✅ 列出已安装 | 两者均可 | P1 |
| **inspect** | ✅ 查看 skill 详情 | ❌ 无 | 参考 ClawHub | P2 |
| **rollback** | ✅ 通过 tag 管理实现 | ❌ 无 | 需新建 | P1 |
| **yank** | ❌ 无 | ❌ 无 | 需全新建设（设计文档 07） | P2 |
| **verify** | ❌ 无签名验证命令 | ❌ 无 | 需全新建设（设计文档 04） | P1 |
| **audit** | ❌ 无 | ❌ 无 | 需全新建设（设计文档 08） | P2 |
| **star/unstar** | ✅ 存在 | ❌ 无 | 社区可选 | P3 |
| **hide/unhide** | ✅ 管理员命令 | ❌ 无 | 管理侧需要 | P2 |
| **ban/set-role** | ✅ 管理员命令 | ❌ 无 | 管理侧需要 | P2 |

### 3.2 安装架构与 Agent 分发

| 关键需求 | ClawHub CLI | vercel-skills CLI | 私有化 Gap | 优先级 |
|---------|------------|-------------------|-----------|--------|
| **多 Agent 支持** | ⚠️ 仅 OpenClaw 相关路径 | ✅ **亮点**：41 个 AI Agent 目录自动检测 | 急需扩展多 Agent 分发 | P1 |
| **Universal 架构** | ❌ 无 | ✅ 共享 `.agents/skills/` + symlink | 推荐借鉴 vercel-skills 架构 | P1 |
| **Symlink 安装** | ❌ 无 | ✅ 避免文件重复 | 推荐借鉴 | P1 |
| **源格式解析** | ⚠️ 仅 Registry URL | ✅ **亮点**：7 种源（shorthand/URL/Git/local/well-known/npm） | 扩展多源支持作为 Registry 备选 | P2 |
| **`@skill` 语法** | ❌ 无 | ✅ `owner/repo@skill-name` 选择安装 | 可借鉴 | P2 |
| **Well-Known 协议** | ❌ 无 | ✅ `/.well-known/skills/index.json` (RFC 8615) | 用于联邦发现 | P3 |
| **npm 同步** | ❌ 无 | ✅ 从 npm 包提取 SKILL.md | 可选生态集成 | P3 |

### 3.3 配置管理

| 关键需求 | ClawHub CLI | vercel-skills CLI | 私有化 Gap | 优先级 |
|---------|------------|-------------------|-----------|--------|
| **多 Registry 配置** | ⚠️ 支持 discovery 机制 + legacy 兼容 | ❌ 硬编码 `skills.sh` | 需实现 `.skillrc.yaml` 多 Registry 配置 | P0 |
| **Lock File** | ⚠️ 单层（version + installedAt） | ✅ 两层（Global v3 + Local v1） | 融合设计：Local（项目 VCS 提交）+ Global（用户偏好） | P0 |
| **CI 环境检测** | ❌ 无 | ✅ 检测 7 种 CI 环境 | 推荐借鉴 | P1 |
| **Private Repo 检测** | ❌ 无 | ✅ GitHub API 可见性警告 | 推荐借鉴 | P2 |
| **Telemetry** | ✅ `telemetry-sync`（安装上报） | ✅ fire-and-forget 事件 | 可选实现 | P2 |

### 3.4 安全能力

| 关键需求 | ClawHub CLI | vercel-skills CLI | 私有化 Gap | 优先级 |
|---------|------------|-------------------|-----------|--------|
| **安装时签名验证** | ❌ 无 | ❌ 无 | 需全新建设（cosign verify-blob） | P1 |
| **安全审计展示** | ❌ 无 CLI 侧展示 | ✅ **亮点**：安装时展示 Gen/Socket/Snyk 报告 | 借鉴——对接内部扫描结果展示 | P1 |
| **Hash 校验** | ⚠️ fingerprint 机制（resolve API） | ⚠️ folder hash 对比 | 标准化 SHA-256 file-level hash 校验 | P0 |

---

## 4. Skill Store（商店）能力矩阵

### 4.1 搜索与发现

| 关键需求 | ClawHub | vercel-skills | 私有化 Gap | 优先级 |
|---------|---------|---------------|-----------|--------|
| **全文搜索** | ✅ lexical fallback 搜索 | ⚠️ 依赖外部 API | 需建设 PostgreSQL FTS（设计文档 06 Phase 1） | P0 |
| **向量搜索** | ✅ Convex vectorSearch（embedding 维度知） | ❌ 无 | 需建设 pgvector（设计文档 06 Phase 2） | P1 |
| **混合搜索** | ✅ vectorScore + lexicalBoost + popularityBoost 融合 | ❌ 无 | 需实现 RRF 融合排序（设计文档 06 Phase 3） | P2 |
| **排序算法** | ✅ slug exact(+1.4) + prefix(+0.8) + name match + log(downloads) | ❌ 无自建排序 | 可参考 ClawHub 排序权重 | P1 |
| **命名空间/可见性过滤** | ❌ 无（所有公开） | ❌ 无 | 需全新建设（私有化核心） | P0 |
| **交互式搜索 UI** | ❌ 无 | ✅ **fzf 风格实时搜索** | 借鉴 vercel-skills CLI 搜索 UX | P1 |
| **`highlightedOnly` 过滤** | ✅ 搜索参数 | ❌ 无 | 可选实现 | P3 |

### 4.2 安全与信任展示

| 关键需求 | ClawHub | vercel-skills | 私有化 Gap | 优先级 |
|---------|---------|---------------|-----------|--------|
| **质量评分展示** | ✅ quality.score + trustTier + signals | ❌ 无 | 展示 ClawHub 风格评分 | P1 |
| **VT 扫描结果** | ✅ vtAnalysis（verdict/analysis） | ❌ 无 | 展示扫描状态 + 详情链接 | P0 |
| **LLM 分析结果** | ✅ llmAnalysis（verdict/confidence/summary/dimensions） | ❌ 无 | 展示安全评估摘要 | P1 |
| **第三方安全报告** | ❌ 无 | ✅ Gen/Socket/Snyk 报告展示 | 可集成第三方报告 + 内部扫描 | P1 |
| **签名状态** | ❌ 无 | ❌ 无 | 需新建签名验证状态展示 | P1 |
| **SBOM 查看** | ❌ 无 | ❌ 无 | 需新建 | P2 |
| **Badges（徽章）** | ✅ highlighted/official/deprecated/redactionApproved | ❌ 无 | 可复用徽章体系 | P2 |

### 4.3 社区与交互

| 关键需求 | ClawHub | vercel-skills | 私有化 Gap | 优先级 |
|---------|---------|---------------|-----------|--------|
| **Stars/收藏** | ✅ 完整 API + 统计 | ❌ 无 | 社区可选 | P3 |
| **评论系统** | ✅ comments 表 + CRUD | ❌ 无 | 社区可选 | P3 |
| **排行榜** | ✅ leaderboards 表 + trending 排序 | ❌ 无 | 社区可选 | P3 |
| **每日统计** | ✅ dailyStats 表 + 聚合 | ❌ 无 | 影响排序准确性 | P2 |
| **下载去重** | ✅ dedupes 表（identity + 小时窗口） | ❌ 无 | 影响统计准确性 | P2 |
| **Fork/Duplicate** | ✅ forkOf 关系 | ❌ 无 | 可选 | P3 |
| **举报机制** | ✅ skillReports 表 + 阈值自动隐藏 | ❌ 无 | 安全治理需要 | P2 |

### 4.4 企业级能力（两个开源方案均不覆盖）

| 关键需求 | ClawHub | vercel-skills | 私有化 Gap | 优先级 |
|---------|---------|---------------|-----------|--------|
| **RBAC（组织/团队/技能）** | ❌ 仅 admin/moderator/user 三级角色 | ❌ 无 | 需全新建设（设计文档 08） | P0 |
| **多租户隔离** | ❌ 全局公开 | ❌ 全局公开 | 需全新建设 namespace + visibility | P0 |
| **代理/镜像层** | ❌ 无 | ❌ 无 | 需全新建设（设计文档 05） | P1 |
| **离线/断网支持** | ❌ 无 | ❌ 无 | 需全新建设 | P1 |
| **审计合规** | ⚠️ 简化 auditLogs（无 IP/UA） | ❌ 仅 telemetry | 需全新建设 append-only 审计 | P1 |
| **cosign 签名体系** | ❌ 无 | ❌ 无 | 需全新建设（设计文档 04） | P1 |
| **策略引擎** | ❌ 无 | ❌ 无 | 需全新建设（设计文档 08 策略引擎） | P1 |
| **Break-glass Token** | ❌ 无 | ❌ 无 | 需全新建设 | P2 |

---

## 5. 实现方案

基于以上能力矩阵分析，提出三套实现方案：

### 5.1 方案 A：ClawHub-First（最大兼容方案）

#### 核心思路

以 ClawHub V1 API 为基座，兼容其 CLI 交互协议，在此基础上逐层叠加私有化扩展能力。

#### 架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                     方案 A: ClawHub-First                        │
│                                                                  │
│  ┌─────────────┐    ┌─────────────────────────────────────────┐  │
│  │ ClawHub CLI  │    │        私有 Registry 服务端              │  │
│  │ (兼容模式)   │───►│                                         │  │
│  │              │    │  ┌──────────┐  ┌──────────────────────┐ │  │
│  │ skill CLI    │───►│  │ V1 API   │  │ 私有扩展 API          │ │  │
│  │ (新建/扩展)  │    │  │ 兼容层   │  │ RBAC/签名/审计/代理   │ │  │
│  └─────────────┘    │  └────┬─────┘  └──────────┬───────────┘ │  │
│                      │       │                    │             │  │
│                      │  ┌────▼────────────────────▼──────────┐ │  │
│                      │  │       业务逻辑层                    │ │  │
│                      │  │  发布流水线 │ 质量门控 │ 安全扫描    │ │  │
│                      │  └────────────┬───────────────────────┘ │  │
│                      │               │                         │  │
│                      │  ┌────────────▼───────────────────────┐ │  │
│                      │  │        数据层                       │ │  │
│                      │  │  PostgreSQL(+pgvector) │ S3/MinIO   │ │  │
│                      │  │  Redis │ Audit Log Store            │ │  │
│                      │  └────────────────────────────────────┘ │  │
│                      └─────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

#### Registry 层实现

| 模块 | 实现策略 | 工作量 |
|------|---------|--------|
| **V1 API 兼容层** | 1:1 实现 ClawHub V1 路由（见 §3.2 完整路由表），修正设计文档中的错误假设：<br>- 上传协议改为逐文件上传 + publish（非 presigned URL 列表）<br>- 错误响应暂保持纯文本（兼容），扩展端点用 JSON<br>- 认证统一 Bearer Token | 4-6 周 |
| **Resolve API** | 新建 `GET /resolve?slug=X&hash=Y`，实现 fingerprint 匹配 | 1 周 |
| **质量门控** | 复用 ClawHub 质量评分机制（signals → score → decision），适配 PostgreSQL | 2 周 |
| **Moderation 状态机** | 采纳 ClawHub 多字段模型（moderationStatus + moderationReason），扩展为显式版本状态 | 1 周 |
| **VT + LLM 扫描** | 复用 ClawHub 扫描逻辑，改为独立表存储（非内嵌） | 2 周 |
| **RBAC** | 全新建设（设计文档 08），ClawHub 仅提供 admin/moderator/user 参考 | 4-6 周 |
| **cosign 签名** | 全新建设（设计文档 04），签名目标改为 fingerprint（非 zip） | 3-4 周 |
| **代理/镜像** | 全新建设（设计文档 05），注意动态 zip 的缓存策略 | 3-4 周 |
| **搜索引擎** | Phase 1: PostgreSQL FTS → Phase 2: pgvector → Phase 3: RRF 融合，参考 ClawHub 排序权重 | 4-6 周 |

#### CLI 层实现

| 模块 | 实现策略 | 工作量 |
|------|---------|--------|
| **基础命令集** | 兼容 ClawHub CLI 全部命令（install/publish/search/update/list/inspect/star/hide） | 3-4 周 |
| **签名命令** | 新增 `skill verify`、`skill publish --sign` | 1-2 周 |
| **多 Registry 配置** | 实现 `.skillrc.yaml`，支持 `--registry` 切换 | 1 周 |
| **Lockfile 扩展** | 在 ClawHub lockfile 基础上扩展 contentHash + registry + riskLevel 字段 | 1 周 |

#### Skill Store 层实现

| 模块 | 实现策略 | 工作量 |
|------|---------|--------|
| **搜索 UI** | 基础 Web UI，复用 ClawHub 排序算法 | 2-3 周 |
| **安全展示** | 展示 VT/LLM 扫描结果 + 签名状态 | 1-2 周 |
| **管理控制台** | RBAC 管理 + Quarantine 审批 + 审计日志 | 3-4 周 |

#### 优势与风险

| 优势 | 风险 |
|------|------|
| ✅ 最大程度兼容 ClawHub 生态，用户迁移成本低 | ⚠️ 受限于 ClawHub API 设计（Convex 特有模式难以 1:1 映射） |
| ✅ 可复用 ClawHub 的质量评分/扫描/搜索逻辑 | ⚠️ 上传协议差异需要兼容层（逐文件上传 vs 两阶段打包） |
| ✅ CLI 用户可无感切换 Registry | ⚠️ 错误响应格式不统一（纯文本 vs JSON） |
| ✅ 已验证的数据模型可减少设计风险 | ⚠️ 动态 zip 生成对缓存层有挑战 |

#### 总工期估算：**20-28 周（5-7 个月）**

---

### 5.2 方案 B：Hybrid-Forge（混合创新方案）

#### 核心思路

**Registry 侧**以 ClawHub 数据模型为参考但重新设计 API，**CLI 侧**融合 vercel-skills 的多 Agent 分发/交互式搜索/多源支持，**Store 侧**结合两者最佳实践。

#### 架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                    方案 B: Hybrid-Forge                               │
│                                                                      │
│  ┌──────────────────────────────────────────┐                        │
│  │          Hybrid CLI                       │                        │
│  │  ┌──────────────┐  ┌──────────────────┐  │                        │
│  │  │ ClawHub 兼容层 │  │ vercel-skills   │  │                        │
│  │  │ (Registry通信) │  │ (Agent分发/UX)   │  │                        │
│  │  └──────┬───────┘  └────────┬─────────┘  │                        │
│  │         │  统一命令接口       │            │                        │
│  │         └────────┬──────────┘             │                        │
│  └──────────────────┼────────────────────────┘                        │
│                     │                                                │
│          ┌──────────▼───────────────────────────────────────┐        │
│          │           私有 Registry 服务端                      │        │
│          │                                                   │        │
│          │  ┌─────────────┐  ┌────────────┐  ┌────────────┐ │        │
│          │  │ RESTful API  │  │ 上游代理层  │  │ Well-Known │ │        │
│          │  │ (新设计,      │  │ ClawHub    │  │ Protocol   │ │        │
│          │  │  参考ClawHub) │  │ + Git 源   │  │ 端点       │ │        │
│          │  └──────┬──────┘  └─────┬──────┘  └─────┬──────┘ │        │
│          │         │               │               │         │        │
│          │  ┌──────▼───────────────▼───────────────▼──────┐  │        │
│          │  │              业务逻辑层                       │  │        │
│          │  │  发布+签名 │ 质量+扫描 │ RBAC │ 审计 │ 代理   │  │        │
│          │  └──────────────────┬──────────────────────────┘  │        │
│          │                    │                              │        │
│          │  ┌─────────────────▼─────────────────────────────┐│        │
│          │  │  PostgreSQL(+pgvector) │ S3/MinIO │ Redis     ││        │
│          │  └───────────────────────────────────────────────┘│        │
│          └───────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────────────┘
```

#### Registry 层实现

| 模块 | 实现策略 | 与方案 A 差异 |
|------|---------|-------------|
| **API 设计** | 全新 RESTful API（参考但不绑定 ClawHub V1），统一 JSON 错误响应，支持 cursor 分页 | 不受 ClawHub API 奇异设计约束 |
| **上传协议** | 新设计两阶段上传：`POST /uploads/init` → presigned URL → `POST /uploads/commit`，同时兼容 ClawHub 逐文件模式 | 更标准的协议设计 |
| **数据模型** | 参考 ClawHub 丰富模型（quality, badges, fingerprint），但规范化为 PostgreSQL 关系模型（VT/LLM 为独立表，Tags 为独立表） | 比 ClawHub 的 Convex 嵌入模型更适合关系查询 |
| **上游代理** | 支持**双上游**：ClawHub Registry + Git 直连源（参考 vercel-skills 源解析） | 多源聚合能力 |
| **Well-Known 协议** | 实现 `/.well-known/skills/index.json` 端点 | 联邦发现能力（vercel-skills 借鉴） |
| **签名体系** | cosign（设计文档 04），签名目标为 fingerprint（适配 ClawHub 实际机制） | 同方案 A |
| **认证授权** | OAuth2 + API Key + Token scope + 多租户 RBAC | 同方案 A |

#### CLI 层实现

| 模块 | 实现策略 | 与方案 A 差异 |
|------|---------|-------------|
| **命令集** | 合并 ClawHub 命令（15+ 命令含管理命令）+ vercel-skills 特色命令 | 更丰富 |
| **交互式搜索** | 借鉴 vercel-skills fzf 风格搜索 UX（实时 debounce + 方向键 + 直接安装） | UX 提升 |
| **多 Agent 分发** | 借鉴 vercel-skills Universal 架构（`.agents/skills/` + symlink） | 支持 41+ Agent |
| **多源安装** | 主源为私有 Registry，备选源：ClawHub（代理）、Git URL、local path、well-known | 7 种源格式 |
| **两层 Lock File** | Global Lock（用户级，追踪更新偏好）+ Local Lock（项目级，提交 VCS），兼容 ClawHub lock 字段 | 比方案 A 更完善 |
| **CI 检测** | 借鉴 vercel-skills CI 环境检测（7 种 CI） | 方案 A 无此能力 |
| **安全展示** | 安装时自动展示 VT/LLM 扫描结果 + 签名状态 + 第三方审计报告 | 融合 vercel-skills 安全 UX |

#### Skill Store 层实现

| 模块 | 实现策略 | 与方案 A 差异 |
|------|---------|-------------|
| **搜索** | 三阶段搜索（FTS → vector → hybrid），前端套用 ClawHub 排序权重 + vercel-skills 交互 UX | 端到端最佳搜索体验 |
| **安全信任面板** | 质量评分 + VT 报告 + LLM 分析 + 签名状态 + 第三方报告，统一安全信任评分 | 最全面的安全展示 |
| **管理控制台** | RBAC + Quarantine 审批 + 审计日志 + Moderation 管理 | 同方案 A |
| **社区功能** | 可选：Stars + 评论 + 排行榜 + Badges | 同方案 A |
| **Well-Known 发现** | 支持网站通过 well-known 协议暴露可用 skills 列表 | vercel-skills 借鉴 |

#### 优势与风险

| 优势 | 风险 |
|------|------|
| ✅ 融合两个开源方案的最佳实践 | ⚠️ 开发工作量最大 |
| ✅ API 设计不受 ClawHub Convex 特异性约束 | ⚠️ 无法直接复用 ClawHub CLI（需适配层） |
| ✅ CLI UX 最佳（交互搜索 + 多 Agent + 多源） | ⚠️ 需维护 ClawHub 兼容 + 新 API 双协议 |
| ✅ Well-Known + 多源 = 最灵活的生态互操作 | ⚠️ 测试矩阵复杂度高 |
| ✅ 两层 Lock File + 签名 = 最安全的客户端 | |

#### 总工期估算：**28-36 周（7-9 个月）**

---

### 5.3 方案 C：Clean-Slate（全新自建方案）

#### 核心思路

完全按照设计文档（01-12）构建，仅将 ClawHub 和 vercel-skills 作为**设计参考**（不兼容其 API 或 CLI），追求最优内部架构。

#### 架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                     方案 C: Clean-Slate                               │
│                                                                      │
│  ┌──────────────────────┐          ┌──────────────────────────────┐  │
│  │   全新 skill CLI      │          │       Skill Store Web UI      │  │
│  │  (不兼容 ClawHub CLI) │          │  (搜索+安全面板+管理控制台)    │  │
│  └──────────┬───────────┘          └──────────────┬───────────────┘  │
│             │                                      │                  │
│  ┌──────────▼──────────────────────────────────────▼───────────────┐  │
│  │                     私有 Registry 服务端                         │  │
│  │                                                                 │  │
│  │  ┌─────────────────────────────────────────────────────────┐    │  │
│  │  │                   全新 REST API                          │    │  │
│  │  │  /skills  /versions  /search  /auth  /audit  /orgs ...  │    │  │
│  │  └──────────────────────┬──────────────────────────────────┘    │  │
│  │                         │                                       │  │
│  │  ┌──────────────────────▼──────────────────────────────────┐    │  │
│  │  │                    业务逻辑层                              │    │  │
│  │  │  两阶段上传 │ cosign签名验证 │ SBOM生成 │ 质量门控       │    │  │
│  │  │  VT扫描 │ LLM审查 │ Quarantine审批 │ RBAC策略引擎     │    │  │
│  │  └──────────────────────┬──────────────────────────────────┘    │  │
│  │                         │                                       │  │
│  │  ┌─────────────────┐ ┌─▼──────────┐ ┌────────────────────────┐ │  │
│  │  │ 上游代理层       │ │ 数据层      │ │ 异步任务引擎           │ │  │
│  │  │ ClawHub 代理     │ │ PostgreSQL  │ │ Redis Stream/NATS      │ │  │
│  │  │ (仅作上游源)     │ │ + pgvector  │ │ 扫描/签名/embedding    │ │  │
│  │  │                 │ │ S3/MinIO    │ │                        │ │  │
│  │  └─────────────────┘ │ Redis       │ └────────────────────────┘ │  │
│  │                       └────────────┘                            │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

#### Registry 层实现

| 模块 | 实现策略 | 与方案 A/B 差异 |
|------|---------|---------------|
| **API 设计** | 完全按设计文档规范，统一 JSON 错误响应 `{ error: { code, message, requestId } }`，cursor 分页 | 不需要兼容层 |
| **上传协议** | 标准两阶段上传：init → presigned URLs → commit，客户端打包 `.skill.zip/.skill.tar.gz` + `manifest.json` | 最标准但不兼容 ClawHub |
| **数据模型** | 完全按设计文档 03 规范化关系模型 | 最干净的 schema |
| **上游代理** | ClawHub 仅作为上游代理源之一（on-demand 缓存 + quarantine 门控） | 代理方式使用 ClawHub，不兼容其 API |
| **签名** | cosign Keyless + Offline 双模（设计文档 04 完整） | 同方案 A/B |
| **RBAC** | 完整 7 角色 + 6 scope + 策略引擎（设计文档 08 完整） | 最完善的权限体系 |
| **搜索** | 三阶段（设计文档 06 完整） | 同方案 B |
| **发布流水线** | 完整流水线（设计文档 07 全部特性） | 最安全的发布流程 |

#### CLI 层实现

| 模块 | 实现策略 | 与方案 A/B 差异 |
|------|---------|---------------|
| **命令集** | 按设计文档 11 规范（§3.2 命令表完整实现） | 无 ClawHub 命令兼容负担 |
| **安装架构** | 参考 vercel-skills 多 Agent 分发（但非兼容，可选采纳） | 可选借鉴 |
| **Lock File** | 设计文档扩展版（含 contentHash, riskLevel, capabilities, sandboxAdvice） | 最丰富的安全元数据 |
| **签名验证** | 安装时默认 cosign verify-blob | 最安全 |

#### Skill Store 层实现

| 模块 | 实现策略 | 与方案 A/B 差异 |
|------|---------|---------------|
| **搜索 UI** | 私有化 Web Store 前端 | 从零建设 |
| **安全信任面板** | 完整安全展示（签名 + SBOM + VT + LLM + 质量分 + Quarantine 状态） | 最全面 |
| **管理控制台** | 完整管理后台 | 最完善 |
| **用户体系** | 组织/团队/角色 管理 UI | 最完善 |

#### 优势与风险

| 优势 | 风险 |
|------|------|
| ✅ 架构最干净，无历史包袱 | 🔴 **用户无法从 ClawHub 无缝迁移** |
| ✅ 完全遵循设计文档规范 | 🔴 开发工作量最大 |
| ✅ 安全能力最完善（签名+SBOM+RBAC+策略引擎） | ⚠️ 生态冷启动——无法直接使用 ClawHub 已有技能 |
| ✅ 无兼容层维护成本 | ⚠️ CLI 需从零教育用户 |
| ✅ 数据模型最规范 | ⚠️ 上游代理可缓解生态冷启动，但增加复杂度 |

#### 总工期估算：**24-32 周（6-8 个月）**

---

## 6. 方案对比决策矩阵

### 6.1 功能维度对比

| 评估维度 | 方案 A (ClawHub-First) | 方案 B (Hybrid-Forge) | 方案 C (Clean-Slate) |
|---------|----------------------|---------------------|--------------------|
| **ClawHub CLI 兼容** | ✅ 完全兼容 | ⚠️ 兼容层适配 | ❌ 不兼容 |
| **ClawHub 数据迁移** | ✅ 低成本 | ✅ 中等成本 | ⚠️ 高成本（需转换层） |
| **API 设计质量** | 🟡 受 ClawHub 约束 | ✅ 较优 | ✅ 最优 |
| **CLI UX** | 🟡 基础 | ✅ 最优（fzf+多Agent+多源） | 🟡 自定义 |
| **安全能力** | ✅ 完善 | ✅ 最全面 | ✅ 最全面 |
| **多 Agent 支持** | ❌ 仅 OpenClaw | ✅ 41+ Agent | ⚠️ 可选借鉴 |
| **生态互操作** | ✅ ClawHub 生态 | ✅ ClawHub + Git + Well-Known | 🟡 仅代理层 |
| **搜索能力** | ✅ 完善 | ✅ 最佳（三阶段+交互UI） | ✅ 完善 |
| **企业级 RBAC** | ✅ 完善 | ✅ 完善 | ✅ 最完善 |
| **架构整洁度** | 🟡 兼容层增加复杂度 | 🟡 双协议维护 | ✅ 最干净 |

### 6.2 非功能维度对比

| 评估维度 | 方案 A | 方案 B | 方案 C |
|---------|--------|--------|--------|
| **开发工期** | 20-28 周 🟢 | 28-36 周 🔴 | 24-32 周 🟡 |
| **技术风险** | 🟡 中（兼容适配风险） | 🟡 中（集成复杂度） | 🟢 低（无兼容约束） |
| **迁移成本** | 🟢 低 | 🟡 中 | 🔴 高 |
| **维护成本** | 🟡 中（兼容层持续维护） | 🔴 高（双协议） | 🟢 低（干净架构） |
| **生态冷启动** | 🟢 低（直接复用 ClawHub 生态） | 🟢 低（多源支持） | 🟡 中（仅代理缓解） |
| **人员要求** | 3-4 人全栈 | 4-6 人全栈 | 3-5 人全栈 |

### 6.3 适用场景推荐

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| **已有大量 ClawHub 技能，需快速私有化** | 方案 A | 最低迁移成本，最快见效 |
| **长期平台建设，需最佳 CLI 体验和生态互操作** | 方案 B | 融合两家最佳实践，长期价值最高 |
| **全新企业环境，无历史包袱，安全合规优先** | 方案 C | 最干净架构，最完善安全能力 |
| **小团队快速验证，后续逐步扩展** | 方案 A → 方案 B | 先兼容上线，再逐步引入 vercel-skills 能力 |

---

## 7. 推荐路径与里程碑

### 7.1 推荐选择：方案 A → 方案 B 渐进路径

综合考虑开发效率、生态兼容、长期演进，推荐 **方案 A 为起点，渐进演进到方案 B**：

```
Phase 1 (M1-M3)          Phase 2 (M4-M6)         Phase 3 (M7-M9)
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  方案 A 核心      │    │  方案 A 完整      │    │  方案 B 融合      │
│                  │    │                  │    │                  │
│ • V1 API 兼容层   │───►│ • RBAC 完整实现   │───►│ • 多 Agent 分发    │
│ • 基础 CLI       │    │ • cosign 签名     │    │ • 交互式搜索 UX   │
│ • 质量门控+扫描   │    │ • SBOM 生成      │    │ • Well-Known 协议  │
│ • 基础搜索(FTS)  │    │ • 代理/镜像层     │    │ • 多源安装支持     │
│ • 认证(Bearer)   │    │ • 向量搜索       │    │ • 两层 Lock File   │
│ • 基础 Lockfile  │    │ • 审计日志完善    │    │ • CI 环境检测     │
│ • 部署(单机)     │    │ • 部署(标准生产)   │    │ • 部署(高安全)    │
└──────────────────┘    └──────────────────┘    └──────────────────┘
      MVP 交付                安全合规达标              最佳体验达标
```

### 7.2 详细里程碑

#### Phase 1：MVP（Month 1-3）

| 里程碑 | 交付物 | 验收标准 | 依赖 |
|--------|--------|---------|------|
| M1.1 数据模型 + API 框架 | PostgreSQL schema + V1 API skeleton | 空行程可运行 | — |
| M1.2 V1 API 兼容层 | Skill CRUD + Download + Search + Resolve | ClawHub CLI 可切换 Registry 正常运作 | M1.1 |
| M1.3 基础认证 | Bearer Token + 用户 CRUD | login + whoami 可用 | M1.1 |
| M1.4 质量门控 + VT 扫描 | 发布流水线（质量评分 + VT 异步扫描 + moderation 状态机） | publish → 质量检测 → VT 扫描 → 上线/隐藏 | M1.2 |
| M1.5 基础 CLI | install / publish / search / update / list | 端到端 skill 生命周期管理 | M1.2, M1.3 |
| M1.6 单机部署 | Docker Compose 一键部署 | 10 分钟内部署可用 | M1.1-M1.5 |

#### Phase 2：安全合规（Month 4-6）

| 里程碑 | 交付物 | 验收标准 | 依赖 |
|--------|--------|---------|------|
| M2.1 RBAC 系统 | 组织/团队/角色/Token scope 完整实现 | 多团队隔离、最小权限可验证 | Phase 1 |
| M2.2 cosign 签名 | 签名 + 验签 + 组织签名策略 | `publish --sign` + `verify` + 安装自动验签 | Phase 1 |
| M2.3 SBOM 生成 | CycloneDX 1.5 自动生成 + 查看端点 | 每个版本可查看 SBOM | M2.2 |
| M2.4 代理/镜像层 | ClawHub 上游代理 + 缓存 + quarantine 门控 | 断网降级可用、上游技能自动代理 | Phase 1 |
| M2.5 搜索增强 | pgvector 向量搜索 + RRF 混合排序 | 语义搜索准确率 > 80% | Phase 1 |
| M2.6 审计日志 | append-only 审计日志 + IP/UA + 查询端点 | 合规审计可追溯 | Phase 1 |
| M2.7 LLM 安全审查 | LLM 安全评估集成（异步 advisory） | 发布后安全报告可查看 | Phase 1 |
| M2.8 标准生产部署 | HA 部署 + 备份恢复 + 监控告警 | RPO 5min / RTO 30min | Phase 1 |

#### Phase 3：最佳体验（Month 7-9）

| 里程碑 | 交付物 | 验收标准 | 依赖 |
|--------|--------|---------|------|
| M3.1 多 Agent 分发 | Universal `.agents/skills/` + symlink 架构 | 一个 skill 自动分发到 N 个 agent 目录 | Phase 2 |
| M3.2 交互式搜索 | fzf 风格 CLI 搜索（debounce + 方向键 + 直接安装） | 用户体验评估通过 | Phase 2 |
| M3.3 多源安装 | Registry + Git URL + local path + well-known | 7 种安装源均可用 | Phase 2 |
| M3.4 两层 Lock File | Global Lock + Local Lock + riskLevel/sandboxAdvice 扩展 | 团队共享 lockfile 可提交 VCS | Phase 2 |
| M3.5 Well-Known 协议 | `/.well-known/skills/index.json` 端点 | 外部 vercel-skills 用户可发现私有 skills | Phase 2 |
| M3.6 Skill Store Web | 搜索 + 安全面板 + 管理控制台 + 社区功能 | Web UI 发布可用 | Phase 2 |
| M3.7 CI 环境检测 | 检测 CI 环境 + 自动非交互模式 | 7 种 CI 环境自动适配 | Phase 2 |
| M3.8 高安全部署 | 断网离线 + HSM 签名 + 全链路审计 | 满足合规检查清单 | Phase 2 |

---

## 附录 A：关键修正清单

基于两份差异分析，以下是在实施任意方案前必须修正的设计文档内容：

| # | 修正项 | 来源 | 影响模块 |
|---|--------|------|---------|
| 1 | 上传协议改为逐文件上传 + publish（非 presigned URL 列表） | ClawHub gap | 02, 04, 07 |
| 2 | 默认 Registry URL 为 `clawhub.ai`（非 `.com`） | ClawHub gap | 01, 02, 11 |
| 3 | 错误响应为纯文本（非 JSON），兼容层需处理两种格式 | ClawHub gap | 02, 11 |
| 4 | 认证统一 Bearer Token（非 `ApiKey` 头） | ClawHub gap | 01, 02, 08 |
| 5 | Tag 管理无 HTTP 端点，需新建非"1:1 兼容" | ClawHub gap | 01, 02 |
| 6 | 搜索无 cursor 分页，分页为新增能力 | ClawHub gap | 01, 02, 06 |
| 7 | Lockfile 无 contentHash/tag/registry 字段 | ClawHub gap | 03, 11 |
| 8 | VT/LLM 分析为内嵌字段非独立表 | ClawHub gap | 03, 07 |
| 9 | 补充 Resolve API（`GET /resolve`） | ClawHub gap | 01, 02, 11 |
| 10 | 补充质量评分系统（quality signals + trust tier） | ClawHub gap | 07 |
| 11 | 签名目标改为 fingerprint（非 zip）——因 ClawHub 动态生成 zip | ClawHub gap | 04 |
| 12 | 补充多 Agent 分发架构评估 | vercel-skills gap | 09, 11 |
| 13 | 补充两层 Lock File 设计评估 | vercel-skills gap | 03, 11 |
| 14 | 补充 Well-Known 协议评估 | vercel-skills gap | 05, 06 |
| 15 | 用户角色修正为 admin/moderator/user 三级（非二级） | ClawHub gap | 01, 08 |

---

## 附录 B：两个开源方案不可替代能力汇总

以下能力在 ClawHub 和 vercel-skills 中**均不存在**，是私有化 Registry 的核心差异化价值，无论选择哪个方案均需全新建设：

| 能力 | 企业价值 | 设计文档 |
|------|---------|---------|
| cosign 包签名 + 验签 | 供应链安全底线 | 04 |
| 完整 RBAC（Org/Team/Skill/Token scope） | 企业多团队协作 | 08 |
| 上游代理 + quarantine 门控 + 离线支持 | 合规 + 网络隔离 | 05 |
| SBOM (CycloneDX) 自动生成 | 软件供应链透明度 | 04 |
| 策略引擎（风险分级 + Sandbox 联动） | 运行时安全 | 08, 09 |
| 完整审计日志（append-only + IP/UA + 查询） | 合规审计 | 08 |
| Break-glass Token | 紧急访问控制 | 08 |
| 命名空间 + 可见性（public/org/private） | 多租户隔离 | 03 |
