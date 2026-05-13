---
name: dtc-category-scanner
description: 探索 DTC 网站类目结构，统计 Top 5 核心一级类目的 SPU 数/均价/中位数，并同步写入飞书汇总行、品牌定位摘要、示例图片与 Wiki 品牌明细页。
user-invocable: true
metadata:
  requires:
    bins: ["lark-cli"]
---

# DTC 网站类目结构 → 飞书写回

当用户要你：
- 探索一个 DTC 网站卖什么类目
- 梳理一级类目结构与商品数
- 计算 Top 5 一级类目的 SPU 数、均价、中位数
- 获取站点总体商品数作为“货盘宽度”
- 把结果写入飞书汇总表指定行
- 生成“品牌定位 + 类目结构摘要”
- 把每个 Top 类目用于 concat 的 top 3 商品明细同步写入飞书 Wiki 品牌 sheet 页

使用这个 skill。

## 开始前必须加载的基础能力

### 1. 联网/网页探索
所有联网探索都使用 shared skill `web-access`。

站点探索、导航梳理、collection/category 页面识别、动态渲染、站内 JSON/API 探测，都按 `web-access` 的方式做。

### 2. 飞书读写
在做任何 Lark 写入前，先使用 shared skills：
- `lark-shared`
- `lark-sheets`

如果目标行包含图片单元格，直接按本 skill 内置的图片写入 SOP 执行，不再额外切换到单独的 image-write skill。

### 3. 外部依赖说明
这个 skill 默认依赖以下外部能力，但它们**不包含在当前 repo 内**：
- `web-access`：负责站点探索、页面读取与动态渲染场景
- `lark-shared`：负责飞书 CLI 的认证、权限与通用安全规则
- `lark-sheets`：负责表格读取、文本/数字写入与工作表操作

如果在独立仓库中复用本 skill，需要先确保这些能力已经可用。

## 输入

至少确认以下信息：
- 品牌域名
- 起始 URL（首页、类目页或 best sellers 页）
- 汇总表 URL
- 汇总表目标行号，或可用于定位目标行的品牌标识
- Wiki 明细表 URL（如果用户要求同步写品牌明细页）
- 是否需要示例图片
- 当前价格口径使用的币种（网站若多币种，先确认页面实际展示币种）

## 核心目标

产出四部分：
1. **Top 5 核心一级类目指标**
   - 类目名称
   - SPU 数
   - 平均价格
   - 中位价格
2. **货盘宽度**
   - 站点总体商品数
   - 与 Top 5 类目 SPU 数分开统计，不能横向相加替代
   - 写入汇总表同一行的 `I` 列
3. **品牌定位 + 类目结构摘要**
   - 总结核心售卖方向、类目结构、价格水平
   - 如果摘要里提到均价，必须写成**整数**，不要保留小数
   - 写入汇总表同一行的 `J` 列
4. **Wiki 品牌明细页**
   - 为每个 Top 类目锁定用于 concat 的 exact top 3 商品
   - 将这同一批商品的明细同步写入每品牌独立 sheet 页

## 标准流程

### Step 1：探索网站类目结构

用 `web-access` 探索：
- 顶部导航 / mega menu
- collection/category 页
- sitemap / products.json / GraphQL / 搜索接口 / 页面嵌入 JSON
- 类目页分页或 lazy load 方式

优先识别站点原生一级 merchandising 结构，而不是先套固定模板。

### Step 2：确定可比较的一级类目层级

将站点中可比较的“一级类目”整理出来。

优先使用：
- 站点原生一级导航类目
- 一级 collection
- 一级 merchandising bucket

默认排除营销型集合，除非用户明确要求纳入：
- New Arrivals
- Best Sellers
- Sale
- Gifts
- 临时 campaign 集合

详细规则见：
- `references/category-metric-rules.md`

### Step 3：采集商品并计算类目指标

对每个候选一级类目：
- 采集商品集合
- 以去重后的 PDP / product handle 作为 SPU
- 计算：
  - SPU 数
  - 均价
  - 中位数

然后按 SPU 数降序选出 Top 5 一级类目。

如果有效一级类目不足 5 个：
- 保留已有类目
- 剩余槽位留空
- 不强行补伪类目

详细统计口径见：
- `references/category-metric-rules.md`

### Step 4：获取货盘宽度

在类目探索完成后，额外获取站点总体商品数，作为汇总表中的 `货盘宽度`。

推荐按以下优先级尝试：
1. Shopify `collections/all/products.json` 或站点级 `products.json`，分页后按 product handle / canonical PDP 去重
2. Shopify Storefront GraphQL（标准 JSON 被拦截时）
3. 站内搜索结果总数（搜索页能稳定返回总数时）
4. `sitemap.xml` / `sitemap_products_*.xml` 中的产品 URL 数量

强制要求：
- `货盘宽度` 是站点级总商品数，不是 Top 5 类目商品数之和
- 必须记录采用了哪一种方法、统计口径、数据来源与局限性
- 统计过程要同步写入一个可回溯的本地 notes 文件，推荐路径为 `scraping_notes/{domain}.md`

### Step 5：生成品牌定位 + 类目结构摘要

摘要应覆盖：
- 品牌核心售卖方向
- Top 5 一级类目结构
- 价格水平
- 套装/主题系列/高客单补充结构（如果存在）

**强制规则：**
- 摘要里的均价要写成整数，如：`均价约 $18`
- 不要写成：`$17.85`
- 这个“整数化”规则只作用于摘要文案，不强制改飞书数值单元格本身

### Step 6：写入飞书前先读表头

每次写入前都必须：
- 读取汇总表表头
- 读取汇总表目标行附近区域
- 如果还要写 Wiki 品牌明细页，先读取一个现有品牌页的表头作为实时 schema 参考
- 确认列位置、目标行号、品牌页 schema 是否正确

不能依赖记忆中的列位。

详细写法见：
- `references/lark-write-sop.md`

### Step 7：锁定每个 Top 类目的 exact top 3 商品

如果用户要求示例图片，或要求同步写 Wiki 品牌明细页：
1. 对每个 Top 类目，从用于统计该一级类目的主 collection/category 页面默认顺序中选取商品
2. 如果页面存在显式 sort 参数或个性化排序，优先使用无个性化的默认排序链接
3. 在同一类目内按 product handle / PDP 去重，并避免重复 image URL
4. 选择前 3 个有效商品；若不足 3 个，则保留 1-2 个
5. 这批商品一旦锁定，后续 concat 图片和 Wiki 明细行都必须复用这一相同集合

### Step 8：写入 Wiki 品牌明细页

如果用户要求同步品牌明细页：
1. 在目标 Wiki 明细表中直接新建一个**空白** sheet 页，不要复制旧页；只有用户明确要求时才允许复制
2. 页名默认使用品牌域名（不含 www）
3. 新建后先读取该页并确认 sheet_id / 表头范围
4. 先写 header row，再批量写 detail rows
5. 明细列优先复用项目现有 A:I 结构：
   - A `domain`
   - B `rank`
   - C `title`
   - D `category`
   - E `category_en`
   - F `price_usd`
   - G `image_url`
   - H `product_url`
   - I `manual_category`
6. 明细行顺序固定为：Top 类目排名升序 → 类目内 top3 顺序升序

### Step 9：写入目标行

常见写入内容：
- `I` 列：货盘宽度（站点总体商品数）
- `J` 列：品牌定位 + 类目结构摘要
- `L:AJ`：Top 5 一级类目（名称 / SPU 数（商品数） / 示例图片 / 平均价格 / 中位价格）

当前这套汇总表 Top 5 区域默认包含“示例图片”列，因此默认流程**包含示例商品图片**。

默认执行方式：
1. 复用 Step 7 已锁定的 exact top 3 商品，不重新选图源商品
2. 下载这批商品的头图到本地临时目录或用户指定目录，并横向拼接成一张 concat 图
3. 先用 `lark-cli sheets +write` 写文本与数字
4. 再用 `lark-cli sheets +write-image` 按**单个单元格 range**逐格写入 concat 图
5. 最后回读验证

图片写入 SOP：
- 每次写图前，必须先读取汇总表表头和目标行附近区域，确认当前 sheet、图片列、目标行和精确单元格 range 都正确
- 如果同一行同时写文本和图片，必须先写文本/数字，再写图片；不要反过来，后续普通 `+write` 可能覆盖图片单元格
- `+write-image` 固定使用单个单元格范围，例如 `Tjn2D7!N46:N46`，不要把多个图片单元格合并成一次写入
- 写图后必须重新读取目标区域，确认目标品牌行仍正确、图片单元格已存在 embed-image 对象，且未被后续文本覆盖
- 如果业务影响很大，再到飞书 UI 中手动确认图片确实嵌入单元格，而不是浮动图片
- 从表格读取到的 `embed-image` JSON 只是读取结果，不可直接作为普通 `+write` 的单元格值写回；恢复真实图片时，必须先拿到本地图片，再调用 `+write-image`

只有在以下情况才不写图片：
- 用户明确要求跳过示例图片
- 目标表实时表头确认没有图片列

除非用户明确要求，否则不要使用 `IMAGE(...)` 公式方案。
当前 skill 的默认且唯一标准图片流程是：本地生成 concat 图后，使用 `lark-cli sheets +write-image` 逐格写入；不要在同一次任务中混用两种图片方案。

### Step 10：回读验证

写完后必须回读目标区域，确认：
- 摘要文案已更新
- Top 5 类目名称顺序正确
- SPU 数、均价、中位数落到了正确单元格
- 如果摘要里写了均价，展示的是整数
- 如果有图片列，图片没有被后续普通写入覆盖
- 如果还写了 Wiki 品牌明细页，header 与 detail rows 已落到正确 sheet，且 detail rows 与 concat 图片使用的是同一批商品

## 强制规则

1. 必须先探索站点实际类目结构，再确定一级类目
2. Top 5 按 SPU 数排序，不按主观印象排序
3. SPU 以去重后的 product handle / PDP 为准，变体不拆开算
4. `货盘宽度` 与 Top 5 类目 SPU 数必须分开统计，不能把类目数相加替代站点总商品数
5. 飞书写入前必须重新读表头；如果同步写 Wiki 品牌明细页，也必须先读实时 schema
6. 摘要中的均价必须写整数
7. 如有图片列，先锁定每类目 exact top 3 商品，再按“文本/数字先写，图片后写”执行
8. Wiki 品牌明细页中的商品行必须与 concat 图片实际使用的商品完全一致，不得在后续步骤重新改选 top 3
9. 每次写入 `货盘宽度` 前，都要把统计方法、来源与局限性写入一个可回溯的本地 notes 文件，推荐路径为 `scraping_notes/{domain}.md`

## 推荐复用的现有模板

- 站点 exploration / 类目采样 / 飞书写回流程可以按本 skill 的标准流程直接执行
- 如果用户所在仓库已有同类 DTC 采集模板，可将其作为实现细节参考，但不要把外部模板当作本 skill 的必需依赖

## 参考文件

- `references/category-metric-rules.md`
- `references/lark-write-sop.md`
- `references/examples.md`
