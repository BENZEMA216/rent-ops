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
git clone https://github.com/BENZEMA216/rent-ops.git ~/.claude/skills/rent
~/.claude/skills/rent/scripts/setup.sh
```

### 项目级安装

仅在当前项目中使用：

```bash
mkdir -p .claude/skills
git clone https://github.com/BENZEMA216/rent-ops.git .claude/skills/rent
.claude/skills/rent/scripts/setup.sh
```

### 检查安装状态

```bash
~/.claude/skills/rent/scripts/doctor.sh
```

### 兼容性

本 skill 遵循 [Agent Skills](https://agentskills.io) 开放标准，兼容所有支持该标准的 AI 工具（Claude Code、Cursor、Gemini CLI、VS Code Copilot 等）。

## 快速开始

```bash
# 安装完成后，在任意目录
/rent
```

首次运行会引导你配置城市、预算、户型等租房需求。

## 多城市支持

所有平台 URL、豆瓣小组 ID、行政区和坐标全部通过 `cities/{pinyin}.yml` 驱动。切换城市只改 `config/profile.yml` 的 `city` 字段即可。

**内置城市**

| 拼音 | 中文 | 子域 code | 状态 |
|------|------|----------|------|
| shenzhen | 深圳 | sz | ✅ 完整 |
| beijing  | 北京 | bj | ⚠️ 豆瓣 group_id 未核实 |
| shanghai | 上海 | sh | ⚠️ 豆瓣 group_id 未核实 |
| guangzhou| 广州 | gz | ⚠️ 豆瓣 group_id 未核实 |
| hangzhou | 杭州 | hz | ⚠️ 豆瓣 group_id 未核实 |
| chengdu  | 成都 | cd | ⚠️ 豆瓣 group_id 未核实 |

未核实 = 作者未亲自验证豆瓣小组 ID。第一次跑 `/rent scrape` 时如果报错，按提示去豆瓣搜 `{城市}租房` 小组把 ID 填进对应 yml。

**新增城市：** 见 [`cities/README.md`](cities/README.md)，基本上就是复制 `shenzhen.yml` 改几个字段。

**地图切换城市：** 改完 `config/profile.yml` 后重跑一次 `scripts/build_city_runtime.py` 生成 `data/city-runtime.json`，刷新浏览器即可。

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
~/.claude/skills/rent/scripts/python.sh ~/.claude/skills/rent/scripts/scrape_douban.py --stealth
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
~/.claude/skills/rent/scripts/python.sh ~/.claude/skills/rent/scripts/scrape_douban.py --cookie-file cookies.json
```

### 方式三：Playwright Storage State

如果你有 Playwright 导出的 storage state 文件（包含 cookie + localStorage）：

```bash
~/.claude/skills/rent/scripts/python.sh ~/.claude/skills/rent/scripts/scrape_douban.py --session-file session.json
```

### 在 AI 助手环境中使用（Claude Code 等）

AI 助手环境没有交互式终端，需要提前准备好登录态：

```bash
# 先在终端交互式登录一次，保存 session
~/.claude/skills/rent/scripts/python.sh ~/.claude/skills/rent/scripts/scrape_douban.py --stealth

# 之后在 AI 助手中使用 --non-interactive 模式
~/.claude/skills/rent/scripts/python.sh ~/.claude/skills/rent/scripts/scrape_douban.py --non-interactive
```

或直接提供 cookie / session 文件：

```bash
~/.claude/skills/rent/scripts/python.sh ~/.claude/skills/rent/scripts/scrape_douban.py --non-interactive --cookie-file cookies.json
```

## 可选依赖

| 依赖 | 用途 | 安装 |
|------|------|------|
| MediaCrawler | 小红书爬虫 | `git clone https://github.com/NanmiCoder/MediaCrawler` + `pip install -r requirements.txt`（需 Python 3.11）|
| 高德 JS API Key | 地图可视化（浏览器渲染） | [console.amap.com](https://console.amap.com) → 创建「Web端(JS API)」Key，填入 `data/map-view.html` |
| 高德 Web 服务 Key | **通勤 + 便利维度硬数据** | [console.amap.com](https://console.amap.com) → 创建「Web服务」Key（另一种类型，不是 JS API），复制 `templates/amap.example.yml` 到 `config/amap.yml` 填 `web_service_key`。5000 次/天免费 |
| 唯果直租 MCP | **直租房源数据源**（5 城） | 见下节 |
| Arc 浏览器 | CDP 模式（最强反检测） | [arc.net](https://arc.net) |

## 可选 MCP 数据源：唯果直租

[唯果 vigolive](https://vigolive.cn) 是国内直租平台，提供**官方 MCP server**，让 Claude Code 直接拿到筛选后的直租房源（100% 直租、字段结构化、带坐标）。覆盖北京 / 上海 / 深圳 / 广州 / 杭州 5 城。

**启用**（5 分钟）：

1. 微信搜"**唯果租房**"小程序 → 我的 → **MCP 连接** → 生成 Key（`vgk_live_xxx`）
2. 复制项目根目录的 `.mcp.json.example` 为 `.mcp.json`，填入 key
3. 编辑 `platforms.yml`，把 `vigolive.enabled` 改 `true`
4. Claude Code 重启后会自动加载 `mcp__vigolive__*` 工具

**启用后的差异**：`/rent scan` 会在 L0 先调 `search_houses` / `search_by_commute` 拿结构化直租数据，爬虫作为兜底。

**限制**：30 次/天免费配额 · 仅 5 城 · 只用 `search_*` 工具（他们的 `evaluate` / `verify` / `query_knowledge` 数据覆盖稀疏，rent-ops 自己的流程更强）。

## 通勤 + 便利：从软评分到硬数据

配好 Web 服务 Key 后，两个评分维度自动从"WebSearch 估算"升级为"真实地图数据"：

```bash
# 通勤：{工作地} → {小区}，返回分钟数 + 换乘次数 + 1-5 分
scripts/python.sh scripts/amap_query.py commute --to "望京SOHO" --pretty

# 便利：500m 内 POI 加权分（超市/便利店/餐饮/地铁/医院/菜市场/健身房）
scripts/python.sh scripts/amap_query.py convenience --location "望京SOHO" --pretty
```

类别和权重在 `config/amap.yml` 可调。无 key 时 CLI 返回 `status: disabled`，agent 自动降级 WebSearch 兜底。

## 数据架构

```
config/
└── profile.yml             # 你的租房需求配置（含 city）

cities/                     # 城市抽象层（跨城市通用）
├── shenzhen.yml            # 深圳：code/豆瓣组/片区/坐标
├── beijing.yml
├── shanghai.yml
├── ...
└── README.md               # 如何新增一个城市

data/
├── listings.md             # 房源跟踪表（single source of truth）
├── listings.json           # 结构化数据（地图用）
├── city-runtime.json       # 城市运行时配置（build_city_runtime.py 生成，地图消费）
├── pipeline.md             # 待评估队列
├── map-view.html           # 高德地图可视化（数据驱动，不写死城市）
├── sample-shenzhen.json    # 深圳示例数据兜底
├── douban_raw.jsonl        # 豆瓣原始数据
└── douban_filtered.jsonl   # 豆瓣筛选数据

modes/
├── _shared.md              # 评分规则（系统层）
├── _profile.md             # 个人画像（用户层）
├── auto-evaluate.md        # 自动评估
├── scan.md                 # 平台扫描
├── scrape.md               # 专用爬虫
├── map.md                  # 地图可视化
├── risk.md                 # 风险检测
├── verify.md               # 假房源检测
└── visit.md                # 看房准备

scripts/
├── setup.sh                # 一键安装（venv + Playwright + Chromium）
├── doctor.sh               # 安装状态检查
├── python.sh               # venv Python 封装
├── scrape_douban.py        # 豆瓣爬虫（支持 --city）
├── build_city_runtime.py   # 生成 data/city-runtime.json
└── lib/
    └── city.py             # 城市 loader（profile + cities/*.yml）

reports/                    # 评估报告
```

## License

MIT
