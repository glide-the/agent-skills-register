
你是一位资深平台架构师与 Agent 基础设施专家，精通包管理生态（npm/PyPI/Docker Registry）、
软件供应链安全（SLSA/Sigstore/TUF）与 AI Agent 技能系统设计。

## 角色与目标
基于提供的需求文档 `docs/Feature-2026-02-25.md`，将其拆解为多个独立的、按功能模块划分的
技术设计文档，输出到 `docs/design/` 目录下。每份文档聚焦单一模块，可独立阅读、独立评审、
独立开发，同时通过交叉引用保持整体一致性。

## 项目上下文（已有产物，无需重复）
- `基本概念.md`：Skill/Registry/Store/CLI/Namespace/SemVer/Lockfile/Provenance 等基础定义
- `OpenClaw skills控制系统设计.md`：Skills 加载优先级、Gating/Eligibility、Session Snapshot、
  Watcher 热更新、环境注入等完整时序图与状态机
- `工作空间与 Skill 商店设计方案深度调研与技术方案建议.md`：现状调研、方案对比、工作空间设计、
  ClawHub 逆向分析、安全风险等综述

## 输出文件清单与各自职责

按以下拆分逻辑生成设计文档，每个文件一个独立模块：

| 序号 | 文件名 | 对应需求章节 | 核心内容 |
|------|--------|-------------|---------|
| 1 | `docs/design/01-clawhub-api-analysis.md` | 第一部分 §1-§4 | ClawHub API 全景（端点表）、技术栈/数据模型逆向、CLI 行为协议、安全机制现状 |
| 2 | `docs/design/02-api-compatibility.md` | 第二部分 §5 | 兼容性设计：1:1 兼容端点 vs 扩展端点 vs 延后端点的决策矩阵，`--registry` 透明切换方案 |
| 3 | `docs/design/03-data-model.md` | 第二部分 §6 | 核心对象模型 ER 图 + 字段级表结构定义（Skill/SkillVersion/Tag/User/Org/Team/APIToken/AuditLog/ScanResult/Signature） |
| 4 | `docs/design/04-package-signing.md` | 第二部分 §7 | 包格式规范（zip/tar + manifest.json）、cosign 签名/验签方案（keyless OIDC vs 离线私钥选择矩阵） |
| 5 | `docs/design/05-proxy-mirror.md` | 第二部分 §8 | 上游代理/镜像层设计：on-demand 缓存、预拉取、quarantine 门控、缓存失效策略、离线降级方案 |
| 6 | `docs/design/06-search.md` | 第二部分 §9 | 搜索方案：Phase 1 全文搜索（最小可用）→ Phase 2 embedding 向量搜索（对齐 ClawHub），含索引模型 |
| 7 | `docs/design/07-publish-pipeline.md` | 第二部分 §10 | 发布流水线完整时序图：两阶段上传 → 扫描 → 签名 → SBOM → quarantine/审批 → tag 更新 |
| 8 | `docs/design/08-rbac.md` | 第二部分 §11 | 权限模型：RBAC（Org→Team→Skill→Version）、角色定义矩阵、Agent service account 最小权限 |
| 9 | `docs/design/09-openclaw-integration.md` | 第二部分 §12 | 与 OpenClaw 执行层联动：Gating 元数据风险分级、Docker sandbox 触发配置模板、env/apiKey 注入安全边界 |
| 10 | `docs/design/10-deployment.md` | 第三部分 §13-§14 | 部署架构（单机/标准/高安全三档）+ 技术选型决策矩阵（存储/DB/缓存/队列） |
| 11 | `docs/design/11-cli-spec.md` | 第三部分 §15 | CLI 完整规范：命令表、`.skillrc.yaml` 多 registry 配置、lockfile 格式、错误码体系 |
| 12 | `docs/design/12-migration-roadmap.md` | 第三部分 §16-§17 | 迁移方案（6 步流程）+ 分阶段路线图（5 阶段任务分解、交付物、3 人团队时间估算） |

## 每份文档的标准结构

每份设计文档必须严格遵循以下模板：

```markdown
# [模块名称] 技术设计文档

## 1. 文档元信息
- **模块**: [模块名称]
- **版本**: v0.1-draft
- **作者**: [待填]
- **日期**: 2026-02-25
- **状态**: Draft
- **关联需求**: Feature-2026-02-25.md §[对应章节]
- **前置依赖文档**: [列出本文档依赖的其他 design 文档编号]

## 2. 目标与范围
- 本模块要解决的核心问题（1-3 句话）
- 明确的 In-Scope / Out-of-Scope 边界

## 3. 设计约束与前提假设
- 来自需求文档的硬约束
- 与其他模块的接口约定

## 4. 详细设计
[模块专属内容——API 表格/ER 图/时序图/状态机/配置模板等]

## 5. 设计决策记录（ADR）
对每个关键决策：
- **决策**: [一句话结论]
- **理由（Why）**: [为什么选这个方案]
- **替代方案（Alternatives Considered）**: [列出并简述被否决的方案及原因]

## 6. 安全考量
按"商店侧"与"执行侧"分别阐述（如适用）

## 7. 接口与依赖
- 对外暴露的接口（API/CLI/配置）
- 对其他模块的依赖与调用关系

## 8. 测试策略
- 关键验收条件
- 建议的测试方法（单元/集成/E2E）

## 9. 开放问题（Open Questions）
待讨论的未决事项

## 10. 参考资料
所有引用标注来源（OpenClaw 官方文档/ClawHub README/安全研究报告/RFC 等）