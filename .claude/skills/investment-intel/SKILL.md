---
name: investment-intel
description: 收集过去 24 小时 CNN / BBC / Bloomberg / Al Jazeera / NBC、以及 X 上 @realDonaldTrump 与 @AggregateOsint 的公开信息，按固定十章模板生成「国际局势（多区域热点）/ 油价与能源 / 金融市场影响 / 后续演化」投资情报简报，保存到 report/<UTC>.md 并 commit+push。使用本 skill 的场景：用户请求做一次投资情报搜集 / 国际局势简报 / 地缘与油价每日更新 / 市场每日盘面摘要。
---

# Investment Intelligence Skill

目标：在一个 Claude Code turn 序列中，可靠地生成一份结构统一的国际局势投资情报简报，保存为 `report/YYYY-MM-DD_HHMM_UTC.md`，并 push 到远端。

## 硬约束（全部必须遵守）

这些规则是为**规避 `API Error: Stream idle timeout - partial response received`** 而设计，任何一条都不得省略：

1. **骨架 + 分章 Edit**：先 `cp report/TEMPLATE.md` 到新文件，再通过多次 `Edit` 逐节填充占位符（`<!-- SECTION_X_Y -->`）。禁止一次性 `Write` 整篇。
2. **每次 Edit 之前，先用一行文本汇报当前步骤**（例如 `Writing section: 3.3 Bloomberg`）。这保证流式输出不空转超过 20 秒。
3. **每个来源一次独立的 WebSearch / WebFetch 调用**。不要把多个源合并到同一次调用。抓完一个源就输出一行进度（例如 `Fetched CNN: 7 items`）。
4. **抓取超时/失败处理**：单个 `WebFetch` 失败一次，立刻改用 `WebSearch` 作为兜底；`WebSearch` 再失败就把该来源标注为 `source unavailable` 并继续。**禁止**对同一 URL 重试。
5. **小节字数上限 ≤ 400 字**（中文字符计），避免单次 Edit 过长触发超时。
6. **时间戳统一用 UTC**：文件名、header、时间线表格都用 UTC。
7. **区域覆盖多元化**：不再局限伊朗，执行摘要与时间线至少覆盖 3 个不同区域（中东 / 俄乌 / 东亚 / 欧美政治 / 突发事件）中存在实质新闻的。若某区域过去 24 小时无新闻，在执行摘要中显式说明 "无重大更新"。
8. **严格按步骤顺序执行**，不得跳步。每步完成后在主线输出一行状态。

## 工作流（严格按此顺序）

### Step 0 — 准备

```bash
TS=$(date -u +"%Y-%m-%d_%H%M_UTC")
SINCE=$(date -u -d '24 hours ago' +"%Y-%m-%d %H:%M UTC")
REPORT=report/${TS}.md
```

- `ls report/` 查看已有报告；若存在历史报告，`Read` 其中最新的一份作为语气 / 长度 / 表格密度参考。
- 若 `report/` 为空或只有 `TEMPLATE.md`，直接用 `TEMPLATE.md` 作为参考。
- 主线输出：`Prep done. TS=<TS>, SINCE=<SINCE>.`

### Step 1 — 建骨架

```bash
cp report/TEMPLATE.md "$REPORT"
```

用一次 `Edit` 把 header 里的 `<!-- REPORT_TIME -->` 替换成 `YYYY-MM-DD HH:MM UTC`，`<!-- SINCE -->` 替换成 `$SINCE`。

主线输出：`Skeleton ready at $REPORT`.

### Step 2 — 逐源抓取（独立调用 + 进度日志）

按下列顺序，每项独立一次调用，抓完一项输出一行 `Fetched <source>: <n items>`：

1. `WebSearch`：`CNN` + 过去 24 小时 + 关键词（Middle East, Russia Ukraine, China Taiwan, oil, markets, breaking）
2. `WebSearch`：`BBC` + 过去 24 小时 + 关键词（same set）
3. `WebFetch https://www.bloomberg.com/` — 取首页头条；失败 fallback → `WebSearch Bloomberg breaking markets oil`
4. `WebSearch`：`Al Jazeera` 中东 / 红海 / 北非突发
5. `WebSearch`：`NBC OR CBS` 市场 / 地缘
6. `WebFetch https://x.com/realDonaldTrump` — 失败 fallback → `WebSearch "Trump Truth Social" last 24 hours`
7. `WebFetch https://x.com/AggregateOsint` — 失败 fallback → `WebSearch AggregateOsint last 24 hours`
8. （可选，仅当前面 7 项总条数 < 10）`WebSearch "breaking" <region>` 补充长尾

所有结果保留在 assistant 的上下文里（不写中间文件），只取**过去 24 小时**的条目；带上 `published at` 或等效时间戳的优先。

### Step 3 — 分章填充（每节一次 Edit，每次 Edit 前输出一行进度）

顺序执行，每条就是一次 `Edit` 调用，占位符对应 `TEMPLATE.md` 中的注释标记：

| 序号 | 占位符 | 要求 |
|---|---|---|
| 3.1 | `SECTION_1_EXEC_SUMMARY` | **强制结构**，见下方「执行摘要硬性格式」 |
| 3.2 | `SECTION_2_TIMELINE` | Markdown 表格；时间 UTC 升序；4-10 行 |
| 3.3 | `SECTION_3_1_CNN` | 3-5 要点 |
| 3.4 | `SECTION_3_2_BBC` | 3-5 要点 |
| 3.5 | `SECTION_3_3_BLOOMBERG` | 3-5 要点（重点财经数据） |
| 3.6 | `SECTION_3_4_ALJAZEERA` | 3-5 要点（中东视角） |
| 3.7 | `SECTION_3_5_NBC` | 3-5 要点（市场 / 民意） |
| 3.8 | `SECTION_4_1_TRUMP` | 2-5 条重要推文 / Truth Social，含时间戳 |
| 3.9 | `SECTION_4_2_OSINT` | 2-5 条关键 OSINT，含时间戳 |
| 3.10 | `SECTION_5_1_OIL_PRICE` | WTI/Brent 价位 + 24h 变动 + 成交量异动 |
| 3.11 | `SECTION_5_2_DRIVERS` | 驱动油价的 3-4 个当前因子 |
| 3.12 | `SECTION_5_3_CONTRADICTIONS` | 价格与基本面的反向信号 |
| 3.13 | `SECTION_5_4_OUTLOOK` | 12-72 小时价格区间 / 概率 |
| 3.14 | `SECTION_6_1_EQUITY` | 主要指数收 / 盘中表现 |
| 3.15 | `SECTION_6_2_FX_SAFEHAVEN` | DXY / JPY / CHF / 黄金 / UST10Y |
| 3.16 | `SECTION_6_3_SECTORS` | 板块轮动 |
| 3.17 | `SECTION_7_HOTSPOTS` | Markdown 表格；每行一个区域议题；仅保留过去 24h 有进展的 |
| 3.18 | `SECTION_8_A` | 情景 A（基准）+ 概率 + 触发条件 |
| 3.19 | `SECTION_8_B` | 情景 B（上行 / 缓和） |
| 3.20 | `SECTION_8_C` | 情景 C（下行 / 升级） |
| 3.21 | `SECTION_9_WATCHLIST` | 未来 24-48h 可观察事件 / 数据发布 / 讲话，1-6 条 |
| 3.22 | `SECTION_10_RISKS` | 尾部风险 / 数据盲区 / 信息可信度 |
| 3.23 | `SOURCES` | 引用列表，Markdown 链接 + 发布时间 |

**写作准则**：
- 引用具体数字（价位、百分比、时间）而非空泛描述。
- 每个小节 ≤ 400 字；超出就拆条。
- 若某源不可用，写 `> 数据来源暂不可用，以其它来源交叉验证为准`，不强编内容。

#### 执行摘要硬性格式（SECTION_1_EXEC_SUMMARY）

**必须**按以下两段式结构写入，且「三条最关键结论」要放在报告最开头，与 Step 5 对用户口头汇报时引用的三条**逐字一致**（包括具体数字、百分比、情景概率）：

```
**三条最关键结论**：

1. **<地缘/能源主事件>** → **<WTI 数值+%>**、**<Brent 数值+%>**，<一句话市场定价结论>。
2. **<股市/风险偏好主结论>**：**<指数 %>**，<VIX / 避险资产补充>。
3. **<尾部/可逆性风险>**：<要点> —— 基准 / 上行 / 下行情景概率定为 **<X/Y/Z>**。

**其它重要背景**：

- **<关键词 1>**：…
- **<关键词 2>**：…
- **多区域覆盖确认**：中东 / 俄乌 / 东亚 / 欧美政治 —— 逐一点名是否有实质更新。
```

规则：
- 三条结论必须包含**具体数字**，禁止只用「大幅下跌」「显著回升」等形容词。
- 三条结论与 Step 5 对用户回复的三条摘要**一字不差**——以报告为准，回复直接引用，不得出现「报告里没写、只在聊天里说」的不一致。
- 「其它重要背景」3-5 条，补充区域覆盖与不在前三条但仍重要的事件。

### Step 4 — 提交并推送

在当前工作分支提交，然后推送到 `main`（用户授权默认目标）：

```bash
git add "$REPORT"
git commit -m "report: intel snapshot $TS"
# 推到 main（普通 fast-forward push，禁止使用 --force）
git push origin HEAD:main
```

push 失败（网络类错误）时指数回退重试最多 4 次（2s/4s/8s/16s）。非网络错误（例如非 fast-forward、保护分支拒绝）立即失败上报用户，**禁止**使用 `--force` 或跳过钩子。

主线输出：`Pushed $REPORT to main`.

### Step 5 — 对用户汇报

用 4-6 行简洁中文回复用户：
- 报告路径
- 3 条最重要结论（从执行摘要挑），并给出未来一周对wti原油合约交易的建议
- 若有来源不可用，点名列出
- 提示用户可在 GitHub branch 上查看

## 兜底与修复

- 若中途任一 `Edit` 因为 `old_string` 不匹配失败：`Read` 当前 `$REPORT` 文件确认占位符实际形态，再重试**同一节**，不要跳过。
- 若过去 24 小时信息极少（例如周末），在对应小节写 `过去 24 小时无实质性更新`，不要凑字。
- 若检测到抓取内容大量重复（镜像站点），在执行摘要加一行 `信息熵低，市场处于消化期`。

## 不做的事

- 不接付费 API、不使用非公开数据
- 不生成实时精确行情（仅引用抓取到的数字）
- 不在一次 turn 内生成整篇报告
- 不对同一 URL 重试抓取
