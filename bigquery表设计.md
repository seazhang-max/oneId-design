# OneID 系统 BigQuery 表设计（OneData 方法论）

## 一、数据分层架构

采用 OneData 方法论的标准四层架构：


| 层级        | 命名前缀   | 用途                   | 数据集         |
| --------- | ------ | -------------------- | ----------- |
| **ODS 层** | `ods_` | 操作数据层：原始数据接入，保持源数据原貌 | `oneid_ods` |
| **DWD 层** | `dwd_` | 明细数据层：清洗后的明细数据，维度退化  | `oneid_dwd` |
| **DWS 层** | `dws_` | 汇总数据层：按主题汇总，轻度聚合     | `oneid_dws` |
| **ADS 层** | `ads_` | 应用数据层：面向应用的数据服务      | `oneid_ads` |


## 二、命名规范

### 2.1 表名格式

```
{层级}_{业务域}_{数据域}_{表名}_{增量标识}
```

**示例**：

- `ods_trade_user_user_profile_wide_di`：ODS层_交易域_用户域_用户信息宽表_日增量
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


| 业务域    | 缩写      | 说明           |
| ------ | ------- | ------------ |
|        |         |              |
| OneID域 | `oneid` | OneID 系统核心数据 |
| 治理域    | `gov`   | 数据治理与运维数据    |


---

## 三、dwd 层 — 操作数据层（oneid_dwd 数据集）

### 3.1 dwd_trade_user_user_profile_wide_di（用户信息宽表）

从各业务系统接入的用户 ID 标识数据，保持源数据原貌。

```sql
CREATE OR REPLACE TABLE `dwd_trade_user_user_profile_wide_di`
(
  -- 数据来源标识
  source_system       STRING        NOT NULL,  -- 来源系统: btc_trade | wechat_mini | app_ios | app_android | web_pc
  source_uid          STRING        NOT NULL,  -- 来源系统的用户 UID
  
  brand_id            STRING,                  -- 品牌ID
  group_id            STRING,                  -- 集团ID
  
  -- 用户 ID 标识（原始值，未归一化）
  phone_raw           STRING,                  -- 原始手机号
  email_raw           STRING,                  -- 原始邮箱
  device_id_raw       STRING,                  -- 原始设备 ID
  openid_raw          STRING,                  -- 原始 OpenID
  unionid_raw         STRING,                  -- 原始 UnionID
  cookie_id_raw       STRING,                  -- 原始 Cookie ID
  third_party_ids     ARRAY<STRUCT<
    id_type           STRING,                  -- 第三方 ID 类型
    id_value          STRING,                  -- 第三方 ID 值
    source_system     STRING                   -- 来源系统
  >>,                                          -- 其他第三方 ID
  
  -- 用户属性
  nickname            STRING,
  gender              STRING,
  birthday            DATE,
  membership_level    STRING,
  
  -- 时间戳
  first_seen_at       TIMESTAMP     NOT NULL,
  last_seen_at        TIMESTAMP     NOT NULL,
  
  -- 数据质量标记
  data_quality_flag   STRING,                  -- clean | suspicious | test_data | bot
  
  -- 元数据
  batch_id            STRING        NOT NULL,
  ingest_timestamp    TIMESTAMP     NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL   -- 数据日期
)
PARTITION BY dt
CLUSTER BY source_system, scope, scope_id
OPTIONS (
  description = 'ODS层：用户信息宽表，从各业务系统接入的用户 ID 标识数据',
  partition_expiration_days = 180
);
```

**示例数据**：


| source_system | source_uid | scope | scope_id | brand_id | phone_raw         | email_raw                                           | openid_raw    | unionid_raw      | device_id_raw    | cookie_id_raw |
| ------------- | ---------- | ----- | -------- | -------- | ----------------- | --------------------------------------------------- | ------------- | ---------------- | ---------------- | ------------- |
| btc_trade     | U001       | brand | 1001     | 1001     | 13812345678       | [ZhangSan@Example.com](mailto:ZhangSan@Example.com) | NULL          | NULL             | NULL             | NULL          |
| wechat_mini   | WX001      | brand | 1001     | 1001     | +86 138 1234 5678 | NULL                                                | openid_def456 | unionid_zhang001 | NULL             | NULL          |
| app_ios       | APP001     | brand | 1001     | 1001     | 13812345678       | NULL                                                | openid_abc123 | NULL             | device_zhang_001 | NULL          |
| btc_trade     | U002       | brand | 1001     | 1001     | 13987654321       | [lisi@example.com](mailto:lisi@example.com)         | NULL          | NULL             | NULL             | NULL          |
| app_ios       | APP002     | brand | 1001     | 1001     | 13987654321       | NULL                                                | openid_ghi012 | NULL             | device_lisi_001  | NULL          |


> **说明**：张三在三个系统有记录（btc_trade、wechat_mini、app_ios），手机号格式不统一（有空格、+86 前缀、大小写）。李四在两个系统有记录。

### 3.2 dwd_user_association_event_di（关联事件表）

记录用户行为事件中多个 ID 标识的共现关系。

```sql
CREATE OR REPLACE TABLE `dwd_trade_user_association_event_di`
(
  -- 事件标识
  event_id            STRING        NOT NULL,
  
  -- 数据来源
  source_system       STRING        NOT NULL,
  event_type          STRING        NOT NULL,  -- login | register | order | browse | payment
  
  -- 层级与范围
  brand_id            STRING,
  group_id            STRING,
  
  -- 事件中包含的 ID 标识（共现关系）
  uid                 STRING,
  phone_raw           STRING,
  email_raw           STRING,
  device_id_raw       STRING,
  openid_raw          STRING,
  unionid_raw         STRING,
  cookie_id_raw       STRING,
  
  -- 事件上下文
  session_id          STRING,
  ip_address          STRING,
  user_agent          STRING,
  
  -- 时间戳
  event_timestamp     TIMESTAMP     NOT NULL,
  
  -- 元数据
  batch_id            STRING        NOT NULL,
  ingest_timestamp    TIMESTAMP     NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY source_system, event_type, scope
OPTIONS (
  description = 'ODS层：关联事件表，记录用户行为事件中 ID 标识的共现关系',
  partition_expiration_days = 180
);
```

**示例数据**：


| event_id | source_system | event_type | scope | scope_id | uid    | phone_raw   | openid_raw    | unionid_raw      | device_id_raw    | cookie_id_raw      |
| -------- | ------------- | ---------- | ----- | -------- | ------ | ----------- | ------------- | ---------------- | ---------------- | ------------------ |
| EVT001   | app_ios       | login      | brand | 1001     | APP001 | 13812345678 | openid_abc123 | NULL             | device_zhang_001 | NULL               |
| EVT002   | wechat_mini   | login      | brand | 1001     | WX001  | 13812345678 | openid_def456 | unionid_zhang001 | NULL             | NULL               |
| EVT003   | app_ios       | login      | brand | 1001     | APP002 | 13987654321 | openid_ghi012 | NULL             | device_lisi_001  | NULL               |
| EVT004   | web_pc        | browse     | brand | 1001     | NULL   | NULL        | NULL          | NULL             | NULL             | cookie_public_wifi |
| EVT005   | web_pc        | browse     | brand | 1001     | NULL   | NULL        | NULL          | NULL             | NULL             | cookie_public_wifi |


> **说明**：EVT001 是张三 APP 登录，同时出现 phone+openid+device_id（三者将建立边）。EVT004/EVT005 是张三和李四在同一 WiFi 下浏览，共享 cookie_public_wifi。

---

## 四、DWD 层 — 明细数据层（oneid_dwd 数据集）

### 4.1 dwd_oneid_graph_vertex_df（点表）

清洗后的图计算点表，每个唯一的 ID 标识值作为一个顶点。

```sql
CREATE OR REPLACE TABLE `oneid_dwd.dwd_oneid_graph_vertex_df`
(
  -- 基础标识
  vertex_id         STRING        NOT NULL,  -- 唯一标识 = id_type + ":" + id_value
  id_type           STRING        NOT NULL,  -- phone | email | device_id | openid | unionid | cookie_id | third_party_id
  id_value          STRING        NOT NULL,  -- 归一化后的标识值

  -- 来源与观测
  source_system     STRING        NOT NULL,
  first_seen_at     TIMESTAMP     NOT NULL,
  last_seen_at      TIMESTAMP     NOT NULL,
  observation_count INT64         NOT NULL,

  -- 分区与层级
  scope             STRING        NOT NULL,
  scope_id          STRING        NOT NULL,

  -- 元数据
  batch_id          STRING        NOT NULL,
  
  -- 分区字段
  dt                DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY scope, scope_id, id_type
OPTIONS (
  description = 'DWD层：图计算点表，每个唯一的 ID 标识值作为一个顶点',
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

```sql
CREATE OR REPLACE TABLE `oneid_dwd.dwd_oneid_graph_edge_df`
(
  -- 边的两端
  src                 STRING        NOT NULL,  -- 源顶点 vertex_id
  dst                 STRING        NOT NULL,  -- 目标顶点 vertex_id

  -- 置信度与共现
  confidence          FLOAT64       NOT NULL,  -- 边置信度（两个 ID 标识属于同一自然人的可信程度）
  co_occurrence_count INT64         NOT NULL,
  first_co_at         TIMESTAMP     NOT NULL,
  last_co_at          TIMESTAMP     NOT NULL,

  -- 来源
  source_system       STRING        NOT NULL,

  -- 分区与层级
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,

  -- 元数据
  batch_id            STRING        NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY scope, scope_id
OPTIONS (
  description = 'DWD层：图计算边表，同一用户共现的 ID 标识之间的关联边',
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
  log_id              STRING        NOT NULL,
  
  -- 关联的原始数据
  source_table        STRING        NOT NULL,  -- ods_trade_user_user_profile_wide_di | ods_trade_user_association_event_di
  source_id           STRING        NOT NULL,
  
  -- 清洗动作
  action_type         STRING        NOT NULL,  -- normalized | filtered | transformed | kept
  
  -- 归一化详情
  normalization_rules ARRAY<STRUCT<
    field_name        STRING,
    rule_applied      STRING,
    before_value      STRING,
    after_value       STRING
  >>,
  
  -- 过滤详情
  filter_reason       STRING,
  filter_rule_id      STRING,
  
  -- 层级
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,
  
  -- 元数据
  batch_id            STRING        NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
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
| LOG001 | ods_trade_user_user_profile_wide_di | U001      | normalized  | [{field:"phone_raw", rule:"remove_prefix_86", before:"+86 138 1234 5678", after:"13812345678"}, {field:"email_raw", rule:"lowercase", before:"[ZhangSan@Example.com](mailto:ZhangSan@Example.com)", after:"[zhangsan@example.com](mailto:zhangsan@example.com)"}] | NULL          |
| LOG002 | ods_trade_user_association_event_di | EVT004    | filtered    | NULL                                                                                                                                                                                                                                                              | public_id     |
| LOG003 | ods_trade_user_association_event_di | EVT005    | filtered    | NULL                                                                                                                                                                                                                                                              | public_id     |


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
  id_value            STRING        NOT NULL,  -- 归一化后的 ID 值

  -- WCC 归属
  component_id        STRING        NOT NULL,  -- 所属连通分量 ID
  component_size      INT64         NOT NULL,  -- 该连通分量的总顶点数（用于识别超大分量）

  -- 该顶点的边统计（辅助排查）
  degree              INT64         NOT NULL,  -- 该顶点的度数（关联边数量）
  neighbor_count      INT64         NOT NULL,  -- 邻居顶点数量
  total_edge_weight   FLOAT64       NOT NULL,  -- 关联边的权重总和

  -- 分区与层级
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,

  -- 元数据
  batch_id            STRING        NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY scope, scope_id, component_id
OPTIONS (
  description = 'DWS层：WCC归属结果表，每个vertex的连通分量归属，支持JOIN排查',
  partition_expiration_days = 90
);
```

**示例数据**：


| vertex_id                                                 | id_type   | id_value                                            | component_id | component_size | degree | neighbor_count | total_edge_weight |
| --------------------------------------------------------- | --------- | --------------------------------------------------- | ------------ | -------------- | ------ | -------------- | ----------------- |
| phone:13812345678                                         | phone     | 13812345678                                         | COMP_ZHANG   | 6              | 5      | 5              | 4.25              |
| email:[zhangsan@example.com](mailto:zhangsan@example.com) | email     | [zhangsan@example.com](mailto:zhangsan@example.com) | COMP_ZHANG   | 6              | 1      | 1              | 0.90              |
| openid:openid_abc123                                      | openid    | openid_abc123                                       | COMP_ZHANG   | 6              | 1      | 1              | 0.80              |
| openid:openid_def456                                      | openid    | openid_def456                                       | COMP_ZHANG   | 6              | 2      | 2              | 1.80              |
| unionid:unionid_zhang001                                  | unionid   | unionid_zhang001                                    | COMP_ZHANG   | 6              | 2      | 2              | 1.80              |
| device:device_zhang_001                                   | device_id | device_zhang_001                                    | COMP_ZHANG   | 6              | 1      | 1              | 0.80              |
| phone:13987654321                                         | phone     | 13987654321                                         | COMP_LI      | 4              | 3      | 3              | 2.50              |
| email:[lisi@example.com](mailto:lisi@example.com)         | email     | [lisi@example.com](mailto:lisi@example.com)         | COMP_LI      | 4              | 1      | 1              | 0.90              |
| openid:openid_ghi012                                      | openid    | openid_ghi012                                       | COMP_LI      | 4              | 1      | 1              | 0.80              |
| device:device_lisi_001                                    | device_id | device_lisi_001                                     | COMP_LI      | 4              | 1      | 1              | 0.80              |


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
  candidate_member_count INT64      NOT NULL,
  
  -- 匹配的历史 OneID（可能有多个）
  matched_history_one_ids ARRAY<STRING>,       -- 匹配到的历史 OneID 列表
  matched_history_count INT64,                 -- 匹配到的历史 OneID 数量
  
  -- 相似度计算
  best_match_one_id   STRING,                  -- 最佳匹配的历史 OneID
  best_match_jaccard  FLOAT64,                 -- 最佳匹配的 Jaccard 相似度
  all_match_scores    ARRAY<STRUCT<
    history_one_id    STRING,                  -- 历史 OneID
    jaccard_score     FLOAT64,                 -- Jaccard 相似度
    shared_id_count   INT64,                   -- 共享 ID 标识数量
    shared_id_types   ARRAY<STRING>            -- 共享的 ID 类型
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
  intervention_applied BOOL         NOT NULL,  -- 是否应用了干预规则
  intervention_rule_ids ARRAY<STRING>,         -- 应用的干预规则 ID 列表
  
  -- 层级
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,
  
  -- 元数据
  batch_id            STRING        NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY scope, scope_id, decision_type
OPTIONS (
  description = 'DWS层：ID映射决策表，记录候选OneID与历史OneID的比对和决策过程',
  partition_expiration_days = 90
);
```

**示例数据**：


| decision_id | candidate_component_id | candidate_member_ids                                                                | decision_type | best_match_one_id | best_match_jaccard | merge_target_one_id | merge_source_one_ids | new_one_id        | intervention_applied |
| ----------- | ---------------------- | ----------------------------------------------------------------------------------- | ------------- | ----------------- | ------------------ | ------------------- | -------------------- | ----------------- | -------------------- |
| DEC001      | COMP_ZHANG             | [phone:138..., email:zhang..., openid:abc, openid:def, unionid:zhang, device:zhang] | merge         | BR:1001:000000099 | 0.33               | BR:1001:000000099   | [BR:1001:000000100]  | NULL              | false                |
| DEC002      | COMP_LI                | [phone:139..., email:lisi..., openid:ghi, device:lisi]                              | create        | NULL              | 0.00               | NULL                | NULL                 | BR:1001:000000101 | false                |


> **说明**：
>
> - **DEC001**：COMP_ZHANG 与历史 OneID BR:1001:000000099（原有 phone+email）匹配，但新增了 openid 和 unionid，Jaccard=0.33（2/6），触发**融合** — 将 BR:1001:000000100（原有 openid+unionid）合并到 BR:1001:000000099
> - **DEC002**：COMP_LI 与所有历史 OneID 无匹配，**新建** BR:1001:000000101

### 5.3 dws_oneid_identity_merge_operation_di（融合操作表）

记录 OneID 融合操作的详细信息。当多个历史 OneID 被合并为一个时，记录融合过程。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_merge_operation_di`
(
  -- 操作标识
  operation_id        STRING        NOT NULL,
  
  -- 融合目标
  target_one_id       STRING        NOT NULL,  -- 融合后保留的 OneID
  source_one_ids      ARRAY<STRING> NOT NULL,  -- 被融合的源 OneID 列表
  source_one_id_count INT64         NOT NULL,
  
  -- 融合原因
  merge_reason        STRING        NOT NULL,  -- merge_reason 说明
  trigger_type        STRING        NOT NULL,  -- auto | manual（自动融合 or 人工干预）
  
  -- 融合前的 OneID 状态快照
  before_snapshot     ARRAY<STRUCT<
    one_id            STRING,                  -- 源 OneID
    status            STRING,                  -- 融合前状态
    id_mapping_count  INT64,                   -- ID 标识数量
    id_mappings_summary ARRAY<STRING>          -- ID 标识摘要
  >>,
  
  -- 融合后的 OneID 状态
  after_id_mapping_count INT64,                -- 融合后的 ID 标识数量
  after_id_mappings   ARRAY<STRUCT<
    id_type           STRING,
    id_value          STRING,
    source_system     STRING,
    is_primary        BOOL,
    weight            FLOAT64
  >>,
  
  -- 属性融合冲突
  attribute_conflicts ARRAY<STRUCT<
    field_name        STRING,                  -- 冲突字段（如 gender, phone）
    source_values     ARRAY<STRING>,           -- 各源 OneID 的值
    resolved_value    STRING,                  -- 融合后采用的值
    resolution_rule   STRING                   -- 冲突解决规则（如 high_priority_override）
  >>,
  
  -- 跨层影响
  cross_layer_impact  BOOL         NOT NULL,  -- 是否影响跨层映射
  affected_cross_layer_mappings INT64,         -- 受影响的跨层映射记录数
  
  -- 层级
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,
  
  -- 元数据
  batch_id            STRING        NOT NULL,
  decision_id         STRING,                  -- 关联的决策 ID
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY scope, scope_id, trigger_type
OPTIONS (
  description = 'DWS层：融合操作表，记录OneID融合过程的详细信息',
  partition_expiration_days = 90
);
```

**示例数据**：


| operation_id | target_one_id     | source_one_ids      | source_one_id_count | merge_reason                          | trigger_type | after_id_mapping_count | attribute_conflicts                                                                                                                                                                                                                                 | cross_layer_impact |
| ------------ | ----------------- | ------------------- | ------------------- | ------------------------------------- | ------------ | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| MERGE001     | BR:1001:000000099 | [BR:1001:000000100] | 1                   | WCC 连通分量包含两个历史 OneID 的成员，Jaccard=0.33 | auto         | 6                      | [{field:"email", source_values:["[zhangsan@old.com](mailto:zhangsan@old.com)","[zhangsan@example.com](mailto:zhangsan@example.com)"], resolved_value:"[zhangsan@example.com](mailto:zhangsan@example.com)", resolution_rule:"time_decay_override"}] | false              |


> **说明**：MERGE001 将 BR:1001:000000100 融合到 BR:1001:000000099，保留目标 OneID。email 字段存在冲突（旧值 [zhangsan@old.com](mailto:zhangsan@old.com) vs 新值 [zhangsan@example.com](mailto:zhangsan@example.com)），按时间衰减规则采用新值。

### 5.4 dws_oneid_identity_split_operation_di（分裂操作表）

记录 OneID 分裂操作的详细信息。当一个历史 OneID 被拆分为多个新 OneID 时，记录分裂过程。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_split_operation_di`
(
  -- 操作标识
  operation_id        STRING        NOT NULL,
  
  -- 分裂目标
  source_one_id       STRING        NOT NULL,  -- 被分裂的源 OneID
  target_one_ids      ARRAY<STRING> NOT NULL,  -- 分裂后的新 OneID 列表
  target_one_id_count INT64         NOT NULL,
  
  -- 分裂原因
  split_reason        STRING        NOT NULL,  -- 分裂原因说明
  trigger_type        STRING        NOT NULL,  -- auto | manual（自动分裂 or 人工干预）
  
  -- 分裂前的 OneID 状态快照
  before_id_mapping_count INT64,                -- 分裂前的 ID 标识数量
  before_id_mappings  ARRAY<STRUCT<
    id_type           STRING,
    id_value          STRING,
    source_system     STRING,
    is_primary        BOOL,
    weight            FLOAT64
  >>,
  
  -- 分裂后的 OneID 状态
  after_snapshot      ARRAY<STRUCT<
    one_id            STRING,                  -- 新 OneID
    id_mapping_count  INT64,                   -- ID 标识数量
    id_mappings       ARRAY<STRUCT<
      id_type         STRING,
      id_value        STRING,
      source_system   STRING,
      is_primary      BOOL
    >>
  >>,
  
  -- 分裂依据
  split_basis         STRING,                  -- 分裂依据（如 wcc_recompute | manual_intervention）
  component_ids       ARRAY<STRING>,           -- 如果基于 WCC 重算，记录新的分量 ID
  
  -- 跨层影响
  cross_layer_impact  BOOL         NOT NULL,
  affected_cross_layer_mappings INT64,
  
  -- 层级
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,
  
  -- 元数据
  batch_id            STRING        NOT NULL,
  decision_id         STRING,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY scope, scope_id, trigger_type
OPTIONS (
  description = 'DWS层：分裂操作表，记录OneID分裂过程的详细信息',
  partition_expiration_days = 90
);
```

**示例数据**：


| operation_id | source_one_id     | target_one_ids                         | target_one_id_count | split_reason | trigger_type | split_basis         | cross_layer_impact |
| ------------ | ----------------- | -------------------------------------- | ------------------- | ------------ | ------------ | ------------------- | ------------------ |
| SPLIT001     | BR:1001:000000102 | [BR:1001:000000103, BR:1001:000000104] | 2                   | 家庭共用账号被错误合并  | manual       | manual_intervention | true               |


> **说明**：SPLIT001 将 BR:1001:000000102 分裂为两个新 OneID，因为发现是家庭共用账号（夫妻两人共用一个手机号）。

### 5.5 dws_oneid_identity_conflict_resolution_di（冲突消解表）

记录阶段4中跨分量冲突的消解过程。当一个 ID 标识同时出现在多个候选 OneID 中时，需要消解冲突。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_conflict_resolution_di`
(
  -- 冲突标识
  conflict_id         STRING        NOT NULL,
  
  -- 冲突的 ID 标识
  conflicting_id_type STRING        NOT NULL,  -- 冲突的 ID 类型
  conflicting_id_value STRING       NOT NULL,  -- 冲突的 ID 值
  
  -- 冲突的候选 OneID 列表
  candidate_one_ids   ARRAY<STRING> NOT NULL,  -- 包含该 ID 标识的候选 OneID 列表
  candidate_count     INT64         NOT NULL,
  
  -- 冲突消解规则
  resolution_rule     STRING        NOT NULL,  -- 消解规则：high_weight_priority | recent_activity_priority | intervention_override
  resolution_detail   STRING,                  -- 消解详情说明
  
  -- 消解结果
  winner_one_id       STRING        NOT NULL,  -- 获胜的 OneID（保留该 ID 标识）
  loser_one_ids       ARRAY<STRING>,           -- 失败的 OneID（移除该 ID 标识）
  
  -- 干预规则影响
  intervention_applied BOOL         NOT NULL,
  intervention_rule_id STRING,                  -- 应用的干预规则 ID
  
  -- 层级
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,
  
  -- 元数据
  batch_id            STRING        NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY scope, scope_id, resolution_rule
OPTIONS (
  description = 'DWS层：冲突消解表，记录跨分量冲突的消解过程',
  partition_expiration_days = 90
);
```

**示例数据**：


| conflict_id | conflicting_id_type | conflicting_id_value | candidate_one_ids         | candidate_count | resolution_rule      | resolution_detail                                  | winner_one_id | loser_one_ids | intervention_applied | intervention_rule_id |
| ----------- | ------------------- | -------------------- | ------------------------- | --------------- | -------------------- | -------------------------------------------------- | ------------- | ------------- | -------------------- | -------------------- |
| CONF001     | phone               | 13812345678          | [COMP_ZHANG, COMP_TEMP_X] | 2               | high_weight_priority | 手机号 13812345678 同时出现在两个候选分量中，按权重优先规则分配给 COMP_ZHANG | COMP_ZHANG    | [COMP_TEMP_X] | false                | NULL                 |


> **说明**：手机号 13812345678 同时出现在 COMP_ZHANG 和一个临时分量 COMP_TEMP_X 中（来自实时链路的临时 OneID），按权重优先规则分配给 COMP_ZHANG。

### 5.6 dws_oneid_identity_oneid_master_df（OneID 主表）

OneID 的权威数据源，包含基础标识、ID 映射、物理属性和 AI 画像。

```sql
CREATE OR REPLACE TABLE `oneid_dws.dws_oneid_identity_oneid_master_df`
(
  -- 基础标识
  one_id              STRING        NOT NULL,
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,
  status              STRING        NOT NULL,  -- active | frozen | merged | split
  version             INT64         NOT NULL,

  -- ID 映射
  id_mappings         ARRAY<STRUCT<
    id_type           STRING,
    id_value          STRING,
    source_system     STRING,
    is_primary        BOOL,
    weight            FLOAT64,
    first_seen_at     TIMESTAMP,
    last_seen_at      TIMESTAMP,
    status            STRING
  >>,

  -- 物理属性
  physical_info       STRUCT<
    nickname            STRING,
    gender              STRING,
    birthday            DATE,
    avatar_url          STRING,
    phone               STRING,
    email               STRING,
    registered_channels ARRAY<STRING>,
    first_register_at   TIMESTAMP,
    membership_level    STRING,
    membership_score    INT64,
    lifetime_order_count  INT64,
    lifetime_order_amount NUMERIC,
    last_order_at         TIMESTAMP,
    info_version        INT64,
    attribute_sources   JSON
  >,

  -- AI 属性
  ai_profile          STRUCT<
    lifecycle_stage     STRING,
    lifecycle_score     FLOAT64,
    purchase_preference ARRAY<STRUCT<
      tag_name          STRING,
      score             FLOAT64,
      source            STRING,
      decay_factor      FLOAT64
    >>,
    category_affinity   JSON,
    price_sensitivity   FLOAT64,
    brand_loyalty       FLOAT64,
    churn_probability     FLOAT64,
    purchase_intent       FLOAT64,
    predicted_lifetime_value NUMERIC,
    customer_segment      STRING,
    activity_preference   ARRAY<STRING>,
    model_version         STRING,
    computed_at           TIMESTAMP,
    next_recompute_at     TIMESTAMP
  >,

  -- 溯源与治理
  source              STRING        NOT NULL,
  batch_id            STRING        NOT NULL,
  confidence          FLOAT64       NOT NULL,

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,
  updated_at          TIMESTAMP     NOT NULL,
  merged_into         STRING,
  split_from          STRING,
  expired_at          TIMESTAMP,
  
  -- 分区字段
  dt                  DATE          NOT NULL
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
  mapping_id          STRING        NOT NULL,

  -- 映射关系
  brand_one_id        STRING,
  group_one_id        STRING,
  global_one_id       STRING,
  mapping_type        STRING        NOT NULL,  -- brand_to_group | brand_to_global | group_to_global

  -- 传递链路
  propagation_path    ARRAY<STRING>,
  propagation_direction STRING,
  propagation_status  STRING,
  last_propagated_at  TIMESTAMP,

  -- 合并建议
  merge_recommendation STRING,
  merge_reason         STRING,
  merge_confidence     FLOAT64,

  -- 置信度
  confidence          FLOAT64       NOT NULL,
  shared_id_count     INT64         NOT NULL,
  shared_id_types     ARRAY<STRING>,

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,
  updated_at          TIMESTAMP     NOT NULL,
  expired_at          TIMESTAMP,
  
  -- 分区字段
  dt                  DATE          NOT NULL
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

---

## 六、ADS 层 — 应用数据层（oneid_ads 数据集）

### 6.1 ads_oneid_gov_intervention_operation_di（干预操作表）

记录所有人工干预操作的全生命周期。

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_intervention_operation_di`
(
  -- 基础标识
  operation_id        STRING        NOT NULL,
  operation_type      STRING        NOT NULL,

  -- 操作目标
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,
  target_one_ids      ARRAY<STRING>,
  target_id_mappings  ARRAY<STRUCT<
    one_id            STRING,
    id_type           STRING,
    id_value          STRING
  >>,

  -- 操作参数
  params              JSON,

  -- 审批流程
  status              STRING        NOT NULL,
  reviewed_by         ARRAY<STRING>,

  -- Dry Run 结果
  dry_run_result      JSON,

  -- 执行结果
  execution_result    JSON,

  -- 干预有效期
  effective_mode      STRING        NOT NULL,
  effective_from      TIMESTAMP     NOT NULL,
  effective_until     TIMESTAMP,

  -- 审计
  created_by          STRING        NOT NULL,
  reason              STRING        NOT NULL,
  related_ticket      STRING,

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,
  updated_at          TIMESTAMP     NOT NULL,
  executed_at         TIMESTAMP,
  rolled_back_at      TIMESTAMP,
  
  -- 分区字段
  dt                  DATE          NOT NULL
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


### 6.2 ads_oneid_gov_intervention_rule_df（干预规则表）

干预操作生效后生成的持久化规则。

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_intervention_rule_df`
(
  -- 规则标识
  rule_id             STRING        NOT NULL,
  rule_type           STRING        NOT NULL,
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,

  -- 规则内容
  target_id_values    ARRAY<STRING>,
  constraint          STRING        NOT NULL,
  constraint_params   JSON,

  -- 来源追溯
  source_operation_id STRING        NOT NULL,
  created_by          STRING        NOT NULL,

  -- 有效期
  effective_mode      STRING        NOT NULL,
  effective_from      TIMESTAMP     NOT NULL,
  effective_until     TIMESTAMP,
  is_active           BOOL          NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
CLUSTER BY scope, scope_id, constraint, is_active
OPTIONS (
  description = 'ADS层：干预规则表，干预操作生效后生成的持久化规则'
);
```

**示例数据**：


| rule_id  | rule_type    | scope | scope_id | target_id_values                          | constraint   | constraint_params        | is_active | source_operation_id |
| -------- | ------------ | ----- | -------- | ----------------------------------------- | ------------ | ------------------------ | --------- | ------------------- |
| RULE_001 | edge_removed | brand | 1001     | [phone:13812345678, openid:openid_abc123] | edge_removed | {reason:"family_device"} | true      | INTV_20260619_001   |


> **说明**：干预操作生效后生成持久化规则 RULE_001，在下次离线批处理时，会移除 phone:13812345678 和 openid:openid_abc123 之间的边，防止再次被 WCC 连通。

### 6.3 ads_oneid_gov_approval_policy_df（审批策略配置表）

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_approval_policy_df`
(
  -- 策略标识
  policy_id           STRING        NOT NULL,
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,

  -- 审批人配置
  reviewer_roles      ARRAY<STRING>,
  reviewer_count      INT64         NOT NULL,
  auto_approve_enabled BOOL        NOT NULL,
  auto_approve_threshold FLOAT64,

  -- 超时策略
  review_timeout_hours INT64        NOT NULL,
  timeout_action      STRING        NOT NULL,

  -- 审批规则明细
  rules               ARRAY<STRUCT<
    operation_type    STRING,
    risk_level        STRING,
    require_approval  BOOL,
    require_dry_run   BOOL,
    max_batch_size    INT64,
    cooldown_hours    INT64,
    require_reason    BOOL
  >>,

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,
  updated_at          TIMESTAMP     NOT NULL,
  is_active           BOOL          NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
CLUSTER BY scope, scope_id
OPTIONS (
  description = 'ADS层：审批策略配置表'
);
```

### 6.4 ads_oneid_gov_cleaning_rules_df（清洗规则配置表）

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_cleaning_rules_df`
(
  -- 规则标识
  rule_id             STRING        NOT NULL,
  rule_name           STRING        NOT NULL,
  rule_type           STRING        NOT NULL,  -- normalization | filter | priority
  
  -- 规则配置
  rule_config         JSON          NOT NULL,
  
  -- 适用范围
  scope               STRING,
  scope_id            STRING,
  
  -- 状态
  is_active           BOOL          NOT NULL,
  priority            INT64         NOT NULL,
  
  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,
  updated_at          TIMESTAMP     NOT NULL,
  created_by          STRING        NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
CLUSTER BY rule_type, is_active
OPTIONS (
  description = 'ADS层：清洗规则配置表'
);
```

### 6.5 ads_oneid_gov_data_source_config_df（数据源配置表）

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_data_source_config_df`
(
  -- 配置标识
  config_id           STRING        NOT NULL,
  source_system       STRING        NOT NULL,
  
  -- 接入配置
  source_type         STRING        NOT NULL,
  source_location     STRING        NOT NULL,
  source_format       STRING,
  
  -- 字段映射
  field_mapping       JSON,
  
  -- 层级分配规则
  scope_assignment    JSON,
  
  -- 调度配置
  schedule_cron       STRING,
  incremental_mode    BOOL,
  incremental_key     STRING,
  
  -- 状态
  is_active           BOOL          NOT NULL,
  
  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,
  updated_at          TIMESTAMP     NOT NULL,
  created_by          STRING        NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
CLUSTER BY source_system, is_active
OPTIONS (
  description = 'ADS层：数据源配置表'
);
```

### 6.6 ads_oneid_gov_audit_log_di（审计日志表）

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_audit_log_di`
(
  -- 日志标识
  log_id              STRING        NOT NULL,

  -- 操作信息
  operation_type      STRING        NOT NULL,
  operation_id        STRING,
  batch_id            STRING,

  -- 变更目标
  scope               STRING        NOT NULL,
  scope_id            STRING        NOT NULL,
  affected_one_ids    ARRAY<STRING>,

  -- 变更前后快照
  before_snapshot     JSON,
  after_snapshot      JSON,
  diff_summary        STRING,

  -- 操作人
  operator_type       STRING        NOT NULL,
  operator_id         STRING        NOT NULL,
  operator_name       STRING,

  -- 原因与审批
  reason              STRING        NOT NULL,
  approved_by         ARRAY<STRING>,

  -- 时间
  created_at          TIMESTAMP     NOT NULL,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY scope, operation_type, operator_type
OPTIONS (
  description = 'ADS层：审计日志表',
  partition_expiration_days = 365
);
```

### 6.7 ads_oneid_gov_rollback_operation_di（回滚操作表）

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_rollback_operation_di`
(
  -- 回滚标识
  rollback_id         STRING        NOT NULL,
  rollback_type       STRING        NOT NULL,

  -- 回滚目标
  target_one_id       STRING,
  target_operation_id STRING,
  target_snapshot_date DATE,

  -- 回滚状态
  status              STRING        NOT NULL,
  dry_run_result      JSON,

  -- 审批
  requested_by        STRING        NOT NULL,
  approved_by         ARRAY<STRING>,
  reason              STRING        NOT NULL,

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,
  executed_at         TIMESTAMP,
  completed_at        TIMESTAMP,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY rollback_type, status
OPTIONS (
  description = 'ADS层：回滚操作表'
);
```

### 6.8 ads_oneid_gov_backfill_task_di（数据回填任务表）

```sql
CREATE OR REPLACE TABLE `oneid_ads.ads_oneid_gov_backfill_task_di`
(
  -- 任务标识
  task_id             STRING        NOT NULL,
  source_operation_id STRING        NOT NULL,

  -- 回填范围
  affected_one_ids    ARRAY<STRING>,
  affected_date_start DATE,
  affected_date_end   DATE,
  target_systems      ARRAY<STRING>,

  -- 回填模式
  mode                STRING        NOT NULL,

  -- 执行状态
  status              STRING        NOT NULL,
  progress            FLOAT64,
  error_message       STRING,

  -- 生命周期
  created_at          TIMESTAMP     NOT NULL,
  started_at          TIMESTAMP,
  completed_at        TIMESTAMP,
  
  -- 分区字段
  dt                  DATE          NOT NULL
)
PARTITION BY dt
CLUSTER BY status, mode
OPTIONS (
  description = 'ADS层：数据回填任务表'
);
```

---

## 七、表关系总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ODS 层 (oneid_ods)                                │
│                                                                     │
│  ods_trade_user_user_profile_wide_di ──┐                            │
│  (用户信息宽表)                         │                            │
│                                        ├──→ 清洗引擎                  │
│  ods_trade_user_association_event_di ──┘   (归一化、过滤)             │
│  (关联事件表)                             │                          │
│                                         │                            │
│  ads_oneid_gov_cleaning_rules_df ───────┘                            │
│  (清洗规则配置)                                                       │
│                                                                     │
└────────────────────────────────┼────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    DWD 层 (oneid_dwd)                                │
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
│    │                              │                                 │
│    ├──→ audit_log_di              │                                 │
│    │                              │                                 │
│    ├──→ rollback_operation_di     │                                 │
│    │                              │                                 │
│    └──→ backfill_task_di          │                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 八、分区与聚类策略总结


| 表                                           | 层级  | 分区键  | 聚类键                                      | 过期策略  | 说明                |
| ------------------------------------------- | --- | ---- | ---------------------------------------- | ----- | ----------------- |
| `ods_trade_user_user_profile_wide_di`       | ODS | `dt` | `source_system, scope, scope_id`         | 180 天 | 原始数据，保留半年         |
| `ods_trade_user_association_event_di`       | ODS | `dt` | `source_system, event_type, scope`       | 180 天 | 同上                |
| `dwd_oneid_graph_vertex_df`                 | DWD | `dt` | `scope, scope_id, id_type`               | 90 天  | 图计算中间结果           |
| `dwd_oneid_graph_edge_df`                   | DWD | `dt` | `scope, scope_id`                        | 90 天  | 同上                |
| `dwd_oneid_graph_cleaning_log_di`           | DWD | `dt` | `scope, action_type, filter_reason`      | 90 天  | 清洗日志              |
| `dws_oneid_identity_wcc_result_df`          | DWS | `dt` | `scope, scope_id, component_id`          | 90 天  | WCC归属结果（vertex粒度） |
| `dws_oneid_identity_mapping_decision_di`    | DWS | `dt` | `scope, scope_id, decision_type`         | 90 天  | ID映射决策过程          |
| `dws_oneid_identity_merge_operation_di`     | DWS | `dt` | `scope, scope_id, trigger_type`          | 90 天  | 融合操作明细            |
| `dws_oneid_identity_split_operation_di`     | DWS | `dt` | `scope, scope_id, trigger_type`          | 90 天  | 分裂操作明细            |
| `dws_oneid_identity_conflict_resolution_di` | DWS | `dt` | `scope, scope_id, resolution_rule`       | 90 天  | 冲突消解明细            |
| `dws_oneid_identity_oneid_master_df`        | DWS | `dt` | `scope, scope_id, status`                | 永不过期  | 核心数据              |
| `dws_oneid_identity_cross_layer_mapping_df` | DWS | `dt` | `mapping_type, brand_one_id`             | 永不过期  | 跨层关系              |
| `ads_oneid_gov_intervention_operation_di`   | ADS | `dt` | `scope, operation_type, status`          | 永不过期  | 操作记录              |
| `ads_oneid_gov_intervention_rule_df`        | ADS | `dt` | `scope, scope_id, constraint, is_active` | 永不过期  | 规则表               |
| `ads_oneid_gov_approval_policy_df`          | ADS | `dt` | `scope, scope_id`                        | 永不过期  | 配置表               |
| `ads_oneid_gov_cleaning_rules_df`           | ADS | `dt` | `rule_type, is_active`                   | 永不过期  | 规则配置              |
| `ads_oneid_gov_data_source_config_df`       | ADS | `dt` | `source_system, is_active`               | 永不过期  | 数据源配置             |
| `ads_oneid_gov_audit_log_di`                | ADS | `dt` | `scope, operation_type, operator_type`   | 365 天 | 审计日志              |
| `ads_oneid_gov_rollback_operation_di`       | ADS | `dt` | `rollback_type, status`                  | 永不过期  | 回滚记录              |
| `ads_oneid_gov_backfill_task_di`            | ADS | `dt` | `status, mode`                           | 永不过期  | 回填任务              |


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
ODS（原始数据）
  ↓ 清洗、规范化
DWD（明细数据）
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

