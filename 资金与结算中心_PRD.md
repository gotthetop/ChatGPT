# 资金与结算中心 PRD

## 一、文档信息

| 版本 | 日期 | 编辑人 | 说明 |
| --- | --- | --- | --- |
| V1.0 | 2026-06-09 | 产品部 | 基于 Amazon 新版 Summary 与 Finance API 重构资金与结算中心 |
| V1.1 | 2026-06-09 | 产品部 | 明确经营账为 Summary 报告区间经营口径，补充利润分析边界与客户侧 CSV 口径 |

## 二、功能定义

### 2.1 功能名称

资金与结算中心

### 2.2 功能定位

统一展示企业的经营账、结算账、资金账，帮助财务、老板、运营负责人和客户成功快速识别：

- 平台经营赚了多少钱
- 亚马逊按结算口径释放了多少钱
- 实际到账多少钱
- Summary 报告区间与接口结算时间之间的差异在哪里
- 差异能否追溯到交易级明细

本功能不再以“绝对平账”为目标，而是以“差异识别、差异解释、资金追踪”为目标。

### 2.3 菜单位置

```text
财务
  资金账户
    资金与结算中心
```

旧功能“回款对账”沉淀为本中心内的资金到账核对能力，不再作为单独的主入口。

### 2.4 一期范围

一期仅支持 Amazon。

支持内容：

- Summary PDF 上传与解析
- Finance API 交易明细同步
- Finance API 结算核算聚合
- 多语言 Summary 科目映射到美国标准科目
- S / T / A 三行结构展示
- 经营账、结算账、资金账统一看板
- 差异识别、差异说明、交易级溯源
- Transfer 与万里汇到账核对
- 页面导出、差异导出、资金导出、结算账导出

不在一期范围：

- TikTok Shop、Temu、Walmart、Shopify 的完整数据接入
- 税务申报自动生成
- 多平台资金预测
- 手工维护复杂会计凭证
- 完整利润组成分析、SKU 利润、广告/采购/头程等深度利润分析

说明：资金与结算中心可展示利润摘要和一级科目构成，但完整利润分析仍放在“结算利润”或“利润报表”模块。

## 三、核心结论

### 3.1 最新数据口径

新版 Amazon 文件与接口存在天然口径差异：

| 账层 | 名称 | 来源 | 统计基准 | 核心用途 |
| --- | --- | --- | --- | --- |
| S | Summary 汇总层 | 用户上传 Summary PDF | Summary 报告区间经营口径 | 经营账、税务报表、收入确认 |
| T | Transaction 明细派生层 | Finance API transaction 明细 | 交易级 + 结算释放状态 / 释放日期 | 差异溯源、交易级解释 |
| A | Accounting 结算核算层 | Finance API 同源数据聚合 | 结算编号组 / 账期 / financial event group | 系统核算主数据、应收应付 |
| F | Fund 资金层 | Amazon Transfer + 万里汇 | Transfer Date / 到账时间 | 现金流、到账核对 |

### 3.2 关键变更

原方案中 T 层来自用户上传 Transaction CSV。最新方案改为：

```text
T 层不再要求用户上传 CSV。
T 层由 Finance API transaction 明细沉淀后派生。
CSV 仅作为后台兜底导入、问题排查和接口异常时的临时补采来源。
```

这样可以消除用户导入负担，也避免现有系统中“解析报表出错”导致的伪差异。

对客户侧表述应统一为：

```text
客户不需要再上传 Transaction CSV。
系统会通过 Amazon Finance API 自动生成交易明细层，用于解释 Summary 与结算接口之间的差异。
CSV 仅作为后台排查或接口异常时的兜底资料。
```

### 3.3 T 与 A 的关系

T 和 A 来自同一批 Finance API 接口数据。

- A 是系统核算主数据，按结算编号组、账期或 financial event group 组织
- T 是交易级明细展开，按 transaction status、release date、标准科目归类

因此：

```text
T ↔ A 的差异不定义为业务差异。
若 T ↔ A 不一致，判定为系统同步、归类、分页、快照或聚合异常。
业务差异主要看 S ↔ API。
```

T 的价值不是“再做一次独立对账”，而是把 S 与接口之间的差异落到交易级、科目级、释放状态级。

### 3.4 Summary PDF 的限制

Summary PDF 不包含结算释放日期，也不包含交易释放状态。

因此：

- PDF 不能用于精确计算 Released / Deferred
- PDF 不能按结算释放日期归月
- PDF 只能作为 Summary 报告区间汇总口径

释放状态、释放日期、延迟结算等信息应来自 Finance API transaction 明细。

### 3.5 附件样本验证结论

基于用户提供的 Amazon 后台文件：

- `2026MarMonthlyUnifiedSummaryE101-US.pdf`
- `2026MarMonthlyUnifiedTransactionE101-US.csv`

验证结果：

- PDF 不包含 `Transaction Status`
- PDF 不包含 `Transaction Release Date`
- PDF 不包含 `settlement id`
- CSV 表头包含 `Transaction Status` 与 `Transaction Release Date`
- CSV 中存在 `Released` 与 `Deferred`
- CSV 中 Deferred 行的 release date 可能为空
- 同一个 2026-03 posted date 报告中，release date 可以跨到 2026-04

这说明 S 的报告区间口径与接口释放时间口径天然不同，系统必须接受差异并解释差异。

## 四、业务模型

### 4.1 S：Summary 汇总层

来源：

- 用户上传 Amazon Summary PDF

职责：

- 解析经营账汇总
- 支持税务报表口径
- 展示 Income / Expenses / Tax / Transfers
- 与接口口径做差异对比

统计基准：

- Summary PDF 报告区间经营口径
- 例如：Account activity from Mar 1, 2026 through Mar 31, 2026

说明：

- 经营账不是结算到账确认口径
- 经营账不建议表述为“配送确认口径”
- 对客户可表述为“Amazon Summary 报告区间内的经营销售、费用、税费和转账汇总”

限制：

- 无交易级明细
- 无结算释放日期
- 无 Transaction Status
- 无 SettlementId

### 4.2 T：Transaction 明细派生层

来源：

- Finance API transaction 明细

职责：

- 沉淀交易级台账
- 记录 transaction status
- 记录 maturity date / release date
- 按 release month、standard account、store、currency 重新聚合
- 解释 S 与接口之间的差异

核心字段：

- transaction_id
- posted_date
- transaction_status
- maturity_date / release_date
- settlement_id / related_identifier
- amount
- currency
- marketplace
- store
- transaction_type
- original_account_name
- standard_account_code
- standard_account_name

状态：

- Released
- Deferred
- Deferred Released

说明：

- Deferred 交易可能暂无 release date
- Released 或 Deferred Released 可用于按释放日期归月
- API 数据应以快照版本沉淀，避免状态回刷导致历史报表不可追溯

### 4.3 A：Accounting 结算核算层

来源：

- Finance API 同源数据聚合

职责：

- 作为系统核算主数据
- 按结算编号组、账期、financial event group 或系统核算批次组织
- 输出应收金额、已释放金额、延迟金额、待释放金额

与 T 的关系：

- A 是聚合结果
- T 是明细展开
- 两者应来自同一数据快照

### 4.4 F：Fund 资金层

来源：

- Amazon Transfer
- 万里汇到账流水

职责：

- 平台转账核对
- 实际到账核对
- 手续费、汇损识别
- 未到账、部分到账、超期到账预警

核心字段：

- transfer_id
- transfer_date
- expected_amount
- transferred_amount
- received_amount
- fee
- exchange_loss
- worldfirst_flow_no
- received_time

## 五、多语言标准科目映射

### 5.1 背景

不同国家 Summary PDF 使用不同语言展示科目，例如：

- 瑞典：Intäkter / Utgifter / Moms / Överföringar
- 德国：Einnahmen / Ausgaben / Steuer
- 法国：Revenus / Dépenses / Taxe
- 日本：収入 / 支出 / 税金

如果原样展示，跨国店铺财务无法快速比较。

### 5.2 目标

所有国家的 Summary 详情页统一展示美国标准科目：

- Income
- Expenses
- Tax
- Transfers
- Product sales
- FBA fees
- Selling fees
- Shipping credits
- Promotional rebates
- Marketplace withheld tax

同时保留原始语言科目，用于追溯和解析异常排查。

### 5.3 映射来源

用户提供的旧版文件：

```text
summary解析（翻译对照）.xlsx
```

可作为新版标准科目字典初始化来源。

该文件已包含：

- 美国标准科目
- 中文翻译
- 费用解释
- 借方 / 贷方
- transaction 明细来源
- API 交易明细来源
- 多国家语言映射

### 5.4 建议数据结构

标准科目表：

```text
account_standard
```

| 字段 | 说明 |
| --- | --- |
| standard_code | 标准科目编码 |
| standard_en_name | 美国标准英文名 |
| zh_name | 中文名 |
| category | Income / Expenses / Tax / Transfers |
| debit_credit | 报表展示方向 |
| sign_rule | 系统正负号规则 |
| description | 前端释义 |
| transaction_formula | T 层映射公式 |
| api_formula | A 层聚合公式 |
| enabled | 是否启用 |

多语言映射表：

```text
account_locale_mapping
```

| 字段 | 说明 |
| --- | --- |
| standard_code | 标准科目编码 |
| marketplace | marketplace id |
| country | 国家 |
| language | 语言 |
| local_name | 原始语言科目 |
| normalized_local_name | 归一化科目名 |
| report_type | Summary PDF / API / CSV fallback |
| valid_from | 生效时间 |
| valid_to | 失效时间 |

### 5.5 注意事项

借方 / 贷方不能直接等同于正负号规则。

Summary PDF 中 Debits / Credits 是报表展示方向；Finance API 或交易明细金额通常已经自带正负号。系统必须单独维护 `sign_rule`，避免重复反向。

## 六、数据同步与计算流程

### 6.1 数据流

```text
用户上传 Summary PDF
        |
        v
S：Summary 汇总层
        |
        v
多语言科目映射为美国标准科目

Finance API 定时同步
        |
        +--> transaction 明细台账
        |       |
        |       v
        |    T：交易级差异溯源层
        |
        +--> 结算核算聚合
                |
                v
             A：系统核算主数据

Amazon Transfer + 万里汇
        |
        v
F：资金到账核对

S / T / A / F
        |
        v
资金与结算中心统一展示
```

### 6.2 Finance API 同步策略

系统应建立 transaction 明细台账，不应在页面查询时临时拼装。

建议流程：

1. 按店铺授权定时拉取 Finance API transaction 明细
2. 按 posted date 做增量同步
3. 保存 transaction status、maturity date、amount、currency、related identifier
4. 对 Deferred 状态保留追踪任务
5. 状态更新为 Deferred Released 后刷新释放日期与释放金额
6. 每次同步生成数据快照版本
7. 页面按 release month、store、currency、standard account 聚合展示

状态快照要求：

- Finance API 能区分延迟发放与已发放，但 API 查询结果通常反映交易当前状态
- 例如 2026-03 的 Deferred 交易，到 2026-06 再导出时可能已经全部变成 Released 或 Deferred Released
- 系统不能只保存当前最终状态，否则无法还原月末时点的延迟结算金额
- transaction 明细台账需要保存状态生命周期，包括首次同步状态、每次状态变更、释放时间、当前状态
- 页面需要同时支持“当前状态视图”和“历史快照视图”

建议保存：

| 字段 | 说明 |
| --- | --- |
| first_seen_status | 首次同步时的状态 |
| current_status | 当前最新状态 |
| status_changed_at | 状态变更时间 |
| maturity_date | 预计或实际释放时间 |
| released_at | 实际释放时间 |
| snapshot_version | 数据快照版本 |
| snapshot_time | 快照生成时间 |

口径说明：

```text
当前状态用于判断现在是否已经释放。
历史快照用于还原当月月末的延迟结算情况。
```

### 6.3 PDF 上传策略

用户只需要上传 Summary PDF。

系统自动识别：

- 平台
- 国家
- 店铺
- 币种
- 报告月份
- Summary 报告区间
- 原始语言科目

同一店铺同一月份重复上传：

- 默认最新版本参与核算
- 保留历史版本
- 可查看版本差异

### 6.4 CSV 定位

Transaction CSV 不作为客户日常必传资料。

CSV 仅用于：

- Finance API 暂不可用时的后台临时补采
- 客服 / 实施排查 Amazon 后台原始报表
- 验证接口与后台报表字段差异
- 异常工单附件

页面不向普通用户暴露“必须导入 CSV”的交互。

## 七、页面结构

### 7.1 首页看板

顶部卡片：

| 卡片 | 指标 |
| --- | --- |
| 经营账 | 经营收入、经营支出、经营利润、经营利润率 |
| 结算账 | 已释放金额、延迟结算金额、待释放金额、释放率 |
| 资金账 | 应到账金额、已到账金额、到账率、未到账金额 |
| 风险预警 | 存在差异店铺、延迟结算金额、超 7 天未到账金额、异常店铺数量 |

利润展示边界：

- 可展示经营利润、结算净额、释放金额对现金流的影响
- 可展示 Income / Expenses / Tax / Transfers 一级构成
- 不展示 SKU 利润、广告成本、采购成本、头程成本、费用率趋势等深度利润分析
- 提供“查看完整利润分析”入口，跳转至结算利润或利润报表模块

中部模块：

- 三账趋势
- 资金链路
- 数据源职责
- 差异预警排行
- 本月数据状态

### 7.2 对账列表

聚合主键：

```text
平台 + 国家 + 店铺 + 币种 + 月份
```

字段：

- 月份
- 国家
- 店铺
- 币种
- Summary 净额
- API 结算净额
- 已释放金额
- 延迟结算金额
- 释放状态
- 释放日期
- 应转金额
- 到账金额
- 差异率
- 对账状态
- 到账状态
- 差异说明
- 操作

### 7.3 详情页

详情页使用抽屉或独立页。

Tab：

- 经营账 S
- 交易明细 T
- 结算核算 A
- 资金账 F
- 差异分析

#### Tab1：经营账 S

展示：

- Summary PDF 来源
- 报告区间
- 原始语言科目
- 美国标准科目
- Income
- Expenses
- Tax
- Transfers
- 所有解析项
- 文件版本

#### Tab2：交易明细 T

展示：

- Finance API transaction 明细
- transaction status
- maturity date / release date
- transaction type
- amount
- standard account
- related identifier
- 可下钻到交易级

#### Tab3：结算核算 A

展示：

- settlement id / financial event group
- 已释放金额
- 延迟金额
- 待释放金额
- API 聚合批次
- T 与 A 的系统一致性检查
- 结算净额一级构成
- 查看完整利润分析入口

不展示：

- SKU 级利润
- 商品成本、采购成本、头程成本
- 广告利润归因
- 深度费用率分析

#### Tab4：资金账 F

展示：

- Transfer amount
- expected amount
- received amount
- fee
- exchange loss
- received time
- 万里汇流水号

#### Tab5：差异分析

自动生成：

- S 与 API 是否存在差异
- 差异金额
- 差异率
- 差异来源
- 可解释金额
- 不可解释金额
- 交易级溯源入口

示例：

```text
Summary PDF 统计的是 2026-03 报告区间。
Finance API 中部分 2026-03 posted transactions 的 release date 落在 2026-04。
因此 S 与 API 在 2026-03 口径下存在时间差。
可追溯交易 1,598 条，合计 $7,026.50。
```

## 八、筛选条件

- 国家
- 店铺
- 币种
- 月份
- 平台
- 统计维度：全部 / 经营账 / 交易明细 / 结算账 / 资金账
- 对账状态：全部 / 平账 / 轻微差异 / 平台差异 / 系统异常
- 到账状态：全部 / 未到账 / 部分到账 / 已到账 / 超期未到账
- 数据状态：Summary 已上传 / Summary 未上传 / API 已同步 / API 异常
- 关键词：店铺名称、SettlementId、TransferId、万里汇流水号、标准科目、原始科目

## 九、状态规则

### 9.1 对账状态

| 状态 | 规则 |
| --- | --- |
| 平账 | S 与 API 差异在阈值内 |
| 轻微差异 | 绝对值小于 1 USD 或等值币种 |
| 平台差异 | 绝对值大于等于 1 USD，且可被 T 层解释 |
| 系统异常 | T 与 A 不一致，或 API 同步/聚合异常 |
| 待补 Summary | API 已同步，但 Summary PDF 未上传 |
| 待同步 API | Summary 已上传，但 API 未同步完成 |

### 9.2 到账状态

| 状态 | 规则 |
| --- | --- |
| 未到账 | 到账金额 = 0 |
| 部分到账 | 到账金额 < 应到账金额 |
| 已到账 | 到账金额 >= 应到账金额 |
| 超期未到账 | Transfer 后超过 N 天仍未到账 |

### 9.3 系统一致性状态

| 状态 | 规则 |
| --- | --- |
| 一致 | T 聚合金额 = A 核算金额 |
| 不一致 | T 聚合金额 != A 核算金额 |
| 待重算 | T 或 A 数据版本更新，但核算结果未刷新 |

## 十、导入与同步管理

页面名称建议：

```text
数据同步
```

模块：

- API 同步状态
- Summary PDF 上传
- 多语言映射状态
- 数据快照版本
- 异常日志

### 10.1 API 同步状态

字段：

- 店铺
- 国家
- 币种
- 同步月份
- transaction 同步状态
- settlement 聚合状态
- 最新同步时间
- 数据快照版本
- 异常原因
- 操作：重新同步 / 重新核算 / 查看日志

### 10.2 Summary PDF 上传

字段：

- 店铺
- 国家
- 币种
- 报告月份
- Summary PDF 状态
- 最近上传时间
- 上传人员
- 解析状态
- 映射状态
- 操作：上传 / 查看 / 重新解析

### 10.3 CSV 兜底入口

仅后台角色可见。

字段：

- CSV 文件
- 使用原因
- 关联工单
- 覆盖范围
- 是否参与正式核算

默认不参与正式核算，仅用于排查。

## 十一、权限

权限项：

- 查看中心
- 查看详情
- 上传 Summary PDF
- 重新解析 Summary
- 触发 API 同步
- 重新核算
- 导出
- 查看资金流水
- 查看接口日志
- 维护标准科目字典
- 维护多语言映射
- 后台 CSV 兜底导入
- 店铺数据权限

## 十二、导出

导出类型：

- 页面导出
- 差异导出
- 资金导出
- Summary 经营账导出
- Transaction 明细导出
- API 结算账导出
- 映射异常导出

导出内容需带：

- 数据源
- 数据快照版本
- 统计口径
- 币种
- 生成时间
- 生成人

## 十三、异常与风控

异常类型：

- Summary 未上传
- Summary 解析失败
- Summary 科目未映射
- API 未授权
- API 同步失败
- API 数据延迟
- T 与 A 聚合不一致
- Transfer 未匹配
- 万里汇流水未匹配
- 超期未到账
- 手续费异常
- 汇损异常

处理方式：

- 列表标记异常
- 详情页展示原因
- 支持导出异常
- 支持重新同步 / 重新解析 / 重新核算
- 支持后台工单排查

## 十四、原型页面

原型包含四个主视图：

1. 首页看板
2. 对账列表
3. 数据同步
4. 导出记录

重点交互：

- 首页直接说明 S / T / A / F 数据源职责
- 对账列表展示差异状态和到账状态
- 详情抽屉展示 S / T / A / F 与差异分析
- 数据同步页展示 API 同步、Summary 上传、映射状态
- 不再要求普通用户上传 Transaction CSV

## 十五、研发实现要点

### 15.1 数据快照

Finance API 数据需要快照化。

原因：

- Deferred 状态会变化
- Deferred Released 后金额与释放日期会回刷
- 历史报表必须可追溯

### 15.2 幂等与重算

同一店铺、月份、数据源、快照版本应支持幂等重算。

重算触发条件：

- Summary PDF 新版本上传
- Finance API 新快照同步完成
- 多语言映射字典更新
- 资金流水重新匹配

### 15.3 正负号处理

系统必须统一金额方向：

- Summary PDF 按 Debits / Credits 展示
- API / transaction 明细金额通常自带正负号
- Transfer / 万里汇金额按资金流入流出展示

需要在标准科目层定义 sign_rule，禁止在解析层临时硬编码。

### 15.4 T 与 A 一致性校验

系统应提供内部校验：

```text
T 按结算组聚合金额 = A 结算核算金额
```

不一致时标记为系统异常，不进入业务差异解释。

## 十六、验收标准

### 16.1 数据验收

- Summary PDF 可正确识别店铺、国家、币种、月份
- Summary 科目可映射为美国标准科目
- Finance API transaction 可沉淀到 T 层
- A 层可由同源 API 数据聚合生成
- T 与 A 可做一致性校验
- Transfer 与万里汇可匹配到账

### 16.2 页面验收

- 首页可展示经营账、结算账、资金账、风险预警
- 对账列表可按店铺月份查看 S / API / F 差异
- 详情页可查看 S / T / A / F
- 差异分析可解释时间差、释放差异、到账差异
- 数据同步页不要求普通用户上传 CSV
- CSV 入口仅后台可见

### 16.3 权限验收

- 普通财务可查看、上传 Summary、导出
- 管理员可重新同步、重新核算
- 后台角色可维护映射字典和使用 CSV 兜底入口
- 店铺权限生效

## 十七、后续规划

二期：

- 支持 TikTok Shop
- 支持 Temu
- 支持 Walmart
- 支持 Shopify
- 支持多平台统一资金中心
- 支持资金预测
- 支持自动生成财务说明

三期：

- 接入税务申报口径
- 生成跨平台经营利润报告
- 支持异常自动工单
- 支持多币种资金预测与汇损分析
