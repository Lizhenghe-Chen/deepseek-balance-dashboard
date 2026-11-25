# 原理与部署

> 返回 [README](../README.md) · 相关：[使用指南](./guide.md)

页面没有后端、没有构建步骤：所有数据只写进你自己的浏览器。本文说明**数据长什么样、Key 存在哪、怎么发布**。

- [数据模型](#-数据模型)
- [采样与保留策略](#-采样与保留策略)
- [隐私与安全](#-隐私与安全)
- [部署到 GitHub Pages](#-部署到-github-pages)

## 🗂 数据模型

所有数据都只存在**你自己的浏览器**里（`localStorage`），没有任何后端。

### 账户档案

```json
{
  "k": "892efcc3",
  "cur": "CNY",
  "lastPoll": 1792811247065,
  "s": [
    { "t": 1792809600000, "b": 561.77 },
    { "t": 1792811247065, "b": 560.52 }
  ]
}
```

| 字段 | 含义 |
|---|---|
| `k` | API Key 的 djb2 指纹（**不是账户 ID**，同账号换 Key 会得到不同指纹） |
| `name` | 你起的名字；为空时回退到 `userTag` 或指纹前 6 位 |
| `userTag` | 导入官方 CSV 时记录下来的 `user_id`，是唯一能跨设备识别的账户线索 |
| `cur` | 币种，一份档案只对应一种（CNY 或 USD） |
| `lastPoll` | 上次轮询时间，同时用于判断档案是否已过期 |
| `bg` | 是否「记住 Key 并在后台采样」：为真时该账户的明文写进 `ds_keys_saved`，且不显示时也继续采样 |
| `s` | 采样点数组：`t` 为毫秒时间戳，`b` 为余额。**每次请求都追加**；48 小时内保留完整采样（分钟级定位），更早的只保留余额变动的点 |

外层是 `{ "active": "指纹|币种", "accounts": { "指纹|币种": 档案 } }`。每条档案保留最近 60 天，最多 5 份；写满配额时先丢最老的那份。旧版 `ds_history_v2`（单档案）与它的备份槽会在首次打开时自动并成档案，一份都不会丢。

导出的 JSON 带上全部账户与官方用量；导入时按档案合并（同一天以时间戳并集为准）。导入旧版单档案 JSON 同样可用。

### 其它本地键

| 键 | 内容 |
|---|---|
| `ds_history_v3` | 全部账户档案（见上），多账户结构 |
| `ds_api_key` | 仅 `sessionStorage`，当前账户的明文 Key，关闭标签页失效 |
| `ds_keys` | 仅 `sessionStorage`，**本标签页用过的全部 Key**（`指纹 → Key`），支撑"直接切换不用重输" |
| `ds_keys_saved` | `localStorage`，只含**勾了「后台 / 记住」**的账户的明文 Key（`指纹 → Key`）；没有则为空 |
| `ds_history_backup` | v2 时代的一次性备份槽，仅用于迁移时并入档案，之后不再写入 |
| `ds_api_key_saved` | v2 时代的单个「记住 Key」槽，已由 Key 环接管；首次打开会并入并删除 |
| `ds_usage_v3` | 导入的官方用量：按**导入批次**保存文件名、账户 `user_id`、Key 名称、币种与原始用量行；**不含 `api_key` 列**。只保留最近约 13 个月的批次，写满时先丢最老的一次导入。旧版 `ds_usage_v2` / `ds_usage_v1` 会自动迁移成一个批次 |
| `ds_alert` | 低余额提醒的阈值与「已提醒」标记 |
| `ds_chart_view` | 趋势图当前视图（`balance` / `spend`） |

## ⏱ 采样与保留策略

| 项 | 规则 |
|---|---|
| 采样频率 | 当前账户：前台每 1 分钟、后台标签页每 5 分钟；勾了「后台」的其它账户：每 5 分钟 / 隐藏时 15 分钟。每次请求都记一条（余额没变也记） |
| 高精度窗口 | 最近 48 小时保留每次采样，可用于分钟级定位 |
| 更早的数据 | 只保留余额发生变动的点（体积约为原采样的 1/5～1/10） |
| 保留上限 | 每份档案 60 天；最多 5 个账户 |
| 配额不足时 | 先丢最老账户的一半采样，再不行丢掉整份最老账户 |

## 🔒 隐私与安全

账户数据仍由浏览器保存，页面不设账户后端。正式域名下每次文档加载/刷新向 Vercount 发送一次匿名 PV：只发送 `https://bunnychen.top` 加页面路径和 `isNewUv: false`，不读取账户存储，不发送 query、hash、凭证或 referrer；统计服务仍可见网络 IP 等连接信息。内部操作不计数，localhost 和 file 不计数。

| 问题 | 答案 |
|---|---|
| Key 会存到哪里？ | 两层「Key 环」：① `sessionStorage` 的 `ds_keys` 存**本标签页用过的全部 Key**（默认的唯一去处，关标签页即清）；② `localStorage` 的 `ds_keys_saved` **只存你勾了「后台 / 记住」的账户**，没勾的账户不会写盘。注意 `sessionStorage` **不是纯内存**——Chromium / Firefox 为支持「会话恢复」会把它写盘，崩溃恢复或「继续上次会话」时可能被还原。点「清除 Key」或关掉该账户的「后台」，对应的明文会立刻从存储中删除 |
| Key 会被上传吗？ | 只会发给 DeepSeek 官方余额接口 `https://api.deepseek.com/user/balance`（`Authorization: Bearer`），不发往任何第三方；计数脚本不读取账户状态 |
| 设置面板里能看到明文 Key 吗？ | 不会。面板只显示 `sk-••••••1234` 这样的掩码与指纹，输入框永远为空，关闭面板即清空 |
| 有第三方脚本吗？ | 只有 Chart.js（jsDelivr CDN，带 SRI 完整性校验，哈希已核对）。页面会因此向 jsDelivr 发起一次请求（对端可见 IP / UA，`referrer` 已设为 `no-referrer`） |
| 历史数据在哪？ | 浏览器 `localStorage` 的 `ds_history_v3` 键（多账户，每份含余额快照与 Key 指纹，**不含 Key**）；字段见上文[数据模型](#-数据模型) |
| 会同时把多个 Key 发出去吗？ | 会，但只发给你勾了「后台」的那些账户、以及当前正在显示的那一个，每次请求只带一个 Key 的 `Authorization`。请求之间错开 1.5 秒，不并发打同一个接口 |
| 导入的 CSV 会被上传吗？ | 不会。文件只在本地用 `File.text()` 解析，仅保存文件名、账户 `user_id`、Key 名称、日期、模型 / 类型 / 数量 / 价格等用量信息到 `ds_usage_v3`；`api_key` 列整列跳过，不读取也不落盘 |
| CSP 能防住 XSS 吗？ | **不能当成保证**。它挡住了 `fetch` / XHR / WebSocket / beacon / 表单 / 图片外发，仅允许 `api.deepseek.com` 与匿名 PV 端点 `events.vercount.one`；但 `script-src` 含 `'unsafe-inline'`（页面用内联 `onclick`），`connect-src` **不管顶层跳转**（`location='…?k='` 能带走 Key），且 CSP 写在 `<meta>` 里、托管方不下发 CSP 响应头——能改页面的人可以连 CSP 一起删掉。真正的防线是「页面里没有任何 HTML 注入点」（全部走 `textContent`） |
| 公开托管会泄露 Key 吗？ | 托管方拿到的是同一份公开代码，看不到你在浏览器里输入的内容；但**同账号的其他 Pages 站点与本站同源**，它们（或浏览器扩展、DevTools）在特定条件下能读到 `sessionStorage`。要高保障：用独立子域，或把 HTML 下到本地用 `file://` 打开。详见[部署 · 公开托管时需要注意的三点](#-部署到-github-pages) |
| 账户之间怎么区分？ | 历史里保存的是 Key 的 **djb2 指纹**（用于区分不同账号），不是 Key 本身。指纹不可逆推回 Key |

一句话总结：

- 你的 Key 只在你和 `api.deepseek.com` 之间流动；**勾了几个「后台」，磁盘上就有几个明文 Key**，其余只在内存里；
- 资产是本机浏览器里的余额快照，不是可被利用的凭据；
- 唯一的系统性风险来自**同源**——想让风险降到最低，就给它一个独立子域，或本地 `file://` 打开。

## 🚀 部署到 GitHub Pages

本仓库本身就是可直接托管的静态站点（根目录 `index.html` + `.nojekyll`），没有构建步骤：

1. 仓库 → **Settings** → **Pages**
2. **Source** 选 **Deploy from a branch**，分支 `main`，目录 `/ (root)`，保存
3. 约一分钟后访问 `https://<用户名>.github.io/<仓库名>/`；之后每次 `git push` 自动重新发布

> 本仓库的账号已给 Pages 绑定自定义域名 `bunnychen.top`（项目站点沿用它，形如 `https://bunnychen.top/<仓库名>/`），因此 `https://lizhenghe-chen.github.io/deepseek-balance-dashboard/` 会 301 跳到 `https://bunnychen.top/deepseek-balance-dashboard/`，两者等效。

`.nojekyll` 让 Pages 跳过 Jekyll 处理，文件按原样发布（`docs/` 下的 `.md` 也会被原样托管）。

浏览器能否直连 DeepSeek：已实测该接口对本站点实际来源（`https://bunnychen.top`）的预检请求回显 `Access-Control-Allow-Origin` 并允许 `Authorization` 头，因此 Pages 版可以直接调用，无需任何代理或后端。

### 公开托管时需要注意的三点

- **站点是公开的，但数据不是**。任何人打开这个网址看到的都是同一份前端代码，各自拥有属于自己的浏览器本地数据；仓库和页面里没有任何密钥。
- **同账号的 Pages 站点共享一个 origin**。站点实际来源是 `https://bunnychen.top`（或未绑自定义域名时的 `https://<用户名>.github.io`），这是一个**账号级** origin：`localStorage` / `sessionStorage` 只按 origin 隔离，不按仓库隔离，因此本站与同账号的其它 Pages 项目（若本站与博客等共用同一域名，也包括博客）共享同一份浏览器存储：
  - 余额历史（`ds_history_v3`）**任何同源页面、任何标签页**都能读到（内容只有余额快照与 Key 指纹，不含 Key），也能被清空或篡改；
  - API Key 存在 `sessionStorage`（仅当前标签页），但如果你在**同一标签页**里跳转到该 origin 下的其他页面，对方在这次会话中可以看到它；
  - 同源页面还可以为整个 origin 注册 Service Worker。**因此同源站点的安全性等于其中 weakest 的那一个**。
  - 在意的话：给本仓库的 Pages 单独配置一个子域名（Settings → Pages → Custom domain，例如 `dash.bunnychen.top`），它就拥有独立 origin，与博客等页面彻底隔离；也可以把 `index.html` 下载到本地双击打开（`file://` 是独立来源，任何网站都读不到它的存储）。
- **必须走 HTTPS**。Pages 默认提供。CSP 的作用是**收窄脚本能主动外发的通道**：`connect-src` 仅允许 `api.deepseek.com` 和匿名 PV 端点 `events.vercount.one`，且没有 `eval` / `new Function`。但它不是「被注入也安全」的保证，具体边界见上文[隐私与安全](#-隐私与安全)。输入框用 `autocomplete="new-password"` 降低被密码管理器收录的概率。

> 也可以部署到 Cloudflare Pages / Netlify / Vercel 等任意静态托管，行为一致。
