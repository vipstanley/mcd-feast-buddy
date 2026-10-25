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

| # | Tool | 在本项目中的作用 |
| --- | --- | --- |
| 1 | `now-time-info` | 判断当前餐段（早餐 / 正餐 / 下午茶），决定可售菜单与活动；同时作为连通性自检 |
| 2 | `query-nearby-stores` | 根据用户位置查询附近可用门店，确定点单门店 |
| 3 | `delivery-query-stores` | 外送场景下查询该收货地址可配送的门店 |
| 4 | `query-meals` | 拉取当前门店可售菜单（分类、餐品编码、标签） |
| 5 | `query-meal-detail` | 查套餐组成与可替换项——用于「换掉忌口单品」而不必整单放弃 |
| 6 | `list-nutrition-foods` | 按营养成分过滤忌口 / 热量约束（如不吃牛肉、控热量） |
| 7 | `query-store-coupons` | 查询当前门店可用优惠券 |
| 8 | `available-coupons` + `auto-bind-coupons` | 拉取并一键领取麦麦省当前可领券，最大化可用优惠 |
| 9 | `query-my-coupons` | 汇总账户下已有券，纳入比价候选池 |
| 10 | `calculate-price` | 对多套候选组合（含券）计算应付总价，选出最优 |
| 11 | `create-order` | 用户确认后创建订单，返回支付链接 |
| 12 | `query-order` | 追踪订单状态与进度 |

## 3. 调用流程

```
用户输入：人数 / 人均预算 / 忌口 / 口味 / 地址
        │
        ├─▶ now-time-info                 确认餐段与时间
        ├─▶ query-nearby-stores           选定门店
        │   （外送则 delivery-query-stores）
        ├─▶ query-meals ──▶ query-meal-detail
        │                   取菜单 + 套餐组成 + 可替换项
        ├─▶ list-nutrition-foods          按忌口/热量过滤，标记可替换项
        ├─▶ auto-bind-coupons ─▶ available-coupons ─▶ query-my-coupons ─▶ query-store-coupons
        │                   汇总全部可用券，纳入候选池
        ├─▶ calculate-price   「单点 / 套餐 / 券组合」多方案统一比价 → 取最优
        │
        ├─▶ （本地）AA 分账：按「谁点了什么」分摊，输出每人应付
        │
        └─▶ create-order ──▶ 返回支付链接 ──▶ query-order 追踪进度
```

## 4. 业务价值

1. **把「比价」这件事交给 AI，而不是让人心算**：套餐、单点、优惠券三套规则混在一起时，人脑几乎算不出最优解；本项目用同一候选池统一枚举，明确说出「省了多少、打了几折、用了哪张券、为什么要拆单」。
2. **让忌口从「整单妥协」变成「精准替换」**：借 `query-meal-detail` 的可替换项能力，只换掉冲突单品，保住套餐优惠，而不是为了一个人放弃整单。
3. **聚餐场景的最后一公里**：把「点单 + 分账 + 支付链接 + 进度追踪」串成一条链路，群聊里发一次就完事。
4. **真实落地而非演示**：全流程 12 个 Tool 均为真实调用，任一环节返回异常都会明确告知用户（如门店售罄、券不可用、429 限流），不伪造结果。

## 5. 实测验证记录

以下均为本项目开发中对麦当劳 MCP 的**真实调用结果**（Token 已脱敏且不入库）：

| 验证项 | 结果 |
| --- | --- |
| 端点连通 | `POST https://mcp.mcd.cn` 返回 `200 OK`；`initialize` 响应 `serverInfo: {name: "mcd-mcp", version: "1.0.0"}`，`protocolVersion: 2024-11-05` |
| `tools/list` | 成功返回完整工具清单及各工具 input / output schema |
| `now-time-info` | 真实调用成功，返回服务器时间 `2026-10-25 11:32:04` |
| `query-my-account` | 真实调用成功，返回积分账户信息（可用积分、累计积分等字段，具体数值已脱敏） |

**端点澄清**：官方两个仓库标注的 `https://mcp.mcd.cn` 即为正确的 Streamable HTTP 端点，本次实测连通；`open.mcd.cn/mcp` 是 Token 申请控制台页面，并非 MCP 端点。

## 6. 安全与合规

- 仓库内不含任何真实 Token / 密钥 / 账号凭证，配置仅使用 `${MCD_MCP_TOKEN}` 环境变量占位符。
- 真实配置存放于本地 `mcp-config.json`，已由 `.gitignore` 排除。
- 项目输出仅供参考，餐品价格与供应状态以麦当劳官方实时结果为准。
