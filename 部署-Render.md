# 部署到 Render · 操作手册

本文档针对 **nanchang-travel**（纯静态多页站 + PWA，无构建步骤）。按顺序做一遍即可上线；后面几节是日常更新与排错。

- 运行形态：Render **Static Site**
- 构建：不需要（没有 npm、没有打包器）
- 发布目录：仓库根（`.gitignore` 已排除 `tools/.cache/` 与 `assets/img/raw/`）
- 配置文件：`render.yaml`（服务名、缓存头、根路径 rewrite）

---

## 0. 部署前本地自检

```bash
cd nanchang-travel

# 1) JS 语法
node --check js/app.js && node --check js/layout.js && node --check js/data.js && node --check sw.js

# 2) 公共壳是否与生成器一致（不一致会导致 6 页顶栏/Tab 漂移）
python3 tools/build-shell.py --check

# 3) 本地跑一遍（必须 127.0.0.1，SW 才会注册）
python3 -m http.server 8901 --bind 127.0.0.1
# 打开 http://127.0.0.1:8901 逐页点一遍
```

确认三件事再推：

- 6 个页面都能打开，无控制台报错
- 顶部 Tab 高亮正确（food 页高亮「打卡」）
- 「我的 → 离线行程」能缓存成功，断网刷新仍可看

---

## 1. 推到 GitHub

### 情况 A：新建独立仓库（推荐）

```bash
cd nanchang-travel
git init
git add .
git commit -m "init: 赣鄱四日行程站"

# 在 GitHub 上先建一个空仓库，比如 nanchang-travel
git branch -M main
git remote add origin git@github.com:<你的用户名>/nanchang-travel.git
git push -u origin main
```

### 情况 B：放进已有仓库的子目录

如果仓库根是 `kaili/`，站点在 `nanchang-travel/`，创建服务时把 **Root Directory** 填 `nanchang-travel`，Publish Directory 保持 `.`。

> 不要提交 `tools/.cache/`（24MB 源字体）和 `assets/img/raw/`（27MB 生成原图），`.gitignore` 已处理；`git status` 里看到它们说明 `.gitignore` 没生效。

---

## 2. 创建 Render Static Site

1. 登录 <https://dashboard.render.com> → 右上 **New +** → **Static Site**
2. 连接 GitHub，选中刚推的仓库
3. 按下表填写：

| 字段 | 填什么 | 说明 |
| --- | --- | --- |
| Name | `nanchang` | **必须与 `render.yaml` 的 `name` 一致**，否则缓存头整段失效（见第 9 节） |
| Branch | `main` | 推送即部署 |
| Root Directory | 留空 / `.`（情况 B 填 `nanchang-travel`） | 站点文件所在目录 |
| Build Command | 留空，或 `echo "静态站点，无需构建"` | 不要写 `npm build`，没这东西 |
| Publish Directory | `.` | 相对 Root Directory |
| Auto-Deploy | `Yes` | 跟 main 走 |
| PR Previews | `No` | 私人行程站，不需要预览环境 |

4. 点 **Create Static Site**，等 1–2 分钟，拿到 `https://<name>.onrender.com`

> 首次部署若服务名不是 `nanchang`，要么改服务名，要么改 `render.yaml` 的 `name` 后重新同步。

---

## 3. 配置缓存头

**这一步不做也能上线，但会出现「改了页面、访客还是旧的」。** 两种方式二选一。

### 方式 A：控制台手动加（最稳）

Service → **Headers** → Add Header，逐条添加：

| Path | Name | Value |
| --- | --- | --- |
| `/sw.js` | `Cache-Control` | `no-cache` |
| `/assets/*` | `Cache-Control` | `public, max-age=300, stale-while-revalidate=604800` |
| `/css/*` | `Cache-Control` | `public, max-age=300, stale-while-revalidate=604800` |
| `/js/*` | `Cache-Control` | `public, max-age=300, stale-while-revalidate=604800` |
| `/*.html` | `Cache-Control` | `public, max-age=0, must-revalidate` |

### 方式 B：Blueprint（render.yaml）

前提：服务是通过 Blueprint 创建的，或已在 Render 的 Blueprint 页面手动 **Sync** 过一次。手动创建的服务不会读 `render.yaml` 的 headers。

### 为什么这样配

- **`sw.js` 必须 `no-cache`**：SW 脚本本身被缓存，更新就永远不生效。
- **HTML 必须协商**：`max-age=0, must-revalidate`，保证部署后立刻拿到新页面。
- **资源不用 `immutable`**：文件名没有内容哈希。原地覆盖字体或图片时，`immutable` 会让浏览器与边缘缓存最长一年返回旧字节；更麻烦的是 SW 的 `cache.add()` 也会命中旧字节，把旧文件写进新缓存，升版本号都救不回来。
- **`/assets/*` 的 `*` 能跨斜杠**（匹配 `/assets/img/x.webp`、`/assets/fonts/x.woff2`），但 `/*.html` **不能**匹配 `trip/index.html` 这类嵌套路径——本站页面都在根目录，够用。

---

## 4. 根路径与 404

`render.yaml` 里有一条 rewrite：

```yaml
routes:
  - type: rewrite
    source: /
    destination: /index.html
```

Render 静态站通常会自动把 `/` 指向 `index.html`；如果访问根路径 404，检查这条 rewrite 是否生效（控制台 → Redirects/Rewrites）。

---

## 5. 绑定自定义域名（可选）

1. Service → **Settings → Custom Domains** → Add Custom Domain
2. 按提示在域名商处加 CNAME（如 `travel.example.com → <name>.onrender.com`）
3. 等待 DNS 生效，Render 自动签发 TLS 证书
4. 把 `manifest.webmanifest` 的 `start_url`、页面里的绝对链接检查一遍（本站全用相对路径，无需改）

> PWA 的 Service Worker 只在 HTTPS 或 localhost 下注册。自定义域名必须等证书状态变成 **Certificate Issued** 再测离线。

---

## 6. 部署后验收

```bash
BASE=https://<name>.onrender.com

# 6 个页面都应是 200
for p in index explore trip pack me food; do
  printf "%-8s %s\n" "$p" "$(curl -s -o /dev/null -w '%{http_code}' $BASE/$p.html)"
done

# 关键缓存头
curl -sI $BASE/sw.js | grep -i cache-control          # 期望 no-cache
curl -sI $BASE/index.html | grep -i cache-control     # 期望 max-age=0, must-revalidate
curl -sI $BASE/assets/fonts/noto-serif-sc-subset.woff2 | grep -i cache-control
curl -sI $BASE/js/app.js | grep -i cache-control

# 资源是否真的存在
curl -s -o /dev/null -w '%{http_code}\n' $BASE/assets/img/tengwang-800.webp
curl -s -o /dev/null -w '%{http_code}\n' $BASE/assets/fonts/noto-serif-sc-subset.woff2
```

浏览器再手工确认：

- [ ] 首页 Banner、图片、字体正常（标题是衬线体，不是系统宋体）
- [ ] 点卡片弹出详情，地址栏出现 `#/p-tengwang`，返回键能关
- [ ] 搜索「滕王阁」「万象城」有结果
- [ ] 「我的 → 离线行程」提示已就绪，开飞行模式刷新仍能看
- [ ] DevTools → Application → Service Workers 状态 `activated`
- [ ] 手机上「添加到主屏幕」可用

---

## 7. 日常更新流程

内容改动（行程、美食、文案）：

```bash
# 1) 改内容（data.js / html / css）
# 2) 如果改了 css/js，必须同时升缓存版本
#    sw.js:      var CACHE = "nanchang-trip-v20";
#    js/app.js:  var CACHE_NAME = "nanchang-trip-v20";
# 3) 如果改了 h1/h2/h3 或 data.js 的 title，重跑字体子集
python3 tools/subset-font.py
# 4) 如果改了顶栏/Tab/热词，重跑公共壳
python3 tools/build-shell.py
# 5) 自检 + 提交
node --check js/app.js && python3 tools/build-shell.py --check
git add -A && git commit -m "update: ..." && git push
```

推上去后 Render 自动部署；访客下次打开会命中新 SW（`skipWaiting` + `clients.claim`），不需要手动清缓存。

### 版本号规则

- 任何 `css/`、`js/`、`html` 改动 → **两个文件同时 +1**（`sw.js` 的 `CACHE` 与 `app.js` 的 `CACHE_NAME`）
- 只改 `data.js`（不动标题）→ 也算 js 改动，同样升版本
- 只改图片、且文件名不变 → 升版本，否则老访客看到旧图
- 只改 README / 行程.md → 不用升，不影响站点

---

## 8. 回滚

Render → Service → **Deploys** → 找到上一个正常版本 → **Redeploy**。

注意：回滚只换服务端文件，**不会**回滚用户浏览器里的 SW 缓存。如果回滚涉及 css/js，仍要确保 `CACHE` 版本与回滚后的 `app.js` 一致，否则缓存会被清空一次（不会坏，只是要重新下载）。

---

## 9. 常见问题排查

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 访问域名 404 / 空白页 | Publish Directory 或 Root Directory 填错 | 站点文件在仓库根就是 `.`；在子目录就把 Root 指向子目录 |
| 缓存头全是 `public, max-age=0, s-maxage=300` | 服务名与 `render.yaml` 的 `name` 不一致，或服务没关联 Blueprint | 改服务名对齐，或在 Blueprint 页面 Sync；也可直接在控制台 Headers 手配 |
| 改了页面，手机还是旧的 | SW 缓存未升版本 | 同时升 `sw.js` 的 `CACHE` 与 `app.js` 的 `CACHE_NAME`，重新部署 |
| 标题字体变成系统宋体 | 改了标题但没重跑 `subset-font.py`，新字不在子集里 | 重跑脚本，注意升 SW 版本 |
| 字体/图片换了但没生效 | `/assets/*` 被缓存，或 SW 预缓存了旧字节 | 确认缓存头是 `max-age=300` 而非 `immutable`；升 SW 版本 |
| PWA 装不上 / 不注册 SW | 非 HTTPS，或用了 `file://` 打开 | 用 `https://` 或 `http://127.0.0.1` |
| 顶栏或底部 Tab 在 6 个页面不一致 | 手改了 HTML，没跑生成器 | `python3 tools/build-shell.py` 后重新部署 |
| 详情浮层打不开，控制台报 key 不存在 | `data-detail` 写了但 `data.js` 里没这个 key | 补数据，或改 `data-detail` |
| 部署日志显示 "no build command" | 正常，静态站不需要构建 | 忽略 |
| 修改 `render.yaml` 没反应 | 服务不是 Blueprint 创建的 | 控制台手动改，或重新用 Blueprint 建服务 |

---

## 10. 附：render.yaml 关键项

```yaml
services:
  - type: web
    name: nanchang              # 必须与 Render 服务名一致
    runtime: static
    buildCommand: echo "静态站点，无需构建"
    staticPublishPath: .         # 发布仓库根
    headers:                     # 扁平 path/name/value，别用嵌套写法
      - path: /sw.js
        name: Cache-Control
        value: no-cache
      - path: /assets/*
        name: Cache-Control
        value: public, max-age=300, stale-while-revalidate=604800
      - path: /css/*
        name: Cache-Control
        value: public, max-age=300, stale-while-revalidate=604800
      - path: /js/*
        name: Cache-Control
        value: public, max-age=300, stale-while-revalidate=604800
      - path: /*.html
        name: Cache-Control
        value: public, max-age=0, must-revalidate
    routes:
      - type: rewrite
        source: /
        destination: /index.html
```

Render 的 header 写法是**扁平**的 `path / name / value`；写成 Cloudflare/Netlify 那种 `pattern + headers[{key,value}]` 会被静默忽略，表现为「配置写了但完全没生效」。

---

## 11. 上线检查清单

- [ ] 本地 `build-shell.py --check` 通过
- [ ] 4 个 JS 文件 `node --check` 通过
- [ ] 仓库里没有 `tools/.cache/`、`assets/img/raw/`
- [ ] Render 服务名 = `render.yaml` 的 `name`
- [ ] 5 条缓存头都在控制台可见
- [ ] 6 个页面 200
- [ ] 字体、图片、图标正常
- [ ] 详情浮层 / 搜索 / 收藏 / 打包勾选可用
- [ ] 离线行程可缓存，断网能看
- [ ] 手机「添加到主屏幕」正常
