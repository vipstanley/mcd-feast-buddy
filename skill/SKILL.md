---
name: mcd-feast
description: 麦麦饭局 — 一句话发起麦当劳聚餐（到店自取），真实拉取门店菜单、自动领取并匹配麦麦省优惠券、用官方 calculate-price 报价、按"谁吃什么"AA 分账，默认停在下单前。当用户提到聚餐、团餐、部门下午茶、多人一起点麦当劳、AA 分账、凑单省钱时使用本 Skill。
---

# 麦麦饭局 McFeast（到店自取 · MVP）

把「一群人点麦当劳」这件扯皮的事，压缩成一句话。本版只做**到店自取**（`beType=1` / `orderType=1`）。

## 前置条件

- WorkBuddy 连接器已启用 `mcd-mcp`（`https://mcp.mcd.cn`，Streamable HTTP，`Authorization: Bearer <Token>`）
- 未配置时提示用户前往 <https://open.mcd.cn/mcp> 手机号登录 → 控制台 → 激活，领取 MCP Token
- **绝不**在对话中打印或回显 Token

## 怎么读工具返回（每个工具都适用）

- 工具返回文本的前半段是"字段说明"（Response Structure），**真正的数据在 `## Original Response` 之后的 JSON 里**，一律以该 JSON 为准
- `success:false` 时读 `code` 与 `message`，按文末「错误处理」表决定是回退、继续还是停下；不要猜测结果继续推进
- 单位：`calculate-price` 返回里的**金额字段**全部是**分**（÷100 展示）；`query-meals` 的 `currentPrice` 是**元**字符串。展示统一为 `¥xx.xx`
- 各工具**只传本文对应小节列出的入参**；`auto-bind-coupons` 的入参就是空对象 `{}`

## 全局硬约束（安全红线）

1. 总价、优惠、每行小计**只能来自 `calculate-price` 的返回**。禁止心算、估算，禁止把 `query-meals` 的现价相加当总价
2. 商品 code **只能来自 `query-meals` / `query-meal-detail` 的返回，或 `query-store-coupons` 返回的 `products[].productCode`**（券适用的商品可能不在当前菜单里——例如 麦辣鸡腿堡3件套 `9900004002` 不在门店 1450713 的菜单中，但 `calculate-price` 接受它并给出用券价）。禁止编造或凭记忆填写
3. 券**只能挂在 `query-store-coupons` 返回的 `products[].productCode` 对应的那一行**；每行最多一个 `couponCode`，同一个 `couponCode` 不得重复使用
4. 到店自取场景**不传 `beCode`**（传了会报错）
5. **下单授权只有一种形式**：用户针对当前这份报价明确说「下单」或「确认下单」。「就要这个」「可以」「好的」「选这个方案」只表示选定方案，**不算下单授权**。未获授权绝不调用 `create-order`——它会创建真实订单
6. 一次方案的工具调用预算 ≤ 10 次；缺信息就停下来问一次，不要猜
7. 券不可用、售罄、`429` 限流、`401` 失效时如实告知，不伪造结果、不编造价格

---

## 阶段一 · 选店选品（目标：拿到 `storeCode`，以及每个人的商品 code）

### 1.1 收集需求（自然语言，最多追问一轮）

- **必填**：人数；城市 + 地标/路名（用于搜店，例："上海市 人民广场"）
- **选填**：人均预算、忌口（不吃辣 / 不要牛肉 / 不吃猪肉 / 素食）、口味偏好、"谁吃什么"的指定
- 缺省值：不设预算上限、无忌口。缺城市或地标时追问一次；仍缺则说明无法搜店并停下

### 1.2 本版不做的事（不要被用户带偏）

- 不做外送 / 麦乐送 / 团餐（`delivery-*`、`beType=2/6`）：用户要外送就说明本版只支持到店自取
- 不做预约取餐（不传 `reservationDate`）
- 不做时间自检（不调 `now-time-info`）：连通性以第一个工具调用成功为准
- 不做多方案穷举比价：只算一份最终篮子（见 2.3）
- 不主动查活动、营养：`campaign-calendar` 仅在用户问"有什么活动"时调；`list-nutrition-foods` 仅在用户要求"控热量"时调

### 1.3 `query-nearby-stores`（1 次）

```json
{"searchType": 2, "beType": 1, "city": "上海市", "keyword": "人民广场"}
```

- `searchType=1`（收藏店）仅当用户明确说"我收藏的店"；收藏为空会返回 `code 600050`，此时改用 `searchType=2`
- 从 `data[]` 中取 `businessStatus:true` 且 `distance` 最小的门店为默认；向用户展示前 3 家的 `storeName / address / distance / storeCode`，用户不改就用默认
- 记住 `storeCode`（**字符串**，如 `"1450713"`）。此后 1.4 / 1.5 / 2.2 / 2.3 / 3.3 这些工具都带 `"storeCode": "<它>", "orderType": 1, "beType": 1`

### 1.4 `query-meals`（1 次）

```json
{"storeCode": "1450713", "orderType": 1, "beType": 1}
```

返回 `data.categories[].meals[].{code, tags}` 与 `data.meals[code].{name, currentPrice, originalPrice, discountType, canWithOrder}`。

**候选池规则**

- `tags` 含 `多人餐` → 多人餐（例：麦麦周末双人餐 ¥49.9，含 2 汉堡 + 2 小食 + 2 饮料）
- `tags` 含 `套餐` 或名称含 `三件套` → 单人套餐（¥22.9–37.5）
- `tags` 含 `单品`，或名称为 麦旋风 / 派 / 薯条 / 小食盘 → 加购或共享项
- **跳过** `discountType` 含"麦金卡"或 `tags` 含 `麦金卡` 的项（需随单购会员卡，MVP 不处理）
- 菜单很长（120+ 项、含图片链接，约 3 万字）：**不要向用户复述菜单**，只在内部记下候选池里 5–10 个 `code / name / currentPrice`，向用户只展示最终选中的项

**配餐启发式**（一次成型，不要反复推翻）

- 每 2 人配 1 份双人餐，余 1 人配 1 份三件套；预算宽裕再加 1 份共享项（麦旋风 / 小食拼盘）
- 总预算 = 人均预算 × 人数；超出时把一份三件套换成"随心配 1+1"（¥13.9 / ¥14.9），只调整一次

**忌口 = 菜名关键词排除**（不依赖营养表）

- 不吃辣 → 排除名称含 `辣`、`麦辣`、`龙焰`、`香骨鸡`；替代：板烧鸡腿堡（单品 `1406` / 三件套 `9900005462`）、麦香鸡套餐 `9900005453`
- 不要牛肉 → 排除 `巨无霸`、`牛堡`、`不素之霸`、`安格斯`、`培根蔬萃`
- 不吃猪肉 → 排除 `猪柳`、`培根`。注意：麦麦周末双人餐第 1 轮默认是猪柳蛋堡且**不可换**，该人不要分配到双人餐
- 素食 → 只配 薯条 / 麦旋风 / 玉米杯 / 派 等单品
- 控热量 → 仅在用户主动要求时调用 `list-nutrition-foods`

每个人都要落到具体的 code；多人餐要写明由哪两人共享。

### 1.5 `query-meal-detail`（≤ 2 次）

```json
{"storeCode": "1450713", "orderType": 1, "beType": 1, "code": "9900015151"}
```

- 返回 `data.rounds[]`，每轮 `choices[]`：`isDefault=1` 为默认项，`diffPrice` 为换项差价（如 `"+ ¥1"`）
- **何时必须调**：给有忌口的人分配的套餐 / 多人餐，若从名称看不出全部子项（如"麦麦周末双人餐"），分配前先查详情核实默认子项是否冲突；名称已说明内容的三件套（如"板烧鸡腿堡三件套"）不用查。无忌口的人不查
- **替换而非放弃**：默认子项冲突时，在**同一轮**的 `choices` 里挑一个不冲突的 code（例：双人餐第 2 轮默认 麦辣鸡腿汉堡 `1440` → 板烧鸡腿堡 `1406`，+¥1）；同一轮没有可替换项（如双人餐第 1 轮只有猪柳蛋堡）→ 该人改配已核实的单人套餐或单品
- 记下 `{轮次 id, 新 code, quantity}`，阶段二作为 `roundList` 传入
- 本文出现的示例 code 只是实测当天的值，**必须以本次返回里出现的 code 为准**

---

## 阶段二 · 报价（目标：一份 `calculate-price` 返回，作为唯一价格来源）

### 2.1 `auto-bind-coupons`（1 次，默认执行）

入参 `{}`。一键领取麦麦省当前可领券（只领券，不花钱）。返回几张就说几张；返回"无可领"或调用失败，**如实说明后继续 2.2**，不要因此停下。

### 2.2 `query-store-coupons`（1 次）

```json
{"storeCode": "1450713", "orderType": 1, "beType": 1}
```

- 返回 `data[].{title, couponId, couponCode, tradeDateTime, products[].{productCode, productName}}`；同一 `couponId` 可能对应多个 `couponCode`（多张券）
- **不要用 `query-my-coupons` 做计价**——它只返回渲染好的文本，没有 `couponId` / `couponCode`
- 券匹配规则（按顺序）：
  1. 购物篮里已有某券的 `products[].productCode` → 把该券的 `couponId + couponCode` 挂到那一行；该行 `quantity` 设为 1，多份拆成多行，**每行用一张不同的 `couponCode`**，券码用完就不再挂
  2. 篮里没有但同类可替换（例：券适用 麦辣鸡腿堡3件套 `9900004002` 用券价 ¥22.9，篮里原是 麦辣鸡腿汉堡三件套 `9900005456` ¥33.5）→ 换成券适用的 code 再挂券，并在输出里写明"为用券换成了 X"。**替换不得违反该人的忌口**（不吃辣的人不能被换成麦辣鸡腿堡3件套）
  3. 免费 / 加购型券（如"免费脆薯饼"）→ 新增一行该券 `products[].productCode` 的商品，`quantity` 1，挂券
  4. 都不行 → 不挂，在输出里列出"已领到但本单用不上"的券

### 2.3 `calculate-price`（算 1 次；最多复算 1 次）

```json
{
  "storeCode": "1450713", "orderType": 1, "beType": 1,
  "items": [
    {"productCode": "9900015151", "quantity": 1,
     "roundList": [{"round": "2", "comboItemList": [{"code": "1406", "quantity": 1}]}]},
    {"productCode": "9900004002", "quantity": 1,
     "couponId": "<来自 query-store-coupons>", "couponCode": "<来自 query-store-coupons>"},
    {"productCode": "9900016073", "quantity": 1},
    {"productCode": "521950", "quantity": 1}
  ]
}
```

- `roundList` 只传**改动过的那一轮**（`round` 是轮次 id 的**字符串**），未改动的轮不传
- 读取：`data.originalPrice`（原价）、`data.price`（实付）、`data.discount`（优惠）、`data.productList[].{productCode, productName, quantity, originalSubtotal, subtotal}`（每行原小计 / 实付小计）、`data.takeWayList[].{code, title}`（取餐方式，下单时要用）。金额字段全部为**分**；`takeWayList` 不是金额
- **只算一份篮子，不做方案 B。** 仅下列两种情况之一触发**一次**复算：① `data.price` 超过总预算 → 按 1.4 降级后复算；② 某行挂券后 `subtotal` 等于 `originalSubtotal`（券未生效）→ 去掉该券（免费券则去掉该行）后复算。复算后不再调整，最后一次返回就是最终方案
- 哪张券生效，看**逐行** `originalSubtotal` 与 `subtotal` 的差；总 `discount` 只用于展示，不用于判断单张券

---

## 阶段三 · 输出（目标：一张能直接发到群里的清单，默认停在下单前）

### 3.1 AA 分账（本地计算，只基于最终那次 `calculate-price` 的 `productList[].subtotal`）

1. 单人项（三件套、单品）→ `subtotal` 归该人；同一行多份由多人各吃一份时，按份数分摊该行 `subtotal`
2. 多人餐 → `subtotal` 在共享者之间均分
3. 共享加购（麦旋风、小食盘）→ 在用户指定的人之间均分，未指定则全员均分
4. 一律用**分**做整数运算，除不尽的余数（尾差）加到最后一人——这是唯一允许的"凑整"
5. 校验：`productList[].subtotal` 之和应等于 `data.price`；若不等（出现餐具费等额外费用），差额全员均分并在输出里注明来源；仍对不上就停止分账并说明原因，**禁止改动任何小计去凑平**
6. 券优惠已体现在该行 `subtotal` 里，默认归该行的人；用户要求"优惠全员均摊"时再按比例重算

### 3.2 输出模板

```
🍔 麦麦饭局方案（{人数} 人 · {storeName}）

✅ 实付 ¥{price/100}（原价 ¥{originalPrice/100}，省 ¥{discount/100}）
   用券：{券 title} → {挂在哪一行}；已为你一键领取 {N} 张麦麦省券

📋 明细（来自 calculate-price）
   · {productName} ×{quantity}   ¥{subtotal/100}  {原 ¥originalSubtotal/100 / 换了什么 / 谁吃}
   …
   ⚠️ 已按忌口处理：{排除 / 替换说明}

💰 AA 分账
   {姓名或"第 1 位"}   ¥xx.xx
   …（合计 = 实付）

🔗 价格以支付页为准。回复「下单」我才会创建订单；回复「换店 / 换菜 / 不要券」可调整
```

- 只写返回里有的数字；"省 ¥x"只在 `discount > 0` 时写；未生效的券写"已领到但本单未生效"
- 不展开特调选项，除非用户问

### 3.3 下单（仅在获得硬约束第 5 条所述授权后执行；比赛演示中不执行）

获得授权后：先问取餐方式（`takeWayList` 中的 `eat-in` 堂食 / `locker-out` 外带），再调用 `create-order`，其 `items` **必须与最终那次 `calculate-price` 完全一致**，并加 `"takeWayCode": "<选中的 code>"`。返回支付链接后引导用户在麦当劳 App 完成支付；用户回复"支付完成 / 失败"后用 `query-order` 查状态。这是真实交易，任何不一致都停下来确认。

---

## 错误处理（先分类，再决定回退、继续还是停下）

| 现象 | 处理 |
|---|---|
| `401` | **停**。Token 无效或过期，提示重新申请并更新连接器配置 |
| `429` | **停**。触发 600 次/分钟限流，复用已拉取的菜单与券数据，稍后重试 |
| `query-nearby-stores` 返回 `code 600050` | **回退**：收藏为空，改 `searchType=2` + 城市 + 关键词 |
| 门店 `businessStatus:false` | **回退**：换下一家营业中的门店 |
| `auto-bind-coupons` 失败 / 无可领 | **继续**：说明状态，用已有券或无券继续报价 |
| `query-store-coupons` 的 `data` 为空 | **继续**：本店 / 本时段无可用券，不挂券报价，并说明"已领到的券本单用不上" |
| 某行挂券后 `subtotal == originalSubtotal` | 该券未生效，按 2.3 复算一次；不用券的总价就是返回里的 `data.originalPrice`，不另算 |
| 商品 code 报错 / 售罄 | **回退**：回到 `query-meals` 选同类替代，不要猜 code |
| 其余 `success:false` | **停**：原文转述 `message`，不要猜结果继续 |

## 参考运行记录（2026-10-25 12:25 真实调用，仅供对照格式；**数字和 code 每天会变，严禁直接复用**）

- 门店：麦当劳上海黄浦华旭国际大厦餐厅 `1450713`
- 篮子：麦麦周末双人餐 ×1 + 麦辣鸡腿堡3件套 ×1（挂卡包券）+ 龙焰鸡腿堡三件套 ×1 + 蓝莓爆爆珠麦旋风 ×1
- `calculate-price` 返回：`originalPrice 13930 / price 11620 / discount 2310` → ¥139.30 → ¥116.20，4 人均 ¥29.05；逐行 `subtotal`：4990 / 2290（原 4600）/ 2690 / 1650
- 双人餐第 2 轮换板烧（`roundList` 只传 `round "2"`）→ `5090`（+¥1）
