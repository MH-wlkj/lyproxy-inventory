# LyProxy 代理库存 — GitHub Pages 部署说明

单文件 `index.html`，零构建、零依赖（Tailwind 走 CDN），直接丢进仓库开 Pages 就能用。

## 一、部署（3 分钟）

```bash
mkdir lyproxy-inventory && cd lyproxy-inventory
# 把 index.html 放进来
git init
git add index.html README.md
git commit -m "feat: lyproxy inventory dashboard"
git branch -M main
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git push -u origin main
```

然后仓库 → **Settings → Pages**：
- Source 选 `Deploy from a branch`
- Branch 选 `main`，目录选 `/ (root)`
- Save，等 1–2 分钟

访问 `https://<用户名>.github.io/<仓库名>/` 即可。

## 二、登录方式

页面打开后会先弹登录面板，两种方式任选：

### 方式 A：账号密码登录
填账号、密码、验证码。验证码是算术题，看图算结果填进去。

> ⚠️ 这个站的验证码有个坑：**每个 uuid 只接受第一次提交**，填错一次就必须点图片刷新重来。如果一直提示「验证码错误」，刷新验证码再试。

### 方式 B：直接粘贴 Token（推荐）

浏览器登录后台后：

1. F12 → **Application**（应用）→ **Local Storage** → `https://ly-proxy.com`
2. 找 key 为 `Admin-Token`（或 `token` / `access_token`）的项，复制 Value
3. 粘贴到页面的「直接粘贴 Token」输入框

Token 会存进浏览器 `localStorage`，下次打开自动登录，不用重复粘。

## 三、功能

| 功能 | 说明 |
|---|---|
| 实时读取 | 拉 `/admin/ip/merchant/list` 全量分页（`pageSize=1000`），一次拿完 |
| 自动刷新 | 默认 60 秒一次，标签页隐藏时暂停 |
| 统计卡片 | 总数 / 在线 / 离线 / 国家数 / 30 天内到期 / 同步时间 |
| 筛选 | 搜索（IP / 账号 / 地区 / 端口）、国家、品类、状态、来源（本地线 / IPNux 线） |
| 一键复制 | 每行「复制」按钮，输出 `ip:port:account:password` 格式 |
| 导出 CSV | 导出当前筛选结果，UTF-8-BOM，Excel 直接打开 |

## 四、改配置

打开 `index.html`，搜 `const CONFIG`：

```js
const CONFIG = {
  apiBase: 'https://ly-proxy.com/api/proxy',  // 接口地址
  pageSize: 100,          // 前端每页行数
  fetchPageSize: 1000,    // 后端单次拉取上限（不要再大，会超时）
  autoRefreshMs: 60000,   // 自动刷新间隔，改成 0 关闭
  storageKey: 'lyproxy_token',
};
```

## 五、跨域（CORS）— 已实测通过，无需代理

我直接对线上接口做了 CORS 预检，`ly-proxy.com` 的网关使用的是**反射式 Origin 回显**，任何来源都会被放行：

```
OPTIONS /api/proxy/login
Origin: https://example.github.io
Access-Control-Request-Method: POST

← HTTP/1.1 200
   Access-Control-Allow-Origin: https://example.github.io   ← 回显我们的 Origin
   Access-Control-Allow-Methods: POST
   Access-Control-Allow-Headers: content-type, authorization
   Access-Control-Allow-Credentials: true
   Access-Control-Max-Age: 1800
```

`GET /captchaImage` 的预检同样通过。**所以 GitHub Pages 直连即可，下面三种方案用不上，留着备用。**

### 备用方案 1：Cloudflare Worker 反向代理

新建 Worker，贴这段：

```js
export default {
  async fetch(request) {
    const url = new URL(request.url);
    const target = 'https://ly-proxy.com/api/proxy' + url.pathname + url.search;

    const cors = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET,POST,OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type,Authorization',
    };
    if (request.method === 'OPTIONS') return new Response(null, { headers: cors });

    const resp = await fetch(target, {
      method: request.method,
      headers: {
        'Content-Type': request.headers.get('Content-Type') || 'application/json',
        'Authorization': request.headers.get('Authorization') || '',
        'User-Agent': 'Mozilla/5.0',
      },
      body: request.method === 'GET' ? undefined : request.body,
    });

    const newResp = new Response(resp.body, resp);
    Object.entries(cors).forEach(([k, v]) => newResp.headers.set(k, v));
    return newResp;
  }
};
```

部署后把 Worker 域名填进页面的「API 地址」框，例如 `https://lyproxy.你的子域.workers.dev`。

### 备用方案 2：本地跑

```bash
python -m http.server 8080
# 浏览器开 http://localhost:8080
```

`localhost` 也可能被拦，配合解法 3 用。

### 备用方案 3：浏览器禁用跨域检查（仅本地调试）

```bash
# Windows
"C:\Program Files\Google\Chrome\Application\chrome.exe" --disable-web-security --user-data-dir=/tmp/chrome-test

# macOS
open -n -a "Google Chrome" --args --disable-web-security --user-data-dir=/tmp/chrome-test
```

## 六、安全提醒

- **Token 等同账号全部权限**，存 `localStorage` 意味着任何能在这个域名下执行 JS 的东西都能读走它。GitHub Pages 是静态托管，不会注入脚本，但别把页面挂到来源不明的镜像站。
- **别把 Token 写进代码提交到仓库**。页面设计成运行时输入，就是为了避免这个。
- 用完后在后台点退出，或清掉 `localStorage`。
