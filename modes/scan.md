# 模式：scan — 主动扫描各平台

按三级策略扫描配置中的租房平台，筛选符合条件的房源，去重后添加到 pipeline。

## ⚠ 强制执行流程（必须严格按此顺序）

**不要跳过任何步骤，不要调换顺序。**

### Step 0: 结构化 MCP 数据源（如已配置）

在爬虫之前，**先检查是否有可用的结构化 MCP 房源源**。当前支持：

**vigolive（唯果直租）** — 如果 `platforms.yml` 中 `vigolive.enabled: true` 且当前 profile.city 在 `vigolive.cities` 列表里：

1. 并发调用：
   ```
   mcp__vigolive__search_houses(city=profile.city, location=work_area,
     price_min=budget.min, price_max=budget.max,
     rent_type=1 if type=="整租" else 2,
     bedrooms=[int(r[0]) for r in profile.rooms], page_size=20)
   mcp__vigolive__search_by_commute(city=profile.city,
     workplace=anchors[0].address, max_commute_minutes=30, limit=20)
   ```
2. 返回的 house 对象字段齐全（`house_id / compound / latitude / longitude /
   rent_price_yuan / tags / near_subway`），直接写入 pipeline（无需再 geocode）
3. **只覆盖 5 城（北京/上海/深圳/广州/杭州）**，其他城市跳过本步，走 Step 1 爬虫兜底
4. **只用 search_*** 这两个工具当数据源。`evaluate_house` / `verify_house` /
   `query_knowledge` 的覆盖太稀，继续走 rent-ops 本身的评分/verify 流程
5. 30 次/天免费配额，超了自动重置（第二天）

### Step 1: 跑专用爬虫（豆瓣）

立即执行，不要先做 WebSearch：

```bash
${CLAUDE_SKILL_DIR}/scripts/python.sh ${CLAUDE_SKILL_DIR}/scripts/scrape_douban.py --stealth --non-interactive
```

读取输出 `${CLAUDE_SKILL_DIR}/data/douban_filtered.jsonl`，按 profile 筛选后加入候选列表。

### Step 2: 跑专用爬虫（小红书，如已安装）

检查 `~/code/MediaCrawler` 是否存在。存在则执行：

```bash
cd ~/code/MediaCrawler && /opt/homebrew/bin/python3.11 main.py --platform xhs --lt qrcode --type search
```

不存在则告知用户：
> "小红书爬虫（MediaCrawler）未安装，跳过。如需安装：`git clone https://github.com/NanmiCoder/MediaCrawler ~/code/MediaCrawler`"

### Step 3: 抓取贝壳/自如（WebFetch）

对 platforms.yml 中 `enabled: true` 且有 `scan_url` 的平台，用 WebFetch 抓取列表页。

如果 WebFetch 返回空或被反爬拦截，**提示用户**：
> "贝壳/自如列表页被反爬拦截，无法直接抓取。你可以：
> 1. 在浏览器中打开 {scan_url}，手动复制几条房源链接给我评估
> 2. 跳过这个平台
>
> 怎么操作？"

### Step 4: WebSearch 补充（兜底）

仅在 Step 1-3 结果不足时，用 WebSearch 补充搜索。

### Step 5: 汇总去重，更新 pipeline

合并所有结果 → 去重 → 筛选 → 写入 pipeline.md + scan-history.tsv。

---

## 执行建议

如果平台数量较多，建议作为 subagent 执行：
```
Agent(
  subagent_type="general-purpose",
  prompt="[modes/_shared.md 内容]\n\n[modes/_profile.md 内容]\n\n[本文件内容]\n\n[profile.yml 和 platforms.yml 内容]",
  description="rent-ops scan"
)
```

## 准备

1. 读取 `${CLAUDE_SKILL_DIR}/platforms.yml` — 平台配置和筛选关键词
2. 读取 `${CLAUDE_SKILL_DIR}/config/profile.yml` — 预算、户型等硬性条件
3. 读取 `${CLAUDE_SKILL_DIR}/data/scan-history.tsv` — 已扫描过的 URL（如果文件存在）
4. 读取 `${CLAUDE_SKILL_DIR}/data/listings.md` — 已在 tracker 中的房源
5. 读取 `${CLAUDE_SKILL_DIR}/data/pipeline.md` — 已在队列中的房源

---

## L1 — WebFetch 抓取（需要 cookie 的平台）

对 `${CLAUDE_SKILL_DIR}/platforms.yml` 中 `enabled: true` 且 strategy 包含 `cookie_playwright` 的平台：

### 前提：Cookie 初始化

**必须明确提示用户**，不要静默跳过：

> "扫描 {平台名} 需要你的登录态。请先在浏览器中登录 {平台名}，然后我用 WebFetch 抓取列表页。
>
> 如果你还没登录，可以先跳过这个平台，后续再扫。
>
> **登录还是跳过？**"

### 扫描流程

用 WebFetch 抓取列表页：

```
WebFetch(url="{scan_url}", prompt="提取所有房源列表项，包括：标题、租金、户型、链接URL。以结构化格式返回。")
```

1. 根据 profile.yml 中的条件构造带筛选参数的 URL：
   - 贝壳：`https://{city}.ke.com/zufang/{区域代码}/rt200600000001l0/` （整租+一居）
   - 自如：`https://{city}.ziroom.com/z/p1-r0-t10/` （整租筛选参数）

2. 从 WebFetch 结果中提取房源信息：
   - 标题、租金、URL、户型
   - 如果结果不完整，尝试翻页 URL（如 `pg2/`、`pg3/`），最多 5 页

3. 对每个房源 URL：
   - 检查是否在 scan-history.tsv / listings.md / pipeline.md 中 → 跳过
   - 检查标题是否符合 filters.keywords_positive/negative → 不符合则跳过
   - 检查租金是否在 budget 范围内 → 超出则跳过
   - 通过所有检查 → 加入候选列表

### 各平台特殊处理

**贝壳找房 (ke.com)：**
- 列表页 URL 格式：`https://{city_code}.ke.com/zufang/`
- `{city_code}` 在 platforms.yml 里模板化，取值来自 `cities/{pinyin}.yml` 的 `code` 字段（深圳=sz、北京=bj、上海=sh、广州=gz、杭州=hz、成都=cd 等）
- 筛选参数拼接在 URL 路径中（区域、价格、户型）
- 房源链接格式：`https://{city_code}.ke.com/zufang/{id}.html`

**自如 (ziroom.com)：**
- 列表页 URL 格式：`https://www.ziroom.com/z/{city_code}/`
- 自如房源多为自营，信息标准化程度高
- 注意区分「整租」和「合租」tab

## L2 — API 扫描

对 strategy 包含 `api` 的平台：

**贝壳 API（如可用）：**
- WebFetch 尝试贝壳的列表接口
- 返回 JSON 结构化数据
- 提取 title、price、url、area 字段
- 作为 L1 的补充，用于快速获取房源列表

如果 API 调用失败或返回异常，静默回退到 L1 或跳过。不要因 API 问题中断整个扫描。

## L2.5 — 专用爬虫扫描（优先级最高）

对有专用爬虫脚本的平台，**优先使用，不要先走 L3 WebSearch**：

### 豆瓣（scripts/scrape_douban.py）

```bash
${CLAUDE_SKILL_DIR}/scripts/python.sh ${CLAUDE_SKILL_DIR}/scripts/scrape_douban.py --stealth --non-interactive
```

- 优先尝试 CDP 模式（接管 Arc 浏览器），降级为 playwright-stealth + Arc cookie 注入
- 目标小组 = `${CLAUDE_SKILL_DIR}/cities/{profile.city}.yml` 的 `douban.group_id`（深圳 613105、其他城市同字段）
- 区域正则由 `cities/{pinyin}.yml` 的 `areas` + `sub_areas` 自动生成
- 触发 misc/sorry 验证页时暂停等待人工过滑块
- 输出：`${CLAUDE_SKILL_DIR}/data/douban_raw.jsonl` + `${CLAUDE_SKILL_DIR}/data/douban_filtered.jsonl`
- 依赖：运行 `${CLAUDE_SKILL_DIR}/scripts/setup.sh` 安装

### 小红书（MediaCrawler）

```bash
cd ~/code/MediaCrawler && /opt/homebrew/bin/python3.11 main.py --platform xhs --lt qrcode --type search
```

- 使用 xhshow 纯算法签名（无需浏览器中间件签名）
- 有已保存的登录 session，通常不需要重新扫码
- 关键配置（MediaCrawler/config/base_config.py）：
  - `KEYWORDS`: 关键词用英文逗号分隔
  - `CRAWLER_MAX_NOTES_COUNT = 20`：每组关键词抓取量
  - `CRAWLER_MAX_SLEEP_SEC = 5`：请求间隔（降低风控）
  - `ENABLE_GET_COMMENTS = False`：不抓评论减少请求量
  - `SAVE_DATA_OPTION = "jsonl"`
- 输出：`~/code/MediaCrawler/data/xhs/jsonl/search_contents_*.jsonl`

**如果 MediaCrawler 未安装**，提示用户：
> "小红书爬虫需要 MediaCrawler，目前未安装。要跳过小红书还是先安装？
> 安装方式：`git clone https://github.com/NanmiCoder/MediaCrawler ~/code/MediaCrawler`"

### 数据整合

爬虫完成后，需要将结果整合到 rent-ops：
1. 读取 douban_filtered.jsonl + MediaCrawler 输出
2. 按 INCLUDE/EXCLUDE/AREA 正则过滤
3. 写入 `${CLAUDE_SKILL_DIR}/data/listings.md` 和 `${CLAUDE_SKILL_DIR}/data/listings.json`（地图用）
4. 更新 `${CLAUDE_SKILL_DIR}/data/pipeline.md`（待评估队列）

## L3 — WebSearch 扫描（兜底）

**仅在以下情况使用 L3：**
- 平台没有专用爬虫，也没有 cookie
- 用户明确选择跳过 cookie 初始化后，作为补充手段
- 作为其他级别的补充（发现更多房源）

对 strategy 包含 `websearch` 的平台：

1. 构建搜索 query：
   - 从 `${CLAUDE_SKILL_DIR}/platforms.yml` 读取 `scan_query` 模板
   - 替换 `{city_name}` 为 profile.yml 中的 `city` 字段（中文名）
   - 替换 `{city_code}` 为 `cities/{pinyin}.yml` 的 `code` 字段（如 sz/bj）
   - 替换 `{keywords}` 为 filters.keywords_positive 中的关键词

2. 执行 WebSearch

3. 从结果中提取房源信息：
   - URL、标题、描述

4. **验证有效性（仅 L3 需要）：**
   WebSearch 结果可能过期。对每个 L3 发现的房源 URL：
   ```
   WebFetch(url="{房源URL}", prompt="这个页面是否包含有效的房源信息？提取标题、价格、联系方式。如果页面是404或已下架，说明无效。")
   ```
   - 页面有房源内容（标题+描述+联系方式） → 有效
   - 404 / 已下架 / 内容为空 → 标记 `skipped_expired`

## 去重

所有级别的结果合并后去重：

1. **URL 精确匹配** — scan-history / listings / pipeline 中已有 → skip
2. **ID 匹配** — 如果已有房源中有相同的联系电话或发帖人昵称 → 标记 `可能重复` 并关联
3. **图片匹配** — 如果配图 URL 与已有房源相同 → 标记 `可能重复` 并关联

标记为 `可能重复` 的房源**仍然加入 pipeline**，但在备注中注明关联的房源编号，让用户决定。

## 结果处理

### 加入 pipeline
对每个通过筛选和去重的新房源，添加到 `${CLAUDE_SKILL_DIR}/data/pipeline.md` 的「待处理」区：
```
- [ ] {url} | {平台} | {小区/标题} {户型} {租金}
```

### 记录历史
在 `${CLAUDE_SKILL_DIR}/data/scan-history.tsv` 中记录所有扫描到的 URL（无论是否通过筛选）：
```
{url}\t{YYYY-MM-DD}\t{平台}\t{标题}\t{状态}
```

状态值：`added`（已加入 pipeline）、`skipped_filter`（被筛选过滤）、`skipped_dup`（重复）、`skipped_expired`（已过期）

## 输出摘要

```
🔍 扫描完成 — {YYYY-MM-DD}
━━━━━━━━━━━━━━━━━━━━━━━━━━
平台扫描：{N} 个
房源发现：{N} 条
筛选通过：{N} 条
重复跳过：{N} 条（已评估或已在 pipeline）
过期跳过：{N} 条（L3 链接已失效）
新增到 pipeline：{N} 条

+ {平台} | {小区} {户型} {租金}
+ {平台} | {小区} {户型} {租金}
...

→ 粘贴链接直接评估，或 /rent batch 批量评估 pipeline
```
