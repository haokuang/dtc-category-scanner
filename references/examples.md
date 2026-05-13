# 示例

## 示例 1：仅写摘要 + Top 5 指标

### 用户目标
- 探索某品牌网站类目结构
- 统计 Top 5 核心一级类目
- 写入飞书指定行

### 预期输出
- 摘要：
  - `以平价高显色彩妆为核心，眼妆/唇妆/颊彩为主，辅以底妆、工具与少量护肤；核心一级类目均价整体处于低中价位，Eyes约$18、Lip约$33、Cheek约$26、Face约$24、Tools约$23。`
  - 上述摘要中的均价来自数值字段后再四舍五入，例如：17.85 → 18、33.02 → 33、25.73 → 26、24.09 → 24、23.45 → 23
- Top 5 指标：
  - Eyes / SPU 47 / 17.85 / 15
  - Lip / SPU 41 / 33.02 / 15
  - Cheek / SPU 33 / 25.73 / 18
  - Face / SPU 32 / 24.09 / 18
  - Tools / SPU 11 / 23.45 / 18

注意：
- 摘要中的均价写整数
- 数值单元格中的平均价格/中位数仍可保留原始精度

## 示例 2：默认包含示例图片时的写入顺序

1. 先读取表头和目标行
2. 如果实时表头仍包含示例图片列，则按默认流程继续补图
3. 为每个类目取 Top 3 商品头图
4. 下载到本地并横向拼接成 concat 图
5. 先写摘要、类目名称、商品数、均价、中位数
6. 再逐格写示例图片
7. 回读验证

## 示例 3：站点探索的最小检查清单

至少确认：
- 首页一级导航有哪些类目
- 是否存在 collection/category 页面
- 是否能从 products.json / GraphQL / 页面 JSON 中稳定拿到商品和价格
- 当前页面价格币种是什么
- 哪些类目属于营销集合，应排除在 Top 5 一级类目之外

## 示例 4：汇总表 + 示例图片 + Wiki 品牌明细页联动

1. 先读取汇总表表头和目标行附近区域
2. 再读取 Wiki 明细表中一个现有品牌页的表头，确认实时 schema
3. 计算 Top 5 一级类目
4. 对每个 Top 类目锁定 exact top 3 商品
5. 先在 Wiki 明细表中新建品牌 sheet 页，并写入 header + detail rows
6. 再写汇总表里的摘要、类目名称、商品数、均价、中位数
7. 最后逐格写 concat 图片
8. 回读汇总表目标区域和 Wiki 品牌页已写区域

示例 detail rows：
- `juviasplace.com / 1 / COFFEE SHOP LIQUID EYESHADOW / Eyes / Eyes / 18 / <image_url> / <product_url> / Eyes`
- `juviasplace.com / 2 / NUBIAN SHADOW STICKS / Eyes / Eyes / 18 / <image_url> / <product_url> / Eyes`
- `juviasplace.com / 3 / EGYPTIAN PEN EYELINER - NOIR / Eyes / Eyes / 15 / <image_url> / <product_url> / Eyes`

其中：
- `rank` 是品牌明细页里的全局顺序号
- 对同一类目而言，连续的 1-3 行就是该类目 concat 图从左到右对应的商品顺序

注意：
- Wiki 明细页中的 3 行商品必须与 Eyes 类目的 concat 图来源完全一致
- 如果某个类目只有 2 个有效商品，则 detail rows 和 concat 也只保留 2 个
- 汇总表仍然保持“文本/数字先写，图片后写”
