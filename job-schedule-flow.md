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

## 5. 容器重启时的 Job 状态恢复机制

### 5.1 启动挂载点

容器重启后的 job 调度初始化链路全部挂在 `server/index.ts` 的 Next.js `app.prepare()` Promise 链中，执行顺序如下：

```typescript
// server/index.ts:62-156
app.prepare()
  .then(async () => {
    // 1. 数据库初始化与迁移
    await checkOverseerrMerge();
    const dbConnection = dataSource.isInitialized
      ? dataSource
      : await dataSource.initialize();
    if (process.env.NODE_ENV === 'production') {
      await dbConnection.runMigrations();
    }

    // 2. 加载配置文件（settings.json）—— cron 表达式从此处读取
    const settings = await getSettings().load();          // server/index.ts:84
    restartFlag.initializeSettings(settings);

    // 3. 网络/代理/DNS 缓存初始化
    // ... proxy, DNS cache setup ...

    // 4. 注册通知 Agent
    notificationManager.registerAgents([...]);            // server/index.ts:132

    // 5. 【关键挂载点】只有用户存在时才启动调度
    const userRepository = getRepository(User);
    const totalUsers = await userRepository.count();
    if (totalUsers > 0) {
      startJobs();                                         // server/index.ts:148
    } else {
      logger.info('Skipping starting the scheduled jobs...');
    }

    // 6. Express 路由与监听
    // ... routes setup, server.listen() ...
  })
```

### 5.2 状态恢复的设计取舍

Jellyseerr **不做任何"运行中状态持久化"恢复**，核心原因与设计如下：

#### （1）内存态即状态，不持久化
所有与 job 相关的运行状态全部是内存变量，重启即丢失，由下一轮调度自动覆盖：

| 状态对象 | 存储位置 | 重启后行为 |
|----------|----------|------------|
| `scheduledJobs[]` 数组 | `server/job/schedule.ts:31` 全局内存 | 重新 `startJobs()` 时重新 push，所有 timer 重新注册 |
| `BaseScanner.running` / `progress` / `sessionId` | 扫描器实例内存字段（如 `plexFullScanner`） | 重新创建时默认 `running=false`，无需恢复 |
| `AsyncLock.locked` / `ee` 队列 | `baseScanner.ts:67` 中的 `AsyncLock` 实例 | 锁随实例一起重置，无挂起等待者 |
| `DownloadTracker.radarrServers / sonarrServers` | `downloadtracker.ts:28-29` | `{}` 空对象，等 1 分钟后的 `download-sync` 填充 |
| `node-cache` 中的 API 缓存 | 各 `ExternalAPI` 的 nodeCache | 丢失，下一次请求重新拉取并缓存 |

#### （2）为什么不恢复进度？
- 数据库中**没有** `job_runs`、`scan_progress` 之类的持久化表，TypeORM 实体里只定义了 `Media`、`MediaRequest`、`Session`、`Blocklist` 等业务实体
- 扫描结果（媒体条目是否存在）本身就落在 `Media` 表里，具有**幂等性**——即便扫描中断，下一次扫描从 `getExisting()` 分支走 UPDATE 而非 INSERT，不会重复创建
- `lastScan` 时间戳持久化在 `settings.json` 的 `plex.libraries[].lastScan` 字段里（`plexRecentScanner` 每次跑完会写入），重启后"最近新增扫描"只拉取 `addedAt > lastScan - 10min` 的数据，不会因重启漏扫

```typescript
// server/lib/scanners/plex/index.ts:101-104
const libraryItems = await this.plexClient.getRecentlyAdded(
  library.id,
  library.lastScan
    ? { addedAt: library.lastScan - 1000 * 60 * 10 }  // 给 10 分钟缓冲，避免漏扫
    : undefined,
  library.type
);
```

#### （3）"假恢复"：startup 时是否立刻跑一次？
不跑。`startJobs()` **只注册定时回调**，不主动 `job.invoke()`，所以首次触发必须等 cron 到点：

```typescript
// server/job/schedule.ts:45-56
scheduledJobs.push({
  ...
  job: schedule.scheduleJob(
    jobs['plex-recently-added-scan'].schedule,  // 注册而非立即执行
    () => { plexRecentScanner.run(); }
  ),
  ...
});
```

如果用户想立刻触发，可以：
- 通过前端 Jobs 页面的 **Run Now** 按钮 → `POST /settings/jobs/:jobId/run`（见 §6.2）
- 或等下一次 cron 触发

#### （4）重启保护：running / sessionId 两层守卫
虽然不做进度恢复，但 `BaseScanner` 的两层机制保证了"旧进程中断 → 新进程启动时扫描器内部状态是干净的"：

- `running=false`（初始值）：如果用户在重启瞬间触发了手动扫描，`loop()` 第一轮检查会失败，避免半初始化执行
- `sessionId`（每个 `startRun()` 新生成 UUID）：保证如果上一次会话因重启中断，后续 `loop()` 递归调用全部 self-abort，不会继续跑老 session

---

## 6. 多触发路径：Cron 触发 vs Webhook/手动触发 vs 直接调用

### 6.1 三条触发路径总览

同一个扫描器（例如 `plexFullScanner.run()`）有**三条完全不同的入口**，其中前两条**共用同一个调度回调**，第三条**绕过调度直接调用实例方法**：

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        三种触发方式对比                                      │
├────────────────────────┬───────────────────────┬─────────────────────────────┤
│ Cron 定时触发           │ Web 手动触发（通用）    │ API 直调（专用）              │
├────────────────────────┼───────────────────────┼─────────────────────────────┤
│ node-schedule 内部     │ POST /settings/jobs/  │ POST /settings/plex/sync     │
│ timer 到点自动执行     │ :jobId/run            │ POST /settings/jellyfin/sync │
│                        │                       │ 等专用端点                    │
├────────────────────────┼───────────────────────┼─────────────────────────────┤
│ 走 scheduleJob 注册时  │ 走 Job.invoke() → 同  │ 直接 scanner.run()           │
│ 传入的 callback        │ 一个 callback          │ 不经过 scheduledJobs 数组    │
└────────────────────────┴───────────────────────┴─────────────────────────────┘
```

### 6.2 路径一：Cron 触发（标准链路）

注册位置：`server/job/schedule.ts` 中每个任务的 `schedule.scheduleJob(cron, callback)`

```typescript
// server/job/schedule.ts:45-53
job: schedule.scheduleJob(
  jobs['plex-recently-added-scan'].schedule,   // cron: '0 */5 * * * *'
  () => {
    logger.info('Starting scheduled job: Plex Recently Added Scan', { label: 'Jobs' });
    plexRecentScanner.run();                   // 直接调用扫描器单例的 run()
  }
),
```

**特点**：
- 到点由 `node-schedule` 的 `Invocation` 队列触发
- 不受 HTTP 请求上下文限制
- 失败**不重试**（扫描器自身 catch 住 + 下一轮调度重试）

### 6.3 路径二：通用手动/Webhook 触发（共用调度回调）

前端 **Run Now** 按钮最终调用 `POST /settings/jobs/:jobId/run`（`server/routes/settings/index.ts:671-689`），这是**所有 13 个 job 共用的通用入口**。关键代码：

```typescript
// server/routes/settings/index.ts:671-689
settingsRoutes.post<{ jobId: string }>('/jobs/:jobId/run', (req, res, next) => {
  const scheduledJob = scheduledJobs.find((job) => job.id === req.params.jobId);

  if (!scheduledJob) {
    return next({ status: 404, message: 'Job not found.' });
  }

  // 【差异化核心】：调用 node-schedule 的 Job.invoke()
  scheduledJob.job.invoke();   // ← 和 Cron 触发共用完全相同的 callback 函数

  return res.status(200).json({
    id: scheduledJob.id,
    running: scheduledJob.running ? scheduledJob.running() : false,
  });
});
```

**Cron 触发 vs 手动触发的差异化细节**：

| 维度 | Cron 触发 | 手动/Webhook 触发（invoke()） |
|------|----------|------------------------------|
| 执行的函数 | `scheduleJob` 注册的 callback（闭包内的 `scanner.run()`） | **同一个 callback**，完全共享 |
| 调用者 | node-schedule 内部 Job.fire() | HTTP 路由 → `job.invoke()` |
| 响应状态 | 无响应（后台任务，写日志） | 同步返回 `200` + 当前状态 JSON |
| `nextInvocation` 重置 | 到点后自动排下一次 | `invoke()` **不影响**下一次调度时间，cron 节奏不变 |
| `cancelFn` 检查 | 不检查 | 不检查（但 `running` 标志在 scanner.run() 内生效） |
| 日志前缀 | `label: 'Jobs'` + 完整任务名 | 完全相同（同 callback 内 logger 调用） |
| 互斥锁生效 | 是（scanner 内部 `running` + `asyncLock`） | 是（完全同前） |

> **关于"Webhook"说明**：Seerr 本身**没有**接收 Radarr/Sonarr/Plex 推送 Webhook 的 HTTP 端点（代码检索 `OnDownload/OnGrab/OnImport` 等关键词无命中）。真正的"外部触发"走的是这条通用 Run API：外部系统 POST `/settings/jobs/:jobId/run` 即可，效果与点 Run Now 完全一致。

### 6.4 路径三：专用 API 直调（绕过调度）

针对 Plex/Jellyfin 两个最重的扫描任务，另外提供了专用端点，**完全不经过 `scheduledJobs` 数组和 `node-schedule`**：

```typescript
// server/routes/settings/index.ts:261-268
settingsRoutes.post('/plex/sync', (req, res) => {
  if (req.body.cancel) {
    plexFullScanner.cancel();                      // 直调实例 cancel()
  } else if (req.body.start) {
    plexFullScanner.run();                         // ← 直接调实例的 run()，不走 invoke
  }
  return res.status(200).json(plexFullScanner.status());
});

// server/routes/settings/index.ts:427-434  Jellyfin 同理
settingsRoutes.post('/jellyfin/sync', (req, res) => {
  if (req.body.cancel) {
    jellyfinFullScanner.cancel();
  } else if (req.body.start) {
    jellyfinFullScanner.run();
  }
  return res.status(200).json(jellyfinFullScanner.status());
});
```

与路径二的本质差异：

| 维度 | 通用 invoke()（路径二） | 专用 scanner.run()（路径三） |
|------|------------------------|-----------------------------|
| 调度层介入 | 经 `node-schedule::Job.invoke` | 完全绕过调度层 |
| 能触发的 job | 全部 13 个（只要在 `scheduledJobs` 数组里） | 只有 Plex/Jellyfin Full Scan 两个 |
| 执行对象 | 闭包内 `plexRecentScanner.run()` 等（不同实例） | 直接操作 `plexFullScanner` 单例 |
| 返回的 running 检查 | `scheduledJob.running?.()`（可能为 undefined） | `scanner.status()` 一定有值 |
| cancel 支持 | 需另走 `POST /jobs/:jobId/cancel` 端点 | 同一个端点 `{ cancel: true }` body 即可 |
| 前端入口 | Jobs 列表的 ▶ Run Now | Settings → Plex/Jellyfin 配置页的 Sync 按钮 |

### 6.5 三条路径最终的汇聚点

无论从哪条路径触发，最终都落到扫描器单例的 `run()` 上，由 §2 的互斥机制保证幂等与安全：

```
Cron timer
   │
   ├─ scheduleJob callback ──► plexFullScanner.run() ──┐
   │                                                    │
POST /jobs/:jobId/run ───► job.invoke() ───────────────┤
   │                                                    ├─► startRun()
   │                                                    │    │
POST /plex/sync  ──────────────────────────────────────┘    └─► loop()
                                                                  │
                                                            asyncLock.dispatch()
                                                                  │
                                                             processMovie()/Show()
```

---

## 8. Job 失败后的 Alert 通知机制

### 8.1 通知系统架构

通知采用 **Manager + Agent** 模式，`NotificationManager` 持有所有注册的 Agent（Discord/Telegram/Slack/Pushbullet/Pushover/Webhook 等），发送时遍历所有满足条件的 Agent：

```typescript
// server/lib/notifications/index.ts:92-114
class NotificationManager {
  private activeAgents: NotificationAgent[] = [];

  public sendNotification(type: Notification, payload: NotificationPayload): void {
    this.activeAgents.forEach((agent) => {
      if (agent.shouldSend()) {
        agent.send(type, payload);  // 每个 agent 内部异步，fire-and-forget
      }
    });
  }
}
```

**支持的通知类型**（`server/lib/notifications/index.ts:6-20`）：
```
MEDIA_PENDING = 2        // 新请求待审批
MEDIA_APPROVED = 4       // 请求已批准
MEDIA_AVAILABLE = 8      // 媒体已下载可用
MEDIA_FAILED = 16        // 请求处理失败 ← 我们关注的类型
TEST_NOTIFICATION = 32   // 测试
MEDIA_DECLINED = 64      // 请求被拒绝
MEDIA_AUTO_APPROVED = 128
ISSUE_CREATED = 256
ISSUE_COMMENT = 512
ISSUE_RESOLVED = 1024
ISSUE_REOPENED = 2048
MEDIA_AUTO_REQUESTED = 4096
```

Agent 注册挂载点（`server/index.ts:132`）：
```typescript
notificationManager.registerAgents([
  new DiscordAgent(settings.notifications.agents.discord),
  new TelegramAgent(settings.notifications.agents.telegram),
  new SlackAgent(settings.notifications.agents.slack),
  new PushoverAgent(settings.notifications.agents.pushover),
  new PushbulletAgent(settings.notifications.agents.pushbullet),
  new EmailAgent(settings.notifications.agents.email),
  new WebhookAgent(settings.notifications.agents.webhook),
  new GotifyAgent(settings.notifications.agents.gotify),
  new PushNotificationAgent(),
  new LunaSeaAgent(settings.notifications.agents.lunasea),
]);
```

### 8.2 MEDIA_FAILED 通知的两个触发挂载点

`MEDIA_FAILED` 只针对"**请求被批准后提交到 Radarr/Sonarr 失败**"这一业务场景，**扫描器/同步 job 本身的失败（API 超时、网络错误等）不会发通知，只写日志**。

两个挂载点都在 `MediaRequestSubscriber`（TypeORM 的 `AfterInsert` / `AfterUpdate` 生命周期 subscriber）里，对应两种失败路径：

#### 挂载点 1：异步 Promise 链中的 .catch()（Radarr/Sonarr 服务端返回错误）

```typescript
// server/subscriber/MediaRequestSubscriber.ts:376-432
radarr
  .addMovie(radarrMovieOptions)        // 异步提交，不阻塞 HTTP 响应
  .then(async (radarrMovie) => {
    // 成功：写回 externalServiceId 等字段
    media.externalServiceId = radarrMovie.id;
    await mediaRepository.save(media);
  })
  .catch(async () => {                 // 失败：触发 MEDIA_FAILED 通知
    // 1. 把请求状态标记为 FAILED 并持久化
    if (entity.status !== MediaRequestStatus.FAILED) {
      entity.status = MediaRequestStatus.FAILED;
      await requestRepository.save(entity);
    }

    // 2. 写 warn 日志（含请求 ID、媒体 ID、radarrMovieOptions）
    logger.warn('Something went wrong sending movie request to Radarr, marking status as FAILED', {...});

    // 3. 【关键】发送失败通知
    MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
  })
```

**触发场景**：Radarr/Sonarr 响应 4xx/5xx、请求体被拒绝、API key 无效、服务端内部错误等。

#### 挂载点 2：同步 try/catch（连接/配置错误）

```typescript
// server/subscriber/MediaRequestSubscriber.ts:446-472
} catch (e) {
  // 连接错误、配置缺失（无默认 Sonarr/Radarr server）等
  entity.status = MediaRequestStatus.FAILED;
  await requestRepository.save(entity);

  logger.warn('Failed to send movie request to Radarr due to connection or configuration error...', {...});

  MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
}
```

**触发场景**：网络超时、DNS 解析失败、Radarr/Sonarr 服务不可达、找不到默认 server 配置等。

### 8.3 通知构造与分发：MediaRequest.sendNotification()

静态方法 `MediaRequest.sendNotification()`（`server/entity/MediaRequest.ts:736-837`）是所有通知类型（含 MEDIA_FAILED）的统一构造器：

```typescript
// server/entity/MediaRequest.ts:736-837
static async sendNotification(entity: MediaRequest, media: Media, type: Notification) {
  const tmdb = new TheMovieDb();
  try {
    // 根据类型设置 event 文案、notifyAdmin、notifySystem
    switch (type) {
      case Notification.MEDIA_FAILED:
        event = `${entity.is4k ? '4K ' : ''}${mediaType} Request Failed`;
        break;  // notifyAdmin 和 notifySystem 保持 true（通知管理员和系统）
      // ...其他类型
    }

    // 拉取 TMDB 元数据（标题、简介、海报）构造完整 payload
    if (entity.type === MediaType.MOVIE) {
      const movie = await tmdb.getMovie({ movieId: media.tmdbId });
      notificationManager.sendNotification(type, {
        media,
        request: entity,
        notifyAdmin: true,      // MEDIA_FAILED 会通知管理员
        notifySystem: true,     // 也会推到系统通知
        notifyUser: undefined,  // 不单独通知请求用户（通过 admin/system 渠道覆盖）
        event,
        subject: `${movie.title} (${year})`,
        message: truncate(movie.overview, 500),
        image: `https://image.tmdb.org/t/p/w600_and_h900_bestv2${movie.poster_path}`,
      });
    }
    // TV show 同理
  } catch (e) {
    // 【自保护】：发送通知本身失败时，只写日志，不向上抛
    logger.error('Something went wrong sending media notification(s)', {...});
  }
}
```

### 8.4 其他 Job 失败的处理方式（只记日志，不发通知）

对比扫描器/同步类 job，它们的错误处理**不会触发任何通知 Agent**，完全依赖日志：

| Job 类型 | 失败位置 | 处理方式 | 发通知？ |
|----------|----------|----------|----------|
| Plex/Jellyfin 扫描（processItem） | `plex/index.ts:209-224` catch | `this.log('Failed to process Plex media', 'error', {...})` | ❌ |
| Plex/Jellyfin 扫描（paginateLibrary） | `plex/index.ts:203` `.catch` 后 reject | 冒泡到 `run()` 的 try/finally，finally 只调 `endRun()` | ❌ |
| Radarr/Sonarr Scanner | `baseScanner.ts` 的 `loop()` | 同样只写日志 | ❌ |
| DownloadTracker.update() | `downloadtracker.ts:110-116` | `logger.error('Unable to get queue from Radarr server...')` | ❌ |
| AvailabilitySync.run() | `availabilitySync.ts` | logger.error + throw | ❌ |
| WatchlistSync.sync() | `watchlistsync.ts` | logger.error | ❌ |
| Blocklist tag processor | `blocklistedTagsProcessor.ts` | logger.error | ❌ |

**设计意图**：扫描/同步类 job 是周期性的（1分钟~1小时），失败后下一轮调度自然重试，通知会刷屏；只有"用户请求被批准但提交到下载器失败"这种需要人工介入的业务失败才推 Alert。

---

## 9. 多 Worker 部署下的任务幂等保证

### 9.1 核心结论：Jellyseerr 设计为单进程部署

代码库中搜索 `cluster`、`worker_threads`、`fork`、`redis`、`bull`、`agenda`、`bee-queue` 等关键词均未命中真实的分布式队列或多 worker 协调逻辑。实际部署模式是：

> **单 Node.js 进程（Next.js + Express）+ 单 node-schedule 调度器 + 内存态锁与缓存**

如果用户强行用 PM2 cluster 模式或 k8s 多副本部署，会存在 §9.4 列出的风险。但代码仍然在**数据库层 + 业务层**设计了多层幂等保护，足以保证数据不脏。

### 9.2 四层幂等保护

```
┌──────────────────────────────────────────────────────────────────────┐
│                     四层幂等保护（从上到下强度递增）                      │
├──────────────────────────────────────────────────────────────────────┤
│  第 1 层：进程内 AsyncLock（内存态，仅单进程有效）                       │
│  第 2 层：业务前置查询（getExisting + 重复请求检查）                     │
│  第 3 层：数据库唯一约束（跨进程的最后硬防线）                            │
│  第 4 层：UPDATE 幂等语义（重复 save() 只覆盖字段，不会新增记录）          │
└──────────────────────────────────────────────────────────────────────┘
```

#### 第 1 层：进程内 AsyncLock（单进程有效）

`server/utils/asyncLock.ts`，以 `tmdbId` 为 key，单进程内把"查询 → 判断 → 写入"变成原子操作。详见 §2.1。

> ⚠️ 多进程下失效：每个进程有独立的 `AsyncLock` 实例，不同进程对同一 `tmdbId` 的操作不会互相阻塞。

#### 第 2 层：业务前置查询

**扫描器侧**（`server/lib/scanners/baseScanner.ts:113` 的 `processMovie()`）：
```typescript
await this.asyncLock.dispatch(tmdbId, async () => {
  const existing = await this.getExisting(tmdbId, MediaType.MOVIE);
  if (existing) {
    // UPDATE 分支：只更新字段，不新建
    existing.status = ...;
    existing.mediaAddedAt = ...;
    await mediaRepository.save(existing);  // 幂等更新
  } else {
    // INSERT 分支：新建 Media
    const newMedia = new Media({...});
    await mediaRepository.save(newMedia);
  }
});
```

**请求侧**（`server/entity/MediaRequest.ts:171-216` 的 `MediaRequest.request()`）：
```typescript
// 查重 1：同媒体 + 同分辨率 已存在非 DECLINED/COMPLETED 的请求 → 拒绝
const existing = await requestRepository.createQueryBuilder('request')
  .leftJoinAndSelect('request.media', 'media')
  .where('request.is4k = :is4k', { is4k: requestBody.is4k })
  .andWhere('media.tmdbId = :tmdbId', { tmdbId: tmdbMedia.id })
  .andWhere('media.mediaType = :mediaType', { mediaType })
  .getMany();

if (existing[0].status !== DECLINED && existing[0].status !== COMPLETED) {
  throw new DuplicateMediaRequestError('Request for this media already exists.');
}

// 查重 2：同用户 + 同媒体 的自动请求 → 拒绝
if (existing.find(r => r.requestedBy.id === requestUser.id && r.isAutoRequest)) {
  throw new DuplicateMediaRequestError('Auto-request for this media and user already exists.');
}
```

#### 第 3 层：数据库唯一约束（跨进程硬防线）

从 Postgres 初始迁移（`server/migration/postgres/1734786061496-InitialMigration.ts`）和实体定义汇总：

| 表 | 唯一约束 | 代码位置 | 作用 |
|----|----------|----------|------|
| `media` | `UQ_41a289eb...` UNIQUE (`tvdbId`) | `server/entity/Media.ts:94` | 同剧集不能插入两条 |
| `media` | `@Index(['tmdbId', 'mediaType'])`（联合索引，查 existing 用） | `server/entity/Media.ts:30` | 逻辑唯一键（不是 DB unique，但业务层强依赖） |
| `watchlist` | `UNIQUE_USER_DB` UNIQUE (`tmdbId`, `requestedById`) | migration:35 | 同用户的同监视列表条目不重复 |
| `blacklist` | `UQ_6bbafa28...` UNIQUE (`tmdbId`) | migration:8 | 同 tmdbId 只能拉黑一次 |
| `user_settings` | UNIQUE (`userId`) | migration:8 | 每个用户只有一份设置 |
| `user_push_subscription` | `UQ_6427d07d...` UNIQUE (`endpoint`, `userId`) + UNIQUE (`auth`) | migration:9 | 推送订阅去重 |
| `media_request` | **没有数据库级 unique** | — | 依赖业务层第 2 层检查 + 状态机防止重复 |

> ⚠️ 注意 `media` 表的 `(tmdbId, mediaType)` 只建了 `@Index` 没建 `UNIQUE`，如果 AsyncLock 在多进程下失效，极端情况下可能插入两条同 tmdbId 同 mediaType 的记录（但 tvdbId unique 会挡住剧集）。

#### 第 4 层：UPDATE 的幂等语义

即便真的重复触发了处理流程，`save()` 对已有实体是 UPDATE，只覆盖字段不会新增：

```typescript
// 例如 availabilitySync.run() 中：
media.status = MediaStatus.AVAILABLE;
await mediaRepository.save(media);  // 重复执行 N 次结果相同
```

`Media` 实体没有"自增计数器"或"累加型字段"，全部是 set-this-value 的覆盖写，天然幂等。

### 9.3 扫描器去重（单进程内）

除了 AsyncLock，扫描器本身还通过三层机制保证单进程内不重入：

```
BaseScanner.startRun() → running=true, sessionId=new UUID()
        │
        ▼
loop() 入口双重检查：
  1. if (!this.running) throw "Sync was aborted"
  2. if (this.sessionId !== sessionId) throw "New session was started. Old session aborted"
        │
        ▼
processMovie() / processShow()
  → AsyncLock.dispatch(tmdbId, callback)  // 媒体级串行化
```

**多进程下的效果**：每个进程独立维护 `running` 和 `sessionId`，所以多实例部署时同一扫描任务会在每个实例各跑一遍。但最终落 DB 时由第 2/3/4 层幂等保护兜底，不会产生重复数据，只是 API 调用翻倍、浪费资源。

### 9.4 多 Worker 强行部署的风险与已知限制

| 风险 | 原因 | 是否有兜底 |
|------|------|------------|
| **cron 任务重复执行 N 次**（N=实例数） | `node-schedule` 是进程内存态，每个实例独立注册 timer | ✅ 扫描/同步幂等，数据不会脏；但 Radarr/Plex/TMDB API 调用翻倍，可能触发对方限流 |
| **AsyncLock 跨进程失效** | 每个进程独立 EventEmitter 实例 | ✅ DB 唯一约束兜底；媒体表 `(tmdbId, mediaType)` 非 unique 有理论重复风险 |
| **缓存不一致** | 各实例独立 node-cache | ✅ 仅性能影响，不影响数据正确性 |
| **DownloadTracker 进度状态漂移** | 各实例独立维护内存态 Map | ⚠️ 前端看到的下载进度可能在多个实例间跳动 |
| **running 状态无法跨进程取消** | 每个实例独立 running 标志 | ⚠️ 在 A 实例点 Cancel，B 实例仍继续跑；最终结果不脏但浪费资源 |

**官方建议**：Jellyseerr/Overseerr 的 Docker 官方部署示例均为单容器。多实例部署需自行前置 Nginx 会话保持 + 单实例专跑定时任务（如通过环境变量 `DISABLE_SCHEDULED_JOBS=true` 在非主实例屏蔽 `startJobs()`，但代码中目前**没有**这个开关，需自行改造）。

---

## 11. Job 执行时间统计与 Metric 暴露

### 11.1 执行时间统计策略

**核心结论**：代码中没有显式的 `Date.now() - startTime` 执行时间记录逻辑，也没有集成 Prometheus / StatsD 等 metric 库。时间相关的统计全部**通过日志 + 进度跟踪**间接暴露。

#### （1）进度跟踪而非耗时跟踪

`BaseScanner` 定义了三个核心字段，用于前端展示扫描进度（而非耗时）：

```typescript
// server/lib/scanners/baseScanner.ts:59-61
protected progress = 0;      // 当前已处理的偏移量（start 参数）
protected items: T[] = [];   // 本批要处理的所有条目
protected totalSize?: number = 0;  // 总数（Plex API 返回的 totalSize）
```

`StatusBase` 接口（`server/lib/scanners/baseScanner.ts:15-19`）统一了所有扫描器的状态结构：
```typescript
export type StatusBase = {
  running: boolean;    // 是否在运行
  progress: number;    // 已处理数量
  total: number;       // 总数
};
```

#### （2）各 Job 的 status() 实现

所有实现 `RunnableScanner` 接口的类都有 `status()` 方法，返回运行状态 + 扩展信息：

| Job 类型 | status() 返回结构 | 代码位置 |
|----------|-------------------|----------|
| Plex 扫描 | `{ running, progress, total, currentLibrary, libraries }` | `plex/index.ts:55-63` |
| Jellyfin 扫描 | `{ running, progress, total, currentLibrary, libraries }` | `jellyfin/index.ts:542-550` |
| Radarr 扫描 | `{ running, progress, total, currentServer, servers }` | `radarr/index.ts:36-44` |
| Sonarr 扫描 | `{ running, progress, total, currentServer, servers }` | `sonarr/index.ts:44-52` |
| Blocklist Tags Processor | `{ running, progress, total }` | `blocklistedTagsProcessor.ts:49-54` |
| AvailabilitySync | 无 status()，仅 `public running = false` | `availabilitySync.ts:23` |
| DownloadTracker | 无 status()，内存态 Map | `downloadtracker.ts` |
| WatchlistSync | 无 status() | `watchlistsync.ts` |

#### （3）Metric 暴露口：`GET /settings/jobs`

所有 job 的状态通过 HTTP API 暴露给前端（`server/routes/settings/index.ts:657-669`）：

```typescript
settingsRoutes.get('/jobs', (_req, res) => {
  return res.status(200).json(
    scheduledJobs.map((job) => ({
      id: job.id,
      name: job.name,
      type: job.type,
      interval: job.interval,
      cronSchedule: job.cronSchedule,
      nextExecutionTime: job.job.nextInvocation(),  // 下一次执行时间
      running: job.running ? job.running() : false,   // 当前是否在运行
    }))
  );
});
```

**暴露的字段解读**：
- `nextExecutionTime`：由 `node-schedule` 的 `Job.nextInvocation()` 方法计算返回 Date 对象
- `running`：由每个 job 在注册时绑定的 `running` getter 提供
  - 扫描器类 job：`running: () => plexRecentScanner.status().running`
  - command 类 job（如 download-sync）：没有 `running` getter，永远返回 `false`

#### （4）日志中的隐式耗时统计

虽然没有显式统计，但可以从日志时间戳推断耗时。扫描器 `startRun()` 和 `endRun()` 都会写日志：

```typescript
// server/lib/scanners/baseScanner.ts:651
this.log('Scan starting', 'info', { sessionId });

// server/lib/scanners/plex/index.ts:147
this.log(this.isRecentOnly ? 'Recently Added Scan Complete' : 'Full Scan Complete', 'info');
```

通过分析日志中同 `sessionId` 的 "Scan starting" 到 "Scan Complete" / "Scan interrupted" 的时间差，可以计算出实际耗时。

#### （5）缺失的 Metric 能力

- ❌ 没有 Prometheus exporter（没有 `prom-client` / `prometheus-api-metrics` 等依赖）
- ❌ 没有 `execution_time_ms` / `job_duration_seconds` 等 counter/gauge
- ❌ 没有成功率/失败率统计
- ❌ 没有 `/metrics` 端点

> **设计取舍**：Jellyseerr 定位为轻量级家庭媒体请求系统，不面向大规模监控场景。运行状态通过前端 Jobs 页面的进度条展示即可，无需专业 metric 系统。

---

## 12. Plex Sync 中途断网时的恢复策略

### 12.1 三层超时保护

断网恢复依赖三层超时机制，从 API 层到业务层逐层兜底：

#### 第 1 层：全局 API 请求超时（可配置）

```typescript
// server/lib/settings/index.ts:184, 629
export interface NetworkSettings {
  apiRequestTimeout: number;  // 默认 10000ms（10秒），用户可在设置中修改
}
```

所有 API 客户端（Plex、Radarr、Sonarr、TMDB 等）创建时都会传入这个 timeout：

```typescript
// server/api/servarr/base.ts:101-111
const timeout = getSettings().network.apiRequestTimeout;
super(url, { apikey: apiKey }, {
  nodeCache: cacheManager.getCache(cacheName).data,
  timeout,  // ← 10 秒超时，断网后最多等待 10 秒就抛错
});

// server/api/externalapi.ts:33-42
this.axios = axios.create({
  baseURL: baseUrl,
  timeout: options.timeout,  // ← 所有 ExternalAPI 子类共享
  // ...
});
```

#### 第 2 层：单条目错误隔离（不影响其他条目）

`processItem()` 内层有独立 try/catch，单个媒体处理失败不会终止整个扫描：

```typescript
// server/lib/scanners/plex/index.ts:208-224
private async processItem(plexitem: PlexLibraryItem) {
  try {
    if (plexitem.type === 'movie') {
      await this.processPlexMovie(plexitem);
    } else if (plexitem.type === 'show' || ...) {
      await this.processPlexShow(plexitem);
    }
  } catch (e) {
    // 只写 error 日志，不 throw，继续处理下一个
    this.log('Failed to process Plex media', 'error', {
      errorMessage: e.message,
      title: plexitem.title,
    });
  }
}
```

**断网场景下的效果**：处理到第 N 个条目时断网 → 第 N 个条目 catch 住打日志 → 继续处理第 N+1 个 → 第 N+1 个也超时打日志 → ... 直到本批全部处理完（每个都超时 10 秒）。

#### 第 3 层：顶层 try/catch/finally 保证状态重置

`run()` 方法的顶层错误处理，确保无论发生什么错误，`running` 标志都会被重置：

```typescript
// server/lib/scanners/plex/index.ts:65-159
public async run(): Promise<void> {
  const sessionId = this.startRun();  // running = true
  try {
    // ... 所有业务逻辑 ...
    // getRecentlyAdded() / paginateLibrary() / processItem() 都在这里
    // 任何一步抛未 catch 的错误都会跳到 catch
  } catch (e) {
    this.log('Scan interrupted', 'error', { errorMessage: e.message });
  } finally {
    this.endRun(sessionId);  // ← 无论成功失败，一定重置 running = false
  }
}
```

### 12.2 断网场景下的具体行为

以 **Plex Full Sync 进行到第 3 批（已处理 120/5000 条）时突然断网** 为例：

```
时间线：
├─ T0: run() 开始，startRun() → running=true, sessionId=uuid1
├─ T1: 第 1 批 (0-19) 处理完成
├─ T2: 第 2 批 (20-39) 处理完成
├─ T3: 第 3 批 (40-59) 处理中，第 42 条时断网
│    ├─ 第 42 条：await this.plexClient.getMetadata(...) → 等待 10s 超时
│    ├─ catch 打 error 日志 → 继续第 43 条
│    ├─ 第 43-59 条：每条都超时 10s → 累计 18*10s = 180s
├─ T4: 第 3 批处理完，setTimeout(4s) 后调 paginateLibrary(60-79)
│    ├─ this.plexClient.getLibraryContents(..., { offset: 60 }) → 超时 10s
│    ├─ throw new Error('getaddrinfo EAI_AGAIN plex.tv')
│    ├─ 冒泡到 run() 的 catch → log('Scan interrupted', ...)
├─ T5: finally 块执行 endRun(sessionId) → running=false
└─ 扫描终止，已处理的 41 条数据已落库，进度丢失
```

### 12.3 恢复策略：断点续传 + 幂等性

扫描中断后，**没有断点续传（不会从第 43 条继续）**，但通过两个机制保证最终一致性：

#### （1）lastScan 时间戳 + 10 分钟缓冲（适用于 Recently Added 扫描）

```typescript
// server/lib/scanners/plex/index.ts:98-107
const libraryItems = await this.plexClient.getRecentlyAdded(
  library.id,
  library.lastScan
    ? { addedAt: library.lastScan - 1000 * 60 * 10 }  // ← 10 分钟缓冲
    : undefined,
  library.type
);
```

**恢复效果**：下一次扫描（5 分钟后 cron 触发，或手动 Run Now）只拉取 `addedAt > lastScan - 10min` 的媒体，中断期间新增的不会漏掉。

#### （2）Full Scan 的幂等恢复

全量扫描没有 `lastScan` 偏移，下一次启动会从 offset=0 重新开始。但由于：
- `getExisting(tmdbId)` 前置查询 → 已处理的走 UPDATE 分支
- 所有字段是覆盖写而非累加 → 重复处理结果相同
- `AsyncLock` 保证单进程内同媒体不并发写

**恢复效果**：重新扫描会重复处理前 41 条，但数据库结果完全一致，只是浪费一些 API 调用和时间。

#### （3）"失败就放弃，下次再说"的设计哲学

代码中**没有**任何重试逻辑：
- ❌ 没有指数退避（`exponential-backoff` 只是 node-gyp 的间接依赖，实际未使用）
- ❌ 没有 `retry: 3` 配置
- ❌ 没有死信队列（DLQ）

**设计意图**：Plex/Radarr/Sonarr 都是本地/局域网服务，网络中断通常是短暂的（重启路由器、短暂掉线）。1 分钟~1 小时的 cron 周期已经足够让服务恢复，重试反而可能加剧网络拥塞。

### 12.4 AvailabilitySync 的额外兜底："宁漏勿错"

可用性同步有一个特殊的断网保护逻辑：如果 API 返回非 404 错误（如 500/超时/网络错误），**假设媒体仍然存在**，避免因网络问题误标记为已删除：

```typescript
// server/lib/availabilitySync.ts:711-724
} catch (ex) {
  if (!ex.message.includes('404')) {
    existsInRadarr = true;  // ← 非 404 错误，不删除，避免误判
    logger.debug('Failure retrieving... from Radarr.', { ... });
  }
}

// Plex 同理（server/lib/availabilitySync.ts:934-947）
} catch (ex) {
  if (!ex.message.includes('404')) {
    existsInPlex = true;    // ← 宁漏勿错
    preventSeasonSearch = true;
  }
}

// Jellyfin 更保守（server/lib/availabilitySync.ts:1071-1085）
} catch (ex) {
  if (!ex.message.includes('404') && !ex.message.includes('500')) {
    existsInJellyfin = true;  // ← 连 500 都排除
  }
}
```

---

## 13. 设计要点总结（最终完整版）

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
| **启动挂载** | `app.prepare()` → `startJobs()`，仅 `totalUsers > 0` 时注册 | `server/index.ts:62-156` | 保证配置/DB 就绪后再注册 timer |
| **重启恢复策略** | 内存态不持久化 + lastScan 持久化 + 幂等扫描 | `server/index.ts`, `plex/index.ts:103` | 无状态设计，重启后由 cron 或手动触发重新跑 |
| **通用手动触发** | `job.invoke()` 复用 cron 注册的 callback | `server/routes/settings/index.ts:671-689` | Run Now / 外部 webhook 触发所有 job，不影响调度节奏 |
| **专用扫描直调** | 直接调用 `scanner.run()` 绕过调度层 | `server/routes/settings/index.ts:261-268`, `427-434` | Plex/Jellyfin 全量扫描的专用启停入口 |
| **失败 Alert 通知** | TypeORM Subscriber 的 AfterUpdate 生命周期 → `MEDIA_FAILED` → `notificationManager.sendNotification()` | `server/subscriber/MediaRequestSubscriber.ts:398-472`, `server/entity/MediaRequest.ts:736-837` | 仅在 Radarr/Sonarr 提交请求失败时推给管理员；扫描类 job 失败只写日志 |
| **多 worker 幂等** | 四层保护：AsyncLock → 前置查询 → DB 唯一约束 → UPDATE 幂等 | `server/utils/asyncLock.ts`, `server/entity/Media.ts`, `server/migration/postgres/1734786061496-InitialMigration.ts` | 保证单实例部署数据正确；多实例不脏数据但重复跑任务 |
| **状态 Metric 暴露** | `GET /settings/jobs` + 各 scanner.status() 返回 `{ running, progress, total, nextExecutionTime }` | `server/routes/settings/index.ts:657-669`, `baseScanner.ts:15-24` | 前端 Jobs 页面展示进度和下次执行时间；无 Prometheus 等专业 metric |
| **断网恢复** | 三层超时（10s API 超时 + 单条目 catch 隔离 + 顶层 finally 重置 running）+ lastScan 10min 缓冲 + 幂等重跑 | `plex/index.ts:65-159`, `baseScanner.ts:208-224`, `settings/index.ts:629` | 断网时安全终止，不脏数据；下一次 cron 自动恢复，Recently Added 扫描不漏 |
| **可用性同步防误删** | 非 404 错误假设媒体存在（宁漏勿错） | `availabilitySync.ts:711-724`, `934-947`, `1071-1085` | 防止网络波动导致已下载媒体被误标记为 DELETED |
