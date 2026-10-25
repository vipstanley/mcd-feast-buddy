# workbuddy.md — WorkBuddy 开发对话上下文

> 本文件用于核验 WorkBuddy 专项奖励资格（进榜 + 真实使用 WorkBuddy + 提交本文件 → 3000 积分）。
> 记录「麦麦饭局 McFeast」在腾讯 WorkBuddy 中的完整开发过程。

---

## 0. 项目概览

- 项目名称：麦麦饭局 McFeast
- 仓库：mcd-feast-buddy（公开）
- 形态：WorkBuddy Skill（核心）+ 网页 Demo（传播）
- 开发环境：macOS / WorkBuddy（连接器 mcd-mcp，Streamable HTTP）
- 开发日期：2026-10-25

## 1. 需求分析与选型

**用户诉求**：参加麦当劳程序员创意开发大赛，用麦当劳 MCP 做一个 AI 作品，并希望在「麦当劳大赛排名」与「WorkBuddy 专项奖励」两边都有机会拿奖。

**选型推理**（基于 2026-10-25 10:00 官方 `RANKING.md` 快照）：

- 省钱 / 优惠券类高度内卷，头部 `mcd-saver-strategist` 1366 Star，同赛道十余个项目
- 派对策划类天花板已封，`mcd-party-planner` 1423 Star
- 聚餐 / 团餐类头部仅 103 Star（`mcd-group-order`），属空白赛道
- 聚餐结果天然要发给朋友确认，**自带传播属性**，利于冲 Star
- 该场景可深度串起 12 个 MCP 工具，最能体现「真实调用麦当劳 MCP」这一评审口径

**结论**：定为「麦麦饭局 McFeast」聚餐一键点方向；形态为 WorkBuddy Skill（核心交付）+ 单页 Demo（传播）。

## 1.1 比赛规则核实

已核实官方 `activityGuidelines.md`：报名截止 2026-10-25 23:59，定榜 10-26 00:00，按公开 Star 数排名、前 100 进榜（门槛当时约 63 Star）。必交文件为 README.md、CONTEST_DECLARATION.md（官方原文不可改）、MCP_INTEGRATION.md、mcp-config.example.json、源代码、workbuddy.md。

## 2. MCP 连通配置

- MCP Server：`M-China/mcd-mcp-server`，端点 `https://mcp.mcd.cn`，Streamable HTTP，请求头 `Authorization: Bearer <Token>`
- 仓库已提供脱敏配置 `mcp-config.example.json`，仅保留 `${MCD_MCP_TOKEN}` 占位符
- 真实配置存放于本地 `mcp-config.json`，已被 `.gitignore` 排除，绝不入库

**当前状态：等待用户领取 Token。**

领取需本人手机号接收短信验证码，且 Token 绑定本人麦当劳会员账号，AI 无法代劳。用户操作路径：打开 <https://open.mcd.cn/mcp> → 手机号登录 → 右上角「控制台」→「激活」→ 同意协议 → 复制 Token，约 30 秒。

连通性自测方案：配置完成后调用 `now-time-info`，能返回当前时间即视为 MCP 已通。若返回 401 则检查 Token 与端点地址。

## 3. 核心逻辑开发

`skill/SKILL.md` 定义了七步工作流：

| 步骤 | 内容 | 涉及 MCP Tool |
|---|---|---|
| 0 | 收集需求（人数 / 方式 / 地址 / 预算 / 忌口，最多追问一轮后用默认值推进） | — |
| 1 | 定餐段与门店（到店取餐 / 外送） | `now-time-info`、`query-nearby-stores`、`delivery-query-stores`、`delivery-query-addresses` |
| 2 | 拉菜单与套餐可替换项 | `query-meals`、`query-meal-detail` |
| 3 | 忌口与热量过滤，原则「**替换而非放弃**」以保住套餐优惠 | `list-nutrition-foods`、`query-meal-detail` |
| 4 | 汇总全部可用券 | `auto-bind-coupons`、`available-coupons`、`query-my-coupons`、`query-store-coupons` |
| 5 | 对「全单点 / 全套餐 / 混搭」多方案统一比价 | `calculate-price` |
| 6 | AA 分账本地计算 | — |
| 7 | 创建订单并追踪（**须用户显式确认**） | `create-order`、`query-order` |

**AA 分账算法**：可明确归属的单品归归属人；共享项按人数均分；优惠券减免按各自商品金额**比例**分摊而非简单均分；配送费均分；保留 2 位小数，分账尾差调整到最后一人，保证各人应付之和恰好等于实付总额。

**安全红线**：未经用户显式确认绝不调用 `create-order`（真实交易）；价格一律以 `calculate-price` 返回为准，禁止自行心算；券不可用、售罄或触发 429 时如实告知，不伪造结果；MCP Token 不在对话中回显。

## 4. 联调与验证

- Token 已由用户本人领取（需手机号短信验证，AI 无法代领）。配置写入本地 `~/.workbuddy/mcp.json`，真实配置文件不入仓库、不回显
- **端点连通验证**：`initialize` 返回 `mcd-mcp v1.0.0`，`protocolVersion 2024-11-05`
- **真实工具调用**：`now-time-info` 返回服务器时间 `2026-10-25 11:32:04`；`query-my-account` 成功返回积分账户信息（数值已脱敏，不外泄 accountId）
- **端点争议澄清**：`mcp.mcd.cn` 实测有效；`open.mcd.cn/mcp` 为领 Token 的控制台页，非 MCP 端点
- `web/index.html` 单页 Demo 已产出，示例数据已在页面上明确标注为「示例」，避免误导

## 5. 方向收缩：只做到店自取，砍掉外送与多方案穷举

拿到真实数据后，第一轮「七步工作流 + 12 个工具」的方案被证明撑不住：

- 外送链路（`delivery-query-stores` / `delivery-query-addresses`）涉及收货地址，参数多、失败率高，且与「到店自取」是两套逻辑
- 「全单点 / 全套餐 / 混搭」多方案穷举会把工具调用次数推到限流边缘，实测中也没有带来更好的价格——因为最优解几乎总是「一份多人餐 + 若干带券套餐」
- `list-nutrition-foods` 做忌口过滤既慢又不可靠（营养表不给「是否含辣」），改用**菜名关键词排除 + `query-meal-detail` 换子项**，更准也更快

于是方向收缩为 **A 瘦身版**：到店自取 · 真菜单 → 真券 → `calculate-price` 真报价 → AA 分账 → **停在下单前**。工具从 12 个减到主链路 6 个。

## 6. 实测打通与 SKILL 重写

用真实 Token 做了 14 次只读调用，打通门店 / 菜单 / 套餐详情 / 门店券 / 带券计价 / 换子项计价六类能力，据此重写 `skill/SKILL.md`（208 行，三段式）。实测中踩到的坑全部写进了 SKILL 的硬约束：

1. **金额单位是分**：`calculate-price` 的 `price` / `originalPrice` / `subtotal` 全为分，`query-meals` 的 `currentPrice` 是元字符串——两者混用会差 100 倍
2. **券适用商品可能不在菜单里**：`9900004002`（麦辣鸡腿堡3件套）不在门店 1450713 的菜单返回中，但 `calculate-price` 接受它并给出用券价。所以商品 code 的合法来源必须包含 `query-store-coupons` 返回的 `products[].productCode`
3. **`roundList` 只传改动过的那一轮**：双人餐有 4 轮，只改第 2 轮就只传 `round: "2"`，多传会报错
4. **`query-my-coupons` 不能用于计价**：只返回渲染好的文本，没有 `couponId` / `couponCode`
5. **工具返回前半段是字段说明**，真实数据在 `## Original Response` 之后的 JSON 里

## 7. 端到端跑通（真实调用，两次）

固定用例：*4 个人到店聚餐，上海市 人民广场 附近，1 个人不吃辣，人均 30 以内。*

2026-10-25 12:57 第一次跑通，13:01 原样复跑第二次，数字完全一致：

| 环节 | 结果 |
| --- | --- |
| `query-nearby-stores` | 4 家门店，取 `1450713` 麦当劳上海黄浦华旭国际大厦餐厅（144 m，营业中） |
| `query-meals` | 15 分类 / 124 餐品 |
| `query-meal-detail` | 双人餐第 2 轮默认麦辣鸡腿汉堡 `1440` → 换板烧鸡腿堡 `1406`（+¥1） |
| `auto-bind-coupons` | 成功领取 3 张券（11.9元麦辣鸡腿堡 / 19.9元王牌炸鸡任选2 / 免费脆薯饼） |
| `query-store-coupons` | 本店可用 1 张，适用 `9900004002` |
| `calculate-price` | `originalPrice 14030` → `price 11720`（¥140.30 → ¥117.20，省 ¥23.10） |
| 券生效校验 | 该行 `4600 → 2290` ✅ |
| AA 分账 | ¥29.57 / ¥29.57 / ¥27.02 / ¥31.04，合计 ¥117.20 = 实付 ✅ |
| 是否下单 | **否**，未调用 `create-order` |

完整入参与逐行返回见 `docs/真实跑通记录.md`，原始 JSON 见 `docs/实测数据/`（券码与 couponId 已遮盖）。

## 8. 交付文件

- README.md（示例输出已换为真实运行数据）
- CONTEST_DECLARATION.md（官方原文，未改动）
- MCP_INTEGRATION.md（工具清单 6+2、调用流程、5 个实测坑）
- mcp-config.example.json（仅环境变量占位符）
- 使用说明.md（三段式工作流 + 真实输出）
- skill/SKILL.md（三段式，208 行）
- docs/真实跑通记录.md、docs/实测数据/
- web/index.html（真实输出 + 「演示停在下单前」）
- workbuddy.md（本文件）
