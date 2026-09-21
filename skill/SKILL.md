---
name: travel-guide
description: "生成精美可分享的旅游攻略HTML页面。输入目的地、游玩日期、天数和必去景点，自动产出包含每日行程、预计游玩时间、预算明细、交通方式、手绘SVG地图（可切换路线、点击景点弹窗+高德导航）和漫风/油画风插画的完整攻略，最后发布成公开可分享链接。适用于：做旅游攻略、行程规划、旅行计划、出游安排、假期安排等场景。用户提到'旅游攻略'、'行程规划'、'旅行计划'、'帮我安排行程'、'XX天去哪玩'时触发。"
---

# Travel Guide — 精美旅游攻略生成器

根据用户提供的目的地、日期、天数和必去景点，产出一个自包含 HTML 攻略页，并用 `lark-cli apps +deploy` 发布成公开可分享链接。

## 输入要素（缺失就问，不要自己猜）

| 要素 | 说明 | 必填 |
|------|------|------|
| 目的地 | 城市/国家/地区 | 是 |
| 游玩日期 | 出发-结束日期或天数 | 是 |
| 必去景点 | 用户点名要去的（可多个） | 是 |
| 住宿点 | 住哪（酒店/火车站/区域），路线从这里放射 | 否，默认市中心 |
| 出发地 | 算大交通预算用 | 否 |
| 预算档次 | 穷游/舒适/豪华 | 否，默认舒适 |
| 粉丝打卡点 | 如某小说/影视剧取景地，与必去景点同问 | 否 |

## 工作流程

### Step 1: 调研

用 `general_search` 搜三类，每类至少一组：
1. `[目的地] 必去景点 门票价格 开放时间 游玩时长`
2. `[目的地] 景点之间 地铁公交 打车 多久`
3. `[目的地] 必吃美食 推荐餐厅 住宿区域`

国庆/五一/暑期等旺季要额外搜"需要预约吗"、"限流"、"提前几天约"。

### Step 2: 排行程

- **地理就近**：同片区景点排同一天，住宿点作为每天路线起点
- **节奏**：Day1 轻松（落地/老城闲逛），中间天重头戏，最后半天留给返程
- **必去景点**必须排进去且放对时段（日出/日落/喂鸽子类）
- **粉丝打卡点**如果和某天行程顺路，**合并进那天的路线**，不要单独拉一条线；用强调色小圆点区分即可
- 每天总游玩 6-8 小时，每个景点标注：起止时间、门票、交通方式（地铁X号线/打车约X分钟¥X）

### Step 3: 预算

人均分类汇总：住宿 × 晚数 / 当地交通 / 门票 / 餐饮（含特色餐）/ 大交通（如有出发地）/ 其他。做成表格，列出合计。

### Step 4: 生成插画

用 `image_gen` 生成，画风统一（默认**吉卜力/日系漫风**，用户要油画风再换）：
1. 封面 hero：城市标志性景观，16:9
2. 每天 1 张当日代表场景，3:2

**必须下载到本地**（image_gen 返回的 URL 会过期），存到产物目录下 `assets/`，HTML 里用相对路径 `assets/xxx.jpg`。

### Step 5: 写 HTML（严格按下面结构）

**单文件自包含**：CSS 内联 `<style>`，JS 内联 `<script>`，原生 JS 不引框架。

页面自上而下：
1. **Hero 封面**：大标题 + 日期天数 + 封面图
2. **手绘 SVG 地图**（核心交互，见下）
3. **每日时间线**：卡片式，每天含标题、亮点、带时间的景点/餐厅节点、当日小插画
4. **预算表**：table-layout: fixed，每列写死宽度
5. **美食推荐**：卡片网格
6. **注意事项**：预约/穿搭/天气/旺季人流
7. Footer

### 手绘 SVG 地图（不要用 Leaflet/高德JS API，会失败）

**已验证失败的方向**：Leaflet + 高德 webrd 瓦片 —— 底图能加载但 polyline/circleMarker 叠加层不可见、按钮无响应。**不要再试真实地图方案**，用户明确接受手绘地图。

**手绘 SVG 方案要点**：
- viewBox 800×600 左右，背景画几条浅灰色道路网格 + 河流/绿地色块（用城市真实大致方位摆点即可，不需要精确经纬度）
- 住宿点用红色五角星/房子图标标注，写"住宿（所有路线起点）"
- 每天一条路线：`<path>` 用 `stroke-dasharray` 虚线，颜色按天固定
  - Day1 蓝 #4a90d9 / Day2 深蓝 #2d5f8a / Day3 金 #e8a838 / Day4 绿 #7bb37b / Day5 紫 #8b5cf6…
  - **不要用 marker-end 箭头**，用户明确拒绝
- 每个景点一个 `<g class="spot-dot">` 圆点 + `<text>` 名称标签
  - 必去景点/粉丝打卡点圆点大一号或换强调色
- **路线切换**：SVG 下方一排按钮"全部路线 / Day1 / Day2…"，点击时只显示对应 `.route-group`，其他 `display:none`
- **点击景点弹窗**：圆点 `onclick` 弹出详情（名称、一句话介绍、预计游玩时间、门票、底部并排"高德地图/百度地图"两个大按钮）
- **高德/百度双导航链接**：
  - **直接用网页版URL当 `<a href>`，target=_blank**，不要用JS scheme唤起（实测在各种手机浏览器里都不可靠：location.href/iframe scheme都会被拦截，微信内置浏览器直接屏蔽）：
    - 高德：`https://uri.amap.com/search?keyword=景点名&city=城市名&callnative=1`（callnative=1让网页版尝试唤起App）
    - 百度：`https://map.baidu.com/mobile/webapp/search/search/flatnew?query=景点名&city=城市名URL编码`（如南京=`南京`）。**不要用** `map.baidu.com/?newmap=1&qt=s&wd=XXX&c=315`——这是百度内部API，直接返回JSON不是网页
    - 弹窗按钮和浮层按钮都用真 `<a>` 标签，href在showSpotDetail/pickNav里动态设置，不要用 `javascript:void(0)` 包JS函数——实测 `onclick="openAmap(_curKw)"` 在手机上经常无反应
    - 用户在网页版地图里会看到"打开App"按钮，自己点即可
  - **时间线里只放一个"🧭 导航"小按钮**（不要并排两个，挤），点击时弹出小浮层让用户选"高德地图/百度地图"，选完才跳转
  - 浮层实现：固定定位 div + 两个 `<a>`，点击按钮时 `pickNav(keyword, event)` 把浮层定位到鼠标位置，点空白处关闭
  - **inline onclick 里不要写 `event.stopPropagation()`**——手机微信/内置浏览器里全局 `event` 对象不稳定，会导致整行JS报错、按钮点不动。改成 `onclick="pickNav('关键词', event)"`，把 `ev.stopPropagation()` 放到函数内部
  - **没有具体地址的行不要导航按钮**：如"抵达XX·入住酒店"（酒店没具体名）、"自由活动"、"秦淮河夜景"（这是活动描述不是POI）这类流程/活动行，不要放导航键；只有具体景点/餐厅/地点才放
  - 数据里只存高德URL，JS用正则从 `keyword=([^&]+)` 提取关键词再拼百度URL，不要重复维护两份数据

### Step 6: 发布成公开链接

用户要"能分享的链接"时才发布（不要默认发布）。有两种渠道，按用户需求选：

**渠道A：GitHub Pages（真正公开，任何人免登录访问，推荐用于对外分享）**
- 把 index.html + assets/ 传到 GitHub 仓库根目录
- Settings → Pages → Branch 选 main，/root 保存
- 等1分钟，链接形如 `https://<用户名>.github.io/<仓库名>/`
- 适合：发给微信好友、朋友圈、不登录就能看
- 注意：图片路径写成 `assets/xxx.jpg`（相对路径），不要多套一层子目录导致 Pages 找不到 index.html

**渠道B：doubao-html（豆包内部发布，需登录豆包才能看）**
```powershell
cd 到产物目录
lark-cli apps +deploy --file-path './xxx.html'
lark-cli apps +release-get --app-id <app_id> --release-id <release_id>
```
`+deploy` 会自动爬取同目录相对路径引用的图片一起打包。**首次返回的 app_id 要记住**，复发更新用 `--app-id`。

## 硬性规则（踩过的坑）

- **写文件用 Write 工具，不要用 PowerShell `>`/`Out-File`**：Windows 默认 GBK 会把中文写成乱码，文件直接打不开。所有中文 HTML/JS 必须 UTF-8 写。
- **图片路径只写在 `src` 或 CSS `url()` 里**：写在 JS 数组/`data-src` 里发布后会裂图。
- **不要加多余模块**：用户没要的"概览卡片""统计数字"之类不要加；保持攻略本身。
- **不用 emoji**，用内联 SVG 图标。
- **路线合并原则**：同一天的打卡点必须合并到该天路线，不要单独拉线；用圆点颜色区分即可。
- **小说/影视剧打卡点不要把原名当标题/标签**：用户提到某小说取景地打卡，正文里可以提，但标题、地图标签、弹窗tag一律用中性描述（如"文艺打卡"），不要把小说名/角色名直接写进标题——用户明确要求不要把小说名作为标题。
- **不要做箭头 marker**：SVG 路线末端不要 `marker-end="url(#arrow)"`。
- **移动端适配（必做）**：
  - `<meta name="viewport" content="width=device-width, initial-scale=1.0">`
  - 卡片网格用 `repeat(auto-fit, minmax(270px, 1fr))`，窄屏自动单列
  - SVG地图 `width:100%; height:auto; display:block`
  - 表格必须外包 `<div style="overflow-x:auto">` 并设 `min-width`，手机上可横滑
  - 写两档媒体查询：`@media (max-width:768px)`（平板/大手机）和 `@media (max-width:480px)`（小手机）
  - 768px 里：hero高度降到52vh、容器padding缩到14px、时间线圆点和缩进缩小、弹窗info grid变1fr 1fr、按钮padding缩小、美食/tips网格变单列
  - 480px 里：hero再降到48vh、弹窗info grid变单列、h1字号降到1.9rem
  - 弹窗 `max-width:460px; width:100%; margin:16px; max-height:85vh; overflow-y:auto`
  - 时间线左侧竖线+圆点布局在小屏缩进从36px缩到28px

## 交付

- 本地版：`present_files` 交付单个 .html 文件（图片在同目录 assets/）
- 公开版：`present_files` 只交付 `online_url` 链接，不要再附 html 文件
- 回复控制在 8 行内，只说重点
