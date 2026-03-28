---
name: lark-crm
version: 2.1.0
description: "飞书轻型 CRM 系统管理：客户管理、商机管理、跟进记录、合同管理、销售报表分析。当用户提到 CRM、客户、商机、合同、跟进记录、销售漏斗、赢单、丢单、销售报表、销售统计、客户查询、客户管理时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 轻型 CRM 系统

> **前置条件：** 先阅读 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md) 了解认证和权限。
> **底层依赖：** 本 skill 所有操作基于 [`../lark-base/SKILL.md`](../lark-base/SKILL.md)，执行 base 命令前必须先阅读 lark-base 的 SKILL.md 和对应 reference 文档。

## 配置初始化（首次使用必须执行）

本 skill 的 base-token、URL、table_id **不在文件中硬编码**，而是通过用户提供的飞书多维表格 URL 自动发现。

**初始化流程**：

1. **检查是否已有配置**：AI 在当前会话中是否已缓存了 `base_token` 和表结构映射。若有，跳到步骤 5。
2. **向用户要 URL**：提示用户提供飞书 CRM 多维表格的 URL，格式如 `https://xxx.feishu.cn/base/XXXXXX?table=tblXXX&view=vewXXX`。
3. **解析 base-token**：从 URL 中提取 `/base/` 后面的那段字符串，即为 `base_token`。
4. **自动发现表结构**：执行以下命令获取所有表及其 table_id：
   ```bash
   lark-cli base +table-list --base-token <base_token>
   ```
   将返回的 `table_name → table_id` 映射缓存到会话中。
5. **后续所有命令**：使用缓存的 `base_token` 和各表的 `table_id` 构造命令。

**配置变更**：若用户说「地址换了」或提供新 URL，重新执行步骤 2-4 刷新缓存。

### 表名与用途对照（用于匹配 table_id）

| 表名关键字 | 用途 |
|------------|------|
| 客户管理 | 客户信息和联系人 |
| 商机管理 | 销售商机和阶段追踪 |
| 客户跟进记录 | 客户跟进活动记录 |
| 合同管理 | 合同和签约信息 |
| 销售人员管理 | 销售团队和区域 |

> 注意：`+table-list` 返回的表名与上述对照表匹配即可确定对应关系。

## 意图路由决策树

```
用户输入
├─ 客户相关（客户/联系人/对接人）
│   ├─ 查询/查看/搜索/找 → +客户查询
│   ├─ 新增/添加/创建/录入 → +客户新增
│   └─ 更新/修改/编辑 → +客户更新
├─ 商机相关（商机/销售漏斗/赢单/丢单/交易/机会）
│   ├─ 查询/查看/漏斗 → +商机查询
│   ├─ 新增/创建 → +商机新增
│   └─ 更新阶段/状态 → +商机更新
├─ 跟进相关（跟进/拜访/电话/沟通/回访）
│   └─ 记录/添加/写 → +跟进记录
├─ 合同相关（合同/签约/协议）
│   ├─ 查询/查看 → +合同查询
│   └─ 新增/录入 → +合同新增
├─ 删除相关（删除/移除/不要了）
│   └─ +记录删除（需 --yes 确认）
├─ 报表分析（报表/统计/分析/排名/总计/汇总/业绩/漏斗）
│   └─ +销售报表（读取 crm-analytics.md，使用 +data-query）
└─ 销售团队（销售团队/销售人员/销售区域）
    └─ +销售团队查询
```

## Shortcuts

> **命令模板约定**：下方示例中 `<base_token>` 和 `<table_id>` 为占位符，执行时替换为配置初始化中缓存的实际值。

### +客户查询

查询客户信息。支持按名称、城市、行业、规模等条件筛选。

```bash
lark-cli base +record-list \
  --base-token <base_token> \
  --table-id <客户管理_table_id> \
  --limit 50
```

**筛选方法**：`+record-list` 返回全量数据后，AI 在本地按条件过滤（按行业、城市、规模等字段匹配）。不要使用 `+view-set-filter`，因为它需要视图编辑权限且必须指定 `--view-id`，只读用户无法使用。

**输出格式**：以表格形式展示客户名称、行业、规模、城市、对接人、客户所有人、最近跟进时间。

### +客户新增

新增客户记录。

**可写字段**（存储字段，可直接写入）：

| 字段名 | 类型 | 必填 | 格式 |
|--------|------|------|------|
| 客户名称 | text | 是 | 字符串 |
| 行业信息 | select | 否 | 单选：电商/互联网/金融/医疗/餐饮/科技/制造 |
| 客户规模 | select | 否 | 单选：<10 / 10-100 / 100-1000 / 1000-10000 / >10000 |
| 所在城市 | text | 否 | 字符串 |
| 客户地址 | location | 否 | `{"lng": 116.39, "lat": 39.90}` |
| 对接人姓名 | text | 否 | 字符串 |
| 对接人职位 | select | 否 | 单选：CEO/CTO/IT负责人/人事负责人/产品负责人/运营负责人 |
| 对接人邮箱 | text | 否 | 邮箱字符串 |
| 联系电话 | number | 否 | 数字 |
| 客户所有人 | user | 否 | `[{"id":"ou_xxx"}]` |

**只读字段**（不要写入）：创建时间、最近跟进时间（lookup）、销售leader（lookup）、客户名称查重（formula）、是否为公海客户（formula）、客户关系管理-对接客户（link）、合同管理-客户名称（link）、客户跟进记录-商机名称（link）。

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <客户管理_table_id> \
  --json '{"客户名称":"XX公司","行业信息":"互联网","客户规模":"10-100","所在城市":"北京","对接人姓名":"张三","对接人职位":"CTO","联系电话":13800000000}'
```

**执行前**：
1. 确认用户提供了必填字段（客户名称）。
2. **校验 select 字段值**：用户输入的 select 类型字段值必须在可选项内。若不在，提示用户可选值列表并确认替换。
3. **查找 user 字段的 open_id**：客户所有人等 user 类型字段需要 open_id（如 `ou_xxx`）。若用户只提供了姓名，先通过以下方式查找：
   - 从 `+record-list` 已有记录的客户所有人字段中匹配同名用户获取 open_id
   - 或通过 `lark-cli contact +search` 搜索员工获取 open_id

### +客户更新

更新已有客户信息。需要 record_id。

```bash
# 先通过 +客户查询 获取 record_id，再更新
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <客户管理_table_id> \
  --record-id recXXXXXX \
  --json '{"所在城市":"上海"}'
```

### +商机查询

查询商机信息或查看销售漏斗。

```bash
# 查询所有商机
lark-cli base +record-list \
  --base-token <base_token> \
  --table-id <商机管理_table_id> \
  --limit 50

# 按阶段聚合统计（销售漏斗）— 读取 crm-analytics.md 获取 DSL
lark-cli base +data-query \
  --base-token <base_token> \
  --json '<参见 crm-analytics.md 销售漏斗模板>'
```

**输出格式**：展示机会名称（formula 字段，如"国信易有限公司-协商议价-..."）、客户名称（link 字段，需查客户管理表解析）、跟进阶段、业务价值、预计交易日期、跟进销售人员。

> **注意**：客户名称是 link 字段，返回 record_id。若需展示可读名称，先查客户管理表建立 ID→名称映射。

### +商机新增

新增商机。需关联到已有客户。

**可写字段**：

| 字段名 | 类型 | 必填 | 格式 |
|--------|------|------|------|
| 客户名称 | link | 是 | `[{"id":"recXXXXXX"}]`（客户管理表的 record_id） |
| 跟进阶段 | select | 否 | 单选：待接触/初步提议/方案报价/协商议价/丢失客户/赢单 |
| 业务价值 | number | 否 | 数字 |
| 商机描述 | text | 否 | 字符串 |
| 预计交易日期 | datetime | 否 | "YYYY-MM-DD HH:mm:ss" |
| 跟进销售人员 | user | 否 | `[{"id":"ou_xxx"}]` |
| 丢单原因 | select(多) | 否 | 多选：如 ["超客户预算"] |
| 丢单原因备注 | text | 否 | 字符串 |

**只读字段**：机会名称（formula）、销售leader（lookup）、销售区域（lookup）、是否已有合同（lookup）、合同管理-商机名称（link）、商机创建时间（datetime 但由系统维护）。

```bash
# 需先通过 +客户查询 获取客户的 record_id
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <商机管理_table_id> \
  --json '{"客户名称":[{"id":"recXXXXXX"}],"跟进阶段":"待接触","业务价值":100000,"商机描述":"客户对方案很感兴趣","预计交易日期":"2026-06-30 00:00:00","跟进销售人员":[{"id":"ou_xxx"}]}'
```

### +商机更新

更新商机阶段或状态。

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <商机管理_table_id> \
  --record-id recXXXXXX \
  --json '{"跟进阶段":"赢单"}'
```

### +跟进记录

添加客户跟进记录。

**可写字段**：

| 字段名 | 类型 | 必填 | 格式 |
|--------|------|------|------|
| 客户名称 | link | 是 | `[{"id":"recXXXXXX"}]`（客户管理表的 record_id） |
| 跟进内容 | text | 是 | 字符串 |
| 跟进形式 | select | 否 | 单选：现场拜访/电话沟通 |
| 跟进人员 | user(多) | 否 | `[{"id":"ou_xxx"}]` |
| 陪同人员 | user(多) | 否 | `[{"id":"ou_xxx"}]` |
| 跟进妙记链接 | text | 否 | URL 字符串 |
| 跟进内容照片 | attachment | 否 | 需通过 +record-upload-attachment 上传 |

**只读字段**：跟进时间（created_at）、商机情况（lookup）。

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <客户跟进记录_table_id> \
  --json '{"客户名称":[{"id":"recXXXXXX"}],"跟进内容":"拜访客户讨论方案","跟进形式":"现场拜访","跟进人员":[{"id":"ou_xxx"}]}'
```

### +合同查询

查询合同信息。

```bash
lark-cli base +record-list \
  --base-token <base_token> \
  --table-id <合同管理_table_id> \
  --limit 50
```

**输出格式**：展示合同编号、客户名称、签约日期、合同金额、签约人员。

> **注意**：客户名称和商机名称是 link 字段，返回 record_id。需先查客户管理表和商机管理表建立 ID→名称映射后再展示。

### +合同新增

新增合同记录。

**可写字段**：

| 字段名 | 类型 | 必填 | 格式 |
|--------|------|------|------|
| 合同编号 | text | 是 | 字符串 |
| 商机名称 | link | 否 | `[{"id":"recXXXXXX"}]`（商机管理表的 record_id） |
| 客户名称 | link | 否 | `[{"id":"recXXXXXX"}]`（客户管理表的 record_id） |
| 签约日期 | datetime | 否 | "YYYY-MM-DD HH:mm:ss" |
| 合同金额 | number | 否 | 数字 |
| 合同附件 | text | 否 | URL 字符串 |
| 合同录入 | user(多) | 否 | `[{"id":"ou_xxx"}]` |

**只读字段**：签约人员（lookup）。

```bash
# 需先通过 +客户查询 和 +商机查询 获取 record_id
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <合同管理_table_id> \
  --json '{"合同编号":"20260329-001","客户名称":[{"id":"recXXXXXX"}],"商机名称":[{"id":"recXXXXXX"}],"签约日期":"2026-03-29 00:00:00","合同金额":500000}'
```

### +销售报表

销售数据分析和报表。**必须先读取 [`references/crm-analytics.md`](references/crm-analytics.md)** 获取 data-query DSL 模板。

常见分析场景：
- 销售漏斗（按阶段统计商机数和金额）
- 赢单率统计
- 按销售统计业绩
- 按区域统计
- 合同金额汇总
- 按行业/客户规模分布

### +记录删除

删除记录（客户、商机、跟进、合同等）。

```bash
lark-cli base +record-delete \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id recXXXXXX \
  --yes
```

**注意**：
- **必须加 `--yes`**，否则报 `unsafe_operation_blocked` 错误
- 删除不可逆，执行前必须确认用户意图
- 如果有关联记录（如客户关联了商机/合同），需先删除子记录再删父记录

### +销售团队查询

查看销售团队信息。

```bash
lark-cli base +record-list \
  --base-token <base_token> \
  --table-id <销售人员管理_table_id> \
  --limit 50
```

## 自然语言时间解析

用户说相对时间时，AI 自行转换为绝对日期：

| 用户表达 | 转换规则 |
|---------|---------|
| 今天 | 当天 |
| 昨天 | 当天 -1 天 |
| 本周 | 本周一 00:00 ~ 当前 |
| 上周 | 上周一 00:00 ~ 上周日 23:59 |
| 本月 | 当月1日 00:00 ~ 当前 |
| 上月 | 上月1日 00:00 ~ 上月最后一天 23:59 |
| 近 N 天 | 当前 -N 天 ~ 当前 |

## 操作纪律

1. **配置优先** — 执行任何 CRM 命令前，确保已完成配置初始化（有缓存的 base_token 和表结构映射）。若无缓存,先走初始化流程。
2. **先读 reference 再执行** — 执行 `+data-query` 前必须先读 [`references/crm-analytics.md`](references/crm-analytics.md)；执行 base 原子命令前必须先读 lark-base 对应 reference 文档。
3. **写记录前确认字段可写性** — 只写存储字段，formula / lookup / 系统字段跳过。如用户提供了 lookup 字段值（如销售Leader），告知该字段为只读，无法直接设置。
4. **关联记录先查 ID** — 新增商机需关联客户时，先通过 `+客户查询` 获取客户的 record_id。
5. **+xxx-list 串行执行** — base 的所有 list 类命令不能并发。
6. **数据分析走 +data-query** — 涉及聚合统计时用 `+data-query`，不要用 `+record-list` 拉全量手动计算。
7. **写入操作前确认用户意图** — 新增/更新/删除记录前确认用户确实要操作。
8. **校验 select 字段** — 用户提供的 select 类型字段值必须在可选项列表内，不在则提示可选项让用户确认。
9. **user 字段需要 open_id** — 用户给姓名时,先从已有记录匹配或通过 `lark-cli contact +search` 查找 open_id。
10. **403 权限错误处理** — 遇到 HTTP 403 时，告知用户需要让 base 管理员授予编辑权限，或通过飞书界面分享表格并设置「可编辑」。
11. **link 字段解析** — 查询结果中的 link 类型字段（如客户名称、商机名称）返回的是 record_id（`[{"id":"recXXXXXX"}]`），不是可读名称。若需展示名称，必须先查对应表建立 ID→名称映射后再输出。
12. **删除必须加 --yes** — `+record-delete` 必须带 `--yes` 参数，否则会报 `unsafe_operation_blocked`。删除前确认用户意图。
13. **data-query 返回格式** — `+data-query` 的 `main_data` 中每个值是 `{"value": xxx}` 格式。金额字段（如 sum 聚合）返回带 ¥ 符号的字符串（如 `"¥9,215,261.00"`），展示时直接使用或解析数字部分做进一步计算。

## 权限

| 操作 | 身份 | 所需条件 |
|------|------|---------|
| 所有 CRM 操作 | `--as user`（默认） | 拥有该 base 的访问权限 |
| 查询 | user | base 只读权限即可 |
| 新增/更新/删除 | user | base 编辑或完全访问权限 |
| data-query 聚合 | user | base 完全访问权限（FA） |

### 重要权限说明
- **data-query 限制**：`+data-query` 命令需要 base 完全访问权限（FA），普通只读用户无法使用。若用户无此权限，建议使用 `+record-list` 拉取数据后本地聚合。
- **并发限制**：base 的所有 list 类命令（如 `+record-list`、`+table-list`）不能并发执行，需串行执行避免 API 限制。
