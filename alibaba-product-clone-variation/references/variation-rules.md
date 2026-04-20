# 商品差异化改写规则

本文件是 SKILL.md 第 5 步的执行细则。改写前必须读完，因为 SKILL.md 主体没有重复其中的禁用词、字符限制和格式约束。

## 你扮演什么角色

一个熟悉国际站 B2B 文案规范的卖点改写者。目标是让一组同款商品的卖点彼此不同，但产品事实完全一致，且符合平台合规要求。

## 改写的输入

调用方会传入：

- `sourceTitle` — 源商品标题
- `sourceSellingPoints` — 源商品的多条卖点（通常 3-5 条）
- `targetProductIds` — 一组目标商品 ID
- `riskBrandNames` — 该类目下的风控品牌词列表（可能为空）

## 总体原则

- **差异化比例 20%-30%**：每条目标商品的卖点集合，对比源商品有 1/4 到 1/3 的措辞发生变化。比例太低没有索引价值，太高容易丢掉原本调优好的关键词。
- **属性逐字保留**：所有数字、单位、材质名、规格参数（尺寸、重量、容量、电压、功率、Bluetooth 版本号、IP 防护等级等）必须照抄源文，不得替换或近似化。
- **目标商品之间互相不同**：N 个目标商品要产出 N 套不同的改写结果。如果偷懒用同一套套到所有目标，差异化的整个目的就废了。
- **语义对齐**：改后含义必须等价。不能把"防水"改成"防泼溅"，不能把"5 年保修"改成"长久保修"。

## 卖点改写策略库

每条目标商品，从源商品的 3-5 条卖点中，选 1-2 条做改写，其余保持原文。改写时从下面 5 种策略中挑 1-2 种，**不同目标商品要选不同的卖点条目和不同的策略组合**，避免出现"所有目标商品都改了第 2 条且都用同义替换"这种模式化结果。

### 策略 1：同义替换

替换形容词、动词为同义表达。

- 原：`High-quality stainless steel construction ensures long-lasting durability`
- 改：`Premium stainless steel build delivers exceptional longevity`

### 策略 2：句式重构

主动/被动互换，或调整主从句结构。

- 原：`Easy to install with simple plug-and-play design`
- 改：`Simple plug-and-play design makes installation effortless`

### 策略 3：视角转换

从产品特性切换为用户收益，或反过来。

- 原：`Supports multiple devices with Bluetooth 5.0 connectivity`
- 改：`Bluetooth 5.0 technology enables seamless multi-device connection`

### 策略 4：场景补充

加入具体使用场景，让卖点更具象。

- 原：`Waterproof design for outdoor activities`
- 改：`Waterproof construction ideal for hiking, camping, and beach use`

### 策略 5：顺序调整

把卖点的排列顺序换一下（不改写任何一条的内容，只是把"第 2 条"挪到"第 4 条"位置）。这条策略可以独立使用，也可以叠加到其他策略上。

## 标题处理

**默认不改标题**。除非用户在第 6 步反馈中明确要求改。

如果要改，仅做：

- 1-2 个修饰词的同义词替换
- 属性词的排列顺序微调
- **产品核心词必须保留**（如 `LED Strip Light` 中的 `LED Strip Light` 不能动）

## 禁用词清单

以下词出现在改写后的文案中，会被平台风控扫到：

| 类别 | 具体词 |
|------|--------|
| 绝对化用词 | `top`, `best`, `hot selling`, `only`, `unique`, `perfect`, `Exclusive` |
| 占位/空值类 | `N/A`, `Not Application`, `None`, `not specified`, `unspecified` |
| 联系方式 | 电话、微信、网址、telegram、whatsapp |
| 特殊符号 | `@ ！ ？ ? $ ^ [ ] { } ~ \` |
| 风控品牌词 | 来自 `riskBrandNames` 入参的所有词 |

## 格式要求

- **3 到 5 条卖点**
- 每条 **100 到 400 字符**，整段总长度 **≤ 2000 字符**
- 每条是自然完整的英文陈述句
- **介词不能放在句首**（不能以 `In`, `On`, `For`, `With` 之类开头）
- 每条采用 `卖点简述:详细内容` 格式（用冒号分隔，简述用 2-4 个词概括，例如 `Durable Build:Premium stainless steel...`）
- **不要用** `Selling point 1:xxx` 这种带编号的形式
- 全英文 + 英文标点
- 首字母大写；介词和连词（`in`, `on`, `at`, `and`, `or`, `for`, `the`, `a`, `an`）保持小写

## 输出结构

为每个目标商品产出如下结构（供 SKILL.md 第 6 步展示和第 7 步写入使用）：

```json
{
  "targetProductId": "<目标商品ID>",
  "productTitle": "<标题，默认与源商品一致>",
  "productSellingPoint": "<差异化卖点1>\n<差异化卖点2>\n<差异化卖点3>",
  "variationDetails": [
    {"item": "卖点2", "strategy": "同义替换", "original": "原文…", "modified": "改写…"},
    {"item": "卖点4", "strategy": "场景补充", "original": "原文…", "modified": "改写…"}
  ],
  "variationPercentage": 25
}
```

`productSellingPoint` 字段是真正写入平台的内容，多条卖点之间用 `\n` 分隔。`variationDetails` 是给用户看的解释，不写入平台。

## 写出后自查

每个目标商品的结果，按以下检查项过一遍。任意一项不通过都要重新改：

- [ ] 数字、单位、规格参数与源商品逐字一致
- [ ] 差异化比例落在 20%-30% 区间
- [ ] 与同批其他目标商品的改写策略不雷同
- [ ] 不含禁用词清单中的任何词
- [ ] 不含风控品牌词（`riskBrandNames` 中的词）
- [ ] 每条字符数在 100-400 之间，总长 ≤ 2000
- [ ] 介词没出现在句首
- [ ] 全英文，首字母大写规范
- [ ] 拼写和语法无误
