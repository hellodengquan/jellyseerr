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

## 13. Watchlist 同步与多用户共享

### 13.1 整体架构

Watchlist 同步是 **Plex 专属功能**，仅当媒体服务器类型为 Plex 时才启动定时任务。Jellyfin/Emby 模式下不存在 watchlist sync。

```
定时任务 (plex-watchlist-sync)
    │
    ▼
WatchlistSync.syncWatchlist()
    │
    ├─ 查询所有有 plexToken 的用户
    │
    ▼ 逐个用户循环
syncUserWatchlist(user)
    │
    ├─ 权限检查 (AUTO_REQUEST)
    ├─ 用户设置检查 (watchlistSyncMovies/watchlistSyncTv)
    │
    ├─ PlexTvAPI.getWatchlist() → 获取远端 Watchlist
    ├─ Media.getRelatedMedia() → 查本地已有媒体
    ├─ 查已有 auto-request → 过滤已请求的
    │
    └─ 对「不可用」项发起新请求
```

### 13.2 多用户共享模型

**文件**: `server/lib/watchlistsync.ts:18-32`

```typescript
public async syncWatchlist() {
  const users = await userRepository
    .createQueryBuilder('user')
    .addSelect('user.plexToken')
    .leftJoinAndSelect('user.settings', 'settings')
    .where("user.plexToken != ''")
    .getMany();

  for (const user of users) {
    await this.syncUserWatchlist(user);  // 串行处理每个用户
  }
}
```

**共享行为**：

| 场景 | 行为 |
|------|------|
| 用户 A 和 B 都 watchlist 了同一电影 | 各自独立创建 auto-request，共享同一个 Media 记录 |
| 用户 A 的 watchlist 中有已可用的媒体 | 跳过，不发请求 |
| 用户 A 的 watchlist 中有已 blocklisted 的媒体 | 跳过，不发请求 |
| 用户 A 的 auto-request 已完成但 Media 被删除 | 重新发起 auto-request |

**数据库模型** (`server/entity/Watchlist.ts:28-68`)：
```typescript
@Entity()
@Unique('UNIQUE_USER_DB', ['tmdbId', 'mediaType', 'requestedBy'])
export class Watchlist {
  @ManyToOne(() => User, { eager: true, onDelete: 'CASCADE' })
  public requestedBy: User;     // 多对一：多个 watchlist 属于同一个用户

  @ManyToOne(() => Media, { eager: true, onDelete: 'CASCADE' })
  public media: Media;          // 多对一：多个 watchlist 关联同一个媒体

  // 联合唯一约束：同一用户对同一媒体只能有一条 watchlist
}
```

**多用户共享的关键**：
- `Media` 表是**全局共享**的：不管哪个用户请求，同一 tmdbId 只有一条 Media 记录
- `Watchlist` 表是**用户维度**的：同一电影可以被多个用户加入 watchlist
- `MediaRequest` 表也是**用户维度**的：同一媒体可以有多个 auto-request（不同用户）

### 13.3 Watchlist 缓存：ETag 机制

**文件**: `server/api/plextv.ts:271-312`

Plex Watchlist 使用 HTTP ETag 实现条件请求：

```typescript
public async getWatchlist({ offset, size }) {
  const watchlistCache = cacheManager.getCache('plexwatchlist');
  let cachedWatchlist = watchlistCache.data.get<PlexWatchlistCache>(this.authToken);

  const response = await this.axios.get<WatchlistResponse>(
    '/library/sections/watchlist/all',
    {
      params: { 'X-Plex-Container-Start': offset, 'X-Plex-Container-Size': size },
      headers: {
        'If-None-Match': cachedWatchlist?.etag,  // 发送上次缓存的 ETag
      },
      baseURL: 'https://discover.provider.plex.tv',
      validateStatus: (status) => status < 400,  // 304 不算错误
    }
  );

  // 200-299：数据有更新
  if (response.status >= 200 && response.status <= 299) {
    cachedWatchlist = {
      etag: response.headers.etag,
      response: response.data,
    };
    watchlistCache.data.set<PlexWatchlistCache>(this.authToken, cachedWatchlist);
  }
  // 304：数据未变化，使用缓存
}
```

**缓存 Key**：`this.authToken`（按用户 Plex Token 区分，天然隔离多用户）

**ETag 流程**：
```
首次请求 → 无 If-None-Match → 200 + ETag → 缓存 { etag, response }
后续请求 → If-None-Match: <etag> →
  ├─ 304 Not Modified → 使用缓存，省带宽
  └─ 200 OK → 数据有变 → 更新缓存
```

### 13.4 Watchlist Item 的 TMDB ID 提取

获取 watchlist 列表后，每个 item 还需要请求详情来获取 tmdbId：

```typescript
const watchlistDetails = await Promise.all(
  cachedWatchlist.response.MediaContainer.Metadata.map(async (watchlistItem) => {
    // 滚动缓存每个 item 的 metadata 详情
    const detailedResponse = await this.getRolling<MetadataResponse>(
      `/library/metadata/${watchlistItem.ratingKey}`,
      { baseURL: 'https://discover.provider.plex.tv' }
    );

    // 从 Guid 数组中提取 tmdb/tvdb ID
    const tmdbString = metadata.Guid?.find(guid => guid.id.startsWith('tmdb'));
    const tvdbString = metadata.Guid?.find(guid => guid.id.startsWith('tvdb'));

    return {
      tmdbId: tmdbString ? Number(tmdbString.id.split('//')[1]) : 0,
      tvdbId: tvdbString ? Number(tvdbString.id.split('//')[1]) : undefined,
      title: metadata.title,
      type: metadata.type,
    };
  })
);
```

**缓存影响**：
- `getRolling` 使用 `plextv` 缓存实例（TTL 7 天），watchlist item 的 metadata 几乎不变
- 20 个 watchlist item 最多触发 20 个 metadata 请求（大多命中滚动缓存）

### 13.5 Watchlist 与 Media 请求的状态过滤

**文件**: `server/lib/watchlistsync.ts:100-117`

不是所有 watchlist item 都会触发请求，有严格的状态过滤：

```typescript
const unavailableItems = response.items.filter((i) => {
  const itemMediaType = i.type === 'show' ? MediaType.TV : MediaType.MOVIE;

  return (
    // 排除已有 auto-request 的（除非 Media 已删除）
    !autoRequestedTmdbIds.has(`${itemMediaType}:${i.tmdbId}`) &&
    // 排除已 blocklisted / 已可用的
    !mediaItems.find(m =>
      m.tmdbId === i.tmdbId &&
      m.mediaType === itemMediaType &&
      (m.status === MediaStatus.BLOCKLISTED ||
        (itemMediaType === MediaType.MOVIE && m.status !== MediaStatus.UNKNOWN && m.status !== MediaStatus.DELETED) ||
        (itemMediaType === MediaType.TV && m.status === MediaStatus.AVAILABLE))
    )
  );
});
```

**过滤规则**：

| 媒体状态 | 电影 | 剧集 | 是否发起请求 |
|---------|------|------|------------|
| UNKNOWN | ✓ | ✓ | ✅ 是 |
| DELETED | ✓ | - | ✅ 是（重新请求） |
| PENDING / PROCESSING | ✓ | - | ❌ 否（已在处理） |
| AVAILABLE | - | ✓ | ❌ 否（已可看） |
| PARTIALLY_AVAILABLE | - | ✓ | ✅ 是（部分可用，仍需请求） |
| BLOCKLISTED | ✓ | ✓ | ❌ 否（黑名单） |

---

## 14. TMDB Rate Limit Hit 后的降级策略

### 14.1 核心结论：无主动降级，依赖缓存屏障 + 静默失败

Jellyseerr **没有实现**：
- 429 状态码的自动退避重试
- 指数退避（exponential backoff）
- 熔断器（circuit breaker）
- 服务降级（graceful degradation to stale data）

但通过**缓存屏障**和**分层错误处理**实现了事实上的降级。

### 14.2 三道防线

```
请求到达
    │
    ▼ 第 1 道：缓存屏障
node-cache 命中？ → 直接返回，根本不触碰 TMDB API
    │
    ▼ 第 2 道：客户端限流
axios-rate-limit (maxRequests: 20, maxRPS: 50)
    │  排队延迟，但不丢弃
    │
    ▼ 第 3 道：业务层错误处理
各 API 方法的 try/catch → 返回空数据/回退数据
```

### 14.3 缓存作为第一道防线

缓存是抵御 rate limit 的**最主要手段**。在正常运行中，绝大多数请求都会命中缓存：

| 缓存类型 | TTL | 命中场景 |
|---------|-----|---------|
| TMDB 元数据 | 6-12h | 详情页、列表页反复访问 |
| TMDB 图片 | 按过期时间 | 前端 CachedImage |
| Plex GUID | 7d | 扫描器 ID 解析 |
| Plex TV Watchlist | 5min | 定时同步 |
| Plex TV Metadata | 7d (滚动) | Watchlist 详情 |

**冷启动场景**：进程重启后所有内存缓存丢失，大量请求同时穿透到 TMDB，此时 rate limit 最容易被打满。

### 14.4 Rate Limit 被打满时的行为

```
TMDB 返回 429
    │
    ▼
axios 抛出 AxiosError (status: 429)
    │
    ▼ ExternalAPI.get() 没有捕获 → 向上传播
    │
    ▼ 调用方的 try/catch 处理：
    │
    ├─ searchMulti() → 返回空结果 { results: [], total_results: 0 }
    ├─ searchMovies() → 返回空结果
    ├─ searchTv() → 返回空结果
    ├─ getMovie() (路由层) → 返回 500 "Unable to retrieve movie."
    ├─ getTvShow() (路由层) → 返回 500 "Unable to retrieve series."
    └─ Scanner 中 → 跳过该条目，继续下一条
```

**用户感知**：

| 场景 | 用户看到 |
|------|---------|
| 搜索页面 | 空结果（无报错） |
| 电影详情页 | 500 错误页 |
| 发现列表 | 部分列表项缺失 |
| Watchlist 同步 | 跳过失败项，下一轮重试 |
| Scanner 同步 | 跳过失败媒体 |

### 14.5 Watchlist Sync 的重试机制

**文件**: `server/lib/watchlistsync.ts:119-194`

Watchlist 同步虽然不是真正的"重试"，但通过**定时轮询**实现了最终一致性：

```typescript
for (const mediaItem of unavailableItems) {
  try {
    await MediaRequest.request({ ... }, user, { isAutoRequest: true });
  } catch (e) {
    // 不同错误不同处理：
    switch (e.constructor) {
      case RequestPermissionError:    // 权限不足 → debug 日志
      case DuplicateMediaRequestError: // 重复请求 → debug 日志
      case QuotaRestrictedError:      // 配额限制 → debug 日志
      case NoSeasonsAvailableError:   // 无可用季 → debug 日志
      case BlocklistedMediaError:     // 黑名单 → 静默忽略（不记日志）
        break;
      default:
        logger.error('Failed to create media request from watchlist', { ... });
    }
  }
}
```

**定时任务调度** (`server/job/schedule.ts:89-107`)：
- Plex 模式下注册 `plex-watchlist-sync` 定时任务
- 调度频率由用户配置决定（默认每隔一段时间运行）
- 每次运行都从头扫描所有用户的 watchlist
- 上一轮失败的请求在下一轮会被重新尝试（因为 Media 状态未改变）

### 14.6 定时扫描器（Scanner）的容错

扫描器同样依赖定时重试而非即时重试：

```
Scanner 同步失败
    │
    ├─ 单条媒体失败 → logger.error + continue（跳过，继续下一条）
    │
    └─ 整个扫描失败 → 进程退出，等待下一轮定时任务
```

**没有 backoff 机制**：即使 TMDB 持续返回 429，定时任务仍按原定频率运行，可能导致连续失败。

---

## 15. Metadata 多语言 i18n 加载衔接

### 15.1 两条独立的 i18n 路径

Jellyseerr 的国际化有两条完全独立的路径：

| 路径 | 作用域 | 实现方式 | 数据源 |
|------|--------|---------|--------|
| 前端 UI i18n | 界面文本（按钮、标签、提示） | react-intl | `src/i18n/locale/*.json` |
| 后端 Metadata i18n | TMDB 元数据（标题、简介） | TMDB API language 参数 | TMDB 服务端 |

```
┌──────────────────────────────────────┐
│  前端 UI i18n                        │
│  IntlProvider + react-intl           │
│  影响范围：按钮、标签、错误消息等      │
├──────────────────────────────────────┤
│  后端 Metadata i18n                  │
│  TMDB API ?language=zh-CN            │
│  影响范围：电影标题、简介、海报等      │
└──────────────────────────────────────┘
两条路径共享同一个 locale 配置（用户设置中的 locale 字段）
```

### 15.2 Locale 的确定链路

**文件**: `server/middleware/auth.ts:36-38`

每个 API 请求的 locale 由中间件确定：

```typescript
req.locale = user?.settings?.locale
  ? user.settings.locale        // 优先用户设置
  : settings.main.locale;       // 回退到全局设置
```

**优先级**：`用户 settings.locale` > `全局 main.locale` > `默认 en`

### 15.3 前端 UI i18n 加载

**文件**: `src/pages/_app.tsx`

#### SSR 阶段（首次加载）

```typescript
CoreApp.getInitialProps = async (initialProps) => {
  const locale = user?.settings?.locale
    ? user.settings.locale
    : currentSettings.locale;

  const messages = await loadLocaleData(locale as AvailableLocale);
  // loadLocaleData 是动态 import：
  //   case 'zh-CN': return import('../i18n/locale/zh_Hans.json');
  //   case 'en':    return import('../i18n/locale/en.json');
  //   default:      return import('../i18n/locale/en.json');

  return { ...appInitialProps, user, messages, locale, currentSettings };
};
```

#### CSR 阶段（切换语言）

```typescript
const [loadedMessages, setMessages] = useState<MessagesType>(messages);
const [currentLocale, setLocale] = useState<AvailableLocale>(locale);

useEffect(() => {
  loadLocaleData(currentLocale).then(setMessages);
}, [currentLocale]);
```

**LanguageContext** 提供全局语言切换：
```tsx
<LanguageContext.Provider value={{ locale: currentLocale, setLocale }}>
  <IntlProvider locale={currentLocale} defaultLocale="en" messages={loadedMessages}>
    {/* 所有子组件通过 useIntl() 获取翻译 */}
  </IntlProvider>
</LanguageContext.Provider>
```

**支持 38 种语言** (`server/types/languages.ts:1-39`)：ar, bg, ca, cs, da, de, en, el, es, es-MX, et, fi, fr, hr, he, hi, hu, it, ja, ko, lb, lt, nb-NO, nl, pl, pt-BR, pt-PT, ro, ru, sq, sr, sv, tr, uk, zh-CN, zh-TW, vi

### 15.4 后端 Metadata i18n 加载

后端元数据的国际化是通过 TMDB API 的 `language` 参数实现的：

**路由层** (`server/routes/movie.ts:20-23`)：
```typescript
const tmdbMovie = await tmdb.getMovie({
  movieId: Number(req.params.id),
  language: (req.query.language as string) ?? req.locale,
  // 优先 URL query → 回退 req.locale
});
```

**API 层** (`server/api/themoviedb/index.ts`)：
```typescript
public getMovie = async ({ movieId, language = this.locale }) => {
  const data = await this.get<TmdbMovieDetails>(
    `/movie/${movieId}`,
    { params: { language, append_to_response: '...' } },
    43200
  );
  return data;
};
```

**language 参数如何影响缓存 Key**：

```
cacheKey = baseUrl + endpoint + JSON.stringify({ language, ... })

/movie/123{"language":"zh-CN","headers":{}}
/movie/123{"language":"en","headers":{}}
```

> ⚠️ **重要**：不同语言的请求会产生不同的缓存 Key。如果用户 A 用中文访问，用户 B 用英文访问同一部电影，会产生两条缓存条目。这意味着 38 种语言最多 38 倍缓存空间。

### 15.5 发现页面的语言定制

**文件**: `server/routes/discover.ts:29-49`

发现页面除了 UI 语言外，还有两个额外的区域/语言维度：

```typescript
export const createTmdbWithRegionLanguage = (user?: User): TheMovieDb => {
  const discoverRegion =
    user?.settings?.streamingRegion === 'all'
      ? ''
      : user?.settings?.streamingRegion ?? settings.main.discoverRegion;

  const originalLanguage =
    user?.settings?.originalLanguage === 'all'
      ? ''
      : user?.settings?.originalLanguage ?? settings.main.originalLanguage;

  return new TheMovieDb({ discoverRegion, originalLanguage });
};
```

**三维度定制**：

| 维度 | 影响范围 | TMDB 参数 | 用户设置 |
|------|---------|----------|---------|
| locale | 元数据语言（标题、简介） | `language` | `settings.locale` |
| streamingRegion | 流媒体提供商（各国不同） | `watch_region` | `settings.streamingRegion` |
| originalLanguage | 原始语言过滤（发现页） | `with_original_language` | `settings.originalLanguage` |

### 15.6 服务端 i18n（通知系统）

**文件**: `server/i18n/index.ts`

服务端也有独立的 i18n 系统，用于发送本地化通知：

```typescript
export function initI18n(): void {
  for (const locale of availableLocales) {
    const filePath = path.join(__dirname, `locale/${locale}.json`);
    const messages = JSON.parse(fs.readFileSync(filePath, 'utf-8'));

    intls.set(locale, createIntl({ locale, messages, defaultLocale: 'en' }, cache));
  }
}
```

**启动时加载** (`server/index.ts:87`)：
```typescript
initI18n();  // 服务启动时一次性加载所有语言的翻译文件到内存
```

**使用场景**：通知模板（Discord、Email、Telegram 等）中需要本地化的文本。

### 15.7 TVDB 的语言映射

**文件**: `server/api/tvdb/interfaces.ts` + `server/api/tvdb/index.ts:350-353`

TVDB 使用不同的语言编码体系，需要映射：

```typescript
const wantedTranslation = convertTmdbLanguageToTvdbWithFallback(
  language,                   // TMDB 格式: 'zh-CN'
  Tvdb.DEFAULT_LANGUAGE       // TVDB 默认: 'eng'
);
```

TMDB 使用 ISO 639-1（如 `zh-CN`），TVDB 使用三字母编码（如 `chi`/`eng`）。`convertTmdbLanguageToTvdbWithFallback` 负责转换，无匹配时回退到英文。

### 15.8 i18n 与缓存的关系

| i18n 类型 | 缓存影响 |
|-----------|---------|
| 前端 UI | 无影响（静态 JSON，无缓存） |
| TMDB Metadata | 不同 language 产生不同缓存 Key，多语言多倍缓存 |
| 服务端通知 | 无影响（内存 Map，启动时全量加载） |
| TVDB Metadata | 语言转换后作为 headers 参数，影响 TVDB 缓存 Key |

> **缓存膨胀风险**：10 个用户使用 10 种不同语言访问同一部电影，会产生 10 条 tmdb 缓存。但 node-cache 的 stdTTL 会自动清理，不会无限增长。

---

## 16. 通知 Channel Adapter 与缓存衔接

### 16.1 Adapter 架构

通知系统采用经典的 **Adapter 模式**，10 个 Channel 共享同一接口：

```
事件触发 (MediaRequestSubscriber / IssueSubscriber / ...)
    │
    ▼
NotificationManager.sendNotification(type, payload)
    │
    ├─ 遍历 activeAgents
    │   ├─ agent.shouldSend() → 检查 enabled + 配置完整性
    │   └─ agent.send(type, payload) → 发送通知
    │
    ▼ 各 Agent 独立处理
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐
│ Discord │ │  Email  │ │Telegram │ │  Webhook │ │  Slack   │ ...
└─────────┘ └─────────┘ └─────────┘ └──────────┘ └──────────┘
     │            │           │            │            │
  axios.post   SMTP/Nodemailer  axios.post   axios.post   axios.post
     │            │           │            │            │
  Discord API   SMTP Server  Telegram API  用户自定义URL  Slack API
```

**文件**: `server/lib/notifications/index.ts:92-115`

```typescript
class NotificationManager {
  private activeAgents: NotificationAgent[] = [];

  public sendNotification(type: Notification, payload: NotificationPayload): void {
    this.activeAgents.forEach((agent) => {
      if (agent.shouldSend()) {
        agent.send(type, payload);  // 各 Agent 独立、并行
      }
    });
  }
}
```

### 16.2 Agent 注册与初始化

**文件**: `server/index.ts:132-143`

所有 Agent 在服务启动时一次性注册，不传 settings 参数（延迟到运行时读取）：

```typescript
notificationManager.registerAgents([
  new DiscordAgent(),     // 无参构造 → 运行时 getSettings()
  new EmailAgent(),
  new GotifyAgent(),
  new NtfyAgent(),
  new PushbulletAgent(),
  new PushoverAgent(),
  new SlackAgent(),
  new TelegramAgent(),
  new WebhookAgent(),
  new WebPushAgent(),
]);
```

### 16.3 Settings 的延迟读取策略

**关键设计**：Agent 构造时不缓存 settings，每次 `send()` / `shouldSend()` 时实时读取。

**文件**: `server/lib/notifications/agents/discord.ts:82-90`

```typescript
class DiscordAgent extends BaseAgent<NotificationAgentDiscord> {
  protected getSettings(): NotificationAgentDiscord {
    if (this.settings) {
      return this.settings;  // 如果有注入的 settings（测试用）
    }
    const settings = getSettings();  // 运行时从 Settings 单例读取
    return settings.notifications.agents.discord;
  }
}
```

**为什么不用缓存？** 因为 `getSettings()` 本身就是内存单例（见第 17 章），读取成本极低（一次属性访问），且保证永远拿到最新配置（管理员修改设置后立即生效，无需重启或刷新缓存）。

### 16.4 通知中的 TMDB 元数据请求与缓存

发送通知时通常需要获取媒体标题和简介，这会触发 TMDB API 调用：

**文件**: `server/subscriber/MediaRequestSubscriber.ts:71-86`

```typescript
const tmdb = new TheMovieDb();

try {
  const movie = await tmdb.getMovie({ movieId: entity.media.tmdbId });
  // ↑ 命中 node-cache（TTL 12h），几乎不会穿透到 TMDB

  notificationManager.sendNotification(Notification.MEDIA_AVAILABLE, {
    subject: `${movie.title} (${movie.release_date.slice(0, 4)})`,
    image: `https://image.tmdb.org/t/p/w300${movie.poster_path}`,
    message: truncate(movie.overview, { length: 500 }),
    // ...
  });
} catch (e) {
  // TMDB 失败 → 不发通知（静默跳过）
  logger.error('Something went wrong sending media available notification');
}
```

**缓存影响**：
- 通知触发时调用的 `getMovie()` / `getTvShow()` 会命中 `tmdb` 缓存实例
- 如果之前已有用户访问过该媒体详情，缓存必然存在
- 只有在冷启动后首个通知触发时才会穿透到 TMDB

### 16.5 Discord Agent 详解

**发送流程** (`server/lib/notifications/agents/discord.ts:233-349`)：

```
send(type, payload)
    │
    ├─ 1. 检查 notifySystem + hasNotificationType → 跳过不匹配的类型
    │
    ├─ 2. 构建用户 @mention 列表
    │    ├─ notifyUser → 查该用户的 discordIds 设置
    │    └─ notifyAdmin → 查所有管理员用户的 discordIds
    │       ↑ 每次通知都查 User 表！无缓存！
    │
    ├─ 3. 确定语言
    │    ├─ useUserLocale=true → 使用被通知用户的 locale
    │    └─ useUserLocale=false → 使用 Discord Agent 配置的 locale
    │
    ├─ 4. buildEmbed(type, payload, locale) → 构建 Discord Embed
    │    └─ getIntl(locale) → 从 i18n 内存 Map 获取格式化器
    │
    └─ 5. axios.post(webhookUrl, payload) → 发送到 Discord
         └─ 失败 → logger.error + return false（不重试）
```

### 16.6 各 Channel 的缓存特点

| Channel | 自身缓存 | 依赖外部缓存 | Settings 读取 | 重试策略 |
|---------|---------|------------|-------------|---------|
| Discord | 无 | TMDB 元数据 (node-cache) | 每次实时读 | 无 |
| Email | 无 | TMDB 元数据 (node-cache) | 每次实时读 | 无 |
| Telegram | 无 | TMDB 元数据 (node-cache) | 每次实时读 | 无 |
| Webhook | 无 | TMDB 元数据 (node-cache) | 每次实时读 | 无 |
| Slack | 无 | TMDB 元数据 (node-cache) | 每次实时读 | 无 |
| WebPush | 无 | 无（纯推送） | 每次实时读 | 无 |
| Gotify | 无 | TMDB 元数据 (node-cache) | 每次实时读 | 无 |
| Ntfy | 无 | TMDB 元数据 (node-cache) | 每次实时读 | 无 |
| Pushbullet | 无 | TMDB 元数据 (node-cache) | 每次实时读 | 无 |
| Pushover | 无 | TMDB 元数据 (node-cache) | 每次实时读 | 无 |

**共同特征**：
- 无自身缓存：通知是即时推送，不需要缓存历史
- 无重试：发送失败只记日志，不重试
- Settings 延迟读取：保证配置变更实时生效
- 依赖 TMDB 缓存：获取媒体标题/海报时受益于 node-cache

### 16.7 通知中 User 查询的性能问题

Discord、Telegram、Email 的 `notifyAdmin` 路径每次都会查询所有用户：

```typescript
// discord.ts:271-291
const userRepository = getRepository(User);
const users = await userRepository.find();  // 全量查询！无缓存！
```

**风险**：用户量大时（1000+），每次通知都全量查 User 表。但通知频率本身不高（请求状态变更才触发），实际影响可控。

---

## 17. Admin 后台 Settings Store

### 17.1 架构：文件持久化的内存单例

Settings Store 采用 **JSON 文件持久化 + 内存单例** 模式，不使用数据库也不使用 node-cache：

```
┌────────────────────────────────────────┐
│         Settings 单例 (内存)            │
│                                        │
│  this.data: AllSettings                │ ← 内存中的完整配置树
│  this.saveLock: Promise                │ ← 防并发写入的锁
│                                        │
│  读取：直接返回 this.data 的属性         │
│  写入：merge → 内存更新 → 写文件         │
└──────────┬─────────────────────────────┘
           │ 启动时 load()
           │ 修改时 save()
           ▼
   CONFIG_DIRECTORY/settings.json         ← 磁盘持久化
```

### 17.2 单例获取

**文件**: `server/lib/settings/index.ts:888-896`

```typescript
let settings: Settings | undefined;

export const getSettings = (initialSettings?: AllSettings): Settings => {
  if (!settings) {
    settings = new Settings(initialSettings);  // 首次创建
  }
  return settings;  // 后续直接返回内存单例
};
```

**关键特性**：
- 进程内全局唯一，任何 `getSettings()` 调用都返回同一对象
- 没有缓存失效问题——它本身就是缓存
- 读取速度是对象属性访问，比数据库查询快 1000x+

### 17.3 加载流程

**文件**: `server/lib/settings/index.ts:812-871`

```typescript
public async load(overrideSettings?: AllSettings, raw = false): Promise<Settings> {
  // 1. 如有 override，直接替换内存数据
  if (overrideSettings) {
    this.data = overrideSettings;
    return this;
  }

  // 2. 从文件读取
  let data;
  try {
    data = await fs.readFile(SETTINGS_PATH, 'utf-8');
  } catch {
    await this.save();  // 文件不存在 → 用默认值创建
  }

  // 3. 解析 + 迁移 + 合并
  if (data && !raw) {
    const parsedJson = JSON.parse(data);
    const migratedData = await runMigrations(parsedJson, SETTINGS_PATH);  // 版本迁移
    const merged = mergeSettings(this.data, migratedData);               // 深度合并
    this.data = merged;
  }

  // 4. 补全缺失字段（apiKey、clientId、sessionSecret、vapidKeys）
  if (!this.data.main.apiKey) { this.data.main.apiKey = this.generateApiKey(); change = true; }
  if (!this.data.clientId)    { this.data.clientId = randomUUID(); change = true; }
  if (!this.data.sessionSecret) { this.data.sessionSecret = randomBytes(32).toString('hex'); change = true; }
  if (!this.data.vapidPublic || !this.data.vapidPrivate) { /* 生成 VAPID keys */ change = true; }

  if (change) { await this.save(); }  // 有变更 → 回写文件
  return this;
}
```

### 17.4 保存流程（防并发写入）

**文件**: `server/lib/settings/index.ts:873-885`

```typescript
public async save(): Promise<void> {
  const savePromise = this.saveLock.then(async () => {
    const tmp = SETTINGS_PATH + '.tmp';
    await fs.writeFile(tmp, JSON.stringify(this.data, undefined, ' '));  // 写临时文件
    await fs.rename(tmp, SETTINGS_PATH);  // 原子重命名
  });

  this.saveLock = savePromise.catch(() => {});  // 防止链断裂
  return savePromise;
}
```

**并发保护**：
- `saveLock` 是一个 Promise 链，每次 save 都等上一次完成后再执行
- 使用 `tmp → rename` 实现原子写入，避免写到一半崩溃导致文件损坏
- 即使某次 save 失败，`catch(() => {})` 保证链不中断

### 17.5 设置的深度合并

**文件**: `server/lib/settings/index.ts:12-15`

```typescript
const mergeSettings = <T>(current: T, incoming: Partial<T>): T =>
  mergeWith({}, current, incoming, (_objValue, srcValue) =>
    Array.isArray(srcValue) ? srcValue : undefined  // 数组直接替换，不合并
  ) as T;
```

**合并策略**：
- 对象：递归深度合并（新增字段保留，已有字段覆盖）
- 数组：直接替换（不合并数组元素，避免旧条目残留）
- 基本类型：直接覆盖

### 17.6 Settings 与缓存的关系

**Settings 自身不使用 node-cache**，但 Settings 的数据**影响缓存行为**：

| Setting 字段 | 影响的缓存行为 |
|-------------|--------------|
| `main.locale` | 默认 language 参数 → 影响 TMDB 缓存 Key |
| `main.discoverRegion` | 默认 watch_region → 影响 Discover 缓存 Key |
| `main.originalLanguage` | 默认 with_original_language → 影响 Discover 缓存 Key |
| `main.cacheImages` | 控制前端是否缓存图片到本地 |
| `metadataSettings.tv/anime` | 选择 TMDB/TVDB → 影响使用哪个缓存实例 |
| `notifications.agents.*` | 通知 Agent 配置（不缓存，实时读取） |
| `network.dnsCache` | DNS 缓存开关和 TTL |
| `network.apiRequestTimeout` | 外部 API 超时时间 |

**配置变更的即时性**：
- 修改 `main.locale` → 下次 API 请求使用新 locale → 产生新的缓存 Key → 穿透到 TMDB
- 旧 locale 的缓存条目自然过期（stdTTL），不需要主动失效

### 17.7 Settings 的公共接口

管理员设置和公共设置是不同的接口：

| 路由 | 权限 | 返回内容 |
|-----|------|---------|
| `/api/v1/settings/public` | 无需认证 | `fullPublicSettings`（只含安全字段） |
| `/api/v1/settings/main` | ADMIN | 完整 main 设置 |
| `/api/v1/settings/notifications` | ADMIN | 完整通知设置 |
| `/api/v1/settings/jobs` | ADMIN | 定时任务配置 |

**公共设置** (`server/lib/settings/index.ts:705-739`) 是从多个 settings 域聚合的安全子集，不暴露 API Key、SMTP 密码等敏感信息。

---

## 18. 首屏 Discover 接口的缓存衔接

### 18.1 前端首屏加载链路

```
用户访问首页 (/)
    │
    ▼ Next.js SSR
_app.tsx → getInitialProps
    │
    ├─ 1. useUser() → /api/v1/auth/me  (SWR 缓存)
    ├─ 2. useSettings() → /api/v1/settings/public  (SWR 缓存)
    │
    ▼ 客户端 hydration
Discover 组件挂载
    │
    ├─ useDiscover('/api/v1/discover/trending')
    │   ├─ SWR Infinite: initialSize=3 → 前 3 页并行请求
    │   ├─ dedupingInterval: 30000 → 30s 内相同请求合并
    │   └─ revalidateFirstPage: false → 首页不自动刷新
    │
    ├─ useDiscover('/api/v1/discover/movies')
    ├─ useDiscover('/api/v1/discover/tv')
    │
    └─ GenreSlider → /api/v1/discover/genreslider/movie
                      /api/v1/discover/genreslider/tv
```

### 18.2 前端 SWR Infinite 缓存策略

**文件**: `src/hooks/useDiscover.ts:67-95`

```typescript
useSWRInfinite<BaseSearchResult<T> & S>(
  (pageIndex, previousPageData) => {
    // 构建分页 URL：endpoint?page=N
    if (previousPageData && pageIndex + 1 > previousPageData.totalPages) {
      return null;  // 已到最后一页，停止加载
    }
    return `${endpoint}?page=${pageIndex + 1}&...`;
  },
  {
    initialSize: 3,             // 首屏并行加载前 3 页
    revalidateFirstPage: false, // 不自动重新验证首页
    dedupingInterval: 30000,    // 30s 内相同请求去重
    revalidateOnFocus: false,   // 切换标签不触发重新验证
  }
);
```

**首屏请求量**：
- Trending: 3 页 × 20 条 = 60 条（或 3 条 if mediaType=all）
- Movies: 3 页 × 20 条 = 60 条
- TV: 3 页 × 20 条 = 60 条
- Genre Slider (Movie): 1 次（内部 N 个 genre 并行）
- Genre Slider (TV): 1 次（内部 N 个 genre 并行）

**总计约 9+ 个并发 API 请求**，但 SWR 的 `dedupingInterval` 会合并 30s 内相同请求。

### 18.3 服务端 Discover 路由的缓存衔接

**文件**: `server/routes/discover.ts:97-182`

每个 Discover 请求经过以下缓存链路：

```
前端 SWR → /api/v1/discover/movies?page=1&language=zh-CN
    │
    ▼
Express 路由 (discoverRoutes.get('/movies'))
    │
    ├─ 1. createTmdbWithRegionLanguage(req.user)
    │     ├─ 读取用户 settings.streamingRegion → 否则全局 settings
    │     ├─ 读取用户 settings.originalLanguage → 否则全局 settings
    │     └─ new TheMovieDb({ discoverRegion, originalLanguage })
    │
    ├─ 2. tmdb.getDiscoverMovies({ page, language: req.locale, ... })
    │     │
    │     ├─ ExternalAPI.get('/discover/movie', { params: { page, language, ... } })
    │     │   │
    │     │   ├─ 生成 cacheKey:
    │     │   │   "https://api.themoviedb.org/3/discover/movie{"page":1,"language":"zh-CN","with_original_language":"zh","watch_region":"CN","headers":{}}"
    │     │   │
    │     │   ├─ cache.get(cacheKey) → 命中 → 直接返回
    │     │   │
    │     │   └─ 未命中 → axios-rate-limit 排队 → 请求 TMDB → cache.set(cacheKey, data, 21600)
    │     │
    │     └─ 返回 TmdbDiscoverMovieResponse
    │
    ├─ 3. Media.getRelatedMedia(user, tmdbIds)
    │     └─ 查询 SQLite/PG 数据库（本地 Media 表），不走 TMDB 缓存
    │         → 返回每条媒体的请求状态 (PENDING/AVAILABLE/etc.)
    │
    ├─ 4. 如果有 keywords → tmdb.getKeywordDetails() → 同样走缓存
    │
    └─ 5. mapMovieResult() → 组装最终响应
         └─ TMDB 数据 + 本地 Media 状态 → JSON 响应
```

### 18.4 缓存 Key 的多维度组合

Discover 接口的缓存 Key 由以下维度组合：

| 维度 | 来源 | 示例值 |
|------|------|-------|
| baseUrl | TheMovieDb 构造参数 | `https://api.themoviedb.org/3` |
| endpoint | API 方法 | `/discover/movie` |
| page | 前端分页参数 | `1` |
| language | req.locale → 前端 language query | `zh-CN` |
| originalLanguage | 用户/全局设置 | `zh` |
| watch_region | 用户 streamingRegion 设置 | `CN` |
| genre / studio / network | 前端筛选参数 | `28` |
| sortBy | 前端排序参数 | `popularity.desc` |
| certification* | 前端分级筛选 | `PG-13` |

**不同用户访问同一页面**：
- 用户 A（中文，中国区）和用户 B（英文，美国区）访问 `/discover/movies`
- 产生的缓存 Key 不同 → 各自独立缓存
- **不会互相命中**，但也不会互相干扰

### 18.5 Genre Slider 的 N+1 缓存问题

**文件**: `server/routes/discover.ts:834-876`

Genre Slider 是 Discover 页面中缓存最密集的接口：

```typescript
discoverRoutes.get('/genreslider/movie', async (req, res) => {
  const genres = await tmdb.getMovieGenres({ language });  // 1 次请求

  await Promise.all(
    genres.map(async (genre) => {
      const genreData = await tmdb.getDiscoverMovies({
        genre: genre.id.toString(),  // 每个体裁 1 次请求
      });
      mappedGenres.push({ id, name, backdrops });
    })
  );
});
```

**请求量**：
- 1 次 `getMovieGenres()` → 命中 `tmdb` 缓存（TTL 24h）
- N 次 `getDiscoverMovies({ genre })` → N 次缓存查询
- 电影通常 16 个体裁 → 最多 17 个 TMDB 请求
- **全部命中缓存时**：0 次穿透到 TMDB
- **冷启动时**：17 次穿透，受 axios-rate-limit 排队

### 18.6 首屏请求优化总结

| 优化手段 | 位置 | 效果 |
|---------|------|------|
| SWR dedupingInterval | 前端 | 30s 内相同请求合并，减少网络流量 |
| SWR initialSize=3 | 前端 | 首屏预加载 3 页，减少翻页延迟 |
| SWR revalidateFirstPage=false | 前端 | 首页不自动刷新，减少无谓请求 |
| node-cache (6-12h TTL) | 服务端 | TMDB 请求绝大多数命中缓存 |
| axios-rate-limit | 服务端 | 防止冷启动时打爆 TMDB |
| Media.getRelatedMedia | 数据库 | 本地状态查询不走 TMDB |
| Genre Slider 缓存 | 服务端 | 体裁数据 24h TTL，Discover 6h TTL |

**冷启动场景**（进程重启后首个用户访问）：
1. 所有 node-cache 为空
2. 首屏 ~9 个 API 请求 → 每个可能触发多次 TMDB 请求
3. Genre Slider 最严重（最多 17 次穿透）
4. axios-rate-limit 会将请求排队，按 50 RPS 限流
5. 用户可能感受到 2-5 秒延迟
6. 后续用户访问全部命中缓存

## 19. 关键设计特点总结

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
15. **Watchlist ETag 缓存**：Plex Watchlist 使用 HTTP ETag 条件请求，304 免传输
16. **多用户串行同步**：Watchlist 逐用户串行处理，Media 全局共享，Request 用户维度隔离
17. **无主动降级**：429 直接失败，依赖缓存屏障 + 定时轮询重试实现最终一致性
18. **双路径 i18n**：前端 UI (react-intl) 与后端 Metadata (TMDB language param) 独立，共享 locale 配置
19. **多语言缓存膨胀**：不同 language 产生不同缓存 Key，多语言用户场景下缓存空间倍增
20. **通知无缓存**：10 个 Channel Adapter 无自身缓存，依赖 TMDB node-cache + Settings 内存单例
21. **Settings 内存单例**：JSON 文件持久化 + 内存单例，读取零延迟，配置变更即时生效
22. **Settings 防并发写入**：Promise 链锁 + tmp→rename 原子写入，保证数据完整性
23. **Discover 首屏优化**：SWR Infinite initialSize=3 + dedupingInterval=30s + revalidateFirstPage=false

---

## 20. 代码位置速查表（完整版）

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
| **Watchlist 同步** | | |
| Watchlist 同步核心 | `server/lib/watchlistsync.ts` | 全部 |
| Watchlist 同步测试 | `server/lib/watchlistsync.test.ts` | 全部 |
| Watchlist 路由 | `server/routes/watchlist.ts` | 全部 |
| Watchlist Entity | `server/entity/Watchlist.ts` | 28-68 |
| Plex TV Watchlist ETag | `server/api/plextv.ts` | 271-312 |
| Watchlist 定时任务 | `server/job/schedule.ts` | 89-107 |
| **Rate Limit 降级** | | |
| 搜索静默降级 | `server/api/themoviedb/index.ts` | 153-172 |
| TVDB Provider 回退 | `server/api/tvdb/index.ts` | 168-196 |
| 路由层 500 错误处理 | `server/routes/movie.ts` | 46-56 |
| Scanner 容错（跳过） | `server/lib/scanners/plex/index.ts` | 218-224 |
| **i18n** | | |
| 服务端 i18n 初始化 | `server/i18n/index.ts` | 全部 |
| 可用语言列表 | `server/types/languages.ts` | 1-39 |
| Locale 中间件 | `server/middleware/auth.ts` | 36-38 |
| 前端 loadLocaleData | `src/pages/_app.tsx` | 28-105 |
| 前端 IntlProvider | `src/pages/_app.tsx` | 199-204 |
| 发现页语言/区域定制 | `server/routes/discover.ts` | 29-49 |
| TVDB 语言映射 | `server/api/tvdb/interfaces.ts` | - |
| **通知 Channel Adapter** | | |
| NotificationManager | `server/lib/notifications/index.ts` | 92-115 |
| Notification 接口/类型 | `server/lib/notifications/agents/agent.ts` | 全部 |
| Discord Agent | `server/lib/notifications/agents/discord.ts` | 全部 |
| Email Agent | `server/lib/notifications/agents/email.ts` | 全部 |
| Telegram Agent | `server/lib/notifications/agents/telegram.ts` | 全部 |
| Webhook Agent | `server/lib/notifications/agents/webhook.ts` | 全部 |
| Slack Agent | `server/lib/notifications/agents/slack.ts` | 全部 |
| Agent 注册 | `server/index.ts` | 132-143 |
| 通知触发 (MediaRequestSubscriber) | `server/subscriber/MediaRequestSubscriber.ts` | 68-86 |
| **Settings Store** | | |
| Settings 类 (单例) | `server/lib/settings/index.ts` | 395-896 |
| getSettings() 工厂函数 | `server/lib/settings/index.ts` | 888-896 |
| Settings.load() 加载 | `server/lib/settings/index.ts` | 812-871 |
| Settings.save() 防并发写入 | `server/lib/settings/index.ts` | 873-885 |
| mergeSettings 深度合并 | `server/lib/settings/index.ts` | 12-15 |
| fullPublicSettings 安全子集 | `server/lib/settings/index.ts` | 705-739 |
| Settings 迁移 | `server/lib/settings/migrator.ts` | 全部 |
| **Discover 首屏** | | |
| Discover 路由 (全部) | `server/routes/discover.ts` | 全部 |
| createTmdbWithRegionLanguage | `server/routes/discover.ts` | 29-49 |
| Genre Slider (Movie) | `server/routes/discover.ts` | 834-876 |
| Genre Slider (TV) | `server/routes/discover.ts` | 878-920 |
| Discover Watchlist | `server/routes/discover.ts` | 922-983 |
| useDiscover 前端 Hook | `src/hooks/useDiscover.ts` | 54-173 |
| SWR Infinite 配置 | `src/hooks/useDiscover.ts` | 89-94 |
