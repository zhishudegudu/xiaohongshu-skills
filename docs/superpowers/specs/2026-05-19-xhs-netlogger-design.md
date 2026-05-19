# XHS NetLogger 设计文档

**日期**：2026-05-19
**分支**：feat/extension-bridge
**目标**：在浏览器扩展中加入小红书网页版网络监听能力，反向推导 XHS 用于检测自动化的"维度"，让插件 / Claude 操作时能规避风控。

参考实现：`D:\OSS\BHP_Production\BHP\modules\net-logger.js`（BOSS 直聘网页版的 webRequest 全量监听器）。

---

## 目标与非目标

### 目标

- 在 xiaohongshu.com 业务域 + 风控上报域采集足够信息，识别 XHS 服务端用来判定自动化的信号（cookie / header / 请求体指纹 / 响应错误码 / Set-Cookie 变化）。
- 数据只在扩展 popup 内消费，不暴露 CLI。
- 隐藏入口：默认对普通使用零感知，通过彩蛋（标题连点 5 次）激活。

### 非目标

- 不做 CLI 命令暴露 `get-netlog / export-netlog`（用户明确不需要）。
- 不做跨会话 diff、两会话对比（方案 C，本期延后）。
- 不做风控域名的自定义配置 UI（本期 hardcode）。
- 不做自动化测试（浏览器扩展 + 真实 XHS 域难离线复现）。
- 不替换现有 404 诊断（interceptor.js 现有逻辑）或 302 观测（background.js webRequest 现有逻辑），netlogger 是新增的独立 listener。

---

## 选定方案

**方案 B：webRequest 全量 + fetch/XHR hook 融合。**

- `chrome.webRequest` 4 阶段监听拿外层 HTTP 信号 + 请求体（指纹上报的关键载荷）
- `interceptor.js` 在 xiaohongshu.com 域 MAIN world hook fetch/XHR 拿响应体 + 应用层 header
- 按 `(method, url, 2s 时间窗)` 模糊关联两侧记录，合并为一条
- popup 面板「时序流」 + 「检测维度」双 tab

排除方案 A（webRequest only，看不到响应体与签名细节）与方案 C（B + 会话对比，工作量太大且 ROI 不确定，先把 B 做扎实）。

---

## 组件结构

| 文件 | 状态 | 职责 |
|---|---|---|
| `extension/netlogger.js` | 新建 | webRequest 4 阶段监听 + 环形缓冲 + storage 持久化 |
| `extension/interceptor.js` | 扩充 | 现有 404 诊断保留；新增"启用 netlog 时"记录全量 fetch/XHR 响应体 + 应用层 header |
| `extension/background.js` | 扩充 | importScripts netlogger.js；接 interceptor 上来的响应体信号；按 url+ts 关联 |
| `extension/popup.html` / `popup.js` | 扩充 | NetLog 卡片（彩蛋激活后显示）+ 时序流 / 检测维度 双 tab |
| `extension/manifest.json` | 扩充 | host_permissions 加风控上报域 |

## 数据流

```
用户请求
  │
  ├─→ chrome.webRequest (netlogger.js, background)
  │     ① onBeforeRequest        → method/url/reqBody bytes
  │     ② onSendHeaders          → 最终请求头（含 cookie/xs/xt/sec-fetch-*)
  │     ③ onHeadersReceived      → status/respHeaders/set-cookie
  │     ④ onCompleted/Error      → 耗时、错误码
  │
  └─→ interceptor.js (MAIN world, 仅 xiaohongshu.com)
        fetch/XHR hook → 响应体 + 应用层 header
        postMessage → content.js → background → netlogger
        ↓ 按 (method+url+时间窗) 模糊关联

netlogger 环形缓冲 (500 条)
  │
  ├─→ 每 10 条 / 关键事件触发写入
  │   chrome.storage.local.netLog
  │
  └─→ popup 拉取 + chrome.runtime push 实时增量
```

---

## 关键设计点

### A. 启用开关 / 彩蛋激活

- `chrome.storage.local.netlogEnabled`（boolean）作为持久化开关
- popup 顶部标题 "XHS Bridge" 连点 5 次切换；500ms 内无点击则计数器重置
- 激活后下方多出一张 "NetLog" 卡片
- 未启用时 netlogger / interceptor 的全部监听器 early return，零开销
- 已激活时再连点 5 次触发确认对话框，确认后关闭

### B. interceptor ↔ netlogger 关联

- 业务域内 interceptor 拿到响应后，按 `(method, url, 最近 2s 内的 webRequest 记录)` 关联回填响应体
- 关联失败的 interceptor 记录独立存入，标记 `_orphan: true`
- 跨域上报（fp / sentry / aegis 等）只走 webRequest 路径，不要求关联响应体

### C. 检测维度自动分类（启发式）

| category | 判定条件 |
|---|---|
| `fingerprint_upload` | host 含 fp / sec / aegis / sentry / track 关键字 OR 请求体含 webdriver / navigator / screen / timezone 字段 |
| `business_error` | status ∈ {401, 403, 461, 999} OR (HTTP 200 + 前端渲染 404) |
| `risk_redirect` | 302 → /404 或 /login |
| `signature_failure` | 302 Location 含 error_code=300031 / 300032 / 300033 |
| `cookie_change` | Set-Cookie 中新键 / 值变更（与上一条同 host 记录比较） |
| `business_api` | 路径含 /api/ 且 status=2xx |
| `page_nav` | resourceType=main_frame |
| `other` | 其他 |

一条 entry 可命中多个 signal（写入 `signals: string[]`），但 `category` 只取主分类（按上表自上而下的优先级）。

### D. 性能与容量

- 环形缓冲 500 条上限
- 单条请求体截断 2KB / 响应体截断 4KB
- 静态资源（image / font / stylesheet）按 `webRequest details.type` 过滤，不入栈
- 启用状态下预估额外开销：~5-10% CPU 在请求峰值时
- chrome.storage.local 限额 5MB，本设计总量预估 < 3MB

---

## 数据 Schema

```typescript
interface NetLogEntry {
  // ── 标识 ──
  id: string;                  // `${ts}_${requestId}`
  requestId: string;           // webRequest requestId
  ts: number;                  // onBeforeRequest 时间戳 (ms)
  tsLabel: string;             // "HH:mm:ss.SSS"

  // ── HTTP 基础 ──
  method: string;
  url: string;
  path: string;                // pathname + 关键 query (xsec_token=…&xsec_source=…)
  host: string;
  resourceType: string;        // main_frame / xhr / fetch / sub_frame
  tabId: number;

  // ── 请求（webRequest + interceptor 融合）──
  reqHeaders: Record<string, string>;     // 仅保留关键白名单
  reqBody: string | null;                 // 截断 2KB；raw bytes 解码或 formData JSON
  reqFingerprint: {
    has_xs: boolean;
    has_xt: boolean;
    has_xsCommon: boolean;
    sec_fetch_site: string | null;
    sec_fetch_mode: string | null;
    referer: string | null;
    origin: string | null;
    ua_prefix: string;                     // user-agent 前 80 字符
    cookie: {
      has_a1: boolean;
      has_web_session: boolean;
      has_webId: boolean;
      has_gid: boolean;
      a1_preview: string | null;
      web_session_preview: string | null;
    };
  };

  // ── 响应 ──
  status: number;
  statusLine: string;          // "200 OK" / "302 Found" / …
  respHeaders: Record<string, string>;
  respBody: string | null;     // 截断 4KB；仅业务域（interceptor 能拿到）
  setCookie: string[] | null;  // 解析后的 cookie 名列表

  // ── 时序 ──
  duration_ms: number;
  err: string | null;

  // ── 分析层（netlogger 计算）──
  category: NetLogCategory;
  signals: string[];           // 例: ["fingerprint_upload", "set_cookie_changed:sec_xxx"]
  cookieDiff: {
    added: string[];
    changed: string[];
    removed: string[];
  } | null;
  redirectTo: string | null;   // 302 Location
  errorCode: string | null;    // 300031/300032/… 从 location query 解析

  _orphan?: true;              // interceptor 未关联到 webRequest 时标记
}

type NetLogCategory =
  | "fingerprint_upload"
  | "business_error"
  | "risk_redirect"
  | "signature_failure"
  | "cookie_change"
  | "business_api"
  | "page_nav"
  | "other";
```

### 关键 cookie 白名单

请求 cookie 字段中以下 key 会被记录到 `reqFingerprint.cookie`：

- `a1`（设备指纹种子）
- `web_session`（登录态）
- `webId`
- `gid`
- 其他不在白名单的 cookie 名记入 `signals` 但值不存

### 关键 header 白名单（reqHeaders）

```
xs, xt, x-s-common, x-t, x-mns-platform,
sec-fetch-site, sec-fetch-mode, sec-fetch-dest, sec-fetch-user,
referer, origin, user-agent,
content-type, accept, accept-language
```

### 响应 header 白名单（respHeaders）

```
location, set-cookie, cache-control, x-request-id,
content-type, server, x-application-context
```

---

## popup NetLog 面板（草图）

```
┌─ XHS Bridge ───────────────────────┐
│ [logo] XHS Bridge       ● connected│  ← 标题连点 5 次激活
├────────────────────────────────────┤
│ 已连接到 bridge server             │
│ 当前账号：xxx                       │
│ [打开小红书]  [退出登录]            │
├────────────────────────────────────┤
│ ▼ NetLog  [enabled] [clear] [⬇json]│  ← 激活后显示
│ ┌──────────────────────────────┐   │
│ │[ 时序流 ] [ 检测维度 ]        │   │
│ ├──────────────────────────────┤   │
│ │ 时序流：                       │   │
│ │ 18:42:11.245 GET /api/sns/web/v1/feed  200 142ms [API]            │
│ │ 18:42:11.512 POST fp.snssdk.com/v1/fp  200 88ms  [指纹↑] ★       │
│ │ 18:42:12.103 GET /explore/abc?xsec_…   302 41ms  [风控跳转] ★    │
│ │ 18:42:12.180 GET /404?error_code=…     200 120ms [API]            │
│ │ …                                                                  │
│ ├──────────────────────────────┤   │
│ │ 检测维度（点击展开详情）：     │   │
│ │ ▾ 指纹上报 (12)                │   │
│ │   fp.snssdk.com/v1/fp  ×8 [body 含 webdriver/screen/…]           │
│ │   sec.xiaohongshu.com  ×4                                         │
│ │ ▾ 签名失败 (3)                 │   │
│ │   error_code=300033  token 与 session/IP 不匹配                  │
│ │ ▾ Cookie 变化 (5)              │   │
│ │   sec_xxx 新增（响应来自 /api/.../init）→ 风控触发标记           │
│ │ ▾ 业务错误 (2) / 风控跳转 (1)  │   │
│ └──────────────────────────────┘   │
└────────────────────────────────────┘
```

- 行点击展开看完整 entry（折叠 JSON 视图）
- `★` 标记需要重点关注的（指纹上报 / 风控跳转）
- "⬇json" 按钮把当前缓冲全量导出为 JSON 文件下载

---

## 错误处理与边缘情况

| 场景 | 处理 |
|---|---|
| webRequest 拿不到 reqBody（POST + 二进制） | `reqBody: "[binary]"`，只记字节数 |
| interceptor 跨标签页污染 | content.js 校验 sender.tab.id 与 background 记录的 tabId 匹配 |
| popup 关闭再开时数据丢失 | 数据存在 chrome.storage.local，popup 重新拉即可 |
| storage 写满（5MB 限制） | 环形缓冲 500 条 + 单条体积上限保证总量 < 3MB |
| 用户切换账号（a1 变化） | 触发自动 clear，避免跨账号污染分析 |
| 风控域名扩展 | host_permissions hardcode；本期不做配置 UI |
| service worker 重启丢内存缓冲 | webRequest listener 在 SW 重启后重新注册；内存丢失但 storage 持久部分保留 |

---

## 实现顺序与工作量估算

| # | 任务 | 改动文件 | 估算 | 验收 |
|---|---|---|---|---|
| 1 | 加风控域名到 host_permissions + 调研真实上报域名清单 | `manifest.json` | ~30 LOC | 装载扩展无错；调研结论补充到本文档 |
| 2 | 创建 netlogger.js：4 阶段 webRequest 监听 + 环形缓冲 + storage 持久化 + 启用开关 | 新建 `extension/netlogger.js` (~300 LOC) | ~3h | 启用后能在 storage 里看到 entries；关闭后零监听器活跃 |
| 3 | background.js 引入 netlogger + 暴露查询/清空消息接口 | `background.js` (~50 LOC) | ~30min | popup 能通过 runtime.sendMessage 拉到 log |
| 4 | interceptor.js 扩充：启用 netlog 时记录全量 fetch/XHR 响应体 + postMessage 上报 | `interceptor.js` (~80 LOC) | ~1.5h | 业务域内能拿到响应体并关联到 netlogger entry |
| 5 | netlogger 关联 + 分类逻辑：interceptor 信号回填 + category/signals/cookieDiff 计算 | `netlogger.js` (~150 LOC) | ~2h | 每条 entry 有正确 category；cookieDiff 能算出 Set-Cookie 变化 |
| 6 | popup UI：彩蛋激活 + NetLog 卡片 + 时序流 tab | `popup.html / popup.js` (~250 LOC) | ~3h | 标题连点 5 次激活；时序流实时刷新；点击行展开详情 |
| 7 | popup UI：检测维度 tab + 折叠分组 + json 导出按钮 | `popup.html / popup.js` (~150 LOC) | ~2h | 5 个分类正确归类；导出 JSON 文件可下载 |
| 8 | 手工冒烟测试：浏览/搜索/故意触发风控/切账号 | — | ~1h | 全场景跑通；分类无误判 |

**估算总量**：约 1000 LOC + 约 13h 集中工作时间。

---

## 主要风险与未知点

1. **风控上报域名清单不确定** — 目前对 XHS 实际使用的指纹上报域名只有部分猜测（fp.snssdk.com 等字节系常见域，但 XHS 是否用同套不确定）。任务 1 包含调研：先用宽松 host_permissions 临时监听一阵，看实际有哪些跨域 POST，再固化清单。
2. **interceptor ↔ webRequest 关联失败率** — 时间窗匹配可能漏关联（特别是高并发短连接）。先用 2s 窗 + url 完全匹配，观察漏匹配率，必要时改用 `Performance.getEntriesByName` 拿浏览器侧 timing 辅助匹配。
3. **chrome.storage.local 跨会话恢复** — Manifest V3 SW 在用户关闭浏览器时会被销毁；storage 持久但内存缓冲来不及写入的 N 条会丢。可接受。
4. **xs / xt 签名内部值无法拿到** — 只能拿到 fetch init.headers 里显式设置的；XHS SDK 若在更深层（如 Service Worker 拦截后注入）注入 header，fetch hook 也拿不到，需 chrome.debugger（本期不做）。
5. **彩蛋激活的 UX** — 5 次点击前无任何视觉反馈，新装扩展用户完全不知道这功能存在 —— 这是设计目标，不算风险。

---

## 测试方案

1. **手工冒烟**：
   - 激活 NetLog → 打开 xiaohongshu.com → 浏览/搜索 → 检查时序流是否完整、检测维度分类是否合理
   - 故意不带 xsec_token 直接访问 /explore/xxx → 验证抓到 `signature_failure` / `risk_redirect` 分类
   - 切账号 → 验证自动 clear 触发
2. **回归**：现有 `cmd_diagnose_404` / `cmd_check_risk` 不受影响（netlogger 是独立 listener，不替换原有逻辑）
3. **性能**：开 / 不开 netlog 在 xhs 首页滚动 30s，对比 SW CPU 占用（人工观察 chrome://serviceworker-internals）
4. 不写自动化测试

---

## 交付物

- 修改后的 `extension/{netlogger.js, background.js, interceptor.js, popup.html, popup.js, manifest.json}`
- 本设计文档
- 后续 brainstorming 出来的 implementation plan（spec 批准后由 writing-plans 产出）
