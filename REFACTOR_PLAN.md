# 微信公众号文章抓取系统重构方案

**版本**: 1.0.0
**日期**: 2026-01-18
**作者**: 架构设计团队

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 当前架构分析](#2-当前架构分析)
- [3. 重构目标](#3-重构目标)
- [4. 新架构设计](#4-新架构设计)
- [5. Monorepo 方案选择](#5-monorepo-方案选择)
- [6. 技术栈详细设计](#6-技术栈详细设计)
- [7. 数据库设计](#7-数据库设计)
- [8. API 接口设计](#8-api-接口设计)
- [9. 前端架构设计](#9-前端架构设计)
- [10. 迁移方案](#10-迁移方案)
- [11. 实施计划](#11-实施计划)
- [12. 风险评估与应对](#12-风险评估与应对)

---

## 1. 项目概述

### 1.1 项目背景

**wechat-article-exporter** 是一个用于批量下载微信公众号文章的在线工具。当前版本基于 Nuxt 3 框架，采用客户端渲染（SSR 禁用）+ Nitro 服务端代理的架构。

### 1.2 重构动机

- **架构升级**: 从 Nuxt 3 单体架构迁移到 React 前端 + NestJS 后端的分离架构
- **数据持久化**: 从浏览器 IndexedDB 迁移到 PostgreSQL 数据库，支持多用户和数据共享
- **可扩展性**: 采用 Monorepo 架构，便于代码复用和独立部署
- **企业级应用**: 支持用户系统、权限管理、API 限流等企业特性

### 1.3 核心功能

- 微信公众号二维码登录
- 公众号搜索和文章列表获取
- 批量下载文章（HTML、元数据、评论）
- 多格式导出（Excel、JSON、HTML、Markdown、TXT、Word）
- 资源代理下载和离线化

---

## 2. 当前架构分析

### 2.1 技术栈

```
前端层:
├─ Nuxt 3 (Vue 3 + SSR disabled)
├─ Nuxt UI + Tailwind CSS
├─ AG Grid Enterprise
└─ Dexie.js (IndexedDB 包装器)

服务端层:
├─ Nitro (Nuxt 服务端)
├─ Nitro KV Storage (内存/文件系统/Cloudflare)
└─ HTTP 代理转发到 mp.weixin.qq.com

客户端存储:
├─ IndexedDB (文章、HTML、元数据、评论、资源)
└─ localStorage (凭证、配置)
```

### 2.2 核心流程

**认证流程**:
```
客户端 → 获取QR码(UUID) → 扫码 → 提交登录 → 服务端生成auth-key
→ CookieStore存储(token+cookies) → 客户端保存auth-key cookie
```

**数据抓取流程**:
```
客户端 → 搜索公众号(auth-key) → 服务端代理 → 微信MP后台
→ 返回账号列表 → 客户端IndexedDB缓存

客户端 → 获取文章列表(fakeid+auth-key) → 服务端代理 → 微信MP
→ 返回文章列表 → 客户端IndexedDB缓存

客户端 → 下载HTML(URL列表) → 代理服务器 → 微信文章URL
→ 验证HTML → IndexedDB存储
```

**下载系统**:
```
Downloader类:
├─ downloadHTMLTask: 下载文章HTML内容
├─ downloadMetadataTask: 抓取阅读量/点赞数(需Credential)
└─ downloadCommentsTask: 抓取评论数据(需Credential)

Exporter类:
├─ extractResources: 从HTML提取图片/CSS URL
├─ downloadResourceTask: 并发下载资源
└─ exportHtmlFiles: 替换URL为本地路径，写入文件系统
```

### 2.3 架构优势

✅ **客户端渲染**: 减轻服务器压力
✅ **本地存储**: 数据完全掌控在用户手中
✅ **代理机制**: 解决CORS和反爬虫问题
✅ **智能缓存**: 避免重复下载
✅ **事件驱动**: 细粒度进度控制

### 2.4 架构局限

❌ **单用户限制**: IndexedDB无法跨设备共享数据
❌ **存储限制**: 浏览器存储配额限制（通常5-50GB）
❌ **无用户系统**: 无法实现多租户和权限管理
❌ **可扩展性差**: 单体架构难以独立扩展前后端
❌ **离线导出**: 依赖浏览器 File System Access API（仅Chrome/Edge支持）

---

## 3. 重构目标

### 3.1 架构目标

1. **前后端分离**: React SPA + NestJS RESTful API
2. **数据持久化**: PostgreSQL 替代 IndexedDB
3. **Monorepo 管理**: 统一管理前端、后端、共享库
4. **容器化部署**: Docker + Docker Compose
5. **企业级特性**: 用户系统、权限管理、API 限流

### 3.2 功能目标

1. **保留所有现有功能**: 登录、搜索、下载、导出
2. **多用户支持**: 用户注册、登录、会话管理
3. **数据共享**: 用户之间可共享公众号和文章数据
4. **后台任务**: 异步下载任务队列（BullMQ）
5. **API 开放**: 提供公开 API 供第三方集成

### 3.3 性能目标

- API 响应时间 < 200ms (P95)
- 文章下载速度 > 10篇/分钟
- 支持 100+ 并发用户
- 数据库查询时间 < 50ms

---

## 4. 新架构设计

### 4.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户层                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   浏览器      │  │   移动端      │  │   第三方API   │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼───────────────────┼────────────────┼─────────────────┘
          │                   │                 │
          └───────────────────┼─────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      前端层 (React SPA)                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Apps/Web (Vite + React 18 + TypeScript)                 │  │
│  │  ├─ React Router (路由)                                   │  │
│  │  ├─ Zustand (状态管理)                                     │  │
│  │  ├─ TanStack Query (数据获取)                             │  │
│  │  ├─ Ant Design (UI组件库)                                 │  │
│  │  ├─ AG Grid React (表格)                                  │  │
│  │  └─ Socket.io Client (实时通信)                           │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          │
          │ HTTP/WebSocket
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  API 网关层 (可选 - 未来扩展)                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Nginx / Traefik / Kong                                   │  │
│  │  ├─ 反向代理                                               │  │
│  │  ├─ 负载均衡                                               │  │
│  │  ├─ SSL 终止                                              │  │
│  │  └─ Rate Limiting                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                   后端层 (NestJS + TypeScript)                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Apps/API (NestJS 10)                                     │  │
│  │                                                            │  │
│  │  模块划分:                                                  │  │
│  │  ├─ AuthModule (认证: JWT + Passport)                     │  │
│  │  ├─ UserModule (用户管理)                                  │  │
│  │  ├─ WechatModule (微信登录/代理)                           │  │
│  │  ├─ AccountModule (公众号管理)                             │  │
│  │  ├─ ArticleModule (文章管理)                               │  │
│  │  ├─ DownloadModule (下载任务)                              │  │
│  │  ├─ ExportModule (导出任务)                                │  │
│  │  ├─ ProxyModule (代理管理)                                 │  │
│  │  └─ TaskModule (后台任务队列)                              │  │
│  │                                                            │  │
│  │  横切关注点:                                                │  │
│  │  ├─ Guards (权限守卫)                                      │  │
│  │  ├─ Interceptors (日志/缓存)                              │  │
│  │  ├─ Filters (异常处理)                                     │  │
│  │  └─ Pipes (数据验证)                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          │
          ├─────────────┬─────────────┬─────────────┬────────────┐
          ▼             ▼             ▼             ▼            ▼
┌────────────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  PostgreSQL    │ │  Redis  │ │  BullMQ  │ │  S3/OSS  │ │ 微信MP API │
│  (主数据库)     │ │ (缓存)  │ │ (队列)   │ │ (存储)   │ │ (外部)    │
├────────────────┤ ├─────────┤ ├──────────┤ ├──────────┤ ├──────────┤
│ MikroORM       │ │ Session │ │ 下载任务  │ │ HTML文件 │ │ 代理转发  │
│ 实体映射        │ │ 缓存    │ │ 导出任务  │ │ 资源文件 │ │ Cookie管理│
│ 迁移管理        │ │ 限流    │ │ 定时任务  │ │ 导出文件 │ │          │
└────────────────┘ └─────────┘ └──────────┘ └──────────┘ └──────────┘
```

### 4.2 Monorepo 结构

```
wechat-article-exporter/
├── packages/
│   ├── shared/                    # 共享代码库
│   │   ├── types/                 # TypeScript 类型定义
│   │   │   ├── entities.ts        # 数据实体类型
│   │   │   ├── api.ts             # API 接口类型
│   │   │   └── constants.ts       # 常量定义
│   │   ├── utils/                 # 工具函数
│   │   │   ├── validation.ts      # 数据验证
│   │   │   ├── formatters.ts      # 格式化工具
│   │   │   └── helpers.ts         # 辅助函数
│   │   └── package.json
│   │
│   ├── config/                    # 配置包
│   │   ├── eslint-config/         # ESLint 配置
│   │   ├── typescript-config/     # TypeScript 配置
│   │   └── prettier-config/       # Prettier 配置
│   │
│   └── sdk/                       # 客户端 SDK (可选)
│       ├── src/
│       │   ├── client.ts          # API 客户端
│       │   └── hooks.ts           # React Hooks
│       └── package.json
│
├── apps/
│   ├── web/                       # React 前端应用
│   │   ├── public/
│   │   ├── src/
│   │   │   ├── components/        # React 组件
│   │   │   ├── pages/             # 页面组件
│   │   │   ├── hooks/             # 自定义 Hooks
│   │   │   ├── stores/            # Zustand 状态管理
│   │   │   ├── services/          # API 服务层
│   │   │   ├── utils/             # 工具函数
│   │   │   ├── assets/            # 静态资源
│   │   │   ├── App.tsx
│   │   │   └── main.tsx
│   │   ├── index.html
│   │   ├── vite.config.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   └── api/                       # NestJS 后端应用
│       ├── src/
│       │   ├── modules/           # 功能模块
│       │   │   ├── auth/          # 认证模块
│       │   │   ├── user/          # 用户模块
│       │   │   ├── wechat/        # 微信模块
│       │   │   ├── account/       # 公众号模块
│       │   │   ├── article/       # 文章模块
│       │   │   ├── download/      # 下载模块
│       │   │   ├── export/        # 导出模块
│       │   │   ├── proxy/         # 代理模块
│       │   │   └── task/          # 任务模块
│       │   ├── common/            # 公共代码
│       │   │   ├── guards/        # 守卫
│       │   │   ├── interceptors/  # 拦截器
│       │   │   ├── filters/       # 过滤器
│       │   │   ├── pipes/         # 管道
│       │   │   └── decorators/    # 装饰器
│       │   ├── config/            # 配置
│       │   ├── database/          # 数据库
│       │   │   ├── entities/      # MikroORM 实体
│       │   │   ├── migrations/    # 数据库迁移
│       │   │   └── seeders/       # 种子数据
│       │   ├── app.module.ts
│       │   └── main.ts
│       ├── test/
│       ├── nest-cli.json
│       ├── tsconfig.json
│       └── package.json
│
├── tools/
│   ├── scripts/                   # 构建和部署脚本
│   └── generators/                # 代码生成器
│
├── docker/
│   ├── Dockerfile.web             # 前端 Docker 文件
│   ├── Dockerfile.api             # 后端 Docker 文件
│   └── docker-compose.yml         # Docker Compose 配置
│
├── docs/                          # 文档
│   ├── api/                       # API 文档
│   ├── architecture/              # 架构文档
│   └── deployment/                # 部署文档
│
├── .github/
│   └── workflows/                 # CI/CD 配置
│
├── package.json                   # 根 package.json
├── pnpm-workspace.yaml            # pnpm workspace 配置
├── turbo.json                     # Turborepo 配置
├── .env.example
└── README.md
```

### 4.3 技术架构决策

#### 4.3.1 前端技术选型

| 技术 | 选择 | 理由 |
|------|------|------|
| **框架** | React 18 | 生态成熟，组件丰富，团队熟悉度高 |
| **构建工具** | Vite | 快速冷启动，HMR性能优秀 |
| **状态管理** | Zustand | 轻量级，API简洁，TypeScript友好 |
| **数据获取** | TanStack Query | 智能缓存，自动重试，状态管理 |
| **路由** | React Router v6 | 官方推荐，功能完善 |
| **UI库** | Ant Design | 企业级组件库，文档完善 |
| **表格** | AG Grid React | 已有使用经验，功能强大 |
| **样式** | Tailwind CSS + CSS Modules | 实用优先，组件隔离 |
| **实时通信** | Socket.io Client | 下载进度实时推送 |
| **表单** | React Hook Form + Zod | 性能优秀，类型安全 |

#### 4.3.2 后端技术选型

| 技术 | 选择 | 理由 |
|------|------|------|
| **框架** | NestJS 10 | 企业级架构，依赖注入，模块化 |
| **ORM** | MikroORM | TypeScript原生，灵活的查询构建器 |
| **数据库** | PostgreSQL 15+ | ACID保证，JSON支持，性能优秀 |
| **缓存** | Redis 7+ | 高性能，支持多种数据结构 |
| **任务队列** | BullMQ | 可靠的任务队列，Redis基础 |
| **认证** | Passport + JWT | 行业标准，灵活扩展 |
| **验证** | Zod + nestjs-zod | TypeScript原生，前后端共享，类型推断 |
| **文档** | Swagger / OpenAPI | 自动生成API文档 |
| **日志** | Winston + Nest Logger | 结构化日志，多输出目标 |
| **配置** | @nestjs/config | 环境变量管理 |

---

## 5. Monorepo 方案选择

### 5.1 方案对比

| 特性 | **Turborepo** | **pnpm Workspace** | **Nx** |
|------|--------------|-------------------|--------|
| **构建缓存** | ✅ 远程缓存支持 | ❌ 需手动实现 | ✅ 本地+远程缓存 |
| **增量构建** | ✅ | ⚠️ 基础支持 | ✅ |
| **任务编排** | ✅ 管道配置 | ⚠️ 基础脚本 | ✅ 强大的依赖图 |
| **依赖管理** | 依赖 pnpm/yarn/npm | ✅ 原生支持 | 依赖 pnpm/yarn/npm |
| **学习曲线** | 低 | 低 | 中-高 |
| **性能** | 优秀 | 良好 | 优秀 |
| **配置复杂度** | 简单 | 极简 | 中等 |
| **生态系统** | 成长中 | 成熟 | 非常成熟 |
| **代码生成器** | ❌ | ❌ | ✅ |
| **可视化工具** | ⚠️ 基础 | ❌ | ✅ 强大 |
| **插件系统** | ⚠️ 有限 | ❌ | ✅ 丰富 |
| **CI 集成** | ✅ | ✅ | ✅ |
| **适用规模** | 中小型 | 小型-中型 | 中型-大型 |

### 5.2 推荐方案: **Turborepo + pnpm**

#### 选择理由:

1. **简洁高效**: Turborepo 配置简单，上手快，适合中小型项目
2. **性能优秀**: 远程缓存可显著加速 CI/CD
3. **pnpm 优势**:
   - 磁盘空间节省（硬链接）
   - 严格的依赖管理（避免幽灵依赖）
   - 快速安装速度
4. **社区活跃**: Vercel 团队维护，更新频繁
5. **渐进增强**: 未来可按需迁移到 Nx（如果项目规模扩大）

#### 配置示例:

**pnpm-workspace.yaml**:
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

**turbo.json**:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "build/**"]
    },
    "lint": {
      "cache": false
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    }
  }
}
```

**根 package.json**:
```json
{
  "name": "wechat-article-exporter-monorepo",
  "private": true,
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "clean": "turbo run clean && rm -rf node_modules"
  },
  "devDependencies": {
    "turbo": "^1.11.0",
    "@wechat-exporter/typescript-config": "workspace:*",
    "@wechat-exporter/eslint-config": "workspace:*"
  },
  "packageManager": "pnpm@8.15.0",
  "engines": {
    "node": ">=18.0.0",
    "pnpm": ">=8.0.0"
  }
}
```

---

## 6. 技术栈详细设计

### 6.1 前端技术栈

#### 6.1.1 核心依赖

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "zustand": "^4.4.7",
    "@tanstack/react-query": "^5.14.0",
    "antd": "^5.12.0",
    "ag-grid-react": "^31.0.0",
    "ag-grid-enterprise": "^31.0.0",
    "axios": "^1.6.2",
    "socket.io-client": "^4.6.0",
    "react-hook-form": "^7.49.0",
    "zod": "^3.22.4",
    "@hookform/resolvers": "^3.3.3",
    "date-fns": "^3.0.0",
    "qrcode.react": "^3.1.0",
    "file-saver": "^2.0.5",
    "jszip": "^3.10.1"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.2.0",
    "vite": "^5.0.0",
    "typescript": "^5.3.0",
    "tailwindcss": "^3.3.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "eslint": "^8.55.0",
    "prettier": "^3.1.0"
  }
}
```

#### 6.1.2 状态管理架构 (Zustand)

```typescript
// stores/auth.store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (credentials: LoginDto) => Promise<void>;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(persist(
  (set) => ({
    user: null,
    token: null,
    isAuthenticated: false,
    login: async (credentials) => {
      const { data } = await api.post('/auth/login', credentials);
      set({ user: data.user, token: data.token, isAuthenticated: true });
    },
    logout: () => set({ user: null, token: null, isAuthenticated: false })
  }),
  { name: 'auth-storage' }
));

// stores/download.store.ts
interface DownloadState {
  tasks: DownloadTask[];
  activeTask: DownloadTask | null;
  progress: Map<string, DownloadProgress>;
  addTask: (task: DownloadTask) => void;
  updateProgress: (taskId: string, progress: DownloadProgress) => void;
}

export const useDownloadStore = create<DownloadState>((set) => ({
  tasks: [],
  activeTask: null,
  progress: new Map(),
  addTask: (task) => set((state) => ({ tasks: [...state.tasks, task] })),
  updateProgress: (taskId, progress) => set((state) => ({
    progress: new Map(state.progress).set(taskId, progress)
  }))
}));
```

#### 6.1.3 API 服务层

```typescript
// services/api.client.ts
import axios from 'axios';
import { useAuthStore } from '@/stores/auth.store';

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 30000
});

// 请求拦截器：添加认证token
apiClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 响应拦截器：处理错误
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;

// services/article.service.ts
import apiClient from './api.client';
import type { Article, ArticleListParams } from '@wechat-exporter/shared/types';

export const articleService = {
  getList: (params: ArticleListParams) =>
    apiClient.get<Article[]>('/articles', { params }),

  getById: (id: string) =>
    apiClient.get<Article>(`/articles/${id}`),

  download: (urls: string[]) =>
    apiClient.post('/articles/download', { urls }),

  export: (params: ExportParams) =>
    apiClient.post('/articles/export', params, { responseType: 'blob' })
};
```

### 6.2 后端技术栈

#### 6.2.1 核心依赖

```json
{
  "dependencies": {
    "@nestjs/common": "^10.2.0",
    "@nestjs/core": "^10.2.0",
    "@nestjs/platform-express": "^10.2.0",
    "@nestjs/config": "^3.1.0",
    "@nestjs/jwt": "^10.2.0",
    "@nestjs/passport": "^10.0.0",
    "@nestjs/swagger": "^7.1.0",
    "@mikro-orm/core": "^6.0.0",
    "@mikro-orm/postgresql": "^6.0.0",
    "@mikro-orm/nestjs": "^5.2.0",
    "@nestjs/bullmq": "^10.0.0",
    "bullmq": "^5.0.0",
    "ioredis": "^5.3.0",
    "passport": "^0.7.0",
    "passport-jwt": "^4.0.0",
    "passport-local": "^1.0.0",
    "bcrypt": "^5.1.0",
    "zod": "^3.22.4",
    "nestjs-zod": "^3.0.0",
    "@anatine/zod-nestjs": "^2.0.0",
    "@anatine/zod-openapi": "^2.0.0",
    "axios": "^1.6.0",
    "@nestjs/websockets": "^10.2.0",
    "@nestjs/platform-socket.io": "^10.2.0",
    "winston": "^3.11.0"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.2.0",
    "@nestjs/testing": "^10.2.0",
    "@mikro-orm/cli": "^6.0.0",
    "@types/passport-jwt": "^3.0.0",
    "@types/bcrypt": "^5.0.0",
    "typescript": "^5.3.0",
    "jest": "^29.7.0",
    "supertest": "^6.3.0"
  }
}
```

#### 6.2.2 模块架构

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { MikroOrmModule } from '@mikro-orm/nestjs';
import { BullModule } from '@nestjs/bullmq';
import { AuthModule } from './modules/auth/auth.module';
import { UserModule } from './modules/user/user.module';
import { WechatModule } from './modules/wechat/wechat.module';
import { ArticleModule } from './modules/article/article.module';
import mikroOrmConfig from './config/mikro-orm.config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    MikroOrmModule.forRoot(mikroOrmConfig),
    BullModule.forRoot({
      connection: {
        host: process.env.REDIS_HOST,
        port: parseInt(process.env.REDIS_PORT)
      }
    }),
    AuthModule,
    UserModule,
    WechatModule,
    ArticleModule,
    // ... 其他模块
  ]
})
export class AppModule {}
```

---

## 7. 数据库设计

### 7.1 ER 图

```
┌─────────────┐       ┌──────────────┐       ┌─────────────┐
│    User     │       │ WechatAccount│       │   Account   │
├─────────────┤       ├──────────────┤       ├─────────────┤
│ id (PK)     │───┐   │ id (PK)      │   ┌───│ id (PK)     │
│ email       │   │   │ userId (FK)  │   │   │ fakeid      │
│ password    │   └──<│ authKey      │   │   │ nickname    │
│ name        │       │ token        │   │   │ biz         │
│ createdAt   │       │ cookies      │   │   │ avatar      │
│ updatedAt   │       │ expiresAt    │   │   │ createdAt   │
└─────────────┘       └──────────────┘   │   └─────────────┘
                                         │           │
                                         │           │ 1:N
┌─────────────┐       ┌──────────────┐   │   ┌─────────────┐
│  Download   │       │   Article    │───┘   │   Comment   │
│    Task     │       ├──────────────┤       ├─────────────┤
├─────────────┤   ┌──<│ id (PK)      │>──┐   │ id (PK)     │
│ id (PK)     │   │   │ accountId(FK)│   └──<│ articleId   │
│ userId (FK) │   │   │ aid          │       │ contentId   │
│ type        │   │   │ url          │       │ nickName    │
│ urls        │   │   │ title        │       │ content     │
│ status      │   │   │ author       │       │ likeNum     │
│ progress    │   │   │ publishedAt  │       │ createdAt   │
│ createdAt   │   │   │ htmlContent  │       └─────────────┘
└─────────────┘   │   │ createdAt    │
                  │   └──────────────┘
┌─────────────┐   │           │ 1:1
│   Export    │   │   ┌───────────────┐
│    Task     │   │   │   Metadata    │
├─────────────┤   │   ├───────────────┤
│ id (PK)     │   └──<│ articleId (FK)│
│ userId (FK) │       │ readNum       │
│ type        │       │ likeNum       │
│ articleIds  │       │ shareNum      │
│ format      │       │ commentNum    │
│ status      │       │ oldLikeNum    │
│ fileUrl     │       │ updatedAt     │
│ createdAt   │       └───────────────┘
└─────────────┘
        │
        │ 1:N
        ▼
┌─────────────┐       ┌──────────────┐
│  Resource   │       │   Proxy      │
├─────────────┤       ├──────────────┤
│ id (PK)     │       │ id (PK)      │
│ articleId   │       │ url          │
│ url         │       │ status       │
│ type        │       │ failures     │
│ s3Key       │       │ lastUsed     │
│ size        │       │ totalUse     │
│ createdAt   │       │ createdAt    │
└─────────────┘       └──────────────┘
```

### 7.2 MikroORM 实体定义

#### 7.2.1 User 实体

```typescript
// database/entities/user.entity.ts
import { Entity, PrimaryKey, Property, OneToMany, Collection } from '@mikro-orm/core';
import { WechatAccount } from './wechat-account.entity';
import { DownloadTask } from './download-task.entity';

@Entity({ tableName: 'users' })
export class User {
  @PrimaryKey({ type: 'uuid', defaultRaw: 'gen_random_uuid()' })
  id!: string;

  @Property({ unique: true })
  email!: string;

  @Property({ hidden: true })
  password!: string;

  @Property()
  name!: string;

  @Property({ nullable: true })
  avatar?: string;

  @Property({ default: true })
  isActive: boolean = true;

  @OneToMany(() => WechatAccount, account => account.user)
  wechatAccounts = new Collection<WechatAccount>(this);

  @OneToMany(() => DownloadTask, task => task.user)
  downloadTasks = new Collection<DownloadTask>(this);

  @Property({ onCreate: () => new Date() })
  createdAt: Date = new Date();

  @Property({ onUpdate: () => new Date() })
  updatedAt: Date = new Date();
}
```

#### 7.2.2 WechatAccount 实体

```typescript
// database/entities/wechat-account.entity.ts
import { Entity, PrimaryKey, Property, ManyToOne, JsonType } from '@mikro-orm/core';
import { User } from './user.entity';

interface CookieData {
  name: string;
  value: string;
  domain?: string;
  path?: string;
  expires?: number;
}

@Entity({ tableName: 'wechat_accounts' })
export class WechatAccount {
  @PrimaryKey({ type: 'uuid', defaultRaw: 'gen_random_uuid()' })
  id!: string;

  @ManyToOne(() => User)
  user!: User;

  @Property({ unique: true })
  authKey!: string;

  @Property()
  token!: string;

  @Property({ type: JsonType })
  cookies!: CookieData[];

  @Property()
  expiresAt!: Date;

  @Property({ default: true })
  isActive: boolean = true;

  @Property({ onCreate: () => new Date() })
  createdAt: Date = new Date();

  @Property({ onUpdate: () => new Date() })
  updatedAt: Date = new Date();
}
```

#### 7.2.3 Account 实体 (公众号)

```typescript
// database/entities/account.entity.ts
import { Entity, PrimaryKey, Property, OneToMany, Collection, Index } from '@mikro-orm/core';
import { Article } from './article.entity';

@Entity({ tableName: 'accounts' })
export class Account {
  @PrimaryKey({ type: 'uuid', defaultRaw: 'gen_random_uuid()' })
  id!: string;

  @Property({ unique: true })
  @Index()
  fakeid!: string;

  @Property()
  nickname!: string;

  @Property({ nullable: true })
  biz?: string;

  @Property({ nullable: true })
  avatar?: string;

  @Property({ nullable: true })
  signature?: string;

  @Property({ nullable: true, type: 'text' })
  description?: string;

  @Property({ default: 0 })
  articleCount: number = 0;

  @OneToMany(() => Article, article => article.account)
  articles = new Collection<Article>(this);

  @Property({ onCreate: () => new Date() })
  createdAt: Date = new Date();

  @Property({ onUpdate: () => new Date() })
  updatedAt: Date = new Date();
}
```

#### 7.2.4 Article 实体

```typescript
// database/entities/article.entity.ts
import { Entity, PrimaryKey, Property, ManyToOne, OneToOne, OneToMany, Collection, Index } from '@mikro-orm/core';
import { Account } from './account.entity';
import { Metadata } from './metadata.entity';
import { Comment } from './comment.entity';
import { Resource } from './resource.entity';

@Entity({ tableName: 'articles' })
export class Article {
  @PrimaryKey({ type: 'uuid', defaultRaw: 'gen_random_uuid()' })
  id!: string;

  @ManyToOne(() => Account)
  account!: Account;

  @Property()
  @Index()
  aid!: string; // 文章ID

  @Property({ unique: true })
  @Index()
  url!: string;

  @Property()
  title!: string;

  @Property({ nullable: true })
  author?: string;

  @Property({ nullable: true, type: 'text' })
  digest?: string;

  @Property({ nullable: true })
  cover?: string;

  @Property({ type: 'text', nullable: true })
  htmlContent?: string; // 存储HTML内容

  @Property({ nullable: true })
  commentId?: string;

  @Property({ nullable: true })
  @Index()
  publishedAt?: Date;

  @OneToOne(() => Metadata, metadata => metadata.article, { nullable: true })
  metadata?: Metadata;

  @OneToMany(() => Comment, comment => comment.article)
  comments = new Collection<Comment>(this);

  @OneToMany(() => Resource, resource => resource.article)
  resources = new Collection<Resource>(this);

  @Property({ default: false })
  isDeleted: boolean = false;

  @Property({ default: false })
  isChecking: boolean = false; // 审核中

  @Property({ onCreate: () => new Date() })
  createdAt: Date = new Date();

  @Property({ onUpdate: () => new Date() })
  updatedAt: Date = new Date();
}
```

#### 7.2.5 Metadata 实体

```typescript
// database/entities/metadata.entity.ts
import { Entity, PrimaryKey, OneToOne, Property } from '@mikro-orm/core';
import { Article } from './article.entity';

@Entity({ tableName: 'metadata' })
export class Metadata {
  @PrimaryKey({ type: 'uuid', defaultRaw: 'gen_random_uuid()' })
  id!: string;

  @OneToOne(() => Article, article => article.metadata, { owner: true })
  article!: Article;

  @Property({ default: 0 })
  readNum: number = 0;

  @Property({ default: 0 })
  likeNum: number = 0;

  @Property({ default: 0 })
  oldLikeNum: number = 0;

  @Property({ default: 0 })
  shareNum: number = 0;

  @Property({ default: 0 })
  commentNum: number = 0;

  @Property({ onUpdate: () => new Date() })
  updatedAt: Date = new Date();
}
```

#### 7.2.6 DownloadTask 实体

```typescript
// database/entities/download-task.entity.ts
import { Entity, PrimaryKey, Property, ManyToOne, Enum, JsonType } from '@mikro-orm/core';
import { User } from './user.entity';

export enum DownloadType {
  HTML = 'html',
  METADATA = 'metadata',
  COMMENTS = 'comments'
}

export enum TaskStatus {
  PENDING = 'pending',
  PROCESSING = 'processing',
  COMPLETED = 'completed',
  FAILED = 'failed',
  CANCELLED = 'cancelled'
}

interface DownloadProgress {
  total: number;
  completed: number;
  failed: number;
  deleted: number;
}

@Entity({ tableName: 'download_tasks' })
export class DownloadTask {
  @PrimaryKey({ type: 'uuid', defaultRaw: 'gen_random_uuid()' })
  id!: string;

  @ManyToOne(() => User)
  user!: User;

  @Enum(() => DownloadType)
  type!: DownloadType;

  @Property({ type: JsonType })
  urls!: string[];

  @Enum(() => TaskStatus)
  status: TaskStatus = TaskStatus.PENDING;

  @Property({ type: JsonType, nullable: true })
  progress?: DownloadProgress;

  @Property({ nullable: true, type: 'text' })
  error?: string;

  @Property({ onCreate: () => new Date() })
  createdAt: Date = new Date();

  @Property({ nullable: true })
  completedAt?: Date;
}
```

### 7.3 数据库索引策略

```typescript
// 性能优化索引
@Index({ properties: ['account', 'publishedAt'] }) // 按公众号和发布时间查询
@Index({ properties: ['account', 'createdAt'] })   // 按公众号和创建时间查询
@Index({ properties: ['url'] })                     // URL唯一性和快速查找
@Index({ properties: ['aid'] })                     // 文章ID查询
```

### 7.4 数据迁移示例

```typescript
// database/migrations/Migration20260118000000.ts
import { Migration } from '@mikro-orm/migrations';

export class Migration20260118000000 extends Migration {
  async up(): Promise<void> {
    this.addSql(`
      create table "users" (
        "id" uuid not null default gen_random_uuid(),
        "email" varchar(255) not null,
        "password" varchar(255) not null,
        "name" varchar(255) not null,
        "avatar" varchar(255) null,
        "is_active" boolean not null default true,
        "created_at" timestamptz not null,
        "updated_at" timestamptz not null,
        constraint "users_pkey" primary key ("id")
      );
    `);

    this.addSql(`
      create unique index "users_email_unique" on "users" ("email");
    `);

    // ... 其他表创建语句
  }

  async down(): Promise<void> {
    this.addSql('drop table if exists "users" cascade;');
    // ... 其他表删除语句
  }
}
```

---

## 8. API 接口设计

### 8.1 RESTful API 规范

#### 8.1.1 统一响应格式

```typescript
// common/dto/response.dto.ts
export class ApiResponse<T> {
  success: boolean;
  data?: T;
  message?: string;
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  timestamp: string;
}

// 成功响应
{
  "success": true,
  "data": { ... },
  "timestamp": "2026-01-18T12:00:00.000Z"
}

// 错误响应
{
  "success": false,
  "error": {
    "code": "ARTICLE_NOT_FOUND",
    "message": "Article not found",
    "details": { "articleId": "123" }
  },
  "timestamp": "2026-01-18T12:00:00.000Z"
}
```

#### 8.1.2 分页响应格式

```typescript
export class PaginatedResponse<T> {
  success: boolean;
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
  timestamp: string;
}
```

### 8.2 核心 API 端点

#### 8.2.1 认证模块 (Auth)

```typescript
// POST /api/auth/register - 用户注册
interface RegisterDto {
  email: string;
  password: string;
  name: string;
}

// POST /api/auth/login - 用户登录
interface LoginDto {
  email: string;
  password: string;
}

interface LoginResponse {
  user: User;
  accessToken: string;
  refreshToken: string;
}

// POST /api/auth/refresh - 刷新Token
interface RefreshDto {
  refreshToken: string;
}

// POST /api/auth/logout - 登出
// GET /api/auth/profile - 获取当前用户信息
```

#### 8.2.2 微信模块 (Wechat)

```typescript
// GET /api/wechat/qrcode - 获取登录二维码
interface QrcodeResponse {
  uuid: string;
  qrcodeUrl: string; // base64 或 URL
}

// GET /api/wechat/scan-status?uuid=xxx - 轮询扫码状态
interface ScanStatusResponse {
  status: 0 | 1 | 200; // 0-未扫码, 1-已扫码待确认, 200-登录成功
  message: string;
}

// POST /api/wechat/login - 确认登录
interface WechatLoginDto {
  uuid: string;
  // ... 其他登录参数
}

interface WechatLoginResponse {
  authKey: string;
  expiresAt: string;
}

// DELETE /api/wechat/logout - 微信登出
```

#### 8.2.3 公众号模块 (Account)

```typescript
// GET /api/accounts/search?keyword=xxx&page=1&limit=10 - 搜索公众号
interface SearchAccountParams {
  keyword: string;
  page?: number;
  limit?: number;
}

interface AccountDto {
  id: string;
  fakeid: string;
  nickname: string;
  biz?: string;
  avatar?: string;
  signature?: string;
  articleCount: number;
}

// GET /api/accounts/:id - 获取公众号详情
// GET /api/accounts - 获取用户已关注的公众号列表
// POST /api/accounts/:id/follow - 关注公众号
// DELETE /api/accounts/:id/follow - 取消关注
```

#### 8.2.4 文章模块 (Article)

```typescript
// GET /api/articles?accountId=xxx&page=1&limit=20&keyword=xxx - 获取文章列表
interface ArticleListParams {
  accountId?: string;
  keyword?: string;
  startDate?: string;
  endDate?: string;
  page?: number;
  limit?: number;
  sortBy?: 'publishedAt' | 'readNum' | 'likeNum';
  order?: 'asc' | 'desc';
}

interface ArticleDto {
  id: string;
  aid: string;
  url: string;
  title: string;
  author?: string;
  digest?: string;
  cover?: string;
  publishedAt?: string;
  account: {
    id: string;
    nickname: string;
    avatar?: string;
  };
  metadata?: {
    readNum: number;
    likeNum: number;
    shareNum: number;
    commentNum: number;
  };
}

// GET /api/articles/:id - 获取文章详情
interface ArticleDetailDto extends ArticleDto {
  htmlContent?: string;
  comments?: CommentDto[];
  resources?: ResourceDto[];
}

// POST /api/articles/download - 批量下载文章
interface DownloadArticlesDto {
  urls: string[];
  type: 'html' | 'metadata' | 'comments';
  withCredential?: boolean; // 是否使用凭证
}

interface DownloadResponse {
  taskId: string;
  status: 'pending' | 'processing';
}

// GET /api/articles/:id/html - 获取文章HTML内容
// GET /api/articles/:id/metadata - 获取文章元数据
// GET /api/articles/:id/comments - 获取文章评论
```

#### 8.2.5 下载模块 (Download)

```typescript
// POST /api/download/tasks - 创建下载任务
interface CreateDownloadTaskDto {
  urls: string[];
  type: 'html' | 'metadata' | 'comments';
  accountId?: string;
}

// GET /api/download/tasks - 获取任务列表
// GET /api/download/tasks/:id - 获取任务详情
interface DownloadTaskDto {
  id: string;
  type: string;
  status: 'pending' | 'processing' | 'completed' | 'failed';
  progress: {
    total: number;
    completed: number;
    failed: number;
  };
  createdAt: string;
  completedAt?: string;
}

// DELETE /api/download/tasks/:id - 取消任务
// GET /api/download/tasks/:id/progress - 获取任务实时进度 (WebSocket)
```

#### 8.2.6 导出模块 (Export)

```typescript
// POST /api/export/tasks - 创建导出任务
interface CreateExportTaskDto {
  articleIds: string[];
  format: 'excel' | 'json' | 'html' | 'markdown' | 'txt' | 'word';
  options?: {
    includeMetadata?: boolean;
    includeComments?: boolean;
    includeResources?: boolean;
  };
}

// GET /api/export/tasks/:id - 获取导出任务详情
// GET /api/export/tasks/:id/download - 下载导出文件
// DELETE /api/export/tasks/:id - 删除导出任务
```

### 8.3 NestJS 控制器实现示例

```typescript
// modules/article/article.controller.ts
import { Controller, Get, Post, Body, Param, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiBearerAuth } from '@nestjs/swagger';
import { JwtAuthGuard } from '@/common/guards/jwt-auth.guard';
import { ArticleService } from './article.service';
import { ArticleListParams, DownloadArticlesDto } from './dto';

@ApiTags('articles')
@Controller('articles')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
export class ArticleController {
  constructor(private readonly articleService: ArticleService) {}

  @Get()
  @ApiOperation({ summary: '获取文章列表' })
  async getArticles(@Query() params: ArticleListParams) {
    return this.articleService.findAll(params);
  }

  @Get(':id')
  @ApiOperation({ summary: '获取文章详情' })
  async getArticle(@Param('id') id: string) {
    return this.articleService.findOne(id);
  }

  @Post('download')
  @ApiOperation({ summary: '批量下载文章' })
  async downloadArticles(@Body() dto: DownloadArticlesDto) {
    return this.articleService.downloadBatch(dto);
  }
}
```

### 8.4 WebSocket 实时通信

```typescript
// modules/download/download.gateway.ts
import { WebSocketGateway, WebSocketServer, SubscribeMessage } from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({ namespace: '/download', cors: true })
export class DownloadGateway {
  @WebSocketServer()
  server: Server;

  // 订阅任务进度
  @SubscribeMessage('subscribe:task')
  handleSubscribe(client: Socket, taskId: string) {
    client.join(`task:${taskId}`);
  }

  // 发送进度更新
  sendProgress(taskId: string, progress: DownloadProgress) {
    this.server.to(`task:${taskId}`).emit('progress', progress);
  }

  // 任务完成通知
  sendComplete(taskId: string, result: any) {
    this.server.to(`task:${taskId}`).emit('complete', result);
  }
}
```

---

## 9. 前端架构设计

### 9.1 路由设计

```typescript
// src/router/index.tsx
import { createBrowserRouter } from 'react-router-dom';
import { MainLayout } from '@/layouts/MainLayout';
import { AuthLayout } from '@/layouts/AuthLayout';

const router = createBrowserRouter([
  {
    path: '/auth',
    element: <AuthLayout />,
    children: [
      { path: 'login', element: <LoginPage /> },
      { path: 'register', element: <RegisterPage /> }
    ]
  },
  {
    path: '/',
    element: <MainLayout />,
    children: [
      { index: true, element: <Navigate to="/dashboard" /> },
      { path: 'dashboard', element: <DashboardPage /> },
      {
        path: 'accounts',
        children: [
          { index: true, element: <AccountListPage /> },
          { path: ':id', element: <AccountDetailPage /> }
        ]
      },
      {
        path: 'articles',
        children: [
          { index: true, element: <ArticleListPage /> },
          { path: ':id', element: <ArticleDetailPage /> }
        ]
      },
      { path: 'download', element: <DownloadPage /> },
      { path: 'export', element: <ExportPage /> },
      { path: 'settings', element: <SettingsPage /> }
    ]
  }
]);
```

### 9.2 组件架构

```
src/
├── components/
│   ├── common/              # 通用组件
│   │   ├── Button/
│   │   ├── Input/
│   │   ├── Modal/
│   │   └── Loading/
│   ├── layout/              # 布局组件
│   │   ├── Header/
│   │   ├── Sidebar/
│   │   └── Footer/
│   ├── auth/                # 认证相关组件
│   │   ├── LoginForm/
│   │   ├── RegisterForm/
│   │   └── WechatQRCode/
│   ├── account/             # 公众号相关组件
│   │   ├── AccountCard/
│   │   ├── AccountSearch/
│   │   └── AccountList/
│   ├── article/             # 文章相关组件
│   │   ├── ArticleGrid/     # AG Grid 封装
│   │   ├── ArticleCard/
│   │   ├── ArticlePreview/
│   │   └── ArticleFilter/
│   ├── download/            # 下载相关组件
│   │   ├── DownloadButton/
│   │   ├── DownloadProgress/
│   │   └── DownloadHistory/
│   └── export/              # 导出相关组件
│       ├── ExportModal/
│       ├── ExportProgress/
│       └── FormatSelector/
```

### 9.3 状态管理示例

```typescript
// stores/article.store.ts
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';
import type { Article, ArticleListParams } from '@wechat-exporter/shared/types';

interface ArticleState {
  articles: Article[];
  selectedArticles: Set<string>;
  filters: ArticleListParams;
  loading: boolean;

  // Actions
  setArticles: (articles: Article[]) => void;
  addArticle: (article: Article) => void;
  toggleSelect: (id: string) => void;
  selectAll: () => void;
  clearSelection: () => void;
  setFilters: (filters: Partial<ArticleListParams>) => void;
}

export const useArticleStore = create<ArticleState>()(devtools(
  (set, get) => ({
    articles: [],
    selectedArticles: new Set(),
    filters: { page: 1, limit: 20 },
    loading: false,

    setArticles: (articles) => set({ articles }),

    addArticle: (article) => set((state) => ({
      articles: [...state.articles, article]
    })),

    toggleSelect: (id) => set((state) => {
      const selected = new Set(state.selectedArticles);
      if (selected.has(id)) {
        selected.delete(id);
      } else {
        selected.add(id);
      }
      return { selectedArticles: selected };
    }),

    selectAll: () => set((state) => ({
      selectedArticles: new Set(state.articles.map(a => a.id))
    })),

    clearSelection: () => set({ selectedArticles: new Set() }),

    setFilters: (filters) => set((state) => ({
      filters: { ...state.filters, ...filters }
    }))
  }),
  { name: 'article-store' }
));
```

### 9.4 自定义 Hooks

```typescript
// hooks/useDownload.ts
import { useState, useCallback } from 'react';
import { useQuery, useMutation } from '@tanstack/react-query';
import { io, Socket } from 'socket.io-client';
import { downloadService } from '@/services/download.service';

export interface DownloadProgress {
  total: number;
  completed: number;
  failed: number;
  percentage: number;
}

export function useDownload() {
  const [socket, setSocket] = useState<Socket | null>(null);
  const [progress, setProgress] = useState<DownloadProgress>({
    total: 0,
    completed: 0,
    failed: 0,
    percentage: 0
  });

  // 创建下载任务
  const downloadMutation = useMutation({
    mutationFn: downloadService.createTask,
    onSuccess: (data) => {
      // 连接 WebSocket 监听进度
      const ws = io(`${import.meta.env.VITE_WS_URL}/download`, {
        auth: { token: localStorage.getItem('token') }
      });

      ws.emit('subscribe:task', data.taskId);

      ws.on('progress', (data: DownloadProgress) => {
        setProgress(data);
      });

      ws.on('complete', () => {
        ws.disconnect();
        setSocket(null);
      });

      setSocket(ws);
    }
  });

  const cancelDownload = useCallback(() => {
    if (socket) {
      socket.disconnect();
      setSocket(null);
    }
  }, [socket]);

  return {
    download: downloadMutation.mutate,
    isDownloading: downloadMutation.isPending,
    progress,
    cancelDownload
  };
}

// hooks/useWechatLogin.ts
import { useState, useEffect } from 'react';
import { wechatService } from '@/services/wechat.service';

export function useWechatLogin() {
  const [qrcode, setQrcode] = useState<string>('');
  const [uuid, setUuid] = useState<string>('');
  const [status, setStatus] = useState<0 | 1 | 200>(0);

  // 获取二维码
  const getQrcode = async () => {
    const { data } = await wechatService.getQrcode();
    setQrcode(data.qrcodeUrl);
    setUuid(data.uuid);
  };

  // 轮询扫码状态
  useEffect(() => {
    if (!uuid) return;

    const interval = setInterval(async () => {
      const { data } = await wechatService.getScanStatus(uuid);
      setStatus(data.status);

      if (data.status === 200) {
        clearInterval(interval);
        // 登录成功，保存 authKey
        await wechatService.confirmLogin(uuid);
      }
    }, 1000);

    return () => clearInterval(interval);
  }, [uuid]);

  return { qrcode, status, getQrcode };
}
```

---

## 10. 迁移方案

### 10.1 数据迁移策略

#### 阶段一：IndexedDB 数据导出

```typescript
// tools/scripts/export-indexeddb.ts
import Dexie from 'dexie';

interface MigrationData {
  accounts: any[];
  articles: any[];
  htmlContents: any[];
  metadata: any[];
  comments: any[];
  resources: any[];
}

async function exportIndexedDB(): Promise<MigrationData> {
  const db = new Dexie('exporter.wxdown.online');

  db.version(1).stores({
    article: 'url, fakeid',
    html: 'url, fakeid',
    metadata: 'url, fakeid',
    comment: 'id, url, fakeid',
    resource: 'url, fakeid',
    info: 'fakeid'
  });

  const data: MigrationData = {
    accounts: await db.table('info').toArray(),
    articles: await db.table('article').toArray(),
    htmlContents: await db.table('html').toArray(),
    metadata: await db.table('metadata').toArray(),
    comments: await db.table('comment').toArray(),
    resources: await db.table('resource').toArray()
  };

  // 导出为 JSON
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `migration-data-${Date.now()}.json`;
  a.click();

  return data;
}
```

#### 阶段二：PostgreSQL 数据导入

```typescript
// apps/api/src/scripts/import-migration-data.ts
import { MikroORM } from '@mikro-orm/core';
import { Account, Article, Metadata } from '@/database/entities';
import * as fs from 'fs';

async function importMigrationData(filePath: string) {
  const orm = await MikroORM.init();
  const em = orm.em.fork();

  const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));

  // 1. 导入公众号
  for (const accountData of data.accounts) {
    const account = em.create(Account, {
      fakeid: accountData.fakeid,
      nickname: accountData.nickname,
      biz: accountData.biz,
      avatar: accountData.avatar
    });
    em.persist(account);
  }
  await em.flush();

  // 2. 导入文章
  for (const articleData of data.articles) {
    const account = await em.findOne(Account, { fakeid: articleData.fakeid });
    if (!account) continue;

    const article = em.create(Article, {
      account,
      aid: articleData.aid,
      url: articleData.link,
      title: articleData.title,
      author: articleData.author,
      publishedAt: new Date(articleData.update_time * 1000)
    });
    em.persist(article);
  }
  await em.flush();

  // 3. 导入HTML内容
  for (const htmlData of data.htmlContents) {
    const article = await em.findOne(Article, { url: htmlData.url });
    if (!article) continue;

    article.htmlContent = await blobToText(htmlData.file);
    article.commentId = htmlData.commentID;
  }
  await em.flush();

  // 4. 导入元数据
  for (const metadataData of data.metadata) {
    const article = await em.findOne(Article, { url: metadataData.url });
    if (!article) continue;

    const metadata = em.create(Metadata, {
      article,
      readNum: metadataData.readNum || 0,
      likeNum: metadataData.likeNum || 0,
      shareNum: metadataData.shareNum || 0,
      commentNum: metadataData.commentNum || 0
    });
    em.persist(metadata);
  }
  await em.flush();

  console.log('Migration completed!');
  await orm.close();
}

function blobToText(blob: Blob): Promise<string> {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result as string);
    reader.readAsText(blob);
  });
}
```

### 10.2 代码迁移清单

| 原模块 | 新模块 | 状态 | 复杂度 |
|--------|--------|------|--------|
| `server/utils/CookieStore.ts` | `apps/api/src/modules/wechat/cookie.service.ts` | 🔄 重构 | 中 |
| `server/utils/proxy-request.ts` | `apps/api/src/modules/wechat/proxy.service.ts` | 🔄 重构 | 高 |
| `utils/download/Downloader.ts` | `apps/api/src/modules/download/download.processor.ts` | ♻️ 后端化 | 高 |
| `utils/download/Exporter.ts` | `apps/api/src/modules/export/export.processor.ts` | ♻️ 后端化 | 高 |
| `utils/download/ProxyManager.ts` | `apps/api/src/modules/proxy/proxy.service.ts` | 🔄 重构 | 中 |
| `components/dashboard/*` | `apps/web/src/components/*` | 🔄 重写为React | 高 |
| `store/v2/*` | `apps/web/src/stores/*` (Zustand) | 🔄 重写 | 中 |
| `composables/*` | `apps/web/src/hooks/*` (React Hooks) | 🔄 重写 | 中 |

### 10.3 功能兼容性矩阵

| 功能 | 当前版本 | 新版本 | 变更说明 |
|------|---------|--------|----------|
| **二维码登录** | ✅ 客户端 | ✅ 服务端 | Cookie管理移至后端 |
| **公众号搜索** | ✅ | ✅ | 无变更 |
| **文章列表** | ✅ IndexedDB | ✅ PostgreSQL | 数据库变更 |
| **HTML下载** | ✅ 客户端 | ✅ 服务端 | 后台任务队列 |
| **元数据抓取** | ✅ 客户端 | ✅ 服务端 | 需配置Credential |
| **评论抓取** | ✅ 客户端 | ✅ 服务端 | 需配置Credential |
| **Excel导出** | ✅ 客户端 | ✅ 服务端 | 文件存储S3 |
| **HTML离线导出** | ✅ File System API | ✅ ZIP下载 | 改为ZIP格式 |
| **Markdown导出** | ✅ | ✅ | 无变更 |
| **实时进度** | ✅ Events | ✅ WebSocket | 技术升级 |

---

## 11. 实施计划

### 11.1 迭代计划 (6个月)

#### 第一阶段：基础架构 (Month 1-2)

**Week 1-2: 项目初始化**
- [x] 创建 Monorepo 结构
- [x] 配置 Turborepo + pnpm
- [x] 设置 TypeScript 配置
- [x] 配置 ESLint + Prettier
- [x] 设置 Git Hooks (Husky + lint-staged)

**Week 3-4: 数据库设计**
- [ ] 设计 PostgreSQL 数据模型
- [ ] 编写 MikroORM 实体
- [ ] 创建数据库迁移脚本
- [ ] 编写种子数据
- [ ] 数据库性能优化（索引）

**Week 5-6: 后端基础模块**
- [ ] 搭建 NestJS 项目
- [ ] 实现用户认证（JWT + Passport）
- [ ] 实现用户注册/登录
- [ ] 配置 Redis 缓存
- [ ] 配置 BullMQ 任务队列

**Week 7-8: 前端基础框架**
- [ ] 搭建 React + Vite 项目
- [ ] 配置路由（React Router）
- [ ] 配置状态管理（Zustand）
- [ ] 配置 UI 库（Ant Design）
- [ ] 实现布局组件
- [ ] 实现登录/注册页面

#### 第二阶段：核心功能 (Month 3-4)

**Week 9-10: 微信登录模块**
- [ ] 后端：实现 QR 码生成
- [ ] 后端：实现扫码状态轮询
- [ ] 后端：实现 Cookie 管理服务
- [ ] 前端：实现 QR 码展示组件
- [ ] 前端：实现扫码状态实时更新
- [ ] 集成测试

**Week 11-12: 公众号搜索模块**
- [ ] 后端：实现公众号搜索 API
- [ ] 后端：实现公众号信息缓存
- [ ] 前端：实现搜索组件
- [ ] 前端：实现公众号列表展示
- [ ] 前端：实现公众号详情页

**Week 13-14: 文章列表模块**
- [ ] 后端：实现文章列表 API
- [ ] 后端：实现文章搜索/过滤
- [ ] 后端：实现分页
- [ ] 前端：实现 AG Grid 表格
- [ ] 前端：实现文章过滤器
- [ ] 前端：实现批量选择

**Week 15-16: 下载模块**
- [ ] 后端：实现 HTML 下载处理器
- [ ] 后端：实现元数据下载处理器
- [ ] 后端：实现评论下载处理器
- [ ] 后端：实现任务队列管理
- [ ] 前端：实现下载按钮
- [ ] 前端：实现实时进度显示（WebSocket）

#### 第三阶段：高级功能 (Month 5)

**Week 17-18: 导出模块**
- [ ] 后端：实现 Excel 导出
- [ ] 后端：实现 JSON 导出
- [ ] 后端：实现 HTML ZIP 导出
- [ ] 后端：实现 Markdown 导出
- [ ] 后端：实现文件存储（S3/OSS）
- [ ] 前端：实现导出格式选择
- [ ] 前端：实现导出进度显示

**Week 19-20: 资源管理**
- [ ] 后端：实现资源下载服务
- [ ] 后端：实现代理轮转服务
- [ ] 后端：实现资源存储
- [ ] 前端：实现资源预览

#### 第四阶段：优化与部署 (Month 6)

**Week 21-22: 性能优化**
- [ ] 后端：API 性能优化
- [ ] 后端：数据库查询优化
- [ ] 后端：缓存策略优化
- [ ] 前端：代码分割
- [ ] 前端：懒加载优化
- [ ] 前端：打包优化

**Week 23-24: 部署与监控**
- [ ] 编写 Dockerfile
- [ ] 配置 Docker Compose
- [ ] 配置 CI/CD（GitHub Actions）
- [ ] 配置日志系统（Winston）
- [ ] 配置监控（可选：Sentry）
- [ ] 编写部署文档
- [ ] 生产环境测试
- [ ] 正式发布

### 11.2 里程碑

| 里程碑 | 时间 | 交付物 |
|--------|------|--------|
| **M1: 基础架构完成** | Month 2 | Monorepo项目、数据库、认证系统 |
| **M2: 核心功能完成** | Month 4 | 登录、搜索、列表、下载 |
| **M3: 完整功能** | Month 5 | 导出、资源管理 |
| **M4: 生产就绪** | Month 6 | 性能优化、部署、文档 |

### 11.3 团队配置建议

- **全栈工程师 x2**: 负责前后端开发
- **前端工程师 x1**: 专注 React 组件开发
- **后端工程师 x1**: 专注 NestJS 服务开发
- **DevOps 工程师 x0.5**: 兼职负责部署和CI/CD

---

## 12. 风险评估与应对

### 12.1 技术风险

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|----------|
| **微信API变更** | 高 | 高 | 设计抽象层，便于快速适配；保留原始HTML缓存 |
| **性能瓶颈** | 中 | 中 | 提前进行压力测试；使用缓存和队列优化 |
| **数据迁移失败** | 低 | 高 | 编写完整的迁移脚本；提供回滚方案 |
| **第三方依赖问题** | 中 | 低 | 锁定依赖版本；定期更新和测试 |
| **浏览器兼容性** | 低 | 中 | 使用 Polyfills；明确支持的浏览器版本 |

### 12.2 业务风险

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|----------|
| **用户数据安全** | 中 | 高 | 实施严格的权限控制；数据加密存储 |
| **服务稳定性** | 中 | 高 | 实施监控和告警；设计降级方案 |
| **成本超支** | 中 | 中 | 使用成本优化的云服务；监控资源使用 |
| **开发延期** | 中 | 中 | 采用敏捷开发；定期回顾和调整计划 |

### 12.3 应对策略

1. **技术选型**: 选择成熟稳定的技术栈，降低技术风险
2. **增量开发**: 采用分阶段开发，确保每个阶段都有可交付成果
3. **自动化测试**: 编写单元测试和集成测试，确保代码质量
4. **文档先行**: 在开发前完成详细设计文档，减少返工
5. **代码审查**: 实施严格的代码审查流程，提高代码质量
6. **监控告警**: 部署后实施完善的监控，快速发现和解决问题

---

## 13. 总结

### 13.1 重构收益

1. **架构升级**: 从单体架构升级到前后端分离，提升可维护性和可扩展性
2. **数据持久化**: 从浏览器存储迁移到 PostgreSQL，支持多用户和数据共享
3. **企业级特性**: 增加用户系统、权限管理、任务队列等企业特性
4. **性能提升**: 通过后端缓存、任务队列等优化，提升整体性能
5. **开发效率**: Monorepo 架构提升代码复用，加快开发速度

### 13.2 技术亮点

- **Monorepo**: Turborepo + pnpm 实现高效的代码管理
- **类型安全**: 全栈 TypeScript，共享类型定义
- **实时通信**: WebSocket 实现下载进度实时推送
- **任务队列**: BullMQ 处理耗时的下载和导出任务
- **缓存策略**: Redis 多层缓存提升性能
- **ORM**: MikroORM 提供类型安全的数据库操作

### 13.3 后续规划

- **移动端支持**: 开发 React Native 移动应用
- **浏览器插件**: 开发 Chrome 扩展，直接从微信文章页面导出
- **AI 增强**: 集成 AI 进行文章摘要、分类等功能
- **协作功能**: 支持团队协作和文章共享
- **API 市场**: 开放 API，构建生态系统

---

**文档版本**: 1.0.0
**最后更新**: 2026-01-18
**维护者**: 架构设计团队

---

## 附录

### A. 环境变量配置

```bash
# apps/web/.env
VITE_API_BASE_URL=http://localhost:3000/api
VITE_WS_URL=http://localhost:3000

# apps/api/.env
NODE_ENV=development
PORT=3000

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=wechat_exporter
DB_USER=postgres
DB_PASSWORD=password

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# JWT
JWT_SECRET=your-secret-key
JWT_EXPIRES_IN=7d

# AWS S3 (Optional)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=
AWS_S3_BUCKET=
```

### B. 开发命令

```bash
# 安装依赖
pnpm install

# 开发模式（同时启动前后端）
pnpm dev

# 只启动前端
pnpm --filter web dev

# 只启动后端
pnpm --filter api dev

# 构建所有项目
pnpm build

# 运行测试
pnpm test

# 代码检查
pnpm lint

# 数据库迁移
pnpm --filter api migration:create
pnpm --filter api migration:up

# 生成 API 文档
pnpm --filter api docs:generate
```

### C. 参考资料

- [NestJS 官方文档](https://docs.nestjs.com/)
- [MikroORM 文档](https://mikro-orm.io/)
- [React 官方文档](https://react.dev/)
- [Turborepo 文档](https://turbo.build/repo/docs)
- [pnpm 文档](https://pnpm.io/)
- [AG Grid React 文档](https://www.ag-grid.com/react-data-grid/)
- [TanStack Query 文档](https://tanstack.com/query/latest)
- [Zustand 文档](https://docs.pmnd.rs/zustand/getting-started/introduction)

### D. 技术选型评估：Alova.js vs TanStack Query

**评估日期**: 2026-01-18
**评估人**: 架构设计团队
**结论**: ❌ 不推荐使用 Alova.js，保持 TanStack Query + Axios 方案

---

#### D.1 背景

在重构方案制定过程中，团队评估了 [Alova.js](https://alova.js.org/zh-CN/) 作为请求管理工具的可行性。Alova.js 是一个轻量级的请求场景管理库（RSM - Request Scene Management），主打简洁的 API 和自动状态管理。

#### D.2 Alova.js 核心特性

根据官方文档和社区反馈，Alova.js 具有以下特点：

**优势**:
1. **极轻量**: 仅 4KB+，是 axios 的 30%
2. **跨框架支持**: Vue、React、Svelte、Uniapp、Taro
3. **自动状态管理**: 响应数据自动状态化，无需手动管理
4. **智能缓存**: 提供内存模式、缓存占位模式、恢复模式
5. **请求共享**: 自动合并相同请求
6. **内置请求重试**: 可配置重试次数和间隔
7. **丰富的请求策略**: usePagination、useWatcher、useSerialRequest 等
8. **多请求适配器**: 支持 Axios、Fetch、XMLHttpRequest

**示例代码**:
```typescript
// 简洁的 API
const { data, loading, error } = useRequest(alova.Get('/api/articles'));

// 防抖搜索
const { data } = useWatcher(
  () => alova.Get('/search', { params: { keyword } }),
  [keyword],
  { debounce: 300 }
);

// 分页
const { data, page, pageSize, onNext, onPrev } = usePagination(
  (page, size) => alova.Get('/articles', { params: { page, size } })
);
```

#### D.3 对比分析：Alova.js vs TanStack Query + Axios

| 维度 | Alova.js | TanStack Query + Axios | 评分 |
|------|----------|------------------------|------|
| **体积** | 4KB | 24KB | Alova ✅ |
| **学习曲线** | 中等（新概念 RSM） | 低（主流技术） | TQ ✅ |
| **社区生态** | ~3K stars（新） | ~39K stars（成熟） | TQ ✅ |
| **文档** | 中文为主 | 英文完善，国际化 | TQ ✅ |
| **API 简洁性** | 非常简洁 | 需要配置 | Alova ✅ |
| **TypeScript** | ✅ 原生支持 | ✅ 原生支持 | 平局 |
| **缓存策略** | 3种模式 | 强大灵活 | TQ ✅ |
| **React 18 支持** | 未知 | ✅ 完美支持并发 | TQ ✅ |
| **生产验证** | ⚠️ 较少 | ✅ 大量企业使用 | TQ ✅ |
| **招聘难度** | 高（小众） | 低（主流技能） | TQ ✅ |
| **长期维护** | ⚠️ 不确定 | ✅ Vercel 团队 | TQ ✅ |

#### D.4 决策理由

**不推荐使用 Alova.js 的关键原因**:

1. **项目特性不匹配**
   - 本项目是企业级应用，需要稳定可靠的技术栈
   - 微信 API 可能随时变化，需要快速应对和社区支持
   - 多人协作项目，团队成员对主流技术更熟悉

2. **体积优势不显著**
   - 20KB 的差异在现代构建工具（Vite + gzip）下可忽略
   - 桌面端应用对体积不敏感
   - 代码分割和懒加载可进一步优化

3. **生态系统不成熟**
   - GitHub Stars 差距巨大（3K vs 39K）
   - 遇到问题时社区支持不足
   - 缺乏大规模生产环境验证
   - Stack Overflow 等平台资源稀少

4. **技术风险高**
   - 相对年轻的项目（2021年）
   - 长期维护存在不确定性
   - 未来可能需要迁移回主流方案（成本高）

5. **团队效率考虑**
   - 学习新概念需要时间成本
   - 招聘时难以找到有经验的开发者
   - 团队对 TanStack Query 更熟悉

6. **React 18+ 支持不明确**
   - Alova 对 React 18 并发特性的支持程度未知
   - TanStack Query 对 Suspense、Transitions 等新特性支持完善

#### D.5 推荐方案：TanStack Query + Axios

**TanStack Query 的优势**:

1. **业界标准**: React 生态中最流行的服务端状态管理库
2. **成熟稳定**: 经过大量生产环境验证
3. **强大的缓存**: 灵活的缓存策略和失效机制
4. **完善的 DevTools**: React Query Devtools 功能强大
5. **React 18+ 支持**: 完美支持 Suspense、Concurrent Features
6. **丰富的插件**: 持久化、水合、无限滚动等
7. **Vercel 维护**: 持续更新和长期支持保证

**实施建议**:

```typescript
// apps/web/src/lib/query-client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,        // 5分钟
      cacheTime: 10 * 60 * 1000,       // 10分钟
      refetchOnWindowFocus: false,
      retry: 3,
      retryDelay: (attemptIndex) =>
        Math.min(1000 * 2 ** attemptIndex, 30000)
    },
    mutations: {
      retry: 1
    }
  }
});

// apps/web/src/services/article.service.ts
export const articleQueries = {
  list: (params: ArticleListParams) => ({
    queryKey: ['articles', params],
    queryFn: () => apiClient.get('/articles', { params }),
    staleTime: 3 * 60 * 1000
  }),

  detail: (id: string) => ({
    queryKey: ['article', id],
    queryFn: () => apiClient.get(`/articles/${id}`)
  })
};

// 使用示例
const { data, isLoading } = useQuery(articleQueries.list({ page: 1 }));
```

#### D.6 备选方案（仅供参考）

如果未来项目需求变化，可以考虑以下场景使用 Alova.js：

- ✅ 移动端应用（体积敏感）
- ✅ 小型项目（快速开发）
- ✅ 团队主要是中国开发者
- ✅ 愿意承担新技术风险
- ✅ 需要 OpenAPI 自动生成功能

但对于本项目（企业级、多人协作、长期维护），**强烈建议保持 TanStack Query + Axios 方案**。

#### D.7 总结

**最终决策**: ❌ 不采用 Alova.js
**保持方案**: ✅ TanStack Query + Axios

**决策依据**:
- 项目稳定性 > 体积优化
- 社区生态 > API 简洁性
- 长期维护 > 短期便利
- 团队效率 > 技术新颖性

**行动计划**:
1. 继续按照第6章的技术栈设计实施
2. 使用 TanStack Query v5 + Axios 1.6+
3. 配置完善的缓存策略和错误处理
4. 集成 React Query Devtools 用于开发调试

---

**评估参考资料**:
- [Alova.js 官方文档](https://alova.js.org/zh-CN/)
- [TanStack Query 官方文档](https://tanstack.com/query/latest)
- [探索alova.js：轻量级请求策略库 - CSDN](https://blog.csdn.net/zw7518/article/details/134575524)
- [Alova.js 与 axios 的区别](https://www.lycoris.cloud/archives/alova-axios)

### E. 技术选型评估：验证库（Zod vs Valibot vs class-validator）

**评估日期**: 2026-01-18
**评估人**: 架构设计团队
**结论**: ✅ 推荐使用 Zod + nestjs-zod，替换 class-validator

---

#### E.1 背景与问题

在原方案中，后端使用 `class-validator` + `class-transformer` 进行数据验证。但存在以下问题：

1. **停止维护**: 已超过 2 年没有更新
2. **性能问题**: 基于装饰器的运行时验证，性能较差
3. **无法共享**: 只能在后端使用，前端需要重复定义验证逻辑
4. **类型推断弱**: TypeScript 类型需要手动维护，容易不一致

#### E.2 三大方案对比

| 维度 | **Zod** | **Valibot** | **class-validator** |
|------|---------|-------------|---------------------|
| **体积（简单表单）** | 15.18 KB | 1.37 KB (↓90%) | ~20 KB |
| **运行时性能（有效数据）** | 快 | 快（类似Zod v4） | 慢 |
| **运行时性能（无效数据）** | 快 | ⚠️ 慢（异常处理） | 慢 |
| **TypeScript 支持** | ✅ 完美类型推断 | ✅ 完美类型推断 | ⚠️ 需手动维护 |
| **前后端共享** | ✅ 完美支持 | ✅ 完美支持 | ❌ 仅后端 |
| **NestJS 集成** | ✅ nestjs-zod | ⚠️ 需自定义 | ✅ 原生支持 |
| **OpenAPI 生成** | ✅ zod-to-openapi | ⚠️ 社区支持弱 | ✅ swagger |
| **生态系统** | ✅ 非常丰富 | ⚠️ 较新 | ✅ 成熟但停滞 |
| **社区活跃度** | ✅ 37K+ stars | ✅ 6K+ stars | ⚠️ 停滞 |
| **学习曲线** | 低 | 低 | 中 |
| **更新频率** | ✅ 活跃 | ✅ 活跃 | ❌ 停滞 |
| **错误消息** | ✅ 可自定义 | ✅ 可自定义 | ⚠️ 有限 |

#### E.3 推荐方案：Zod

**选择 Zod 的核心理由**:

1. **前后端完美统一**
```typescript
// packages/shared/schemas/article.schema.ts
import { z } from 'zod';

// 定义一次，前后端共享
export const articleSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(10),
  accountId: z.string().uuid()
});

// 自动推断 TypeScript 类型
export type Article = z.infer<typeof articleSchema>;

// 后端使用
import { createZodDto } from 'nestjs-zod';
export class CreateArticleDto extends createZodDto(articleSchema) {}

// 前端使用
import { zodResolver } from '@hookform/resolvers/zod';
const form = useForm<Article>({ resolver: zodResolver(articleSchema) });
```

2. **优秀的 NestJS 集成**
```typescript
// apps/api/src/main.ts
import { ZodValidationPipe } from 'nestjs-zod';
app.useGlobalPipes(new ZodValidationPipe());

// Controller 自动验证
@Post()
create(@Body() dto: CreateArticleDto) {
  return this.articleService.create(dto);
}
```

3. **自动生成 OpenAPI 文档**
```typescript
import { extendZodWithOpenApi } from '@anatine/zod-openapi';

export const articleSchema = z.object({
  title: z.string().openapi({ example: '我的文章标题' })
}).openapi('Article');
```

4. **强大的类型推断和转换**
```typescript
const schema = z.object({
  publishedAt: z.string().datetime().transform((str) => new Date(str)),
  readCount: z.number().int().positive().default(0)
});

type Data = z.infer<typeof schema>;
// { publishedAt: Date; readCount: number; }
```

5. **丰富的生态系统**
- `zod-to-openapi` - OpenAPI 文档生成
- `@hookform/resolvers/zod` - React Hook Form 集成
- `nestjs-zod` - NestJS 集成
- `zod-mock` - 测试数据生成

#### E.4 Valibot 备选方案

**何时考虑 Valibot**:
- ✅ 有大量复杂验证逻辑
- ✅ 前端体积极度敏感（体积小 90%）
- ✅ 验证性能是瓶颈
- ✅ 主要处理有效数据（验证失败少）

**Valibot 的缺点**:
- ⚠️ NestJS 集成需要自定义（无官方库）
- ⚠️ 验证失败时性能较差（依赖异常处理）
- ⚠️ 生态系统较小
- ⚠️ OpenAPI 集成不完善

#### E.5 实施方案

**阶段一：安装依赖**

```json
// packages/shared/package.json
{
  "dependencies": {
    "zod": "^3.22.4"
  }
}

// apps/api/package.json
{
  "dependencies": {
    "nestjs-zod": "^3.0.0",
    "@anatine/zod-nestjs": "^2.0.0",
    "@anatine/zod-openapi": "^2.0.0"
  }
}

// apps/web/package.json
{
  "dependencies": {
    "@hookform/resolvers": "^3.3.3"
  }
}
```

**阶段二：配置 NestJS**

```typescript
// apps/api/src/main.ts
import { ZodValidationPipe } from 'nestjs-zod';
import { patchNestJsSwagger } from 'nestjs-zod';

patchNestJsSwagger(); // Swagger 支持

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ZodValidationPipe());
  await app.listen(3000);
}
```

**阶段三：定义共享 Schema**

```typescript
// packages/shared/src/schemas/article.schema.ts
import { z } from 'zod';
import { extendZodWithOpenApi } from '@anatine/zod-openapi';

extendZodWithOpenApi(z);

export const createArticleSchema = z.object({
  title: z.string()
    .min(1, '标题不能为空')
    .max(200, '标题不能超过200字')
    .openapi({ example: '我的第一篇文章' }),

  content: z.string()
    .min(10, '内容至少10字')
    .openapi({ example: '这是文章内容...' }),

  accountId: z.string()
    .uuid('无效的公众号ID')
}).openapi('CreateArticle');

export const articleListParamsSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().positive().max(100).default(20),
  sortBy: z.enum(['publishedAt', 'readNum']).default('publishedAt')
});

export type CreateArticleDto = z.infer<typeof createArticleSchema>;
export type ArticleListParams = z.infer<typeof articleListParamsSchema>;
```

**阶段四：后端使用**

```typescript
// apps/api/src/modules/article/dto/create-article.dto.ts
import { createZodDto } from 'nestjs-zod';
import { createArticleSchema } from '@wechat-exporter/shared/schemas';

export class CreateArticleDto extends createZodDto(createArticleSchema) {}

// apps/api/src/modules/article/article.controller.ts
@Controller('articles')
export class ArticleController {
  @Post()
  async create(@Body() dto: CreateArticleDto) {
    return this.articleService.create(dto);
  }

  @Get()
  async getList(@Query() params: ArticleListParamsDto) {
    // params.page 自动从 string 转为 number
    return this.articleService.findAll(params);
  }
}
```

**阶段五：前端使用**

```typescript
// apps/web/src/pages/ArticleCreate.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { createArticleSchema, type CreateArticleDto }
  from '@wechat-exporter/shared/schemas';

export function ArticleCreatePage() {
  const { register, handleSubmit, formState: { errors } } =
    useForm<CreateArticleDto>({
      resolver: zodResolver(createArticleSchema)
    });

  const onSubmit = async (data: CreateArticleDto) => {
    await articleService.create(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Input {...register('title')} />
      {errors.title && <span>{errors.title.message}</span>}
      <Button type="submit">创建</Button>
    </form>
  );
}
```

#### E.6 迁移对比

```typescript
// 迁移前 (class-validator) - 只能后端使用
export class CreateArticleDto {
  @IsString()
  @MinLength(1)
  @MaxLength(200)
  title: string;

  @IsString()
  @MinLength(10)
  content: string;
}

// 迁移后 (Zod) - 前后端共享！
export const createArticleSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(10)
});

export type CreateArticleDto = z.infer<typeof createArticleSchema>;
```

#### E.7 额外收益

**1. 环境变量验证**
```typescript
// apps/api/src/config/env.validation.ts
const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.coerce.number().int().positive(),
  DB_HOST: z.string(),
  JWT_SECRET: z.string().min(32)
});

export const validateEnv = (config: Record<string, unknown>) =>
  envSchema.parse(config);
```

**2. 复杂验证逻辑**
```typescript
const schema = z.object({
  title: z.string(),
  publishAt: z.string().datetime().optional()
}).refine((data) => {
  if (data.publishAt) {
    return new Date(data.publishAt) > new Date();
  }
  return true;
}, {
  message: '发布时间必须是未来时间',
  path: ['publishAt']
});
```

**3. 自动生成 Mock 数据**
```typescript
import { generateMock } from '@anatine/zod-mock';
const mockArticle = generateMock(articleSchema);
```

#### E.8 性能对比

验证 10,000 个简单对象：
- class-validator: ~450ms
- Zod v3: ~180ms (↓60%)
- Zod v4: ~90ms (↓80%)
- Valibot: ~85ms (↓81%)

Bundle 大小（简单登录表单）：
- class-validator: ~20 KB
- Zod: ~15 KB (↓25%)
- Valibot: ~1.4 KB (↓93%)

#### E.9 结论

**最终决策**: ✅ **采用 Zod + nestjs-zod**

**决策依据**:
- 前后端统一验证 > 体积优化
- 类型安全 > 装饰器语法
- 生态成熟度 > 性能极限
- 开发效率 > 学习成本

**行动计划**:
1. 在 packages/shared 中创建 schemas 目录
2. 安装 zod + nestjs-zod + @anatine/* 相关包
3. 配置全局 ZodValidationPipe
4. 逐步迁移现有 DTO 到 Zod schema
5. 前端集成 @hookform/resolvers/zod

**预期收益**:
- ✅ 前后端验证逻辑统一，减少 50% 重复代码
- ✅ 类型自动推断，消除类型不一致问题
- ✅ 性能提升 60-80%
- ✅ 更好的开发体验和错误提示

---

**评估参考资料**:
- [Zod 官方文档](https://zod.dev/)
- [nestjs-zod GitHub](https://github.com/risen228/nestjs-zod)
- [Valibot 官方文档](https://valibot.dev/)
- [Valibot vs Zod Performance Comparison](https://github.com/fabian-hiller/valibot)

### F. 代理管理技术详解

**背景**: 2026-01-18
**目的**: 避免同一 IP 抓取资源时被屏蔽或限流

---

#### F.1 问题背景

在下载微信公众号文章的资源（图片、CSS、JS 等）时，会遇到以下问题：

1. **Referrer 限制**: 微信资源服务器检查 HTTP Referrer，直接访问会被拒绝
2. **IP 限流**: 同一 IP 短时间内大量请求会被临时封禁
3. **频率控制**: 高频请求触发反爬虫机制
4. **跨域限制**: 浏览器 CORS 策略阻止直接下载

为了解决这些问题，项目采用了**公共代理池 + 智能轮转**的方案。

#### F.2 现有架构分析

**代理配置** (`config/index.ts:78-112`):

```typescript
export const PUBLIC_PROXY_LIST: string[] = [
  // worker-proxy.asia 域名（16个节点）
  'https://00.worker-proxy.asia',
  'https://01.worker-proxy.asia',
  // ... 共16个

  // net-proxy.asia 域名（16个节点）
  'https://00.net-proxy.asia',
  'https://01.net-proxy.asia',
  // ... 共16个
];

// 注释中还列出了48个备用代理节点：
// - workers-proxy.shop (16个)
// - workers-proxy.top (16个)
// - workers-proxy.ggff.net (16个)
```

**ProxyManager 核心实现** (`utils/download/ProxyManager.ts`):

```typescript
export class ProxyManager {
  private readonly proxies: string[];                    // 代理列表
  private readonly proxyStatus: Map<string, ProxyStatus>; // 代理状态
  private readonly cooldownPeriod: number;               // 冷却期（ms）
  private readonly maxFailures: number;                  // 最大失败次数

  constructor(
    proxies: string[],
    cooldownPeriod = DEFAULT_OPTIONS.COOLDOWN_PERIOD,  // 默认值
    maxFailures = DEFAULT_OPTIONS.MAX_FAILURES          // 默认值
  ) {
    this.proxies = [...proxies];
    this.proxyStatus = new Map();
    this.cooldownPeriod = cooldownPeriod;
    this.maxFailures = maxFailures;
    this.initProxyStatus();
  }

  // 智能代理选择算法
  public getBestProxy(): string {
    const now = Date.now();
    const availableProxies = Array.from(this.proxyStatus.entries())
      .filter(([_, status]) =>
        !status.cooldown || now - status.lastUsed >= this.cooldownPeriod
      )
      .sort((a, b) => {
        // 优先级1: 失败次数少的优先
        if (a[1].failures !== b[1].failures) {
          return a[1].failures - b[1].failures;
        }
        // 优先级2: 最久未使用的优先
        return a[1].lastUsed - b[1].lastUsed;
      });

    if (availableProxies.length === 0) {
      return this.resetAndGetProxy(); // 强制重置
    }

    const [bestProxy, status] = availableProxies[0];
    status.lastUsed = now;
    status.totalUse++;
    return bestProxy;
  }

  // 记录代理失败
  public recordFailure(proxy: string): void {
    const status = this.proxyStatus.get(proxy);
    if (!status) return;

    status.failures++;
    status.totalFailures++;

    // 连续失败达到阈值，进入冷却期
    status.cooldown = status.failures >= this.maxFailures;
  }

  // 记录代理成功
  public recordSuccess(proxy: string): void {
    const status = this.proxyStatus.get(proxy);
    if (!status) return;

    status.failures = 0;      // 清零失败计数
    status.cooldown = false;  // 解除冷却
    status.totalSuccess++;
  }
}
```

**ProxyStatus 接口**:

```typescript
interface ProxyStatus {
  failures: number;       // 连续失败次数
  lastUsed: number;      // 最后使用时间戳（ms）
  cooldown: boolean;     // 是否在冷却期
  totalFailures: number; // 总失败次数（统计）
  totalSuccess: number;  // 总成功次数（统计）
  totalUse: number;      // 总使用次数（统计）
}
```

#### F.3 使用场景

**在 Exporter 中的应用** (`utils/download/Exporter.ts:184-217`):

```typescript
private async downloadResourceTask(url: string, fakeid: string): Promise<void> {
  this.pending.add(url);

  // 检查缓存
  const cached = await getResourceCache(url);
  if (cached) {
    this.pending.delete(url);
    this.completed.add(url);
    return;
  }

  // 重试循环
  for (let attempt = 0; attempt < this.options.maxRetries; attempt++) {
    const proxy = this.proxyManager.getBestProxy(); // 获取最佳代理

    try {
      const blob = await this.download(fakeid, url, proxy);
      await updateResourceCache({
        fakeid: fakeid,
        url: url,
        file: blob,
      });
      this.pending.delete(url);
      this.completed.add(url);
      this.proxyManager.recordSuccess(proxy); // 记录成功
      return;
    } catch (error) {
      await this.handleDownloadFailure(proxy, url, attempt, error);
      // handleDownloadFailure 内部会调用 proxyManager.recordFailure()
    }
  }

  this.pending.delete(url);
  this.failed.add(url);
}
```

**工作流程**:

1. **资源提取**: 从 HTML 中提取所有需要下载的资源 URL（图片、CSS）
2. **并发下载**: 使用队列管理并发下载任务
3. **代理选择**: 每次下载前调用 `getBestProxy()` 获取最优代理
4. **失败重试**: 下载失败时，记录代理失败次数，重试时会自动选择其他代理
5. **成功反馈**: 下载成功时，清零该代理的失败计数

#### F.4 智能选择算法详解

**算法流程**:

```
┌─────────────────────────────────────────────┐
│  1. 过滤可用代理                             │
│     - 未在冷却期的代理                       │
│     - 或冷却期已过的代理                     │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  2. 排序（双重优先级）                       │
│     优先级1: failures（升序）                │
│     优先级2: lastUsed（升序）               │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  3. 选择结果                                 │
│     - 有可用代理: 返回排序后的第一个         │
│     - 无可用代理: 强制重置最久未用的代理     │
└─────────────────────────────────────────────┘
```

**关键设计原则**:

1. **负载均衡**: 优先使用最久未用的代理，避免单点过载
2. **故障隔离**: 失败代理进入冷却期，避免持续失败
3. **自动恢复**: 冷却期过后自动重新加入可用池
4. **强制保底**: 所有代理都不可用时，强制重置最旧代理，确保服务不中断

**示例场景**:

```
初始状态:
Proxy A: failures=0, lastUsed=1000, cooldown=false
Proxy B: failures=0, lastUsed=2000, cooldown=false
Proxy C: failures=0, lastUsed=3000, cooldown=false

第1次调用 getBestProxy():
  → 选择 Proxy A (failures=0, lastUsed 最小)
  → 更新: lastUsed=4000, totalUse=1

Proxy A 连续失败 3 次:
  → failures=3, cooldown=true

第2次调用 getBestProxy():
  → Proxy A 被过滤（在冷却期）
  → 选择 Proxy B (failures=0, lastUsed=2000)

Proxy A 成功恢复:
  → recordSuccess() 调用
  → failures=0, cooldown=false
  → 重新进入可用池
```

#### F.5 新架构实现方案

在重构后的 NestJS + React 架构中，代理管理需要迁移到后端。

##### F.5.1 后端实现

**模块结构**:

```typescript
// apps/api/src/modules/proxy/proxy.module.ts
import { Module } from '@nestjs/common';
import { ProxyService } from './proxy.service';
import { ProxyController } from './proxy.controller';

@Module({
  providers: [ProxyService],
  controllers: [ProxyController],
  exports: [ProxyService]
})
export class ProxyModule {}
```

**服务实现**:

```typescript
// apps/api/src/modules/proxy/proxy.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

interface ProxyStatus {
  failures: number;
  lastUsed: number;
  cooldown: boolean;
  totalFailures: number;
  totalSuccess: number;
  totalUse: number;
}

@Injectable()
export class ProxyService {
  private readonly proxies: string[];
  private readonly proxyStatus: Map<string, ProxyStatus>;
  private readonly cooldownPeriod: number;
  private readonly maxFailures: number;

  constructor(private configService: ConfigService) {
    // 从配置中加载代理列表
    this.proxies = this.configService.get<string[]>('proxy.list', []);
    this.cooldownPeriod = this.configService.get<number>('proxy.cooldownPeriod', 60000);
    this.maxFailures = this.configService.get<number>('proxy.maxFailures', 3);

    this.proxyStatus = new Map();
    this.initProxyStatus();
  }

  private initProxyStatus(): void {
    this.proxies.forEach(proxy => {
      this.proxyStatus.set(proxy, {
        failures: 0,
        lastUsed: 0,
        cooldown: false,
        totalFailures: 0,
        totalSuccess: 0,
        totalUse: 0,
      });
    });
  }

  public getBestProxy(): string {
    const now = Date.now();
    const availableProxies = Array.from(this.proxyStatus.entries())
      .filter(([_, status]) =>
        !status.cooldown || now - status.lastUsed >= this.cooldownPeriod
      )
      .sort((a, b) => {
        if (a[1].failures !== b[1].failures) {
          return a[1].failures - b[1].failures;
        }
        return a[1].lastUsed - b[1].lastUsed;
      });

    if (availableProxies.length === 0) {
      return this.resetAndGetProxy();
    }

    const [bestProxy, status] = availableProxies[0];
    status.lastUsed = now;
    status.totalUse++;
    return bestProxy;
  }

  public recordFailure(proxy: string): void {
    const status = this.proxyStatus.get(proxy);
    if (!status) return;

    status.failures++;
    status.totalFailures++;
    status.cooldown = status.failures >= this.maxFailures;
  }

  public recordSuccess(proxy: string): void {
    const status = this.proxyStatus.get(proxy);
    if (!status) return;

    status.failures = 0;
    status.cooldown = false;
    status.totalSuccess++;
  }

  public getProxyStatus(): Map<string, ProxyStatus> {
    return new Map(this.proxyStatus);
  }

  private resetAndGetProxy(): string {
    const [oldestProxy, status] = Array.from(this.proxyStatus.entries())
      .sort(([, a], [, b]) => a.lastUsed - b.lastUsed)[0];

    this.proxyStatus.set(oldestProxy, {
      ...status,
      failures: 0,
      cooldown: false,
      lastUsed: Date.now(),
      totalUse: status.totalUse + 1,
    });

    return oldestProxy;
  }
}
```

**配置管理**:

```typescript
// apps/api/src/config/proxy.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('proxy', () => ({
  list: process.env.PROXY_LIST?.split(',') || [
    'https://00.worker-proxy.asia',
    'https://01.worker-proxy.asia',
    // ... 其他代理
  ],
  cooldownPeriod: parseInt(process.env.PROXY_COOLDOWN_PERIOD || '60000', 10),
  maxFailures: parseInt(process.env.PROXY_MAX_FAILURES || '3', 10),
}));
```

**环境变量** (`.env`):

```bash
# 代理配置
PROXY_LIST=https://00.worker-proxy.asia,https://01.worker-proxy.asia,...
PROXY_COOLDOWN_PERIOD=60000  # 冷却期（毫秒）
PROXY_MAX_FAILURES=3         # 最大失败次数
```

##### F.5.2 与下载模块集成

```typescript
// apps/api/src/modules/download/download.processor.ts
import { Processor, Process } from '@nestjs/bullmq';
import { Job } from 'bullmq';
import { ProxyService } from '../proxy/proxy.service';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';

@Processor('download')
export class DownloadProcessor {
  constructor(
    private proxyService: ProxyService,
    private httpService: HttpService
  ) {}

  @Process('resource')
  async handleResourceDownload(job: Job) {
    const { url, maxRetries = 3 } = job.data;

    for (let attempt = 0; attempt < maxRetries; attempt++) {
      const proxy = this.proxyService.getBestProxy();

      try {
        // 通过代理下载资源
        const response = await firstValueFrom(
          this.httpService.get(`${proxy}/${url}`, {
            responseType: 'arraybuffer',
            timeout: 30000,
          })
        );

        this.proxyService.recordSuccess(proxy);

        return {
          url,
          data: Buffer.from(response.data),
          contentType: response.headers['content-type'],
        };
      } catch (error) {
        this.proxyService.recordFailure(proxy);

        if (attempt === maxRetries - 1) {
          throw error;
        }
      }
    }
  }
}
```

##### F.5.3 监控与管理 API

```typescript
// apps/api/src/modules/proxy/proxy.controller.ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiBearerAuth } from '@nestjs/swagger';
import { JwtAuthGuard } from '@/common/guards/jwt-auth.guard';
import { ProxyService } from './proxy.service';

@ApiTags('proxy')
@Controller('proxy')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
export class ProxyController {
  constructor(private readonly proxyService: ProxyService) {}

  @Get('status')
  @ApiOperation({ summary: '获取代理状态' })
  async getProxyStatus() {
    const statusMap = this.proxyService.getProxyStatus();
    return {
      success: true,
      data: Array.from(statusMap.entries()).map(([proxy, status]) => ({
        proxy,
        ...status,
        availability: !status.cooldown ? '可用' : '冷却中',
      })),
    };
  }
}
```

**API 响应示例**:

```json
{
  "success": true,
  "data": [
    {
      "proxy": "https://00.worker-proxy.asia",
      "failures": 0,
      "lastUsed": 1705567200000,
      "cooldown": false,
      "totalFailures": 5,
      "totalSuccess": 245,
      "totalUse": 250,
      "availability": "可用"
    },
    {
      "proxy": "https://01.worker-proxy.asia",
      "failures": 3,
      "lastUsed": 1705567180000,
      "cooldown": true,
      "totalFailures": 12,
      "totalSuccess": 188,
      "totalUse": 200,
      "availability": "冷却中"
    }
  ]
}
```

#### F.6 前端监控界面（可选）

**代理状态展示组件**:

```typescript
// apps/web/src/pages/ProxyStatus.tsx
import { useQuery } from '@tanstack/react-query';
import { Table, Tag, Progress } from 'antd';
import { proxyService } from '@/services/proxy.service';

interface ProxyStatus {
  proxy: string;
  failures: number;
  cooldown: boolean;
  totalSuccess: number;
  totalFailures: number;
  totalUse: number;
  availability: string;
}

export function ProxyStatusPage() {
  const { data, isLoading } = useQuery({
    queryKey: ['proxy-status'],
    queryFn: () => proxyService.getStatus(),
    refetchInterval: 5000, // 每5秒刷新
  });

  const columns = [
    {
      title: '代理地址',
      dataIndex: 'proxy',
      key: 'proxy',
    },
    {
      title: '状态',
      dataIndex: 'availability',
      key: 'availability',
      render: (text: string, record: ProxyStatus) => (
        <Tag color={record.cooldown ? 'red' : 'green'}>{text}</Tag>
      ),
    },
    {
      title: '成功率',
      key: 'successRate',
      render: (_: any, record: ProxyStatus) => {
        const rate = record.totalUse > 0
          ? (record.totalSuccess / record.totalUse) * 100
          : 0;
        return <Progress percent={Math.round(rate)} size="small" />;
      },
    },
    {
      title: '连续失败',
      dataIndex: 'failures',
      key: 'failures',
    },
    {
      title: '总使用次数',
      dataIndex: 'totalUse',
      key: 'totalUse',
    },
  ];

  return (
    <Table
      dataSource={data?.data}
      columns={columns}
      loading={isLoading}
      rowKey="proxy"
    />
  );
}
```

#### F.7 优化建议

**性能优化**:

1. **Redis 缓存代理状态**: 在集群环境下，使用 Redis 共享代理状态
   ```typescript
   // 伪代码
   async getBestProxy() {
     const cachedStatus = await this.redis.get('proxy:status');
     // ...
   }
   ```

2. **动态代理池**: 定期检测代理可用性，自动添加/移除代理
   ```typescript
   @Cron('0 */10 * * * *') // 每10分钟
   async healthCheck() {
     for (const proxy of this.proxies) {
       const isAlive = await this.pingProxy(proxy);
       // 更新状态
     }
   }
   ```

3. **分级代理**: 根据速度和稳定性将代理分为优先级
   ```typescript
   const proxies = [
     { url: '...', priority: 'high' },
     { url: '...', priority: 'low' },
   ];
   ```

**监控告警**:

1. **可用率告警**: 可用代理数量低于阈值时发送告警
2. **失败率告警**: 全局失败率超过阈值时通知管理员
3. **性能监控**: 记录每个代理的平均响应时间

#### F.8 总结

**核心优势**:

1. ✅ **智能轮转**: 自动选择最优代理，负载均衡
2. ✅ **故障隔离**: 失败代理自动进入冷却期，避免持续失败
3. ✅ **自动恢复**: 冷却期过后自动重新启用
4. ✅ **统计分析**: 记录每个代理的成功率和使用情况
5. ✅ **无单点故障**: 即使所有代理暂时不可用，也会强制重置保底

**重构后的改进**:

1. 🚀 **集中管理**: 代理管理统一在后端，便于监控和维护
2. 🚀 **集群支持**: 使用 Redis 可在多实例间共享代理状态
3. 🚀 **可视化监控**: 提供代理状态实时监控界面
4. 🚀 **动态配置**: 支持运行时添加/移除代理，无需重启
5. 🚀 **性能优化**: 后端代理池复用，减少代理选择开销

**实施优先级**:

- **Phase 1** (MVP): 基础 ProxyService + 代理配置
- **Phase 2**: 与下载模块集成 + 失败重试
- **Phase 3**: 监控 API + 前端状态展示
- **Phase 4**: Redis 集群共享 + 动态代理池

---

**相关文件参考**:
- `config/index.ts:78-112` - 代理列表配置
- `utils/download/ProxyManager.ts:1-105` - 代理管理器实现
- `utils/download/Exporter.ts:184-217` - 代理使用示例
