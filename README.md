# investment_intel

自动化投资情报简报仓库。每次调用 Claude Code Skill `/investment-intel`，即生成一份覆盖**过去 8 小时**的国际局势简报，保存到 `report/` 目录，文件名以 UTC 时间戳命名。

## 覆盖范围

- **地缘 / 政治**：中东、俄乌、东亚（台海 / 朝鲜半岛 / 中日韩）、欧美政治、其他突发
- **能源**：WTI / Brent 油价、驱动因子、12-72h 展望
- **金融市场**：主要股指、外汇、避险资产、板块轮动
- **后续演化**：3 档情景（A/B/C）+ 概率 + 关注清单

## 数据来源

- CNN · BBC · Bloomberg · Al Jazeera · NBC / CBS
- X：[@realDonaldTrump](https://x.com/realDonaldTrump) · [@AggregateOsint](https://x.com/AggregateOsint)

## 如何使用

在本仓库内启动 Claude Code 会话后，直接调用：

```
/investment-intel
```

Skill 会自动：

1. 抓取过去 8 小时的公开信息（每源一次独立调用）
2. 按 `report/TEMPLATE.md` 的十章结构填充
3. 保存为 `report/YYYY-MM-DD_HHMM_UTC.md`
4. Commit 并 push 到开发分支

## 目录结构

```
.
├── .claude/skills/investment-intel/SKILL.md   # Skill 定义与 workflow
├── report/
│   ├── TEMPLATE.md                             # 固定十章骨架
│   └── YYYY-MM-DD_HHMM_UTC.md                  # 历次报告（UTC 时间戳）
└── README.md
```

## 报告结构（十章）

1. 执行摘要
2. 过去 8 小时关键事件时间线
3. 国际媒体综合（CNN / BBC / Bloomberg / Al Jazeera / NBC）
4. Twitter/X 小道消息
5. 能源市场分析
6. 金融市场影响
7. 区域热点聚焦（多议题）
8. 后续演化可能性（情景 A/B/C）
9. 关注清单（未来 24-48 小时）
10. 风险提示

## 免责声明

本仓库所有报告由自动化系统整合公开信息生成，仅供研究参考，**不构成任何投资建议**。
