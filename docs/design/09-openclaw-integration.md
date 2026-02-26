# 09 — OpenClaw 执行层联动设计

> **Design Doc** · 私有 Skill Registry  
> **状态**：Draft  
> **作者**：Platform Arch Team  
> **关联 Feature**：§12 与 OpenClaw 执行层联动  
> **依赖文档**：[03-data-model](03-data-model.md) · [04-package-signing](04-package-signing.md) · [07-publish-pipeline](07-publish-pipeline.md) · [08-rbac](08-rbac.md)

---

## 1. 目标与范围

### 1.1 目标

| # | 目标 | 验收标准 |
|---|------|----------|
| G1 | 将 Skill Registry 元数据与 OpenClaw 执行层 Gating 机制打通 | metadata.openclaw.requires.* 字段在注册/安装/执行三个环节均被检验 |
| G2 | 根据 Skill 风险等级自动触发 Docker sandbox 运行 | 高风险 Skill 100% 在 sandbox 内执行 |
| G3 | 控制 env/apiKey 注入的安全边界 | 敏感凭据不泄露到 prompt/log/非 sandbox 环境 |
| G4 | 为运维提供可配置的风险分级策略 | 策略变更不需要重启 Agent 或 Registry |

### 1.2 范围

- **In Scope**：Gating 元数据规范、风险分级矩阵、sandbox 触发配置模板、env/apiKey 注入安全边界、与 OpenClaw 加载/执行生命周期的交互
- **Out of Scope**：Docker sandbox 本身的实现细节（由 OpenClaw 提供）、Agent 调度与编排

---

## 2. 设计约束与前提假设

| 约束/假设 | 来源 | 说明 |
|-----------|------|------|
| OpenClaw 技能加载优先级 | OpenClaw 文档 | workspace > local > bundled > extraDirs |
| SKILL.md frontmatter 规范 | AgentSkills 兼容 | `name`/`description` 必填，`metadata` 可扩展 |
| OpenClaw Gating 机制 | OpenClaw 文档 | `metadata.openclaw.requires.{bins,anyBins,env,config}` + `primaryEnv` |
| Docker sandbox 支持 | OpenClaw 文档 | `agents.defaults.sandbox` / per-agent sandbox 配置 |
| env 注入生命周期 | OpenClaw 文档 | per-run 注入 host process.env → run 结束恢复 |
| sandbox 不继承 host env | OpenClaw 文档 | 需要显式传入或烘焙到镜像 |

---

## 3. 详细设计

### 3.1 Skill 生命周期与检验点

```mermaid
flowchart LR
  subgraph Registry["Registry（商店侧）"]
    Publish["发布时\n静态分析"]
    Index["索引时\n风险分级"]
  end

  subgraph Install["安装侧"]
    Download["下载时\n签名+策略校验"]
    Extract["解压时\nmanifest校验"]
  end

  subgraph Execute["执行侧（OpenClaw）"]
    Load["加载时\nGating 过滤"]
    Gate["调用时\nsandbox 决策"]
    Run["运行时\nenv 注入"]
  end

  Publish --> Index --> Download --> Extract --> Load --> Gate --> Run

  style Registry fill:#e3f2fd,stroke:#1e88e5
  style Install fill:#e8f5e9,stroke:#43a047
  style Execute fill:#fff3e0,stroke:#ef6c00
```

### 3.2 Gating 元数据规范

#### SKILL.md frontmatter 扩展字段

```yaml
# SKILL.md frontmatter 示例（完整 Gating 声明）
---
name: database-migration
description: "自动化数据库迁移工具"
version: 1.2.0
metadata:
  openclaw:
    # Gating 条件（OpenClaw 内置支持）
    requires:
      bins:
        - psql          # 必须存在的二进制
        - pg_dump
      anyBins:
        - docker        # 至少有一个即可
        - podman
      env:
        - DATABASE_URL  # 必须设置的环境变量
      config:
        - skills.entries.database-migration.apiKey  # 必须配置的 config 项
    primaryEnv: DATABASE_URL  # apiKey 映射到此 env
    install:
      nodeManager: true  # 安装提示

    # === Registry 扩展字段（私有 Skill Registry 专用）===
    risk:
      level: high        # 风险等级声明：low / medium / high / critical
      reasons:
        - "requires direct database access"
        - "executes shell commands"
    sandbox:
      required: true     # 强制 sandbox 运行
      network: restricted  # none / restricted / full
      workspaceAccess: rw   # none / ro / rw
      allowedBinds:       # 允许的 bind mount 路径
        - "/data/migrations:ro"
      denyBinds:          # 禁止的 bind mount 路径
        - "/etc"
        - "/root"
    capabilities:
      exec: true          # 声明使用 exec/shell 能力
      network: true       # 声明需要网络访问
      fileWrite: true     # 声明需要文件写入
      browser: false      # 声明使用浏览器
---
```

#### 字段定义表

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `metadata.openclaw.requires.bins` | string[] | 否 | 必须存在的系统二进制列表 |
| `metadata.openclaw.requires.anyBins` | string[] | 否 | 至少存在一个的二进制列表 |
| `metadata.openclaw.requires.env` | string[] | 否 | 必须设置的环境变量 |
| `metadata.openclaw.requires.config` | string[] | 否 | 必须设置的 OpenClaw config 项 |
| `metadata.openclaw.primaryEnv` | string | 否 | apiKey 映射的目标环境变量名 |
| `metadata.openclaw.risk.level` | enum | 否 | 风险等级：`low` / `medium` / `high` / `critical` |
| `metadata.openclaw.risk.reasons` | string[] | 否 | 风险原因说明 |
| `metadata.openclaw.sandbox.required` | boolean | 否 | 是否强制 sandbox 运行 |
| `metadata.openclaw.sandbox.network` | enum | 否 | 网络访问：`none` / `restricted` / `full` |
| `metadata.openclaw.sandbox.workspaceAccess` | enum | 否 | 工作空间访问：`none` / `ro` / `rw` |
| `metadata.openclaw.sandbox.allowedBinds` | string[] | 否 | 允许的 bind mount 路径（`path[:mode]`） |
| `metadata.openclaw.sandbox.denyBinds` | string[] | 否 | 禁止的 bind mount 路径 |
| `metadata.openclaw.capabilities` | object | 否 | 能力声明 map |

### 3.3 风险分级矩阵

Registry 在发布时和索引时对 Skill 进行风险分级，分级结果存入 `SkillVersion.riskLevel` 字段。

#### 自动分级规则

```mermaid
flowchart TB
  Start["解析 SKILL.md\nfrontmatter + 指令内容"]
  
  Start --> C1{"声明 exec/bins\n或含 shell 命令？"}
  C1 -->|Yes| HighBase["base = high"]
  C1 -->|No| C2{"声明 network\n或含外联 URL？"}
  
  C2 -->|Yes| MedBase["base = medium"]
  C2 -->|No| C3{"声明 fileWrite\n或 workspaceAccess=rw？"}
  
  C3 -->|Yes| MedBase2["base = medium"]
  C3 -->|No| LowBase["base = low"]

  HighBase --> Scan["静态规则扫描"]
  MedBase --> Scan
  MedBase2 --> Scan
  LowBase --> Scan

  Scan --> S1{"命中恶意 pattern？\n(凭据收集/可疑下载等)"}
  S1 -->|Yes| Critical["final = critical\n→ quarantine"]
  S1 -->|No| S2{"来源可信？\n(内部 CI + 签名)"}
  
  S2 -->|Yes| Lower["final = max(base-1, low)"]
  S2 -->|No| Keep["final = base"]
  
  Critical --> Done["写入 riskLevel"]
  Lower --> Done
  Keep --> Done
```

#### 分级对照表

| 风险等级 | 触发条件 | Registry 侧处置 | 执行侧建议 |
|---------|---------|-----------------|-----------|
| **low** | 纯文本 SOP，无 exec/network/file | 自动发布 | 宿主直接运行 |
| **medium** | 声明 network / fileWrite / browser | 自动发布 + 安全审查通知 | 建议 sandbox |
| **high** | 声明 exec/bins / 含 shell 命令 | quarantine（需审批） | 强制 sandbox |
| **critical** | 命中恶意 pattern / 扫描异常 | 拒绝发布 / quarantine + 告警 | 拒绝加载 |

### 3.4 Sandbox 触发配置模板

Registry 发布的 Skill 携带风险分级和 sandbox 建议。OpenClaw 侧通过配置模板将这些建议转化为实际 sandbox 行为。

#### OpenClaw 配置模板（`openclaw.json`）

```jsonc
{
  "skills": {
    // 全局策略：风险等级 → sandbox 行为映射
    "riskPolicy": {
      "low": {
        "sandbox": false
      },
      "medium": {
        "sandbox": true,
        "sandboxConfig": {
          "network": "restricted",
          "workspaceAccess": "ro",
          "binds": []
        }
      },
      "high": {
        "sandbox": true,
        "sandboxConfig": {
          "network": "none",
          "workspaceAccess": "ro",
          "binds": [],
          "cpuLimit": "1.0",
          "memoryLimit": "512m",
          "timeout": 300
        }
      },
      "critical": {
        "action": "block",  // 拒绝加载
        "reason": "Critical risk skills are blocked by policy"
      }
    },

    // Per-skill 覆盖（优先级高于全局策略）
    "entries": {
      "database-migration": {
        "enabled": true,
        "sandbox": true,
        "sandboxConfig": {
          "network": "restricted",
          "workspaceAccess": "rw",
          "binds": ["/data/migrations:ro"],
          "env": {
            "DATABASE_URL": "${skills.entries.database-migration.apiKey}"
          }
        },
        "apiKey": "postgres://user:pass@host/db"
      },
      "code-review": {
        "enabled": true,
        "riskOverride": "low"  // 手动覆盖风险等级
      }
    }
  },

  "agents": {
    "defaults": {
      "sandbox": true  // 全局默认启用 sandbox
    }
  }
}
```

#### Sandbox 配置决策流

```mermaid
flowchart TB
  Start["Skill 被 Agent 调用"]
  
  Start --> PerSkill{"entries.{skill}\n有 sandbox 配置？"}
  PerSkill -->|Yes| UsePerSkill["使用 per-skill config"]
  PerSkill -->|No| RiskPolicy{"读取 riskLevel\n查找 riskPolicy 映射"}
  
  RiskPolicy --> HasPolicy{"有匹配策略？"}
  HasPolicy -->|Yes| CheckBlock{"action == block？"}
  CheckBlock -->|Yes| Block["拒绝加载\n返回错误"]
  CheckBlock -->|No| UsePolicy["使用 riskPolicy config"]
  HasPolicy -->|No| AgentDefault{"agents.defaults\n.sandbox == true？"}
  
  AgentDefault -->|Yes| DefaultSandbox["使用默认 sandbox"]
  AgentDefault -->|No| NoSandbox["宿主直接运行"]

  UsePerSkill --> Merge["合并最终 sandbox 配置"]
  UsePolicy --> Merge
  DefaultSandbox --> Merge
  NoSandbox --> Execute["执行 Skill"]
  Merge --> Execute
```

### 3.5 env / apiKey 注入安全边界

OpenClaw 在每次 Agent run 时将 `skills.entries.*.env` / `skills.entries.*.apiKey` 注入 `process.env`，run 结束后恢复。这一机制存在安全隐患，需要严格的边界控制。

#### 注入生命周期

```mermaid
sequenceDiagram
  participant Config as openclaw.json
  participant OClaw as OpenClaw Runtime
  participant Env as Host process.env
  participant SB as Docker Sandbox
  participant Skill as Skill Execution

  Note over OClaw: Agent Run 开始
  OClaw->>Config: 读取 skills.entries.*.env/apiKey
  OClaw->>Env: 注入 env 变量 (host)
  
  alt Sandbox 模式
    OClaw->>SB: 创建容器 (显式传入 env)
    SB->>Skill: 执行 Skill（隔离环境）
    Note over SB: sandbox 内 env 由 sandboxConfig.env 控制
    Note over SB: 不继承 host process.env
    Skill-->>SB: 完成
    SB-->>OClaw: 结果
  else 宿主模式
    OClaw->>Skill: 执行 Skill（宿主环境）
    Note over Skill: 可访问全部 host process.env
    Note over Skill: ⚠️ 风险：env 变量可能被其他进程读取
    Skill-->>OClaw: 结果
  end

  Note over OClaw: Agent Run 结束
  OClaw->>Env: 恢复原始 env
```

#### 安全约束规则

| 规则 | 说明 | 实施方 |
|------|------|--------|
| **R1：Sandbox 内 env 最小化** | sandbox 容器仅传入该 Skill 声明需要的 env，不传入其他 Skill 的凭据 | OpenClaw Runtime |
| **R2：禁止 env 出现在 prompt** | System prompt 构建时不得包含 `apiKey` / `env` 值（仅传变量名） | OpenClaw Runtime |
| **R3：禁止 env 出现在日志** | Agent 运行日志中对已知 env key 的值做脱敏 (`***`) | OpenClaw Runtime |
| **R4：高风险 Skill 强制 sandbox** | `riskLevel >= high` 的 Skill 不允许在宿主模式运行 | OpenClaw + Policy |
| **R5：跨 Skill 隔离** | Skill A 的 apiKey 不可被 Skill B 访问（sandbox 间隔离；宿主模式依赖进程隔离） | OpenClaw Runtime |
| **R6：凭据轮换通知** | Registry 可标记某 Skill 的 apiKey binding，管理员可触发轮换提醒 | Registry API |

#### 安全的 env 传递配置示例

```yaml
# 推荐：通过 sandboxConfig.env 显式声明传入的变量
skills:
  entries:
    database-migration:
      apiKey: "postgres://user:pass@host/db"  # 存储在 openclaw.json（宿主侧）
      sandbox: true
      sandboxConfig:
        env:
          DATABASE_URL: "${apiKey}"  # 仅此变量传入 sandbox
          # 不传入 HOME, PATH 等无关变量
        network: restricted
        workspaceAccess: ro

    code-review:
      env:
        GITHUB_TOKEN: "ghp_xxx"
      sandbox: true
      sandboxConfig:
        env:
          GITHUB_TOKEN: "${env.GITHUB_TOKEN}"
        network: restricted  # 仅允许访问 github.com
```

### 3.6 Registry ↔ OpenClaw 数据流

```mermaid
flowchart TB
  subgraph Registry["Private Skill Registry"]
    Meta["Skill 元数据\n(riskLevel, capabilities, sandbox建议)"]
    Artifact["Skill 包\n(zip + signature)"]
    Policy["安全策略\n(信任策略, 风险门控)"]
  end

  subgraph CLI["Agent CLI"]
    Install["skill install\n下载 + 签名校验"]
    Lockfile["lock.json\n(version, hash, riskLevel)"]
  end

  subgraph OpenClaw["OpenClaw Runtime"]
    Loader["Skill Loader\n(workspace/local/bundled)"]
    Gating["Gating Filter\n(requires.* check)"]
    SandboxMgr["Sandbox Manager\n(risk→sandbox mapping)"]
    EnvMgr["Env Manager\n(per-run inject/restore)"]
  end

  Meta -->|API| Install
  Artifact -->|download| Install
  Policy -->|trust verification| Install
  Install --> Lockfile
  
  Lockfile -->|riskLevel cache| SandboxMgr
  Loader -->|SKILL.md parse| Gating
  Gating -->|pass| SandboxMgr
  SandboxMgr -->|config| EnvMgr
  EnvMgr -->|"inject env → execute"| Run["Skill Run"]
```

### 3.7 Lockfile 扩展（riskLevel 缓存）

安装时将 Registry 返回的 `riskLevel` 写入 lockfile，OpenClaw 可直接读取而无需重新计算。

```json
{
  "version": "1",
  "skills": {
    "database-migration": {
      "version": "1.2.0",
      "registry": "private",
      "contentHash": "sha256:abc123...",
      "signature": "verified",
      "riskLevel": "high",
      "capabilities": ["exec", "network", "fileWrite"],
      "sandboxAdvice": {
        "required": true,
        "network": "restricted",
        "workspaceAccess": "rw"
      },
      "installedAt": "2026-03-15T10:00:00Z"
    }
  }
}
```

---

## 4. 设计决策记录（ADR）

### ADR-INT-001：风险分级由 Registry 侧计算

- **决策**：风险分级在 Registry 发布/索引时计算，结果随元数据分发
- **理由**：
  - Registry 拥有最完整的分析能力（静态扫描 + 恶意检测 + 历史审计）
  - 避免每个 OpenClaw 实例重复计算
  - 分级结果写入 lockfile 后可离线使用
- **替代方案**：OpenClaw 本地计算（延迟高、能力弱、不一致）
- **风险**：lockfile 中的 riskLevel 可能被篡改 → 通过签名校验缓解

### ADR-INT-002：Sandbox 建议 vs 强制

- **决策**：Registry 提供 `sandboxAdvice`，OpenClaw 最终决策
- **理由**：
  - Registry 不控制执行环境，只能建议
  - 运维团队需要灵活覆盖策略（per-skill override）
  - OpenClaw 可能升级 sandbox 实现，需要兼容空间
- **替代方案**：Registry 强制决定 sandbox 行为（耦合太紧）
- **折中**：对 `riskLevel = high/critical`，Registry 在 API 响应中标记 `sandboxRequired = true`，OpenClaw 必须尊重（否则记录策略违规审计）

### ADR-INT-003：env 注入采用显式声明

- **决策**：sandbox 模式下，env 必须在 `sandboxConfig.env` 中显式列出，不做全量透传
- **理由**：
  - 全量透传会将不相关的凭据暴露给 Skill
  - 显式声明便于审计和最小权限
  - OpenClaw 文档明确 sandbox 不继承 host process.env
- **替代方案**：全量透传 + denylist（难以穷举需要屏蔽的变量）

---

## 5. 安全考量

### 5.1 商店侧

| 威胁 | 攻击向量 | 缓解措施 |
|------|---------|---------|
| 风险等级欺骗 | 发布者不声明 capabilities 以降低分级 | 静态扫描 + 内容分析覆盖声明式字段，不仅依赖自声明 |
| 恶意 metadata 注入 | frontmatter 中注入恶意 YAML | YAML 解析后严格 schema 校验 + 白名单字段 |
| Lockfile 篡改 | 修改 lockfile 中的 riskLevel | 签名覆盖 lockfile 完整性；OpenClaw 可对 riskLevel 做二次校验 |

### 5.2 执行侧

| 威胁 | 攻击向量 | 缓解措施 |
|------|---------|---------|
| sandbox 逃逸 | 容器漏洞或错误配置的 binds | 限制 allowedBinds + denyBinds + 最小权限容器配置 |
| 凭据窃取 | Skill 读取非自身的 env 变量 | R1+R5：sandbox 间隔离 + 显式 env 声明 |
| 提示注入 | Skill 指令诱导 Agent 暴露 env | R2：env 值不出现在 system prompt |
| Side-channel 泄露 | 通过日志/错误信息泄露 env | R3：日志脱敏 + 错误消息不含 env 值 |
| watcher 热更新投毒 | 运行中替换 skill 文件 | OpenClaw watcher 刷新后重新校验签名 + hash |
| Workspace 覆盖攻击 | 在 workspace/skills/ 放入同名 skill | workspace 优先级最高 → 限制 skills/ 目录写入者 + 完整性监测 |

---

## 6. 接口与依赖

### 6.1 Registry API 扩展（风险与建议）

| Method | Path | 说明 |
|--------|------|------|
| GET | `/skills/{slug}` | 响应中包含 `riskLevel`、`capabilities`、`sandboxAdvice` |
| GET | `/skills/{slug}/versions/{version}/risk` | 查询指定版本的完整风险评估详情 |
| GET | `/download?slug=...&version=...` | 响应 header 中包含 `X-Risk-Level`、`X-Sandbox-Required` |

### 6.2 依赖关系

| 依赖组件 | 用途 |
|---------|------|
| 07-publish-pipeline | 发布时触发风险分级计算 |
| 04-package-signing | 签名覆盖 lockfile 完整性 |
| 08-rbac | 策略覆盖需要 OrgAdmin+ 权限 |
| OpenClaw Runtime | 实际执行 sandbox/env 注入 |
| OpenClaw Gating | 处理 requires.* 过滤 |

---

## 7. 测试策略

| 层次 | 覆盖内容 | 方法 |
|------|---------|------|
| **单元测试** | 风险分级规则 | SKILL.md fixtures → 断言 riskLevel |
| **单元测试** | sandbox 配置决策树 | 给定 riskLevel + per-skill config → 断言最终 sandbox config |
| **集成测试** | 发布→分级→安装→lockfile | 端到端 publish + install → 验证 lockfile 含 riskLevel |
| **集成测试** | OpenClaw 加载阶段 | 安装 high-risk skill → OpenClaw 启动 → 验证 sandbox 生效 |
| **安全测试** | env 隔离 | sandbox A 内尝试读取 skill B 的 env → 断言不可见 |
| **安全测试** | 风险等级欺骗 | 不声明 capabilities 但含 exec 命令 → 断言静态扫描提升分级 |
| **安全测试** | lockfile 篡改 | 修改 lockfile 中 riskLevel → 安装/校验失败 |

---

## 8. 开放问题

| # | 问题 | 建议方向 | 优先级 |
|---|------|---------|--------|
| Q1 | OpenClaw 是否会原生支持读取 lockfile 中的 riskLevel？ | 需要与 OpenClaw 团队协调；否则通过 config 模板实现 | P1 |
| Q2 | 如何处理 workspace skill（非 Registry 安装）的风险分级？ | 本地扫描工具 + 默认 high risk | P2 |
| Q3 | sandbox image 版本如何管理？ | 建议随 OpenClaw 版本固定 + 企业可自定义 base image | P2 |
| Q4 | watcher 热更新后是否需要重新校验签名？ | 建议强制校验；性能影响需评估 | P1 |
| Q5 | 是否需要支持"能力声明↔实际行为"的运行时监测？ | v2 方向：sandbox 内行为审计 + 异常告警 | P3 |

---

## 9. 参考资料

| 来源 | 说明 |
|------|------|
| Feature §12 | 与 OpenClaw 执行层联动需求 |
| OpenClaw Skills 控制系统设计 | Gating、加载优先级、watcher、session snapshot |
| OpenClaw 安全文档 | sandbox 配置、tool policy、access control before intelligence |
| 深度调研报告 §执行侧安全 | Docker sandbox 最佳实践、最小权限工具策略 |
| 深度调研报告 §Gating 元数据 | metadata.openclaw.requires.* 字段规范 |
