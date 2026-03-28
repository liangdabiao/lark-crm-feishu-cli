# CRM 操作命令详解

> **前置条件：** 先阅读 [`../lark-shared/SKILL.md`](../../lark-shared/SKILL.md) 和 [`../lark-base/SKILL.md`](../../lark-base/SKILL.md)。

本文档提供每个 CRM Shortcut 的详细命令参数、完整示例和注意事项。

## 通用参数

所有命令共用：
```bash
--base-token bascndXmwM3qJOVZjUR6LS4uSng
--as user  # 默认，可省略
```

## +客户查询

### 查询所有客户

```bash
lark-cli base +record-list \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --limit 50
```

### 按关键词筛选客户

通过视图筛选实现：

```bash
# Step 1: 设置视图筛选条件
lark-cli base +view-set-filter \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --format json \
  --json '{"field_name":"客户名称","operator":"contains","value":["关键词"]}'

# Step 2: 使用返回的 view_id 查询
lark-cli base +record-list \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --view-id <返回的 view_id> \
  --limit 50
```

### 按行业筛选

```bash
lark-cli base +view-set-filter \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --format json \
  --json '{"field_name":"行业信息","operator":"is","value":["互联网"]}'
```

### 按城市筛选

```bash
lark-cli base +view-set-filter \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --format json \
  --json '{"field_name":"所在城市","operator":"is","value":["北京"]}'
```

### 按客户所有人筛选

```bash
lark-cli base +view-set-filter \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --format json \
  --json '{"field_name":"客户所有人","operator":"is","value":["ou_xxx"]}'
```

### 输出字段顺序

查询结果以友好表格展示，优先展示以下字段：
1. 客户名称
2. 行业信息
3. 客户规模
4. 所在城市
5. 对接人姓名 / 对接人职位
6. 客户所有人
7. 最近跟进时间
8. 是否为公海客户

---

## +客户新增

### 完整示例

```bash
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --json '{
    "客户名称": "XX科技有限公司",
    "行业信息": "科技",
    "客户规模": "10-100",
    "所在城市": "上海",
    "对接人姓名": "张三",
    "对接人职位": "CTO",
    "对接人邮箱": "zhangsan@example.com",
    "联系电话": 13800138000,
    "客户所有人": [{"id": "ou_xxx"}]
  }'
```

### 最小必填示例

```bash
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --json '{"客户名称": "新客户公司"}'
```

### 选项值枚举

**行业信息**：电商、互联网、金融、医疗、餐饮、科技、制造

**客户规模**：<10、10-100、100-1000、1000-10000、>10000

**对接人职位**：CEO、CTO、IT负责人、人事负责人、产品负责人、运营负责人

### 注意事项

- 创建前先检查客户名称是否重复（系统有自动查重 formula 字段）
- 客户地址字段为 location 类型，需要传入 `{"lng": xx, "lat": xx}` 格式
- 成功后返回 record_id，用于后续关联商机和跟进记录

---

## +客户更新

```bash
# 更新单个字段
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --record-id recXXXXXX \
  --json '{"所在城市": "深圳"}'

# 更新多个字段
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblVw8OKO1Yz2Lcl \
  --record-id recXXXXXX \
  --json '{"行业信息": "金融", "客户规模": ">10000", "对接人姓名": "李四"}'
```

---

## +商机查询

### 查询所有商机

```bash
lark-cli base +record-list \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblMXhNvEvnEU3W2 \
  --limit 50
```

### 按阶段筛选

```bash
# 查看赢单商机
lark-cli base +view-set-filter \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblMXhNvEvnEU3W2 \
  --format json \
  --json '{"field_name":"跟进阶段","operator":"is","value":["赢单"]}'
```

**跟进阶段枚举**：待接触、初步提议、协商议价、丢失客户、赢单

### 按销售人员筛选

```bash
lark-cli base +view-set-filter \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblMXhNvEvnEU3W2 \
  --format json \
  --json '{"field_name":"跟进销售人员","operator":"is","value":["ou_xxx"]}'
```

### 输出字段

1. 机会名称（formula 自动生成）
2. 客户名称（关联显示）
3. 跟进阶段
4. 业务价值
5. 预计交易日期
6. 跟进销售人员
7. 销售区域（lookup）

---

## +商机新增

### 操作步骤

1. 先通过 `+客户查询` 获取目标客户的 record_id
2. （可选）通过 `+销售团队查询` 获取销售人员的 ou_id
3. 执行新增

### 完整示例

```bash
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblMXhNvEvnEU3W2 \
  --json '{
    "客户名称": [{"id": "recCDUyzLO"}],
    "跟进阶段": "待接触",
    "业务价值": 150000,
    "商机描述": "客户对产品很感兴趣，初步沟通",
    "预计交易日期": "2026-06-30 00:00:00",
    "跟进销售人员": [{"id": "ou_8e7564b2971c064ae9af2b5b2444fdc8"}]
  }'
```

### 关键字段说明

- **客户名称**：link 类型，必须传入客户管理表的 record_id（不是客户名称文本）
- **跟进阶段**：如不指定，默认可留空
- **业务价值**：数字类型，不带千分位
- **预计交易日期**：datetime 格式 "YYYY-MM-DD HH:mm:ss"

---

## +商机更新

### 更新阶段

```bash
# 推进到下一阶段
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblMXhNvEvnEU3W2 \
  --record-id recXXXXXX \
  --json '{"跟进阶段": "协商议价"}'

# 标记赢单
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblMXhNvEvnEU3W2 \
  --record-id recXXXXXX \
  --json '{"跟进阶段": "赢单"}'

# 标记丢单（需填写原因）
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblMXhNvEvnEU3W2 \
  --record-id recXXXXXX \
  --json '{"跟进阶段": "丢失客户", "丢单原因": ["超客户预算"], "丢单原因备注": "客户预算上限50万"}'
```

**丢单原因枚举**：超客户预算（可能还有其他选项，可通过 `+field-search-options` 查看）

---

## +跟进记录

### 操作步骤

1. 先通过 `+客户查询` 获取目标客户的 record_id
2. 添加跟进记录

### 完整示例

```bash
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblKdCGHyWqK0JaR \
  --json '{
    "客户名称": [{"id": "recCDUyzLO"}],
    "跟进内容": "电话沟通客户需求，客户对方案很感兴趣",
    "跟进形式": "电话沟通",
    "跟进人员": [{"id": "ou_xxx"}],
    "陪同人员": [{"id": "ou_yyy"}]
  }'
```

### 简化示例

```bash
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblKdCGHyWqK0JaR \
  --json '{"客户名称": [{"id": "recCDUyzLO"}], "跟进内容": "拜访客户讨论方案"}'
```

**跟进形式枚举**：现场拜访、电话沟通

### 注意事项

- 跟进时间为 `created_at` 类型，自动记录创建时间，无需手动填写
- 如需上传照片附件，先创建记录获取 record_id，再通过 `+record-upload-attachment` 上传

---

## +合同查询

### 查询所有合同

```bash
lark-cli base +record-list \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblziKesi4SJyUbO \
  --limit 50
```

### 输出字段

1. 合同编号
2. 客户名称（关联显示）
3. 商机名称（关联显示）
4. 签约日期
5. 合同金额
6. 签约人员（lookup）
7. 合同录入

---

## +合同新增

### 操作步骤

1. 先获取关联的商机 record_id（通过 `+商机查询`）
2. 先获取关联的客户 record_id（通过 `+客户查询`）
3. 新增合同

### 完整示例

```bash
lark-cli base +record-upsert \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblziKesi4SJyUbO \
  --json '{
    "合同编号": "20260328-001",
    "商机名称": [{"id": "rec7VApeAO"}],
    "客户名称": [{"id": "recCDUyzLO"}],
    "签约日期": "2026-03-28 00:00:00",
    "合同金额": 500000,
    "合同录入": [{"id": "ou_xxx"}]
  }'
```

---

## +销售团队查询

```bash
lark-cli base +record-list \
  --base-token bascndXmwM3qJOVZjUR6LS4uSng \
  --table-id tblCgqK5WIclezX9 \
  --limit 50
```

### 输出字段

1. 销售姓名（formula 自动生成）
2. 销售区域
3. 销售 Leader

### 当前团队结构

| Leader | 区域 | 下属 |
|--------|------|------|
| 于小宁 | 华北/华南/华中/华东 | 沈小茜、黄泡泡、王小铭、万小奇、陈小葵、周北北 |
| 蔡佑程 | 华北/华中 | 董小飞、林小智 |

### 销售人员 ID 映射

用于写入 user 字段时的 `ou_id`：

| 姓名 | ou_id | 区域 | Leader |
|------|-------|------|--------|
| 沈小茜 | ou_b78fd580a0695b6bbfd30abb00175c7b | 华北 | 于小宁 |
| 黄泡泡 | ou_7d7ce9bbc232b53c71d8e184cb1d21e3 | 华南 | 于小宁 |
| 王小铭 | ou_8e7564b2971c064ae9af2b5b2444fdc8 | 华南 | 于小宁 |
| 万小奇 | ou_921de94cd11471ce6352b8ba638bbd0e | 华中 | 于小宁 |
| 董小飞 | ou_367ead136946e1c4ef1d378fc9a00dca | 华北 | 蔡佑程 |
| 林小智 | ou_5c04ada6ede37e5ce68ec53a05047140 | 华中 | 蔡佑程 |
| 陈小葵 | ou_584b7ff89e549184ca7c831846639853 | 华南 | 于小宁 |
| 周北北 | ou_911f79b574bbf7f1164370e9ac940545 | 华东 | 于小宁 |
