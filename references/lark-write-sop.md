# 飞书写回 SOP

## 开始前必须先读

在任何飞书读写前，先使用 shared skills：
- `lark-shared`
- `lark-sheets`

如果目标区域含图片单元格，直接按本 skill 内置的图片写入 SOP 执行，不再额外切换到单独的 image-write skill。

## 1. 每次写入前先读表头

**不能依赖记忆中的列位。**

如果本次同时写汇总表和 Wiki 品牌明细页：
- 先读取汇总表表头区域
- 再读取汇总表目标行附近区域
- 再读取 Wiki 明细表中一个现有品牌页的表头，确认实时 schema

如果只写其中一处，也先读取对应目标区域。

示例：
```bash
lark-cli sheets +read \
  --url "<sheet_url>" \
  --range "<sheet_id>!A1:AJ3"

lark-cli sheets +read \
  --url "<sheet_url>" \
  --range "<sheet_id>!A50:AJ56"
```

确认：
- 当前 sheet 是否正确
- 摘要列位置
- Top 5 类目槽位位置
- 目标行号是否正确
- 如果还要写 Wiki 品牌明细页，现有品牌页的字段顺序是否正确

## 2. 当前项目常见列结构

这个项目当前在线表头对应的常见写法是：
- 货盘宽度：`I` 列
- 品牌定位 + 类目结构摘要：`J` 列
- best sellers 链接：`K` 列
- Top 5 类目区域：`L:AJ`

每个类目占 5 列：
- 类目名称
- 商品数
- 示例图片
- 平均价格
- 中位价格

但这只是**当前常见结构**，写之前仍然必须以实时表头为准。
如果旧模板或旧脚本里看到 `I/J` 含义相反，以实时表头为准，不要沿用旧记忆。

## 3. 先定位目标行 / 目标品牌页

如果用户没有直接给汇总表行号：
- 先读品牌识别列
- 根据域名/品牌名定位目标行

如果本次要补“货盘宽度”：
- 在写表前先确认 `I` 列表头确实是“货盘宽度”
- 先完成站点总体商品数统计
- 再把统计方法、来源、局限性写入一个可回溯的本地 notes 文件，推荐路径为 `scraping_notes/{domain}.md`

如果用户要求同步写 Wiki 品牌明细页：
- 先确认目标 Wiki 表 URL
- 默认为当前品牌新建一个空白 sheet 页，不复制旧页；只有用户明确要求时才允许复制
- sheet 页标题默认使用品牌域名（不含 www）
- 新建后先确认返回的 sheet_id，再读取该页 A1:I5 或当前实际表头范围

不要在未确认目标行或目标品牌页的情况下直接写入。

## 4. 先写文本和数字

先用 `lark-cli sheets +write` 写：
- `I` 列货盘宽度（站点总体商品数）
- `J` 列摘要文本
- 类目名称
- SPU 数（商品数）
- 均价
- 中位数
- 如果有 Wiki 品牌明细页，先写 header row 和 detail rows

写入时：
- range 必须是精确闭区间
- 不要写开放范围
- 数值保持数值类型

示例：
```bash
lark-cli sheets +write \
  --url "<sheet_url>" \
  --range "<sheet_id>!I55:J55" \
  --values '[[14536,"品牌定位摘要"]]'

lark-cli sheets +write \
  --url "<sheet_url>" \
  --range "<sheet_id>!L55:AJ55" \
  --values '[["Eyes",47,"",17.85,15,"Lip",41,"",33.02,15]]'
```

## 5. 默认补示例图片，并单格写图

当前项目这张汇总表的 Top 5 区域默认包含示例图片列，因此默认流程应补示例图片。

执行顺序：
1. 先为每个类目锁定 exact Top 3 商品集合
2. 默认以用于统计该一级类目的主 collection/category 页面默认顺序取商品；若页面存在显式 sort 参数或个性化排序，优先使用无个性化的默认排序链接；若不足 3 个则保留现有商品继续；同一类目内避免重复商品/重复图片
3. 这批商品同时作为：
   - concat 图的图片来源
   - Wiki 品牌明细页的 detail rows 来源
4. 下载到本地临时目录或用户指定目录，并横向拼接成一张 concat 图
5. 先完成文本/数字写入
6. 再按本 skill 内置的单格写图 SOP，逐格写 concat 图
7. 默认使用 `lark-cli sheets +write-image` 逐格写图；除非用户明确要求，否则不要使用 `IMAGE(...)` 公式方案
8. 不要把多张图混成一次批量写图，也不要在同一次任务中混用两种图片方案

只有在以下情况才跳过图片：
- 用户明确要求不写示例图片
- 实时表头确认当前目标区域没有图片列

示例：
```bash
lark-cli sheets +write-image \
  --url "<sheet_url>" \
  --range "<sheet_id>!N55:N55" \
  --image "./tmp/<brand>-<cat>-concat.jpg"
```

## 6. 摘要里的均价写法

摘要文案如果提到均价：
- 必须写整数
- 如：`Eyes 约 $18`
- 不写：`Eyes 约 $17.85`

数值单元格是否保留小数，按表格字段需求决定，不受摘要文案规则影响。

## 6. Wiki 品牌明细页写入 SOP

若用户要求同步写 Wiki 品牌明细页：
1. 先新建空白 sheet 页，不复制旧页
2. 页名默认使用品牌域名（不含 www）
3. 新建后立即读取该页表头区域
4. 先写 header row，再写 detail rows
5. detail rows 默认优先复用当前项目已验证的 A:I 结构：
   - A `domain`
   - B `rank`（当前流程中定义为品牌明细页的全局顺序号；按 Top 类目排名升序写入，同一类目内再按 top3 顺序写入，因此可用来回推 concat 内的先后顺序）
   - C `title`
   - D `category`
   - E `category_en`
   - F `price_usd`
   - G `image_url`
   - H `product_url`
   - I `manual_category`
6. detail rows 顺序固定为：Top 类目排名升序 → 类目内 top3 顺序升序；因此同一类目的连续 1-3 行就是该类目 concat 图对应的商品顺序
7. 每个 detail row 必须对应 concat 图实际使用的商品，不能后补另一批商品

示例：
```bash
lark-cli api POST "/open-apis/sheets/v3/spreadsheets/<wiki_spreadsheet_token>/sheets" \
  --data '{"title":"<brand_domain>"}'

lark-cli sheets +write \
  --url "<wiki_sheet_url>" \
  --range "<new_sheet_id>!A1:I4" \
  --values '[["domain","rank","title","category","category_en","price_usd","image_url","product_url","manual_category"],["juviasplace.com",1,"COFFEE SHOP LIQUID EYESHADOW","Eyes","Eyes",18,"https://...","https://...","Eyes"]]'
```

## 7. 写后回读验证

写完后必须读取目标区域：
```bash
lark-cli sheets +read \
  --url "<sheet_url>" \
  --range "<sheet_id>!J55:AJ55"
```

如果还写了 Wiki 品牌明细页，再回读品牌页：
```bash
lark-cli sheets +read \
  --url "<wiki_sheet_url>" \
  --range "<new_sheet_id>!A1:I20"
```

检查：
- `I` 列货盘宽度是否写到正确单元格
- `J` 列摘要是否写到正确单元格
- Top 5 类目顺序是否正确
- 商品数、均价、中位数是否在正确列
- 摘要里的均价是否为整数
- 若有图片列，图片未被普通写入覆盖
- Wiki 品牌明细页 header 是否正确
- Wiki 品牌明细页每个类目是否只写了与 concat 图一致的 1-3 个商品
- 推荐路径为 `scraping_notes/{domain}.md` 的本地 notes 文件是否已经补上货盘宽度统计方法与局限性

## 8. 常见错误

### 错误 1：没读表头就按旧记忆写列
后果：
- 写错列
- 覆盖别的字段

修复：
- 重新读表头
- 确认正确列位后重写

### 错误 2：先写图片，后写文本
后果：
- 图片被后续普通 `+write` 覆盖

修复：
- 重新写文本/数字
- 再逐格恢复图片

### 错误 3：摘要里仍保留两位小数
后果：
- 不符合当前项目摘要口径

修复：
- 将摘要中的均价改为四舍五入后的整数表达
