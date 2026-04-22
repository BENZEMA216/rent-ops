---
name: rent
description: AI 租房助手 — 扫描各平台房源、智能评估打分、跨平台去重、高德地图可视化、风险预警、看房准备。当用户提到租房、找房、房源评估、看房时使用。
user-invocable: true
argument-hint: "[子命令或房源链接]"
compatibility: Requires Python 3.9+, Playwright, playwright-stealth. Optional: MediaCrawler (小红书), 高德地图 API Key (地图可视化).
allowed-tools: Read Write Edit Bash WebSearch WebFetch Glob Grep
---

# rent-ops — AI 租房助手

## 路径约定

本 skill 的所有文件路径都相对于 `${CLAUDE_SKILL_DIR}`（即本 SKILL.md 所在目录）。

读写文件时，始终使用 `${CLAUDE_SKILL_DIR}/` 前缀解析路径。执行 shell 命令时同理。

---

## Mode 路由

根据 `$ARGUMENTS` 确定执行哪个模式：

| 输入 | Mode |
|------|------|
| （空 / 无参数） | `discovery` — 显示命令菜单 |
| 房源链接或描述文本（非子命令） | **`auto-evaluate`** |
| `scan` | `scan` |
| `tracker` | `tracker` |
| `risk` | `risk` |
| `verify` | `verify` — 假房源检测 |
| `visit` | `visit` |
| `map` | `map` — 打开地图可视化 |
| `scrape` | `scrape` — 执行豆瓣/小红书爬虫 |
| `compare` | 提示「v0.2 开发中」 |
| `negotiate` | 提示「v0.2 开发中」 |
| `batch` | 提示「v0.2 开发中」 |

**自动检测：** 如果 `$ARGUMENTS` 不是已知子命令，但包含房源链接（ke.com、ziroom.com、58.com、douban.com、xiaohongshu.com、xianyu.com），自动走 `auto-evaluate`。

如果 `$ARGUMENTS` 既不是子命令也不像房源链接，显示 discovery。

---

## Discovery Mode（无参数）

显示：

```
🏠 rent-ops — AI 租房助手

可用命令：

/rent {粘贴链接}          → 自动评估（抓取 + 打分 + 风险扫描 + 入 tracker）
/rent scan               → 主动扫描各平台找房（WebSearch/Playwright）
/rent scrape             → 执行豆瓣/小红书爬虫（Playwright stealth + MediaCrawler）
/rent map                → 打开高德地图可视化（所有房源定位展示）
/rent tracker            → 查看/管理房源跟踪表
/rent risk {小区名}       → 搜集该小区避雷信息
/rent verify {链接}       → 假房源检测（价格异常/隔断/二房东/平台专属检查）
/rent visit              → 看房准备（路线 + checklist + 砍价话术）

即将支持：
/rent compare            → 多房源横向对比
/rent negotiate {房源#}   → 砍价策略生成
/rent batch              → 批量评估

直接粘贴房源链接即可开始评估。
```

---

## Context 加载规则

确定 mode 后，加载对应文件再执行。所有路径基于 `${CLAUDE_SKILL_DIR}/`：

### 需要 _shared.md + _profile.md + mode 文件：
- `auto-evaluate`, `scan`, `verify`

加载顺序：先读 `${CLAUDE_SKILL_DIR}/modes/_shared.md`，再读 `${CLAUDE_SKILL_DIR}/modes/_profile.md`（用户配置覆盖系统默认），最后读 `${CLAUDE_SKILL_DIR}/modes/{mode}.md`。

### 需要 _shared.md + _profile.md + mode 文件 + tracker 数据：
- `visit`

额外读取 `${CLAUDE_SKILL_DIR}/data/listings.md` 获取已有房源信息。

### 需要 _shared.md + _profile.md + mode 文件 + platforms.yml：
- `scrape`

额外读取 `${CLAUDE_SKILL_DIR}/platforms.yml` 获取爬虫配置和运行命令。

### 独立 mode（只读自己的 mode 文件）：
- `tracker`, `risk`, `map`

---

## 环境依赖

### 必须

运行安装脚本即可自动配置所有依赖（Python venv + Playwright + Chromium）：

```bash
${CLAUDE_SKILL_DIR}/scripts/setup.sh
```

检查安装状态：

```bash
${CLAUDE_SKILL_DIR}/scripts/doctor.sh
```

### 推荐（增强功能）

| 依赖 | 用途 | 安装 |
|------|------|------|
| MediaCrawler | 小红书爬虫 | `git clone https://github.com/NanmiCoder/MediaCrawler` + `pip install -r requirements.txt`（需 Python 3.11）|
| 高德地图 JS API Key | 地图可视化（浏览器） | [console.amap.com](https://console.amap.com) → 创建「Web端(JS API)」Key，填入 `data/map-view.html` |
| 高德地图 Web 服务 Key | **通勤 + 便利维度硬数据**（通勤分钟数、POI 密度） | [console.amap.com](https://console.amap.com) → 创建「Web服务」Key（不同于 JS API），复制 `templates/amap.example.yml` 到 `config/amap.yml` 填 `web_service_key`。每日 5000 次免费 |
| 唯果直租 MCP | **结构化直租房源数据源**（5 城：京/沪/深/广/杭） | 微信搜"唯果租房"小程序 → 我的 → MCP 连接 → 生成 Key。`cp .mcp.json.example .mcp.json` 填入 `vgk_live_xxx`。Claude Code 会自动加载 `mcp__vigolive__*` 工具。30 次/天免费 |
| Arc 浏览器 | CDP 模式（最强反检测） | [arc.net](https://arc.net) |

---

## Onboarding（首次运行）

**每次会话首次触发 /rent 时**，在执行任何 mode 之前，静默检查以下文件：

0. `${CLAUDE_SKILL_DIR}/.venv/bin/python3` 是否存在？如果不存在，提示用户运行 `${CLAUDE_SKILL_DIR}/scripts/setup.sh`。
1. `${CLAUDE_SKILL_DIR}/config/profile.yml` 是否存在？
2. `${CLAUDE_SKILL_DIR}/modes/_profile.md` 是否存在？
3. `${CLAUDE_SKILL_DIR}/data/listings.md` 是否存在？
4. `${CLAUDE_SKILL_DIR}/platforms.yml` 是否存在？

**如果 `modes/_profile.md` 缺失**，从 `${CLAUDE_SKILL_DIR}/modes/_profile.template.md` 静默复制。

**如果其他文件缺失，进入引导模式。** 不要在配置完成前执行评估或扫描。

### Step 1: 用户信息（必需）
如果 `config/profile.yml` 缺失，询问：

> "我需要了解你的租房需求，帮你配置一下：
> - 你在哪个城市？（内置：深圳 / 北京 / 上海 / 广州 / 杭州 / 成都；其他按 `${CLAUDE_SKILL_DIR}/cities/README.md` 补一份）
> - 工作地点在哪？（地铁站、小区或详细地址，任意格式）
> - 预算范围？
> - 整租还是合租？偏好什么户型？
> - 有什么绝对不能接受的？（红线）
> - 还有什么特别在意的？"

根据回答生成 `${CLAUDE_SKILL_DIR}/config/profile.yml`。`city` 字段可填中文名（深圳）或拼音（shenzhen）。

若用户城市不在内置列表里，引导用户：
1. 复制 `cities/shenzhen.yml` 为 `cities/{pinyin}.yml`
2. 按 `cities/README.md` 填 `code`（平台子域）/ `douban.group_id` / 主要 `areas`

### Step 2: 平台配置（推荐）
如果 `platforms.yml` 缺失：

> "我会配置常用的租房平台（贝壳、自如、豆瓣、小红书、安居客、房天下、58）。所有 URL 都已模板化，会按 profile.yml 里的 city 自动适配。"

从 `${CLAUDE_SKILL_DIR}/templates/platforms.example.yml` 复制到 `${CLAUDE_SKILL_DIR}/platforms.yml`。**不要再手动改 URL**，`{city_code}` 占位符由 `scripts/lib/city.py` 解析。

### Step 3: Tracker 初始化
如果 `data/listings.md` 不存在，创建空 tracker：

```markdown
# 房源跟踪

| # | 日期 | 平台 | 小区 | 户型 | 租金 | 评分 | 状态 | Report | 备注 |
|---|------|------|------|------|------|------|------|--------|------|
```

同时创建空的 `${CLAUDE_SKILL_DIR}/data/pipeline.md`：

```markdown
# 待评估房源

## 待处理

## 已处理
```

### Step 4: 了解用户（重要）
基础配置完成后，主动询问更多上下文：

> "基本配置好了。你可以告诉我更多关于你的情况，帮助我更准确地评估：
> - 你是一个人住还是跟人合租？养宠物吗？
> - 在家做饭多吗？（厨房重要性）
> - 有没有之前租房踩过的坑？
>
> 这些信息我会存到你的个人画像里，越详细我找房越准。"

将回答存入 `${CLAUDE_SKILL_DIR}/modes/_profile.md`。

### Step 5: 就绪
所有文件就位后确认：

> "配置完成！你可以：
> - 粘贴房源链接让我评估
> - `/rent scan` 让我去各平台找房
> - `/rent` 查看所有命令"

---

## 主要文件

| 文件 | 路径 | 功能 |
|------|------|------|
| 用户配置 | `config/profile.yml` | 筛选条件和偏好（含 `city`） |
| 用户画像 | `modes/_profile.md` | 自由文本个人画像 |
| 城市配置 | `cities/*.yml` | 每城一份：code、豆瓣组、片区、坐标 |
| 城市 loader | `scripts/lib/city.py` | 读 profile + 加载 city yml |
| 运行时生成 | `scripts/build_city_runtime.py` | 生成 `data/city-runtime.json`（地图用） |
| 高德 API 配置 | `config/amap.yml` | Web 服务 Key + POI 类别权重 |
| 高德客户端 | `scripts/lib/amap.py` | geocode / POI / 路径规划 |
| 高德查询 CLI | `scripts/amap_query.py` | 通勤 / POI / 便利分（agent 调用） |
| 房源跟踪表 | `data/listings.md` | single source of truth |
| 待评估队列 | `data/pipeline.md` | 扫描后待评估 |
| 扫描历史 | `data/scan-history.tsv` | 去重用 |
| 平台配置 | `platforms.yml` | 抓取配置（URL 里用 `{city_code}` 模板） |
| 地图运行时 | `data/city-runtime.json` | 合并 profile + city yml，map 消费 |
| 地图数据 | `data/listings.json` | 结构化数据（地图用） |
| 地图页面 | `data/map-view.html` | 高德地图可视化（数据驱动，不写死城市） |
| 深圳示例数据 | `data/sample-shenzhen.json` | city=shenzhen 且无 listings.json 时兜底 |
| 豆瓣原始数据 | `data/douban_raw.jsonl` | 爬取原始数据 |
| 豆瓣筛选数据 | `data/douban_filtered.jsonl` | 筛选后数据 |
| 豆瓣爬虫 | `scripts/scrape_douban.py` | 支持 `--city`；默认读 profile.yml |
| 评估报告 | `reports/` | 每套房源的评估报告 |

所有路径相对于 `${CLAUDE_SKILL_DIR}/`。

---

## 伦理约束

- **永远不自动签约或提交申请**
- **不捏造房源信息或评分** — 数据不足时如实标注
- **控制抓取频率** — 尊重平台 ToS
- **隐私保护** — 用户数据只存在本地
