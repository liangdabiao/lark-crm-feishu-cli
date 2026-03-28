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

```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<商机管理_table_id>" }
  },
  "dimensions": [
    { "field_name": "销售区域", "alias": "dim_region" }
  ],
  "measures": [
    { "field_name": "销售区域", "aggregation": "count_all", "alias": "opp_count" },
    { "field_name": "业务价值", "aggregation": "sum", "alias": "total_value" }
  ],
  "sort": [
    { "field_name": "total_value", "order": "desc" }
  ],
  "pagination": { "limit": 100 },
  "shaper": { "format": "flat" }
}
```

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

### 9. 赢单商机按月份统计（合同时间趋势)

```json
{
  "datasource": {
    "type": "table",
    "table": { "tableId": "<合同管理_table_id>" }
  },
  "dimensions": [],
  "measures": [
    { "field_name": "合同金额", "aggregation": "sum", "alias": "monthly_total" },
    { "field_name": "合同金额", "aggregation": "count_all", "alias": "monthly_count" }
  ],
  "sort": [],
  "pagination": { "limit": 100 },
  "shaper": { "format": "flat" }
}
```

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

1. **data-query 限制**：需要 base 完全访问权限（FA）；不支持 formula / lookup / 关联等字段作为 dimensions 或 measures 的 `field_name`。
2. **字段名精确匹配**：DSL 中的 `field_name` 必须与表字段名完全一致（含大小写），不可凭猜测填写。
3. **alias 规范**：alias 必须唯一且建议使用英文下划线格式（如 `dim_stage`、`total_value`）。
4. **shaper 必填**：每个查询都必须包含 `"shaper": {"format": "flat"}`。
5. **pagination.limit**：最大 5000。
6. **聚合字段类型限制**：sum/avg 只能用于数字类型字段；count_all 可用于任何字段。
7. **输出解读**：data-query 返回的是聚合结果（不是原始记录），结果在 `main_data` 中，每个值是 `{"value": xxx}` 格式。
