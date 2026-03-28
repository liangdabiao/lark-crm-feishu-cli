# CRM 数据分析与报表

> **前置条件：** 先阅读 [`../lark-base/SKILL.md`](../../lark-base/SKILL.md) 和 [`lark-base-data-query.md`](../../lark-base/references/lark-base-data-query.md)。
> **重要：** 执行 `+data-query` 前必须先阅读 lark-base 的 data-query reference 文档。

**零配置说明**： 所有 DSL 模板中的 `tableId` 和 base-token 不再硬编码，执行前替换为配置初始化中缓存的值。

每次使用模板前，用缓存中的 `<base_token>` 和 `<对应表_table_id>` 替换。

参见下方每个模板的注释。

**通用参数**

```bash
lark-cli base +data-query \
  --base-token <base_token> \
  --dsl '<DSL>'
```

DSL 模板中 tableId 占位符说明：

- `<商机管理_table_id>` → 商机管理表 table_id
- `<客户管理_table_id>` → 客户管理表 table_id
- `<合同管理_table_id>` → 合同管理表 table_id
- `<销售人员管理_table_id>` → 销售人员管理表 table_id
- `<客户跟进记录_table_id>` → 客户跟进记录表 table_id

## 分析场景模板

### 1. 销售漏斗（按阶段统计商机数和金额）

展示每个销售阶段的商机数量和业务价值总额。

```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<商机管理_table_id>" }
  },
  "dimensions": [
    { "field_name": "跟进阶段", "alias": "dim_stage" }
  ],
  "measures": [
    { "field_name": "跟进阶段", "aggregation": "count_all", "alias": "opp_count" },
    { "field_name": "业务价值", "aggregation": "sum", "alias": "total_value" }
  ],
  "sort": [
    { "field_name": "total_value", "order": "desc" }
  ],
  "pagination": { "limit": 100 },
  "shaper": { "format": "flat" }
}
```

**输出示例**：
| 跟进阶段 | 商机数 | 业务价值总额 |
|---------|--------|------------|
| 赢单 | 3 | 8,195,261 |
| 协商议价 | 1 | 1,750,000 |
| ... | ... | ... |

### 2. 赢单率统计

统计总商机数和赢单数，计算赢单率。需要分别查询赢单数和总数后计算。

详见"赢单率统计"专项说明。

需要先执行以下两个查询，然后计算赢单率：

**查询赢单数**：
```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<商机管理_table_id>" }
  },
  "dimensions": [],
  "measures": [
    { "field_name": "跟进阶段", "aggregation": "count_all", "alias": "win_count" }
  ],
  "filters": {
    "type": 1,
    "conjunction": "and",
    "conditions": [
      { "field_name": "跟进阶段", "operator": "is", "value": ["赢单"] }
    ]
  },
  "pagination": { "limit": 1 },
  "shaper": { "format": "flat" }
}
```

**查询总商机数**：
```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<商机管理_table_id>" }
  },
  "dimensions": [],
  "measures": [
    { "field_name": "跟进阶段", "aggregation": "count_all", "alias": "total_count" }
  ],
  "pagination": { "limit": 1 },
  "shaper": { "format": "flat" }
}
```

**赢单率** = 赢单数 / 总商机数

### 3. 按销售人员统计业绩

```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<商机管理_table_id>" }
  },
  "dimensions": [
    { "field_name": "跟进销售人员", "alias": "dim_sales" }
  ],
  "measures": [
    { "field_name": "跟进销售人员", "aggregation": "count_all", "alias": "opp_count" },
    { "field_name": "业务价值", "aggregation": "sum", "alias": "total_value" }
  ],
  "sort": [
    { "field_name": "total_value", "order": "desc" }
  ],
  "pagination": { "limit": 100 },
  "shaper": { "format": "flat" }
}
```

### 4. 按销售区域统计

> **注意**：商机管理表的「销售区域」是 lookup 字段，`+data-query` 不支持 lookup 字段。需采用两步方案：先从销售人员管理表获取「销售人员 → 销售区域」映射，再从商机管理表拉取记录后本地按区域聚合。

**步骤 1**：查询销售人员管理表，建立「跟进销售人员 open_id → 销售区域」映射。

```bash
lark-cli base +record-list \
  --base-token <base_token> \
  --table-id <销售人员管理_table_id> \
  --limit 100
```

从返回结果的「销售」字段提取 open_id，与「销售区域」字段建立映射。

**步骤 2**：查询商机管理表，拉取全部商机记录。

```bash
lark-cli base +record-list \
  --base-token <base_token> \
  --table-id <商机管理_table_id> \
  --limit 200
```

**步骤 3**：AI 本地聚合 — 遍历商机记录，通过「跟进销售人员」的 open_id 查映射表得到「销售区域」，按区域汇总商机数和业务价值。

**输出示例**：
| 销售区域 | 商机数 | 业务价值总额 |
|---------|--------|------------|
| 华东地区 | 5 | ¥2,500,000 |
| 华北地区 | 3 | ¥1,200,000 |
| ... | ... | ... |

### 5. 合同金额汇总

```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<合同管理_table_id>" }
  },
  "dimensions": [],
  "measures": [
    { "field_name": "合同金额", "aggregation": "sum", "alias": "total_contract" },
    { "field_name": "合同金额", "aggregation": "count_all", "alias": "contract_count" },
    { "field_name": "合同金额", "aggregation": "avg", "alias": "avg_contract" },
    { "field_name": "合同金额", "aggregation": "max", "alias": "max_contract" }
  ],
  "pagination": { "limit": 1 },
  "shaper": { "format": "flat" }
}
```

**输出示例**：
| 合同总额 | 合同数量 | 平均合同金额 | 最大合同金额 |
|---------|---------|------------|------------|
| 9,200,232 | 5 | 1,840,046 | 5,000,232 |

### 6. 按行业分布（客户管理表）

```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<客户管理_table_id>" }
  },
  "dimensions": [
    { "field_name": "行业信息", "alias": "dim_industry" }
  ],
  "measures": [
    { "field_name": "行业信息", "aggregation": "count_all", "alias": "customer_count" }
  ],
  "sort": [
    { "field_name": "customer_count", "order": "desc" }
  ],
  "pagination": { "limit": 100 },
  "shaper": { "format": "flat" }
}
```

### 7. 按客户规模分布
```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<客户管理_table_id>" }
  },
  "dimensions": [
    { "field_name": "客户规模", "alias": "dim_scale" }
  ],
  "measures": [
    { "field_name": "客户规模", "aggregation": "count_all", "alias": "customer_count" }
  ],
  "sort": [
    { "field_name": "customer_count", "order": "desc" }
  ],
  "pagination": { "limit": 100 },
  "shaper": { "format": "flat" }
}
```

### 8. 按城市分布
```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<客户管理_table_id>" }
  },
  "dimensions": [
    { "field_name": "所在城市", "alias": "dim_city" }
  ],
  "measures": [
    { "field_name": "所在城市", "aggregation": "count_all", "alias": "customer_count" }
  ],
  "filters": {
    "type": 1,
    "conjunction": "and",
    "conditions": [
      { "field_name": "所在城市", "operator": "isNotEmpty", "value": [] }
    ]
  },
  "sort": [
    { "field_name": "customer_count", "order": "desc" }
  ],
  "pagination": { "limit": 100 },
  "shaper": { "format": "flat" }
}
```

### 9. 合同按签约日期趋势

按签约日期分组，统计每个日期的合同金额和数量。

> **限制**：`+data-query` 不支持日期截断（如按月/季/年分组），使用签约日期作为 dimension 会按**精确日期**分组。若需按月份聚合，需用 `+record-list` 拉取合同记录后本地按年月聚合。

```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<合同管理_table_id>" }
  },
  "dimensions": [
    { "field_name": "签约日期", "alias": "dim_sign_date" }
  ],
  "measures": [
    { "field_name": "合同金额", "aggregation": "sum", "alias": "daily_total" },
    { "field_name": "合同金额", "aggregation": "count_all", "alias": "daily_count" }
  ],
  "sort": [
    { "field_name": "dim_sign_date", "order": "asc" }
  ],
  "pagination": { "limit": 5000 },
  "shaper": { "format": "flat" }
}
```

**输出示例**：
| 签约日期 | 合同金额 | 合同数量 |
|---------|---------|---------|
| 2022/12/04 | ¥9,200,232 | 5 |
| 2026/03/29 | ¥200,000 | 1 |

**按月聚合替代方案**：若用户要求按月份查看趋势，使用 `+record-list` 拉取合同数据，AI 本地按签约日期的年-月部分分组统计。

### 10. 丢单原因分析
```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<商机管理_table_id>" }
  },
  "dimensions": [
    { "field_name": "丢单原因", "alias": "dim_loss_reason" }
  ],
  "measures": [
    { "field_name": "丢单原因", "aggregation": "count_all", "alias": "loss_count" },
    { "field_name": "业务价值", "aggregation": "sum", "alias": "loss_value" }
  ],
  "filters": {
    "type": 1,
    "conjunction": "and",
    "conditions": [
      { "field_name": "跟进阶段", "operator": "is", "value": ["丢失客户"] }
    ]
  },
  "sort": [
    { "field_name": "loss_count", "order": "desc" }
  ],
  "pagination": { "limit": 100 },
  "shaper": { "format": "flat" }
}
```

## 注意事项

1. **data-query 限制**：需要 base 完全访问权限（FA）；不支持 formula / lookup / 关联等字段作为 dimensions / measures / filters / sort 的 `field_name`。以下是 CRM 各表中常见的 **不可用于 data-query 的字段**：
   - 商机管理表：机会名称（formula）、销售leader（lookup）、销售区域（lookup）、是否已有合同（lookup）、合同管理-商机名称（link）
   - 客户管理表：最近跟进时间（lookup）、销售leader（lookup）、是否为公海客户（formula）、客户名称查重（formula）
   - 合同管理表：签约人员（lookup）
   - 如需按上述字段聚合，请使用 `+record-list` 拉取数据后本地计算。
2. **字段名精确匹配**：DSL 中的 `field_name` 必须与表字段名完全一致（含大小写），不可凭猜测填写。
3. **alias 规范**：alias 必须唯一且建议使用英文下划线格式（如 `dim_stage`、`total_value`）。
4. **shaper 必填**：每个查询都必须包含 `"shaper": {"format": "flat"}`。
5. **pagination.limit**：最大 5000。
6. **聚合字段类型限制**：sum/avg 只能用于数字类型字段；count_all 可用于任何字段。
7. **输出解读**：data-query 返回的是聚合结果（不是原始记录），结果在 `main_data` 中，每个值是 `{"value": xxx}` 格式。
8. **金额返回格式**：sum/avg 等数值聚合的结果值可能是带货币符号的字符串（如 `"¥9,215,261.00"`），而非纯数字。展示时可直接使用；若需进一步计算，需去除 ¥ 符号和逗号后转为数值。count_all 返回的是纯数字。
