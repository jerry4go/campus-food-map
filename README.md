# 校园美食地图助手 🍜

基于高德地图 API 的校园周边美食发现与路线规划工具。单页应用，支持 IP/浏览器定位、关键词搜索餐厅、距离/评分/智能推荐排序、步行路线规划，并适配移动端。

## ✨ 功能特性

- **定位**：浏览器精确定位（HTTPS）→ 公网 IP 定位 → 默认位置三级降级；定位偏差时可在信息窗把某家店铺"设为我的位置"重新校准
- **美食搜索**：按关键词（川菜/烧烤/奶茶…）搜索周边 3km 餐厅，展示名称、距离、人均价格、评分
- **排序**：距离最近 / 评分最高 / 智能推荐（距离+评分+价格综合打分）
- **筛选**：价格档次（平价 <30 / 中档 30–80 / 高档 >80）、口味类型
- **推荐理由**：依据 `food-recommend-skill` 规则自动生成（如"人均 35 元，适合宿舍聚餐，步行 10 分钟可达"）
- **路线规划**：步行路线，展示距离、用时、分步指引，地图绘制蓝色路线
- **地图**：高德 JS API 底图，红色餐厅标记，点击弹信息窗，列表项内联展开路线
- **降级**：API 请求失败时使用 Mock 数据，保证页面可用

## 📁 目录结构

```
.
├── index.html      # 页面源码（本地可读版，不含 Key）
├── worker.js       # Cloudflare Worker 单文件版（HTML 内联 + REST 代理）← 部署这个
└── README.md
```

> `worker.js` 由 `index.html` 转义内联生成。改页面后请改 `index.html` 再重新生成 `worker.js`。

## 🏗 架构

```
浏览器 ──/api/place /api/regeo /api/walking …──> Cloudflare Worker ──Key(Secret)──> 高德 REST API
   │                                                   ↑ env.AMAP_KEY（前端不可见）
   └──JS API 脚本（前端 Key + 域名白名单）──> 高德 JS API（底图）
```

- 前端**不硬编码任何 Key**
- 所有 REST 请求经 Worker `/api/*` 代理，Key 在服务端注入
- JS API Key 由 `/api/config` 运行时下发（JS API 必须在浏览器执行）

## 🚀 部署到 Cloudflare Worker

1. Cloudflare Dashboard → **Workers & Pages** → **Create** → **Worker**
2. 编辑代码，粘贴 `worker.js` 全部内容 → **Deploy**
3. **Settings → Variables and Secrets** 添加变量（见下表）
4. 访问 `https://<worker名>.<子域>.workers.dev`
5. 高德控制台把 JS API Key 的**域名白名单**加上 `*.workers.dev`（或你的自定义域名）

### 环境变量

| 变量名 | 说明 | 类型 | 必填 |
|--------|------|------|:---:|
| `AMAP_KEY` | 高德「**Web服务**」Key，用于 REST 代理 | **Secret** | ✅ |
| `AMAP_JS_KEY` | 高德「**Web端(JS API)**」Key，前端底图 | Variable | ✅ |
| `AMAP_SECURITY_CODE` | 高德**安全密钥**（数字签名开启时才需要） | Variable | ⬜ |

## 🔑 高德 Key 说明（双 Key 方案）

高德 Key 按平台区分，安全机制不同：

| Key 类型 | 用途 | 是否暴露 | 安全手段 |
|---------|------|:---:|---------|
| Web端(JS API) | 地图渲染/标记/路线 | 是（必然） | **域名白名单** |
| Web服务 | 地理编码/POI/路线等 REST | 否 | **Worker 代理 + Secret** |

- 控制台创建 Key 时，服务平台选 **Web端(JS API)** 才会出现"域名白名单"
- 数字签名关闭时可不配 `securityJsCode`；开启则必须配
- Cloudflare Worker 出口 IP 不固定，Web服务 Key 的 IP 白名单**留空**，靠 Secret 保护

## 🧩 Worker 代理路由

| 前端路径 | 高德接口 |
|---------|---------|
| `/api/ip` | `/v3/ip` |
| `/api/regeo` | `/v3/geocode/regeo` |
| `/api/geo` | `/v3/geocode/geo` |
| `/api/place` | `/v3/place/text` |
| `/api/walking` | `/v3/direction/walking` |
| `/api/transit` | `/v3/direction/transit/integrated` |
| `/api/config` | 下发 `jsKey` / `securityCode` |

## 🛠 本地开发

`index.html` 依赖 `/api/*`，需 Worker 环境才能完整运行。两种方式：

1. 部署到 Cloudflare Worker 后调试（推荐）
2. 本地用 `wrangler dev` 运行 Worker

```bash
npm install -g wrangler
wrangler dev
```

## ⚠️ 注意事项

- 请勿把任何 Key 写进 `index.html` 或提交到仓库
- 若 Key 曾明文暴露，请到高德控制台**重置/新建**
- 定位精度：浏览器 geolocation（米级，需 HTTPS）> IP 定位（城市级）；偏差时用"设为我的位置"手动校准