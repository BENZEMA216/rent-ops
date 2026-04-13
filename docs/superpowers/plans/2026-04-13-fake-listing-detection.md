# Fake Listing Detection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add systematic fake listing detection to rent-ops with a new `modes/verify.md`, integrated into the evaluation pipeline and listings-view.html card UI.

**Architecture:** New `modes/verify.md` centralizes all detection logic (11 rules across common + platform-specific layers). Two trigger modes: lite (auto during evaluation, 5 common checks) and full (`/rent verify`, all checks). Results stored in `listings.json` verify field and rendered as risk badges on listing cards.

**Tech Stack:** Claude Code Skill (markdown modes), HTML/CSS/JS (listings-view.html), JSON (listings.json)

---

### Task 1: Create `modes/verify.md` — Common Detection Layer

**Files:**
- Create: `modes/verify.md`

This is the core file. It contains all detection logic the agent follows when running verify.

- [ ] **Step 1: Create `modes/verify.md` with header, mode routing, and platform identification**

```markdown
# 模式：verify — 假房源检测

对房源执行真伪校验，识别假房源、钓鱼低价、二房东冒充等风险。

## 输入

- `/rent verify {url}` — 完整检测（通用层 + 平台专属层）
- 评估 pipeline 自动调用 — 快检（仅通用层）

如果未提供 URL 或文本，询问：

> "请粘贴要校验的房源链接或描述文本。"

## 模式选择

| 调用方式 | 模式 | 检测范围 |
|---|---|---|
| `/rent verify {url}` | full | 通用层 5 项 + 平台专属层 |
| auto-evaluate 步骤 0.5 调用 | lite | 通用层 5 项 |

## 平台识别

根据 URL 域名自动选择平台专属检测项：

| URL 包含 | 平台标识 | 专属检测 |
|---|---|---|
| `douban.com` | 豆瓣 | 发帖人主页、评论区揭露、文案模板 |
| `xiaohongshu.com` | 小红书 | 评论区翻车、博主主页、图片风险 |
| `ke.com` / `lianjia.com` | 贝壳 | 经纪人挂房量、房源状态 |
| `58.com` / `anjuke.com` | 58/安居客 | 同图检测、刷新频率、认证状态 |
| `ziroom.com` | 自如 | 用户差评 |
| 其他 / 纯文本 | 未知 | 仅通用层 |

纯文本输入（无 URL）仅跑通用层。
```

- [ ] **Step 2: Add common detection layer rules to `modes/verify.md`**

Append after the platform identification section:

```markdown
## 风险等级

每项检测产出一个信号：

| 等级 | 含义 |
|---|---|
| `RED` | 高风险，疑似假房源 |
| `YELLOW` | 中风险，需留意 |
| `GREEN` | 无异常 |

### 综合判定

- 1+ 个 `RED` → 整体「高风险」→ **硬拦截**，提示用户确认是否继续
- 2+ 个 `YELLOW`（且无 RED）→ 整体「中风险」→ **软警告**，报告中醒目标注
- 其余 → 「低风险」→ 正常

---

## 通用检测层（所有平台，lite + full 都跑）

### 1. 价格异常

**数据获取：** WebSearch `"{小区名} 租金均价" OR "{商圈} 整租均价"` 获取参考价。

如果 `data/listings.md` 中有该房源的历史价格记录，同时检查价格变动。

| 条件 | 等级 |
|---|---|
| 月租低于商圈均价 30%+ | RED |
| 月租低于均价 15-30% | YELLOW |
| 7 天内降价 30%+（有历史记录时） | RED |
| 14 天内降价 15-30%（有历史记录时） | YELLOW |
| 无历史记录 → 跳过变动检测 | — |

**注意：** 评估模式（lite）中，性价比维度已经会 WebSearch 均价，直接复用该结果，不重复搜索。

### 2. 隔断检测

纯规则匹配，不需要额外搜索。基于步骤 0 抓取到的户型和面积字段。

| 条件 | 等级 |
|---|---|
| 一居室面积 < 15㎡ | RED |
| "两室" / "两房" / "2室" 但面积 ≤ 30㎡ | RED |
| 一居室面积 15-20㎡ | YELLOW |

如果面积字段缺失，标注「面积未知，无法判断」，不评级。

### 3. 信息一致性

由 agent 分析步骤 0 抓取到的标题和正文，判断是否存在矛盾。

| 条件 | 等级 |
|---|---|
| 标题写「整租」但正文描述合租/单间 | RED |
| 标题价格与正文价格不一致 | RED |
| 标题户型与正文户型不一致 | RED |
| 正文缺少小区名、户型、价格等关键信息 | YELLOW |
| 描述过于模糊（如全文仅一句话） | YELLOW |

### 4. 联系方式风险

基于步骤 0 抓取到的联系信息。

| 条件 | 等级 |
|---|---|
| 明确要求转账/交押金才能看房 | RED |
| 无任何联系方式（无电话、无微信、无平台私信入口） | RED |
| 只留微信号，不留电话 | YELLOW |

### 5. 挂牌时长

基于步骤 0 抓取到的发布时间字段。

| 条件 | 等级 |
|---|---|
| 挂牌超过 90 天未下架 | RED |
| 挂牌 45-90 天 | YELLOW |

如果发布时间缺失，标注「发布时间未知，无法判断」，不评级。
```

- [ ] **Step 3: Commit common detection layer**

```bash
cd /tmp/rent-ops-fix
git add modes/verify.md
git commit -m "feat(verify): add verify mode with common detection layer

5 common checks: price anomaly, partition detection, info consistency,
contact risk, listing age. Supports lite/full modes with RED/YELLOW/GREEN
risk levels."
```

---

### Task 2: Add Platform-Specific Detection Layer to `modes/verify.md`

**Files:**
- Modify: `modes/verify.md`

- [ ] **Step 1: Append platform-specific rules for all 5 platforms**

Append to `modes/verify.md` after the common detection layer:

```markdown
---

## 平台专属检测层（仅 full 模式）

以下检测仅在 `/rent verify` 完整模式下执行。根据平台识别结果选择对应检测项。

### 豆瓣（douban.com）

#### 6. 发帖人主页分析

**数据获取：** 从房源帖页面提取发帖人主页链接，访问主页查看：
- 注册时间
- 发帖历史（是否全是房源帖）
- 是否在多个城市发房源

| 条件 | 等级 |
|---|---|
| 注册 < 7 天 且 发帖全是房源 | RED |
| 在 3+ 个不同城市发房源帖 | RED |
| 注册 < 30 天 | YELLOW |

如果主页信息不可访问（隐私设置），标注「发帖人主页不可访问」，不评级。

#### 7. 评论区揭露

**数据获取：** 读取房源帖的评论区内容。

| 条件 | 等级 |
|---|---|
| 评论中出现"中介""假房源""骗子""已租出还挂着"等关键词 3+ 条（不同用户） | RED |
| 1-2 条质疑评论 | YELLOW |

#### 8. 文案模板

**数据获取：** 无需额外搜索，agent 直接分析正文。

| 条件 | 等级 |
|---|---|
| 全文为套话无个性化细节（如"精装修拎包入住交通便利周边配套齐全"无具体小区/楼层/朝向） | RED |
| 部分模板化用语但有具体信息 | YELLOW |

---

### 小红书（xiaohongshu.com）

#### 6. 评论区翻车

**数据获取：** WebSearch `site:xiaohongshu.com "{笔记标题或关键词}"` 或直接从笔记页面读取评论。

| 条件 | 等级 |
|---|---|
| 评论中出现"去了不是这套""和图片不符""照骗"等 3+ 条 | RED |
| 1-2 条类似质疑 | YELLOW |

#### 7. 博主主页分析

**数据获取：** 从笔记页面提取博主主页，检查近期笔记内容。

| 条件 | 等级 |
|---|---|
| 近 30 条笔记全是不同房源 | RED |
| 房源笔记占比 > 70% | YELLOW |

#### 8. 图片风险

**数据获取：** agent 分析笔记配图。

| 条件 | 等级 |
|---|---|
| 图片带其他平台水印（如贝壳、自如 logo） | RED |
| 明显专业精修（HDR 过度、广角畸变、调色统一） | RED |
| 仅有官方样板图，无实拍 | YELLOW |

---

### 贝壳（ke.com / lianjia.com）

#### 6. 经纪人挂房量

**数据获取：** 从房源详情页提取经纪人信息，WebSearch `site:ke.com "{经纪人姓名}" 租房` 查看挂房数。

| 条件 | 等级 |
|---|---|
| 同一经纪人 50+ 套在挂 | RED |
| 30-50 套 | YELLOW |

#### 7. 房源状态验证

**数据获取：** 访问房源 URL 检查页面状态。

| 条件 | 等级 |
|---|---|
| 页面 404 / 显示"已下架"但仍可搜到 | RED |
| 上架超 60 天未更新 | YELLOW |

---

### 58同城 / 安居客（58.com / anjuke.com）

#### 6. 同图检测

**数据获取：** 提取房源配图 URL，WebSearch 图片 URL 看是否被多条房源使用。

| 条件 | 等级 |
|---|---|
| 同图出现在 3+ 条不同房源 | RED |
| 出现在 2 条不同房源 | YELLOW |

#### 7. 刷新频率

**数据获取：** 从房源页面提取"最近刷新"时间信息。

| 条件 | 等级 |
|---|---|
| 每天刷新 | RED |
| 2-3 天刷新一次 | YELLOW |

#### 8. 认证状态

**数据获取：** 从房源页面提取发布者认证信息。

| 条件 | 等级 |
|---|---|
| 未认证 + 价格低于均价 15%+ | RED |
| 未认证（价格正常） | YELLOW |

---

### 自如（ziroom.com）

#### 6. 用户差评

**数据获取：** WebSearch `"{房源名称}" OR "{小区名} 自如" 差评 OR 投诉`

| 条件 | 等级 |
|---|---|
| 搜到 3+ 条不同用户的差评（甲醛、照片不符、设施损坏等） | RED |
| 1-2 条差评 | YELLOW |
```

- [ ] **Step 2: Commit platform-specific layer**

```bash
cd /tmp/rent-ops-fix
git add modes/verify.md
git commit -m "feat(verify): add platform-specific detection rules

Platform rules for Douban (poster analysis, comments, template text),
XHS (comment backlash, blogger profile, image risk), Beike (agent
volume, listing status), 58/Anjuke (image reuse, refresh frequency,
certification), Ziroom (user complaints)."
```

---

### Task 3: Add Output Format to `modes/verify.md`

**Files:**
- Modify: `modes/verify.md`

- [ ] **Step 1: Append output format section and data update instructions**

Append to `modes/verify.md`:

```markdown
---

## 输出

### Full 模式输出（`/rent verify`）

```
🔍 房源真伪校验：{小区名} {户型}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
平台：{平台名} | 综合判定：🔴 高风险 / 🟡 中风险 / 🟢 低风险

── 通用检测 ──
  {🔴/🟡/🟢} 价格异常 — {说明}
  {🔴/🟡/🟢} 隔断检测 — {说明}
  {🔴/🟡/🟢} 信息一致性 — {说明}
  {🔴/🟡/🟢} 联系方式 — {说明}
  {🔴/🟡/🟢} 挂牌时长 — {说明}

── {平台名}专属检测 ──
  {🔴/🟡/🟢} {检测项} — {说明}
  ...（逐项列出该平台的专属检测结果）

{综合建议}
```

综合建议规则：
- 高风险：`⚠️ 高风险（{N} RED / {N} YELLOW）\n建议跳过。{一句话说明最主要的风险}`
- 中风险：`⚠️ 中风险（{N} YELLOW）\n建议谨慎，看房时重点核实以上问题。`
- 低风险：`✅ 低风险，未发现明显异常。`

### Lite 模式输出（评估时自动触发）

高风险拦截：
```
⚠️ 快检发现 {N} 个高风险信号：{逗号分隔的信号摘要}。是否继续评估？
```

中风险警告：
```
⚠️ 快检发现 {N} 个需注意信号：{逗号分隔}。继续评估，报告中将标注。
```

通过：
```
✅ 快检通过，无异常信号。继续评估...
```

### 数据更新

verify 完成后，更新 `data/listings.json` 中对应房源的 `verify` 字段：

```json
{
  "verify": {
    "level": "red",
    "date": "2026-04-13",
    "signals": [
      {"rule": "price_anomaly", "level": "red", "detail": "低于均价38%"},
      {"rule": "new_account", "level": "red", "detail": "注册5天，仅房源帖"}
    ]
  }
}
```

`rule` 字段使用以下标识符：

| 标识符 | 检测项 |
|---|---|
| `price_anomaly` | 价格异常 |
| `partition` | 隔断检测 |
| `info_inconsistency` | 信息一致性 |
| `contact_risk` | 联系方式风险 |
| `listing_age` | 挂牌时长 |
| `poster_profile` | 发帖人主页分析（豆瓣） |
| `comment_exposure` | 评论区揭露（豆瓣）/ 评论区翻车（小红书） |
| `template_text` | 文案模板（豆瓣） |
| `blogger_profile` | 博主主页分析（小红书） |
| `image_risk` | 图片风险（小红书）/ 同图检测（58） |
| `agent_volume` | 经纪人挂房量（贝壳） |
| `listing_status` | 房源状态验证（贝壳） |
| `refresh_frequency` | 刷新频率（58） |
| `certification` | 认证状态（58） |
| `user_complaints` | 用户差评（自如） |

如果该房源已在 `data/listings.md` tracker 中，在备注列追加风险摘要（如 `🔴价格异常`）。

如果该房源不在 tracker 中（独立 verify 调用），不自动添加到 tracker。
```

- [ ] **Step 2: Commit output format**

```bash
cd /tmp/rent-ops-fix
git add modes/verify.md
git commit -m "feat(verify): add output format and data update rules

Full mode shows per-check results with RED/YELLOW/GREEN indicators.
Lite mode shows one-line summary. Results saved to listings.json
verify field with structured signal data."
```

---

### Task 4: Register `/rent verify` in SKILL.md

**Files:**
- Modify: `.claude/skills/rent-ops/SKILL.md`

- [ ] **Step 1: Add `verify` to the mode routing table**

In `.claude/skills/rent-ops/SKILL.md`, find the mode routing table and add `verify`:

Replace this line:
```
| `visit` | `visit` |
```

With:
```
| `verify` | `verify` — 假房源检测 |
| `visit` | `visit` |
```

- [ ] **Step 2: Add `verify` to the Discovery Mode menu**

In the Discovery Mode section, find:
```
/rent risk {小区名}       → 搜集该小区避雷信息
```

Add after it:
```
/rent verify {链接}       → 假房源检测（价格异常/隔断/二房东/平台专属检查）
```

- [ ] **Step 3: Add `verify` to Context loading rules**

In the "Context 加载规则" section, find:
```
### 需要 _shared.md + _profile.md + mode 文件：
- `auto-evaluate`, `scan`
```

Change to:
```
### 需要 _shared.md + _profile.md + mode 文件：
- `auto-evaluate`, `scan`, `verify`
```

- [ ] **Step 4: Commit SKILL.md changes**

```bash
cd /tmp/rent-ops-fix
git add .claude/skills/rent-ops/SKILL.md
git commit -m "feat(verify): register /rent verify command in SKILL.md

Adds verify to mode routing, discovery menu, and context loading rules."
```

---

### Task 5: Integrate Verify Lite into `modes/auto-evaluate.md`

**Files:**
- Modify: `modes/auto-evaluate.md`

- [ ] **Step 1: Insert step 0.5 between step 0 and step 1**

In `modes/auto-evaluate.md`, find:
```
## 步骤 1 — 红线检查
```

Insert before it:
```markdown
## 步骤 0.5 — 假房源快检

在评估前对房源执行快速真伪校验（verify lite 模式，仅通用层 5 项检测）。

**执行 `modes/verify.md` 中通用检测层的全部 5 项检查。**

使用步骤 0 已抓取的房源信息（户型、面积、价格、联系方式、发布时间）和即将进行的 WebSearch 结果（商圈均价）。

### 结果处理

- **高风险（1+ RED）：**

  > "⚠️ 快检发现 {N} 个高风险信号：{信号摘要}。是否继续评估？"

  等待用户确认。用户说继续 → 照常评估，报告中醒目标注。用户说跳过 → 标记状态为 `放弃`（原因：快检高风险）。

- **中风险（2+ YELLOW，无 RED）：**

  > "⚠️ 快检发现 {N} 个需注意信号：{信号摘要}。继续评估，报告中将标注。"

  不拦截，直接继续评估。

- **低风险：**

  > "✅ 快检通过，无异常信号。继续评估..."

  直接继续。

### verify 结果复用

快检结果传递给后续步骤，避免重复搜索：
- 步骤 2 **风险维度** → 直接使用快检结果映射为 1-5 分（0 RED 0 YELLOW = 5 分，1 YELLOW = 4 分，2+ YELLOW = 3 分，1 RED = 2 分，2+ RED = 1 分）
- 步骤 2 **可靠性维度** → 联系方式风险检测结果纳入可靠性评分

```

- [ ] **Step 2: Commit auto-evaluate changes**

```bash
cd /tmp/rent-ops-fix
git add modes/auto-evaluate.md
git commit -m "feat(verify): integrate lite verify into auto-evaluate pipeline

Adds step 0.5 between data extraction and red-line check. High risk
triggers hard block, medium risk shows warning, low risk passes through.
Verify results reused by risk and reliability scoring dimensions."
```

---

### Task 6: Update `modes/_shared.md` to Reference Verify Results

**Files:**
- Modify: `modes/_shared.md`

- [ ] **Step 1: Update risk and reliability dimensions**

In `modes/_shared.md`, find the scoring table:
```
| **房东/中介可靠性** | 直租 vs 二房东、信息真实度 | 房东直租+信息一致 | 正规中介 | 疑似二房东/信息矛盾 |
| **风险** | 隔断、避雷帖、负面口碑 | 无风险信号 | 有轻微负面 | 多个红旗 |
```

Replace with:
```
| **房东/中介可靠性** | 直租 vs 二房东、信息真实度 | 房东直租+信息一致 | 正规中介 | 疑似二房东/信息矛盾 |
| **风险** | 假房源检测信号（verify 结果） | verify 无风险信号 | verify 有 YELLOW | verify 有 RED |
```

- [ ] **Step 2: Add verify result mapping note after the scoring table**

Find:
```
### 评分解读
```

Insert before it:
```markdown
### verify 结果映射

如果评估前已执行 verify（步骤 0.5 快检或独立 `/rent verify`），以下维度直接使用 verify 结果，不重复搜索：

- **风险维度：** 0 RED + 0 YELLOW = 5, 1 YELLOW = 4, 2+ YELLOW = 3, 1 RED = 2, 2+ RED = 1
- **可靠性维度：** 联系方式风险 GREEN = 不影响, YELLOW = 降 1 分, RED = 降 2 分

其余维度（性价比、通勤、房况、安全、便利、灵活性）不受 verify 影响，按原有逻辑评分。

```

- [ ] **Step 3: Commit _shared.md changes**

```bash
cd /tmp/rent-ops-fix
git add modes/_shared.md
git commit -m "feat(verify): map verify results to risk/reliability scores

Risk dimension now directly uses verify signal counts. Reliability
dimension incorporates contact risk results. Avoids duplicate WebSearch."
```

---

### Task 7: Update `modes/risk.md` to Delegate to Verify

**Files:**
- Modify: `modes/risk.md`

- [ ] **Step 1: Add verify reference to risk.md**

In `modes/risk.md`, find:
```
### 4. 二房东检测
```

Replace the entire section 4 and section 5:
```
### 4. 二房东检测

如果有该房源的发帖人信息（联系电话或昵称），搜索：
- `"{联系电话}" 租房` — 看同一号码是否在多个不同小区挂房
- `"{发帖人昵称}" 租房` — 看是否批量发帖

同一人在 3 个以上不同小区挂房源 → 高概率二房东。

### 5. 隔断线索

如果有房源面积信息，检查：
- 一居室面积 < 20㎡ → 疑似隔断
- 户型描述与面积不匹配（如 "两室" 但只有 30㎡）
```

With:
```
### 4. 假房源检测

如果用户提供了具体房源链接，建议使用 `/rent verify {url}` 获取更全面的假房源检测结果，包括：
- 二房东检测（联系方式跨小区、发帖人主页分析）
- 隔断检测（面积 vs 户型匹配）
- 价格异常检测
- 平台专属检测（评论区、图片风险等）

详见 `modes/verify.md`。

如果用户只提供了小区名（无具体链接），仍按以下规则检测：
- `"{联系电话}" 租房` — 看同一号码是否在多个不同小区挂房
- `"{发帖人昵称}" 租房` — 看是否批量发帖
- 一居室面积 < 15㎡ / 两室 ≤ 30㎡ → 疑似隔断
```

- [ ] **Step 2: Commit risk.md changes**

```bash
cd /tmp/rent-ops-fix
git add modes/risk.md
git commit -m "refactor(risk): delegate fake listing checks to verify mode

risk.md now points to /rent verify for detailed per-listing fake
detection. Retains basic checks for community-name-only queries."
```

---

### Task 8: Add Risk Badges to `data/listings-view.html`

**Files:**
- Modify: `data/listings-view.html`

- [ ] **Step 1: Add CSS styles for risk badges**

In `data/listings-view.html`, find:
```css
  .card.highlight::before { content: "⭐ 推荐"; position: absolute; top: 12px; right: 12px; font-size: 11px; background: #0071e3; color: #fff; padding: 2px 8px; border-radius: 10px; }
```

Add after it:
```css
  .card.risk-red { border-color: #c62828; }
  .card.risk-red::after { content: "🔴 疑似假房源"; position: absolute; top: 12px; right: 12px; font-size: 11px; background: #c62828; color: #fff; padding: 2px 8px; border-radius: 10px; }
  .card.risk-yellow { border-color: #f57f17; }
  .card.risk-yellow::after { content: "🟡 需留意"; position: absolute; top: 12px; right: 12px; font-size: 11px; background: #f57f17; color: #fff; padding: 2px 8px; border-radius: 10px; }
  .card.risk-red.highlight::before { display: none; }

  .risk-detail { font-size: 12px; color: #c62828; margin-top: 8px; padding-top: 8px; border-top: 1px solid #f5f5f7; }
  .risk-detail.yellow { color: #f57f17; }
```

- [ ] **Step 2: Add data attributes and risk detail row to example cards**

This step documents the pattern for the agent to follow when generating/updating listings-view.html. Find the first card:

```html
<div class="card highlight" data-area="后海" data-price="7100" data-direct="false" data-cat="false">
```

The agent should add `data-risk="none"` attribute. When verify finds risks, the agent sets `data-risk="red"` or `data-risk="yellow"` and adds the corresponding CSS class and a risk detail div before the card-footer:

```html
<div class="card risk-red" data-area="后海" data-price="2800" data-direct="false" data-cat="false" data-risk="red">
  <div class="card-top">...</div>
  <div class="card-meta">...</div>
  <div class="card-desc">...</div>
  <div class="risk-detail">🔴 价格偏离均价38% · 发帖人注册5天</div>
  <div class="card-footer">...</div>
</div>
```

Add `data-risk="none"` to all existing cards (use replace-all on the pattern). Find:
```
data-cat="false">
```
No — since some cards have `data-cat="true"`, we need to add it after the last data attribute on each card div. The agent generating/updating this file should add `data-risk` to each card.

For now, add `data-risk="none"` to all existing card divs by appending it. This is best done per-card, but since the file is agent-generated, document the pattern here and add risk filter button.

- [ ] **Step 3: Add risk filter button**

Find:
```html
  <button class="filter-btn" onclick="filterCards('budget7k', this)">≤7000元</button>
```

Add after it:
```html
  <button class="filter-btn" onclick="filterCards('risk', this)">⚠️ 有风险</button>
```

- [ ] **Step 4: Add risk filter logic to filterCards function**

Find:
```javascript
    } else if (type === 'budget7k') {
      const price = parseInt(card.dataset.price);
      show = price > 0 && price <= 7000;
    }
```

Add after it:
```javascript
    } else if (type === 'risk') {
      show = card.dataset.risk === 'red' || card.dataset.risk === 'yellow';
    }
```

- [ ] **Step 5: Commit listings-view.html changes**

```bash
cd /tmp/rent-ops-fix
git add data/listings-view.html
git commit -m "feat(verify): add risk badge styles and filter to listings view

Cards with verify results show red/yellow risk badges. Added risk
filter button to filter bar. Risk detail row shows signal summary."
```

---

### Task 9: Final Review and Push

**Files:**
- All modified files

- [ ] **Step 1: Verify all files are consistent**

```bash
cd /tmp/rent-ops-fix
git log --oneline -10
```

Check that all 7 commits are present (Tasks 1-8, some tasks share commits).

- [ ] **Step 2: Verify verify.md is complete and self-contained**

```bash
cd /tmp/rent-ops-fix
cat modes/verify.md | head -5
wc -l modes/verify.md
```

Verify the file has all sections: mode routing, platform identification, common layer (5 checks), platform layer (5 platforms), output format, data update rules.

- [ ] **Step 3: Verify SKILL.md has verify registered**

```bash
cd /tmp/rent-ops-fix
grep -n "verify" .claude/skills/rent-ops/SKILL.md
```

Should show verify in mode routing, discovery menu, and context loading.

- [ ] **Step 4: Push to origin**

```bash
cd /tmp/rent-ops-fix
git push origin main
```
