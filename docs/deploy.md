# 部署到 GitHub Pages

> 返回 [README](../README.md) · 相关：[隐私与安全](./security.md)

本仓库本身就是可直接托管的静态站点（根目录 `index.html` + `.nojekyll`），没有构建步骤：

1. 仓库 → **Settings** → **Pages**
2. **Source** 选 **Deploy from a branch**，分支 `main`，目录 `/ (root)`，保存
3. 约一分钟后访问 `https://<用户名>.github.io/<仓库名>/`；之后每次 `git push` 自动重新发布

> 本仓库的账号已给 Pages 绑定自定义域名 `bunnychen.top`（项目站点沿用它，形如 `https://bunnychen.top/<仓库名>/`），因此 `https://lizhenghe-chen.github.io/deepseek-balance-dashboard/` 会 301 跳到 `https://bunnychen.top/deepseek-balance-dashboard/`，两者等效。

`.nojekyll` 让 Pages 跳过 Jekyll 处理，文件按原样发布（本目录下的 `.md` 也会被原样托管）。

浏览器能否直连 DeepSeek：已实测该接口对本站点实际来源（`https://bunnychen.top`）的预检请求回显 `Access-Control-Allow-Origin` 并允许 `Authorization` 头，因此 Pages 版可以直接调用，无需任何代理或后端。

## 公开托管时需要注意的三点

- **站点是公开的，但数据不是**。任何人打开这个网址看到的都是同一份前端代码，各自拥有属于自己的浏览器本地数据；仓库和页面里没有任何密钥。
- **同账号的 Pages 站点共享一个 origin**。站点实际来源是 `https://bunnychen.top`（或未绑自定义域名时的 `https://<用户名>.github.io`），这是一个**账号级** origin：`localStorage` / `sessionStorage` 只按 origin 隔离，不按仓库隔离，因此本站与同账号的其它 Pages 项目（若本站与博客等共用同一域名，也包括博客）共享同一份浏览器存储：
  - 余额历史（`ds_history_v3`）**任何同源页面、任何标签页**都能读到（内容只有余额快照与 Key 指纹，不含 Key），也能被清空或篡改；
  - API Key 存在 `sessionStorage`（仅当前标签页），但如果你在**同一标签页**里跳转到该 origin 下的其他页面，对方在这次会话中可以看到它；
  - 同源页面还可以为整个 origin 注册 Service Worker。**因此同源站点的安全性等于其中 weakest 的那一个**。
  - 在意的话：给本仓库的 Pages 单独配置一个子域名（Settings → Pages → Custom domain，例如 `dash.bunnychen.top`），它就拥有独立 origin，与博客等页面彻底隔离；也可以把 `index.html` 下载到本地双击打开（`file://` 是独立来源，任何网站都读不到它的存储）。
- **必须走 HTTPS**。Pages 默认提供。CSP 的作用是**收窄脚本能主动外发的通道**：`connect-src` 只允许 `api.deepseek.com`，且没有 `eval` / `new Function`。但它不是「被注入也安全」的保证，具体边界见 [隐私与安全](./security.md)。输入框用 `autocomplete="new-password"` 降低被密码管理器收录的概率。

> 也可以部署到 Cloudflare Pages / Netlify / Vercel 等任意静态托管，行为一致。
