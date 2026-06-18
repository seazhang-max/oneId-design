# OneID 系统 BigQuery 表设计（OneData 方法论）

## 一、数据分层架构

采用 OneData 方法论的标准四层架构：


| 层级        | 命名前缀   | 用途                   | 数据集         |
| --------- | ------ | -------------------- | ----------- |
| **ODS 层** | `ods_` | 操作数据层：原始数据接入，保持源数据原貌 | `oneid_ods` |
| **DWD 层** | `dwd_` | 明细数据层：清洗后的明细数据，维度退化  | `oneid_dwd` |
| **DWS 层** | `dws_` | 汇总数据层：按主题汇总，轻度聚合     | `oneid_dws` |
| **ADS 层** | `ads_` | 应用数据层：面向应用的数据服务      | `oneid_ads` |


> **说明**：OneID 系统不直接读取 ODS 原始数据，而是从上游已经整理好的 DWD 层用户明细与行为事件数据开始构建点表、边表和后续 OneID 资产。

## 二、命名规范

### 2.1 表名格式

```
{层级}_{业务域}_{数据域}_{表名}_{增量标识}
```

**示例**：

- `dwd_trade_user_user_profile_wide_di`：DWD层_交易域_用户域_用户信息宽表_日增量
- `dwd_oneid_graph_vertex_df`：DWD层_OneID域_图计算域_点表_日全量
- `dws_oneid_identity_oneid_master_df`：DWS层_OneID域_身份域_OneID主表_日全量

### 2.2 增量标识


| 标识    | 含义                       | 适用场景       |
| ----- | ------------------------ | ---------- |
| `_di` | 日增量（daily incremental）   | 每天新增/变更的数据 |
| `_df` | 日全量（daily full）          | 每天全量快照     |
| `_mi` | 月增量（monthly incremental） | 每月增量       |
| `_mf` | 月全量（monthly full）        | 每月全量       |


### 2.3 业务域划分


| 业务域    | 缩写        | 说明            |
| ------ | --------- | ------------- |
|        |           |               |
| OneID域 | `oneid`   | OneID 系统核心数据  |
| 治理域    | `gov`     | 数据治理与运维数据     |
| 服务域    | `service` | 面向下游业务消费的服务配置 |


---

## 三、DWD 层 — 上游整理明细数据（oneid_dwd 数据集）

### 3.1 dwd_trade_user_user_profile_wide_di（用户信息宽表）

从上游 DWD 层接入的用户 ID 标识数据，已经完成基础整理，作为 OneID 图构建的源明细。

```sql
CREATE OR REPLACE TABLE `oneid_dwd.dwd_trade_user_user_profile_wide_di`
(
  -- 数据来源标识
  source_system       STRING        NOT NULL,  -- 来源系统: btc_trade | wechat_mini | app_ios | app_android | web_pc
  source_uid          STRING        NOT NULL,  -- 来源系统的用户 UID
  
  -- 层级与业务范围
  brand_id            STRING,                  -- 品牌 ID
  group_id            STRING,                  -- 集团 ID
  business_domain     STRING,                  -- 业务域: ecommerce | content | app | delivery
  
  -- 用户 ID 标识（原始值，未归一化）
  phone_raw           STRING,                  -- 原始手机号
  email_raw           STRING,                  -- 原始邮箱
  device_id_raw       STRING,                  -- 原始设备 ID
  openid_raw          STRING,                  -- 原始 OpenID
  unionid_raw         STRING,                  -- 原始 UnionID
  cookie_id_raw       STRING,                  -- 原始 Cookie ID
  third_party_ids     ARRAY<STRUCT<  -- 其他第三方 ID 列表
    id_type           STRING,                  -- ID 类型
    id_value          STRING,                  -- ID 值
    source_system     STRING                   -- 来源系统
  >>,                                          -- 其他第三方 ID
  
  -- 用户属性
  nickname            STRING,  -- 用户昵称
  gender              STRING,  -- 用户性别
  birthday            DATE,  -- 用户生日
  membership_level    STRING,  -- 会员等级
  
  -- 时间戳
  first_seen_at       TIMESTAMP     NOT NULL,  -- 时间戳字段
  last_seen_at        TIMESTAMP     NOT NULL,  -- 时间戳字段
  
  -- 数据质量标记
  data_quality_flag   STRING,                  -- clean | suspicious | test_data | bot
  
  -- 元数据
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  ingest_timestamp    TIMESTAMP     NOT NULL,  -- 时间戳字段
  
  -- 分区字段
  dt                  DATE          NOT NULL   -- 数据日期
)
PARTITION BY dt
CLUSTER BY source_system, scope, scope_id
OPTIONS (
  description = 'DWD层：用户信息宽表，OneID 从该表读取已整理的用户 ID 标识数据',
  partition_expiration_days = 180
);
```

**示例数据**：


| source_system | source_uid | scope | scope_id | business_domain | brand_id | phone_raw         | email_raw                                           | openid_raw    | unionid_raw      | device_id_raw    | cookie_id_raw |
| ------------- | ---------- | ----- | -------- | --------------- | -------- | ----------------- | --------------------------------------------------- | ------------- | ---------------- | ---------------- | ------------- |
| btc_trade     | U001       | brand | 1001     | ecommerce       | 1001     | 13812345678       | [ZhangSan@Example.com](mailto:ZhangSan@Example.com) | NULL          | NULL             | NULL             | NULL          |
| wechat_mini   | WX001      | brand | 1001     | app             | 1001     | +86 138 1234 5678 | NULL                                                | openid_def456 | unionid_zhang001 | NULL             | NULL          |
| app_ios       | APP001     | brand | 1001     | app             | 1001     | 13812345678       | NULL                                                | openid_abc123 | NULL             | device_zhang_001 | NULL          |
| btc_trade     | U002       | brand | 1001     | ecommerce       | 1001     | 13987654321       | [lisi@example.com](mailto:lisi@example.com)         | NULL          | NULL             | NULL             | NULL          |
| app_ios       | APP002     | brand | 1001     | app             | 1001     | 13987654321       | NULL                                                | openid_ghi012 | NULL             | device_lisi_001  | NULL          |


> **说明**：张三在三个系统有记录（btc_trade、wechat_mini、app_ios），手机号格式不统一（有空格、+86 前缀、大小写）。李四在两个系统有记录。

### 3.2 dwd_trade_user_association_event_di（关联事件表）

记录用户行为事件中多个 ID 标识的共现关系。

```sql
CREATE OR REPLACE TABLE `oneid_dwd.dwd_trade_user_association_event_di`
(
  -- 事件标识
  event_id            STRING        NOT NULL,  -- 事件唯一标识
  
  -- 数据来源
  source_system       STRING        NOT NULL,  -- 来源系统标识
  event_type          STRING        NOT NULL,  -- login | register | order | browse | payment
  
  -- 层级与范围
  brand_id            STRING,  -- 品牌 ID
  group_id            STRING,  -- 集团 ID
  business_domain     STRING,                  -- 业务域: ecommerce | content | app | delivery
  
  -- 事件中包含的 ID 标识（共现关系）
  uid                 STRING,  -- 事件中的业务用户 ID
  phone_raw           STRING,  -- 原始手机号
  email_raw           STRING,  -- 原始邮箱
  device_id_raw       STRING,  -- 原始设备 ID
  openid_raw          STRING,  -- 原始 OpenID
  unionid_raw         STRING,  -- 原始 UnionID
  cookie_id_raw       STRING,  -- 原始 Cookie ID
  
  -- 事件上下文
  session_id          STRING,  -- 会话 ID
  ip_address          STRING,  -- 客户端 IP 地址
  user_agent          STRING,  -- 客户端 User-Agent
  
  -- 时间戳
  event_timestamp     TIMESTAMP     NOT NULL,  -- 时间戳字段
  
  -- 元数据
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  ingest_timestamp    TIMESTAMP     NOT NULL,  -- 时间戳字段
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY source_system, event_type, scope
OPTIONS (
  description = 'DWD层：关联事件表，记录已整理行为事件中的 ID 标识共现关系',
  partition_expiration_days = 180
);
```

**示例数据**：


| event_id | source_system | event_type | scope | scope_id | business_domain | uid    | phone_raw   | openid_raw    | unionid_raw      | device_id_raw    | cookie_id_raw      |
| -------- | ------------- | ---------- | ----- | -------- | --------------- | ------ | ----------- | ------------- | ---------------- | ---------------- | ------------------ |
| EVT001   | app_ios       | login      | brand | 1001     | app             | APP001 | 13812345678 | openid_abc123 | NULL             | device_zhang_001 | NULL               |
| EVT002   | wechat_mini   | login      | brand | 1001     | app             | WX001  | 13812345678 | openid_def456 | unionid_zhang001 | NULL             | NULL               |
| EVT003   | app_ios       | login      | brand | 1001     | app             | APP002 | 13987654321 | openid_ghi012 | NULL             | device_lisi_001  | NULL               |
| EVT004   | web_pc        | browse     | brand | 1001     | ecommerce       | NULL   | NULL        | NULL          | NULL             | NULL             | cookie_public_wifi |
| EVT005   | web_pc        | browse     | brand | 1001     | ecommerce       | NULL   | NULL        | NULL          | NULL             | NULL             | cookie_public_wifi |


> **说明**：EVT001 是张三 APP 登录，同时出现 phone+openid+device_id（三者将建立边）。EVT004/EVT005 是张三和李四在同一 WiFi 下浏览，共享 cookie_public_wifi。

---

## 四、DWD 层 — OneID 图计算明细（oneid_dwd 数据集）

### 4.1 dwd_oneid_graph_vertex_df（点表）

清洗后的图计算点表，每个唯一的 ID 标识值作为一个顶点。

> **说明**：该表是 Spark 图计算链路中的 DataFrame 中间结果，作为 GraphFrames/WCC 的输入使用，不作为下游业务查询或服务发布的直接消费表。

```sql
CREATE OR REPLACE TABLE `oneid_dwd.dwd_oneid_graph_vertex_df`
(
  -- 基础标识
  vertex_id         STRING        NOT NULL,  -- 唯一标识 = id_type + ":" + id_value
  id_type           STRING        NOT NULL,  -- ID 类型
  id_value          STRING        NOT NULL,  -- ID 值

  -- 来源与观测
  source_system     STRING        NOT NULL,  -- 来源系统标识
  first_seen_at     TIMESTAMP     NOT NULL,  -- 时间戳字段
  last_seen_at      TIMESTAMP     NOT NULL,  -- 时间戳字段
  observation_count INT64         NOT NULL,  -- 数量统计字段

  -- 分区与层级
  scope             STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id          STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global

  -- 元数据
  batch_id          STRING        NOT NULL,  -- 计算或接入批次号
  
  -- 分区字段
  dt                DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, id_type
OPTIONS (
  description = 'DWD层：Spark DataFrame 图计算点表，每个唯一的 ID 标识值作为一个顶点，作为 GraphFrames/WCC 输入',
  partition_expiration_days = 90
);
```

**示例数据**：


| vertex_id                                                 | id_type   | id_value                                            | source_system | scope | scope_id | observation_count |
| --------------------------------------------------------- | --------- | --------------------------------------------------- | ------------- | ----- | -------- | ----------------- |
| phone:13812345678                                         | phone     | 13812345678                                         | btc_trade     | brand | 1001     | 3                 |
| email:[zhangsan@example.com](mailto:zhangsan@example.com) | email     | [zhangsan@example.com](mailto:zhangsan@example.com) | btc_trade     | brand | 1001     | 1                 |
| openid:openid_abc123                                      | openid    | openid_abc123                                       | app_ios       | brand | 1001     | 1                 |
| openid:openid_def456                                      | openid    | openid_def456                                       | wechat_mini   | brand | 1001     | 1                 |
| unionid:unionid_zhang001                                  | unionid   | unionid_zhang001                                    | wechat_mini   | brand | 1001     | 1                 |
| device:device_zhang_001                                   | device_id | device_zhang_001                                    | app_ios       | brand | 1001     | 1                 |
| phone:13987654321                                         | phone     | 13987654321                                         | btc_trade     | brand | 1001     | 2                 |
| email:[lisi@example.com](mailto:lisi@example.com)         | email     | [lisi@example.com](mailto:lisi@example.com)         | btc_trade     | brand | 1001     | 1                 |
| openid:openid_ghi012                                      | openid    | openid_ghi012                                       | app_ios       | brand | 1001     | 1                 |
| device:device_lisi_001                                    | device_id | device_lisi_001                                     | app_ios       | brand | 1001     | 1                 |


> **说明**：张三贡献了 6 个 vertex（phone、email、2 个 openid、unionid、device_id），李四贡献了 4 个 vertex。公共 cookie 被过滤，不生成 vertex。

### 4.2 dwd_oneid_graph_edge_df（边表）

清洗后的图计算边表，同一用户在同一业务事件中共现的 ID 标识之间建立边。

> **说明**：该表是 Spark 图计算链路中的 DataFrame 中间结果，作为 GraphFrames/WCC 的边输入使用，不作为下游业务查询或服务发布的直接消费表。

```sql
CREATE OR REPLACE TABLE `oneid_dwd.dwd_oneid_graph_edge_df`
(
  -- 边的两端
  src                 STRING        NOT NULL,  -- 源顶点 vertex_id
  dst                 STRING        NOT NULL,  -- 目标顶点 vertex_id

  -- 置信度与共现
  confidence          FLOAT64       NOT NULL,  -- 边置信度（两个 ID 标识属于同一自然人的可信程度）
  co_occurrence_count INT64         NOT NULL,  -- 数量统计字段
  first_co_at         TIMESTAMP     NOT NULL,  -- 时间戳字段
  last_co_at          TIMESTAMP     NOT NULL,  -- 时间戳字段

  -- 来源
  source_system       STRING        NOT NULL,  -- 来源系统标识

  -- 分区与层级
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global

  -- 元数据
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id
OPTIONS (
  description = 'DWD层：Spark DataFrame 图计算边表，同一用户共现的 ID 标识之间的关联边，作为 GraphFrames/WCC 输入',
  partition_expiration_days = 90
);
```

**示例数据**：


| src                      | dst                                                       | confidence | co_occurrence_count | source_system |
| ------------------------ | --------------------------------------------------------- | ---------- | ------------------- | ------------- |
| phone:13812345678        | email:[zhangsan@example.com](mailto:zhangsan@example.com) | 0.90       | 1                   | btc_trade     |
| phone:13812345678        | openid:openid_abc123                                      | 0.80       | 1                   | app_ios       |
| phone:13812345678        | device:device_zhang_001                                   | 0.80       | 1                   | app_ios       |
| phone:13812345678        | openid:openid_def456                                      | 0.85       | 1                   | wechat_mini   |
| phone:13812345678        | unionid:unionid_zhang001                                  | 0.85       | 1                   | wechat_mini   |
| unionid:unionid_zhang001 | openid:openid_def456                                      | 0.95       | 1                   | wechat_mini   |
| phone:13987654321        | email:[lisi@example.com](mailto:lisi@example.com)         | 0.90       | 1                   | btc_trade     |
| phone:13987654321        | openid:openid_ghi012                                      | 0.80       | 1                   | app_ios       |
| phone:13987654321        | device:device_lisi_001                                    | 0.80       | 1                   | app_ios       |


> **说明**：张三的 6 个 vertex 之间形成了 6 条边（高权重连通），李四的 4 个 vertex 之间形成了 3 条边。由于公共 cookie 被过滤，张三和李四之间**没有边**。

### 4.3 dwd_oneid_graph_cleaning_log_di（清洗日志表）

记录数据清洗过程，用于审计和数据质量分析。

```sql
CREATE OR REPLACE TABLE `oneid_dwd.dwd_oneid_graph_cleaning_log_di`
(
  -- 日志标识
  log_id              STRING        NOT NULL,  -- 日志唯一标识
  
  -- 关联的源明细数据
  source_table        STRING        NOT NULL,  -- dwd_trade_user_user_profile_wide_di | dwd_trade_user_association_event_di
  source_id           STRING        NOT NULL,  -- 源明细记录 ID
  
  -- 清洗动作
  action_type         STRING        NOT NULL,  -- normalized | filtered | transformed | kept
  
  -- 归一化详情
  normalization_rules ARRAY<STRUCT<  -- 归一化规则执行明细
    field_name        STRING,  -- 被处理的字段名
    rule_applied      STRING,  -- 应用的规则名称
    before_value      STRING,  -- 处理前字段值
    after_value       STRING  -- 处理后字段值
  >>,
  
  -- 过滤详情
  filter_reason       STRING,  -- 数据被过滤的原因
  filter_rule_id      STRING,  -- 命中的过滤规则 ID
  
  -- 层级
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global
  
  -- 元数据
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, action_type, filter_reason
OPTIONS (
  description = 'DWD层：数据清洗日志表，记录数据清洗过程',
  partition_expiration_days = 90
);
```

**示例数据**：


| log_id | source_table                        | source_id | action_type | normalization_rules                                                                                                                                                                                                                                               | filter_reason |
| ------ | ----------------------------------- | --------- | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| LOG001 | dwd_trade_user_user_profile_wide_di | U001      | normalized  | [{field:"phone_raw", rule:"remove_prefix_86", before:"+86 138 1234 5678", after:"13812345678"}, {field:"email_raw", rule:"lowercase", before:"[ZhangSan@Example.com](mailto:ZhangSan@Example.com)", after:"[zhangsan@example.com](mailto:zhangsan@example.com)"}] | NULL          |
| LOG002 | dwd_trade_user_association_event_di | EVT004    | filtered    | NULL                                                                                                                                                                                                                                                              | public_id     |
| LOG003 | dwd_trade_user_association_event_di | EVT005    | filtered    | NULL                                                                                                                                                                                                                                                              | public_id     |


> **说明**：LOG001 记录了张三手机号的归一化过程（去 +86、去空格）和邮箱的小写化。LOG002/LOG003 记录了公共 WiFi cookie 被过滤。

---

## 五、DWS 层 — 汇总数据层（oneid_dws 数据集）

### 5.1 dws_oneid_identity_wcc_result_df（WCC 归属结果表）

WCC 算法收敛后，每个 vertex 的连通分量归属结果。按 vertex 粒度存储（而非按分量聚合），方便后续与 vertex/edge 表 JOIN 做排查分析。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_wcc_result_df`
(
  -- 顶点信息
  vertex_id           STRING        NOT NULL,  -- 顶点 ID = id_type:id_value
  id_type             STRING        NOT NULL,  -- ID 类型
  id_value            STRING        NOT NULL,  -- ID 值

  -- WCC 归属
  component_id        STRING        NOT NULL,  -- 所属连通分量 ID
  component_size      INT64         NOT NULL,  -- 该连通分量的总顶点数（用于识别超大分量）

  -- 该顶点的边统计（辅助排查）
  degree              INT64         NOT NULL,  -- 该顶点的度数（关联边数量）
  neighbor_count      INT64         NOT NULL,  -- 邻居顶点数量
  total_edge_confidence FLOAT64     NOT NULL,  -- 关联边的置信度总和

  -- 分区与层级
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global

  -- 元数据
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, component_id
OPTIONS (
  description = 'DWS层：WCC归属结果表，每个vertex的连通分量归属，支持JOIN排查',
  partition_expiration_days = 90
);
```

**示例数据**：


| vertex_id                                                 | id_type   | id_value                                            | component_id | component_size | degree | neighbor_count | total_edge_confidence |
| --------------------------------------------------------- | --------- | --------------------------------------------------- | ------------ | -------------- | ------ | -------------- | --------------------- |
| phone:13812345678                                         | phone     | 13812345678                                         | COMP_ZHANG   | 6              | 5      | 5              | 4.25                  |
| email:[zhangsan@example.com](mailto:zhangsan@example.com) | email     | [zhangsan@example.com](mailto:zhangsan@example.com) | COMP_ZHANG   | 6              | 1      | 1              | 0.90                  |
| openid:openid_abc123                                      | openid    | openid_abc123                                       | COMP_ZHANG   | 6              | 1      | 1              | 0.80                  |
| openid:openid_def456                                      | openid    | openid_def456                                       | COMP_ZHANG   | 6              | 2      | 2              | 1.80                  |
| unionid:unionid_zhang001                                  | unionid   | unionid_zhang001                                    | COMP_ZHANG   | 6              | 2      | 2              | 1.80                  |
| device:device_zhang_001                                   | device_id | device_zhang_001                                    | COMP_ZHANG   | 6              | 1      | 1              | 0.80                  |
| phone:13987654321                                         | phone     | 13987654321                                         | COMP_LI      | 4              | 3      | 3              | 2.50                  |
| email:[lisi@example.com](mailto:lisi@example.com)         | email     | [lisi@example.com](mailto:lisi@example.com)         | COMP_LI      | 4              | 1      | 1              | 0.90                  |
| openid:openid_ghi012                                      | openid    | openid_ghi012                                       | COMP_LI      | 4              | 1      | 1              | 0.80                  |
| device:device_lisi_001                                    | device_id | device_lisi_001                                     | COMP_LI      | 4              | 1      | 1              | 0.80                  |


> **说明**：WCC 产出两个连通分量 — COMP_ZHANG（张三，6 个 vertex）和 COMP_LI（李四，4 个 vertex）。因为公共 cookie 被过滤，两人没有被错误连通。

### 5.2 dws_oneid_identity_mapping_decision_di（ID映射决策表）

阶段4的核心过程表，记录本轮候选 OneID 与历史 OneID 的增量比对结果和映射决策。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_mapping_decision_di`
(
  -- 基础标识
  decision_id         STRING        NOT NULL,  -- 决策唯一 ID
  
  -- 本轮候选 OneID
  candidate_component_id STRING     NOT NULL,  -- WCC 产出的候选分量 ID
  candidate_member_ids ARRAY<STRING> NOT NULL, -- 候选分量的成员 ID 标识列表
  candidate_member_count INT64      NOT NULL,  -- 数量统计字段
  
  -- 匹配的历史 OneID（可能有多个）
  matched_history_one_ids ARRAY<STRING>,       -- 匹配到的历史 OneID 列表
  matched_history_count INT64,                 -- 匹配到的历史 OneID 数量
  
  -- 顶点匹配计算（基于 ID 标识是否有相同）
  best_match_one_id   STRING,                  -- 最佳匹配的历史 OneID（共享 ID 标识数量最多的）
  best_match_shared_count INT64,               -- 最佳匹配的共享 ID 标识数量
  all_match_details   ARRAY<STRUCT<  -- 候选分量与所有历史 OneID 的匹配明细
    history_one_id    STRING,                  -- 历史 OneID
    shared_id_count   INT64,                   -- 共享 ID 标识数量
    shared_id_types   ARRAY<STRING>,           -- 共享 ID 类型列表
    shared_id_values  ARRAY<STRING>            -- 共享的 ID 标识值列表（vertex_id）
  >>,
  
  -- 决策结果
  decision_type       STRING        NOT NULL,  -- create | keep | merge | split
  decision_reason     STRING,                  -- 决策原因说明
  
  -- 融合决策详情（decision_type = merge 时）
  merge_target_one_id STRING,                  -- 融合目标 OneID（保留此 ID）
  merge_source_one_ids ARRAY<STRING>,          -- 被融合的源 OneID 列表
  merge_confidence    FLOAT64,                 -- 融合置信度
  
  -- 分裂决策详情（decision_type = split 时）
  split_source_one_id STRING,                  -- 被分裂的源 OneID
  split_target_one_ids ARRAY<STRING>,          -- 分裂后的新 OneID 列表
  
  -- 新增决策详情（decision_type = create 时）
  new_one_id          STRING,                  -- 新分配的 OneID
  
  -- 干预规则影响
  intervention_applied BOOL         NOT NULL,  -- 是否应用人工干预规则
  intervention_rule_ids ARRAY<STRING>,         -- 应用的干预规则 ID 列表
  
  -- 层级
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global
  
  -- 元数据
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, decision_type
OPTIONS (
  description = 'DWS层：ID映射决策表，记录候选OneID与历史OneID的比对和决策过程',
  partition_expiration_days = 90
);
```

**示例数据**：


| decision_id | candidate_component_id | candidate_member_ids                                                                | decision_type | best_match_one_id | best_match_shared_count | merge_target_one_id | merge_source_one_ids | new_one_id        | intervention_applied |
| ----------- | ---------------------- | ----------------------------------------------------------------------------------- | ------------- | ----------------- | ----------------------- | ------------------- | -------------------- | ----------------- | -------------------- |
| DEC001      | COMP_ZHANG             | [phone:138..., email:zhang..., openid:abc, openid:def, unionid:zhang, device:zhang] | merge         | BR:1001:000000099 | 2                       | BR:1001:000000099   | [BR:1001:000000100]  | NULL              | false                |
| DEC002      | COMP_LI                | [phone:139..., email:lisi..., openid:ghi, device:lisi]                              | create        | NULL              | 0                       | NULL                | NULL                 | BR:1001:000000101 | false                |


> **说明**：
>
> - **DEC001**：COMP_ZHANG 与历史 OneID BR:1001:000000099（原有 phone+email）有 2 个共享顶点（phone:138...、email:zhang...），同时与 BR:1001:000000100（原有 openid+unionid）也有共享顶点，候选分量同时包含两个历史 OneID 的成员，触发**融合** — 将 BR:1001:000000100 合并到 BR:1001:000000099
> - **DEC002**：COMP_LI 与所有历史 OneID 都没有共享顶点，**新建** BR:1001:000000101

### 5.3 dws_oneid_identity_merge_operation_di（融合操作表）

记录 OneID 融合操作的结果和原因。当需要查看融合前后的完整成员列表或画像时，通过 `dt + batch_id` 回查 `dws_oneid_identity_oneid_master_df` 的前后快照，不在操作表中重复保存镜像数据。若每天只保留一轮全量结果，按 `dt` 追溯即可；若一天存在多轮计算，需要保留批次级快照并按 `batch_id` 定位。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_merge_operation_di`
(
  -- 操作标识
  operation_id        STRING        NOT NULL,  -- 操作唯一标识

  -- 融合结果
  target_one_id       STRING        NOT NULL,  -- 融合后保留的 OneID
  source_one_ids      ARRAY<STRING> NOT NULL,  -- 被融合的源 OneID 列表
  source_one_id_count INT64         NOT NULL,  -- 数量统计字段

  -- 融合原因
  merge_reason        STRING        NOT NULL,  -- 融合或合并原因说明
  merge_reason_code   STRING        NOT NULL,  -- wcc_connected | cross_layer_guidance | manual_intervention | rollback_replay
  trigger_type        STRING        NOT NULL,  -- auto | manual | rollback
  merge_confidence    FLOAT64,  -- 融合置信度
  shared_id_count     INT64,  -- 数量统计字段
  shared_id_types     ARRAY<STRING>,  -- 共享 ID 类型列表

  -- 层级
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global

  -- 元数据
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  decision_id         STRING,  -- 关联决策 ID
  source_operation_id STRING,                  -- 来源操作 ID
  operation_date      DATE          NOT NULL,  -- 用于定位前后快照分区；多批次场景需结合 batch_id

  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, trigger_type
OPTIONS (
  description = 'DWS层：融合操作表，只记录融合结果与原因；完整前后状态通过oneid_master按dt追溯',
  partition_expiration_days = 90
);
```

**示例数据**：


| operation_id | target_one_id     | source_one_ids      | source_one_id_count | merge_reason                      | merge_reason_code | trigger_type | merge_confidence | shared_id_count | shared_id_types | operation_date |
| ------------ | ----------------- | ------------------- | ------------------- | --------------------------------- | ----------------- | ------------ | ---------------- | --------------- | --------------- | -------------- |
| MERGE001     | BR:1001:000000099 | [BR:1001:000000100] | 1                   | WCC 连通分量包含两个历史 OneID 的成员，共享 2 个顶点 | wcc_connected     | auto         | 0.92             | 2               | [phone,email]   | 2026-06-18     |


> **说明**：MERGE001 将 BR:1001:000000100 融合到 BR:1001:000000099。融合前状态查 `oneid_master_df` 的上一批次或上一日快照，融合后状态查本次 `dt + batch_id` 对应快照。

### 5.4 dws_oneid_identity_split_operation_di（分裂操作表）

记录 OneID 分裂操作的结果和原因。当需要查看分裂前后的完整成员列表时，通过 `dt + batch_id` 回查 `dws_oneid_identity_oneid_master_df` 的前后快照。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_split_operation_di`
(
  -- 操作标识
  operation_id        STRING        NOT NULL,  -- 操作唯一标识

  -- 分裂结果
  source_one_id       STRING        NOT NULL,  -- 被分裂的源 OneID
  target_one_ids      ARRAY<STRING> NOT NULL,  -- 分裂后的新 OneID 列表
  target_one_id_count INT64         NOT NULL,  -- 数量统计字段

  -- 分裂原因
  split_reason        STRING        NOT NULL,  -- 分裂原因说明
  split_reason_code   STRING        NOT NULL,  -- wcc_recompute | id_unbind | manual_intervention | rollback_replay
  trigger_type        STRING        NOT NULL,  -- auto | manual | rollback
  split_basis         STRING,                  -- 分裂依据说明
  component_ids       ARRAY<STRING>,           -- 如果基于 WCC 重算，记录新的分量 ID

  -- 层级
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global

  -- 元数据
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  decision_id         STRING,  -- 关联决策 ID
  source_operation_id STRING,                  -- 来源操作 ID
  operation_date      DATE          NOT NULL,  -- 用于定位前后快照分区；多批次场景需结合 batch_id

  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, trigger_type
OPTIONS (
  description = 'DWS层：分裂操作表，只记录分裂结果与原因；完整前后状态通过oneid_master按dt追溯',
  partition_expiration_days = 90
);
```

**示例数据**：


| operation_id | source_one_id     | target_one_ids                         | target_one_id_count | split_reason | split_reason_code   | trigger_type | split_basis             | operation_date |
| ------------ | ----------------- | -------------------------------------- | ------------------- | ------------ | ------------------- | ------------ | ----------------------- | -------------- |
| SPLIT001     | BR:1001:000000102 | [BR:1001:000000103, BR:1001:000000104] | 2                   | 家庭共用账号被错误合并  | manual_intervention | manual       | 人工确认家庭共用账号，需要拆分手机号与设备标识 | 2026-06-18     |


> **说明**：SPLIT001 将 BR:1001:000000102 分裂为两个新 OneID。分裂前后明细通过 `oneid_master_df` 的相邻 `dt` 分区追溯。

### 5.5 dws_oneid_identity_conflict_resolution_di（冲突消解表）

记录阶段 4 中跨分量冲突的消解结果和原因。当一个 ID 标识同时出现在多个候选 OneID 中时，只记录最终归属；完整候选成员和最终 OneID 状态通过 WCC 结果表、决策表和 OneID 主表分区追溯。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_conflict_resolution_di`
(
  -- 冲突标识
  conflict_id         STRING        NOT NULL,  -- 唯一标识字段

  -- 冲突的 ID 标识
  conflicting_id_type STRING        NOT NULL,  -- 类型字段
  conflicting_id_value STRING       NOT NULL,  -- 取值字段

  -- 冲突范围
  candidate_one_ids   ARRAY<STRING> NOT NULL,  -- 候选 OneID 列表
  candidate_count     INT64         NOT NULL,  -- 数量统计字段

  -- 消解结果
  winner_one_id       STRING        NOT NULL,  -- 冲突获胜 OneID
  loser_one_ids       ARRAY<STRING>,  -- 冲突失败 OneID 列表
  resolution_rule     STRING        NOT NULL,  -- high_weight_priority | recent_activity_priority | intervention_override
  resolution_detail   STRING,  -- 冲突消解详情

  -- 干预规则影响
  intervention_applied BOOL         NOT NULL,  -- 是否应用人工干预规则
  intervention_rule_id STRING,  -- 应用的人工干预规则 ID

  -- 层级
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global

  -- 元数据
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  decision_id         STRING,  -- 关联决策 ID
  operation_date      DATE          NOT NULL,  -- 操作发生日期

  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, resolution_rule
OPTIONS (
  description = 'DWS层：冲突消解表，只记录冲突消解结果与原因；完整状态通过WCC/decision/master分区追溯',
  partition_expiration_days = 90
);
```

**示例数据**：


| conflict_id | conflicting_id_type | conflicting_id_value | candidate_one_ids         | candidate_count | resolution_rule      | resolution_detail                                  | winner_one_id | loser_one_ids | intervention_applied | operation_date |
| ----------- | ------------------- | -------------------- | ------------------------- | --------------- | -------------------- | -------------------------------------------------- | ------------- | ------------- | -------------------- | -------------- |
| CONF001     | phone               | 13812345678          | [COMP_ZHANG, COMP_TEMP_X] | 2               | high_weight_priority | 手机号 13812345678 同时出现在两个候选分量中，按权重优先规则分配给 COMP_ZHANG | COMP_ZHANG    | [COMP_TEMP_X] | false                | 2026-06-18     |


> **说明**：手机号 13812345678 同时出现在 COMP_ZHANG 和一个临时分量 COMP_TEMP_X 中，最终按权重优先规则归属 COMP_ZHANG。

### 5.6 dws_oneid_identity_oneid_master_df（OneID 主表）

OneID 的权威数据源，包含基础标识、ID 映射、物理属性和 AI 画像。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_oneid_master_df`
(
  -- 基础标识
  one_id              STRING        NOT NULL,  -- OneID 唯一标识
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global
  status              STRING        NOT NULL,  -- active | frozen | merged | split
  version             INT64         NOT NULL,  -- OneID 版本号

  -- ID 映射
  id_mappings         ARRAY<STRUCT<  -- OneID 关联的标准化 ID 列表
    id_type           STRING,  -- ID 类型
    id_value          STRING,  -- ID 值
    source_system     STRING,  -- 来源系统标识
    is_primary        BOOL,  -- 是否主 ID
    weight            FLOAT64,  -- 权重字段
    first_seen_at     TIMESTAMP,  -- 时间戳字段
    last_seen_at      TIMESTAMP,  -- 时间戳字段
    status            STRING  -- 记录状态
  >>,

  -- 物理属性
  physical_info       STRUCT<  -- 物理属性结构体
    nickname            STRING,  -- 用户昵称
    gender              STRING,  -- 用户性别
    birthday            DATE,  -- 用户生日
    avatar_url          STRING,  -- 头像 URL
    phone               STRING,  -- 主手机号
    email               STRING,  -- 主邮箱
    registered_channels ARRAY<STRING>,  -- 注册渠道列表
    first_register_at   TIMESTAMP,  -- 时间戳字段
    membership_level    STRING,  -- 会员等级
    membership_score    INT64,  -- 会员积分
    lifetime_order_count  INT64,  -- 数量统计字段
    lifetime_order_amount NUMERIC,  -- 累计消费金额
    last_order_at         TIMESTAMP,  -- 时间戳字段
    info_version        INT64,  -- 用户属性版本号
    attribute_sources   JSON  -- 属性字段来源映射
  >,

  -- AI 属性
  ai_profile          STRUCT<  -- AI 画像结构体
    lifecycle_stage     STRING,  -- 生命周期阶段
    lifecycle_score     FLOAT64,  -- 生命周期评分
    purchase_preference ARRAY<STRUCT<  -- 购买偏好标签列表
      tag_name          STRING,  -- 偏好标签名称
      score             FLOAT64,  -- 评分字段
      source            STRING,  -- 数据来源或计算来源
      decay_factor      FLOAT64  -- 时间衰减因子
    >>,
    category_affinity   JSON,  -- 品类亲和度映射
    price_sensitivity   FLOAT64,  -- 价格敏感度
    brand_loyalty       FLOAT64,  -- 品牌忠诚度
    churn_probability     FLOAT64,  -- 流失概率
    purchase_intent       FLOAT64,  -- 近期购买意向
    predicted_lifetime_value NUMERIC,  -- 预测生命周期价值
    customer_segment      STRING,  -- 客户分群
    activity_preference   ARRAY<STRING>,  -- 活跃渠道偏好
    model_version         STRING,  -- 模型版本号
    computed_at           TIMESTAMP,  -- 时间戳字段
    next_recompute_at     TIMESTAMP  -- 时间戳字段
  >,

  -- 溯源与治理
  source              STRING        NOT NULL,  -- 数据来源或计算来源
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  confidence          FLOAT64       NOT NULL,  -- 置信度

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  updated_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  merged_into         STRING,  -- 合并后的目标 OneID
  split_from          STRING,  -- 拆分前的源 OneID
  expired_at          TIMESTAMP,  -- 时间戳字段
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, status
OPTIONS (
  description = 'DWS层：OneID 主表，统一身份实体的权威数据源'
);
```

**示例数据**：

**张三的 OneID**：


| 字段            | 值                                                                                                                                                                                                                                                                                                                                              |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| one_id        | BR:1001:000000099                                                                                                                                                                                                                                                                                                                              |
| scope         | brand                                                                                                                                                                                                                                                                                                                                          |
| scope_id      | 1001                                                                                                                                                                                                                                                                                                                                           |
| status        | active                                                                                                                                                                                                                                                                                                                                         |
| version       | 3                                                                                                                                                                                                                                                                                                                                              |
| id_mappings   | [{phone, 13812345678, btc_trade, true, 0.9}, {email, [zhangsan@example.com](mailto:zhangsan@example.com), btc_trade, false, 0.9}, {openid, openid_abc123, app_ios, false, 0.85}, {openid, openid_def456, wechat_mini, false, 0.85}, {unionid, unionid_zhang001, wechat_mini, false, 0.95}, {device_id, device_zhang_001, app_ios, false, 0.8}] |
| physical_info | {nickname:"张三", gender:"male", phone:"13812345678", email:"[zhangsan@example.com](mailto:zhangsan@example.com)", registered_channels:["app_ios","wechat_mini","btc_trade"], membership_level:"gold", lifetime_order_count:15, lifetime_order_amount:28500.00}                                                                                  |
| ai_profile    | {lifecycle_stage:"active", customer_segment:"high_value", churn_probability:0.05, purchase_intent:0.82, brand_loyalty:0.75}                                                                                                                                                                                                                    |
| confidence    | 0.92                                                                                                                                                                                                                                                                                                                                           |
| source        | cyberbiz                                                                                                                                                                                                                                                                                                                                       |
| batch_id      | BATCH_20260618_001                                                                                                                                                                                                                                                                                                                             |


**李四的 OneID**：


| 字段            | 值                                                                                                                                                                                                                                       |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| one_id        | BR:1001:000000101                                                                                                                                                                                                                       |
| scope         | brand                                                                                                                                                                                                                                   |
| scope_id      | 1001                                                                                                                                                                                                                                    |
| status        | active                                                                                                                                                                                                                                  |
| version       | 1                                                                                                                                                                                                                                       |
| id_mappings   | [{phone, 13987654321, btc_trade, true, 0.9}, {email, [lisi@example.com](mailto:lisi@example.com), btc_trade, false, 0.9}, {openid, openid_ghi012, app_ios, false, 0.85}, {device_id, device_lisi_001, app_ios, false, 0.8}]             |
| physical_info | {nickname:"李四", gender:"male", phone:"13987654321", email:"[lisi@example.com](mailto:lisi@example.com)", registered_channels:["app_ios","btc_trade"], membership_level:"silver", lifetime_order_count:3, lifetime_order_amount:1200.00} |
| ai_profile    | {lifecycle_stage:"active", customer_segment:"growth", churn_probability:0.25, purchase_intent:0.45, brand_loyalty:0.30}                                                                                                                 |
| confidence    | 0.88                                                                                                                                                                                                                                    |
| source        | cyberbiz                                                                                                                                                                                                                                |
| batch_id      | BATCH_20260618_001                                                                                                                                                                                                                      |


### 5.7 dws_oneid_identity_cross_layer_mapping_df（跨层映射表）

三层 OneID 之间的映射关系、合并建议和传递链路。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_cross_layer_mapping_df`
(
  -- 映射标识
  mapping_id          STRING        NOT NULL,  -- 跨层映射记录唯一标识

  -- 映射关系
  brand_one_id        STRING,  -- 品牌层 OneID
  group_one_id        STRING,  -- 集团层 OneID
  global_one_id       STRING,  -- 全局层 OneID
  mapping_type        STRING        NOT NULL,  -- brand_to_group | brand_to_global | group_to_global

  -- 传递链路
  propagation_path    ARRAY<STRING>,  -- 跨层传递路径
  propagation_direction STRING,  -- 跨层传递方向
  propagation_status  STRING,  -- 跨层传递状态
  last_propagated_at  TIMESTAMP,  -- 时间戳字段

  -- 合并建议
  merge_recommendation STRING,  -- 跨层合并建议
  merge_reason         STRING,  -- 融合或合并原因说明
  merge_confidence     FLOAT64,  -- 融合置信度

  -- 置信度
  confidence          FLOAT64       NOT NULL,  -- 置信度
  shared_id_count     INT64         NOT NULL,  -- 数量统计字段
  shared_id_types     ARRAY<STRING>,  -- 共享 ID 类型列表

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  updated_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  expired_at          TIMESTAMP,  -- 时间戳字段
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY mapping_type, brand_one_id
OPTIONS (
  description = 'DWS层：跨层映射表，记录三层 OneID 之间的映射关系'
);
```

**示例数据**：


| mapping_id | brand_one_id      | group_one_id      | global_one_id     | mapping_type    | confidence | shared_id_count | shared_id_types                            | merge_recommendation |
| ---------- | ----------------- | ----------------- | ----------------- | --------------- | ---------- | --------------- | ------------------------------------------ | -------------------- |
| CLM001     | BR:1001:000000099 | GR:2001:000000050 | NULL              | brand_to_group  | 0.85       | 5               | [phone, email, openid, unionid, device_id] | keep                 |
| CLM002     | BR:1001:000000101 | GR:2001:000000051 | NULL              | brand_to_group  | 0.80       | 3               | [phone, email, openid]                     | keep                 |
| CLM003     | BR:1001:000000099 | NULL              | GL:0000:000000030 | brand_to_global | 0.70       | 3               | [phone, email, unionid]                    | keep                 |


> **说明**：张三的品牌 OneID 映射到集团 OneID GR:2001:000000050（共享 5 个 ID 标识，置信度 0.85）和全局 OneID GL:0000:000000030（共享 3 个 ID 标识，置信度 0.70）。

### 5.8 dws_oneid_identity_original_id_mapping_df（OneID-原始业务ID映射表）

面向下游业务系统的权威映射表，用于按层级、业务域和来源系统查询 OneID 与原始业务 ID 的关系。该表弥补 `oneid_master.id_mappings` 只保存标准化身份标识、无法直接还原业务系统 UID 的问题。

> **定位说明**：该表是“当前/分区快照型”映射表，适合回答“某个 UID 当前归属哪个 OneID”。如果要展示某个 UID 的 OneID 变化过程，例如 `U001` 从 `BR:1001:000000100` 被融合到 `BR:1001:000000099`，应查询 5.9 的原始业务 ID 归属变更流水表，而不是只依赖本表。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_original_id_mapping_df`
(
  -- OneID 标识
  one_id              STRING        NOT NULL,  -- OneID 唯一标识
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global

  -- 业务隔离维度
  business_domain     STRING        NOT NULL,  -- ecommerce | content | app | delivery
  source_system       STRING        NOT NULL,  -- btc_trade | wechat_mini | app_ios | app_android | web_pc

  -- 原始业务 ID
  source_uid          STRING        NOT NULL,  -- 来源系统中的用户 UID
  original_id_type    STRING        NOT NULL,  -- ecommerce_uid | app_uid | wechat_uid | external_uid | account_id
  original_id_value   STRING        NOT NULL,  -- 原始业务 ID 值，保持源系统口径

  -- 关联到 OneID 的标准化身份标识
  normalized_id_type  STRING,                  -- phone | email | device_id | openid | unionid | cookie_id | third_party_id
  normalized_id_value STRING,                  -- 归一化后的标识值
  normalized_vertex_id STRING,                 -- id_type:id_value

  -- 映射质量
  confidence          FLOAT64       NOT NULL,  -- OneID 与原始业务 ID 的关联置信度
  mapping_basis       STRING        NOT NULL,  -- direct_binding | event_co_occurrence | cross_layer_guidance | manual_intervention
  is_primary          BOOL          NOT NULL,  -- 是否为该 OneID 在此业务域的主业务 ID
  status              STRING        NOT NULL,  -- active | deprecated | suspicious | removed

  -- 生命周期
  effective_from      TIMESTAMP     NOT NULL,  -- 生效开始时间
  effective_until     TIMESTAMP,  -- 生效结束时间
  first_seen_at       TIMESTAMP     NOT NULL,  -- 时间戳字段
  last_seen_at        TIMESTAMP     NOT NULL,  -- 时间戳字段

  -- 溯源
  source_event_ids    ARRAY<STRING>,  -- 溯源事件 ID 列表
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号
  created_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  updated_at          TIMESTAMP     NOT NULL,  -- 时间戳字段

  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, business_domain, source_system
OPTIONS (
  description = 'DWS层：OneID 与原始业务 ID 的权威映射表，供下游业务系统按层级和业务域查询'
);
```

**示例数据**：


| one_id            | scope | scope_id | business_domain | source_system | source_uid | original_id_type | original_id_value | normalized_id_type | normalized_id_value | confidence | mapping_basis       | is_primary |
| ----------------- | ----- | -------- | --------------- | ------------- | ---------- | ---------------- | ----------------- | ------------------ | ------------------- | ---------- | ------------------- | ---------- |
| BR:1001:000000099 | brand | 1001     | ecommerce       | btc_trade     | U001       | ecommerce_uid    | U001              | phone              | 13812345678         | 0.92       | direct_binding      | true       |
| BR:1001:000000099 | brand | 1001     | app             | app_ios       | APP001     | app_uid          | APP001            | device_id          | device_zhang_001    | 0.85       | event_co_occurrence | false      |
| BR:1001:000000099 | brand | 1001     | app             | wechat_mini   | WX001      | wechat_uid       | WX001             | unionid            | unionid_zhang001    | 0.95       | direct_binding      | false      |


### 5.9 dws_oneid_identity_original_id_change_di（原始业务ID-OneID归属变更流水表）

记录原始业务 ID 在 OneID 体系中的归属变化过程，用于回答“某个 UID 的 OneID 是如何变化的”。该表按变更事件增量写入，不存储每日未变化的快照；当前归属仍以 5.8 的映射表为准。

典型场景：

- **融合**：某个 UID 原归属 `source_one_id`，因 OneID 合并后归属到 `target_one_id`。
- **分裂**：某个 UID 从原 OneID 中被剥离，归属到新的 OneID。
- **解绑/重绑**：人工干预或规则调整导致 UID 与 OneID 的关系移除或重新绑定。
- **回滚**：单用户或任务级回滚导致 UID 归属恢复到历史 OneID。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_original_id_change_di`
(
  -- 变更标识
  change_id           STRING        NOT NULL,  -- 变更事件唯一标识
  change_type         STRING        NOT NULL,  -- bind | reassign | merge_retarget | split_reassign | unbind | rollback | manual_adjust

  -- OneID 层级
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global

  -- 业务隔离维度
  business_domain     STRING        NOT NULL,  -- ecommerce | content | app | delivery
  source_system       STRING        NOT NULL,  -- btc_trade | wechat_mini | app_ios | app_android | web_pc

  -- 原始业务 ID
  source_uid          STRING        NOT NULL,  -- 来源系统中的用户 UID
  original_id_type    STRING        NOT NULL,  -- ecommerce_uid | app_uid | wechat_uid | external_uid | account_id
  original_id_value   STRING        NOT NULL,  -- 原始业务 ID 值，保持源系统口径

  -- OneID 归属变化
  from_one_id         STRING,                  -- 变更前 OneID；首次绑定时为空
  to_one_id           STRING,                  -- 变更后 OneID；解绑时为空
  from_status         STRING,                  -- 变更前映射状态: active | deprecated | suspicious | removed
  to_status           STRING,                  -- 变更后映射状态: active | deprecated | suspicious | removed

  -- 变更原因与证据
  change_reason       STRING        NOT NULL,  -- 变更原因摘要
  decision_type       STRING,                  -- create | keep | merge | split | conflict_resolve | intervention | rollback
  mapping_basis       STRING,                  -- direct_binding | event_co_occurrence | cross_layer_guidance | manual_intervention
  confidence_before   FLOAT64,                 -- 变更前置信度
  confidence_after    FLOAT64,                 -- 变更后置信度
  evidence_vertex_ids ARRAY<STRING>,           -- 触发变更的标准化顶点列表

  -- 溯源
  decision_id         STRING,                  -- 关联 dws_oneid_identity_mapping_decision_di
  operation_id        STRING,                  -- 关联 merge/split/intervention/rollback 操作
  source_event_ids    ARRAY<STRING>,           -- 溯源事件 ID 列表
  batch_id            STRING        NOT NULL,  -- 计算或接入批次号

  -- 时间
  changed_at          TIMESTAMP     NOT NULL,  -- 变更发生时间
  created_at          TIMESTAMP     NOT NULL,  -- 记录创建时间

  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, source_system, source_uid
OPTIONS (
  description = 'DWS层：原始业务 ID 的 OneID 归属变更流水表，用于展示 UID 的 OneID 变化过程'
);
```

**示例数据**：


| change_id | change_type    | scope | scope_id | source_system | source_uid | original_id_type | original_id_value | from_one_id       | to_one_id         | decision_type | change_reason                     | batch_id     | dt         |
| --------- | -------------- | ----- | -------- | ------------- | ---------- | ---------------- | ----------------- | ----------------- | ----------------- | ------------- | --------------------------------- | ------------ | ---------- |
| CHG001    | bind           | brand | 1001     | btc_trade     | U001       | ecommerce_uid    | U001              | NULL              | BR:1001:000000100 | create        | 首次进入 OneID 体系，分配新 OneID           | 202606170001 | 2026-06-17 |
| CHG002    | merge_retarget | brand | 1001     | btc_trade     | U001       | ecommerce_uid    | U001              | BR:1001:000000100 | BR:1001:000000099 | merge         | WCC 连通分量命中多个历史 OneID，保留 000000099 | 202606180001 | 2026-06-18 |
| CHG003    | split_reassign | brand | 1001     | btc_trade     | U001       | ecommerce_uid    | U001              | BR:1001:000000099 | BR:1001:000000121 | split         | 人工确认家庭共用账号，UID 从原 OneID 中剥离       | 202606190001 | 2026-06-19 |


**常用查询**：

```sql
-- 查看某个来源 UID 的 OneID 变化过程
SELECT
  changed_at,
  change_type,
  from_one_id,
  to_one_id,
  decision_type,
  change_reason,
  batch_id
FROM `oneid_dws.dws_oneid_identity_original_id_change_di`
WHERE scope = 'brand'
  AND scope_id = '1001'
  AND source_system = 'btc_trade'
  AND source_uid = 'U001'
ORDER BY changed_at;
```

---

## 六、ADS 层 — 应用数据层（oneid_ads 数据集）

### 6.1 ads_oneid_gov_intervention_operation_di（干预操作表）

记录所有人工干预操作的全生命周期。

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_intervention_operation_di`
(
  -- 基础标识
  operation_id        STRING        NOT NULL,  -- 操作唯一标识
  operation_type      STRING        NOT NULL,  -- 操作类型

  -- 操作目标
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global
  target_one_ids      ARRAY<STRING>,  -- 目标 OneID 列表
  target_id_mappings  ARRAY<STRUCT<  -- 操作目标 ID 映射列表
    one_id            STRING,  -- OneID 唯一标识
    id_type           STRING,  -- ID 类型
    id_value          STRING  -- ID 值
  >>,

  -- 操作参数
  params              JSON,  -- 操作参数 JSON

  -- 审批流程
  status              STRING        NOT NULL,  -- 记录状态
  reviewed_by         ARRAY<STRING>,  -- 审核人列表

  -- Dry Run 结果
  dry_run_result      JSON,  -- Dry Run 结果 JSON

  -- 执行结果
  execution_result    JSON,  -- 执行结果 JSON

  -- 干预有效期
  effective_mode      STRING        NOT NULL,  -- 生效模式
  effective_from      TIMESTAMP     NOT NULL,  -- 生效开始时间
  effective_until     TIMESTAMP,  -- 生效结束时间

  -- 审计
  created_by          STRING        NOT NULL,  -- 创建人
  reason              STRING        NOT NULL,  -- 操作原因
  related_ticket      STRING,  -- 关联工单

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  updated_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  executed_at         TIMESTAMP,  -- 时间戳字段
  rolled_back_at      TIMESTAMP,  -- 时间戳字段
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, operation_type, status
OPTIONS (
  description = 'ADS层：干预操作表，记录人工干预 OneID 合并/拆分的操作'
);
```

**示例数据**：

假设运营发现张三的 openid_abc123 实际上是张三弟弟的账号（家庭共用设备），需要解绑。


| 字段                 | 值                                                                                                                                                                      |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| operation_id       | INTV_20260619_001                                                                                                                                                      |
| operation_type     | id_unbind                                                                                                                                                              |
| scope              | brand                                                                                                                                                                  |
| scope_id           | 1001                                                                                                                                                                   |
| target_one_ids     | [BR:1001:000000099]                                                                                                                                                    |
| target_id_mappings | [{one_id:"BR:1001:000000099", id_type:"openid", id_value:"openid_abc123"}]                                                                                             |
| status             | completed                                                                                                                                                              |
| reviewed_by        | [admin_wang]                                                                                                                                                           |
| effective_mode     | permanent                                                                                                                                                              |
| created_by         | operator_li                                                                                                                                                            |
| reason             | "openid_abc123 是张三弟弟的账号，属于家庭共用设备场景，需要解绑"                                                                                                                               |
| dry_run_result     | {summary:{total_affected_one_ids:1, updated_one_ids:1, cross_layer_impact:true}, risk_assessment:{risk_level:"low", risk_factors:[{factor:"影响范围小", severity:"info"}]}} |


### 6.8 ads_oneid_gov_cross_layer_guidance_config_df（跨层合并指导配置表）

控制品牌/集团层是否引用上层 OneID 结果作为合并指导，以及发生冲突时采用本层优先还是上层优先。

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_cross_layer_guidance_config_df`
(
  -- 配置标识
  config_id           STRING        NOT NULL,  -- 配置唯一标识

  -- 被指导的目标层级
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global
  brand_id            STRING,  -- 品牌 ID
  group_id            STRING,  -- 集团 ID

  -- 引用的上层结果
  enabled             BOOL          NOT NULL,  -- 是否启用
  reference_scope     STRING        NOT NULL,  -- group | global
  reference_scope_id  STRING,  -- 引用的上层范围 ID
  guidance_mode       STRING        NOT NULL,  -- recommendation_only | auto_apply | require_review
  priority_strategy   STRING        NOT NULL,  -- local_first | upper_first | higher_confidence_first

  -- 生效阈值
  min_merge_confidence FLOAT64      NOT NULL,  -- 最低合并置信度
  min_shared_id_count INT64         NOT NULL,  -- 最低共享 ID 数量
  allowed_id_types    ARRAY<STRING>,  -- 允许的 ID 类型列表
  max_merge_oneid_count INT64,                -- 单次建议最多合并的下层 OneID 数
  require_manual_review_above_count INT64,  -- 超过该合并数量需人工审核

  -- 冲突处理
  conflict_policy     STRING        NOT NULL,  -- keep_local | follow_upper | review
  risk_level          STRING        NOT NULL,  -- 风险等级

  -- 生命周期
  is_active           BOOL          NOT NULL,  -- 是否有效
  effective_from      TIMESTAMP     NOT NULL,  -- 生效开始时间
  effective_until     TIMESTAMP,  -- 生效结束时间
  created_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  updated_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  created_by          STRING        NOT NULL,  -- 创建人

  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, scope_id, enabled
OPTIONS (
  description = 'ADS层：跨层合并指导配置表，按品牌/集团控制是否引用上层OneID指导本层合并'
);
```

### 6.10 ads_oneid_gov_audit_log_di（审计日志表）

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_audit_log_di`
(
  -- 日志标识
  log_id              STRING        NOT NULL,  -- 日志唯一标识

  -- 操作信息
  operation_type      STRING        NOT NULL,  -- 操作类型
  operation_id        STRING,  -- 操作唯一标识
  batch_id            STRING,  -- 计算或接入批次号

  -- 变更目标
  scope               STRING        NOT NULL,  -- OneID 层级: brand | group | global
  scope_id            STRING        NOT NULL,  -- 层级范围 ID: 品牌ID / 集团ID / global
  affected_one_ids    ARRAY<STRING>,  -- 受影响 OneID 列表

  -- 变更前后快照
  before_snapshot     JSON,  -- 变更前快照 JSON
  after_snapshot      JSON,  -- 变更后快照 JSON
  diff_summary        STRING,  -- 变更摘要

  -- 操作人
  operator_type       STRING        NOT NULL,  -- 操作人类型
  operator_id         STRING        NOT NULL,  -- 操作人 ID
  operator_name       STRING,  -- 操作人姓名

  -- 原因与审批
  reason              STRING        NOT NULL,  -- 操作原因
  approved_by         ARRAY<STRING>,  -- 审批人列表

  -- 时间
  created_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY scope, operation_type, operator_type
OPTIONS (
  description = 'ADS层：审计日志表',
  partition_expiration_days = 365
);
```

### 6.11 ads_oneid_gov_rollback_operation_di（回滚操作表）

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_rollback_operation_di`
(
  -- 回滚标识
  rollback_id         STRING        NOT NULL,  -- 回滚操作唯一标识
  rollback_type       STRING        NOT NULL,  -- 回滚类型

  -- 回滚目标
  target_one_id       STRING,  -- 目标 OneID
  target_operation_id STRING,  -- 目标操作 ID
  target_snapshot_date DATE,  -- 目标快照日期

  -- 回滚状态
  status              STRING        NOT NULL,  -- 记录状态
  dry_run_result      JSON,  -- Dry Run 结果 JSON

  -- 审批
  requested_by        STRING        NOT NULL,  -- 申请人
  approved_by         ARRAY<STRING>,  -- 审批人列表
  reason              STRING        NOT NULL,  -- 操作原因

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,  -- 时间戳字段
  executed_at         TIMESTAMP,  -- 时间戳字段
  completed_at        TIMESTAMP,  -- 时间戳字段
  
  -- 分区字段
  dt                  DATE          NOT NULL  -- 数据分区日期
)
PARTITION BY dt
CLUSTER BY rollback_type, status
OPTIONS (
  description = 'ADS层：回滚操作表'
);
```

---

## 七、表关系总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DWD 层 (oneid_dwd)                                │
│                                                                     │
│  dwd_trade_user_user_profile_wide_di ──┐                            │
│  (上游已整理用户明细)                    │                            │
│                                        ├──→ OneID 清洗/图构建          │
│  dwd_trade_user_association_event_di ──┘   (归一化、过滤、生成点边)     │
│  (上游已整理关联事件)                       │                          │
│                                         │                            │
│  ads_oneid_gov_cleaning_rules_df ───────┘                            │
│  (清洗规则配置)                                                       │
│                                                                     │
│  dwd_oneid_graph_vertex_df ──────────┐                              │
│  (点表)                               │                              │
│                                      ├──→ WCC 算法                   │
│  dwd_oneid_graph_edge_df ────────────┘   (连通分量计算)               │
│  (边表)                               │                              │
│                                       │                              │
│  dwd_oneid_graph_cleaning_log_di      │                              │
│  (清洗日志)                            │                              │
│                                                                     │
└────────────────────────────────┼────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    DWS 层 (oneid_dws)                                │
│                                                                     │
│  dws_oneid_identity_wcc_result_df                                   │
│  (WCC 归属结果：每个 vertex 的 component 归属 + 度数统计)              │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────┐                    │
│  │ 阶段4：ID 映射决策与冲突消解                    │                    │
│  │                                             │                    │
│  │  dws_oneid_identity_mapping_decision_di     │                    │
│  │  (候选 vs 历史比对，决策：create/keep/merge/split)                 │
│  │         │                                   │                    │
│  │         ├──→ merge_operation_di (融合操作)    │                    │
│  │         ├──→ split_operation_di (分裂操作)    │                    │
│  │         └──→ conflict_resolution_di (冲突消解)│                    │
│  └─────────────────────────────────────────────┘                    │
│         │                                                           │
│         ▼                                                           │
│  dws_oneid_identity_oneid_master_df ◄──────────────────┐            │
│  (OneID 主表)                                          │            │
│         │                                              │            │
│         │ (各层 OneID)                                  │            │
│         ▼                                              │            │
│  dws_oneid_identity_cross_layer_mapping_df ────────────┘            │
│  (跨层映射表)                                                        │
│         │                                                           │
│         │ (合并建议)                                                 │
│         ▼                                                           │
│  触发干预 or 自动合并                                                 │
│                                                                     │
│  dws_oneid_identity_original_id_mapping_df                           │
│  (OneID-原始业务ID映射，供下游业务域查询)                              │
│                                                                     │
│  dws_oneid_identity_original_id_change_di                            │
│  (原始业务ID归属变更流水，展示 UID 的 OneID 变化过程)                    │
│                                                                     │
└────────────────────────────────┼────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ADS 层 (oneid_ads)                                │
│                                                                     │
│  ads_oneid_gov_intervention_operation_di ──→ intervention_rule_df   │
│    │                              │                                 │
│    │                              │ (持久化规则，参与下次批处理)       │
│    ├──→ approval_policy_df        │                                 │
│    ├──→ id_list_df                │                                 │
│    ├──→ id_type_weight_config_df  │                                 │
│    ├──→ cross_layer_guidance_config_df                              │
│    ├──→ domain_view_config_df                                        │
│    │                              │                                 │
│    ├──→ audit_log_di              │                                 │
│    │                              │                                 │
│    ├──→ rollback_operation_di     │                                 │
│    │                              │                                 │
│    └──→ backfill_task_di          │                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 九、OneData 建模要点

### 9.1 分层职责


| 层级      | 职责               | 数据特征           |
| ------- | ---------------- | -------------- |
| **ODS** | 保持源数据原貌，不做业务逻辑处理 | 原始数据、可追溯、有生命周期 |
| **DWD** | 清洗、规范化、维度退化      | 明细数据、质量可控、可重建  |
| **DWS** | 按主题汇总、轻度聚合       | 汇总数据、面向分析、核心资产 |
| **ADS** | 面向应用的数据服务        | 应用数据、高度定制、直接消费 |


### 9.2 数据流向

```
DWD（上游已整理明细数据）
  ↓ 汇总、聚合
DWS（汇总数据）
  ↓ 应用定制
ADS（应用数据）
```

### 9.3 命名规范遵循

- ✅ 分层前缀：`ods_`, `dwd_`, `dws_`, `ads_`
- ✅ 业务域标识：`trade`（交易域）, `oneid`（OneID域）, `gov`（治理域）
- ✅ 数据域标识：`user`（用户域）, `graph`（图计算域）, `identity`（身份域）
- ✅ 增量标识：`_di`（日增量）, `_df`（日全量）
- ✅ 表名格式：`{层级}_{业务域}_{数据域}_{表名}_{增量标识}`

