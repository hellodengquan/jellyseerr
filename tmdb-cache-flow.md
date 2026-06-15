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

## 7. 关键设计特点总结

1. **双层缓存**：前端 SWR + 服务端 node-cache，最大限度减少 API 调用
2. **分级 TTL**：不同类型数据使用不同过期时间，平衡新鲜度和性能
3. **滚动刷新**：对配置类数据使用后台刷新，保证用户永远快速响应
4. **精细 Key**：包含所有参数（语言、地区、分页等），确保缓存正确性
5. **无请求合并**：服务端无并发去重，高并发场景下可能出现缓存击穿
6. **列表详情分离**：列表和详情使用不同 API 端点，缓存不共享
