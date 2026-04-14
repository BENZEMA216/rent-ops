# 模式：map — 高德地图可视化

在浏览器中打开房源地图，展示所有已跟踪的房源位置。

## 前提

- 高德地图 JS API Key（Web端类型）— 需在 [console.amap.com](https://console.amap.com) 申请
- API Key 和安全密钥已配置在 `${CLAUDE_SKILL_DIR}/data/map-view.html` 中

## 执行

### 1. 确认数据文件

检查 `${CLAUDE_SKILL_DIR}/data/listings.json` 是否存在且非空。如果不存在或为空：
- 提示用户先运行 `/rent scrape` 或 `/rent scan` 获取房源数据

### 2. 启动 HTTP 服务

高德地图 JS API 不支持 `file://` 协议，必须通过 HTTP 访问：

```bash
cd ${CLAUDE_SKILL_DIR}/data && python3 -m http.server 8765 &
```

检查端口是否已被占用（`lsof -i :8765`），如已占用则跳过启动。

### 3. 打开地图

```bash
open http://localhost:8765/map-view.html
```

## 地图功能

### 标记
- 红色脉冲圆点：工作地点（深圳湾口岸）
- 彩色圆点：房源位置，颜色按区域区分
- 大圆（有数字）：有明确价格的房源，数字 = 月租/千元
- 小圆（?）：价格待询

### 筛选器
顶部 chip 筛选：按区域（后海/南油/蛇口/深圳湾/前海/科技园）、按价格（≤7000）、按平台（小红书/豆瓣）

### 详情面板
点击圆点弹出右侧面板：
- 平台来源 + 小区名
- 价格 + 户型
- 距工作地直线距离
- 「查看原帖」链接

### 地理编码
新抓取的房源通过高德 PlaceSearch API 自动定位小区坐标：
- 从标题提取小区名（如「云海天城」「万科云城」）→ POI 搜索 → 真实坐标
- 提取失败时降级到区域中心点，面板标注「大致位置」
- 右下角显示定位进度

## 更新地图数据

地图数据来自 `${CLAUDE_SKILL_DIR}/data/listings.json`。更新流程：

1. 运行 `/rent scrape`（爬取新数据）
2. 数据整合脚本会自动更新 `listings.json`
3. 刷新浏览器页面即可看到新房源

## API Key 配置

用户需要在 `${CLAUDE_SKILL_DIR}/data/map-view.html` 中配置两个值：

```html
<script>
window._AMapSecurityConfig = { securityJsCode: '你的安全密钥' };
</script>
<script src="https://webapi.amap.com/maps?v=2.0&key=你的API_Key&plugin=..."></script>
```

获取方式：
1. 访问 [console.amap.com](https://console.amap.com)
2. 创建应用 → 添加 Key → 选择「Web端(JS API)」
3. 复制 Key 和安全密钥到上述位置
