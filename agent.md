# EO-Random-Photo Agent Guide

## 项目概述

EO-Random-Photo 是一个基于 **腾讯云 EdgeOne Pages** 的高性能随机图片 API 服务。它利用边缘函数 (Edge Functions) 实现毫秒级响应，自动根据设备类型返回横屏/竖屏壁纸，并内置防盗链、DDoS 防御、管理面板等功能。

## 技术栈

- **运行时**: Cloudflare Workers / EdgeOne Pages Functions
- **语言**: TypeScript (Edge Functions) + JavaScript (Node.js 脚本)
- **图片处理**: FFmpeg (WebP 转换与压缩)
- **配置存储**: EdgeOne KV (读写) / 环境变量 (只读回退)
- **管理面板**: 纯静态 HTML (Aurora UI + Glassmorphism 设计)
- **模块系统**: ESM (`"type": "module"`)

## 项目结构

```
EO-Random-Photo/
├── functions/                  # EdgeOne Edge Functions (核心业务逻辑)
│   ├── _middleware.ts          # 全局中间件：CORS、防盗链、DDoS 防护
│   ├── random.ts               # 随机图片 API 主入口 (/random)
│   ├── health.ts               # 健康检查端点 (/health)
│   ├── env.d.ts                # TypeScript 类型声明 (Env, KV)
│   ├── api/
│   │   └── admin.ts            # 管理 API：配置读写 (/api/admin)
│   ├── data/
│   │   └── manifest.json       # 【自动生成】图片索引清单
│   └── utils/
│       └── config.ts           # 配置管理：KV/ENV 双模 + 全局缓存
├── public/                     # 静态资源 (部署输出目录)
│   ├── _headers                # CDN 静态 CORS 头配置
│   ├── admin/
│   │   └── index.html          # Aurora 管理面板 (中/英双语)
│   └── images/
│       ├── horizontal/         # 横屏图片 (h_001.webp, h_002.webp, ...)
│       └── vertical/           # 竖屏图片 (v_001.webp, v_002.webp, ...)
├── scripts/                    # 构建脚本
│   ├── optimize-images.js      # 图片优化：格式转换 + 重命名 + 压缩
│   └── generate-manifest.js    # 生成 manifest.json 图片索引
├── edgeone.json                # EdgeOne 部署配置 (CORS 头 + 缓存策略)
├── package.json                # 项目依赖与脚本
└── tsconfig.json               # TypeScript 配置
```

## 常用命令

```bash
# 图片优化（将新增图片转换为 WebP 并按序号重命名）
node scripts/optimize-images.js

# 生成图片索引清单
npm run generate:manifest

# 构建（等同 generate:manifest）
npm run build

# 本地开发
npm run dev
```

## 图片处理完整流程

这是新增图片后的标准操作流程，**必须严格按顺序执行**：

### 第一步：放置新图片

将新图片文件放入对应目录：
- 横屏壁纸 → `public/images/horizontal/`
- 竖屏壁纸 → `public/images/vertical/`

支持的输入格式：`.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`

### 第二步：运行图片优化脚本

```bash
node scripts/optimize-images.js
```

**脚本行为详解** (`scripts/optimize-images.js`)：

1. **扫描目录**：遍历 `public/images/vertical/` 和 `public/images/horizontal/`
2. **文件排序**：对目录内所有文件按文件名字母序排序
3. **序号分配**：按排序后的顺序，从 1 开始为每个文件分配序号
4. **命名规则**：
   - 竖屏：`v_001.webp`, `v_002.webp`, ...
   - 横屏：`h_001.webp`, `h_002.webp`, ...
5. **跳过已处理**：如果文件名已经是目标名称（如 `v_001.webp`），则跳过
6. **FFmpeg 转换**：对未处理的文件执行转换：
   ```
   ffmpeg -y -v error -i "原文件" -c:v libwebp -q:v 75 -compression_level 6 -preset photo -map_metadata -1 "目标.webp"
   ```
   - `-c:v libwebp`：WebP 编码器
   - `-q:v 75`：质量 75（Google 推荐最佳平衡点）
   - `-compression_level 6`：最高压缩等级（最慢但文件最小）
   - `-preset photo`：照片优化预设
   - `-map_metadata -1`：移除所有 EXIF 元数据
7. **清理原文件**：转换成功后删除原始文件（如 `.jpg`, `.png`）

**前置依赖**：系统需安装 FFmpeg 并加入 PATH。

### 第三步：生成 manifest 索引

```bash
npm run generate:manifest
```

**脚本行为详解** (`scripts/generate-manifest.js`)：

1. 扫描 `public/images/vertical/` 和 `public/images/horizontal/` 两个目录
2. 过滤出图片文件（`.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`, `.avif`）
3. 生成 `functions/data/manifest.json`，结构如下：
   ```json
   {
     "vertical": ["/images/vertical/v_001.webp", ...],
     "horizontal": ["/images/horizontal/h_001.webp", ...]
   }
   ```

### 第四步：提交代码

**必须提交的文件**：
- 新增/变更的 `public/images/` 下的 `.webp` 文件
- 更新后的 `functions/data/manifest.json`

## 核心业务逻辑

### 请求处理链

```
客户端请求
    ↓
_middleware.ts (全局中间件)
    ├── OPTIONS → 返回 CORS 预检响应
    ├── X-Internal-Request: true → 放行（内部代理请求）
    ├── /admin/* → 直接放行
    ├── /images/* → 放行 + 注入 CORS 头
    └── 其他请求 → 防盗链检查 → 放行或拒绝(403)
    ↓
random.ts / health.ts / api/admin.ts (具体处理函数)
```

### random.ts — 随机图片 API (`/random`)

核心流程：
1. **加载配置**：从中间件的 `context.data` 获取，或重新调用 `getConfig(env)`
2. **防盗链**：检查 Referer 是否在白名单中（同域名自动允许）
3. **设备适配**：
   - `?type=v` → 竖屏，`?type=h` → 横屏
   - 未指定时通过 User-Agent 正则自动判断移动设备
4. **随机选图**：从 `manifest.json` 对应数组中随机取一张
5. **响应策略**：
   - `?redirect=true` → 302 重定向到图片 URL
   - 默认 → 代理模式，Edge Function 直接 fetch 图片并返回内容（URL 不变）
6. **DDoS 模式**：开启时添加 `Cache-Control: s-maxage` 启用边缘缓存

### config.ts — 配置管理

采用 **KV 优先 + ENV 回退** 的双模架构：

```
getConfig(env) 调用链：
    1. 检查全局变量缓存 (globalConfigCache) → 命中则直接返回（仅 ENV 模式）
    2. 检查 KV 是否可用 (EO_KV 全局变量)
       ├── KV 有数据 → 解析返回，附带 fallbackPassword
       └── KV 绑定但空 → 从 ENV 读取，标记 kvBound=true
    3. 无 KV → 从环境变量读取，存入全局缓存
```

**Config 接口**：
```typescript
interface Config {
    publicAccess: boolean;      // 是否公开访问
    whitelist: string[];        // 防盗链白名单域名
    adminPassword?: string;     // 管理密码
    fallbackPassword?: string;  // 环境变量兜底密码
    source: 'KV' | 'ENV';      // 配置来源
    kvBound: boolean;           // KV 是否已绑定
    ddosMode: boolean;          // DDoS 防御模式
    ddosCacheTimeout: number;   // DDoS 缓存时间（秒）
    publicImages: string[];     // 公开图片列表（绕过防盗链）
}
```

### _middleware.ts — 全局中间件

处理所有进入 Edge Function 的请求：
- **CORS**：为 `/images/*` 静态资源和所有 API 响应注入跨域头
- **内部请求放行**：`X-Internal-Request: true` 头标记的请求直接放行，防止 `random.ts` 代理 fetch 时的死循环
- **防盗链**：检查 Referer 域名是否匹配白名单或同域
- **公开图片**：`publicImages` 列表中的文件不受防盗链限制
- **DDoS 模式**：无 Referer 且非公开访问时直接返回 403

### admin.ts — 管理 API (`/api/admin`)

- **GET**：返回当前配置（JSON）
- **POST**：保存配置到 KV（需要 `Authorization: Bearer <password>` 认证）
- **双密码机制**：KV 密码与环境变量 `ADMIN_PASSWORD` 均可认证，确保忘记 KV 密码时仍可登录

### health.ts — 健康检查 (`/health`)

用于 UptimeKuma 等监控接入：
1. 检查 manifest 是否有图片
2. 随机选取一张图片发 HEAD 请求验证可访问性
3. 返回 JSON 状态报告（包含响应时间、图片数量统计）

## 部署配置

### edgeone.json

```json
{
    "headers": [
        { "source": "/images/*", "headers": [
            { "key": "Access-Control-Allow-Origin", "value": "*" },
            { "key": "Access-Control-Allow-Methods", "value": "GET, HEAD, OPTIONS" }
        ]}
    ],
    "caches": [
        { "source": "/images/*", "cacheTtl": 3600 }
    ]
}
```

- `/images/*` 静态资源缓存 1 小时
- 静态 CORS 头确保重定向模式下图片可跨域加载

### 环境变量

| 变量名 | 必填 | 说明 |
|---|---|---|
| `ADMIN_PASSWORD` | 是 | 管理面板登录密码 |
| `EO_PUBLIC_ACCESS` | 否 | 是否公开（默认 `false`） |
| `EO_WHITELIST` | 否 | 白名单域名（逗号分隔） |
| `EO_DDOS_MODE` | 否 | DDoS 防御模式（默认 `false`） |
| `EO_CACHE_TIMEOUT` | 否 | DDoS 缓存秒数（默认 `5`） |
| `EO_PUBLIC_IMAGES` | 否 | 公开图片文件名（逗号分隔） |

### KV 绑定

在 EdgeOne 控制台将 KV 命名空间绑定到变量名 `EO_KV`，即可启用读写配置模式。

## 开发注意事项

- **manifest.json 是自动生成的**：不要手动编辑，始终通过脚本生成
- **图片命名规则**：脚本会自动处理，新图片放进目录即可，名称随意
- **FFmpeg 是必需的**：图片优化脚本依赖系统 FFmpeg，确保已安装
- **KV 通过全局变量访问**：EdgeOne 的 KV 通过 `declare const EO_KV` 全局变量访问，不是 `env.EO_KV`
- **图片总是 WebP 格式**：优化脚本统一输出 WebP，质量 75，压缩等级 6
- **代理模式是默认行为**：`/random` 默认直接返回图片内容，需要 `?redirect=true` 才走 302
- **修改配置后无需重新部署**：KV 模式下配置实时生效；ENV 模式需重新部署
