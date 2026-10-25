# MCP 集成说明

> 说明本项目实际使用的麦当劳 MCP Server、Tool、调用流程与业务价值。

## 1. 使用的 MCP Server

| 项 | 值 |
| --- | --- |
| 官方仓库 | `M-China/mcd-mcp-server` |
| 接入地址 | `https://mcp.mcd.cn` |
| 传输协议 | Streamable HTTP |
| 认证方式 | 请求头 `Authorization: Bearer <MCP Token>` |
| 限流 | 600 次 / 分钟（超限返回 429） |
| Token 申请 | <https://open.mcd.cn/mcp>（手机号登录 → 控制台 → 激活） |

配置文件仅以环境变量占位符形式提供（`mcp-config.example.json`）：

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": { "Authorization": "Bearer ${MCD_MCP_TOKEN}" }
    }
  }
}
```

## 2. 实际调用的 Tool

本版只做**到店自取**（`beType=1` / `orderType=1`）。主链路固定使用 6 个 Tool，`query-meal-detail` 按需，另有 2 个仅在用户明确授权下单后使用。

| # | Tool | 在本项目中的作用 | 使用时机 |
| --- | --- | --- | --- |
| 1 | `query-nearby-stores` | 按城市 + 地标查询附近门店，取营业中且最近的一家作为点单门店 | 每次必调（1 次） |
| 2 | `query-meals` | 拉取该门店当前可售菜单（分类、餐品编码、价格、标签），构建候选池 | 每次必调（1 次） |
| 3 | `query-meal-detail` | 查套餐 / 多人餐的组成与可替换项，用于「只换掉忌口单品」而不放弃整单 | 有忌口者分配到组合餐时（≤ 2 次） |
| 4 | `auto-bind-coupons` | 一键领取麦麦省当前可领券（只领券，不花钱） | 每次必调（1 次） |
| 5 | `query-store-coupons` | 查询本店当前可用券及其适用商品 `productCode`，用于把券挂到对应行 | 每次必调（1 次） |
| 6 | `calculate-price` | **唯一价格来源**：对最终购物篮（含券、含换项）计算原价 / 实付 / 每行小计 | 算 1 次，最多复算 1 次 |
| 7 | `create-order` | 用户针对当前报价明确说「下单 / 确认下单」后创建订单并返回支付链接 | **仅在获得下单授权后**；本版未实测 |
| 8 | `query-order` | 追踪订单状态与进度 | 下单后；本版未实测 |

另有 2 个按用户主动要求才调用：`campaign-calendar`（用户问「有什么活动」）、`list-nutrition-foods`（用户要求「控热量」）。

> 说明：`now-time-info`、`query-my-coupons`、`delivery-*` 系列在本版中**不使用**——餐段由门店菜单实时决定，`query-my-coupons` 只返回渲染文本、没有 `couponId` / `couponCode` 因而无法用于计价，外送不在本版范围内。

## 3. 调用流程

```
用户输入：人数 / 城市 + 地标 / 人均预算 / 忌口
        │
        ├─▶ query-nearby-stores           选定门店（营业中、最近）→ storeCode
        │
        ├─▶ query-meals                   拉菜单，按忌口关键词排除后配餐
        │      └─▶ query-meal-detail       忌口者分到组合餐时，查明子项并替换
        │
        ├─▶ auto-bind-coupons             一键领取麦麦省可领券
        │      └─▶ query-store-coupons    取本店可用券及适用 productCode
        │
        ├─▶ calculate-price               ★唯一价格来源：原价 / 实付 / 每行小计
        │
        ├─▶ （本地）AA 分账：按「谁吃什么」分摊，尾差加到最后一人
        │
        └─▶ 输出可直接发到群里的清单，默认【停在下单前】
               └─▶ （仅当用户明确说「下单」）create-order ──▶ query-order
```

## 4. 业务价值

1. **价格只信一个来源**：套餐、单点、优惠券三套规则混在一起时人脑算不清，本项目把 `calculate-price` 定为唯一价格来源，禁止心算与估算，输出的原价 / 实付 / 每行小计都能在返回里逐一核对。
2. **让忌口从「整单妥协」变成「精准替换」**：借 `query-meal-detail` 的可替换项能力，只换掉冲突的那一轮子项，保住套餐优惠，而不是为了一个人放弃整单。
3. **聚餐场景的最后一公里**：把「点单 + 领券 + 报价 + AA 分账」串成一条链路，输出一张能直接发到群里的清单；需要时再一步下单。
4. **真实落地而非演示**：主链路全部为真实调用（见第 5 节两次端到端跑通），任一环节异常都会如实告知用户（门店未营业、券不可用、429 限流、401 Token 失效），不伪造结果。

## 5. 实测验证记录

以下均为本项目开发中对麦当劳 MCP 的**真实调用结果**（Token 已脱敏且不入库）：

| 验证项 | 结果 |
| --- | --- |
| 端点连通 | `POST https://mcp.mcd.cn` 返回 `200 OK`；`initialize` 响应 `serverInfo: {name: "mcd-mcp", version: "1.0.0"}`，`protocolVersion: 2024-11-05` |
| `tools/list` | 成功返回完整工具清单及各工具 input / output schema |
| `now-time-info` | 真实调用成功，返回服务器时间 `2026-10-25 11:32:04` |
| `query-my-account` | 真实调用成功，返回积分账户信息（可用积分、累计积分等字段，具体数值已脱敏） |

**端点澄清**：官方两个仓库标注的 `https://mcp.mcd.cn` 即为正确的 Streamable HTTP 端点，本次实测连通；`open.mcd.cn/mcp` 是 Token 申请控制台页面，并非 MCP 端点。

### 端到端跑通（2026-10-25 12:57–13:01，两次均通过）

固定用例：*4 个人到店聚餐，上海市 人民广场 附近，1 个人不吃辣，人均 30 以内。*

| 环节 | 结果 |
| --- | --- |
| `query-nearby-stores` | 4 家门店，取 `1450713` 麦当劳上海黄浦华旭国际大厦餐厅（144 m，营业中） |
| `query-meals` | 15 分类 / 124 餐品 |
| `query-meal-detail` | 双人餐第 2 轮默认麦辣鸡腿汉堡 `1440` → 换板烧鸡腿堡 `1406`（+¥1） |
| `auto-bind-coupons` | 成功领取 3 张券：11.9元麦辣鸡腿堡、19.9元王牌炸鸡任选2（有效期 10-26 起、仅周一至周五，故本单未用）、免费脆薯饼（未出现在本店可用列表） |
| `query-store-coupons` | 本店可用 1 张：麦辣鸡腿堡三件套，适用 `9900004002` |
| `calculate-price` | `originalPrice 14030` → `price 11720`（¥140.30 → **¥117.20**，省 ¥23.10） |
| 券是否生效 | 第 4 行 `4600 → 2290`，券生效 ✅ |
| AA 分账 | 4 人 ¥29.57 / ¥29.57 / ¥27.02 / ¥31.04，合计 ¥117.20 = 实付 ✅ |
| 是否下单 | **否**，未调用 `create-order`，停在确认前 |

同一用例原样复跑第二次，返回数字与第一轮完全一致；13:21 另由只读客户端用相同入参独立复算，仍为 `14030 → 11720`（原始返回 `docs/实测数据/calculate-price_4人篮子_换板烧_11720.json`）。完整入参与逐行返回见 **[`docs/真实跑通记录.md`](docs/真实跑通记录.md)**，原始 JSON 见 `docs/实测数据/`。

### 实测中的关键发现

1. **券适用的商品可能不在门店菜单里**：`9900004002`（麦辣鸡腿堡3件套）不在门店 1450713 的菜单返回中，但 `calculate-price` 接受它并给出用券价。因此商品 code 的合法来源必须包含 `query-store-coupons` 返回的 `products[].productCode`，不能只看菜单。
2. **金额单位是分**：`calculate-price` 的 `price` / `originalPrice` / `discount` / `subtotal` 全部为分，`query-meals` 的 `currentPrice` 是元字符串，展示时需统一换算。
3. **`roundList` 只传改动过的那一轮**：双人餐 4 轮中只改第 2 轮，就只传 `round: "2"`，其余轮保持默认。
4. **`query-my-coupons` 不能用于计价**：它只返回渲染好的文本，没有 `couponId` / `couponCode`。
5. **工具返回的前半段是字段说明**，真实数据在 `## Original Response` 之后的 JSON 里。

## 6. 安全与合规

- 仓库内不含任何真实 Token / 密钥 / 账号凭证，配置仅使用 `${MCD_MCP_TOKEN}` 环境变量占位符。
- 真实配置存放于本地 `mcp-config.json`，已由 `.gitignore` 排除。
- 项目输出仅供参考，餐品价格与供应状态以麦当劳官方实时结果为准。
