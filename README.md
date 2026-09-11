# 🖼️ EdgeOne Random Photo API

[![Deploy to EdgeOne Pages](https://img.shields.io/badge/Deploy%20to-EdgeOne%20Pages-blue?style=for-the-badge&logo=tencent-cloud)](https://edgeone.ai/)

[简体中文](./README.md) | [English](./README_EN.md)

一个基于 **腾讯云 EdgeOne Pages** 构建的高性能、安全、可定制的随机图片 API。

✨ **核心特性**：
*   **🚀 边缘计算**：基于 Edge Functions，毫秒级响应。
*   **⚡ 零开销配置**：实现 **全局变量缓存** (Env 模式) 与 **Context 共享**，确保每个实例/请求仅解析一次配置，极致压榨边缘性能。
*   **📱 自适应设备**：自动根据访问设备（手机/电脑）返回横屏或竖屏壁纸。
*   **🔒 智能防盗链**：
    *   **全局拦截**：不仅保护 API，也保护图片静态资源。
    *   **白名单机制**：仅允许授权域名调用。
    *   **特定公开**：支持将特定图片设为"公开"，由任何站点引用。
*   **🖼️ 高保真压缩**：内置 FFmpeg 脚本支持生成超低体积、高画质的 WebP 图片 (`-q 75 -m 6`)。
*   **🛡️ DDoS 防御模式**：开启后启用 CDN 微缓存，单节点抗击万级 QPS 攻击。
*   **⚙️ 双模配置**：
    *   **KV 模式**：通过管理面板**实时热更新**配置。
    *   **Env 模式**：通过环境变量配置（只读，适合无 KV 用户）。
*   **� Aurora 管理面板**：采用 Aurora UI + Glassmorphism 设计语言，支持 **中/英文切换**，自动保存语言偏好。

---

## 🛠️ 快速部署

### 1. 准备图片
将您的壁纸上传到仓库：
*   竖屏图片：`public/images/vertical/`
*   横屏图片：`public/images/horizontal/`

### 2. 优化图片与生成清单
提交代码前，运行脚本优化图片并生成索引：
```bash
# 1. 深度优化图片体积 (高画质 WebP)
node scripts/optimize-images.js

# 2. 更新图片清单
npm run generate:manifest
```
> Git 提交注意事项：请确保将生成的 `functions/data/manifest.json` 一并提交。

### 3. EdgeOne Pages 部署
1.  进入 [EdgeOne Pages 控制台](https://console.cloud.tencent.com/edgeone/pages)。
2.  新建项目，连接 Git 仓库。
3.  **构建设置**：
    *   Build Command: `npm run generate:manifest` (或 `npm run build`)
    *   Output Directory: `public`
4.  点击部署。

---

## ⚙️ 配置说明

您可以通过 **环境变量 (推荐新手)** 或 **KV 存储 (推荐进阶)** 进行配置。优先读取 KV，若未配置则读取环境变量。

### 方法 A: 环境变量 (只读模式)
在 EdgeOne Pages 项目设置中添加以下变量：

| 变量名 | 类型 | 示例值 | 说明 |
| :--- | :--- | :--- | :--- |
| `ADMIN_PASSWORD` | String | `mypassword` | **[必填]** 管理后台登录密码 |
| `EO_PUBLIC_ACCESS` | Boolean | `false` | 是否允许全站公开（建议关闭） |
| `EO_WHITELIST` | String | `example.com,blog.me` | 防盗链白名单（逗号分隔） |
| `EO_DDOS_MODE` | Boolean | `false` | 是否开启 DDoS 防御模式 |
| `EO_CACHE_TIMEOUT` | Number | `5` | DDoS 模式下的缓存时间（秒） |
| `EO_PUBLIC_IMAGES` | String | `banner.jpg,logo.png` | **公开图片列表**，这些图片允许任何域名引用 |

### 方法 B: KV 存储 (读写可控模式)
1.  在控制台创建 KV 命名空间（名称任意，如 `Random`）。
2.  在 Pages 设置 -> **函数绑定** 中，将变量名 `EO_KV` 绑定到该命名空间。
3.  **首次初始化**：访问管理后台 `https://您的域名/admin/`，使用环境变量 `ADMIN_PASSWORD` 登录。
4.  在后台保存任意配置后，KV 即自动生效。

> **🔐 混合密码机制**：环境变量 `ADMIN_PASSWORD` 作为**永久兜底密码**，即使 KV 中配置了新密码，原密码仍可登录。这确保您在忘记 KV 密码时仍能恢复访问。

---

## 🔌 API 使用指南

### 1. 获取随机图片
**Endpoint**: `/random`

| 参数 | 说明 |
| :--- | :--- |
| `type` | (可选) `h`=横屏, `v`=竖屏。不传则根据 User-Agent 自动判断。 |
| `redirect` | (可选) `true`=返回 302 重定向到图片地址；不传或 `false`=直接返回图片内容 (Proxy 模式)。 |

**示例**：
```html
<!-- 直接返回图片内容 (推荐，浏览器地址栏不改变) -->
<img src="https://api.your-site.com/random" />
<img src="https://api.your-site.com/random?type=v" />

<!-- 302 重定向到实际地址 (旧版行为) -->
<img src="https://api.your-site.com/random?redirect=true" />
```

### 2. 健康检查端点 (用于 UptimeKuma 监控)
**Endpoint**: `/health`

返回服务健康状态，实际验证图片可用性，使用 HEAD 请求不消耗流量。

| 状态码 | 含义 |
| :--- | :--- |
| `200` | 所有检查通过，服务正常 |
| `503` | 服务异常 |

**响应示例**：
```json
{
  "status": "ok",
  "timestamp": 1706438400000,
  "responseTime": "45ms",
  "checks": {
    "manifest": true,
    "randomApi": true,
    "imageAccess": true
  }
}
```

**UptimeKuma 配置**：
- **监控类型**：HTTP(s)
- **URL**：`https://您的域名/health`
- **预期状态码**：`200`
- **关键词**：`"status":"ok"`

### 3. 引用特定图片 (公开图片)
如果您在配置中将 `banner.jpg` 加入了 `EO_PUBLIC_IMAGES` 白名单，则可以直连：

```html
<img src="https://api.your-site.com/images/horizontal/banner.jpg" />
```
*注意：未加入白名单的图片，直接访问会被 403 拦截。*

---

## 🛡️ 安全策略详解

### 防盗链 (Referer Check)
系统会拦截所有非白名单域名的请求。
*   例外 1：配置了 `EO_PUBLIC_ACCESS=true` (全站公开)。
*   例外 2：请求的图片在 `EO_PUBLIC_IMAGES` 列表中。

### DDoS 防御模式
开启后：
1.  **强制缓存**：API 响应包含 `s-maxage=5`，CDN 节点直接缓存跳转结果。这意味着 5 秒内所有用户会看到同一张图片，但能极大降低源站负载。
2.  **严格检查**：强制要求请求携带 Referer（除非是公开图片），拦截脚本攻击。

---

## 📂 项目结构
```
├── functions/
│   ├── _middleware.ts    # 全局权限控制 (防盗链/CORS)
│   ├── random.ts         # 随机图片逻辑
│   ├── health.ts         # 健康检查端点
│   ├── env.d.ts          # TypeScript 类型定义
│   ├── api/
│   │   └── admin.ts      # 管理后台 API
│   ├── data/
│   │   └── manifest.json # 图片索引 (自动生成)
│   └── utils/
│       └── config.ts     # 双模配置读取器 (KV/Env)
├── public/
│   ├── admin/            # Aurora UI 管理后台
│   └── images/           # 图片仓库
├── scripts/              # 构建脚本
└── package.json
```

---
Powered by Tencent Cloud EdgeOne Pages
