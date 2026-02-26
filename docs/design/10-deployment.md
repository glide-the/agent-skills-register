# 10 — 部署架构与技术选型

> **Design Doc** · 私有 Skill Registry  
> **状态**：Draft  
> **作者**：Platform Arch Team  
> **关联 Feature**：§13-§14 部署架构与技术选型  
> **依赖文档**：[03-data-model](03-data-model.md) · [05-proxy-mirror](05-proxy-mirror.md) · [06-search](06-search.md) · [07-publish-pipeline](07-publish-pipeline.md)

---

## 1. 目标与范围

### 1.1 目标

| # | 目标 | 验收标准 |
|---|------|----------|
| G1 | 提供三档部署方案覆盖不同规模 | 单机/标准/高安全三种部署均有文档化拓扑 |
| G2 | 同一代码基座适配全部部署形态 | 通过配置（非代码分支）切换部署档位 |
| G3 | 给出技术选型决策矩阵 | 存储/数据库/缓存/队列/搜索均有对比和推荐 |
| G4 | 部署方案支持可断网/半断网 | 高安全档可完全内网运行 |
| G5 | 运维可观测性达到最小可用 | QPS/延迟/队列深度/错误率监控到位 |

### 1.2 范围

- **In Scope**：三档部署拓扑、技术选型矩阵、运维监控最小清单、容量规划公式、备份恢复策略
- **Out of Scope**：Kubernetes Helm Chart 模板细节、CI/CD Pipeline 具体实现

---

## 2. 设计约束与前提假设

| 约束/假设 | 来源 | 说明 |
|-----------|------|------|
| 按部署复杂度三档分级 | Feature §13 | 单机→标准生产→高安全/可断网 |
| 同一代码基座 | 架构原则 | 环境差异通过配置/特性开关/部署编排解决 |
| S3 兼容对象存储 | Feature §14 | 支持 AWS S3、CloudFlare R2、MinIO |
| PostgreSQL 为元数据库 | 03-data-model | pgvector 扩展满足向量搜索需求 |
| 3 人团队 | Feature §17 | 1 后端 + 1 全栈 + 1 平台/安全 |

---

## 3. 详细设计

### 3.1 整体架构参考

```mermaid
flowchart TB
  subgraph Clients["客户端层"]
    CLI["Agent CLI"]
    UI["Web UI"]
    CI["CI/CD Pipeline"]
  end

  subgraph Gateway["接入层"]
    LB["负载均衡 / 反向代理\n(Nginx / Caddy / ALB)"]
    WAF["WAF\n(可选)"]
  end

  subgraph App["应用层"]
    API["Registry API\n(Node.js / Go)"]
    Auth["Auth Service\n(OAuth2/OIDC)"]
    Worker["Async Worker\n(扫描/签名/SBOM)"]
  end

  subgraph Data["数据层"]
    DB["PostgreSQL\n+ pgvector"]
    Cache["Redis\n(缓存 + 限流)"]
    Queue["消息队列\n(Redis Stream / NATS)"]
    ObjStore["对象存储\n(S3 / R2 / MinIO)"]
    Search["搜索引擎\n(Meilisearch，可选)"]
  end

  subgraph Sec["安全层"]
    Scanner["安全扫描器\n(静态规则 + VirusTotal)"]
    Signer["签名服务\n(cosign)"]
  end

  subgraph Obs["可观测性"]
    Metrics["Prometheus\n+ Grafana"]
    Logs["日志收集\n(Loki / ELK)"]
    Trace["分布式追踪\n(Jaeger / OTLP)"]
  end

  CLI & UI & CI --> LB
  LB --> WAF --> API
  API --> Auth
  API --> DB & Cache & ObjStore & Search
  API --> Queue
  Queue --> Worker
  Worker --> Scanner & Signer
  Worker --> DB & ObjStore
  API & Worker --> Metrics & Logs & Trace
```

### 3.2 三档部署方案

#### 档位一：单机 / 小团队（PoC / MVP）

**适用场景**：≤10 人团队、评估验证、开发测试

```mermaid
flowchart TB
  subgraph SingleNode["单节点"]
    direction TB
    Caddy["Caddy\n(反向代理 + TLS)"]
    API["Registry API"]
    PG["PostgreSQL"]
    Redis["Redis"]
    MinIO["MinIO\n(本地对象存储)"]
  end

  CLI["CLI / Agent"] --> Caddy
  Caddy --> API
  API --> PG & Redis & MinIO

  note["所有组件运行在单台机器\nDocker Compose 部署\n扫描同步执行（无队列）"]:::note
  classDef note fill:#fff9c4,stroke:#f9a825,color:#333;
```

| 维度 | 规格 |
|------|------|
| **部署方式** | Docker Compose（单 `docker-compose.yml`） |
| **机器配置** | 2 vCPU / 4GB RAM / 50GB SSD |
| **反向代理** | Caddy（自动 HTTPS） |
| **数据库** | PostgreSQL 16（单实例，含 pgvector） |
| **缓存** | Redis 7（单实例） |
| **对象存储** | MinIO（单节点，数据目录映射到宿主机） |
| **搜索** | PostgreSQL FTS（Phase 1） |
| **队列** | 无（扫描同步执行） |
| **扫描** | 同步模式：API 进程内直接执行静态规则扫描 |
| **备份** | pg_dump 定时 + MinIO 目录 rsync |
| **监控** | 基础健康检查（`/health` endpoint） |

#### 档位二：标准生产

**适用场景**：10-200 人团队、正式使用

```mermaid
flowchart TB
  subgraph LBLayer["接入层"]
    LB["Nginx / ALB\n(TLS + 限流)"]
  end

  subgraph AppCluster["应用层 (2-4 replica)"]
    API1["API #1"]
    API2["API #2"]
    W1["Worker #1"]
    W2["Worker #2"]
  end

  subgraph DataLayer["数据层"]
    PG_Primary["PostgreSQL Primary"]
    PG_Replica["PostgreSQL Replica"]
    Redis_Primary["Redis Primary"]
    Redis_Replica["Redis Replica"]
    S3["S3 / R2\n(托管对象存储)"]
    MS["Meilisearch\n(可选)"]
  end

  subgraph Queue["消息队列"]
    RedisStream["Redis Stream"]
  end

  CLI["CLI"] --> LB
  LB --> API1 & API2
  API1 & API2 --> PG_Primary & Redis_Primary & S3 & MS
  API1 & API2 --> RedisStream
  RedisStream --> W1 & W2
  PG_Primary --> PG_Replica
  Redis_Primary --> Redis_Replica
```

| 维度 | 规格 |
|------|------|
| **部署方式** | Kubernetes（Helm Chart）或 Docker Compose + systemd |
| **API 副本** | 2-4 pods，HPA 按 CPU/QPS 自动扩缩 |
| **Worker 副本** | 2 pods（扫描/签名可独立扩缩） |
| **数据库** | PostgreSQL 16 主从（RDS/CloudSQL/自建） + pgvector |
| **缓存** | Redis 7 主从（ElastiCache/Memorystore/自建） |
| **对象存储** | AWS S3 / CloudFlare R2 / 兼容 S3 的托管服务 |
| **搜索** | Meilisearch 单节点（可选：PostgreSQL FTS 作为降级方案） |
| **队列** | Redis Stream（复用 Redis 实例） |
| **反向代理** | Nginx + Let's Encrypt 或 ALB/CLB |
| **监控** | Prometheus + Grafana + Loki 或云厂商监控 |
| **备份** | PostgreSQL 自动备份（PITR）+ S3 版本化 |

#### 档位三：高安全 / 可断网

**适用场景**：金融/政务/军工、完全内网、合规审计要求高

```mermaid
flowchart TB
  subgraph DMZ["隔离区 (DMZ)"]
    ImportGW["导入网关\n(单向同步)"]
  end

  subgraph InternalNet["内网"]
    subgraph LBLayer["接入层"]
      LB["内网 LB + mTLS"]
      WAF["WAF"]
    end

    subgraph AppCluster["应用层"]
      API["API (3+ replica)"]
      Worker["Worker (2+ replica)"]
      Auth["内网 OIDC / LDAP"]
    end

    subgraph DataLayer["数据层"]
      PG["PostgreSQL HA\n(Patroni / Citus)"]
      Redis["Redis Sentinel"]
      MinIO_HA["MinIO HA\n(Erasure Coding)"]
      Search["Meilisearch / pgvector"]
    end

    subgraph SecLayer["安全层"]
      HSM["HSM / KMS\n(签名密钥)"]
      Scanner["离线扫描器"]
      SIEM["SIEM\n(审计日志导出)"]
    end
  end

  External["外部上游\n(ClawHub)"] -->|"离线导出 bundle"| ImportGW
  ImportGW -->|"扫描 + 审批后导入"| API
  
  InternalCLI["内网 CLI"] --> LB
  LB --> WAF --> API
  API --> Auth & PG & Redis & MinIO_HA & Search
  API --> Worker
  Worker --> Scanner & HSM
  Worker --> PG & MinIO_HA
  API --> SIEM
```

| 维度 | 规格 |
|------|------|
| **部署方式** | Kubernetes（air-gapped registry）或 VM + Ansible |
| **网络** | 完全内网，mTLS 服务间通信 |
| **认证** | 内网 OIDC / LDAP / AD 集成（无外网 OAuth 依赖） |
| **对象存储** | MinIO HA（Erasure Coding，4+ 节点） |
| **数据库** | PostgreSQL HA（Patroni + pgBouncer） |
| **签名** | HSM / 内网 KMS 管理私钥（不依赖 Sigstore 公共服务） |
| **上游同步** | 隔离区导入网关：离线 bundle 导入 → 扫描 → 审批 → 入库 |
| **审计** | 全量审计日志 → SIEM 导出（Splunk / ELK） |
| **备份** | DB 双站点同步 + 对象存储跨机房复制 + 加密离线备份 |

### 3.3 技术选型决策矩阵

#### 对象存储

| 维度 | AWS S3 | CloudFlare R2 | MinIO |
|------|--------|---------------|-------|
| **类型** | 托管云服务 | 托管云服务 | 自建 / 私有化 |
| **S3 兼容** | 原生 | 完全兼容 | 完全兼容 |
| **Egress 费用** | 按量计费 | 免费 | 不适用（内网） |
| **延迟** | 区域内 <10ms | 全球 edge <50ms | 取决于部署位置 |
| **HA** | 99.999999999% | 高（CloudFlare 基础设施） | 需自建 Erasure Coding |
| **离线支持** | ❌ | ❌ | ✅ |
| **推荐场景** | 标准生产（AWS 用户） | 标准生产（多云/成本敏感） | 单机/高安全/断网 |

**推荐**：标准生产选 R2（零 egress 适合高下载量场景）或 S3（AWS 生态用户）；高安全/断网选 MinIO。

#### 元数据数据库

| 维度 | PostgreSQL + pgvector | PostgreSQL + 外置搜索 |
|------|----------------------|----------------------|
| **全文搜索** | 内置 FTS（tsvector/tsquery） | 内置 FTS + 外置增强 |
| **向量搜索** | pgvector 扩展 | 外置：Meilisearch / OpenSearch |
| **运维复杂度** | 低（单一数据库） | 中（需维护额外搜索集群） |
| **搜索质量** | 中（FTS 满足基本需求） | 高（外置搜索支持更丰富的分词/排序） |
| **推荐档位** | 单机 + 标准生产 | 标准生产（高搜索需求）+ 高安全 |

**推荐**：v1 统一 PostgreSQL + pgvector；搜索增强按需引入 Meilisearch。

#### 缓存

| 维度 | Redis | 无缓存 |
|------|-------|--------|
| **用途** | 元数据缓存 + 限流 + 队列 | — |
| **是否需要** | 标准/高安全必要；单机可选 | 单机极小规模 |
| **部署模式** | 单实例 / 主从 / Sentinel | — |

**推荐**：统一使用 Redis（单机档位也建议启用以支持限流）。

#### 消息队列

| 维度 | Redis Stream | NATS | 无队列 |
|------|-------------|------|--------|
| **复杂度** | 低（复用 Redis） | 中（独立部署） | 最低 |
| **可靠性** | 中（ACK + 持久化） | 高（JetStream） | — |
| **推荐档位** | 标准生产 | 高安全 | 单机 |

**推荐**：标准生产用 Redis Stream（减少组件数）；高安全可选 NATS JetStream。

#### 搜索引擎

| 维度 | PostgreSQL FTS | Meilisearch | OpenSearch |
|------|---------------|-------------|-----------|
| **部署复杂度** | 零（内置） | 低（单二进制） | 高（JVM 集群） |
| **搜索质量** | 基础 | 高（typo-tolerance, facets） | 最高（可定制） |
| **向量搜索** | pgvector | v1.6+ 实验性 | 插件支持 |
| **推荐档位** | 单机 / 标准 Phase 1 | 标准 Phase 2+ | 高安全/大规模 |

### 3.4 运维与监控

#### 指标体系（最小可用清单）

| 分类 | 指标 | 说明 | 阈值建议 |
|------|------|------|---------|
| **可用性** | API 成功率 | 2xx / (2xx + 5xx) | > 99.9% |
| **性能** | p95 响应时间 | 含鉴权 + 业务逻辑 | < 200ms |
| **性能** | p99 响应时间 | 尾部延迟 | < 500ms |
| **吞吐** | QPS | 每秒请求数 | 视档位容量 |
| **下载** | 下载成功率 | 完整下载 / 总下载请求 | > 99.5% |
| **发布** | 扫描队列深度 | 待处理扫描任务数 | < 50 |
| **发布** | 扫描平均耗时 | 单版本完整扫描时间 | < 60s |
| **安全** | Quarantine 累积数 | 待审批版本数 | 按需告警 |
| **安全** | 签名校验失败数 | 每小时失败次数 | > 0 立即告警 |
| **安全** | 401/403 率 | 鉴权失败百分比 | > 5% 告警 |
| **安全** | 429 率 | 限流触发百分比 | > 10% 告警 |
| **存储** | 对象存储使用量 | 总 artifact 大小 | 容量预警线 80% |
| **缓存** | 缓存命中率 | Redis 命中 / 总请求 | > 80% |
| **数据库** | 连接池使用率 | active / max | > 80% 告警 |

#### 日志规范

| 日志类型 | 内容 | 存储策略 |
|---------|------|---------|
| **请求日志** | requestId, method, path, status, latency, userId | 保留 90 天 |
| **审计日志** | action, actor, resource, result, timestamp, IP | 永久保留（append-only） |
| **下载日志** | slug, version, digest, userId, IP | 保留 180 天 |
| **发布日志** | slug, version, fileHashes, signatureId, scanResult | 永久保留 |
| **错误日志** | stack trace, requestId, context | 保留 30 天 |

#### 分布式追踪

关键链路需全程 trace（带 traceId/spanId）：

1. **发布链路**：upload init → file upload → commit → scan → sign → gate decision → tag update
2. **安装链路**：resolve tag → download → verify signature → verify hash → extract
3. **搜索链路**：query parse → search backend → rank → response

### 3.5 容量规划

#### 存储估算公式

```
月新增存储 = 每日发布版本数 × 平均版本包大小 × 30

示例：
- 每日发布 20 个版本
- 平均版本包大小 500KB（Skill 通常为文本 + 少量资源）
- 月新增 = 20 × 0.5MB × 30 = 300MB/月
- 含签名/SBOM = 300MB × 1.2 = 360MB/月
- 年增长 ≈ 4.3GB（不含冗余/版本化）
```

#### 数据库估算

```
每版本元数据 ≈ 2KB（skill + version + tags + audit）
月新增行数 ≈ 每日发布 × 30 × 4（skill + version + audit × 2）
= 20 × 30 × 4 = 2400 行/月

索引膨胀：通常为数据量的 2-3 倍
FTS 索引：额外 50% 空间
pgvector 索引（如启用）：额外 100% 空间
```

### 3.6 备份与恢复

| 组件 | 备份策略 | RPO | RTO | 恢复方法 |
|------|---------|-----|-----|---------|
| PostgreSQL | 每日全量 + WAL 归档（PITR）| 5 min | 30 min | pg_restore / PITR |
| Redis | AOF + 每小时 RDB snapshot | 1 hour | 10 min | 从 RDB/AOF 恢复 |
| 对象存储 | 版本化启用 + 跨区域复制 | 0 (不可变) | 10 min | 从版本历史恢复 |
| Meilisearch | 每日 dump export | 1 day | 30 min | dump import |
| 配置 | Git 管理 | 0 | 5 min | git checkout + apply |

---

## 4. 设计决策记录（ADR）

### ADR-DEP-001：三档共用同一代码基座

- **决策**：单一代码仓库，通过配置文件/环境变量/特性开关切换部署形态
- **理由**：
  - 降低维护成本，避免多分支代码分化
  - 配置驱动便于测试（CI 可跑全部档位的集成测试）
  - Docker Compose / Kubernetes / Ansible 仅是部署编排差异
- **替代方案**：按档位拆分代码库（维护负担高）、微服务拆分（MVP 阶段过度设计）
- **实现方式**：
  - `config.yaml` 中 `deployment.tier: single | standard | secure`
  - 特性开关：`features.asyncScan: true/false`、`features.externalSearch: true/false`

### ADR-DEP-002：Redis Stream 而非独立 MQ

- **决策**：标准生产档位使用 Redis Stream 作为异步任务队列
- **理由**：
  - 复用 Redis 实例，减少组件数量和运维负担
  - 消费者组 + ACK 满足扫描任务的可靠投递
  - 扫描任务量级有限（非高吞吐场景）
- **替代方案**：RabbitMQ（重量级）、NATS JetStream（高安全档可选）、数据库轮询（性能差）

### ADR-DEP-003：R2 为默认推荐对象存储

- **决策**：默认推荐 CloudFlare R2 用于标准生产
- **理由**：
  - 零 egress 费用对 Skill 下载密集型场景有显著成本优势
  - 完全 S3 兼容，切换成本低
  - 全球 edge 缓存降低下载延迟
- **替代方案**：S3（AWS 用户生态优势）、MinIO（断网必选）
- **切换成本**：S3 兼容 API → 仅改 endpoint + credentials

---

## 5. 安全考量

### 5.1 商店侧

| 威胁 | 缓解措施 |
|------|---------|
| 数据库被攻破 | 加密存储敏感字段（API Key hash）+ 网络隔离 + 最小权限 DB 用户 |
| 对象存储未授权访问 | 预签名 URL（有时效）+ 存储桶策略 + 加密（SSE-S3/SSE-KMS） |
| Redis 未授权 | 绑定内网 + requirepass + TLS（生产环境） |
| 服务间通信被窃听 | 标准生产：TLS；高安全：mTLS |
| DDoS / 滥用 | WAF + 限流（Redis 令牌桶）+ 429 响应 |

### 5.2 执行侧

| 威胁 | 缓解措施 |
|------|---------|
| 高安全环境外网泄露 | 完全内网部署 + 隔离区单向同步 |
| 备份被篡改 | 备份加密 + 校验和 + 异地存储 |
| 配置泄露 | 敏感配置通过环境变量/Vault 注入，不持久化到文件 |

---

## 6. 接口与依赖

### 6.1 部署配置接口

```yaml
# config.yaml 核心配置结构
server:
  host: "0.0.0.0"
  port: 8080
  tls:
    enabled: true
    certFile: "/certs/server.crt"
    keyFile: "/certs/server.key"

deployment:
  tier: "standard"  # single | standard | secure

database:
  url: "postgres://user:pass@host:5432/registry"
  maxConnections: 20
  enablePgvector: true

redis:
  url: "redis://host:6379"
  password: ""
  tls: false

objectStorage:
  provider: "s3"  # s3 | r2 | minio
  endpoint: "https://xxx.r2.cloudflarestorage.com"
  bucket: "skill-artifacts"
  accessKey: "${OBJ_ACCESS_KEY}"
  secretKey: "${OBJ_SECRET_KEY}"
  region: "auto"

search:
  backend: "postgres"  # postgres | meilisearch
  meilisearch:
    url: "http://search:7700"
    apiKey: "${MEILI_API_KEY}"

queue:
  backend: "redis-stream"  # redis-stream | nats | sync
  nats:
    url: "nats://host:4222"

features:
  asyncScan: true
  externalSearch: false
  vectorSearch: false
  proxyUpstream: true

auth:
  issuer: "https://auth.mycorp.internal"
  audience: "skill-registry"

upstream:
  clawhub:
    url: "https://clawhub.com"
    enabled: true
    cacheTTL: "1h"

monitoring:
  prometheus:
    enabled: true
    port: 9090
  tracing:
    enabled: true
    exporter: "otlp"
    endpoint: "http://jaeger:4317"
```

### 6.2 依赖关系

| 依赖组件 | 用途 |
|---------|------|
| 03-data-model | 数据库 schema / migration |
| 05-proxy-mirror | 上游代理缓存配置 |
| 06-search | 搜索引擎选型与索引结构 |
| 07-publish-pipeline | 异步 Worker 扫描任务设计 |
| 08-rbac | Auth Service 集成 |

---

## 7. 测试策略

| 层次 | 覆盖内容 | 方法 |
|------|---------|------|
| **冒烟测试** | 各档位部署是否可用 | Docker Compose up → 健康检查 → 基础 CRUD |
| **集成测试** | 组件间通信 | API → DB + Redis + S3 + Search 全链路 |
| **性能测试** | 各档位 capacity | k6/vegeta 压测：标准生产目标 500 QPS / p99 < 500ms |
| **容灾测试** | 单组件故障 | kill DB/Redis/S3 → 验证降级行为 + 恢复 |
| **安全测试** | 网络隔离 | 高安全档：扫描外网不可达 + mTLS 验证 |
| **备份恢复** | 数据完整性 | 模拟全量恢复 → 验证数据一致 |
| **升级测试** | 零停机升级 | 滚动更新 → 验证请求无中断 |

---

## 8. 开放问题

| # | 问题 | 建议方向 | 优先级 |
|---|------|---------|--------|
| Q1 | API 服务用 Node.js 还是 Go？ | 建议 Node.js（团队技能栈 + ClawHub 生态一致）；Go 适合高性能需求 | P1 |
| Q2 | Kubernetes 是否为标准生产必选？ | 不必选；Docker Compose + systemd 可覆盖中等规模 | P2 |
| Q3 | 是否需要多区域部署？ | v1 不需要；v2 评估跨区域只读副本 | P3 |
| Q4 | 高安全档的导入网关如何审批？ | 建议 Web UI 审批流 + API 审批接口 | P2 |
| Q5 | 监控是否需要自建 Grafana 看板？ | 提供默认 dashboard JSON，运维按需定制 | P2 |

---

## 9. 参考资料

| 来源 | 说明 |
|------|------|
| Feature §13-§14 | 部署架构与技术选型需求 |
| 深度调研报告 §私有化部署 | 三档方案描述 |
| 深度调研报告 §运维监控 | 最小可用清单 |
| 深度调研报告 §成本估算 | 对象存储计费模型 |
| CloudFlare R2 定价 | 零 egress 特性参考 |
| PostgreSQL + pgvector 文档 | 向量搜索扩展 |
| MinIO 文档 | Erasure Coding 高可用 |
