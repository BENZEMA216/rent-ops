<p align="center">
  <img src="docs/banner.png" alt="rent-ops banner" width="100%">
</p>

<h1 align="center">rent-ops</h1>
<p align="center">AI 租房助手 — Agent Skill</p>
<p align="center">自动爬取各平台房源 · 高德地图可视化 · 八维度智能评分 · 风险预警</p>

---

## 功能

| 命令 | 功能 |
|------|------|
| `/rent {链接}` | 粘贴房源链接，自动评估打分 |
| `/rent scrape` | 爬取豆瓣租房小组 + 小红书房源 |
| `/rent map` | 高德地图可视化所有房源 |
| `/rent scan` | WebSearch 扫描各平台 |
| `/rent risk {小区}` | 搜集社交媒体避雷信息 |
| `/rent verify {链接}` | 假房源检测（价格异常/隔断/二房东/平台专属检查） |
| `/rent visit` | 看房准备（路线 + checklist + 砍价话术） |
| `/rent tracker` | 管理房源跟踪表 |

## 安装

### 全局安装（推荐）

在任意目录下都能使用 `/rent`：

```bash
# 1. 克隆到 Claude Code skills 目录
git clone https://github.com/BENZEMA216/rent-ops.git ~/.claude/skills/rent

# 2. 安装 Python 依赖
cd ~/.claude/skills/rent
pip3 install playwright playwright-stealth
playwright install chromium
```

### 项目级安装

仅在当前项目中使用：

```bash
mkdir -p .claude/skills
git clone https://github.com/BENZEMA216/rent-ops.git .claude/skills/rent
cd .claude/skills/rent && pip3 install playwright playwright-stealth && playwright install chromium
```

### 兼容性

本 skill 遵循 [Agent Skills](https://agentskills.io) 开放标准，兼容所有支持该标准的 AI 工具（Claude Code、Cursor、Gemini CLI、VS Code Copilot 等）。

## 快速开始

```bash
# 安装完成后，在任意目录
/rent
```

首次运行会引导你配置城市、预算、户型等租房需求。

## 技术栈

- **爬虫**: Playwright + playwright-stealth（豆瓣）、MediaCrawler + xhshow 签名（小红书）
- **反检测**: CDP 接管真实浏览器 / stealth 补丁 / Arc cookie 注入
- **地图**: 高德地图 JS API v2.0 + PlaceSearch 地理编码
- **评估**: 八维度评分体系（性价比/通勤/房况/安全/便利/信誉/风险/灵活性）
- **交互**: Agent Skill，纯对话式操作

## 豆瓣登录配置

豆瓣爬虫（`/rent scrape`）需要登录态才能访问租房小组内容。提供三种方式：

### 方式一：交互式登录（推荐新手）

在终端直接运行脚本，浏览器会打开豆瓣页面，手动登录后按 Enter 继续：

```bash
python3 ~/.claude/skills/rent/scripts/scrape_douban.py --stealth
```

登录成功后会自动保存 session 到 `data/douban_session.json`，后续运行无需重复登录。

### 方式二：手动导出 Cookie

从浏览器导出豆瓣 cookie，保存为 JSON 文件：

1. 在 Chrome / Edge / Arc 中登录豆瓣
2. 安装浏览器扩展 [EditThisCookie](https://www.editthiscookie.com/) 或使用 DevTools
3. 导出 `.douban.com` 域下的 cookie，保存为 JSON 数组格式：

```json
[
  {
    "name": "dbcl2",
    "value": "你的值",
    "domain": ".douban.com",
    "path": "/",
    "secure": true,
    "httpOnly": true
  }
]
```

4. 运行时指定 cookie 文件：

```bash
python3 ~/.claude/skills/rent/scripts/scrape_douban.py --cookie-file cookies.json
```

### 方式三：Playwright Storage State

如果你有 Playwright 导出的 storage state 文件（包含 cookie + localStorage）：

```bash
python3 ~/.claude/skills/rent/scripts/scrape_douban.py --session-file session.json
```

### 在 AI 助手环境中使用（Claude Code 等）

AI 助手环境没有交互式终端，需要提前准备好登录态：

```bash
# 先在终端交互式登录一次，保存 session
python3 ~/.claude/skills/rent/scripts/scrape_douban.py --stealth

# 之后在 AI 助手中使用 --non-interactive 模式
python3 ~/.claude/skills/rent/scripts/scrape_douban.py --non-interactive
```

或直接提供 cookie / session 文件：

```bash
python3 ~/.claude/skills/rent/scripts/scrape_douban.py --non-interactive --cookie-file cookies.json
```

## 可选依赖

| 依赖 | 用途 | 安装 |
|------|------|------|
| MediaCrawler | 小红书爬虫 | `git clone https://github.com/NanmiCoder/MediaCrawler` + `pip install -r requirements.txt`（需 Python 3.11）|
| 高德地图 API Key | 地图可视化 + POI 搜索 | 在 [console.amap.com](https://console.amap.com) 注册，创建「Web端(JS API)」类型的 Key |
| Arc 浏览器 | CDP 模式（最强反检测） | [arc.net](https://arc.net) |

## 数据架构

```
config/
└── profile.yml          # 你的租房需求配置

data/
├── listings.md          # 房源跟踪表（single source of truth）
├── listings.json        # 结构化数据（地图用）
├── pipeline.md          # 待评估队列
├── map-view.html        # 高德地图可视化
├── douban_raw.jsonl     # 豆瓣原始数据
└── douban_filtered.jsonl # 豆瓣筛选数据

modes/
├── _shared.md           # 评分规则（系统层）
├── _profile.md          # 个人画像（用户层）
├── auto-evaluate.md     # 自动评估
├── scan.md              # 平台扫描
├── scrape.md            # 专用爬虫
├── map.md               # 地图可视化
├── risk.md              # 风险检测
├── verify.md            # 假房源检测
└── visit.md             # 看房准备

reports/                 # 评估报告
scripts/
└── scrape_douban.py     # 豆瓣爬虫
```

## License

MIT
