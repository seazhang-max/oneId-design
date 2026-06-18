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
  status: Enum,                // 状态: "active" | "merged" | "split"
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

# OneID 生成流程

## 概述

OneID 的生成是一个从**原始 ID 标识数据**到**统一身份实体**的流水线过程。三层（品牌/集团/全局）各自独立执行以下完整流程，数据范围不同但处理逻辑一致。

整体流程分为 7 个阶段：


| 阶段  | 名称           | 核心产出           | 执行频率  | 技术                |
| --- | ------------ | -------------- | ----- | ----------------- |
| 1   | 数据采集与标准化清洗   | 标准化点表 + 边表     | 每日/实时 | spark             |
| 2   | 图数据加载与预处理    | 已缓存的干净图        | 每日批处理 | spark             |
| 3   | WCC 算法执行     | 候选 OneID（连通分量） | 每日批处理 | spark graphframes |
| 4   | ID 映射决策与冲突消解 | 确定的 OneID 映射表  | 每日批处理 | spark             |
| 5   | 属性融合与画像沉淀    | OneID 宽表（含画像）  | 每日批处理 | spark             |
| 6   | 服务发布与缓存预热    | 在线服务就绪         | 每日批处理 | go                |
| 7   | 跨层映射表计算与向下传递 | 跨层映射表 + 合并建议   | 每日批处理 | spark             |


---

## 阶段 1：数据采集与标准化清洗

### 目标

将来自 各业务渠道的原始 ID 标识数据，清洗为统一的**点表（Vertex Table）和边表（Edge Table）**，作为图计算的输入。

### 输入

- 业务系统提供的用户 ID 标识数据：手机号、邮箱、设备 ID、OpenID、UnionID、Cookie ID、第三方 ID 等
- 各业务渠道的登录/注册/下单/行为日志

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

**边置信度计算规则**：

置信度反映两个 ID 标识属于同一自然人的可信程度。置信度越高，WCC 连通时越倾向于合并。


| 关联类型                  | 置信度  | 说明                   |
| --------------------- | ---- | -------------------- |
| 手机号 ↔ 邮箱              | 0.9  | 强关联：同一账号绑定的手机号和邮箱    |
| 手机号 ↔ 设备 ID           | 0.8  | 强关联：登录态下的设备          |
| 手机号 ↔ OpenID          | 0.85 | 强关联：微信绑定手机号          |
| UnionID ↔ OpenID      | 0.95 | 极强关联：同一微信用户在不同小程序的标识 |
| 设备 ID ↔ Cookie ID     | 0.6  | 中等关联：未登录态下的设备-浏览器关联  |
| Cookie ID ↔ Cookie ID | 0.4  | 弱关联：同浏览器不同会话         |


### 输出

- 标准化点表（Parquet 格式，分区存储）
- 标准化边表（Parquet 格式，分区存储）

### 三层差异


| 层级  | 数据范围              |
| --- | ----------------- |
| 品牌层 | 仅该品牌下的 数据和行为日志    |
| 集团层 | 该集团下所有品牌的 数据和行为日志 |
| 全局层 | 所有集团的全部 数据和行为日志   |


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

将本轮候选 OneID 与历史 OneID 进行 JOIN，基于顶点（ID 标识）是否有相同来判断关系：

```
对于每个本轮候选 OneID C_new：
  找到历史 OneID 库中与 C_new 有相同顶点的历史 OneID H_old
  （即 C_new 和 H_old 的 id_mappings 中存在相同的 id_value）
```

#### 4.2 融合/分裂判定

基于 C_new 与历史 OneID 的顶点相同情况，做出以下决策：


| 场景       | 判定条件                                | 决策                                       |
| -------- | ----------------------------------- | ---------------------------------------- |
| **新增**   | C_new 和所有历史 OneID 都没有顶点相同           | 分配新的 OneID                               |
| **保持不变** | C_new 和某个 H_old 有顶点相同               | 保持 H_old 的 OneID 不变，更新成员列表               |
| **融合**   | C_new 和多个历史 OneID 都有顶点相同            | 将多个历史 OneID 合并为一个，原 OneID 标记为 `merged`   |
| **分裂**   | 某个 H_old 因点删除或边删除，导致其顶点在本轮被分散到多个候选中 | 将 H_old 拆分为多个新 OneID，原 OneID 标记为 `split` |


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

## 人工干预 OneID 合并拆分机制

### 概述

WCC 算法的自动合并/拆分基于图拓扑和权重规则，无法覆盖所有业务场景。人工干预机制为运营和数据团队提供对 OneID 合并与拆分的**手动控制能力**，包括强制合并、强制拆分、边权重调整、ID 解绑等操作，并通过审批工作流和 Dry Run 模拟确保操作的安全性和可追溯性。

**设计原则**：

- **干预优先**：人工干预操作的优先级高于自动计算结果，在阶段 4（ID 映射决策与冲突消解）中，干预规则作为硬约束参与决策
- **可审计**：所有干预操作记录完整的审计日志，支持回溯
- **可回滚**：每次干预操作产生版本快照，支持撤销
- **最小影响**：干预操作的影响范围通过 Dry Run 预先评估，避免意外波及

### 干预操作类型

#### 操作类型一览


| 操作类型      | 枚举值                  | 说明                               | 风险等级 | 审批要求        |
| --------- | -------------------- | -------------------------------- | ---- | ----------- |
| **强制合并**  | `force_merge`        | 将两个独立的 OneID 合并为一个               | 高    | 必须审批        |
| **强制拆分**  | `force_split`        | 将一个 OneID 中的部分 ID 标识剥离，生成新 OneID | 高    | 必须审批        |
| **ID 解绑** | `id_unbind`          | 从 OneID 中移除某个 ID 标识（不生成新 OneID）  | 中    | 单条免审批，批量需审批 |
| **边权重调整** | `edge_weight_adjust` | 调整两个 ID 标识之间的关联权重                | 中    | 免审批（影响范围有限） |
| **边删除**   | `edge_delete`        | 删除两个 ID 标识之间的关联边                 | 中    | 免审批         |
| **边新增**   | `edge_add`           | 在两个 ID 标识之间新增关联边                 | 中    | 免审批         |
| **冻结**    | `freeze`             | 冻结 OneID，阻止后续自动合并/拆分             | 低    | 免审批         |
| **解冻**    | `unfreeze`           | 解除 OneID 的冻结状态                   | 低    | 免审批         |


#### 操作数据结构

```
InterventionOperation {
  // ── 基础标识 ──
  operation_id: String,              // 操作唯一 ID
  operation_type: Enum,              // 操作类型（见上表）
  
  // ── 操作目标 ──
  scope: Enum,                       // 作用层级: "brand" | "group" | "global"
  scope_id: String,                  // 品牌ID / 集团ID / "global"
  target_one_ids: List<String>,      // 涉及的 OneID 列表
  target_id_mappings: List<IdMappingRef>,  // 涉及的 ID 标识（用于解绑、边操作）
  
  // ── 操作参数 ──
  params: InterventionParams,        // 操作参数（不同操作类型对应不同参数）
  
  // ── 审批流程 ──
  workflow: ApprovalWorkflow,        // 审批工作流状态
  
  // ── 执行状态 ──
  status: Enum,                      // "draft" | "pending_review" | "approved" | "rejected" | "executing" | "completed" | "failed" | "rolled_back"
  dry_run_result: DryRunResult?,     // Dry Run 模拟结果
  execution_result: ExecutionResult?, // 实际执行结果
  
  // ── 干预有效期 ──
  effective_mode: Enum,              // "permanent" | "temporary" | "until_next_full_rebuild"
  effective_from: Timestamp,
  effective_until: Timestamp?,       // 临时干预的过期时间
  
  // ── 审计 ──
  created_by: String,                // 操作发起人
  reviewed_by: List<String>,         // 审批人列表
  reason: String,                    // 操作原因说明
  related_ticket: String?,           // 关联的工单/Issue 编号
  
  // ── 生命周期 ──
  created_at: Timestamp,
  updated_at: Timestamp,
  executed_at: Timestamp?,
  rolled_back_at: Timestamp?
}
```

#### 操作参数（InterventionParams）

```
InterventionParams {
  // ── 强制合并参数 ──
  merge_target_one_id: String?,      // 合并目标 OneID（保留此 ID，另一个合并进来）
  merge_strategy: Enum?,             // "keep_first" | "keep_second" | "auto"（保留哪个 OneID 作为主 ID）
  
  // ── 强制拆分参数 ──
  split_id_values: List<String>?,    // 要从原 OneID 中剥离的 ID 标识值列表
  new_one_id_policy: Enum?,          // "auto_assign" | "specified"（自动生成新 OneID 或指定）
  
  // ── ID 解绑参数 ──
  unbind_id_type: Enum?,             // 要解绑的 ID 类型
  unbind_id_value: String?,          // 要解绑的 ID 值
  unbind_reason: Enum?,              // "wrong_binding" | "shared_device" | "family_account" | "other"
  
  // ── 边权重调整参数 ──
  edge_src: String?,                 // 边的源顶点 vertex_id
  edge_dst: String?,                 // 边的目标顶点 vertex_id
  new_weight: Float?,                // 新权重值
  weight_override_mode: Enum?,       // "absolute" | "multiply"（绝对值覆盖或乘以系数）
  
  // ── 边新增参数 ──
  edge_co_occurrence_source: String?, // 新增边的共现来源说明
  
  // ── 批量操作参数 ──
  batch_mode: Boolean?,              // 是否批量操作
  batch_file_url: String?,           // 批量操作文件地址（CSV/JSON）
  batch_max_count: Integer?          // 批量操作最大条数限制
}
```

#### IdMappingRef（ID 标识引用）

```
IdMappingRef {
  one_id: String,                    // 所属 OneID
  id_type: Enum,                     // ID 类型
  id_value: String                   // ID 值
}
```

### 审批工作流（Approval Workflow）

#### 工作流状态机

```
                    ┌──────────┐
                    │  draft   │ ← 创建草稿
                    └────┬─────┘
                         │ 提交
                         ▼
                  ┌──────────────┐
            ┌─────│ pending_review│
            │     └──────┬───────┘
            │            │
     ┌──────┴──────┐     │
     │             │     │
     ▼             ▼     │
┌──────────┐ ┌──────────┐│
│ rejected │ │ approved ││
└──────────┘ └────┬─────┘│
                  │      │
                  ▼      │
            ┌──────────┐ │
            │ executing │ │
            └────┬─────┘ │
                 │       │
          ┌──────┴──────┐│
          │             ││
          ▼             ▼│
    ┌───────────┐ ┌────────────┐
    │ completed │ │   failed   │
    └───────────┘ └──────┬─────┘
                         │ 回滚
                         ▼
                  ┌──────────────┐
                  │ rolled_back  │
                  └──────────────┘
```

#### 审批规则配置

```
ApprovalPolicy {
  // ── 基础配置 ──
  policy_id: String,
  scope: Enum,                       // 适用的层级
  scope_id: String,                  // 适用的品牌/集团
  
  // ── 审批规则 ──
  rules: List<ApprovalRule>,
  
  // ── 审批人配置 ──
  reviewer_roles: List<String>,      // 审批角色（如 "data_admin"、"biz_owner"）
  reviewer_count: Integer,           // 最少审批人数
  auto_approve_enabled: Boolean,     // 是否启用自动审批（低风险操作）
  auto_approve_threshold: Float?,    // 自动审批的置信度阈值
  
  // ── 超时策略 ──
  review_timeout_hours: Integer,     // 审批超时时间（小时）
  timeout_action: Enum               // "auto_reject" | "escalate"（超时自动拒绝或升级）
}

ApprovalRule {
  operation_type: Enum,              // 操作类型
  risk_level: Enum,                  // 风险等级: "low" | "medium" | "high" | "critical"
  require_approval: Boolean,         // 是否需要审批
  require_dry_run: Boolean,          // 是否必须先执行 Dry Run
  max_batch_size: Integer?,          // 单次批量操作最大条数
  cooldown_hours: Integer?,          // 同一目标 OneID 两次操作的最短间隔
  require_reason: Boolean            // 是否必须填写操作原因
}
```

**默认审批规则**：


| 操作类型                  | 风险等级   | 需要审批 | 需要 Dry Run | 最大批量 |
| --------------------- | ------ | ---- | ---------- | ---- |
| `force_merge`         | high   | ✅ 是  | ✅ 是        | 10   |
| `force_split`         | high   | ✅ 是  | ✅ 是        | 10   |
| `id_unbind`（单条）       | medium | ❌ 否  | ❌ 否        | 1    |
| `id_unbind`（批量）       | medium | ✅ 是  | ❌ 否        | 100  |
| `edge_weight_adjust`  | medium | ❌ 否  | ❌ 否        | 无限制  |
| `edge_delete`         | medium | ❌ 否  | ❌ 否        | 无限制  |
| `edge_add`            | medium | ❌ 否  | ❌ 否        | 无限制  |
| `freeze` / `unfreeze` | low    | ❌ 否  | ❌ 否        | 无限制  |


### Dry Run 模拟

#### 目标

在执行干预操作之前，模拟该操作对 OneID 体系的影响范围，帮助审批人和操作人评估风险。

#### Dry Run 流程

```
提交干预操作（status = "draft"）
       │
       ▼
┌──────────────────────────────────┐
│  Dry Run 引擎                     │
│                                  │
│  1. 加载当前 OneID 映射表快照      │
│  2. 在沙箱中模拟执行干预操作        │
│  3. 计算影响范围                  │
│  4. 生成 DryRunResult             │
└──────────────┬───────────────────┘
               │
               ▼
        DryRunResult 返回给操作人
        审批人基于结果决定是否批准
```

#### DryRunResult 数据结构

```
DryRunResult {
  // ── 直接影响 ──
  affected_one_ids: List<OneIDChange>,     // 直接受影响的 OneID 变更列表
  
  // ── 级联影响 ──
  cascade_effects: CascadeEffect,          // 级联影响（跨层映射、下游报表等）
  
  // ── 统计摘要 ──
  summary: DryRunSummary,
  
  // ── 风险评估 ──
  risk_assessment: RiskAssessment,
  
  // ── 模拟时间 ──
  simulated_at: Timestamp,
  simulation_duration_ms: Long
}

OneIDChange {
  one_id: String,                          // 受影响的 OneID
  change_type: Enum,                       // "merged" | "split" | "updated" | "frozen" | "unfrozen"
  before_snapshot: OneIDSnapshot,          // 变更前快照
  after_snapshot: OneIDSnapshot,           // 变更后快照
  diff_description: String                 // 可读的变更描述
}

OneIDSnapshot {
  one_id: String,
  id_mapping_count: Integer,               // ID 标识数量
  id_mappings_summary: List<String>,       // ID 标识摘要（类型+值）
  profile_summary: ProfileSummary,         // 画像摘要
  linked_one_ids: List<String>             // 关联的其他 OneID（跨层映射）
}

CascadeEffect {
  affected_cross_layer_mappings: Integer,  // 受影响的跨层映射记录数
  affected_downstream_reports: List<String>, // 受影响的下游报表/系统列表
  affected_user_count: Integer,            // 波及的用户数（合并后 OneID 变化的用户）
  profile_merge_conflicts: Integer,        // 属性融合冲突数
  estimated_rebuild_time_seconds: Integer  // 预估重建耗时
}

DryRunSummary {
  total_affected_one_ids: Integer,         // 受影响的 OneID 总数
  new_one_ids_created: Integer,            // 新建的 OneID 数
  merged_one_ids: Integer,                 // 被合并的 OneID 数
  split_one_ids: Integer,                  // 被拆分的 OneID 数
  updated_one_ids: Integer,                // 被更新的 OneID 数
  cross_layer_impact: Boolean              // 是否影响跨层映射
}

RiskAssessment {
  risk_level: Enum,                        // "low" | "medium" | "high" | "critical"
  risk_factors: List<RiskFactor>,          // 风险因子列表
  recommendations: List<String>            // 建议措施
}

RiskFactor {
  factor: String,                          // 风险因子描述
  severity: Enum,                          // "info" | "warning" | "critical"
  detail: String                           // 详细说明
}
```

**风险因子示例**：


| 风险因子      | 严重度      | 触发条件                                            |
| --------- | -------- | ----------------------------------------------- |
| 合并涉及高价值用户 | warning  | 被合并的 OneID 中有 `customer_segment = "high_value"` |
| 跨层映射受影响   | warning  | 干预操作导致跨层映射表需要更新                                 |
| 大批量操作     | critical | 单次操作影响 > 100 个 OneID                            |
| 属性融合冲突    | warning  | 合并的两个 OneID 存在矛盾属性（如不同性别）                       |
| 近期已有干预记录  | info     | 目标 OneID 在近 7 天内已有干预操作                          |


### 干预与自动计算的协调机制

#### 干预规则表（Intervention Rule Table）

人工干预操作生效后，会生成**干预规则**写入干预规则表。在后续离线批处理的阶段 4（ID 映射决策与冲突消解）中，干预规则作为硬约束参与决策。

```
InterventionRule {
  rule_id: String,                     // 规则唯一 ID
  rule_type: Enum,                     // 规则类型（见下）
  scope: Enum,                         // 作用层级
  scope_id: String,                    // 品牌ID / 集团ID / "global"
  
  // ── 规则内容 ──
  target_id_values: List<String>,      // 规则涉及的 ID 标识值
  constraint: Enum,                    // "must_merge" | "must_split" | "must_not_merge" | "weight_override" | "edge_removed" | "edge_added"
  constraint_params: Map<String, Any>, // 规则参数（如权重覆盖的新值）
  
  // ── 来源追溯 ──
  source_operation_id: String,         // 来源干预操作 ID
  created_by: String,                  // 操作人
  
  // ── 有效期 ──
  effective_mode: Enum,                // "permanent" | "temporary" | "until_next_full_rebuild"
  effective_from: Timestamp,
  effective_until: Timestamp?,
  is_active: Boolean                   // 是否当前生效
}
```

**干预规则类型与阶段 4 的集成方式**：


| 规则类型 | `constraint` 值    | 在阶段 4 中的作用                          |
| ---- | ----------------- | ----------------------------------- |
| 强制合并 | `must_merge`      | 在融合/分裂判定前，强制将指定的两个 OneID 合并，跳过重叠度计算 |
| 强制拆分 | `must_split`      | 在融合/分裂判定前，强制将指定 ID 标识从当前 OneID 中剥离  |
| 禁止合并 | `must_not_merge`  | 即使重叠度高于阈值，也不允许合并指定的 OneID 对         |
| 权重覆盖 | `weight_override` | 在边表构建时，使用干预指定的权重替代自动计算的权重           |
| 边删除  | `edge_removed`    | 在图预处理（阶段 2）时，移除指定的边，阻止连通            |
| 边新增  | `edge_added`      | 在图预处理（阶段 2）时，新增指定的边，促进连通            |


#### 干预执行时序

```
离线批处理开始（每日 T+1）
       │
       ▼
┌──────────────────────────────────┐
│  阶段 1：数据采集与标准化清洗       │
│                                  │
│  • 正常采集数据                    │
│  • 应用「权重覆盖」规则             │
│  • 应用「边新增」规则               │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  阶段 2：图数据加载与预处理         │
│                                  │
│  • 正常构建 GraphFrame             │
│  • 应用「边删除」规则               │
│    （在超级节点剪枝后、WCC 前移除边） │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  阶段 3：WCC 算法执行              │
│                                  │
│  • 正常执行 WCC                    │
│  • 产出候选 OneID                  │
│    （已反映边删除/新增的影响）        │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  阶段 4：ID 映射决策与冲突消解      │
│                                  │
│  ① 加载干预规则表                  │
│  ② 执行「强制合并」规则             │
│    → 在候选 OneID 与历史比对前，    │
│      先强制合并指定的 OneID 对      │
│  ③ 执行「强制拆分」规则             │
│    → 在融合/分裂判定前，            │
│      先将指定 ID 标识从 OneID 剥离  │
│  ④ 执行「禁止合并」规则             │
│    → 在融合判定时，                 │
│      跳过被禁止合并的 OneID 对      │
│  ⑤ 正常执行融合/分裂判定            │
│    （排除已干预的部分）              │
└──────────────────────────────────┘
```

#### 干预冻结机制

当 OneID 被标记为 `frozen` 状态时：

- 自动计算流程**跳过**该 OneID 的合并/拆分判定
- 该 OneID 的成员 ID 标识列表保持不变
- 新的边数据即使会连通到该 OneID，也不会触发自动合并
- 解冻后，下一次离线批处理恢复正常计算

```
// 阶段 4 中的冻结检查伪代码
对于每个候选 OneID C_new：
  对于每个匹配的历史 OneID H_old：
    如果 H_old.status == "frozen"：
      跳过该 H_old 的融合/分裂判定
      保持 H_old 的成员列表不变
      C_new 中与 H_old 重叠的 ID 标识保留在 H_old 中
    否则：
      正常执行融合/分裂判定
```

### 关系拓扑图查询

#### 目标

提供可视化的关系拓扑数据，支撑前端界面展示某个 OneID 下挂载的所有 ID 标识及其关联关系，辅助运营人员判断合并是否正确。

#### 拓扑查询接口

```
TopologyQueryRequest {
  one_id: String,                      // 查询目标 OneID
  scope: Enum,                         // 层级
  depth: Integer,                      // 展开深度（默认 1，最大 3）
  include_edge_weights: Boolean,       // 是否包含边权重
  include_co_occurrence: Boolean,      // 是否包含共现信息
  filter_id_types: List<Enum>?         // 过滤特定 ID 类型
}

TopologyGraph {
  // ── 中心节点 ──
  center: TopologyNode,
  
  // ── 节点列表 ──
  nodes: List<TopologyNode>,
  
  // ── 边列表 ──
  edges: List<TopologyEdge>,
  
  // ── 元信息 ──
  total_node_count: Integer,
  total_edge_count: Integer,
  generated_at: Timestamp
}

TopologyNode {
  node_id: String,                     // 节点 ID = vertex_id
  id_type: Enum,                       // ID 类型
  id_value: String,                    // ID 值（脱敏展示）
  source_systems: List<String>,        // 来源系统列表
  is_primary: Boolean,                 // 是否为主 ID
  weight: Float,                       // 综合权重
  first_seen_at: Timestamp,
  last_seen_at: Timestamp,
  observation_count: Long,             // 观测次数
  node_status: Enum,                   // "normal" | "frozen" | "intervened"
  intervention_flag: Boolean,          // 是否有干预规则作用于该节点
  display_label: String                // 前端展示标签（如 "手机号: 138****5678"）
}

TopologyEdge {
  src: String,                         // 源节点 node_id
  dst: String,                         // 目标节点 node_id
  weight: Float,                       // 边权重
  co_occurrence_count: Long,           // 共现次数
  source_system: String,               // 共现来源
  last_co_at: Timestamp,               // 最近共现时间
  edge_status: Enum,                   // "auto" | "intervened" | "deleted"
  intervention_source: String?         // 若干预产生，记录来源操作 ID
}
```

#### 拓扑图展示示例

```
查询: OneID = BR:1001:000000042

                    ┌─────────────────────┐
                    │  BR:1001:000000042  │
                    │  (中心 OneID 节点)   │
                    └──────────┬──────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
            ▼                  ▼                  ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │ phone:       │  │ email:       │  │ openid:      │
   │ 138****5678  │  │ u***@ex.com  │  │ oX***d       │
   │ (主ID, w=0.9)│  │ (w=0.9)      │  │ (w=0.85)     │
   │ 来源: APP    │  │ 来源: 小程序  │  │ 来源: 小程序  │
   └──────┬───────┘  └──────┬───────┘  └──────────────┘
          │                 │
          │    ┌────────────┘
          │    │
          ▼    ▼
   ┌──────────────┐  ┌──────────────┐
   │ device_id:   │  │ cookie_id:   │
   │ abc***def    │  │ ck***123     │
   │ (w=0.8)      │  │ (w=0.4)      │
   │ 来源: APP    │  │ 来源: Web    │
   │ [🔒 干预标记] │  │              │
   └──────────────┘  └──────────────┘

边信息:
  phone ↔ email:    w=0.90, 共现 15 次, 来源: APP 注册
  phone ↔ openid:   w=0.85, 共现 12 次, 来源: 小程序登录
  phone ↔ device:   w=0.80, 共现 30 次, 来源: APP 登录
  email ↔ openid:   w=0.70, 共现 8 次,  来源: 小程序
  device ↔ cookie:  w=0.40, 共现 3 次,  来源: Web 浏览
  
干预标记:
  ⚠️ device_id:abc***def 有干预规则: edge_weight_adjust
     操作人: 张三, 原因: "该设备为家庭共用设备，已调低权重"
```

### 全链路审计日志（Audit Log）

#### 审计日志数据结构

```
AuditLog {
  log_id: String,                      // 日志唯一 ID
  
  // ── 操作信息 ──
  operation_type: Enum,                // "auto_merge" | "auto_split" | "force_merge" | "force_split" | "id_unbind" | "edge_weight_adjust" | "edge_delete" | "edge_add" | "freeze" | "unfreeze" | "rollback"
  operation_id: String?,               // 关联的干预操作 ID（自动操作为空）
  batch_id: String?,                   // 关联的计算批次号（自动操作时填写）
  
  // ── 变更目标 ──
  scope: Enum,
  scope_id: String,
  affected_one_ids: List<String>,      // 受影响的 OneID 列表
  
  // ── 变更前后快照 ──
  before_snapshot: Json,               // 变更前完整快照
  after_snapshot: Json,                // 变更后完整快照
  diff_summary: String,                // 可读的变更摘要
  
  // ── 操作人 ──
  operator_type: Enum,                 // "system" | "human"
  operator_id: String,                 // 系统标识或用户 ID
  operator_name: String?,              // 用户姓名（人工操作时）
  
  // ── 原因与审批 ──
  reason: String,                      // 操作原因
  approved_by: List<String>?,          // 审批人列表（人工操作时）
  
  // ── 时间 ──
  created_at: Timestamp
}
```

### 版本回滚机制

#### 回滚类型


| 回滚类型        | 说明                    | 触发场景              |
| ----------- | --------------------- | ----------------- |
| **单用户回滚**   | 将特定 OneID 回滚到指定历史版本   | 某个用户的合并/拆分被确认有误   |
| **任务级回滚**   | 撤销某次干预操作的全部影响         | 干预操作执行后发现异常       |
| **全量时间点回滚** | 将整个 OneID 映射表回滚到某天的状态 | 算法事故，大批量 OneID 出错 |


#### 回滚流程

```
发起回滚请求
       │
       ▼
┌──────────────────────────────────┐
│  1. 定位目标版本快照               │
│     • 单用户: 查该 OneID 的历史版本 │
│     • 任务级: 查操作前后的全量快照   │
│     • 全量: 查指定日期的全量快照     │
│                                  │
│  2. Dry Run 模拟回滚影响           │
│     • 计算回滚影响的 OneID 数量     │
│     • 评估对跨层映射的影响          │
│     • 评估对下游系统的影响          │
│                                  │
│  3. 审批（全量回滚需高级审批）       │
│                                  │
│  4. 执行回滚                      │
│     • 恢复 OneID 映射表            │
│     • 恢复跨层映射表               │
│     • 刷新 Redis / HBase 缓存     │
│     • 触发下游数仓重算             │
│                                  │
│  5. 记录审计日志                   │
└──────────────────────────────────┘
```

#### 回滚数据结构

```
RollbackOperation {
  rollback_id: String,
  rollback_type: Enum,                 // "single_user" | "task_level" | "full_snapshot"
  
  // ── 回滚目标 ──
  target_one_id: String?,              // 单用户回滚时的目标 OneID
  target_operation_id: String?,        // 任务级回滚时的目标操作 ID
  target_snapshot_date: Date?,         // 全量回滚时的目标日期
  
  // ── 回滚状态 ──
  status: Enum,                        // "pending" | "dry_running" | "pending_approval" | "approved" | "executing" | "completed" | "failed"
  dry_run_result: DryRunResult?,
  
  // ── 审批 ──
  requested_by: String,
  approved_by: List<String>?,
  reason: String,
  
  // ── 生命周期 ──
  created_at: Timestamp,
  executed_at: Timestamp?,
  completed_at: Timestamp?
}
```

### 历史数据回填（Data Backfill）

当 OneID 发生变更（干预操作或回滚）后，下游数仓和画像表的历史数据可能需要修正。

#### 回填策略

```
BackfillTask {
  task_id: String,
  source_operation_id: String,         // 触发回填的操作 ID
  
  // ── 回填范围 ──
  affected_one_ids: List<String>,      // 需要回填的 OneID 列表
  affected_date_range: DateRange,      // 受影响的数据日期范围
  target_systems: List<String>,        // 目标系统: ["clickhouse", "hive", "redis"]
  
  // ── 回填模式 ──
  mode: Enum,                          // "full_rewrite" | "incremental_patch"
  
  // ── 执行状态 ──
  status: Enum,                        // "pending" | "running" | "completed" | "failed"
  progress: Float,                     // 进度百分比
  started_at: Timestamp?,
  completed_at: Timestamp?,
  error_message: String?
}
```

**回填触发规则**：

- 单用户回滚：自动触发该 OneID 的全量历史数据回填
- 任务级回滚：自动触发受影响 OneID 列表的回填
- 全量回滚：自动触发全量回填任务（耗时较长，异步执行）
- 干预操作（合并/拆分）：可选触发回填，由操作人在提交时选择

---

## 需求完成状态


| 需求                                 | 状态    | 实现位置              |
| ---------------------------------- | ----- | ----------------- |
| 支持向下传递 OneID 关系，作为集团/品牌 OneID 合并依据 | ✅ 已完成 | 阶段 7：跨层映射表计算与向下传递 |
| 支持干预 OneID 合并拆分                    | ✅ 已完成 | 人工干预 OneID 合并拆分机制 |


