# ohlc — K 线与经济事件数据仓库

K 线与经济事件数据的实体目录。app（Kyline）端通过存储根目录 `~/.config/kyline/`
下的软链访问：

- `~/.config/kyline/tickers` → `tickers/`（Kyline 的 `dataFolder`，品种 K 线）
- `~/.config/kyline/events` → `events/`（Kyline 的 `calendarFolder`，经济事件）

## 目录结构

```
tickers/                      品种 K 线（dataFolder 里只放品种，一个子目录一个品种）
  SPY/ QQQ/ VIX/ VXN/           1m 四品种（见下节）；日线 SPY/QQQ/VIX 2020 起、VXN 2026 起
  <成分股>/ × ~518              SPY/QQQ 成分股，仅日线全历史（见「成分股全历史」节）
  XBTUSD/ ETHUSD/               BitMEX 加密 K 线（1m 逐日 + 1d 月度，bucketed API，2026-10 入库）
    <SYM>_1m_<YYYY-MM-DD>.csv     1m K 线，按天一个文件
    <SYM>_D_<YYYY-MM>.csv         日线，按月一个文件
events/                       经济事件日历（Forex Factory），按月一个 CSV
yahoo-daily-sync-state.json   成分股日线同步状态（断点续传/增量判断）
readme.md                     本文件
```

表头统一为 `date,open,high,low,close,volume,average,barCount`；`date` 列是
美东本地时间字符串（Kyline 按 UTC 直接解析，不做时区换算）。

app 的自有产物不放在本仓库：实时当日文件在 `~/.config/kyline/live/`，前复权
因子缓存在 `~/.config/kyline/adjust/`（可再生数据，随时可从 Yahoo 全量重算），
已同步周区间状态在 `~/.config/kyline/calendar-sync-state.json`。

## K 线来源：IBKR 历史 + Yahoo 增量

大跨度历史由 IBKR Workstation 一次性导出，之后每天新增的增量由 Kyline 的
Yahoo 定时同步（默认 30 分钟一轮）追加。两者文件命名 / 字段 / 时间约定完全
一致，无缝衔接。各段分界（实测，2026-09 核对）：

| 数据 | IBKR 段 | Yahoo 段 |
|---|---|---|
| 1m（四品种） | 2020-01-02 ~ 2026-08-14 | 2026-08-17 起 |
| 日线 SPY/QQQ/VIX | 2020-01-02 ~ 2026-07-31 | 2026-08-03 起 |
| 日线 VXN | —（未做过 IBKR 日线导出） | 2026-01-02 起（全量） |

唯一混合文件：`tickers/VXN/VXN_1m_2026-08-14.csv`——IBKR 写到 15:59，Yahoo
同步从 16:00 起追加了盘后段。

**按行分辨来源的指纹**（`average`/`barCount` 两列）：

- IBKR 行：两列有值（分钟均价 / 成交笔数），价格两位小数，volume 带 `.0`；
- Yahoo 行：两列留空（Yahoo 没有这两项），价格是裸 float（含 float32 尾差，
  如 `760.1170043945312`）。

**Yahoo 段的落盘语义**（Kyline `yahooSync`）：未复权价（不请求 adjclose，与
IBKR 对齐）、含盘前盘后；只落盘完整日——当天的数据盘中不写，美东盘后
20:00 收尾后由下一轮同步把整天一次写入。盘中实时数据存在 app 侧的
`~/.config/kyline/live/`（当日文件，超 7 天自动清理），不进本目录。
历史遗留：2026-08-21 ~ 09-18 期间日线曾在盘中写过半根并冻结（追加语义不会
自行修正），2026-09-19 已按 Yahoo 1d 官方值重拉修复（40 根）。

## 成分股全历史（scanline 扫描宇宙）

2026-09-25 起新增：SPY/QQQ **当前成分股**（S&P500 503 + 纳指100 101 → 并集
518）的日线全历史（各股自 IPO 起），供 scanline 扫描（也让 1Q EMA20 首次
可扫：季线 EMA20 需 60 季收敛，2020 年起的短历史扫 1Q 恒空）。由
`sync_yahoo_daily.py` 从 Yahoo v8 chart 接口全量拉取：

- 成分名单：Wikipedia S&P500 成分表 + Nasdaq 官方 API（nasdaq100），
  `BRK.B`→`BRK-B` Yahoo 记法；重跑自动跟随指数调仓更新宇宙
- 请求必须走 curl_cffi 的 Chrome TLS 指纹（裸 python-requests 无论带不带
  cookie 都被 429，2026-09 实测）；串行 + 1~2s 随机抖动，429/5xx 指数退避，
  连续 3 品种限流即熔断中止
- 落盘指纹与上节 Yahoo 段一致：裸 float 价格、整数 volume、
  `average/barCount` 留空；只落完整日（美东 20:00 收尾语义），止于最近
  完整交易日
- **只写新建月份文件，绝不改动既有文件**（SPY/QQQ/VIX/VXN 的
  IBKR/kyline 管理段不受影响；成分股目录全归本脚本管）；增量更新 = 直接
  重跑，已到最新的品种自动跳过
- 状态在 `data/yahoo-daily-sync-state.json`（每品种 last_bar/文件清单/
  上次失败明细）；注意 SPY/QQQ 自身的既有月份更晚，state 里 last_bar 偏旧，
  每轮重跑会白拉一次全历史（无副作用），属已知小浪费

## events/ — 经济事件日历

按月一个 `YYYY-MM.csv`，表头
`datetime,currency,title,impact,forecast,previous,actual`（datetime 为美东
时间，impact = High/Medium/Low）。来源分两段：

- 历史段（2006-12 ~ 2025-04）：HuggingFace `Forex_Factory_Calendar` 数据集
  一次性导入（kyline 仓库 `scripts/ff-history-to-csv.mjs`），缺口由
  wayback 回填脚本补；
- 近期段：Kyline 定时同步 Forex Factory 周更 feed 追加。

手动补的行与同步行共存同一文件：合并键 `datetime+currency+title`，已有行
胜出（同步只增不删，手动补过的 actual 永不被覆盖）。

---

本仓库曾位于 backtest 仓库内的 `data/` 子目录，2026-10-11 起独立为
`vibe/ohlc`（远端 `specv/ohlc`）；`tickers/`、`events/` 与本 readme 已全量
入库，Kyline 经存储根软链访问，路径不变。
