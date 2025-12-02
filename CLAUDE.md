# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**wechat-article-exporter** is an online tool for batch downloading WeChat Official Account (公众号) articles. It supports exporting articles with metadata (view counts, comments) in multiple formats (HTML, JSON, Excel, Markdown, TXT, Word) with 100% style preservation for HTML exports. The application is client-side rendered (SSR disabled) and supports Docker and Cloudflare deployment.

**Core Principle**: The tool leverages WeChat Official Account's backend article search functionality to retrieve all articles from a target account.

## Development Commands

```bash
# Development
yarn dev              # Start development server
yarn debug            # Start dev server with Node inspector

# Building
yarn build            # Build for production

# Docker
yarn docker:build     # Build Docker image (uses version from package.json)
yarn docker:publish   # Push Docker image to registry

# Preview (Cloudflare)
yarn preview          # Build for Cloudflare Pages and preview locally
```

**Node Version**: Requires Node.js >= 22 (specified in package.json engines)

**Package Manager**: Uses Yarn 1.22.22

## Architecture

### Technology Stack
- **Framework**: Nuxt 3 (SSR disabled, client-side only)
- **UI**: Nuxt UI with Tailwind CSS
- **Database**: Dexie.js (IndexedDB wrapper) for client-side storage
- **Grid**: AG Grid Enterprise for article tables
- **Runtime**: Nitro with configurable KV storage (memory, filesystem, or cloud)

### Key Architectural Patterns

#### 1. Server-Side Proxy Architecture
The backend proxies requests to WeChat's MP (公众号) platform to avoid CORS issues:

- **Cookie Management**: `server/utils/CookieStore.ts` manages session cookies server-side
- **Proxy Layer**: `server/utils/proxy-request.ts` forwards requests to `mp.weixin.qq.com`
- **Authentication Flow**:
  - QR code login generates `uuid` cookie (client-side)
  - Successful login creates `auth-key` cookie (HTTP-only, 4-day expiry)
  - `auth-key` maps to MP session token in server-side KV store
  - Subsequent requests use `auth-key` header or cookie

API routes in `server/api/`:
- `web/login/*` - QR code login flow
- `web/mp/*` - MP backend operations (search accounts, fetch articles)
- `web/misc/*` - Utilities (comments, albums, status checks)
- `public/v1/*` - Public API endpoints for external usage

#### 2. Client-Side Data Management
Uses Dexie.js with IndexedDB (database name: `exporter.wxdown.online`):

**Store Structure** (`store/v2/`):
- `article` - Article metadata cache
- `html` - Downloaded article HTML content
- `metadata` - View counts, likes, shares
- `comment` / `comment_reply` - Comment data
- `resource` / `resource-map` - Asset URLs and file mappings for offline HTML export
- `info` - Account information (indexed by `fakeid`)
- `api` - API call history for debugging

All stores support `fakeid` indexing for per-account data management.

#### 3. Download & Export System

**Download Pipeline** (`utils/download/`):
- `BaseDownload.ts` - Base class with event emitter pattern
- `Downloader.ts` - Handles three download types:
  - `html` - Article content
  - `metadata` - View counts (requires credentials, max concurrency: 2)
  - `comments` - Comment data (requires credentials, max concurrency: 2)
- Queue-based concurrent processing (configurable concurrency)
- Progress tracking via events: `download:begin`, `download:progress`, `download:finish`

**Export Pipeline** (`utils/download/Exporter.ts`):
- Supports 6 formats: Excel, JSON, HTML, TXT, Markdown, Word
- HTML export workflow:
  1. Extract resource URLs from cached HTML
  2. Download resources concurrently (images, styles, scripts)
  3. Replace URLs with local paths
  4. Write to File System Access API (browser-native directory selection)
- Uses event-driven progress: `export:begin`, `export:download`, `export:write`, `export:finish`

**Composables**:
- `useDownloader.ts` - Vue wrapper for Downloader with reactive progress
- `useExporter.ts` - Vue wrapper for Exporter with phase tracking
- `useBatchDownload.ts` - Batch operation orchestration

#### 4. Proxy Manager for Asset Downloads
`utils/download/ProxyManager.ts` manages a pool of public proxy servers for bypassing image referrer restrictions. Proxies rotate on failure to ensure reliable asset downloads.

#### 5. Credentials System
For fetching view counts and comments, the app requires WeChat credentials (captured via packet inspection). Stored in `localStorage` as `auto-detect-credentials:credentials`. Credentials expire after 25 minutes (`CREDENTIAL_LIVE_MINUTES` in `config/index.ts`).

### Component Organization

- `components/dashboard/` - Main dashboard UI (sidebar, nav, actions)
- `components/grid/` - AG Grid custom renderers and utilities
- `components/search/` - Account/article search forms
- `components/selector/` - Account selection for albums and articles
- `components/setting/` - User preference panels
- `components/modal/` - Login, confirm dialogs
- `components/preview/` - Article preview with shadow DOM renderer
- `components/global/` - Shared components (credentials dialog, search)

### Page Routes

- `/` - Landing page
- `/dashboard` - Main app (layout)
  - `/dashboard/account` - Manage logged-in accounts
  - `/dashboard/article` - Article grid and operations
  - `/dashboard/album` - Album-based article view
  - `/dashboard/single` - Single article download
  - `/dashboard/api` - Public API documentation
  - `/dashboard/proxy` - Proxy settings
  - `/dashboard/settings` - User preferences
  - `/dashboard/support` - Help/support info
- `/dev/*` - Development/testing pages

## Configuration & Environment

### Runtime Config (`nuxt.config.ts`)
- `NUXT_AGGRID_LICENSE` - AG Grid Enterprise license key
- `NUXT_SENTRY_DSN` / `NUXT_SENTRY_ORG` / `NUXT_SENTRY_PROJECT` / `NUXT_SENTRY_AUTH_TOKEN` - Sentry error tracking
- `NUXT_UMAMI_ID` / `NUXT_UMAMI_HOST` - Umami analytics
- `NITRO_KV_DRIVER` - KV storage driver (`memory`, `fs`, or cloud provider)
- `NITRO_KV_BASE` - KV storage base path (for `fs` driver)

### Key Constants (`config/index.ts`)
- `ARTICLE_LIST_PAGE_SIZE = 20` - Max articles per page (WeChat API limit)
- `ACCOUNT_LIST_PAGE_SIZE = 5` - Accounts per search result
- `PUBLIC_PROXY_LIST` - Public proxy servers for asset downloads
- `CREDENTIAL_LIVE_MINUTES = 25` - Credential expiration time
- `USER_AGENT` - Fixed UA string for MP requests

## Common Workflows

### Adding a New Download Type
1. Add type to `DownloadType` union in `Downloader.ts`
2. Implement processing logic in `processTask()` method
3. Add event handlers in composable `useDownloader.ts`
4. Wire up UI in relevant dashboard page component

### Adding a New Export Format
1. Add type to `ExportType` union in `Exporter.ts`
2. Implement export method (e.g., `exportPdfFiles()`)
3. Call from `startExport()` switch statement
4. Add handler in `useExporter.ts` composable
5. Add UI trigger in export settings

### Extending the API
1. Create new route file in `server/api/public/v1/` (follows Nuxt file-based routing)
2. Use `proxyMpRequest()` for MP backend calls
3. Add API definition to `apis` array in `config/index.ts` for auto-documentation
4. Test with public API page at `/dashboard/api`

## Important Notes

- **No SSR**: Application is fully client-side (`ssr: false` in nuxt.config)
- **Storage Limits**: IndexedDB quotas vary by browser; monitor usage with `StorageUsage.vue`
- **Concurrency Limits**: Metadata and comment downloads limited to 2 concurrent requests to avoid rate limiting
- **File System API**: HTML/TXT/Markdown/Word exports require browser support for File System Access API (Chrome/Edge 86+)
- **Telemetry**: Build-time telemetry sends config to `NUXT_TELEMETRY_URL` if `NUXT_TELEMETRY=true`

## Deployment

### Docker
The Dockerfile uses multi-stage builds:
1. Build stage: Installs deps and runs `yarn build` (generates `.output/`)
2. Runtime stage: Copies `.output/`, runs as non-root `node` user
3. Exposes port 3000, starts Nitro server with `node server/index.mjs`

Environment variables for Docker:
- `NODE_ENV=production`
- `NITRO_KV_DRIVER=fs` (recommended for Docker)
- `NITRO_KV_BASE=.data/kv`

### Cloudflare Pages
Use `yarn preview` to test Cloudflare build locally (uses `NITRO_PRESET=cloudflare_pages`).

## Debugging

- Set `debugMpRequest: true` in runtime config to log MP requests/responses (dev only)
- Sentry integration captures client-side errors in production
- API call history stored in `api` IndexedDB table for troubleshooting
- Use `/dev/` pages for isolated component testing
