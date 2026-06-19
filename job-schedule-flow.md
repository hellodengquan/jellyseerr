# 队列与 Job 调度链路分析

## 1. 定时任务（Scheduled Jobs）

### 1.1 核心调度机制

项目使用 `node-schedule` 库实现基于 cron 表达式的定时任务调度，核心代码位于 `server/job/schedule.ts`。

#### 调度入口
```typescript
// server/job/schedule.ts:33-263
export const startJobs = (): void => {
  const jobs = getSettings().jobs;
  // 根据媒体服务器类型（Plex/Jellyfin）注册不同的任务
  // ...
};
```

#### 任务定义结构
```typescript
// server/job/schedule.ts:20-29
interface ScheduledJob {
  id: JobId;                     // 任务唯一标识
  job: schedule.Job;            // node-schedule Job 实例
  name: string;                 // 任务名称
  type: 'process' | 'command';  // 任务类型
  interval: 'seconds' | 'minutes' | 'hours' | 'days' | 'fixed';  // 时间粒度
  cronSchedule: string;         // cron 表达式
  running?: () => boolean;      // 运行状态检查
  cancelFn?: () => void;        // 取消函数
}
```

### 1.2 任务列表与默认调度

| Job ID | 名称 | 默认 cron | 执行频率 | 类型 |
|--------|------|-----------|----------|------|
| `plex-recently-added-scan` | Plex 最近新增扫描 | `0 */5 * * * *` | 每 5 分钟 | process |
| `plex-full-scan` | Plex 全库扫描 | `0 0 3 * * *` | 每天 3:00 | process |
| `plex-refresh-token` | Plex Token 刷新 | 配置决定 | 固定时间 | process |
| `plex-watchlist-sync` | Plex 监视列表同步 | `0 */3 * * * *` | 每 3 分钟 | process |
| `jellyfin-recently-added-scan` | Jellyfin 最近新增扫描 | `0 */5 * * * *` | 每 5 分钟 | process |
| `jellyfin-full-scan` | Jellyfin 全库扫描 | `0 0 3 * * *` | 每天 3:00 | process |
| `radarr-scan` | Radarr 扫描 | `0 0 * * * *` | 每小时 | process |
| `sonarr-scan` | Sonarr 扫描 | `0 0 * * * *` | 每小时 | process |
| `availability-sync` | 媒体可用性同步 | `0 0 * * * *` | 每小时 | process |
| `download-sync` | 下载同步 | `0 * * * * *` | 每分钟 | command |
| `download-sync-reset` | 下载同步重置 | `0 0 1 * * *` | 每天 1:00 | command |
| `image-cache-cleanup` | 图片缓存清理 | `0 0 2 * * *` | 每天 2:00 | process |
| `process-blocklisted-tags` | 黑名单标签处理 | `0 0 0 * * *` | 每天 0:00 | process |

配置定义在 `server/lib/settings/index.ts:569-600`。

### 1.3 调度执行流程

```
┌─────────────────────────────────────────────────────────┐
│                   应用启动时                            │
│  startJobs() 被调用（server/index.ts）                  │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  从 settings.json 读取 jobs 配置                        │
│  根据 mediaServerType 决定启用 Plex 还是 Jellyfin 任务   │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  为每个任务调用 schedule.scheduleJob(cron, callback)    │
│  创建 ScheduledJob 对象存入 scheduledJobs 数组          │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  node-schedule 内部维护定时器，到达 cron 触发点时        │
│  执行对应 callback，调用具体业务逻辑（如 scanner.run()）  │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 互斥控制（Mutex Control）

### 2.1 AsyncLock - 基于媒体 ID 的细粒度锁

为避免并发创建相同媒体条目导致数据库唯一约束冲突，项目实现了基于 `EventEmitter` 的异步锁机制。

#### 核心实现
```typescript
// server/utils/asyncLock.ts:8-52
class AsyncLock {
  private locked: { [key: string]: boolean } = {};
  private ee = new EventEmitter();

  private acquire = async (key: string) => {
    return new Promise((resolve) => {
      if (!this.locked[key]) {
        this.locked[key] = true;
        return resolve(undefined);
      }
      // 已锁定，监听解锁事件
      const nextAcquire = () => {
        if (!this.locked[key]) {
          this.locked[key] = true;
          this.ee.removeListener(key, nextAcquire);
          return resolve(undefined);
        }
      };
      this.ee.on(key, nextAcquire);
    });
  };

  private release = (key: string): void => {
    delete this.locked[key];
    setImmediate(() => this.ee.emit(key));
  };

  public dispatch = async (
    key: string | number,
    callback: () => Promise<void>
  ) => {
    const skey = String(key);
    await this.acquire(skey);
    try {
      await callback();
    } finally {
      this.release(skey);
    }
  };
}
```

#### 工作原理
1. **acquire**: 尝试获取锁，如果已被锁定则通过 `EventEmitter` 等待解锁事件
2. **release**: 释放锁，通过 `setImmediate` 异步通知等待者（避免释放过程阻塞）
3. **dispatch**: 封装 acquire -> execute -> release 的完整流程，使用 `finally` 确保锁一定会被释放

#### 使用场景
在 `BaseScanner` 中，每个扫描器实例都拥有自己的 `AsyncLock` 实例：

```typescript
// server/lib/scanners/baseScanner.ts:67
readonly asyncLock = new AsyncLock();

// server/lib/scanners/baseScanner.ts:113
protected async processMovie(tmdbId: number, options: ProcessOptions = {}): Promise<void> {
  await this.asyncLock.dispatch(tmdbId, async () => {
    // 查询是否已存在 -> 创建或更新 -> 保存
    // 同一 tmdbId 的操作会被串行化
  });
}

// server/lib/scanners/baseScanner.ts:287
protected async processShow(tmdbId: number, ...): Promise<void> {
  await this.asyncLock.dispatch(tmdbId, async () => {
    // 剧集处理逻辑
  });
}
```

### 2.2 Scanner 运行状态互斥

扫描器基类通过 `running` 标志和 `sessionId` 机制防止重复运行：

```typescript
// server/lib/scanners/baseScanner.ts:646-672
protected startRun(): string {
  const sessionId = randomUUID();
  this.sessionId = sessionId;
  this.running = true;
  return sessionId;
}

// server/lib/scanners/baseScanner.ts:687-725
protected async loop(processFn, { start = 0, end = this.bundleSize, sessionId }): Promise<void> {
  if (!this.running) {
    throw new Error('Sync was aborted.');
  }
  if (this.sessionId !== sessionId) {
    throw new Error('New session was started. Old session aborted.');
  }
  // ... 处理逻辑
}
```

在 `schedule.ts` 中，任务注册时绑定 `running` 检查函数：
```typescript
// server/job/schedule.ts:54-55
running: () => plexRecentScanner.status().running,
cancelFn: () => plexRecentScanner.cancel(),
```

---

## 3. 退避与限流策略（Backoff & Rate Limiting）

本项目没有使用显式的指数退避库（`exponential-backoff` 只是 `node-gyp` 的间接依赖），而是通过多种机制组合实现稳定性保障。

### 3.1 API 限流（Rate Limiting）

使用 `axios-rate-limit` 对外部 API 调用进行限流：

```typescript
// server/api/externalapi.ts:45-50
if (options.rateLimit) {
  this.axios = rateLimit(this.axios, {
    maxRequests: options.rateLimit.maxRequests,
    maxRPS: options.rateLimit.maxRPS,
  });
}
```

### 3.2 请求超时控制

所有外部 API 请求都配置了超时时间：

```typescript
// server/api/servarr/base.ts:101
const timeout = getSettings().network.apiRequestTimeout;
// 传递给 ExternalAPI 构造函数

// server/api/externalapi.ts:36
timeout: options.timeout,
```

### 3.3 处理速率控制（Throttling）

#### 扫描器批量处理与延迟
`BaseScanner.loop()` 方法实现了分批处理和批次间延迟：

```typescript
// server/lib/scanners/baseScanner.ts:12-13
const BUNDLE_SIZE = 20;        // 每批处理数量
const UPDATE_RATE = 4 * 1000;  // 批次间隔 4 秒

// server/lib/scanners/baseScanner.ts:713-723
await new Promise<void>((resolve, reject) =>
  setTimeout(() => {
    this.loop(processFn, {
      start: start + this.bundleSize,
      end: end + this.bundleSize,
      sessionId,
    })
      .then(() => resolve())
      .catch((e) => reject(new Error(e.message)));
  }, this.updateRate)  // 每批处理后等待 updateRate 毫秒
);
```

#### TMDB API 调用限流
在黑名单标签处理器中，对 TMDB API 调用进行 250ms 延迟：

```typescript
// server/job/blocklistedTagsProcessor.ts:20
const TMDB_API_DELAY_MS = 250;

// server/job/blocklistedTagsProcessor.ts:127
await this.processResults(response, tag, type, em);
await new Promise((res) => setTimeout(res, TMDB_API_DELAY_MS));
```

### 3.4 缓存策略（Cache as Backoff）

通过多层缓存减少重复 API 调用，间接起到退避作用：

#### 固定 TTL 缓存
```typescript
// server/api/externalapi.ts:56-77
protected async get<T>(endpoint: string, config?: AxiosRequestConfig, ttl?: number): Promise<T> {
  const cacheKey = this.serializeCacheKey(endpoint, ...);
  const cachedItem = this.cache?.get<T>(cacheKey);
  if (cachedItem) {
    return cachedItem;  // 缓存命中，直接返回
  }
  const response = await this.axios.get<T>(endpoint, config);
  if (this.cache && ttl !== 0) {
    this.cache.set(cacheKey, response.data, ttl ?? DEFAULT_TTL); // 默认 5 分钟
  }
  return response.data;
}
```

#### Rolling 缓存（后台刷新）
对于不经常变化的数据，采用缓存即将过期时后台刷新的策略：

```typescript
// server/api/externalapi.ts:104-137
protected async getRolling<T>(endpoint: string, config?: AxiosRequestConfig, ttl?: number): Promise<T> {
  const cachedItem = this.cache?.get<T>(cacheKey);
  if (cachedItem) {
    const keyTtl = this.cache?.getTtl(cacheKey) ?? 0;
    // 缓存即将过期（剩余时间 < TTL - 10秒）时，后台异步刷新
    if (keyTtl - (ttl ?? DEFAULT_TTL) * 1000 < Date.now() - DEFAULT_ROLLING_BUFFER) {
      this.axios.get<T>(endpoint, config).then((response) => {
        this.cache?.set(cacheKey, response.data, ttl ?? DEFAULT_TTL);
      });
    }
    return cachedItem;  // 立即返回旧缓存
  }
  // ... 首次请求逻辑
}
```

### 3.5 下载队列同步的容错处理

`DownloadTracker` 在获取队列失败时仅记录日志，不抛出异常，任务会在下一个调度周期重试：

```typescript
// server/lib/downloadtracker.ts:110-116
try {
  await radarr.refreshMonitoredDownloads();
  const queueItems = await radarr.getQueue();
  // ... 处理队列
} catch {
  logger.error(`Unable to get queue from Radarr server: ${server.name}`, {
    label: 'Download Tracker',
  });
}
```

---

## 4. 完整调度链路示例：Plex 最近新增扫描

```
1. 定时触发 (cron: 0 */5 * * * *)
   │
   ▼
2. schedule.ts:47-53
   logger.info('Starting scheduled job: Plex Recently Added Scan')
   plexRecentScanner.run()
   │
   ▼
3. plex/index.ts:65-139
   const sessionId = this.startRun()  // 设置 running=true, 生成 sessionId
   for (const library of this.libraries) {
     const libraryItems = await this.plexClient.getRecentlyAdded(...)
     this.items = uniqWith(...)        // 按 ratingKey 去重
     await this.loop(this.processItem.bind(this), { sessionId })
   }
   │
   ▼
4. baseScanner.ts:687-725
   loop(processFn, { start, end, sessionId })
   ├─ 检查 !this.running → 抛出 "Sync was aborted"
   ├─ 检查 this.sessionId !== sessionId → 抛出 "New session was started"
   ├─ 调用 this.processItems(processFn, slicedItems)  // Promise.all 并行处理
   └─ setTimeout(UPDATE_RATE=4s) 后递归调用下一批
      │
      ▼
5. plex/index.ts 中的 processItem()
   ├─ 解析媒体的 tmdbId/tvdbId/imdbId
   └─ 调用 this.processMovie() 或 this.processShow()
      │
      ▼
6. baseScanner.ts:113
   this.asyncLock.dispatch(tmdbId, async () => {
     const existing = await this.getExisting(tmdbId, MediaType.MOVIE)
     if (existing) {
       // 更新现有媒体状态
     } else {
       // 创建新媒体条目
     }
     await mediaRepository.save(...)
   })
   │
   ▼
7. 完成所有批次后，更新 library.lastScan = Date.now()
   this.endRun(sessionId)  // 设置 running=false
```

---

## 5. 设计要点总结

| 机制 | 实现方式 | 关键文件 | 目的 |
|------|----------|----------|------|
| 定时调度 | node-schedule + cron 表达式 | `server/job/schedule.ts` | 按计划执行后台任务 |
| 媒体级互斥 | AsyncLock (EventEmitter) | `server/utils/asyncLock.ts` | 防止同一媒体重复创建导致 DB 唯一约束冲突 |
| 扫描器级互斥 | running 标志 + sessionId | `server/lib/scanners/baseScanner.ts` | 防止扫描任务重入 |
| API 限流 | axios-rate-limit | `server/api/externalapi.ts` | 避免触发第三方 API 限流 |
| 处理限流 | 分批 + setTimeout | `server/lib/scanners/baseScanner.ts` | 控制数据库写入压力 |
| TMDB 限流 | setTimeout(250ms) | `server/job/blocklistedTagsProcessor.ts` | 遵守 TMDB API 速率限制 |
| 容错退避 | 缓存 + 静默失败 + 下次重试 | `server/api/externalapi.ts`, `server/lib/downloadtracker.ts` | 避免单个失败导致整体任务崩溃 |
| 后台刷新 | getRolling 缓存策略 | `server/api/externalapi.ts` | 减少响应延迟，同时保持数据新鲜度 |
