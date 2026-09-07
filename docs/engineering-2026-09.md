# 🛡️ NEWBEE 站点工程健康记录 (2026-09-07)

> 体检 → 修复 → 复检 闭环。工具链: codex-seo 武器库 (`analyze_*.py`)。

## 体检发现的真实问题 (2026-09-07)

| 项目 | 修复前 | 问题 |
|---|---|---|
| 安全头 (analyze_technical security) | **40/100** | CSP / HSTS / X-Frame-Options / X-Content-Type-Options / Referrer-Policy 全缺；X-Powered-By 指纹泄露 |
| 结构化数据 (analyze_schema) | **84/100** | 首页只有 WebSite；缺 WebPage、Organization(带 logo/sameAs)；文章页 publisher 无 url/logo |

**注**: analyze_content 报"首页 277 字"与 analyze_visual 报"无 H1"均为**误报** (实测首页 1877 字、有 H1)，本地分析器结论需人工交叉验证。

## 修复内容

### 1. 安全响应头 (nginx, `/etc/nginx/sites-enabled/newbee`)

5 项全加, 覆盖首页/文章页/工具页/静态资源 + www 301:

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' https://newbeebnb.cn https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self'; frame-ancestors 'self'
```

- `proxy_hide_header X-Powered-By` 去除 Express 指纹
- **CSP 注意**: 页面含内联 onclick → 必须 `unsafe-inline`; 工具页 `btc-dca-calculator.html` 引 Chart.js CDN → 必须放行 `https://cdn.jsdelivr.net`
- nginx add_header 继承规则: 子层(if 块)有 add_header 则父层不继承 → 安全头须加在 `location /` 内与 if 块同层

### 2. 结构化数据 (Schema.org)

**首页** `/var/www/html/index.html`: WebSite → + @graph(WebPage + Organization)

```json
{"@context":"https://schema.org","@graph":[
  {"@type":"WebPage","@id":".../#webpage","isPartOf":{"@id":".../#website"},"about":{"@id":".../#organization"}},
  {"@type":"Organization","@id":".../#organization","name":"NEWBEE交易社区","url":"https://newbeebnb.cn/","logo":{...},"sameAs":["https://space.bilibili.com/261535048","https://github.com/ShadowMiner/newbeebnb"]}
]}
```

**文章页** `/var/www/html/server/utils/ssrArticle.js`: Article 的 author/publisher 补全 `url` + `logo` (pm2 restart newbee-api 生效)

## 复检结果 (同工具复测)

| 项目 | 修复前 | 修复后 |
|---|---|---|
| analyze_schema | 84/100 | **100/100** (valid, Organization/WebPage/WebSite 全识别) |
| 安全头 | 全缺 | **5 头全生效**, 首页/文章页/301 全覆盖 |
| 真实浏览器冒烟 | — | 首页/文章/工具三类页面 0 资源错误, Chart.js 正常 |

## 性能基线 (PageSpeed Insights API 实测, 2026-09-07)

- **性能 91/100** · LCP 0.4s · CLS 0 · TBT 0ms (desktop)
- analyze_technical 86/100 的 CWV 启发式 (81) 为低估, 真实性能优秀

## 备份

- `/tmp/newbee.nginx.bak-20260907*` (nginx 配置)
- `/tmp/ssrArticle.js.bak-20260907` (文章页渲染)
- `/tmp/index-org-schema.bak-20260907` (首页 index.html)

---
*记录人: NEWBEE Bot · 方法: codex-seo 武器库体检 → nginx/Express 修复 → 真实浏览器复检*
