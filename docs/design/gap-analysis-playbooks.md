# Playbooks.com 对比调研 — 差异分析文档

> **文档类型**：竞品差异分析  
> **日期**：2026-02-25  
> **对比基线**：  
> - 我方设计文档：`docs/design/01~12` 系列设计文档 + 3 篇差异分析  
> - 竞品：[Playbooks](https://playbooks.com/)（playbooks.com）  
> **调研方法**：访问 Playbooks 官网、/skills、/mcp、/tools、/advertise、/login、/privacy、/terms、技能详情页、Skill Security Scanner 页面  
> **信息限制**：Playbooks 无公开 API 文档站（/docs 返回 404）、无公开 Changelog、无公开 GitHub 源码仓库。以下分析基于可公开访问的页面内容。

---

## 目录

1. [Playbooks 产品概述](#1-playbooks-产品概述)
2. [功能对比总览表](#2-功能对比总览表)
3. [按功能域详细对比（A–L）](#3-按功能域详细对比al)
4. [Playbooks 独有能力清单](#4-playbooks-独有能力清单)
5. [架构差异分析](#5-架构差异分析)
6. [结论与建议](#6-结论与建议)

---

## 1. Playbooks 产品概述

### 1.1 产品定位

Playbooks 是一个**免费、策展型的 AI Agent 技能与 MCP 服务器目录平台**，由 [Ian Nuttall](https://x.com/iannuttall) 创建，面向开发者提供可复用的 Agent 指令（Skills）和 MCP 服务器配置的发现与安装服务。

| 维度 | 说明 |
|------|------|
| **产品形态** | Web 目录 + 轻量 CLI（`npx playbooks`） |
| **商业模式** | 免费使用，通过广告变现（$400–$1,000/月） |
| **内容来源** | GitHub 仓库为数据源，用户通过 GitHub OAuth 提交 |
| **认证方式** | GitHub OAuth（`read:user` + `user:email` scope） |
| **法人实体** | 英国注册（依据 Terms 的 Governing Law 条款） |

### 1.2 核心数据规模（截至 2026-02-25）

| 指标 | 数值 |
|------|------|
| Skills 总量 | ~787 页（每页约 30 个，估计 **20,000+** 条） |
| MCP Servers 总量 | ~414 页（估计 **12,000+** 条） |
| 月独立访客 | 63,000 |
| 月页面浏览量 | 170,000 |
| 每访客页面数 | 2.7 |

### 1.3 支持的 Agent 工具

Claude Code、Cursor、Cline、Windsurf、Zed、Amp、Codex CLI、Roo Code、VS Code、Gemini CLI 以及任何 MCP 客户端。

---

## 2. 功能对比总览表

| 功能域 | 我方设计 | Playbooks 对应能力 | 对齐度 | 差异说明 |
|--------|---------|-------------------|--------|---------|
| **A. Registry API** | 三层路由、ClawHub v1 兼容、cursor 分页、版本协商 | 无公开 REST API，仅 Web 页面 + `npx playbooks` CLI | 🔴 缺失 | Playbooks 无 Registry API 概念，非包管理平台 |
| **B. 数据模型** | 12+ PostgreSQL 实体、状态机、JSONB、pgvector | Skills/MCP 列表 + GitHub 仓库元数据 | 🔴 缺失 | 无版本管理、无组织/团队模型、无审计日志 |
| **C. 包签名** | cosign/Sigstore、manifest.json、SBOM | 无 | 🔴 缺失 | 无任何签名或完整性验证机制 |
| **D. RBAC** | 4 层资源、7 角色、25+ 权限、策略引擎 | GitHub OAuth 登录，无角色区分 | 🔴 缺失 | 仅有匿名浏览 + GitHub 登录两种状态 |
| **E. 发布扫描** | 三并行扫描器、聚合决策树、Quarantine | Skill Security Scanner（6 维度静态分析） | 🟡 部分对齐 | Playbooks 有安全扫描但不阻止安装，仅作参考报告 |
| **F. 代理/镜像** | 上游缓存、预拉取、断路器、离线降级 | 无 | 🔴 缺失 | Playbooks 为公共目录，不存在代理/缓存层 |
| **G. 搜索** | 三阶段搜索（FTS→向量→混合+RRF） | Web 搜索 + 排序过滤（Trending/语言/Official） | 🟡 部分对齐 | Playbooks 有搜索和排序但实现细节未公开 |
| **H. 发布流水线** | 两阶段上传、6 阶段流水线、审批门控 | GitHub 提交 + 人工审核 | 🟡 部分对齐 | Playbooks 通过 Web 提交，有人工 review，无自动化流水线 |
| **I. OpenClaw 联动** | Gating 元数据、风险分级、Sandbox 触发 | 展示 `installed on` 工具列表，无执行层联动 | 🔴 缺失 | Playbooks 为目录服务，不参与执行层 |
| **J. 部署架构** | 三档部署（单机/生产/断网）、14+ SLI | SaaS 托管（无自部署选项） | 🔴 缺失 | 完全不同的产品形态 |
| **K. CLI** | 16+ 命令、Lockfile v2、多 Registry | `npx playbooks add/find`（轻量 CLI） | 🟡 部分对齐 | CLI 仅支持安装/搜索，无版本管理/签名/审计 |
| **L. 迁移路线** | 6 步迁移、5 阶段路线图 | N/A | 🔴 缺失 | 不适用——Playbooks 非迁移目标 |

---

## 3. 按功能域详细对比（A–L）

### A. Registry API 层

#### Playbooks 实际能力

Playbooks **没有公开的 REST API 文档**（`/docs` 返回 404）。面向用户的交互方式为：
- **Web 浏览**：`/skills`、`/mcp`、`/tools` 页面
- **CLI**：`npx playbooks add skill <owner/repo> --skill <name>` 安装技能；`npx playbooks find skill` 搜索技能
- **GitHub Raw**：技能实际文件托管在 GitHub 仓库中，通过 GitHub API 获取

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| ClawHub v1 兼容 API | 无 REST API，无兼容性概念 | 🔴 缺失 |
| 三层路由（兼容/扩展/内部） | 无 API 分层 | 🔴 缺失 |
| API 版本协商 | 无 API 版本控制 | 🔴 缺失 |
| `/health` 健康检查 | ⚠️ 未公开 | 🔴 缺失 |
| cursor-based 分页 | Web 端有分页（Page 1 of 787），实现方式未知 | 🟡 部分对齐 |
| `search/skills/download/publish/whoami` | 仅有 Web 搜索 + CLI 安装，无标准 API | 🔴 缺失 |
| `tags/delete/undelete/stars/resolve` | 有 Tags/Topics 展示，无用户操作 API | 🔴 缺失 |

#### 差异影响评估

Playbooks 本质上是一个**内容发现平台**而非 **Package Registry**。它不提供标准化的 REST API 供第三方集成，所有操作依赖 Web UI 或 `npx playbooks` CLI。这与我方设计的完整 Registry API 定位根本不同。

---

### B. 核心数据模型

#### Playbooks 实际能力

从技能详情页（如 `anthropics/skills/frontend-design`）观察到的数据字段：

| 数据类别 | 可见字段 |
|---------|---------|
| **技能基本信息** | name, description, overview, how it works, when to use, best practices, example use cases, FAQ |
| **源码信息** | 关联 GitHub 仓库、文件列表（SKILL.md, LICENSE.txt 等）、文件内容预览 |
| **评分指标** | Skill Score（数值型，如 20）、Health Score（iA 100/100）、Stars 数 |
| **统计数据** | Weekly Installs、Telemetry First Seen 日期、First Seen 日期 |
| **分类标签** | Tags（如 `official`, `python`）、Topics（如 `frontend`, `design`）、Trigger Phrases |
| **安装分布** | `installed on` 各工具的安装百分比（claude-code 87%, cursor 74%...） |
| **安全状态** | 安全标记（如 `safe`） |

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| 12+ PostgreSQL 实体 | ⚠️ 未公开数据库结构，推测为简化模型 | 🔴 缺失 |
| Skill 状态机（active→hidden→quarantined→deleted） | 无可见状态机，仅有安全标记 | 🔴 缺失 |
| SkillVersion 状态机 | **无版本概念**，技能直接映射到 GitHub 仓库文件夹 | 🔴 缺失 |
| JSONB 扩展字段 | ⚠️ 未公开 | 🔴 缺失 |
| pgvector 向量索引 | ⚠️ 未公开搜索实现 | ⚠️ 未公开 |
| 软删除 + 审计日志 | ⚠️ 未公开 | 🔴 缺失 |
| Organization/Team 模型 | 按 GitHub owner/repo 组织（如 `anthropics/skills`） | 🟡 部分对齐 |

#### 差异影响评估

Playbooks 的数据模型以 **GitHub 仓库为中心**，一个 Skill 对应一个 GitHub 仓库中的文件夹路径（如 `anthropics/skills/tree/HEAD/skills/frontend-design`）。这种模型天然缺乏版本管理、状态机、组织权限等企业级特性，但在**技能发现和质量评估**方面（Skill Score、Health Score、安装遥测分布）有独特的数据维度。

---

### C. 安全 — 包签名

#### Playbooks 实际能力

**无任何包签名相关功能。** 技能直接从 GitHub 仓库获取，信任模型完全依赖 GitHub 平台本身的安全性。

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| cosign/Sigstore 签名 | 无 | 🔴 缺失 |
| manifest.json v1 格式 | 无 manifest | 🔴 缺失 |
| Fulcio 短期证书 + Rekor 日志 | 无 | 🔴 缺失 |
| 组织级签名策略 | 无 | 🔴 缺失 |
| SBOM 自动生成 | 无 | 🔴 缺失 |

#### 差异影响评估

Playbooks 不处理包的完整性和来源验证。这是我方设计的**核心企业级差异化优势**之一。Playbooks 的信任模型是"信任 GitHub 仓库所有者"，适合开放社区但不适合企业安全要求。

---

### D. 安全 — RBAC

#### Playbooks 实际能力

- **认证**：仅 GitHub OAuth（`read:user` + `user:email` scope）
- **角色**：无可见角色区分。用户登录后可提交技能，经人工审核后上线。
- **权限**：
  - 匿名用户：浏览、搜索、通过 CLI 安装
  - 登录用户：提交技能/MCP 服务器、购买广告
  - 审核者/管理员：⚠️ 未公开（推测存在内部审核角色）

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| 4 层资源层级 | 扁平结构（无层级） | 🔴 缺失 |
| 7 种角色 | 约 2 种（匿名/登录用户）+ ⚠️ 未公开的内部管理角色 | 🔴 缺失 |
| 25+ 权限矩阵 | 无权限矩阵 | 🔴 缺失 |
| 3 种身份类型（Human/Service/Agent） | 仅 Human（GitHub 用户） | 🔴 缺失 |
| 6 种 API Token scope | 无 Token 管理 | 🔴 缺失 |
| Agent 最小权限原则 | N/A | 🔴 缺失 |
| YAML 策略引擎 | 无 | 🔴 缺失 |
| Break-glass 紧急令牌 | 无 | 🔴 缺失 |

#### 差异影响评估

Playbooks 作为免费公共目录，不需要复杂的 RBAC。其权限模型极度简化——"任何人可浏览，GitHub 登录后可提交"。这与我方面向企业的多租户权限需求完全不同。

---

### E. 安全 — 发布扫描

#### Playbooks 实际能力

Playbooks 提供了一个 **Skill Security Scanner** 工具（`/tools/skill-scanner`），具备以下能力：

| 维度 | 检测内容 |
|------|---------|
| **Remote Execution** | 检测下载并执行远程代码而未验证的模式 |
| **Data Exfiltration** | 标记将本地数据发送到外部服务器或隐蔽通道的尝试 |
| **Secret Access** | 捕获读取 API Token、凭据和其他密钥存储的行为 |
| **Persistence** | 发现可能让技能在重启后存活的修改 |
| **Destructive Ops** | 识别可能导致不可逆数据丢失的命令 |
| **Obfuscation** | 发现编码载荷和其他隐藏代码实际行为的尝试 |

**工作流程**：输入 GitHub 链接 → 拉取 SKILL.md 及相关文件 → 运行规则模式分析 → 输出评分 + 裁决 + 发现列表（含文件和行引用）。

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| 三并行扫描器（静态 + VT + LLM） | 单个静态模式分析器（6 维度） | 🟡 部分对齐 |
| 扫描聚合决策树 | 输出评分 + 裁决 | 🟡 部分对齐 |
| 静态规则分类（5 类） | 6 维度分类（Remote Exec/Exfil/Secret/Persist/Destruct/Obfusc） | 🟢 对齐 |
| VirusTotal 集成 | 无 | 🔴 缺失 |
| LLM 内容审查 | 无（⚠️ 未公开是否有 LLM 辅助审查） | 🔴 缺失 |
| Quarantine + 人工审批门控 | **人工审核提交**（审核通过才上线） | 🟡 部分对齐 |
| 扫描阻止发布 | 扫描仅作参考报告，**不阻止安装** | 🔴 缺失 |

#### 差异影响评估

Playbooks 的安全扫描是一个**辅助工具**，用户可在安装前自行检查，但平台本身不将扫描结果作为发布门控。其 6 维度分类与我方静态规则的 5 分类（CRED_HARVEST/CMD_INJECT/SOCIAL_ENG/EXFIL/OBFUSCATION）高度重叠，可互相参考。Playbooks 的 "Persistence" 和 "Destructive Ops" 两个维度是我方设计中未显式列出的，**建议参考**。

---

### F. 代理/镜像层

#### Playbooks 实际能力

**无代理/镜像层功能。** Playbooks 作为公共 SaaS 目录，用户直接从 GitHub 获取技能文件。

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| 上游 ClawHub 按需缓存 | 无 | 🔴 缺失 |
| Redis 短缓存 + 制品永久缓存 | 无 | 🔴 缺失 |
| 预拉取/白名单同步 | 无 | 🔴 缺失 |
| Quarantine 门控上游版本 | 无 | 🔴 缺失 |
| 离线降级（断路器） | 无 | 🔴 缺失 |
| 离线导出/导入 | 无 | 🔴 缺失 |

#### 差异影响评估

代理/镜像层是我方设计中最核心的企业级功能之一，Playbooks 因产品定位不同完全不需要此类能力。这一功能域无对比意义。

---

### G. 搜索

#### Playbooks 实际能力

Playbooks 提供了功能较丰富的 Web 搜索体验：

| 功能 | 说明 |
|------|------|
| **文本搜索** | Skills 和 MCP Servers 的搜索框 |
| **排序** | Trending（默认）、其他排序选项 |
| **过滤** | 按语言（All languages）、Official only 筛选 |
| **分页** | 页码式分页（Page 1 of 787） |
| **分类** | Skills 和 MCP Servers 两个独立分类 |
| **CLI 搜索** | `npx playbooks find skill` 命令 |
| **质量信号** | Skill Score、Health Score、GitHub Stars、Weekly Installs |

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| Phase 1：PostgreSQL FTS（加权 A–D） | Web 搜索存在，实现未公开 | 🟡 部分对齐 |
| Phase 2：pgvector embedding | ⚠️ 未公开是否使用向量搜索 | ⚠️ 未公开 |
| Phase 3：混合搜索 + RRF 融合 | ⚠️ 搜索排名算法未公开 | ⚠️ 未公开 |
| 中文支持 | ⚠️ 未公开是否支持非英文搜索 | ⚠️ 未公开 |
| 搜索信号：下载量 + 时效衰减 | 有 Trending 排序 + Stars + Weekly Installs | 🟢 对齐 |
| cursor-based 分页 | 页码式分页（offset-based） | 🟡 部分对齐 |

#### 差异影响评估

Playbooks 的搜索在**用户体验层面**较成熟（排序、过滤、质量信号展示），但搜索引擎的技术实现未公开。值得注意的是 Playbooks 的**Trending 排序**算法和多维质量信号（Skill Score + Health Score + 安装遥测）在搜索排名中的应用，这是我方可参考的方向。

---

### H. 发布流水线

#### Playbooks 实际能力

Playbooks 的发布流程：

1. 用户在 GitHub 上创建/维护 Skill 仓库
2. 通过 Playbooks Web 界面提交（需 GitHub 登录）
3. **人工审核**（"We review submissions to keep quality high"）
4. 审核通过后上线

| 特性 | 说明 |
|------|------|
| 发布入口 | Web 界面提交（Login with GitHub） |
| 质量控制 | 人工审核 + Skill Score/Health Score 自动评分 |
| 内容来源 | GitHub 仓库（非上传包） |
| 版本管理 | **无版本**——跟踪 GitHub 仓库最新状态 |
| 归属/署名 | Creators get attribution and a link back to their GitHub |

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| 两阶段上传（init URL → 上传 → commit） | Web 提交 → 人工审核 | 🔴 缺失 |
| 6 阶段流水线 | 提交 → 审核 → 上线（约 2–3 步） | 🟡 部分对齐 |
| SBOM 自动生成 | 无 | 🔴 缺失 |
| Tag 更新即回滚 | 无版本/Tag 概念 | 🔴 缺失 |
| 发布状态轮询 API | 无 API | 🔴 缺失 |
| 版本不可变 + yank | 无版本概念 | 🔴 缺失 |
| 签名验证 | 无 | 🔴 缺失 |
| 自动安全扫描 | 有 Skill Security Scanner（但未集成到发布流程中） | 🟡 部分对齐 |

#### 差异影响评估

Playbooks 的发布模型极度简化：**GitHub 是 Source of Truth**，Playbooks 只做索引和展示。这种模型的优势是低摩擦（作者只需维护 GitHub 仓库），劣势是无法保证版本一致性和供应链安全。与我方的企业级自动化流水线设计根本不同。

---

### I. OpenClaw 执行层联动

#### Playbooks 实际能力

Playbooks 展示了技能在不同 Agent 工具上的**安装分布遥测**数据：

| 功能 | 说明 |
|------|------|
| 安装工具分布 | 显示各工具的安装百分比（claude-code 87%, cursor 74%...） |
| 安装引导 | 提供针对不同工具的安装命令/配置片段 |
| MCP 配置 | 为每个 MCP 服务器提供不同客户端的配置模板（Claude, Cursor, Cline 等） |

但不参与执行层的任何控制：

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| Gating 元数据 | 无 | 🔴 缺失 |
| 自动风险分级矩阵 | 无（仅有安全扫描报告） | 🔴 缺失 |
| Sandbox 触发配置 | 无 | 🔴 缺失 |
| Env/ApiKey 注入安全边界 | 无 | 🔴 缺失 |
| Lockfile 扩展 | 无 Lockfile | 🔴 缺失 |
| X-Risk-Level 响应头 | 无 API | 🔴 缺失 |

#### 差异影响评估

Playbooks 作为目录服务，**不参与也不控制技能的运行时行为**。其安装遥测数据（哪些工具安装了哪些技能）是有价值的生态洞察，但与运行时安全无关。

---

### J. 部署架构

#### Playbooks 实际能力

Playbooks 为纯 **SaaS 托管服务**，无自部署选项。推测技术栈：

| 推断项 | 依据 |
|--------|------|
| **前端** | Next.js / Vercel 风格 SSR（基于 URL 结构和页面交互） |
| **认证** | NextAuth.js / Auth.js（GitHub OAuth callback 路径 `/api/auth/callback/github`） |
| **数据源** | GitHub API（技能内容） + 自有数据库（评分/统计/遥测） |
| **CDN/托管** | ⚠️ 未公开（可能 Vercel 或类似） |
| **支付** | Stripe（广告订阅） |

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| 三档部署（单机/生产/断网） | SaaS only | 🔴 缺失 |
| 统一代码库 + 配置驱动 | ⚠️ 未公开 | ⚠️ 未公开 |
| 14+ 监控 SLI | ⚠️ 未公开 | ⚠️ 未公开 |
| 备份/恢复（RPO 5min / RTO 30min） | ⚠️ 未公开 | ⚠️ 未公开 |
| 可断网/半断网运行 | 不可能（SaaS 依赖网络） | 🔴 缺失 |

---

### K. CLI

#### Playbooks 实际能力

Playbooks 提供了轻量 CLI（通过 `npx playbooks` 运行）：

| 命令 | 说明 |
|------|------|
| `npx playbooks add skill <owner/repo> --skill <name>` | 安装指定技能到本地 |
| `npx playbooks find skill` | 搜索/发现技能 |

**特点**：
- **无需安装**：通过 `npx` 即时运行
- **GitHub 路径式引用**：`owner/repo` 作为技能标识符（如 `anthropics/skills`）
- **工具感知**：安装时自动检测 Agent 工具类型，放置到对应目录
- **无登录要求**：匿名即可使用

#### 逐项对比

| 我方功能 | Playbooks | 对齐度 |
|---------|-----------|--------|
| 16+ 命令 | ~2 个命令（add/find） | 🔴 缺失 |
| `login/whoami` | 无（CLI 无需认证） | 🔴 缺失 |
| `search` | `npx playbooks find skill` | 🟢 对齐 |
| `install` | `npx playbooks add skill <ref>` | 🟢 对齐 |
| `list/update/uninstall` | 无 | 🔴 缺失 |
| `inspect` | 无（可在 Web 上查看详情） | 🔴 缺失 |
| `publish` | 无（通过 Web 提交） | 🔴 缺失 |
| `tag/rollback/yank/delete` | 无 | 🔴 缺失 |
| `config` | 无 | 🔴 缺失 |
| `audit/verify` | 无 | 🔴 缺失 |
| 多 Registry 配置 | 无（单一来源） | 🔴 缺失 |
| Lockfile v2 | 无 Lockfile | 🔴 缺失 |
| OAuth2 Device Flow + API Key | 无认证 | 🔴 缺失 |
| 信任策略配置 | 无 | 🔴 缺失 |
| 20+ 错误码体系 | ⚠️ 未公开 | ⚠️ 未公开 |
| JSON 输出模式 | ⚠️ 未公开 | ⚠️ 未公开 |

#### 差异影响评估

Playbooks CLI 的设计哲学是**极简**——安装和搜索是唯一需要的操作，其他管理功能在 Web 上完成或不需要（因为 GitHub 仓库是 Source of Truth）。值得注意的是其 `npx` 零安装体验和 GitHub 路径式引用的用户体验设计。

---

### L. 迁移路线

不适用。Playbooks 是独立平台，非迁移目标。

---

## 4. Playbooks 独有能力清单（我方设计未涉及）

| # | 功能 | 描述 | 参考价值 | 建议阶段 |
|---|------|------|---------|---------|
| 1 | **Skill Score 质量评分** | 对技能进行多维量化评分（数值型），用于排名和发现 | **高** | 阶段二：搜索增强 |
| 2 | **Health Score 健康评分** | iA 评分（如 100/100），评估技能健康度 | **高** | 阶段二：搜索增强 |
| 3 | **多工具安装遥测分布** | 追踪技能在不同 Agent 工具上的安装分布百分比（claude-code 87%, cursor 74%...） | **高** | 阶段三：生态分析 |
| 4 | **MCP 服务器目录** | 除 Skills 外，独立管理 MCP 服务器配置和文档 | **中** | 阶段三：扩展 |
| 5 | **Skill Bundles（技能包）** | 将相关技能打包为 Bundle 一键安装（如"React 全栈 Bundle"） | **高** | 阶段二：安装体验 |
| 6 | **Trigger Phrases** | 为每个技能标注触发短语，帮助 Agent 判断何时使用该技能 | **中** | 阶段二：元数据 |
| 7 | **多 Agent 工具适配** | 为 10+ 种 Agent 工具提供安装引导和配置片段 | **中** | 阶段三 |
| 8 | **广告平台** | 浮动 Banner、固定卡片、信息流广告三种广告形式，月费制 | **低** | 不建议 |
| 9 | **npx 零安装 CLI** | 通过 `npx playbooks` 即时使用，无需全局安装 | **中** | 阶段一：CLI DX |
| 10 | **Official 标记** | 区分官方认证技能和社区提交技能 | **高** | 阶段一：信任层 |
| 11 | **安全扫描 6 维度** | Persistence（持久化检测）和 Destructive Ops（破坏性操作）两个维度在我方扫描规则中未显式列出 | **高** | 阶段一：安全扫描 |
| 12 | **Topics 主题标签** | 细粒度功能领域分类（frontend/design/ux/accessibility 等）,与 Tags 分离 | **中** | 阶段二：分类体系 |
| 13 | **Weekly Installs** | 按周统计安装量（非仅累计），反映技能活跃度 | **中** | 阶段二：统计增强 |

---

## 5. 架构差异分析

### 5.1 技术栈对比

| 维度 | 我方设计 | Playbooks（推断） |
|------|---------|------------------|
| **后端** | Node.js / Go + PostgreSQL + Redis | Next.js API Routes / Server Actions（⚠️ 推断） |
| **数据库** | PostgreSQL + pgvector | ⚠️ 未公开（可能 Postgres / PlanetScale / Supabase） |
| **对象存储** | S3 / R2 / MinIO | 不需要（内容在 GitHub 上） |
| **缓存** | Redis（2–5min TTL） | ⚠️ 未公开 |
| **搜索** | PostgreSQL FTS → pgvector | ⚠️ 未公开 |
| **认证** | OAuth2 + API Key + OIDC | NextAuth.js + GitHub OAuth（⚠️ 推断） |
| **支付** | N/A | Stripe（广告业务） |
| **CLI** | 自研 `skill` CLI（Node.js / Go） | `npx playbooks`（npm 包） |
| **签名** | cosign / Sigstore | 无 |
| **扫描** | 静态规则 + VirusTotal + LLM | 静态规则（6 维度） |

### 5.2 部署模型对比

| 维度 | 我方设计 | Playbooks |
|------|---------|-----------|
| **部署形态** | 自托管（On-Premise / Private Cloud） | SaaS（公有云托管） |
| **多档适配** | 单机 PoC / 标准生产 / 高安全断网 | 仅 SaaS |
| **离线支持** | 完整离线降级（断路器 + 导出/导入） | 无（需联网） |
| **自定义** | 配置驱动全定制 | 无定制能力 |
| **运维** | 用户自运维 | 平台运维 |

### 5.3 安全模型对比

| 维度 | 我方设计 | Playbooks |
|------|---------|-----------|
| **信任模型** | 零信任 + 渐进式信任建立 | 信任 GitHub + 人工审核 |
| **包完整性** | cosign 签名 + SHA256 contentHash | 依赖 GitHub 的完整性保障 |
| **供应链安全** | SBOM + 来源证明 + Rekor 透明日志 | 无 |
| **访问控制** | RBAC（7 角色 × 25+ 权限） | 认证/匿名二元 |
| **安全扫描** | 三并行异步 + 门控阻断 | 按需静态扫描（不阻断） |
| **审计** | Append-only AuditLog | 无可见审计日志 |
| **Agent 安全** | 最小权限 + IP 绑定 + 短期 Token | N/A |

### 5.4 扩展性对比

| 维度 | 我方设计 | Playbooks |
|------|---------|-----------|
| **内容扩展** | Skills（核心） | Skills + MCP Servers + Bundles + Tools |
| **生态兼容** | ClawHub v1 兼容 + OpenClaw 执行层联动 | 10+ Agent 工具适配（展示层） |
| **第三方集成** | API 驱动，CI/CD 友好 | Web 为主，CLI 辅助 |
| **自定义策略** | YAML 策略引擎（可热更新） | 无 |
| **多 Registry** | 支持（.skillrc.yaml） | 单一来源 |

---

## 6. 结论与建议

### 6.1 Playbooks 的核心竞争力 / 差异化价值

1. **低摩擦的发现体验**：免费、无需登录的浏览和安装体验，`npx` 零安装 CLI，降低了用户入门门槛
2. **多工具生态覆盖**：同时覆盖 10+ 种 Agent 工具，统一了碎片化的 Skills/MCP 发现入口
3. **质量信号体系**：Skill Score + Health Score + 安装遥测分布 + Weekly Installs，形成了多维质量评价框架
4. **Skill Bundles**：技能包概念降低了批量采纳的摩擦
5. **社区规模**：20,000+ Skills、12,000+ MCP Servers 的目录规模形成了内容壁垒
6. **安全扫描工具**：6 维度静态分析提供了有价值的安全参考

### 6.2 我方设计的优势领域

1. **企业级安全**：cosign 签名、SBOM、Quarantine 门控、RBAC 策略引擎——Playbooks 在这一维度几乎完全空白
2. **供应链完整性**：包的不可变版本控制 + contentHash + Rekor 透明日志，vs Playbooks 的"GitHub 仓库就是真相"
3. **私有化部署**：三档部署方案支持断网/半断网，vs Playbooks 的纯 SaaS
4. **版本管理**：完整 semver + dist-tag + yank + rollback，vs Playbooks 的无版本模型
5. **Agent 运行时安全**：Gating 元数据 + 风险分级 + Sandbox 触发，vs Playbooks 的纯展示层
6. **审计与合规**：Append-only 审计日志 + 完整权限矩阵，vs Playbooks 的零审计能力
7. **离线/代理能力**：上游代理 + 缓存 + 离线降级，vs Playbooks 的联网依赖

### 6.3 建议补充或调整的设计点（按优先级排列）

| 优先级 | 建议 | 来源 | 影响 |
|--------|------|------|------|
| **P0** | 新增 **Persistence（持久化检测）** 和 **Destructive Ops（破坏性操作检测）** 两个静态扫描维度 | Playbooks Skill Scanner | 完善安全扫描覆盖面 |
| **P0** | 引入 **Official / Verified 标记**机制，区分官方认证技能 | Playbooks Official tag | 建立信任层级 |
| **P1** | 设计 **Skill Quality Score** 多维评分体系（安全评分 + 完整性评分 + 活跃度评分 + 社区评分） | Playbooks Skill Score + Health Score | 提升搜索排名和发现质量 |
| **P1** | 支持 **Skill Bundles（技能包）** 概念——一组相关技能的原子安装 | Playbooks Bundles | 降低批量采纳摩擦 |
| **P1** | 增加 **Weekly Installs** 时间窗口统计（而非仅累计下载量），用于 Trending 排序 | Playbooks Weekly Installs | 提供更准确的活跃度信号 |
| **P2** | 扩展**安装遥测**，追踪安装技能的 Agent 工具类型和分布 | Playbooks installed-on 遥测 | 为生态分析和兼容性测试提供数据 |
| **P2** | 在元数据中增加 **Trigger Phrases** 字段，帮助 Agent 判断技能触发条件 | Playbooks Trigger Phrases | 改善 Agent 自动技能选择能力 |
| **P2** | 考虑将 **MCP Server 配置** 作为一等公民资源类型（除 Skills 外） | Playbooks MCP 目录 | 扩大 Registry 覆盖范围 |
| **P3** | CLI 增加 `npx` 零安装运行支持（npm 包方式） | Playbooks npx CLI | 降低初次体验门槛 |
| **P3** | 引入 **Topics（主题标签）** 与 **Tags（版本标签）** 的分离 | Playbooks Tags vs Topics | 更灵活的分类体系 |

### 6.4 总结定位差异

```
┌─────────────────────────────────────────────────────────────────┐
│                        产品定位光谱                              │
│                                                                  │
│  开放目录          目录+轻量管理          企业级 Registry         │
│  (Discovery)       (Discovery+Mgmt)      (Full Lifecycle)        │
│                                                                  │
│  ◀── Playbooks ──▶               ◀──── 我方设计 ────▶           │
│                                                                  │
│  特点:免费/社区/     (中间地带:        特点:私有化/安全/           │
│  多工具/低摩擦       如ClawHub)        RBAC/签名/版本/            │
│                                        离线/审计                  │
└─────────────────────────────────────────────────────────────────┘
```

**核心结论**：Playbooks 与我方设计解决**不同层面的问题**。Playbooks 是面向个人开发者和开源社区的**技能发现平台**，而我方设计的是面向企业的**私有化 Skill Registry 管理系统**。两者在安全、版本管理、权限控制、部署方式上完全不重叠。Playbooks 在**发现体验、质量评分体系和多工具生态覆盖**方面有值得借鉴的设计，但其技术深度不足以作为企业级竞品。更准确地说，Playbooks 可视为我方 **Skill Store 展示层**的参考对象，而非 **Registry 服务端**的竞品。
