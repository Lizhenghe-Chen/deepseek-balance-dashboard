# 隐私与安全

> 返回 [README](../README.md) · 相关：[部署到 GitHub Pages](./deploy.md) · [数据模型](./data-model.md)

页面没有后端、没有统计、没有第三方分析；所有数据只写进你自己浏览器的存储。

| 问题 | 答案 |
|---|---|
| Key 会存到哪里？ | 默认：仅内存 + 当前标签页的 `sessionStorage`（不写 `localStorage`）。注意：`sessionStorage` **不是纯内存**——Chromium / Firefox 为支持「会话恢复」会把它写盘，浏览器崩溃恢复或「继续上次会话」时 Key 可能被还原；正常关闭标签页后其它页面读不到。可选：设置里勾选「记住 Key」后，明文写入本机 `localStorage`（`ds_api_key_saved`），新标签页 / 重开浏览器都能直接刷新；取消勾选或点「清除 Key」会立刻删除这份明文 |
| Key 会被上传吗？ | 只会发给 DeepSeek 官方余额接口 `https://api.deepseek.com/user/balance`（`Authorization: Bearer`），不发往任何第三方；页面没有后端、没有统计 |
| 设置面板里能看到明文 Key 吗？ | 不会。面板只显示 `sk-••••••1234` 这样的掩码与指纹，输入框永远为空，关闭面板即清空 |
| 有第三方脚本吗？ | 只有 Chart.js（jsDelivr CDN，带 SRI 完整性校验，哈希已核对）。页面会因此向 jsDelivr 发起一次请求（对端可见 IP / UA，`referrer` 已设为 `no-referrer`） |
| 历史数据在哪？ | 浏览器 `localStorage` 的 `ds_history_v3` 键（多账户，每份含余额快照与 Key 指纹，不含 Key）；字段见 [数据模型](./data-model.md) |
| 导入的 CSV 会被上传吗？ | 不会。文件只在本地用 `File.text()` 解析，仅保存文件名、账户 `user_id`、Key 名称、日期、模型 / 类型 / 数量 / 价格等用量信息到 `ds_usage_v3`；`api_key` 列整列跳过，不读取也不落盘 |
| CSP 能防住 XSS 吗？ | **不能当成保证**。它挡住了 `fetch` / XHR / WebSocket / beacon / 表单 / 图片外发，只放行 `api.deepseek.com`；但 `script-src` 含 `'unsafe-inline'`（页面用内联 `onclick`），`connect-src` **不管顶层跳转**（`location='…?k='` 能带走 Key），且 CSP 写在 `<meta>` 里、托管方不下发 CSP 响应头——能改页面的人可以连 CSP 一起删掉。真正的防线是「页面里没有任何 HTML 注入点」（全部走 `textContent`） |
| 公开托管会泄露 Key 吗？ | 托管方拿到的是同一份公开代码，看不到你在浏览器里输入的内容；但**同账号的其他 Pages 站点与本站同源**，它们（或浏览器扩展、DevTools）在特定条件下能读到 `sessionStorage`。要高保障：用独立子域，或把 HTML 下到本地用 `file://` 打开。详见 [部署 · 公开托管时需要注意的三点](./deploy.md) |
| 账户之间怎么区分？ | 历史里保存的是 Key 的 **djb2 指纹**（用于区分不同账号），不是 Key 本身。指纹不可逆推回 Key |

## 一句话总结

- 你的 Key 只在你和 `api.deepseek.com` 之间流动；
- 资产是本机浏览器里的余额快照，不是可被利用的凭据；
- 唯一的系统性风险来自**同源**——想让风险降到最低，就给它一个独立子域，或本地 `file://` 打开。
