# 模式：scan — 主动扫描各平台

按三级策略扫描配置中的租房平台，筛选符合条件的房源，去重后添加到 pipeline。

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

## L1 — Cookie + Playwright 扫描

对 `${CLAUDE_SKILL_DIR}/platforms.yml` 中 `enabled: true` 且 strategy 包含 `cookie_playwright` 的平台：

### 前提：Cookie 导入

在扫描前，检查是否有该平台的 cookie：
```
$B cookies
```

如果没有目标平台的 cookie，提示用户：

> "扫描 {平台名} 需要你的登录态。我可以从你的浏览器导入 cookie：
> `$B cookie-import-browser --domain {cookie_domain}`
> 或者你也可以先跳过这个平台。导入还是跳过？"

### 扫描流程

```
$B goto {scan_url}
$B snapshot -i
```

1. 根据 profile.yml 中的条件设置平台筛选器（价格范围、户型、区域）：
   - 用 `$B click` / `$B fill` / `$B select` 操作筛选控件
   - 如果页面有区域筛选，选择工作地点附近的区域

2. 遍历房源列表：
   - `$B snapshot -i` 获取列表页元素
   - 提取每个房源：标题、租金、URL
   - 如果有分页，翻页继续（最多 5 页）

3. 对每个房源 URL：
   - 检查是否在 scan-history.tsv / listings.md / pipeline.md 中 → 跳过
   - 检查标题是否符合 filters.keywords_positive/negative → 不符合则跳过
   - 检查租金是否在 budget 范围内 → 超出则跳过
   - 通过所有检查 → 加入候选列表

### 各平台特殊处理

**贝壳找房 (ke.com)：**
- 列表页 URL 格式：`https://{city}.ke.com/zufang/`
- 筛选器：价格、户型、区域都有下拉菜单
- 房源链接格式：`https://{city}.ke.com/zufang/{id}.html`

**自如 (ziroom.com)：**
- 列表页 URL 格式：`https://{city}.ziroom.com/z/`
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

## L2.5 — 专用爬虫扫描

对有专用爬虫脚本的平台，优先使用：

### 豆瓣（scripts/scrape_douban.py）

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/scrape_douban.py --stealth
```

- 优先尝试 CDP 模式（接管 Arc 浏览器），降级为 playwright-stealth + Arc cookie 注入
- 自动翻页浏览「深圳租房」小组最新帖子
- 触发 misc/sorry 验证页时暂停等待人工过滑块
- 输出：`${CLAUDE_SKILL_DIR}/data/douban_raw.jsonl` + `${CLAUDE_SKILL_DIR}/data/douban_filtered.jsonl`
- 依赖：`pip install playwright playwright-stealth`

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

### 数据整合

爬虫完成后，需要将结果整合到 rent-ops：
1. 读取 douban_filtered.jsonl + MediaCrawler 输出
2. 按 INCLUDE/EXCLUDE/AREA 正则过滤
3. 写入 `${CLAUDE_SKILL_DIR}/data/listings.md` 和 `${CLAUDE_SKILL_DIR}/data/listings.json`（地图用）
4. 更新 `${CLAUDE_SKILL_DIR}/data/pipeline.md`（待评估队列）

## L3 — WebSearch 扫描

对 strategy 包含 `websearch` 的平台：

1. 构建搜索 query：
   - 从 `${CLAUDE_SKILL_DIR}/platforms.yml` 读取 `scan_query` 模板
   - 替换 `{city}` 为 profile.yml 中的城市
   - 替换 `{keywords}` 为 filters.keywords_positive 中的关键词

2. 执行 WebSearch

3. 从结果中提取房源信息：
   - URL、标题、描述

4. **验证有效性（仅 L3 需要）：**
   WebSearch 结果可能过期。对每个 L3 发现的房源 URL：
   ```
   $B goto {url}
   $B text
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
