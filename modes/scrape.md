# 模式：scrape — 执行豆瓣/小红书专用爬虫

运行专用爬虫脚本抓取房源数据，比 scan 模式的 WebSearch 覆盖更全。

## 依赖检查

执行前检查依赖是否就绪：

```bash
python3 -c "from playwright.async_api import async_playwright; print('playwright OK')"
python3 -c "from playwright_stealth import Stealth; print('stealth OK')"
```

如果缺失，提示用户安装：
```
pip3 install playwright playwright-stealth && playwright install chromium
```

## 豆瓣爬虫

**脚本：** `${CLAUDE_SKILL_DIR}/scripts/scrape_douban.py`

### 前提
- 用户已有豆瓣账号，并在 Arc 浏览器中登录过豆瓣
- 用户已加入目标租房小组（默认：深圳租房，ID 613105）

### 运行

```bash
# 默认模式（自动尝试 CDP → stealth → 手动验证）
python3 ${CLAUDE_SKILL_DIR}/scripts/scrape_douban.py

# 强制 stealth 模式（推荐，无需重启 Arc）
python3 ${CLAUDE_SKILL_DIR}/scripts/scrape_douban.py --stealth

# 强制 CDP 模式（需先以调试模式启动 Arc）
# 先运行：/Applications/Arc.app/Contents/MacOS/Arc --remote-debugging-port=9222
python3 ${CLAUDE_SKILL_DIR}/scripts/scrape_douban.py --cdp
```

### 输出
- `${CLAUDE_SKILL_DIR}/data/douban_raw.jsonl` — 所有抓到的帖子（追加写入）
- `${CLAUDE_SKILL_DIR}/data/douban_filtered.jsonl` — 匹配筛选条件的帖子（覆盖写入）
- `${CLAUDE_SKILL_DIR}/data/douban_session.json` — 浏览器 session（后续复用）

### 触发验证页时
脚本检测到 `misc/sorry` 页面会：
1. 尝试点击验证按钮（打开 Tencent 滑块）
2. 暂停并提示用户手动拖动滑块
3. 用户按 Enter 后继续抓取

## 小红书爬虫

**工具：** MediaCrawler（外部项目）

### 前提
- MediaCrawler 已 clone 到 `~/code/MediaCrawler`
- Python 3.11 环境下安装了依赖（`pip install -r requirements.txt`）
- 首次需手机扫码登录（后续复用 session）

### 配置

在运行前确认 `~/code/MediaCrawler/config/base_config.py` 中的关键参数：

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `PLATFORM` | `"xhs"` | 固定 |
| `KEYWORDS` | 根据用户 profile 调整 | 英文逗号分隔 |
| `CRAWLER_MAX_NOTES_COUNT` | `20` | 每组关键词抓取量（降低风控） |
| `CRAWLER_MAX_SLEEP_SEC` | `5` | 请求间隔秒数 |
| `ENABLE_GET_COMMENTS` | `False` | 不抓评论，减少请求 |
| `SAVE_DATA_OPTION` | `"jsonl"` | 输出格式 |
| `HEADLESS` | `False` | 需要可见浏览器 |

根据 `${CLAUDE_SKILL_DIR}/config/profile.yml` 中的城市、区域、户型生成关键词，更新到 `base_config.py`。

### 运行

```bash
cd ~/code/MediaCrawler && /opt/homebrew/bin/python3.11 main.py --platform xhs --lt qrcode --type search
```

- 如果有已保存的 session，通常不需要重新扫码
- 如果提示登录，会打开浏览器窗口显示二维码

### 输出
- `~/code/MediaCrawler/data/xhs/jsonl/search_contents_YYYY-MM-DD.jsonl`

## 数据整合

爬虫完成后，将结果合并到 rent-ops 数据：

1. 读取 `${CLAUDE_SKILL_DIR}/data/douban_filtered.jsonl` + MediaCrawler 输出
2. 按 profile.yml 中的条件过滤（区域、户型、预算、排除词）
3. 去重（URL 精确匹配 + scan-history.tsv）
4. 更新 `${CLAUDE_SKILL_DIR}/data/listings.md`（tracker）
5. 更新 `${CLAUDE_SKILL_DIR}/data/listings.json`（地图数据）
6. 记录到 `${CLAUDE_SKILL_DIR}/data/scan-history.tsv`

## 输出摘要

```
🕷 爬虫完成 — {YYYY-MM-DD}
━━━━━━━━━━━━━━━━━━━━━━━━━━
豆瓣：抓取 {N} 条，筛选 {N} 条
小红书：抓取 {N} 条，筛选 {N} 条
新增到 tracker：{N} 条
地图数据已更新

→ /rent map 查看地图 · /rent tracker 管理房源
```
