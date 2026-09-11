# 开发交接 · Xpc每日任务

> 给「接手这个项目的下一次对话」看：换电脑、隔很久再来、换一个 AI 助手，读这一份就能接上。
> 如果本机的项目记忆目录（`.workbuddy/memory/`）还没搬过来，**以本文件为准**。

---

## 一句话

给 Xpc 做的手机端个人工作台（PWA）：管每天的待办、行程、复盘。纯前端单文件，数据存在浏览器本地，可选同步到自己的私有 Gist。

## 位置

| | |
|---|---|
| 线上 | https://xpchengx.github.io/xpc-daily/ |
| 仓库 | https://github.com/Xpchengx/xpc-daily （**公开**，因为 GitHub Pages 免费版只支持公开仓）|
| 工程 | 本文件所在目录 |
| 部署 | GitHub Pages，`main` 分支 / 根目录，push 后自动发布 |

用户数据（待办、日志等）**不在仓库里**，存在浏览器本地 + 用户自己的私有 Gist。仓库公开不会泄露数据。

## 文件

```
index.html      全部界面 + 逻辑（约 3000 行，全内联 —— 原因见下方约束）
manifest.json   PWA 配置（"添加到主屏幕"用）
icon.png        桌面图标
HANDOVER.md     本文件
部署指南.html    给用户看的上线步骤（历史文件，可保留）
风格预览.html    最初定配色用的三版预览（历史文件，可保留）
```

## 技术约束（改代码前必读）

1. **所有代码内联在 `index.html`，不许拆文件。** 验收用的冒烟测试只加载 index.html，拆出去的文件它抓不到，等于测试白跑。
2. **卡片折叠编号 `data-fold="cN"` 是写死的，新卡片只能用新编号，绝不许重排。** 用户的折叠状态按编号存在 Store 里，重排会让状态串到别的卡片。目前用到 c27。
3. **不能在多个页面共用同一套渲染的地方用 `getElementById` 取值。** 同一条数据会同时出现在多个页面（一条待办会出现在「今天」页、「待办」页、计划日历里），id 会重复，`getElementById` 只返回第一个 —— 曾经因此丢掉用户写的备注。一律用 `btn.closest('.note-edit')` 这种从点击元素往上找的写法。
4. **不许用 `setInterval` 轮询**（验收脚本会拦）。同步只在打开时和手动点击时拉取。
5. **所有用户输入必须过 `Util.esc()`** 再拼进 HTML。
6. 主题色全在 `:root` 的 CSS 变量里，改色只改变量。当前是"护眼提亮"版暗色（底 `#141b23`、正文 `#d7dfe8`、强调 `#58a6ff`）。
7. 无构建步骤、无 npm 依赖，改完刷新即生效。

## 代码结构

| 模块 | 职责 |
|---|---|
| `Util` | 转义、日期、当前时间等工具 |
| `Store` | 存储抽象。`list/upsert/softDelete` 管条目；`listDaily/upsertDaily` 是"按天存"（行程完成状态用它）；`NO_SYNC` 控制哪些 key 不上云；`mergeList` 管双端合并（含墓碑） |
| `UI` | toast / 底部常驻提示条 / banner |
| `GistSync` | 同步到 GitHub 私有 Gist（浏览器直连 GitHub API）。**默认关闭** |
| `Haptics` | 震动反馈。安卓走 `navigator.vibrate`；iPhone 用屏幕外原生开关兜底（不保证生效） |
| `Picker` | 自绘时间滚轮（**有边界、不循环**）+ 日历面板，替代系统原生控件 |
| `Fold` | 全站卡片折叠（点标题栏收起），状态存 `fold` 这个 key |
| `App` | 路由与启动 |
| `Todo` | 待办：挂时间、P1/P2/P3、跨天顺延、责任人、备注、手动排序 |
| `Drag` | 待办手动排序（长按约 0.3 秒拖动，**限同优先级内**） |
| `Today` | 「今天」页（含"已办"卡片） |
| `Plan` | 计划：日历视图（默认）+ 列表视图；行程支持日期和责任人 |
| `Recap` | 复盘：日报三个数 + 折线图 + 生成图卡 |
| `Goal` / `Log` / `Docs` | 年度目标 / 工作日志 / 云文档入口 |

**模块顺序有讲究**：`Todo.refresh()` 会调用 `Today/Plan/Recap` 的 render，所以那几个要定义在 Todo 之后（或调用处用 `typeof` 兜底）。

功能块都用 `/* ==== 功能：XX START/END ==== */` 包着，加功能请插新块、别改旧块。

## 关键设计决策（用户确认过的，别自作主张推翻）

1. **待办的 `due` 是"原始应完成日"，永不改写。** 拖了几天靠「今天 − due」现算。这样双端合并不会打架，用户改系统时间也不会乱。
2. **待办在日历里的归天规则**：已完成 → 实际完成那天；未完成且已到期（含顺延中）→ 今天；未完成未到期 → 原定那天。所以拖了 5 天的任务出现在"今天"格，不会赖在 5 天前。
3. **排序：优先级是大顺序，拖动只能在同优先级内换位**（跨组会被钳回）。组内没拖过时按"拖得久的在前"，拖过就按手动顺序（存 `ord`）。有「↺ 按规则排」一键回自动。
4. **每天重复的行程不进日历格子**（否则每格都满，看不出哪天有特别的事），单独列在日历下方。
5. **工作日志默认不上云**（设置页有开关，可打开；开关状态本身参与同步）。
6. **待办页只显示"今天完成的"**，历史完成记录归复盘页按天统计。
7. **复盘页的数字是实时算的，不存快照** —— todo 有 `createdAt`/`doneAt`，任意一天都能算，改数据不会出现"日报和实际对不上"。
8. **删除用软删除（墓碑）**：`softDelete` 留一条带 `_d` 标记的记录，否则另一台设备同步回来会把它复活。`list()` 过滤墓碑，`Store.get()` 底层保留。

## 开发流程

改完按这个顺序走，别跳：

```bash
# 1. 静态检查（拦死链、缺 id、未转义等）
python "~/.workbuddy/skills/bys-personal-dashboard/scripts/validate_dashboard.py" .

# 2. 冒烟测试（headless 加载 index.html，9 条基础断言）
node "~/.workbuddy/skills/bys-personal-dashboard/scripts/smoke_test.js" .

# 3. 本项目自己的回归测试（4 个文件，在会话工作目录的 .workbuddy/ 下）
#    t-cal.js（待办进日历）/ t-owner.js（责任人）/ t-recap.js（排序+复盘）/ t-fix3.js（日历拆组+折线图）
NODE_PATH="~/.workbuddy/skills/bys-personal-dashboard/node_modules" node t-xxx.js <工程目录>

# 4. 提交推送
git add -A && git commit -m "..." && git push origin main

# 5. 等 40 秒左右，验证线上真的是新版
curl -s https://xpchengx.github.io/xpc-daily/ | grep -c "<这次新加的特征字符串>"
```

> 依赖：技能包 `bys-personal-dashboard`（验收脚本都在里面，**不跟账号走，新电脑要重装**）；冒烟测试需要 `jsdom`，在技能目录下 `npm install jsdom`，跑时带 `NODE_PATH`。
> 写回归测试时注意：jsdom 里 `getBoundingClientRect()` 永远返回全 0，拖动排序的落点计算依赖它，要手动 mock 每个元素的 rect。

## 踩过的坑（省你几小时）

1. **GitHub Pages 发布会卡住**。症状：推上去了、仓库里是新版，但线上一直是旧版，等 20 分钟都不动。
   - 先分清是"没推上"还是"没构建"：抓 `https://raw.githubusercontent.com/Xpchengx/xpc-daily/main/index.html`（绕开 Pages）确认仓库内容；再看线上响应头的 `Last-Modified`（**GMT，+8 才是北京时间**）是不是上一版的时间。
   - 修法：`git commit --allow-empty -m "chore: 触发重建" && git push`，实测 40 秒内发布。
   - 不要以为是缓存问题 —— 加 `?t=时间戳`、换路径都没用，`X-Cache: HIT` 只是 CDN 状态。
2. **XSS 断言不能用 `innerHTML` 判。** 浏览器序列化属性时只把 `"` 编码成 `&quot;`，属性值里的 `<` `>` 是原样输出的，会误报"没转义"。正确判法是查真实 DOM 有没有多出来的元素：`!box.querySelector('img')`。
3. **jsdom 不认构造参数里的 `userAgent`**，要改 UA 得用 `beforeParse(win){ Object.defineProperty(win.navigator,'userAgent',{...}) }`。
4. **写"应该消失"类的断言特别容易漏取反**（`assert(includes(...))` 本意是"不包含"）。写完回头看一眼。
5. **新增一套行样式时，`.done` 态的视觉要一并补齐。** `.item`（待办页/今天页）和 `.pday-item`（日历里）是两套 —— 曾经只改了 `.item`，导致日历里勾掉的待办方框不变色，看着像没勾上。
6. **内联 `node -e` 脚本超过约 120 行会撞 bash 引号解析错误**，改成写成 `.js` 文件再跑。

## 待办 / 下一步

- **同步还没启用**：需要用户自己生成一个只勾 `gist` 权限的 GitHub token，填进设置页（两台设备填同一个）。**token 不要经手 AI**。
- **第二期：WPS 云文档快照对比**（用户明确要的，已定"分期做"）
  - 用户用的是 **WPS 个人版**（金山文档），不是企业版
  - 需求：统一管理云文档 + 规整统计 + 每日总结他人更新 + **两次快照之间表格变动对比**
  - 已定路径：**表格快照对比在本地算**（不依赖 AI、不花钱、不外传数据）；"每日自动跑"做不到（静态页 + 边缘函数没有定时任务），只能做成"打开工作台时拉一次"
  - 风险：WPS 开放平台域名**没有返回跨域许可头**（实测过），浏览器不能直连，必须加服务端代理；个人版能否拿到接口权限尚未验证
- 用户偏好：**暗色 + 护眼**（不要纯黑底、不要纯白字）；沟通要**讲清技术限制和代价再让他选**，不要画饼。

## 换电脑后要补的东西

登录同一账号，**任务/对话记录会自动同步**；但这些在本地、不跟账号走：

- 技能包 `bys-personal-dashboard`（验收脚本在里面）
- 记忆文件（`~/.workbuddy/MEMORY.md` + 工程目录 `.workbuddy/memory/` 下的工作日志）
- 身份文件（`SOUL.md` / `IDENTITY.md` / `USER.md`）
- 连接器（金山文档等要重新授权）

工程代码本身不用搬 —— 就在这个仓库里，`git clone` 即可。
