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

## 7. 设计要点总结（补充）

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
