# 本仓库相对上游的改动

上游：<https://github.com/aa1355721445-lgtm/dsh-daily-schedule>
本仓库：0.4.0 → **0.4.2**，两处改动，都是"让日程真的知道两个人之间发生过什么"以及"别因为一次 JSON 截断就整天没有日程"。

## 0.4.1 · 接入真实上下文（聊天流水 / 日记 / 长期记忆）

**问题**：`lib/generate.js` 的 `buildPrompt()` 只喂了人设、当天类型、最近三天日程摘要、衣柜和一句 hint——完全没有聊天记录，所以生成的日程和两人真聊过的事脱节（比如前一天约好"明天面试"，第二天日程里没有这件事）。

**改动**：

- 新增 `export function recentLifeContext(date, cfg = getConfig())`，按此顺序收集并裁剪：

  | 来源 | 范围 | 上限 |
  | --- | --- | --- |
  | `<DSH_HOME>/qqbot/days/<date>.md`（每天取末尾） | 最近 3 天 | 每天 30 行 / 1600 字 |
  | `<DSH_HOME>/qqbot/diary/*.md` | 最近 2 篇 | 900 字 |
  | `<DSH_HOME>/soul-md/memory/core.md` | 长期记忆 | 1200 字 |

  拼成三段带标题的文本：`【…和<partnerName>的聊天片段】`、`【…她自己写的日记】`、`【长期记忆】`。

- `buildPrompt()` 增加 `【最近真实发生过的事（聊天记录 / 她的日记 / 长期记忆）】` 段，并加一条规则：

  > 说好的约定 / 正在追的东西 / 挂心的事要排进对应时段，别硬编。

- `cfg.useChatContext === false` 可整体关闭（老行为）。

**验证**：`recentLifeContext('2026-11-27')` 返回 2697 字（含 11/25、11/26 聊天片段）；生成 2026-11-28 的日程里出现了 `回小林，问他二轮面试有没有准信`、`把魔王那本翻了两页`、`陌生链接一律没点` —— 都来自真实聊天。

## 0.4.2 · JSON 解析加固

**问题**：新提示词更长、更容易让模型把 JSON 写断，`parseSchedule` 直接 `JSON.parse` 抛错，整天日程就没了：

```
日程 JSON 解析失败：Expected ',' or ']' after array element in JSON at position 3141
```

**改动**：

- `function salvageItems(s)`：从 `"items"` 之后第一个 `[` 开始逐字符扫描（跟踪字符串 / 转义 / 嵌套深度），记录每个完整 item 对象的结束位置，把已完成的部分补齐 `]` 后单独 `JSON.parse` —— 丢弃最后那个被截断的段落，保住前面十几段。
- `function fieldOf(s, name)`：正则抠出 `"title"` / `"mood"` 的字符串值，配合 salvage 结果组装 `{title, mood, items}`。
- `parseSchedule` 的 catch 先尝试 salvage，段数 ≥ 4 就用它，否则抛原错。
- 提示词加一条：JSON 字符串里不要出现英文双引号 `"` 和换行符，要引号就用 `「」`，`detail` 控制在 40 字以内。
- `generateDay` 最多尝试 2 次，第二次在 prompt 末尾追加 `【重试】上一次输出不是合法 JSON（可能被截断了）…最后一段 end 必须是 24:00。`
- 每次解析失败都把模型原文写到 `~/.dsh/daily-schedule/last-raw.txt`（≤20000 字符），方便排查。

**验证**：同一套提示词重新生成 2026-11-28 成功（29.6 s，19 段）。

## 安装

```powershell
dsh plugin --profile desktop add "dsh-daily-schedule@file:D:/DSH/plugins/dsh-daily-schedule"
```

**注意**：DSH 按进程缓存插件的模块路径。如果这个插件之前是用 `link:` 装到临时目录的，换成本仓库的 `file:` 之后必须重启 DSH 宿主，否则会一直去旧路径找 `lib/index.js` 并报 `ENOENT`。
