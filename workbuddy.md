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

## 5. 交付文件

- README.md
- CONTEST_DECLARATION.md（官方原文，未改动）
- MCP_INTEGRATION.md
- mcp-config.example.json（仅环境变量占位符）
- workbuddy.md（本文件）
