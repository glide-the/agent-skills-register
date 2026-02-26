# 设计文档 vs vercel-skills 差异分析报告

> **生成日期**: 2026-02-25  
> **对比对象**:  
> - **设计文档** (`docs/design/01~10`): 私有化 Skill Registry 完整技术设计（10 篇）  
> - **vercel-skills** (`/Users/dmeck/project/vercel-skills`): `skills` CLI v1.4.1（npm 包，Vercel Labs 开源）

---

## 一、项目定位差异（根本性差异）

| 维度 | 设计文档（私有 Registry） | vercel-skills |
|------|--------------------------|---------------|
| **产品形态** | 服务端 Registry + CLI 客户端（完整 npm-style 生态） | **纯 CLI 客户端**（无自有服务端） |
| **包来源** | 自建 Registry（兼容 ClawHub API） + 上游代理 | Git 仓库 / URL / well-known / skills.sh 搜索 API |
| **版本管理** | semver + dist-tag（latest/stable/canary） | **无版本概念**，以 Git tree SHA（folder hash）追踪变更 |
| **发布模型** | CLI → 两阶段上传 → 扫描 → 签名 → 审批 → 发布 | **无发布流程**——推送到 GitHub 即"发布" |
| **用户模型** | 多租户（User/Org/Team/RBAC） | **无用户系统**——仅使用 GitHub Token 提高 API 限速 |
| **安全哲学** | 深度防御（签名 + quarantine + sandbox + RBAC） | 轻量级安全审计（第三方报告展示，不阻止安装） |

> **核心结论**: vercel-skills 是一个大社区的开放生态 CLI 客户端；设计文档描述的是一个企业级私有化 Registry 系统。两者解决不同层面的问题，vercel-skills 可视为"安装侧"参考实现，但设计文档要求的"服务端"能力在 vercel-skills 中**完全不存在**。

---

## 二、功能模块逐项对比

### 2.1 API 端点（设计文档 01 & 02）

| 设计文档要求 | 端点 | vercel-skills 状态 | 说明 |
|-------------|------|-------------------|------|
| Skill 搜索 | `GET /search` | ⚠️ 部分覆盖 | 通过 `skills.sh/api/search` 外部 API 实现，非自建 |
| Skill 列表 | `GET /skills` | ❌ 缺失 | 无 Registry 端点 |
| Skill 详情 | `GET /skills/{slug}` | ❌ 缺失 | 安装靠直接指向 Git URL |
| 版本列表 | `GET /skills/{slug}/versions` | ❌ 缺失 | 无版本概念 |
| 包下载 | `GET /download` | ❌ 缺失 | 直接 git clone / HTTP fetch |
| 文件查看 | `GET /skills/{slug}/file` | ❌ 缺失 | — |
| 身份认证 | `GET /whoami` | ❌ 缺失 | 无认证体系 |
| Token 管理 | `POST /auth/token` | ❌ 缺失 | 仅用 GitHub Token 做限速 |
| 发布 | `POST /skills`, `POST /uploads` | ❌ 缺失 | 无发布流程 |
| 删除/恢复 | `DELETE /skills/{slug}`, `POST /undelete` | ❌ 缺失 | — |
| Tag 管理 | `POST /skills/{slug}/tags` | ❌ 缺失 | — |
| **扩展端点** | — | — | — |
| 健康检查 | `GET /health` | ❌ 缺失 | — |
| Yank 版本 | `POST /skills/{slug}/versions/{ver}/yank` | ❌ 缺失 | — |
| 签名查询 | `GET /skills/{slug}/signatures` | ❌ 缺失 | — |
| 审计日志 | `GET /audit` | ❌ 缺失 | — |
| 组织管理 | `/orgs/*` | ❌ 缺失 | — |
| Team 管理 | `/orgs/{org}/teams/*` | ❌ 缺失 | — |
| Quarantine | `/quarantine/*` | ❌ 缺失 | — |
| SBOM/Provenance | `/skills/{slug}/versions/{ver}/sbom` | ❌ 缺失 | — |

**覆盖率: 1/20+ 端点（仅搜索通过外部 API 部分覆盖）**

---

### 2.2 数据模型（设计文档 03）

| 设计文档模型 | 字段数 | vercel-skills 对应 | 覆盖度 |
|-------------|--------|-------------------|--------|
| **Skill** | 15+ (slug, namespace, owner, visibility, risk_level, status…) | `Skill { name, description, path }` — 3 个核心字段 | 🔴 ~20% |
| **SkillVersion** | 15+ (version, content_hash, status, quarantine_reason…) | 无——folder hash 替代版本 | 🔴 0% |
| **Tag** | 4 (name, version_id…) | 无 | 🔴 0% |
| **User** | 8+ (provider, is_admin, external_created_at…) | 无用户模型 | 🔴 0% |
| **Organization** | 5+ | 无 | 🔴 0% |
| **Team** | 4+ | 无 | 🔴 0% |
| **OrgMember / TeamMember** | 4+ | 无 | 🔴 0% |
| **TeamSkill** | 3+ | 无 | 🔴 0% |
| **APIToken** | 10+ (scopes, allowed_skills, expires_at…) | 仅 `GITHUB_TOKEN` / `GH_TOKEN` 环境变量 | 🔴 ~5% |
| **AuditLog** | 12+ (append-only) | 仅 telemetry fire-and-forget 事件 | 🔴 ~10% |
| **ScanResult** | 8+ (scanner, severity, findings…) | 第三方审计结果（Gen/Socket/Snyk），不持久化 | 🔴 ~15% |
| **Signature** | 9+ (cosign bundle, certificate…) | 无 | 🔴 0% |
| **Lock File** | — | ✅ 两层 Lock（Global v3 + Local v1） | 🟡 概念对齐，结构差异大 |

---

### 2.3 包格式与签名（设计文档 04）

| 特性 | 设计文档 | vercel-skills |
|------|---------|---------------|
| **包格式** | `.skill.zip` / `.skill.tar.gz` + `manifest.json` | **无打包**——直接复制/symlink SKILL.md + 相关文件 |
| **manifest.json** | 完整规范（20+ 字段含 contentHash, openclaw metadata） | **无 manifest** |
| **Content Hash** | `SHA256(sort(files.map(f => f.sha256)).join("\n"))` | Local lock: SHA-256 from disk files；Global lock: GitHub tree SHA |
| **cosign 签名** | Keyless OIDC + Offline Key 双模式 | ❌ **无任何签名** |
| **签名验证** | 安装时 hash check + cosign verify-blob | ❌ 无 |
| **SBOM** | CycloneDX 1.5 自动生成 | ❌ 无 |
| **包大小限制** | 50MB | 无限制（取决于 git clone） |
| **组织级签名策略** | `requireSignature`, `trustedIdentities`, `unsignedPolicy` | ❌ 无 |

---

### 2.4 代理/镜像层（设计文档 05）

| 特性 | 设计文档 | vercel-skills |
|------|---------|---------------|
| **上游代理** | on-demand 缓存 + 预拉取 + quarantine 门控 | ❌ 无代理层——直接访问源 |
| **缓存策略** | 分层 TTL（Redis + Object Storage） | ❌ 无缓存（每次安装重新 clone/fetch） |
| **离线支持** | Export/Import 离线包 + stale-while-revalidate | ❌ 无离线支持（断网即不可用） |
| **断路器** | healthy→degraded→circuitOpen→halfOpen 状态机 | ❌ 无 |
| **多 Registry 配置** | `.skillrc.yaml` 多 registry + 优先级 | ❌ 无（skills.sh 硬编码） |
| **Quarantine 门控** | 自动扫描 → 审批/拒绝 | ❌ 无 |

---

### 2.5 搜索（设计文档 06）

| 特性 | 设计文档 | vercel-skills |
|------|---------|---------------|
| **Phase 1: 全文搜索** | PostgreSQL FTS + 加权字段 + GIN 索引 | ⚠️ 依赖 `skills.sh/api/search` 外部 API |
| **Phase 2: 向量搜索** | pgvector + embedding (OpenAI/bge-m3) | ❌ 无 |
| **Phase 3: 混合搜索** | RRF 融合 + 下载热度 + 时效衰减 | ❌ 无 |
| **命名空间/可见性过滤** | `namespace=corp&status=active` | ❌ 无——全局公开搜索 |
| **交互式 UI** | — | ✅ **亮点**: fzf 风格实时搜索 + debounce + 方向键导航 |
| **安装集成** | — | ✅ 搜索选中后直接安装 |

> vercel-skills 的交互式搜索 UI 设计值得借鉴，但后端搜索能力远不及设计文档要求。

---

### 2.6 发布流水线（设计文档 07）

| 特性 | 设计文档 | vercel-skills |
|------|---------|---------------|
| **两阶段上传** | Init → presigned URL → Commit | ❌ 无（推 Git 即发布） |
| **静态规则扫描** | CRED_HARVEST, CMD_INJECT, SOCIAL_ENG, EXFIL, OBFUSCATION | ❌ 无服务端扫描 |
| **VirusTotal 扫描** | 异步 + blocking | ❌ 无 |
| **LLM 内容审查** | 异步 advisory | ❌ 无 |
| **自定义策略** | 可配置 | ❌ 无 |
| **签名验证** | cosign verify | ❌ 无 |
| **SBOM 生成** | CycloneDX 自动生成 | ❌ 无 |
| **Quarantine/审批** | Gate 决策引擎 | ❌ 无 |
| **Tag 管理** | set/move tag = rollback | ❌ 无 tag 概念 |
| **Yank（不可逆隐藏）** | yank 保留元数据 | ❌ 无 |
| **安全审计展示** | — | ⚠️ 安装时展示 Gen/Socket/Snyk 报告（仅展示，不阻止安装） |

---

### 2.7 RBAC 权限模型（设计文档 08）

| 特性 | 设计文档 | vercel-skills |
|------|---------|---------------|
| **角色体系** | 7 角色 (PlatformAdmin → Viewer/Auditor) | ❌ **无角色系统** |
| **资源层级** | Platform→Org→Team→Skill→Version | ❌ 无 |
| **Token Scopes** | 6 种 scope (read/write/admin:skills/audit/org/policy) | ❌ 无（GitHub Token 仅用于限速） |
| **身份类型** | Human (OAuth) + Service Account (API Key) + Agent (Session Token) | ❌ 无身份系统 |
| **策略引擎** | Scope + Role + IP + Time + Approval + Trust + Version policies | ❌ 无 |
| **双重校验** | Role + Scope 双验证 | ❌ 无 |
| **Break-glass Token** | 紧急访问 + 高优先级审计告警 | ❌ 无 |

---

### 2.8 OpenClaw 集成（设计文档 09）

| 特性 | 设计文档 | vercel-skills |
|------|---------|---------------|
| **Gating 元数据** | `risk.level`, `sandbox.*`, `capabilities` | ❌ 无风险元数据 |
| **风险自动分级** | low/medium/high/critical 矩阵 | ❌ 无 |
| **Sandbox 触发** | riskPolicy → Docker sandbox 配置 | ❌ 无 sandbox 概念 |
| **Env/ApiKey 注入安全** | 6 条安全规则（R1-R6） | ❌ 无 |
| **Lockfile 扩展** | riskLevel, capabilities, sandboxAdvice | ❌ Lock file 无安全字段 |
| **Agent 路径检测** | 专属 OpenClaw 路径 | ✅ OpenClaw 多目录检测（`.openclaw`, `.clawdbot`, `.moltbot`） |

> vercel-skills 对 OpenClaw 仅做了路径检测，未集成任何安全联动。

---

### 2.9 部署架构（设计文档 10）

| 特性 | 设计文档 | vercel-skills |
|------|---------|---------------|
| **三档部署** | 单机 → 标准生产 → 高安全/断网 | ❌ 无部署方案（纯 CLI） |
| **对象存储** | S3/R2/MinIO 选型 | ❌ 无（文件直接存本地磁盘） |
| **数据库** | PostgreSQL + pgvector | ❌ 无（JSON lock files 替代） |
| **缓存/队列** | Redis / Redis Stream / NATS | ❌ 无 |
| **监控** | Prometheus + Grafana + Loki + 7 个 SLI | ❌ 无 |
| **备份恢复** | RPO 5min / RTO 30min | ❌ 无 |
| **config.yaml** | 统一配置 + feature flags | ❌ 无（硬编码配置） |

---

## 三、vercel-skills 独有功能（设计文档未覆盖）

以下是 vercel-skills 实现了但设计文档中**未明确要求或未涉及**的功能：

| 功能 | vercel-skills 实现 | 设计文档建议 |
|------|-------------------|-------------|
| **41+ Agent 支持** | 自动检测 41 个 AI Agent 目录并安装 | 仅聚焦 OpenClaw |
| **Universal Agent 架构** | 共享 `.agents/skills/` + symlink 到各 agent 目录 | 未涉及多 agent 分发 |
| **Symlink 安装模式** | 避免文件重复，跨 agent symlink | 未涉及 |
| **Git 多格式源解析** | GitHub/GitLab/HuggingFace/Mintlify/well-known 7 种源 | 仅 Registry URL |
| **`@skill` 语法** | `owner/repo@skill-name` 选择性安装 | 未涉及 |
| **Well-Known 协议** | `/.well-known/skills/index.json` (RFC 8615) | 未涉及，可作为互操作补充 |
| **node_modules 同步** | 从 npm 包中提取 SKILL.md | 未涉及 |
| **Plugin Manifest** | `.claude-plugin/marketplace.json` 集成 | 未涉及 |
| **交互式 fzf 搜索** | 自定义终端 UI + debounce | 设计文档搜索是 API 侧 |
| **Source Aliases** | 短名映射（如 `coinbase/agentWallet`→ 实际仓库） | 未涉及 |
| **Private Repo 检测** | GitHub API 检查仓库可见性并警告 | 未涉及 |
| **CI 环境检测** | 检测 7 种 CI 环境调整行为 | 未涉及 |
| **npm provenance 发布** | CLI 本身发布带 npm provenance | 未涉及 |

---

## 四、可借鉴点

从 vercel-skills 的成熟实现中，以下设计模式值得设计文档方案借鉴：

### 4.1 多 Agent 分发架构
vercel-skills 的 Universal/Symlink 模式解决了"一个 skill → N 个 agent 目录"的去重问题。设计文档当前仅聚焦 OpenClaw，若未来扩展应考虑类似架构。

### 4.2 两层 Lock File 设计
- **Global Lock**（用户级，追踪更新）  
- **Local Lock**（项目级，可提交 VCS，团队共享）  
设计文档的 lockfile 仅描述了单层，可参考两层分离。

### 4.3 交互式搜索 UX
fzf 风格的实时搜索 + debounce + 方向键 + 直接安装，用户体验极佳。设计文档的 CLI 规范可补充交互模式。

### 4.4 Source Parser 灵活性
vercel-skills 支持 7 种源格式（shorthand、URL、local、well-known 等），设计文档的 CLI 目前仅计划 `--registry` 单源。可在扩展阶段支持 Git 直连作为备选安装源。

### 4.5 Well-Known Protocol
`/.well-known/skills/index.json` 是一个轻量级技能发现协议，适合作为设计文档"联邦搜索"的补充。

### 4.6 安全审计展示
安装时自动展示 Gen/Socket/Snyk 三方安全评估结果的 UX 模式，可在私有 CLI 中复用（对接内部扫描结果）。

---

## 五、总结矩阵

| 设计文档模块 | 覆盖度 | 评级 | 说明 |
|-------------|--------|------|------|
| 01 - ClawHub API 分析 | 5% | 🔴 | vercel-skills 不对接 ClawHub API |
| 02 - API 兼容性 | 0% | 🔴 | 无 Registry 服务端 |
| 03 - 数据模型 | 10% | 🔴 | 仅有简单 Skill 类型 + Lock 文件 |
| 04 - 包签名 | 0% | 🔴 | 无签名、无 manifest |
| 05 - 代理/镜像 | 0% | 🔴 | 无代理、无缓存、无离线 |
| 06 - 搜索 | 20% | 🟡 | 有搜索 UI，依赖外部 API |
| 07 - 发布流水线 | 5% | 🔴 | 仅安全审计展示（不阻止） |
| 08 - RBAC | 0% | 🔴 | 无权限系统 |
| 09 - OpenClaw 集成 | 10% | 🔴 | 仅路径检测 |
| 10 - 部署架构 | 0% | 🔴 | 纯 CLI，无服务端部署 |
| **CLI 命令丰富度** | — | 🟢 | vercel-skills CLI 更丰富（10 命令 vs 设计文档 6 命令） |
| **Agent 生态覆盖** | — | 🟢 | vercel-skills 支持 41 agent，设计文档聚焦 OpenClaw |
| **源格式灵活性** | — | 🟢 | 7 种源 vs 设计文档仅 Registry URL |

---

## 六、结论与建议

### 6.1 vercel-skills 不能替代设计文档方案
vercel-skills 是一个**安装侧 CLI 工具**，解决的是"从哪里获取 skill 并放到正确位置"的问题。设计文档描述的是一个完整的**私有化 Registry 系统**（服务端 + 安全流水线 + RBAC + 审计），两者定位完全不同。vercel-skills 的 10 个设计模块中仅搜索和 CLI UX 有部分参考价值。

### 6.2 建议的集成策略

```
┌─────────────────────────────────────────────────────┐
│              集成架构建议                              │
│                                                     │
│  vercel-skills (安装侧)    私有 Registry (服务端)      │
│  ┌───────────────────┐    ┌──────────────────────┐  │
│  │ 多 Agent 分发      │    │ API 兼容层 (ClawHub)  │  │
│  │ Symlink 架构       │    │ 发布流水线            │  │
│  │ 源解析 (7 种)      │◄──►│ 签名验证             │  │
│  │ 交互式搜索 UX     │    │ RBAC/Quarantine      │  │
│  │ Lock File         │    │ 代理/镜像层           │  │
│  │ Well-Known 协议   │    │ 向量搜索              │  │
│  └───────────────────┘    └──────────────────────┘  │
│        ↑                           ↑                 │
│    CLI 客户端侧（可借鉴）     服务端侧（需全部新建）   │
└─────────────────────────────────────────────────────┘
```

1. **短期**: 将 vercel-skills 的 CLI UX 模式（交互搜索、多 agent 分发、两层 lock）作为私有 CLI 的设计参考
2. **中期**: 扩展私有 CLI 支持 vercel-skills 的源格式（Git URL、well-known）作为 Registry 之外的备选安装源
3. **长期**: 考虑让私有 Registry 实现 well-known 协议端点，使 vercel-skills 用户可直接发现私有 skills

### 6.3 设计文档方案的不可替代性

以下设计文档能力在 vercel-skills 中**完全不存在**，是私有化的核心价值：

| 能力 | 企业价值 |
|------|---------|
| cosign 包签名 + 验签 | 供应链安全底线 |
| Quarantine + 审批流 | 恶意技能防护 |
| RBAC (Org/Team/Skill) | 企业多团队协作 |
| 发布扫描流水线 | 准入控制 |
| 上游代理 + 离线支持 | 合规 + 网络隔离 |
| 风险分级 + Sandbox 联动 | 运行时安全 |
| 审计日志 (append-only) | 合规审计 |
| SBOM | 软件供应链透明度 |
