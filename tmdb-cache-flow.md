# TMDB 元数据缓存流程分析

## 1. 整体架构概览

Jellyseerr 采用**双层缓存架构**：前端 SWR 缓存 + 服务端 node-cache 缓存，用于减少对 TMDB API 的直接请求。

```
┌─────────────────┐
│  前端浏览器     │
│  (SWR 缓存)    │
└────────┬────────┘
         │ HTTP 请求
         ▼
┌─────────────────┐
│  服务端 Express │
│  (node-cache)   │
└────────┬────────┘
         │ 缓存未命中
         ▼
┌─────────────────┐
│  TMDB API       │
└─────────────────┘
```

---

## 2. 服务端缓存核心实现

### 2.1 缓存管理器 (CacheManager)

**文件**: `server/lib/cache.ts`

采用单例模式管理多个缓存实例，每个外部 API 对应一个独立缓存：

```typescript
class CacheManager {
  private availableCaches: Record<AvailableCacheIds, Cache> = {
    tmdb: new Cache('tmdb', 'The Movie Database API', {
      stdTtl: 21600,        // 默认 6 小时
      checkPeriod: 60 * 30, // 每 30 分钟检查过期
    }),
    // ... 其他缓存
  };
}
```

**TMDB 缓存配置**：
- 默认 TTL：21600 秒（6 小时）
- 过期检查周期：1800 秒（30 分钟）
- 底层实现：`node-cache` 内存缓存库

### 2.2 缓存 Key 生成

**文件**: `server/api/externalapi.ts:146-155`

缓存 Key 由基础 URL + 端点 + 参数序列化组成：

```typescript
private serializeCacheKey(
  endpoint: string,
  options?: Record<string, unknown>
) {
  if (!options) {
    return `${this.baseUrl}${endpoint}`;
  }
  return `${this.baseUrl}${endpoint}${JSON.stringify(options)}`;
}
```

**示例 Key**：
```
https://api.themoviedb.org/3/discover/movie{"page":1,"sort_by":"popularity.desc","language":"en"}
```

> **注意**：params 和 headers 都会参与 Key 生成，因此不同语言/地区的请求会有独立的缓存条目。

---

## 3. 请求合并（去重）机制

### 3.1 前端层面：SWR Deduplication

**文件**: `src/hooks/useDiscover.ts:92`

前端使用 SWR (stale-while-revalidate) 进行请求去重：

```typescript
const { data, error, size, setSize, isValidating, mutate } = useSWRInfinite(
  getKey,
  {
    initialSize: 3,
    revalidateFirstPage: false,
    dedupingInterval: 30000, // 30 秒内相同 key 的请求会被合并
    revalidateOnFocus: false,
  }
);
```

**工作原理**：
- `dedupingInterval: 30000`：30 秒内，相同 key 的多个请求会被合并为一个
- 适用于同一页面内多个组件发起相同请求的场景
- 避免重复的网络请求和渲染

### 3.2 服务端层面：无显式请求合并

**重要发现**：服务端 `ExternalAPI` 的 `get()` 方法**没有**实现并发请求合并（in-flight request deduplication）。

```typescript
// server/api/externalapi.ts:56-77
protected async get<T>(
  endpoint: string,
  config?: AxiosRequestConfig,
  ttl?: number
): Promise<T> {
  const cacheKey = this.serializeCacheKey(endpoint, { ... });
  const cachedItem = this.cache?.get<T>(cacheKey);
  
  if (cachedItem) {
    return cachedItem; // 缓存命中，直接返回
  }

  // ❌ 缓存未命中时，所有并发请求都会直接发送到 TMDB
  const response = await this.axios.get<T>(endpoint, config);

  if (this.cache && ttl !== 0) {
    this.cache.set(cacheKey, response.data, ttl ?? DEFAULT_TTL);
  }

  return response.data;
}
```

**后果**：如果 N 个并发请求同时到达且缓存未命中，会同时向 TMDB 发送 N 个请求，然后 N 次写入缓存（最后一次写入生效）。

---

## 4. 过期判断策略

### 4.1 标准 TTL 过期

默认使用标准 TTL（Time-To-Live）策略，缓存到期后自动失效：

| 数据类型 | TTL (秒) | TTL (时长) | 代码位置 |
|---------|---------|-----------|---------|
| 默认 (DEFAULT_TTL) | 300 | 5 分钟 | `externalapi.ts:8` |
| TMDB 列表/搜索 | 21600 | 6 小时 | `cache.ts:47` |
| 电影/剧集详情 | 43200 | 12 小时 | `themoviedb/index.ts:296` |
| 地区/语言配置 | 86400 | 24 小时 | `themoviedb/index.ts:1009` |
| 关键词详情 | 604800 | 7 天 | `themoviedb/index.ts:1215` |
| 认证分级 | 604800 | 7 天 | `themoviedb/index.ts:1178` |

**示例 - 电影详情接口**：
```typescript
// server/api/themoviedb/index.ts:277-296
public getMovie = async ({ movieId, language }) => {
  const data = await this.get<TmdbMovieDetails>(
    `/movie/${movieId}`,
    { params: { ... } },
    43200 // 显式指定 12 小时 TTL
  );
  return data;
};
```

### 4.2 滚动刷新 (Rolling Refresh)

**文件**: `server/api/externalapi.ts:104-137`

对于某些接口（如 Sonarr/Radarr 配置、Plex TV 元数据），使用滚动刷新策略：

```typescript
protected async getRolling<T>(
  endpoint: string,
  config?: AxiosRequestConfig,
  ttl?: number
): Promise<T> {
  const cacheKey = this.serializeCacheKey(...);
  const cachedItem = this.cache?.get<T>(cacheKey);

  if (cachedItem) {
    const keyTtl = this.cache?.getTtl(cacheKey) ?? 0;

    // 当缓存即将过期时（剩余时间 < 10 秒），后台异步刷新
    if (
      keyTtl - (ttl ?? DEFAULT_TTL) * 1000 <
      Date.now() - DEFAULT_ROLLING_BUFFER // DEFAULT_ROLLING_BUFFER = 10000ms
    ) {
      this.axios.get<T>(endpoint, config).then((response) => {
        this.cache?.set(cacheKey, response.data, ttl ?? DEFAULT_TTL);
      });
    }
    return cachedItem; // 立即返回旧数据，不阻塞
  }

  // 缓存未命中，正常请求
  const response = await this.axios.get<T>(endpoint, config);
  if (this.cache && ttl !== 0) {
    this.cache.set(cacheKey, response.data, ttl ?? DEFAULT_TTL);
  }
  return response.data;
}
```

**核心特点**：
- **先返回旧数据**：保证响应速度
- **后台刷新**：缓存接近过期时（过期前 10 秒），异步发起新请求
- **用户无感知**：下次请求时获得新数据

> **注意**：TMDB 的主要接口（发现、搜索、详情）都使用标准 TTL 而非滚动刷新。滚动刷新主要用于 Servarr 配置和 Plex 元数据。

### 4.3 过期检查机制

`node-cache` 的 `checkperiod` 配置决定了主动清理过期条目的频率：
- TMDB 缓存：每 30 分钟检查一次
- 检查通过 `setInterval` 实现，遍历所有 key 并删除已过期的

---

## 5. 缓存命中链路

### 5.1 前端列表页链路（以发现电影为例）

```
前端组件 (DiscoverMovies)
        │
        ▼
useDiscover Hook (useSWRInfinite)
        │
        ├─ 命中 SWR 前端缓存？→ 直接返回，结束
        │
        ▼ 未命中
axios GET /api/v1/discover/movies?page=1&sortBy=popularity.desc
        │
        ▼
Express 路由 (server/routes/discover.ts:97)
        │
        ▼
createTmdbWithRegionLanguage() → 创建 TheMovieDb 实例
        │
        ▼
tmdb.getDiscoverMovies()
        │
        ▼
ExternalAPI.get()
        │
        ├─ 生成 cacheKey
        ├─ cache.get(cacheKey)
        │    ├─ 命中 → 直接返回缓存数据 ✓
        │    └─ 未命中 → 继续
        │
        ▼ 未命中
axios GET https://api.themoviedb.org/3/discover/movie
        │
        ▼
TMDB 返回数据
        │
        ▼
cache.set(cacheKey, data, ttl) → 写入缓存
        │
        ▼
返回前端
```

### 5.2 详情页链路

**文件**: `src/pages/movie/[movieId]/index.tsx` (SSR)

详情页使用 Next.js 的 `getServerSideProps` 进行服务端渲染：

```typescript
export const getServerSideProps: GetServerSideProps = async (ctx) => {
  // 服务端直接调用本地 API
  const response = await axios.get<MovieDetailsType>(
    `http://localhost:5055/api/v1/movie/${ctx.query.movieId}`,
    { headers: { cookie: ctx.req.headers.cookie } }
  );

  return { props: { movie: response.data } };
};
```

**服务端处理** (`server/routes/movie.ts:16`):
```typescript
movieRoutes.get('/:id', async (req, res) => {
  const tmdb = new TheMovieDb();
  const tmdbMovie = await tmdb.getMovie({
    movieId: Number(req.params.id),
    language: req.locale,
  });
  // ... 映射数据并返回
});
```

### 5.3 列表与详情的缓存关系

**关键结论：列表页和详情页不共享缓存**

| 页面 | API 端点 | TMDB 接口 | 缓存 Key |
|-----|---------|----------|---------|
| 发现列表 | `/api/v1/discover/movies` | `GET /discover/movie` | 包含分页、排序等参数 |
| 电影详情 | `/api/v1/movie/:id` | `GET /movie/:id` | 包含 movieId，带 append_to_response |

**原因**：
1. TMDB API 端点不同：`/discover/movie` vs `/movie/{id}`
2. 返回数据结构不同：列表是精简信息，详情包含完整元数据（credits, videos, keywords 等）
3. 缓存 Key 完全不同，无法复用

> **设计考量**：列表页数据量小但访问频繁，详情页数据量大但访问相对较少。分开缓存有利于控制内存使用。

### 5.4 跨页面数据复用

前端 SWR 缓存可以在**相同 API 端点**的页面间复用：

- ✅ 从「发现电影第1页」跳转到「发现电影第2页」再返回：第1页数据在 SWR 缓存中
- ✅ 从「搜索结果」点击详情再返回：搜索结果列表在 SWR 缓存中
- ❌ 从「列表卡片」进入「详情页」：详情数据需要重新请求（服务端缓存可命中）

---

## 6. 代码位置速查表

| 功能 | 文件路径 | 关键行 |
|-----|---------|-------|
| 缓存管理器 | `server/lib/cache.ts` | 45-87 |
| ExternalAPI 基类 | `server/api/externalapi.ts` | 全部 |
| 标准 GET 缓存 | `server/api/externalapi.ts` | 56-77 |
| 滚动刷新 GET | `server/api/externalapi.ts` | 104-137 |
| 缓存 Key 生成 | `server/api/externalapi.ts` | 146-155 |
| TMDB API 封装 | `server/api/themoviedb/index.ts` | 127-1363 |
| 发现路由 | `server/routes/discover.ts` | 97-182 |
| 电影详情路由 | `server/routes/movie.ts` | 16-57 |
| 搜索路由 | `server/routes/search.ts` | 11-61 |
| 前端 useDiscover | `src/hooks/useDiscover.ts` | 54-173 |
| SWR 全局配置 | `src/pages/_app.tsx` | 191-198 |
| 电影详情页 (SSR) | `src/pages/movie/[movieId]/index.tsx` | 14-33 |

---

## 7. 失效驱动批量重建（Invalidations）

### 7.1 缓存失效的触发方式

Jellyseerr 的缓存失效有**三种方式**：

| 方式 | 触发场景 | 粒度 | 代码位置 |
|-----|---------|-----|---------|
| TTL 自动过期 | 时间到期 | 单条 Key | `node-cache` 内置 |
| 手动全量 Flush | 管理员在设置页操作 | 整个缓存 | `settings/index.ts:781-793` |
| 定时清理过期 | `image-cache-cleanup` 定时任务 | 过期图片 | `schedule.ts:227-244` |

### 7.2 手动全量 Flush（管理员操作）

**文件**: `server/routes/settings/index.ts:781-793`

管理员在「Settings → Jobs & Cache」页面点击「Flush Cache」按钮触发：

```typescript
settingsRoutes.post<{ cacheId: AvailableCacheIds }>(
  '/cache/:cacheId/flush',
  (req, res, next) => {
    const cache = cacheManager.getCache(req.params.cacheId);

    if (cache) {
      cache.flush();  // 调用 node-cache 的 flushAll()
      return res.status(204).send();
    }
    next({ status: 404, message: 'Cache not found.' });
  }
);
```

**缓存管理器 Flush 实现** (`server/lib/cache.ts:40-42`)：
```typescript
public flush(): void {
  this.data.flushAll();  // 清空该缓存实例的所有 Key
}
```

> **注意**：这是**全量清空**，不支持按特定 movieId/tvId 做精确失效。清空后所有请求都会穿透到 TMDB。

### 7.3 图片缓存的定时清理

**文件**: `server/job/schedule.ts:227-244`

图片缓存不使用 node-cache，而是直接存文件系统。通过定时任务清理过期图片：

```typescript
// 每 24 小时执行一次
scheduledJobs.push({
  id: 'image-cache-cleanup',
  name: 'Image Cache Cleanup',
  type: 'process',
  interval: 'hours',
  cronSchedule: jobs['image-cache-cleanup'].schedule,
  job: schedule.scheduleJob(jobs['image-cache-cleanup'].schedule, () => {
    ImageProxy.clearCache('tmdb');     // 清理 TMDB 过期图片
    ImageProxy.clearCache('avatar');   // 清理用户头像过期图片
  }),
});
```

**清理逻辑** (`server/lib/imageproxy.ts:28-69`)：
```typescript
public static async clearCache(key: string) {
  const cacheDirectory = path.join(baseCacheDirectory, key);
  const files = await promises.readdir(cacheDirectory);

  for (const file of files) {
    // 文件名格式: {maxAge}.{expireAt}.{etag}.{extension}
    const [, expireAtSt] = imageFile.split('.');
    const expireAt = Number(expireAtSt);
    const now = Date.now();

    if (now > expireAt) {
      await promises.rm(path.join(filePath), { recursive: true });
      deletedImages += 1;
    }
  }
}
```

### 7.4 头像变更触发的精确失效

**文件**: `server/routes/avatarproxy.ts:52-116`

用户头像缓存支持**精确失效**（基于版本号对比）：

```typescript
export async function checkAvatarChanged(user: User) {
  // 1. 从 Jellyfin/Emby 获取远端 Last-Modified / ETag
  const headResponse = await axios.head(jellyfinAvatarUrl);
  const remoteVersion = ...;  // Last-Modified 或 ETag

  // 2. 与数据库中的 avatarVersion 对比
  if (user.avatarVersion && user.avatarVersion === remoteVersion) {
    return { changed: false };  // 未变更，跳过
  }

  // 3. 版本不一致 → 精确删除该用户的头像缓存
  const avatarImageCache = await initAvatarImageProxy();
  await avatarImageCache.clearCachedImage(jellyfinAvatarUrl);  // 精确失效

  // 4. 重新拉取并缓存新头像
  const imageData = await avatarImageCache.getImage(jellyfinAvatarUrl, fallbackUrl);

  // 5. 更新数据库中的版本号和哈希
  user.avatarVersion = remoteVersion;
  user.avatarETag = computeImageHash(imageData.imageBuffer);
  await getRepository(User).save(user);

  return { changed: true, etag: newHash };
}
```

**精确失效的删除逻辑** (`server/lib/imageproxy.ts:187-226`)：
```typescript
public async clearCachedImage(path: string) {
  const cacheKey = this.getCacheKey(path);  // 通过 URL 反推 cacheKey
  const directory = join(this.getCacheDirectory(), cacheKey);
  await promises.rm(directory, { recursive: true });  // 删除该图片的目录
}
```

### 7.5 TMDB API 数据：无精确失效

**重要发现**：TMDB 的电影/剧集元数据（非图片）**没有精确失效机制**。

- 不支持按 `tmdbId` 单条刷新
- 也没有在媒体请求被批准/完成后触发失效
- **唯一方式**：等 TTL 自然过期，或管理员手动全量 Flush

**设计考量**：TMDB 元数据变更频率低（电影详情基本不变），TTL 6-12 小时足够。精确失效需要维护 Key → tmdbId 的反向索引，实现复杂收益低。

---

## 8. 本地缓存与持久化存储的共存

### 8.1 四层存储体系

Jellyseerr 实际采用**四层存储**，而非用户提到的 SQLite + Redis：

```
┌──────────────────────────────────────────┐
│  Layer 1: 前端 SWR 内存缓存              │  ← 浏览器内存
├──────────────────────────────────────────┤
│  Layer 2: 服务端 node-cache 内存缓存     │  ← Node.js 进程内存
├──────────────────────────────────────────┤
│  Layer 3: 文件系统图片缓存               │  ← CONFIG_DIRECTORY/cache/images/
├──────────────────────────────────────────┤
│  Layer 4: SQLite / PostgreSQL 数据库     │  ← CONFIG_DIRECTORY/db/db.sqlite3
└──────────────────────────────────────────┘
```

> **关键澄清**：项目**不使用 Redis**。`pnpm-lock.yaml` 中仅有 Redis 相关的间接依赖引用，代码中没有任何 Redis 客户端。

### 8.2 各层分工

| 层级 | 存储介质 | 存储内容 | 生命周期 |
|-----|---------|---------|---------|
| SWR 缓存 | 浏览器内存 | API 响应 JSON | 会话级别，可配置 dedupingInterval |
| node-cache | Node 进程内存 | TMDB/Servarr/Plex API JSON 响应 | TTL 到期，或进程重启丢失 |
| 文件系统 | 磁盘文件 | TMDB 图片、用户头像 | 文件名编码 expireAt，定时任务清理 |
| SQLite/PG | 磁盘数据库 | 用户、请求、媒体、配置等业务数据 | 永久，进程重启不丢失 |

### 8.3 SQLite / PostgreSQL 的角色

**文件**: `server/datasource.ts:50-147`

数据库**不存 TMDB 元数据缓存**，只存业务数据：

```typescript
// 默认使用 SQLite
const devConfig: DataSourceOptions = {
  type: 'sqlite',
  database: `${process.env.CONFIG_DIRECTORY}/db/db.sqlite3`,
  enableWAL: true,  // 启用 WAL 模式提升并发性能
  entities: ['server/entity/**/*.ts'],
  migrations: ['server/migration/sqlite/**/*.ts'],
};

// 也可通过环境变量切换到 PostgreSQL
export const isPgsql = process.env.DB_TYPE === 'postgres';
```

**数据库 Entity 列表**（不含缓存相关表）：
- `User` - 用户信息
- `Media` - 媒体条目状态
- `MediaRequest` - 媒体请求记录
- `Season` / `SeasonRequest` - 剧集季信息
- `Issue` / `IssueComment` - 问题反馈
- `Watchlist` - 关注列表
- `Notification` - 通知设置

> **为什么不把 TMDB 缓存放 SQLite？** 
> 1. node-cache 内存读写比磁盘快 1000x+
> 2. TMDB 数据是临时的、可重建的，不需要持久化
> 3. 进程重启后缓存空冷启动可接受，下一次请求会穿透到 TMDB 并回填

### 8.4 DNS 缓存（额外层）

**文件**: `server/utils/dnsCache.ts:1-26`

除了四层存储外，还有 DNS 解析缓存：

```typescript
export function initializeDnsCache({ forceMinTtl, forceMaxTtl }) {
  dnsCache = new DnsCacheManager({
    logger,
    forceMinTtl: forceMinTtl * 1000,  // 强制最小 TTL
    forceMaxTtl: forceMaxTtl * 1000,  // 强制最大 TTL
  });
  dnsCache.initialize();  // 全局 hook Node.js 的 dns.lookup
}
```

作用：避免每次请求 TMDB 都做 DNS 解析，减少 TCP 握手前的延迟。

### 8.5 各层之间的切换逻辑

**不存在「主动切换」机制**，而是**分层穿透**：

```
请求到达
  │
  ├─ 是图片请求？
  │    ├─ 是 → 先走 Layer 3 文件系统缓存
  │    │         ├─ 命中 → 返回
  │    │         └─ 未命中 → 从远端拉取 → 写入 Layer 3 → 返回
  │    └─ 否 → 走 Layer 2 node-cache
  │              ├─ 命中 → 返回
  │              └─ 未命中 → 请求 TMDB → 写入 Layer 2 → 返回
  │
  ▼
所有响应都可被前端 Layer 1 SWR 再缓存一次
```

进程重启后的冷启动：
- Layer 1 SWR：随浏览器刷新清空
- Layer 2 node-cache：全部丢失，需要重新穿透填充
- Layer 3 文件系统：**保留**（在磁盘上）
- Layer 4 数据库：**保留**

---

## 9. 速率限制与重试的衔接

### 9.1 客户端速率限制：axios-rate-limit

**文件**: `server/api/externalapi.ts:4,45-50`

每个 ExternalAPI 子类都可选配置速率限制，底层使用 `axios-rate-limit` 包：

```typescript
import rateLimit from 'axios-rate-limit';

// ExternalAPI 构造函数中
if (options.rateLimit) {
  this.axios = rateLimit(this.axios, {
    maxRequests: options.rateLimit.maxRequests,  // 窗口内最大请求数
    maxRPS: options.rateLimit.maxRPS,            // 每秒最大请求数
  });
}
```

**TMDB 速率限制配置** (`server/api/themoviedb/index.ts:142-145`)：
```typescript
rateLimit: {
  maxRequests: 20,   // 最多 20 个并发/窗口内请求
  maxRPS: 50,        // 每秒最多 50 个请求
},
```

**TMDB 图片代理速率限制** (`server/routes/imageproxy.ts:12-15`)：
```typescript
_tmdbImageProxy = new ImageProxy('tmdb', 'https://image.tmdb.org', {
  rateLimitOptions: {
    maxRequests: 20,
    maxRPS: 50,
  },
});
```

### 9.2 axios-rate-limit 工作原理

`axios-rate-limit` 通过 axios 请求拦截器实现排队：

```
应用代码 → axios.get('/movie/123')
             │
             ▼
        rateLimit 拦截器
             │
             ├─ 当前已排队请求数 < maxRequests 且 1s 内请求数 < maxRPS
             │    → 立即放行，发送到 TMDB
             │
             └─ 超限
                  → 请求进入内部队列
                  → 等令牌桶有空位时取出发送
                  → Promise 在此期间保持 pending
```

**关键特性**：
- 请求**不会被丢弃**，只是延迟执行
- 队列在内存中，进程重启丢失
- 队列堆积过多时会导致请求延迟非常高

### 9.3 重试机制：无内置重试

**重要发现**：项目**没有实现 axios 自动重试**。

代码中不存在：
- `axios-retry` 包
- 自定义 response interceptor 实现重试
- 429 / 5xx 状态码的退避重试逻辑

**ExternalAPI 构造函数中只有请求拦截器** (`server/api/externalapi.ts:43`)：
```typescript
this.axios.interceptors.request.use(requestInterceptorFunction);
// 没有注册 response interceptor 做重试
```

**后果**：
- TMDB 返回 429 Too Many Requests → 直接抛出异常，不会重试
- 网络超时 → 直接失败，不会重试
- 调用方（如 `MediaRequestSubscriber`）收到异常后也只是记日志，不重试

### 9.4 完整的限流链路图

```
前端请求
  │
  ▼
服务端路由 (Express)
  │
  ├─ 部分路由有 express-rate-limit（如 /api/v1/settings/logs）
  │     → windowMs: 60s, max: 50 次
  │
  ▼
TheMovieDb / ImageProxy 实例
  │
  ├─ cache.get(cacheKey)  → 命中直接返回
  │
  ▼ 缓存未命中
axios-rate-limit 拦截器
  │
  ├─ 当前请求数 < 20 && RPS < 50 → 放行
  └─ 超限 → 排队等待
  │
  ▼
axios 发送 HTTP 请求到 TMDB
  │
  ├─ 200 OK → cache.set() → 返回数据
  ├─ 429 Too Many Requests → 直接抛异常 ❌（不重试）
  └─ 网络错误 / 超时 → 直接抛异常 ❌（不重试）
```

### 9.5 速率限制配置汇总

| 模块 | maxRequests | maxRPS | 代码位置 |
|-----|------------|--------|---------|
| TMDB API | 20 | 50 | `themoviedb/index.ts:142-145` |
| TMDB 图片 | 20 | 50 | `imageproxy.ts:12-15` |
| TVDB API | 可配置 | 可配置 | `tvdb/index.ts:57-60` |
| TVDB 图片 | 20 | 50 | `imageproxy.ts:24-27` |
| Settings Logs API | - | - (1min 内 50 次) | `settings/index.ts:540` |

---

## 10. TMDB API 重试回退机制（Retry/Fallback）

### 10.1 概述：无自动重试，多层 Fallback

Jellyseerr 的 TMDB API 调用**没有内置自动重试**（没有 axios-retry、没有 429 退避），但在业务逻辑层面有多层 **fallback**（降级/回退）策略。

| 回退类型 | 场景 | 回退行为 | 代码位置 |
|---------|------|---------|---------|
| 语言回退 | 请求语言无 overview | 回退到英文版本 | `routes/movie.ts:40-43` |
| Provider 回退 | TVDB 接口失败 | 回退到纯 TMDB 数据 | `api/tvdb/index.ts:188-191` |
| 搜索回退 | 搜索接口抛异常 | 返回空结果集 | `api/themoviedb/index.ts:165-172` |
| ID 回退 | 只有 IMDb/TVDB/AniDB ID | 通过 TMDB 查找对应 tmdbId | `scanners/plex/index.ts:382-532` |
| 空数据回退 | TVDB 季数据为空 | 返回空 season 结构 | `api/tvdb/index.ts:330-333` |

### 10.2 语言回退（Language Fallback）

**文件**: `server/routes/movie.ts:39-43` 与 `server/routes/tv.ts:47-53`

TMDB 存在一个已知 bug：当请求的语言没有对应 overview 时，不会自动回退到英文，而是返回空字符串。

```typescript
// 电影详情路由
const data = mapMovieDetails(tmdbMovie, media, onUserWatchlist);

// TMDB issue where it doesnt fallback to English when no overview is available in requested locale.
if (!data.overview) {
  const tvEnglish = await tmdb.getMovie({ movieId: Number(req.params.id) });
  data.overview = tvEnglish.overview;  // 用英文 overview 填充
}
```

**注意事项**：
- 这是**路由层**的回退，不是 `TheMovieDb` 类内部的回退
- 会触发第二次 API 调用（但第二次会命中缓存，因为默认语言是英文）
- 仅针对 `overview` 字段，其他字段（标题、海报等）仍保留原语言

### 10.3 TVDB → TMDB 数据回退

**文件**: `server/api/tvdb/index.ts:168-196`

Tvdb 类实现 `TvShowProvider` 接口，内部先请求 TMDB，再尝试用 TVDB 数据增强。任何一步失败都回退到上一层。

```typescript
public async getTvShow({ tvId, language }): Promise<TmdbTvDetails> {
  try {
    // 第 1 层：先从 TMDB 获取基础数据
    const tmdbTvShow = await this.tmdb.getTvShow({ tvId, language });

    try {
      // 第 2 层：尝试用 TVDB 增强（季数据更准确）
      await this.refreshToken();
      const tvdbId = this.getTvdbIdFromTmdb(tmdbTvShow);

      if (this.isValidTvdbId(tvdbId)) {
        return await this.enrichTmdbShowWithTvdbData(tmdbTvShow, tvdbId);
      }
      return tmdbTvShow;  // TVDB 不可用，回退到纯 TMDB
    } catch (error) {
      // TVDB 增强失败 → 回退到纯 TMDB 数据
      this.handleError('Failed to fetch TV show details', error);
      return tmdbTvShow;
    }
  } catch (error) {
    // TMDB 本身失败 → 再试一次纯 TMDB（？这里逻辑有点怪）
    this.handleError('Failed to fetch TV show details', error);
    return this.tmdb.getTvShow({ tvId, language });
  }
}
```

**回退层级**：
```
请求 getTvShow()
    │
    ├─ TMDB 请求成功？
    │    ├─ 是 → 尝试 TVDB 增强
    │    │      ├─ TVDB 成功 → 返回增强数据
    │    │      └─ TVDB 失败 → 返回纯 TMDB 数据 ←────┐
    │    │                                                │
    │    └─ 否 → 抛出异常 → catch 中再试一次 TMDB ──────┘
    │                （实际上是重复请求，可能也会失败）
    │
    ▼
最终结果
```

> **设计疑问**：最外层 catch 中再次调用 `this.tmdb.getTvShow()` 实际上是重试一次 TMDB，但如果第一次失败了，第二次大概率也会失败。这可能是为了给缓存一个机会，或者是代码遗留问题。

### 10.4 搜索接口的静默回退

**文件**: `server/api/themoviedb/index.ts:153-173`

搜索接口失败时不抛异常，而是返回空结果集：

```typescript
public searchMulti = async ({ query, page = 1, includeAdult = false, language }): Promise<TmdbSearchMultiResponse> => {
  try {
    const data = await this.get<TmdbSearchMultiResponse>('/search/multi', {
      params: { query, page, include_adult: includeAdult, language },
    });
    return data;
  } catch {
    // 静默回退：返回空结果，不报错
    return {
      page: 1,
      results: [],
      total_pages: 1,
      total_results: 0,
    };
  }
};
```

**设计考量**：搜索是体验型功能，失败了不应让整个页面崩溃。空结果比错误页面对用户更友好。

### 10.5 ID 解析回退链（Scanner 层）

**文件**: `server/lib/scanners/plex/index.ts:382-532`

Plex 扫描器从各种 guid 格式中提取 ID，形成长长的回退链：

```
Plex Item GUID
    │
    ├─ plex:// 格式（新版 Plex Agent）
    │    ├─ Guid 数组中有 tmdb:// → 直接提取 tmdbId ✓
    │    ├─ Guid 数组中有 imdb:// → 通过 IMDb 查 TMDB → tmdbId ✓
    │    └─ Guid 数组中有 tvdb:// → 通过 TVDB 查 TMDB → tmdbId ✓
    │
    ├─ imdb://tt1234567 → 通过 IMDb 查 TMDB → tmdbId ✓
    │
    ├─ tmdb://12345 → 直接提取 tmdbId ✓
    │
    ├─ tvdb://12345 → 通过 TVDB 查 TMDB → tmdbId ✓
    │
    ├─ themoviedb://12345 → 直接提取 tmdbId ✓
    │
    ├─ hama://tvdb-12345 → TVDB → 查 TMDB → tmdbId ✓
    │
    └─ hama://anidb-12345 → AniDB 映射表
             ├─ 有 tvdbId → TVDB → 查 TMDB → tmdbId ✓
             ├─ 有 tmdbId → 直接用 tmdbId ✓
             └─ 有 imdbId → IMDb → 查 TMDB → tmdbId ✓
```

**缓存优化** (`plex/index.ts:386-396`)：
```typescript
const guidCache = cacheManager.getCache('plexguid');
const cachedGuids = guidCache.data.get<MediaIds>(plexitem.ratingKey);

if (cachedGuids) {
  mediaIds = cachedGuids; // 命中缓存，跳过整个 ID 解析链
}
```

`plexguid` 缓存 TTL 为 **7 天**（`server/lib/cache.ts:65-68`），因为 Plex 媒体的 ID 几乎不会变。

---

## 11. 多数据源元数据 Normalize

### 11.1 三层 Normalize 架构

Jellyseerr 有三层数据归一化，确保来自不同 source 的数据最终以统一格式呈现：

```
┌───────────────────────────────────────────────────┐
│  Layer 1: Provider 接口层                         │
│  TheMovieDb / Tvdb → TmdbTvDetails 统一接口       │
├───────────────────────────────────────────────────┤
│  Layer 2: Scanner ID 归一化                       │
│  Plex / Jellyfin → MediaIds { tmdbId, imdbId... } │
├───────────────────────────────────────────────────┤
│  Layer 3: Model 映射层                            │
│  mapMovieDetails / mapTvDetails → MovieDetails    │
└───────────────────────────────────────────────────┘
```

### 11.2 Layer 1：Provider 接口归一化

**文件**: `server/api/provider.ts` (接口定义) + `server/api/themoviedb/index.ts` + `server/api/tvdb/index.ts`

`TvShowProvider` 接口定义了统一的方法签名：

```typescript
interface TvShowProvider {
  getTvShow: (options: { tvId: number; language?: string }) => Promise<TmdbTvDetails>;
  getTvSeason: (options: { tvId: number; seasonNumber: number; language?: string }) => Promise<TmdbSeasonWithEpisodes>;
  // ...
}
```

**关键设计**：两个 Provider 都返回 `TmdbTvDetails` 类型，而不是各自的原生类型。
- `TheMovieDb`：直接返回 TMDB 原生数据
- `Tvdb`：内部调用 TMDB 获取基础数据，再用 TVDB 的季/集数据覆盖增强，最终返回同类型

**元数据提供者选择** (`server/api/metadata.ts:7-39`)：
```typescript
export const getMetadataProvider = async (mediaType) => {
  const settings = await getSettings();
  
  // 电影永远用 TMDB
  if (mediaType == 'movie') return new TheMovieDb();
  
  // TV/Anime 可配置为 TVDB 或 TMDB
  if (mediaType == 'tv' && settings.metadataSettings.tv == MetadataProviderType.TVDB) {
    return await Tvdb.getInstance();
  }
  if (mediaType == 'anime' && settings.metadataSettings.anime == MetadataProviderType.TVDB) {
    return await Tvdb.getInstance();
  }
  
  return new TheMovieDb(); // 默认 TMDB
};
```

**调用方无感切换** (`server/routes/tv.ts:24-32`)：
```typescript
const metadataProvider = tmdbTv.keywords.results.some(k => k.id === ANIME_KEYWORD_ID)
  ? await getMetadataProvider('anime')
  : await getMetadataProvider('tv');

const tv = await metadataProvider.getTvShow({ tvId, language });
// 不管底层是 TMDB 还是 TVDB，返回类型都是 TmdbTvDetails
```

### 11.3 Layer 2：Scanner ID 归一化

**Plex 扫描器** (`server/lib/scanners/plex/index.ts:382-532`)：
- 支持 7+ 种 GUID 格式（plex://, imdb://, tmdb://, tvdb://, themoviedb://, hama://tvdb, hama://anidb）
- 最终输出统一的 `MediaIds` 结构：`{ tmdbId, imdbId?, tvdbId?, isHama? }`
- **tmdbId 是唯一主键**，所有其他 ID 都用来反查 tmdbId

**Jellyfin 扫描器** (`server/lib/scanners/jellyfin/index.ts:48-121`)：
- 从 `ProviderIds` 对象中提取（Tmdb, TheMovieDb, Imdb, AniDB）
- 同样以 tmdbId 为最终主键
- 回退链：AniDB → TMDB/IMDb → TMDB

**统一处理入口** (`server/lib/scanners/baseScanner.ts:95-150`)：
```typescript
protected async processMovie(tmdbId: number, options?: ProcessOptions) {
  // 所有 scanner 都走这个统一入口
  // 只需要 tmdbId，底层数据源透明
  const mediaRepository = getRepository(Media);
  const existing = await this.getExisting(tmdbId, MediaType.MOVIE);
  // ...
}
```

### 11.4 Layer 3：Model 映射层

**文件**: `server/models/Movie.ts`、`server/models/Tv.ts`、`server/models/Search.ts`、`server/models/common.ts`

Model 层将 TMDB/TVDB 的原始 API 响应映射为前端使用的标准化格式。

**命名转换**：snake_case → camelCase
```typescript
// server/models/Movie.ts:103-154
export const mapMovieDetails = (movie, media?, userWatchlist?): MovieDetails => ({
  id: movie.id,
  title: movie.title,
  backdropPath: movie.backdrop_path,      // snake → camel
  posterPath: movie.poster_path,
  originalTitle: movie.original_title,
  releaseDate: movie.release_date,
  voteAverage: movie.vote_average,
  voteCount: movie.vote_count,
  // ...
  credits: {
    cast: movie.credits.cast.map(mapCast),  // 递归映射
    crew: movie.credits.crew.map(mapCrew),
  },
  externalIds: mapExternalIds(movie.external_ids),
});
```

**列表 vs 详情的不同映射**：
- `mapMovieResult`：精简字段，用于列表/搜索结果（10 个字段左右）
- `mapMovieDetails`：完整字段，用于详情页（40+ 字段，含 credits、keywords、watchProviders 等）

### 11.5 Plex vs Jellyfin 扫描器对比

| 维度 | Plex Scanner | Jellyfin Scanner |
|-----|-------------|------------------|
| 数据源 | Plex API (XML → JSON) | Jellyfin API (JSON) |
| ID 来源 | GUID 字符串正则匹配 | ProviderIds 对象直接读取 |
| Agent 类型 | 7+ 种（plex/imdb/tmdb/tvdb/hama） | 4 种（Tmdb/TheMovieDb/Imdb/AniDB） |
| 媒体信息 | Media 数组（多版本） | MediaSources 数组 |
| 4K 检测 | `videoResolution === '4k'` | 视频宽度 > 2000px |
| GUID 缓存 | 有（plexguid，7 天） | 无 |
| Hama/Anime 支持 | 完整支持 | 部分支持（AniDB ID） |
| 基础类 | BaseScanner | BaseScanner |

### 11.6 Normalize 与缓存的关系

**两层缓存各自独立**：
1. **API 层缓存**：缓存原始 TMDB/TVDB 响应，由 `ExternalAPI.get()` 管理
2. **Scanner 层缓存**：缓存 ID 解析结果（plexguid），由扫描器直接调用 cacheManager

Model 层的 `map*` 函数**不涉及缓存**，每次调用都重新计算。

> **优化空间**：map 函数是纯函数，如果对同一数据反复调用可以加 memoization。但实际中详情页只调用一次，列表页每条结果也只映射一次，收益不大。

---

## 12. 缓存 Key Collision 处理

### 12.1 核心结论：无显式 Collision 处理

Jellyseerr **没有任何显式的缓存 Key 碰撞检测或处理机制**。依赖于缓存 Key 的设计来避免碰撞。

### 12.2 命名空间隔离

**第一层隔离：按 API 分缓存实例**

`server/lib/cache.ts:45-78` 中为每个外部 API 创建独立的 `Cache` 实例：

| 缓存 ID | 用途 | TTL |
|--------|------|-----|
| `tmdb` | TMDB API 数据 | 21600s (6h) |
| `tvdb` | TVDB API 数据 | 21600s (6h) |
| `radarr` | Radarr API 数据 | 300s (5min) |
| `sonarr` | Sonarr API 数据 | 300s (5min) |
| `rt` | Rotten Tomatoes | 43200s (12h) |
| `imdb` | IMDB (via Radarr) | 43200s (12h) |
| `github` | GitHub API | 21600s (6h) |
| `plexguid` | Plex GUID → MediaIds | 604800s (7d) |
| `plextv` | Plex TV API | 604800s (7d) |
| `plexwatchlist` | Plex Watchlist | 300s (5min) |

每个实例是独立的 `NodeCache` 对象，拥有各自的内存存储空间。**跨 API 的 Key 碰撞是不可能的**，因为根本不在同一个 Map 里。

### 12.3 同 API 内的 Key 生成

**文件**: `server/api/externalapi.ts:146-155`

同一个缓存实例内，Key 生成公式为：

```
cacheKey = baseUrl + endpoint + JSON.stringify(options)
```

其中 GET 请求的 `options` 包含：
- `params`：所有查询参数（page, language, sort_by 等）
- `headers`：所有请求头

**示例 Key**：
```
https://api.themoviedb.org/3/movie/123{"language":"zh-CN","headers":{}}
```

**设计特点**：
- ✅ `baseUrl` 前缀：即使不同 API 用同一个缓存实例（实际上不会），也不会撞 Key
- ✅ 所有 params 参与：不同语言、分页、排序都是独立 Key
- ✅ headers 参与：不同认证 token（如 TVDB 的 Bearer token）也是独立 Key
- ⚠️ `JSON.stringify` 依赖对象 key 顺序：如果 params 以不同顺序传入，会生成不同 Key

### 12.4 潜在碰撞风险点

#### 风险 1：params key 顺序不一致

`JSON.stringify({a:1,b:2})` 和 `JSON.stringify({b:2,a:1})` 产生不同字符串。

**是否实际会发生？** 不会。因为每个 API 方法调用 `this.get()` 时，`params` 对象的 key 顺序在代码中是固定的。例如：

```typescript
// server/api/themoviedb/index.ts
this.get('/discover/movie', {
  params: {
    page,           // 永远第一个
    sort_by: sortBy, // 永远第二个
    language,       // 永远第三个
    // ...
  },
});
```

只要代码不重构改变 key 顺序，就不会有碰撞问题。

#### 风险 2：TheMovieDb 多实例共享同一缓存

**文件**: `server/api/themoviedb/index.ts:141`

所有 `TheMovieDb` 实例都共享同一个 `tmdb` 缓存实例：

```typescript
super(
  'https://api.themoviedb.org/3',
  { api_key: '431a8708161bcd1f1fbe7536137e61ed' },
  {
    nodeCache: cacheManager.getCache('tmdb').data,  // 所有实例共用
    rateLimit: { ... },
  }
);
```

**不同 locale 的实例**：
- 实例 A：`locale = 'zh-CN'`
- 实例 B：`locale = 'en-US'`

它们生成的缓存 Key 会不同吗？

**答案**：会不同。因为 `language` 参数会出现在 params 中：
```typescript
// getMovie 方法
const data = await this.get<TmdbMovieDetails>(
  `/movie/${movieId}`,
  { params: { language, ... } },
  43200
);
```
每个调用都显式传入 `language`，因此 Key 中包含语言信息，不同 locale 的实例不会发生 Key 碰撞。

#### 风险 3：POST 请求的 Key

POST 请求的 Key 还包含 `data`：

```typescript
// server/api/externalapi.ts:85-88
const cacheKey = this.serializeCacheKey(endpoint, {
  config: config?.params,
  ...(data ? { data } : {}),  // POST body 也参与 Key 生成
});
```

只要 body 内容不同，Key 就不同。

### 12.5 TVDB 与 TMDB 的 Key 关系

TVDB 的 `getTvShow()` 内部会调用 `this.tmdb.getTvShow()`，即**一次请求写入两个缓存**：

```
Tvdb.getTvShow(tvId=123, language='zh')
    │
    ├─ 查 tvdb 缓存（key 含 tvdb baseUrl）
    │    └─ 未命中 → 继续
    │
    ├─ 调用 this.tmdb.getTvShow(tvId=123, language='zh')
    │    ├─ 查 tmdb 缓存 → 命中/未命中
    │    └─ 写入 tmdb 缓存
    │
    ├─ 调用 TVDB API 增强数据
    │    └─ 写入 tvdb 缓存
    │
    └─ 返回合并结果
```

**两个缓存互不干扰**，因为：
- 使用不同的 `Cache` 实例（`tvdb` vs `tmdb`）
- `baseUrl` 不同（`https://api4.thetvdb.com/v4` vs `https://api.themoviedb.org/3`）

### 12.6 图片缓存的 Key

**文件**: `server/lib/imageproxy.ts:148-156`

图片缓存使用 MD5 哈希作为目录名，避免路径特殊字符问题：

```typescript
private getCacheKey(path: string) {
  const hash = createHash('md5');
  hash.update(path);
  const digest = hash.digest('hex');
  return `${this.cacheKey}-${digest}`;
}
```

**碰撞概率**：MD5 是 128 位哈希，对于图片 URL 这种规模的数据，碰撞概率可以忽略不计。

### 12.7 Collision 风险评估

| 场景 | 碰撞概率 | 严重性 | 备注 |
|-----|---------|--------|------|
| 跨 API 碰撞 | 0 | 高 | 独立缓存实例，完全隔离 |
| 同 API 不同参数 | ≈0 | 中 | JSON.stringify key 顺序固定 |
| 同 API 同参数不同 locale | ≈0 | 中 | language 参与 Key |
| 图片 URL 碰撞 | ≈0 | 低 | MD5 哈希，概率可忽略 |
| 多实例写入同一 Key | 无影响 | - | 最后写入者胜出，内容相同 |

**结论**：当前架构下缓存 Key 碰撞的风险极低，不需要额外的碰撞检测机制。

---

## 13. 关键设计特点总结

1. **四层缓存**：前端 SWR + 服务端 node-cache + 文件系统图片缓存 + DNS 缓存
2. **分级 TTL**：不同类型数据使用不同过期时间，平衡新鲜度和性能
3. **滚动刷新**：对配置类数据使用后台刷新，保证用户永远快速响应
4. **精细 Key**：包含所有参数（语言、地区、分页等），确保缓存正确性
5. **无请求合并**：服务端无并发去重，高并发场景下可能出现缓存击穿
6. **列表详情分离**：列表和详情使用不同 API 端点，缓存不共享
7. **无精确失效**：TMDB 元数据只支持 TTL 过期和全量 Flush，不支持按 tmdbId 单条失效
8. **无 Redis**：依赖 node-cache 内存缓存 + SQLite/PG 持久化 + 文件系统图片缓存
9. **无自动重试**：axios 没有配置重试，429/网络错误直接失败
10. **客户端限流**：axios-rate-limit 排队机制，保护 TMDB 不被打爆，但可能造成请求延迟
11. **多层 Fallback**：语言回退、Provider 回退、ID 解析回退链，确保可用性
12. **三层 Normalize**：Provider 接口层 + Scanner ID 层 + Model 映射层，多数据源统一输出
13. **命名空间隔离**：10 个独立缓存实例 + baseUrl 前缀，从架构上避免 Key 碰撞
14. **TMDB 为中心**：所有扫描器都以 tmdbId 为主键，其他 ID 反查 TMDB

---

## 14. 代码位置速查表（完整版）

| 功能 | 文件路径 | 关键行 |
|-----|---------|-------|
| **缓存核心** | | |
| 缓存管理器 | `server/lib/cache.ts` | 45-87 |
| cache.flush() 全量清空 | `server/lib/cache.ts` | 40-42 |
| ExternalAPI 基类 | `server/api/externalapi.ts` | 全部 |
| 标准 GET 缓存 | `server/api/externalapi.ts` | 56-77 |
| 滚动刷新 GET | `server/api/externalapi.ts` | 104-137 |
| axios-rate-limit 接入 | `server/api/externalapi.ts` | 45-50 |
| 缓存 Key 生成 | `server/api/externalapi.ts` | 146-155 |
| TMDB API 封装 + 限流配置 | `server/api/themoviedb/index.ts` | 127-147 |
| TVDB API 封装 | `server/api/tvdb/index.ts` | 42-66 |
| 元数据 Provider 选择器 | `server/api/metadata.ts` | 7-39 |
| **图片缓存** | | |
| 图片缓存代理 | `server/lib/imageproxy.ts` | 全部 |
| 图片缓存精确删除 | `server/lib/imageproxy.ts` | 187-226 |
| 图片缓存 Key (MD5) | `server/lib/imageproxy.ts` | 148-156 |
| 过期图片定时清理 | `server/job/schedule.ts` | 227-244 |
| 头像版本检查与失效 | `server/routes/avatarproxy.ts` | 52-116 |
| **缓存失效** | | |
| 管理员 Flush 缓存路由 | `server/routes/settings/index.ts` | 781-807 |
| 头像缓存精确失效 | `server/routes/avatarproxy.ts` | 52-116 |
| **Retry/Fallback** | | |
| 语言回退（overview 空） | `server/routes/movie.ts` | 39-43 |
| TVDB → TMDB Provider 回退 | `server/api/tvdb/index.ts` | 168-196 |
| 搜索接口静默回退 | `server/api/themoviedb/index.ts` | 153-173 |
| Plex Scanner ID 解析回退链 | `server/lib/scanners/plex/index.ts` | 382-532 |
| Jellyfin Scanner ID 提取 | `server/lib/scanners/jellyfin/index.ts` | 48-121 |
| Plex GUID 缓存 | `server/lib/scanners/plex/index.ts` | 386-396 |
| **数据 Normalize** | | |
| Provider 接口定义 | `server/api/provider.ts` | - |
| Plex Scanner getMediaIds | `server/lib/scanners/plex/index.ts` | 382-532 |
| Jellyfin GUID 归一化 | `server/utils/jellyfin.ts` | 1-15 |
| BaseScanner 统一处理入口 | `server/lib/scanners/baseScanner.ts` | 95-150 |
| mapMovieDetails（详情映射） | `server/models/Movie.ts` | 103-154 |
| mapTvDetails（剧集映射） | `server/models/Tv.ts` | 163-229 |
| mapMovieResult（列表映射） | `server/models/Search.ts` | 71-91 |
| mapTvResult（剧集列表映射） | `server/models/Search.ts` | 93-113 |
| 公共映射（cast/crew/externalIds） | `server/models/common.ts` | 全部 |
| **存储层** | | |
| 数据源配置 (SQLite/PG) | `server/datasource.ts` | 50-147 |
| DNS 缓存初始化 | `server/utils/dnsCache.ts` | 1-26 |
| **前端** | | |
| 前端 useDiscover | `src/hooks/useDiscover.ts` | 54-173 |
| SWR 全局配置 | `src/pages/_app.tsx` | 191-198 |
| CachedImage 前端组件 | `src/components/Common/CachedImage/index.tsx` | 全部 |
| 媒体请求 Subscriber | `server/subscriber/MediaRequestSubscriber.ts` | 全部 |
| 发现路由 | `server/routes/discover.ts` | 97-182 |
| 电影详情路由 | `server/routes/movie.ts` | 16-57 |
| 搜索路由 | `server/routes/search.ts` | 11-61 |
| 电影详情页 (SSR) | `src/pages/movie/[movieId]/index.tsx` | 14-33 |
