## OneID 定义

OneID（统一身份标识）是面向 SaaS 电商平台的用户身份识别体系，用于将同一个自然人在不同业务场景下产生的多个分散 ID 标识（手机号、邮箱、设备 ID、OpenID、Cookie ID 等）关联为唯一的身份实体。

### 核心概念

- **本质**：OneID 是一个全局唯一的虚拟身份标识，代表一个"自然人"，而非某个具体业务系统中的账号
- **目的**：解决"同一个用户在不同渠道/品牌/集团拥有多个账号"的身份碎片化问题，实现用户身份的跨系统、跨业务统一识别
- **生成方式**：基于图计算中的连通分量算法（WCC），将共享 ID 标识的用户节点聚合为同一个连通分量，每个连通分量对应一个 OneID

### 三层架构

OneID 采用三层独立的 ID 空间设计，各层独立计算、互不依赖：


| 层级               | 范围   | 数据边界          | 合并逻辑           |
| ---------------- | ---- | ------------- | -------------- |
| **品牌层 (Brand)**  | 单个品牌 | 仅该品牌的原始用户数据   | 品牌内同一自然人合并     |
| **集团层 (Group)**  | 单个集团 | 该集团下所有品牌的原始数据 | 同一自然人在集团内跨品牌合并 |
| **全局层 (Global)** | 所有集团 | 所有集团的原始数据     | 同一自然人跨集团合并     |


### 数据结构

#### OneID 主实体

```
OneID {
  // ── 基础标识 ──
  one_id: String,              // OneID 唯一标识（编码规则见下方）
  scope: Enum,                 // 层级: "brand" | "group" | "global"
  scope_id: String,            // 品牌ID / 集团ID / "global"
  status: Enum,                // 状态: "active" | "frozen" | "merged" | "split"
  version: Integer,            // 乐观锁版本号，每次更新 +1

  // ── ID 映射 ──
  id_mappings: List<IdMapping>,  // 关联的原始 ID 标识列表

  // ── 用户画像 ──
  physical_info: PhysicalInfo,  // 物理属性：从业务系统直接采集融合的客观事实
  ai_profile: AIProfile,        // AI 属性：通过算法/模型计算得出的衍生标签

  // ── 溯源与治理 ──
  source: String,              // 数据来源标识（如 "cyberbiz"）
  batch_id: String,            // 生成/更新该 OneID 的计算批次号
  confidence: Float,           // OneID 置信度 (0.0~1.0)，由 WCC 连通强度和 ID 标识数量决定

  // ── 生命周期 ──
  created_at: Timestamp,       // 首次创建时间
  updated_at: Timestamp,       // 最近一次更新时间
  merged_into: String?,        // 若 status="merged"，指向合并后的目标 OneID
  split_from: String?,         // 若 status="split"，指向拆分前的源 OneID
  expired_at: Timestamp?       // 过期时间（长期不活跃用户可标记过期）
}
```

#### IdMapping（ID 标识映射）

```
IdMapping {
  id_type: Enum,               // 标识类型: "phone" | "email" | "device_id" | "openid" | "unionid" | "cookie_id" | "third_party_id"
  id_value: String,            // 标识值（归一化后，如手机号去 +86 前缀、邮箱转小写）
  source_system: String,       // 来源系统（如 "wechat_mini_program"、"app_ios"、"web_pc"）
  is_primary: Boolean,         // 是否为主 ID（用于冲突消解时的优先级判定）
  weight: Float,               // 权重（手机号 > 邮箱 > 设备ID > CookieID，用于 WCC 边权重）
  first_seen_at: Timestamp,    // 首次观测到该标识的时间
  last_seen_at: Timestamp,     // 最近观测时间（用于时间衰减计算）
  status: Enum                 // "active" | "deprecated" | "suspicious"
}
```

#### PhysicalInfo（物理属性 — 直接采集，客观事实）

```
PhysicalInfo {
  // ── 个人基本信息 ──
  nickname: String?,           // 昵称（取最高优先级来源）
  gender: Enum?,               // 性别: "male" | "female" | "unknown"
  birthday: Date?,             // 生日
  avatar_url: String?,         // 头像 URL
  phone: String?,              // 手机号（主 ID 对应的手机号）
  email: String?,              // 邮箱（主 ID 对应的邮箱）
  registered_channels: List<String>,  // 注册渠道列表
  first_register_at: Timestamp?,      // 首次注册时间

  // ── 会员属性（来自 BTC 交易系统） ──
  membership_level: String?,   // 会员等级
  membership_score: Integer,   // 会员积分

  // ── 交易统计（聚合统计值，非模型计算） ──
  lifetime_order_count: Integer,   // 累计订单数
  lifetime_order_amount: Decimal,  // 累计消费金额
  last_order_at: Timestamp?,       // 最近下单时间

  // ── 融合元数据 ──
  info_version: Integer,           // 版本号
  attribute_sources: Map<String, String>  // 每个字段的来源系统（溯源审计）
}
```

#### AIProfile（AI 属性 — 算法/模型计算得出）

```
AIProfile {
  // ── 生命周期（规则引擎 / 状态机计算） ──
  lifecycle_stage: Enum,       // "new" | "active" | "silent" | "churned"
  lifecycle_score: Float,      // 生命周期判定置信度

  // ── 偏好模型（协同过滤 / 行为序列建模） ──
  purchase_preference: List<PreferenceTag>,   // 购买偏好标签 + 权重
  category_affinity: Map<String, Float>,      // 品类亲和度（品类 → 0~1 分数）
  price_sensitivity: Float,                   // 价格敏感度 (0.0~1.0)
  brand_loyalty: Float,                       // 品牌忠诚度 (0.0~1.0)

  // ── 行为预测（机器学习模型） ──
  churn_probability: Float,    // 流失概率 (0.0~1.0)
  purchase_intent: Float,      // 近期购买意向 (0.0~1.0)
  predicted_lifetime_value: Decimal?,  // 预测生命周期价值 (LTV)

  // ── 分群标签（聚类 / 规则引擎） ──
  customer_segment: Enum,      // 客户分群: "high_value" | "growth" | "at_risk" | "low_engagement"
  activity_preference: List<String>,  // 活跃渠道偏好

  // ── 模型元数据 ──
  model_version: String,       // 模型版本号（如 "churn_model_v2.3"）
  computed_at: Timestamp,      // 最近一次计算时间
  next_recompute_at: Timestamp? // 下次重算时间（调度用）
}
```

#### PreferenceTag（偏好标签）

```
PreferenceTag {
  tag_name: String,            // 标签名（如 "运动鞋"、"护肤品"）
  score: Float,                // 偏好强度 (0.0~1.0)
  source: Enum,                // 计算来源: "collaborative_filtering" | "behavior_sequence" | "rule_based"
  decay_factor: Float          // 时间衰减因子（越久远的行为权重越低）
}
```

#### 跨层映射表（独立存储）

```
CrossLayerMapping {
  mapping_id: String,          // 映射记录唯一 ID
  
  // ── 映射关系 ──
  brand_one_id: String?,       // 品牌层 OneID
  group_one_id: String?,       // 集团层 OneID
  global_one_id: String?,      // 全局层 OneID
  mapping_type: Enum,          // "brand_to_group" | "brand_to_global" | "group_to_global"
  
  // ── 传递链路（新增）──
  propagation_path: List<String>,  // 传递路径，如 ["GL:0000:001", "GR:2001:001", "BR:1001:001"]
  propagation_direction: Enum,     // 传递方向: "downstream" (向下) | "upstream" (向上)
  propagation_status: Enum,        // 传递状态: "pending" | "propagated" | "failed"
  last_propagated_at: Timestamp?,  // 最近一次传递时间
  
  // ── 合并建议（新增）──
  merge_recommendation: Enum,      // 合并建议: "merge" | "split" | "keep" | "review"
  merge_reason: String?,           // 合并原因说明
  merge_confidence: Float?,        // 合并建议置信度
  
  // ── 置信度 ──
  confidence: Float,           // 映射置信度（基于 ID 标识交集的重叠度）
  shared_id_count: Integer,    // 共享 ID 标识数量
  shared_id_types: List<Enum>, // 共享的 ID 标识类型（如 ["phone", "email"]）
  
  // ── 生命周期 ──
  created_at: Timestamp,
  updated_at: Timestamp,
  expired_at: Timestamp?       // 过期时间（长期未更新的映射可标记过期）
}
```

#### OneID 编码规则

```
格式: {scope_prefix}:{scope_id_short}:{sequence}
示例:
  BR:1001:000000001     → 品牌层, 品牌1001, 序号1
  GR:2001:000000001     → 集团层, 集团2001, 序号1
  GL:0000:000000001     → 全局层, 序号1

编码要求:
  - 固定长度，便于索引和排序
  - scope_prefix 区分层级，避免 ID 空间冲突
  - sequence 使用零填充的递增序号
```

### 关键特征

1. **唯一性**：在同一 scope 内，一个 OneID 对应一个自然人
2. **独立性**：三层 OneID 各自独立计算，无层级间数据依赖
3. **可映射性**：通过跨层映射表（CrossLayerMapping）支持层级间的 ID 关联查询
4. **动态演化**：支持增量更新，用户身份关系会随新数据不断校准（融合/分裂）
5. **实时+离线双链路**：实时链路秒级生成临时 OneID，离线链路次日校准融合，保证最终一致性

---

# OneID 生成标准流程 — 详细说明

## 概述

OneID 的生成是一个从**原始 ID 标识数据**到**统一身份实体**的流水线过程。三层（品牌/集团/全局）各自独立执行以下完整流程，数据范围不同但处理逻辑一致。

整体流程分为 7 个阶段：


| 阶段  | 名称           | 核心产出           | 执行频率  |
| --- | ------------ | -------------- | ----- |
| 1   | 数据采集与标准化清洗   | 标准化点表 + 边表     | 每日/实时 |
| 2   | 图数据加载与预处理    | 已缓存的干净图        | 每日批处理 |
| 3   | WCC 算法执行     | 候选 OneID（连通分量） | 每日批处理 |
| 4   | ID 映射决策与冲突消解 | 确定的 OneID 映射表  | 每日批处理 |
| 5   | 属性融合与画像沉淀    | OneID 宽表（含画像）  | 每日批处理 |
| 6   | 服务发布与缓存预热    | 在线服务就绪         | 每日批处理 |
| 7   | 跨层映射表计算与向下传递 | 跨层映射表 + 合并建议   | 每日批处理 |


---

## 阶段 1：数据采集与标准化清洗

### 目标

将来自 各业务渠道的原始 ID 标识数据，清洗为统一的**点表（Vertex Table）和边表（Edge Table）**，作为图计算的输入。

### 输入

- BTC 交易系统提供的用户 ID 标识数据：手机号、邮箱、设备 ID、OpenID、UnionID、Cookie ID、第三方 ID 等
- 各业务渠道的登录/注册/行为日志

### 处理步骤

#### 1.1 归一化（Normalization）

对所有 ID 标识值执行标准化处理，确保同一标识在不同来源下能被正确匹配：


| ID 类型     | 归一化规则                                 |
| --------- | ------------------------------------- |
| 手机号       | 去除国际区号前缀（如 +86）、去除空格和横杠、统一为纯数字格式      |
| 邮箱        | 转为小写、去除首尾空格、去除子地址标记（如 Gmail 的 `+tag`） |
| 设备 ID     | 转为小写、去除横杠（如 IDFV/IDFA 统一格式）           |
| OpenID    | 保持原值（已由微信平台保证唯一性）                     |
| UnionID   | 保持原值                                  |
| Cookie ID | 转为小写                                  |


#### 1.2 脏数据过滤

识别并排除无效、欺诈或测试数据，避免污染图计算结果：

- **测试数据**：标记为测试的账号、明显测试手机号（如 13800000000）
- **爬虫/机器人**：基于行为频率和模式识别的爬虫账号
- **异常 ID**：格式不合法的手机号/邮箱、全零设备 ID 等
- **公共 ID**：公共设备上的共享 Cookie ID、公共 WiFi 的设备 ID（高度数节点的前兆）

#### 1.3 构建点表与边表

**点表（Vertex Table）**：每个唯一的 ID 标识值作为一个顶点

```
Vertex {
  vertex_id: String,       // 唯一标识 = id_type + ":" + id_value（直接拼接，避免 hash 碰撞）
                            // 示例: "phone:13812345678"、"email:user@example.com"
  id_type: Enum,           // 标识类型
  id_value: String,        // 归一化后的标识值
  source_system: String,   // 来源系统
  first_seen_at: Timestamp,
  last_seen_at: Timestamp,
  observation_count: Long  // 观测次数（用于权重计算）
}
```

**边表（Edge Table）**：同一用户在同一个业务会话/设备/渠道中共同出现的 ID 标识之间建立边

```
Edge {
  src: String,             // 源顶点 vertex_id
  dst: String,             // 目标顶点 vertex_id
  weight: Float,           // 边权重（基于 ID 类型优先级和共现频率）
  co_occurrence_count: Long, // 共现次数
  first_co_at: Timestamp,  // 首次共现时间
  last_co_at: Timestamp,   // 最近共现时间
  source_system: String    // 共现来源系统
}
```

**边权重计算规则**：

权重反映两个 ID 标识属于同一自然人的可信程度。权重越高，WCC 连通时越倾向于合并。


| 关联类型                  | 基础权重 | 说明                   |
| --------------------- | ---- | -------------------- |
| 手机号 ↔ 邮箱              | 0.9  | 强关联：同一账号绑定的手机号和邮箱    |
| 手机号 ↔ 设备 ID           | 0.8  | 强关联：登录态下的设备          |
| 手机号 ↔ OpenID          | 0.85 | 强关联：微信绑定手机号          |
| UnionID ↔ OpenID      | 0.95 | 极强关联：同一微信用户在不同小程序的标识 |
| 设备 ID ↔ Cookie ID     | 0.6  | 中等关联：未登录态下的设备-浏览器关联  |
| Cookie ID ↔ Cookie ID | 0.4  | 弱关联：同浏览器不同会话         |


权重还会结合**时间衰减因子**进行调节：

```
final_weight = base_weight × time_decay_factor
time_decay_factor = exp(-λ × days_since_last_co_occurrence)
```

其中 λ 为衰减系数（推荐值 0.01），使得久远的共现关系权重逐渐降低。

### 输出

- 标准化点表（Parquet 格式，分区存储）
- 标准化边表（Parquet 格式，分区存储）

### 三层差异


| 层级  | 数据范围                  |
| --- | --------------------- |
| 品牌层 | 仅该品牌下的 BTC 数据和行为日志    |
| 集团层 | 该集团下所有品牌的 BTC 数据和行为日志 |
| 全局层 | 所有集团的全部 BTC 数据和行为日志   |


---

## 阶段 2：图数据加载与预处理（GraphFrames）

### 目标

将阶段 1 产出的点表和边表加载为 GraphFrames 图对象，并执行预处理操作以优化后续 WCC 算法的性能和正确性。

### 输入

- 标准化点表 + 边表

### 处理步骤

#### 2.1 构建 GraphFrame

```scala
import org.graphframes.GraphFrame

val vertices = spark.read.parquet("path/to/vertices")
val edges = spark.read.parquet("path/to/edges")

val g = GraphFrame(vertices, edges)
```

此时 GraphFrame 是惰性求值的，尚未触发实际计算。

#### 2.2 超级节点剪枝

超级节点（Super Node）是指度数（degree）异常高的顶点，通常对应公共设备 ID、公共 Cookie、公共 WiFi 出口 IP 等。如果不剪枝，这些节点会将大量不相关的用户错误地连通到同一个分量中，严重破坏 OneID 的准确性。

```scala
// 计算每个顶点的度数
val degreeThreshold = 1000  // 可配置

val vertexDegrees = g.degrees  // (id, degree)

// 识别超级节点
val superNodes = vertexDegrees.filter($"degree" > degreeThreshold)

// 从边表中移除与超级节点关联的边
val cleanEdges = g.edges
  .join(superNodes, $"src" === superNodes("id"), "left_anti")
  .join(superNodes, $"dst" === superNodes("id"), "left_anti")

// 重建图
val cleanGraph = GraphFrame(g.vertices, cleanEdges)
```

**剪枝策略**：

- 硬剪枝：度数超过阈值的节点直接移除其所有边
- 软剪枝（可选）：保留超级节点本身，但降低其关联边的权重至接近 0
- 被剪枝的超级节点单独记录，供风控模块使用（可能是羊毛党/欺诈信号）

#### 2.3 显式缓存

GraphFrames 的 WCC 算法会多次遍历顶点和边。如果不显式缓存，每次 Action 都会重新从磁盘读取数据，导致大量重复 I/O。

```scala
cleanGraph.vertices.cache()
cleanGraph.edges.cache()
```

#### 2.4 设置 Checkpoint 目录

WCC 算法是迭代算法，每轮迭代都会在上轮结果上追加 transform 操作，导致 Spark 的血缘链（Lineage）越来越长，最终触发 StackOverflow。通过设置 Checkpoint 目录，每轮迭代将中间结果持久化到 HDFS，截断血缘链。

```scala
spark.sparkContext.setCheckpointDir("hdfs:///checkpoint/wcc")
```

### 输出

- 预处理完成的 GraphFrame 对象（已缓存、已剪枝、已设置 checkpoint）

---

## 阶段 3：WCC 算法执行

### 目标

执行弱连通分量（Weakly Connected Components）算法，将图中相互连通的顶点聚合为同一个连通分量。每个连通分量即为一个**候选 OneID**。

### 输入

- 预处理完成的 GraphFrame

### 处理步骤

#### 3.1 执行 WCC 算法

```scala
val result = cleanGraph.connectedComponents.setCheckpointInterval(5)
val components = result.run()
```

- `connectedComponents()` 会触发多轮迭代 Job
- 每轮迭代中，每个顶点将自己的 Component ID（初始为自身 vertex_id）传播给邻居
- 邻居收到更小的 Component ID 后更新自身值，继续传播
- 直到没有顶点的 Component ID 发生变化，算法收敛

#### 3.2 Checkpoint 与断点恢复

```scala
.setCheckpointInterval(5)  // 每 5 轮写一次 Checkpoint
```

- 每 N 轮将当前所有顶点的 Component ID 写入 HDFS Checkpoint
- 如果某轮迭代失败（如集群故障），可从最近的 Checkpoint 恢复，无需从头开始
- Checkpoint 同时截断血缘链，避免 DAG 过长

#### 3.3 产出候选 OneID

算法收敛后，每个顶点被分配一个 Component ID。同一个 Component ID 下的所有顶点属于同一个连通分量，即候选同一个 OneID。

```scala
// 按 Component ID 聚合，查看每个候选 OneID 包含的 ID 标识
val candidateOneIDs = components
  .groupBy("component")
  .agg(
    collect_list("vertex_id").as("member_ids"),
    count("*").as("member_count")
  )
```

### 收敛性保证

- WCC 算法在有限图上必然收敛（每轮至少有一个顶点的 Component ID 变小或不变）
- 时间复杂度：O(E × k)，其中 E 为边数，k 为图的直径（最长最短路径）
- 实际电商场景中，图的直径通常很小（小世界网络特性），收敛速度快

### 输出

- 每个顶点的 Component ID 映射表
- 候选 OneID 列表及其成员 ID 标识

### 三层差异

三层各自独立执行 WCC，由于数据范围不同（品牌 < 集团 < 全局），同一个自然人在不同层级可能被分配到不同的 Component ID：

- **品牌层**：同一品牌内的 ID 标识连通 → 品牌 OneID
- **集团层**：跨品牌的 ID 标识连通（通过手机号/邮箱/UnionID 等跨品牌标识） → 集团 OneID
- **全局层**：跨集团的 ID 标识连通 → 全局 OneID

---

## 阶段 4：ID 映射决策与冲突消解

### 目标

将阶段 3 产出的候选 OneID 与历史 OneID 库进行增量比对，决定每个候选 OneID 是**新建**、**融合**到已有 OneID、还是**分裂**已有 OneID。确保 OneID 的**稳定性**（不频繁变化）和**准确性**（一对一映射）。

### 输入

- 本轮候选 OneID（Component ID + 成员列表）
- 历史 OneID 库（上一轮产出的正式 OneID）

### 处理步骤

#### 4.1 JOIN 历史 OneID 库 — 增量比对

将本轮候选 OneID 与历史 OneID 进行 JOIN，基于 ID 标识的交集计算相似度：

```
对于每个本轮候选 OneID C_new：
  找到历史 OneID 库中与其 ID 标识交集最大的历史 OneID H_old
  计算重叠度 = |C_new ∩ H_old| / |C_new ∪ H_old|  (Jaccard 相似度)
```

#### 4.2 融合/分裂判定

基于重叠度和其他因素，做出以下决策：


| 场景       | 判定条件                            | 决策                                       |
| -------- | ------------------------------- | ---------------------------------------- |
| **新增**   | C_new 与所有历史 OneID 的重叠度均低于阈值     | 分配新的 OneID                               |
| **保持不变** | C_new 与某个 H_old 的重叠度高于阈值（如 0.8） | 保持 H_old 的 OneID 不变，更新成员列表               |
| **融合**   | C_new 与多个历史 OneID 均有较高重叠度       | 将多个历史 OneID 合并为一个，原 OneID 标记为 `merged`   |
| **分裂**   | 某个 H_old 的 ID 标识在本轮被分散到多个候选中    | 将 H_old 拆分为多个新 OneID，原 OneID 标记为 `split` |


**融合决策的附加考量**：

- **权重因素**：高权重 ID 标识（如手机号）的匹配比低权重 ID（如 Cookie ID）更有说服力
- **时效因素**：最近共现的 ID 关联比久远的关联更可信
- **优先级因素**：主 ID（`is_primary = true`）的变更需要更严格的判定条件

#### 4.3 分配新 OneID / 拆分旧 ID

- **新增**：按照编码规则生成新 OneID（`{scope_prefix}:{scope_id_short}:{sequence}`）
- **融合**：保留优先级最高的历史 OneID，其余标记为 `merged`，`merged_into` 指向保留的 OneID
- **分裂**：为每个子分量分配新 OneID，原 OneID 标记为 `split`，`split_from` 记录来源

#### 4.4 强制一对一兜底

在任何情况下，都必须保证**一个自然人对应一个 OneID**（在同一 scope 内）。如果出现一个 ID 标识同时出现在多个 OneID 中的情况（跨分量冲突），按以下优先级消解：

1. 高权重 ID 标识所在的 OneID 优先保留该标识
2. 同等权重下，最近活跃的 OneID 优先
3. 最终确保每个 ID 标识只属于一个 OneID

### 输出

- 正式 OneID 映射表（OneID → ID 标识列表）
- 变更记录（新增/融合/分裂的操作日志，用于审计和回溯）

---

## 阶段 5：属性融合与画像沉淀

### 目标

为每个确定的 OneID 聚合来自多源的用户属性，构建完整的用户画像（PhysicalInfo + AIProfile）。

### 输入

- 正式 OneID 映射表
- 交易系统的用户属性数据
- 行为日志数据

### 处理步骤

#### 5.1 多源属性对齐

同一个 OneID 关联的多个 ID 标识可能来自不同业务系统，各系统提供的用户属性可能存在冲突。对齐规则：


| 策略       | 适用字段                                        | 规则                          |
| -------- | ------------------------------------------- | --------------------------- |
| **高优覆盖** | nickname, avatar_url, gender                | 按来源系统优先级取值（如微信 > APP > Web） |
| **多值聚合** | registered_channels, activity_preference    | 合并所有来源的值，去重                 |
| **时间衰减** | phone, email, membership_level              | 取最近更新的值，但保留历史值用于审计          |
| **数值聚合** | lifetime_order_count, lifetime_order_amount | 跨品牌/渠道求和                    |
| **最早取值** | first_register_at, birthday                 | 取所有来源中最早/最可靠的值              |


#### 5.2 标签体系挂载

基于聚合后的属性和行为数据，计算 AI 标签：

**规则引擎标签**（确定性规则）：

- `lifecycle_stage`：基于最近活跃时间 + 订单频率的状态机
  - 注册 ≤ 30 天且无订单 → `new`
  - 最近 30 天有活跃/订单 → `active`
  - 30~90 天无活跃 → `silent`
  - > 90 天无活跃 → `churned`

**模型计算标签**（机器学习）：

- `churn_probability`：基于行为序列的流失预测模型
- `purchase_intent`：基于浏览/加购/收藏行为的购买意向模型
- `predicted_lifetime_value`：基于历史消费模式的 LTV 预测模型
- `purchase_preference` / `category_affinity`：基于协同过滤的偏好模型
- `customer_segment`：基于聚类的客户分群

#### 5.3 质量校验门禁

在输出最终 OneID 宽表之前，执行质量校验：

**宏观指标监控**：

- OneID 总量变化率（日环比不超过 ±5%，异常则告警）
- 平均每个 OneID 关联的 ID 标识数量（应在合理范围内）
- 连通分量大小分布（超大分量可能意味着剪枝不足）
- 融合/分裂操作数量（异常激增可能意味着数据源变更）

**微观抽样审计**：

- 随机抽样 N 个 OneID，人工核验 ID 标识关联的合理性
- 检查是否存在明显的错误合并（如不同性别的手机号被合并）

### 输出

- OneID 宽表（包含 OneID 基础信息 + PhysicalInfo + AIProfile）
- 质量校验报告

---

## 阶段 6：服务发布与缓存预热

### 目标

将阶段 5 产出的 OneID 宽表发布到各类存储系统，支撑上层业务的实时查询和分析需求。

### 输入

- 通过质量校验的 OneID 宽表
- 跨层映射表（CrossLayerMapping）

### 处理步骤

#### 6.1 多存储引擎写入


| 存储系统                | 用途          | 数据格式                 | 查询模式                            |
| ------------------- | ----------- | -------------------- | ------------------------------- |
| **Redis**           | 实时点查（毫秒级）   | Hash: OneID → 用户画像摘要 | `GET oneid:{id}`                |
| **HBase**           | 大规模点查（十毫秒级） | 行键: OneID，列族: 画像字段   | `GET 'oneid_table', '{one_id}'` |
| **ClickHouse**      | OLAP 分析查询   | 宽表，按 scope/品牌/集团分区   | SQL 聚合查询                        |
| **Hive/MaxCompute** | 离线数据挖掘      | ORC/Parquet 格式宽表     | 离线批处理 SQL                       |


#### 6.2 热点缓存预热

在业务低峰期（如凌晨 3:00~5:00），提前将高频访问的 OneID 数据加载到 Redis 缓存中：

- 高价值用户（`customer_segment = "high_value"`）的 OneID
- 近 7 天活跃用户的 OneID
- 集团级 VIP 用户的 OneID

#### 6.3 版本快照与回滚机制

- 每次批处理完成后，对 OneID 全量数据创建版本快照
- 保留最近 N 天（如 30 天）的历史快照
- 如果新版本数据出现质量问题，可快速回滚到上一个快照
- 快照存储于 HDFS/S3，采用增量存储以节省空间

#### 6.4 跨层映射表生成与更新

在各层 OneID 均计算完成后，通过 ID 标识的交集自动生成/更新跨层映射表（详细逻辑见阶段 7）。

### 输出

- 各存储系统中的 OneID 数据已更新
- Redis 热点缓存已预热
- 跨层映射表已更新
- 版本快照已创建

---

## 阶段 7：跨层映射表计算与向下传递

### 目标

在各层 OneID 独立计算完成后，计算跨层映射关系。跨层映射表本身就是"向下传递"的载体——通过查询映射表，可以回答"在更大数据范围下，哪些低层 OneID 其实是同一个人"，为集团层和品牌层的 OneID 合并提供决策依据。

### 核心概念

**"向下传递"的本质**：不是一个新的计算步骤，而是跨层映射表的一个查询视角。

- 给定一个全局 OneID，查到它对应的所有集团 OneID → 如果 > 1 个，说明这些集团 OneID 在全局视角下是同一个人
- 给定一个集团 OneID，查到它对应的所有品牌 OneID → 如果 > 1 个，说明这些品牌 OneID 在集团视角下是同一个人

### 输入

- 品牌层 OneID 映射表（阶段 4 产出）
- 集团层 OneID 映射表（阶段 4 产出）
- 全局层 OneID 映射表（阶段 4 产出）
- 各层 OneID 的 ID 标识列表（id_mappings）

### 处理步骤

#### 7.1 计算跨层映射关系

通过 ID 标识的交集，计算三层之间的映射关系：

```
// 品牌层 → 集团层映射
对于每个品牌层 OneID B：
  对于该品牌所属集团下的每个集团层 OneID G：
    计算 shared_ids = B.id_mappings ∩ G.id_mappings  (基于 id_value 匹配)
    如果 shared_ids 非空:
      confidence = |shared_ids| / |B.id_mappings ∪ G.id_mappings|
      shared_id_types = shared_ids.map(id => id.id_type).distinct()
      写入 CrossLayerMapping(
        brand_one_id = B.one_id,
        group_one_id = G.one_id,
        mapping_type = "brand_to_group",
        confidence = confidence,
        shared_id_count = |shared_ids|,
        shared_id_types = shared_id_types
      )

// 品牌层 → 全局层映射
对于每个品牌层 OneID B：
  对于每个全局层 OneID GL：
    计算 shared_ids = B.id_mappings ∩ GL.id_mappings
    如果 shared_ids 非空:
      confidence = |shared_ids| / |B.id_mappings ∪ GL.id_mappings|
      shared_id_types = shared_ids.map(id => id.id_type).distinct()
      写入 CrossLayerMapping(
        brand_one_id = B.one_id,
        global_one_id = GL.one_id,
        mapping_type = "brand_to_global",
        confidence = confidence,
        shared_id_count = |shared_ids|,
        shared_id_types = shared_id_types
      )

// 集团层 → 全局层映射
对于每个集团层 OneID G：
  对于每个全局层 OneID GL：
    计算 shared_ids = G.id_mappings ∩ GL.id_mappings
    如果 shared_ids 非空:
      confidence = |shared_ids| / |G.id_mappings ∪ GL.id_mappings|
      shared_id_types = shared_ids.map(id => id.id_type).distinct()
      写入 CrossLayerMapping(
        group_one_id = G.one_id,
        global_one_id = GL.one_id,
        mapping_type = "group_to_global",
        confidence = confidence,
        shared_id_count = |shared_ids|,
        shared_id_types = shared_id_types
      )
```

#### 7.2 向下查询：发现"更大数据范围下的同一个人"

跨层映射表建立后，通过向下查询，可以发现低层 OneID 在更大数据范围下其实是同一个人。

**场景示例**：

```
集团 G1 下有品牌 A，集团 G2 下有品牌 C

品牌 A 的用户: {phone:138, email:a@brandA.com}
品牌 C 的用户: {phone:139, email:a@brandA.com}

三层独立计算后：
  集团 G1 层: GR:G1:001 = {phone:138, email:a@brandA.com}
  集团 G2 层: GR:G2:001 = {phone:139, email:a@brandA.com}
    （两个集团各自计算，认为这是两个不同的人）
  
  全局层: 因为 email:a@brandA.com 是共享 ID，全局层把两个集团连通了
    GL:001 = {phone:138, phone:139, email:a@brandA.com}

跨层映射：
  GR:G1:001 → GL:001  (共享 email:a@brandA.com)
  GR:G2:001 → GL:001  (共享 email:a@brandA.com)

向下查询：
  查询 GL:001 对应的所有集团 OneID → [GR:G1:001, GR:G2:001]
  发现：GR:G1:001 和 GR:G2:001 在全局视角下是同一个人
  合并建议：GR:G1:001 和 GR:G2:001 应该合并
```

**查询逻辑**：

```
// 全局层 → 集团层：发现跨集团的同一个人
对于每个全局层 OneID GL：
  查询跨层映射表，找到所有映射到 GL 的集团层 OneID 列表：[G1, G2, ..., Gn]
  
  如果 n > 1：
    // 多个集团层 OneID 映射到同一个全局层 OneID
    // 说明这些集团 OneID 在全局视角下是同一个自然人
    生成合并建议：
      target_global_one_id = GL.one_id
      source_group_one_ids = [G1.one_id, G2.one_id, ..., Gn.one_id]
      merge_recommendation = "merge"
      merge_reason = "全局层 OneID {GL.one_id} 包含多个集团层 OneID，它们在全局视角下是同一个自然人"
      merge_confidence = 各映射记录的 confidence 平均值

// 集团层 → 品牌层：发现跨品牌的同一个人
对于每个集团层 OneID G：
  查询跨层映射表，找到所有映射到 G 的品牌层 OneID 列表：[B1, B2, ..., Bm]
  
  如果 m > 1：
    // 多个品牌层 OneID 映射到同一个集团层 OneID
    // 说明这些品牌 OneID 在集团视角下是同一个自然人
    生成合并建议：
      target_group_one_id = G.one_id
      source_brand_one_ids = [B1.one_id, B2.one_id, ..., Bm.one_id]
      merge_recommendation = "merge"
      merge_reason = "集团层 OneID {G.one_id} 包含多个品牌层 OneID，它们在集团视角下是同一个自然人"
      merge_confidence = 各映射记录的 confidence 平均值
```

#### 7.3 冲突检测

跨层映射表还可以用于检测异常情况：

```
冲突 1：一个品牌层 OneID 映射到多个集团层 OneID
  说明：该品牌层 OneID 在集团层被分裂为多个实体
  处理：标记为待审核，可能需要人工判断是否正确

冲突 2：一个集团层 OneID 映射到多个全局层 OneID
  说明：该集团层 OneID 在全局层被分裂为多个实体
  处理：标记为待审核

冲突 3：映射置信度低于阈值（如 < 0.5）
  说明：ID 标识交集较小，映射关系可能不可靠
  处理：标记为待审核
```

#### 7.4 合并决策执行

基于合并建议，可以自动执行合并或标记为待人工审核：

```
对于每个合并建议：
  如果 merge_confidence >= 0.8：
    自动执行合并
    更新 OneID 状态：原 OneID 标记为 merged，merged_into 指向目标 OneID
  否则：
    标记为待人工审核
    写入审核队列
```

### 输出

- 跨层映射表（CrossLayerMapping）已更新
- 合并建议已生成（merge_recommendation）
- 冲突检测结果（待审核列表）
- 合并执行结果（自动合并 or 待人工审核）

### 三层差异


| 层级  | 映射计算范围            | 向下查询视角                    |
| --- | ----------------- | ------------------------- |
| 品牌层 | 计算与所属集团层、全局层的映射关系 | 查询集团层 OneID 对应的所有品牌 OneID |
| 集团层 | 计算与所属全局层的映射关系     | 查询全局层 OneID 对应的所有集团 OneID |
| 全局层 | 计算与所有集团层的映射关系     | 无（全局层是最高层）                |


---

## 实时链路补充

### 目标

在离线批处理（T+1）的间隙，为新用户或新行为提供秒级的身份识别能力。

### 处理流程

```
用户事件（注册/登录/浏览/下单）
       │
       ▼
┌──────────────────────────────────┐
│  Flink 实时处理引擎               │
│                                  │
│  1. 提取事件中的 ID 标识          │
│  2. 查询 Redis 缓存的 OneID 映射  │
│  3. 如果匹配到已有 OneID → 复用    │
│  4. 如果未匹配 → 生成临时 OneID    │
│     (格式: TEMP:{uuid})          │
│  5. 写入 Redis（TTL = 24h）       │
└──────────────┬───────────────────┘
               │
               ▼
        临时 OneID 生效（秒级）
               │
               │  次日离线批处理时
               ▼
┌──────────────────────────────────┐
│  离线 WCC 校准融合                │
│                                  │
│  1. 将临时 OneID 的 ID 标识       │
│     纳入阶段 1 的数据采集          │
│  2. WCC 重新计算连通分量           │
│  3. 阶段 4 中临时 OneID 被        │
│     融合到正式 OneID 或分配新 ID    │
│  4. 更新 Redis，替换临时 OneID     │
└──────────────────────────────────┘
```

### 最终一致性保证

- 实时链路保证**秒级可用**：新用户首次访问即可获得临时 OneID
- 离线链路保证**最终准确**：次日 WCC 校准后，临时 OneID 被替换为正式 OneID
- 过渡期内，通过 Redis 中的映射关系（临时 OneID → 正式 OneID）保证查询的连续性

---

## 三层并行执行说明

品牌层、集团层、全局层各自独立执行阶段 1~6，互不依赖。阶段 7 在所有层的阶段 6 完成后执行：

```
                    ┌─────────────────────┐
                    │   原始数据源      │
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │  品牌层流程   │  │  集团层流程   │  │  全局层流程   │
    │             │  │             │  │             │
    │ 数据范围:    │  │ 数据范围:    │  │ 数据范围:    │
    │ 单品牌       │  │ 集团下全品牌  │  │ 全集团       │
    │             │  │             │  │             │
    │ 阶段1~6     │  │ 阶段1~6     │  │ 阶段1~6     │
    │ 独立执行     │  │ 独立执行     │  │ 独立执行     │
    └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
           │                │                │
           ▼                ▼                ▼
     品牌 OneID        集团 OneID        全局 OneID
           │                │                │
           └────────────────┼────────────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │ 阶段7: 跨层映射表   │
                  │ 计算与向下传递      │
                  │                   │
                  │ • 计算跨层映射关系   │
                  │ • 向下传递 OneID   │
                  │ • 生成合并建议     │
                  │ • 冲突检测        │
                  └───────────────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │ 跨层映射表         │
                  │ CrossLayerMapping │
                  │ + 合并建议         │
                  └───────────────────┘
```

**关键约束**：

- 阶段 1~6 可以完全并行计算，无数据依赖
- 阶段 7 必须在所有层的阶段 6 完成后执行（依赖各层 OneID 结果）
- 各层对"同一个人"的判定可能不同（因为数据范围不同，连通分量结果不同），这是预期行为
- 跨层映射表通过向下传递机制，为集团层和品牌层的 OneID 合并提供决策依据

---

## 需求完成状态


| 需求                                 | 状态    | 实现位置                 |
| ---------------------------------- | ----- | -------------------- |
| 支持向下传递 OneID 关系，作为集团/品牌 OneID 合并依据 | ✅ 已完成 | 阶段 7：跨层映射表计算与向下传递    |
| 支持干预 OneID 合并拆分                    | ⏳ 待设计 | 待补充：人工干预机制、合并/拆分审核流程 |


