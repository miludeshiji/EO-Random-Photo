# 🖼️ EdgeOne Random Photo API

[![Deploy to EdgeOne Pages](https://img.shields.io/badge/Deploy%20to-EdgeOne%20Pages-blue?style=for-the-badge&logo=tencent-cloud)](https://edgeone.ai/)

[简体中文](./README.md) | [English](./README_EN.md)

A high-performance, secure, and customizable Random Photo API built on **Tencent Cloud EdgeOne Pages**.

✨ **Key Features**:
*   **🚀 Edge Computing**: Powered by Edge Functions for millisecond-level response times.
*   **⚡ Zero-Overhead Config**: Implementation of **Global Variable Caching** (for Env mode) and **Context Sharing**, ensuring configuration is parsed only once per isolate/request.
*   **📱 Adaptive Design**: Automatically serves vertical or horizontal wallpapers based on the user's device.
*   **🔒 Smart Hotlink Protection**:
    *   **Global Protection**: Protects both the API and static image resources.
    *   **Whitelist**: Only authorized domains can access your resources.
    *   **Public Images**: Designate specific images as "Public" for universal access.
*   **🖼️ High-Fidelity Compression**: Built-in FFmpeg script for generating ultra-low size, high-quality WebP images (`-q 75 -m 6`).
*   **🛡️ DDoS Defense Mode**: Enables micro-caching on CDN nodes to withstand high-concurrency attacks (10k+ QPS).
*   **⚙️ Hybrid Configuration**:
    *   **KV Mode**: Real-time configuration updates via a visual admin panel.
    *   **Env Mode**: Read-only configuration via Environment Variables (for users without KV).
*   **🎨 Aurora Admin Panel**: Built with Aurora UI + Glassmorphism design language, featuring **CN/EN i18n** support.

---

## 🛠️ Quick Deployment

### 1. Prepare Images
Upload your wallpapers to the repository:
*   Vertical images: `public/images/vertical/`
*   Horizontal images: `public/images/horizontal/`

### 2. Optimize & Generate Manifest
Before committing, run the scripts to optimize images and generate the index:
```bash
# 1. Optimize images (High quality WebP, significantly reduces size)
node scripts/optimize-images.js

# 2. Update the image manifest
npm run generate:manifest
```
> **Note**: Ensure `functions/data/manifest.json` is included in your Git commit.

### 3. Deploy to EdgeOne Pages
1.  Go to [EdgeOne Pages Console](https://console.cloud.tencent.com/edgeone/pages).
2.  Create a new project and connect your Git repository.
3.  **Build Settings**:
    *   Build Command: `npm run generate:manifest` (or `npm run build`)
    *   Output Directory: `public`
4.  Click **Deploy**.

---

## ⚙️ Configuration

You can configure the project using **Environment Variables** (ReadOnly) or **KV Storage** (Read/Write). KV is preferred. If KV is missing, it falls back to Env Vars.

### Method A: Environment Variables (Read-Only)
Add these variables in EdgeOne Pages settings:

| Variable | Type | Example | Description |
| :--- | :--- | :--- | :--- |
| `ADMIN_PASSWORD` | String | `mypassword` | **[Required]** Admin panel password |
| `EO_PUBLIC_ACCESS` | Boolean | `false` | Allow global public access (Default: false) |
| `EO_WHITELIST` | String | `example.com,blog.me` | Allowed domains (comma separated) |
| `EO_DDOS_MODE` | Boolean | `false` | Enable DDoS Defense Mode |
| `EO_CACHE_TIMEOUT` | Number | `5` | Cache duration in seconds for DDoS Mode |
| `EO_PUBLIC_IMAGES` | String | `banner.jpg,logo.png` | **Public Images List**. These files bypass hotlink protection. |

### Method B: KV Storage (Read/Write)
1.  Create a KV Namespace (any name, e.g., `Random`) in the console.
2.  Bind it to `EO_KV` in Pages Settings -> **Functions Binding**.
3.  **First-time setup**: Access `https://your-domain/admin/` and login with `ADMIN_PASSWORD` from Env Vars.
4.  Save any configuration to activate KV mode.

> **🔐 Hybrid Password**: The `ADMIN_PASSWORD` environment variable serves as a **permanent fallback**. Even if you set a new password in KV, the original Env password still works. This ensures account recovery.

---

## 🔌 API Usage

### 1. Get Random Image
**Endpoint**: `/random`

| Param | Description |
| :--- | :--- |
| `type` | (Optional) `h`=Horizontal, `v`=Vertical. Auto-detects if omitted. |
| `redirect` | (Optional) `true`=Returns 302 Redirect; Omitted or `false`=Returns Image Content directly (Proxy Mode). |

```html
<!-- Direct Image Return (Recommended, URL stays /random) -->
<img src="https://api.your-site.com/random" />
<img src="https://api.your-site.com/random?type=v" />

<!-- 302 Redirect to actual path (Legacy behavior) -->
<img src="https://api.your-site.com/random?redirect=true" />
```

### 2. Health Check (for UptimeKuma)
**Endpoint**: `/health`

Returns service health status with actual validation of image availability.

| Status Code | Meaning |
| :--- | :--- |
| `200` | All checks passed |
| `503` | Service unhealthy |

**Response Example**:
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

**UptimeKuma Configuration**:
- **Monitor Type**: HTTP(s)
- **URL**: `https://your-domain/health`
- **Expected Status**: `200`
- **Keyword**: `"status":"ok"`

### 3. Access Specific Public Image
If `banner.jpg` is in your `EO_PUBLIC_IMAGES` whitelist:

```html
<img src="https://api.your-site.com/images/horizontal/banner.jpg" />
```
*Note: Images not in the whitelist will return 403 Forbidden if accessed directly from an unauthorized domain.*

---

## 🛡️ Security Policies

### Hotlink Protection
Intercepts requests from non-whitelisted domains.
*   **Exception 1**: `EO_PUBLIC_ACCESS=true` (Global Public).
*   **Exception 2**: The requested file is in `EO_PUBLIC_IMAGES`.

### DDoS Defense Mode
When enabled:
1.  **Micro-Caching**: API responses include `s-maxage=5`, causing the CDN to cache the redirect. All users see the same image for 5 seconds, reducing origin load.
2.  **Strict Check**: Requests must have a valid Referer (unless it is a Public Image request).

---

## 📂 Project Structure
```
├── functions/
│   ├── _middleware.ts    # Global Access Control & CORS
│   ├── random.ts         # Random Image Logic
│   ├── health.ts         # Health Check Endpoint
│   ├── env.d.ts          # TypeScript Type Definitions
│   ├── api/
│   │   └── admin.ts      # Admin API
│   ├── data/
│   │   └── manifest.json # Image Index (auto-generated)
│   └── utils/
│       └── config.ts     # Config Loader (KV/Env)
├── public/
│   ├── admin/            # Aurora UI Admin Dashboard
│   └── images/           # Image Assets
├── scripts/              # Build Scripts
└── package.json
```

---
Powered by Tencent Cloud EdgeOne Pages
