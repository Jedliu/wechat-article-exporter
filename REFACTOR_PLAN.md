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
| **验证** | class-validator + class-transformer | 装饰器语法，与NestJS集成 |
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
    "class-validator": "^0.14.0",
    "class-transformer": "^0.5.1",
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
