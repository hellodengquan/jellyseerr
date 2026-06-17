# Jellyseerr 点播请求全链路代码分析

## 一、整体流程概述

用户点播影视后的完整链路如下：

```
用户前端点播
    ↓
[API 层] POST /api/v1/request
    ↓
[鉴权层] 会话/API Key 认证 + 权限校验
    ↓
[业务层] MediaRequest.request() - 请求创建
    ├─ 权限检查 (REQUEST / REQUEST_4K 等)
    ├─ 配额检查 (movie/tv quota)
    ├─ 重复请求检查
    ├─ 黑名单检查
    ├─ 应用覆盖规则 (OverrideRule)
    └─ 保存 Media + MediaRequest 实体
    ↓
[事件订阅] MediaRequestSubscriber
    ├─ AfterInsert / AfterUpdate 触发
    ├─ sendToRadarr() / sendToSonarr() - 异步发送到媒体服务器
    └─ updateParentStatus() - 更新父级 Media 状态
    ↓
[通知层] NotificationManager
    ├─ AfterInsert 触发 notifyNewRequest()
    ├─ AfterUpdate 触发 notifyApprovedOrDeclined()
    └─ 遍历所有通知 Agent (Discord/Webhook/Email 等) 发送通知
    ↓
[定时任务层] 周期性扫描
    ├─ DownloadTracker - 每分钟拉取下载队列
    ├─ *arr Scanner - 每日扫描 Radarr/Sonarr 库
    ├─ Plex/Jellyfin Scanner - 扫描媒体服务器库
    └─ AvailabilitySync - 定期检查媒体可用性
    ↓
[状态回写] 扫描器更新本地 Media 状态
    └─ BaseScanner.processMovie() / processShow()
        ├─ AsyncLock 防止并发冲突
        ├─ 更新 Media.status / Media.status4k
        ├─ 更新 Season 状态
        └─ 触发 MEDIA_AVAILABLE 通知
```

---

## 二、鉴权机制实现

### 2.1 认证中间件

**文件**: `server/middleware/auth.ts:9-58`

支持两种认证方式：

1. **API Key 认证** (X-API-Key 头)
   ```typescript
   if (req.header('X-API-Key') === settings.main.apiKey) {
     // 可选 X-API-User 头指定操作用户
     const userId = req.header('X-API-User') ? Number(...) : 1;
     user = await userRepository.findOne({ where: { id: userId } });
   }
   ```

2. **会话认证** (Session Cookie)
   ```typescript
   if (req.session?.userId) {
     user = await userRepository.findOne({ where: { id: req.session.userId } });
   }
   ```

### 2.2 权限系统

**文件**: `server/lib/permissions.ts:1-74`

采用**位掩码权限**设计，每个权限是 2 的幂次方，支持组合权限检查：

```typescript
export enum Permission {
  NONE = 0,
  ADMIN = 2,
  MANAGE_REQUESTS = 16,
  REQUEST = 32,
  AUTO_APPROVE = 128,
  REQUEST_4K = 1024,
  REQUEST_4K_MOVIE = 2048,
  REQUEST_4K_TV = 4096,
  REQUEST_ADVANCED = 8192,
  // ... 其他权限
}
```

权限检查函数支持 `AND` / `OR` 模式：
```typescript
hasPermission(
  permissions: Permission | Permission[],
  value: number,
  options: { type: 'and' | 'or' } = { type: 'and' }
): boolean
```

**关键权限检查点**:
- `server/entity/MediaRequest.ts:80-112` - 请求前检查 `REQUEST` / `REQUEST_4K` 权限
- `server/entity/MediaRequest.ts:354-367` - 自动审批检查 `AUTO_APPROVE` 权限
- `server/routes/request.ts:668` - 审批操作检查 `MANAGE_REQUESTS` 权限

### 2.3 配额系统

**文件**: `server/entity/MediaRequest.ts:114-120`

在请求创建前检查用户配额：
```typescript
const quotas = await requestUser.getQuota();
if (requestBody.mediaType === MediaType.MOVIE && quotas.movie.restricted) {
  throw new QuotaRestrictedError('Movie Quota exceeded.');
}
```

---

## 三、节流与队列机制

### 3.1 扫描器节流

**文件**: `server/lib/scanners/baseScanner.ts:687-725`

采用**分批处理 + 间隔等待**实现节流：

```typescript
protected async loop(
  processFn: (item: T) => Promise<void>,
  {
    start = 0,
    end = this.bundleSize,  // 默认 20 条/批
    sessionId,
  } = {}
): Promise<void> {
  // 处理当前批次
  await this.processItems(processFn, slicedItems);
  
  // 等待后递归处理下一批 (默认间隔 4 秒)
  await new Promise<void>((resolve) =>
    setTimeout(() => {
      this.loop(processFn, {
        start: start + this.bundleSize,
        end: end + this.bundleSize,
        sessionId,
      }).then(() => resolve());
    }, this.updateRate)  // 默认 4000ms
  );
}
```

### 3.2 API 速率限制

**文件**: `server/api/externalapi.ts:45-50`

使用 `axios-rate-limit` 对外部 API 调用进行限流：
```typescript
if (options.rateLimit) {
  this.axios = rateLimit(this.axios, {
    maxRequests: options.rateLimit.maxRequests,
    maxRPS: options.rateLimit.maxRPS,
  });
}
```

#### 3.2.1 Rate-Limit 分布式部署分析

**当前实现的局限性**:
- `axios-rate-limit` 基于**内存计数器**实现，不支持分布式部署
- 多实例部署时，每个实例独立计数，无法共享限流状态
- 可能导致总请求量超过外部 API 的实际限制

**技术原理**:
`axios-rate-limit` 内部维护一个请求队列和时间窗口计数器：
```typescript
// axios-rate-limit 内部原理（示意）
const requestQueue = [];
let requestCount = 0;
let windowStart = Date.now();

// 新请求进入时检查窗口
if (Date.now() - windowStart > 1000) {
  windowStart = Date.now();
  requestCount = 0;
}

if (requestCount < maxRPS) {
  requestCount++;
  return axios(config);  // 立即执行
} else {
  return new Promise(resolve => {
    requestQueue.push({ config, resolve });  // 排队等待
  });
}
```

**分布式部署的潜在方案**:
1. **集中式限流**: 使用 Redis 实现分布式令牌桶或漏桶算法
2. **实例分流**: 通过负载均衡按 API Key 或用户 ID 做会话粘滞，确保同一外部服务的请求路由到同一实例
3. **配额分摊**: 将总配额除以实例数，每个实例独立限制
4. **降级策略**: 当检测到 429 Too Many Requests 时，触发全局熔断并在所有实例间同步

**当前代码中的规避措施**:
- `server/lib/cache.ts` 的多级缓存设计减少重复请求
- `getRolling()` 方法的后台刷新策略分散请求峰值
- 扫描器的 `updateRate` 间隔配置避免突发流量

#### 3.2.2 Rate-Limit 多节点 Redis 同步改造方案

**现状分析**:
- TypeORM 的 peerDependencies 中声明了 `ioredis ^5.0.4` 和 `redis ^3.1.1`，但 Jellyseerr 核心代码**未实际使用 Redis**
- `node-cache`（`server/lib/cache.ts`）和 `axios-rate-limit` 均为纯内存实现

**Redis 分布式令牌桶改造架构**:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Node A      │     │  Node B      │     │  Node N      │
│ (axios-     │     │ (axios-     │     │ (axios-     │
│  rate-      │     │  rate-      │     │  rate-      │
│  limit)     │     │  limit)     │     │  limit)     │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            ▼
                    ┌───────────────┐
                    │    Redis      │
                    │  ┌─────────┐  │
                    │  │令牌桶键 │  │
                    │  │ - radarr:│  │
                    │  │  tokens  │  │
                    │  │ - sonarr:│  │
                    │  │  tokens  │  │
                    │  │ - tmdb:  │  │
                    │  │  tokens  │  │
                    │  └─────────┘  │
                    └───────────────┘
```

**Redis Lua 脚本实现原子令牌桶**:
```typescript
// 建议的分布式限流脚本（需改造 externalapi.ts）
const acquireTokenScript = `
  local key = KEYS[1]
  local capacity = tonumber(ARGV[1])
  local rate = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])

  local state = redis.call('HMGET', key, 'tokens', 'timestamp')
  local tokens = tonumber(state[1]) or capacity
  local timestamp = tonumber(state[2]) or now

  local elapsed = now - timestamp
  local refilled = math.floor(elapsed * rate / 1000)
  tokens = math.min(capacity, tokens + refilled)
  timestamp = now

  if tokens > 0 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'timestamp', timestamp)
    redis.call('PEXPIRE', key, 60000)  -- 1 分钟自动过期
    return 1
  end
  return 0
`;

// 键命名空间：rate_limit:tmdb、rate_limit:radarr:server1
// 每个 API Key 独立限流，避免互相影响
```

**多节点同步策略**:
1. **读多写少场景**: 本地缓存 50ms，每 50ms 批量同步 Redis（减少网络开销）
2. **写入失败降级**: Redis 不可用时自动 fallback 到内存限流，日志标记 `fallback=memory`
3. **预热加载**: 节点启动时从 Redis 加载当前令牌数，避免冷启动过载

---

### 3.3 AsyncLock 死锁检测机制

**文件**: `server/utils/asyncLock.ts:1-54`

防止同一媒体的并发写入冲突，基于 EventEmitter 实现：

```typescript
public dispatch = async (
  key: string | number,    // 通常是 tmdbId
  callback: () => Promise<void>
) => {
  const skey = String(key);
  await this.acquire(skey);  // 获取锁，等待其他操作完成
  try {
    await callback();        // 执行业务逻辑
  } finally {
    this.release(skey);      // 释放锁
  }
};
```

#### 3.3.1 死锁风险分析

**死锁形成条件**:
1. **互斥持有**: `locked[key] = true` 标记持有锁
2. **循环等待**: 嵌套调用 `dispatch()` 且 key 顺序相反时可能发生
3. **不可抢占**: 锁只能由持有者释放
4. **持有并等待**: 持有锁 A 时等待锁 B

**死锁预防设计**:

1. **无界监听器预警**:
   ```typescript
   constructor() {
     this.ee.setMaxListeners(0);  // 移除默认 10 个监听器限制
   }
   ```
   > **风险**: 虽然移除了限制，但大量等待者（>1000）可能暗示潜在死锁。实际部署中应监控 `ee.listenerCount(key)`，超过阈值时告警。

2. **非递归锁设计**:
   - 同一 key 不可重入，会导致永久阻塞
   - 设计要求：`dispatch` 回调内严禁再次调用同一 key 的 `dispatch`

3. **`setImmediate` 释放策略**:
   ```typescript
   private release = (key: string): void => {
     delete this.locked[key];
     setImmediate(() => this.ee.emit(key));  // 异步释放，避免同步递归
   };
   ```
   - 将释放操作放到事件循环的下一个 tick
   - 防止 `dispatch` 嵌套调用导致的同步死锁

**死锁检测机制（需运维侧补充）**:
```typescript
// 建议增加的监控逻辑
public getDeadlockCandidates(timeoutMs = 30000): string[] {
  const now = Date.now();
  const candidates: string[] = [];
  for (const [key, locked] of Object.entries(this.locked)) {
    if (locked && (this.lockAcquiredAt[key] || 0) < now - timeoutMs) {
      candidates.push(key);
    }
  }
  return candidates;
}
```

**典型死锁场景示例**:
```typescript
// ❌ 错误：嵌套调用同一 key 会死锁
asyncLock.dispatch(123, async () => {
  await asyncLock.dispatch(123, async () => { /* ... */ });  // 永久等待
});

// ✅ 正确：避免嵌套，或使用不同的锁粒度
```

#### 3.3.2 AsyncLock 跨进程死锁识别

**现状分析**:
当前 `AsyncLock` 基于进程内 `EventEmitter`，多实例部署时：
- 实例 A 持有锁 key=123
- 实例 B 无法感知，也会尝试获取锁
- 导致两个实例同时操作同一 tmdbId，触发 SQLite `UNIQUE constraint failed` 错误

**Redis Redlock 分布式锁 + 死锁识别改造方案**:

```typescript
// 建议的跨进程锁识别（需改造 asyncLock.ts）
class DistributedAsyncLock {
  private processLocks: Map<string, { pid: string; acquiredAt: number; stack: string }> = new Map();
  private readonly REDIS_KEY_PREFIX = 'jellyseerr:lock:';
  private readonly DEADLOCK_THRESHOLD = 30000;  // 30 秒超时

  public dispatch = async (key: string | number, callback: () => Promise<void>) => {
    const skey = String(key);
    const lockValue = `${process.pid}:${this.instanceId}:${Date.now()}`;

    // 1. 先检查跨进程死锁：查询 Redis 是否有长期未释放的锁
    const existingLock = await this.redis.get(this.REDIS_KEY_PREFIX + skey);
    if (existingLock) {
      const [remotePid, remoteInstanceId, acquiredTimestamp] = existingLock.split(':');
      const heldMs = Date.now() - Number(acquiredTimestamp);

      if (heldMs > this.DEADLOCK_THRESHOLD) {
        // 死锁检测：超过 30 秒未释放，强制解锁
        const released = await this.redis.del(this.REDIS_KEY_PREFIX + skey);
        logger.warn('Deadlock detected and force-released', {
          label: 'AsyncLock',
          lockKey: skey,
          heldBy: { remotePid, remoteInstanceId, heldMs },
          released: released > 0,
          fingerprint: `deadlock:${skey}:${acquiredTimestamp}`,  // Sentry 幂等键
        });
      } else {
        // 正常等待：订阅 Redis Pub/Sub 释放通知
        await this.waitForRedisLock(skey);
      }
    }

    // 2. 获取分布式锁（SET NX PX 原子操作）
    const acquired = await this.redis.set(
      this.REDIS_KEY_PREFIX + skey,
      lockValue,
      'PX',
      this.DEADLOCK_THRESHOLD,  // 自动过期兜底
      'NX'
    );

    if (!acquired) {
      throw new Error(`Lock contention detected for key ${skey}`);
    }

    // 3. 记录进程内堆栈，用于本地死锁诊断
    this.processLocks.set(skey, {
      pid: process.pid.toString(),
      acquiredAt: Date.now(),
      stack: new Error().stack || '',  // 捕获获取锁时的调用栈
    });

    try {
      await callback();
    } finally {
      // 4. 释放锁（先比对锁持有者，避免误删）
      const currentHolder = await this.redis.get(this.REDIS_KEY_PREFIX + skey);
      if (currentHolder === lockValue) {
        await this.redis.del(this.REDIS_KEY_PREFIX + skey);
      }
      this.processLocks.delete(skey);

      // 5. Pub/Sub 通知等待者
      await this.redis.publish(`${this.REDIS_KEY_PREFIX}release:${skey}`, lockValue);
    }
  };

  // 跨进程死锁诊断 API
  public getDeadlockReport(): DeadlockReport[] {
    const now = Date.now();
    const reports: DeadlockReport[] = [];

    for (const [key, info] of this.processLocks.entries()) {
      const heldMs = now - info.acquiredAt;
      if (heldMs > this.DEADLOCK_THRESHOLD * 0.8) {  // 80% 阈值时预警
        reports.push({
          key,
          heldMs,
          warning: heldMs > this.DEADLOCK_THRESHOLD ? 'DEADLOCK_SUSPECTED' : 'APPROACHING_TIMEOUT',
          stackTrace: info.stack,
          pid: info.pid,
        });
      }
    }
    return reports;
  }
}
```

**死锁识别键 (Sentry Fingerprint) 映射表**:
| 场景 | Fingerprint 模板 | Sentry 级别 |
|------|-----------------|-------------|
| 本地持有超 80% 阈值 | `['asyncLock-warning', key, instanceId]` | warning |
| Redis 锁超 30s 强制释放 | `['asyncLock-deadlock-force-release', key, acquiredTimestamp]` | error |
| 获取锁竞争失败超过 5 次/分钟 | `['asyncLock-contention-high', key]` | warning |
| Redlock 多节点不一致 | `['asyncLock-redlock-inconsistency', key]` | fatal |

### 3.4 异步锁 (AsyncLock) 使用场景

**文件**: `server/utils/asyncLock.ts:1-54`

防止同一媒体的并发写入冲突，基于 EventEmitter 实现：

```typescript
public dispatch = async (
  key: string | number,    // 通常是 tmdbId
  callback: () => Promise<void>
) => {
  const skey = String(key);
  await this.acquire(skey);  // 获取锁，等待其他操作完成
  try {
    await callback();        // 执行业务逻辑
  } finally {
    this.release(skey);      // 释放锁
  }
};
```

**使用场景**:
- `BaseScanner.processMovie()` - 电影处理时按 tmdbId 加锁
- `BaseScanner.processShow()` - 剧集处理时按 tmdbId 加锁

### 3.5 缓存机制

**文件**: `server/lib/cache.ts:1-91`

使用 `node-cache` 为不同外部 API 配置独立缓存：

| 缓存 ID | TTL | 用途 |
|---------|-----|------|
| tmdb | 21600s (6h) | TMDB API 数据 |
| radarr | 300s (5min) | Radarr API 数据 |
| sonarr | 300s (5min) | Sonarr API 数据 |
| plexguid | 604800s (7d) | Plex GUID 映射 |

**滚动缓存策略** (`externalapi.ts:104-137`):
- 缓存过期前 10 秒触发后台刷新
- 立即返回旧数据，不阻塞用户请求

---

## 四、媒体服务器同步逻辑

### 4.1 请求发送到 Radarr/Sonarr

**文件**: `server/subscriber/MediaRequestSubscriber.ts:183-818`

触发时机：`AfterInsert` 和 `AfterUpdate` 事件

```typescript
public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
  await this.sendToRadarr(event.entity as MediaRequest);
  await this.sendToSonarr(event.entity as MediaRequest);
  await this.updateParentStatus(event.entity as MediaRequest);
}
```

**异步发送设计** (关键在 `sendToRadarr` 方法):

```typescript
// 异步执行，不阻塞 UI
radarr
  .addMovie(radarrMovieOptions)
  .then(async (radarrMovie) => {
    // 成功：回写外部服务 ID
    media.externalServiceId = radarrMovie.id;
    media.externalServiceSlug = radarrMovie.titleSlug;
    media.serviceId = radarrSettings?.id;
    await mediaRepository.save(media);
  })
  .catch(async () => {
    // 失败：标记为 FAILED
    entity.status = MediaRequestStatus.FAILED;
    await requestRepository.save(entity);
    // 发送失败通知
    MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
  })
  .finally(() => {
    // 清除相关缓存
    radarr.clearCache({ tmdbId: movie.id, externalId: ... });
  });
```

**服务选择逻辑**:
1. 优先使用请求指定的 `serverId`
2. 否则查找 `isDefault && is4k === entity.is4k` 的默认服务器
3. 支持 `rootFolder`、`profileId`、`tags` 的请求级覆盖

### 4.2 下载进度追踪

**文件**: `server/lib/downloadtracker.ts:1-227`

定时任务 (`download-sync`) 每分钟执行一次：

```typescript
public updateDownloads() {
  this.updateRadarrDownloads();
  this.updateSonarrDownloads();
}

private async updateRadarrDownloads() {
  // 调用 Radarr API 刷新监控的下载
  await radarr.refreshMonitoredDownloads();
  // 获取队列信息
  const queueItems = await radarr.getQueue();
  // 缓存到内存
  this.radarrServers[server.id] = queueItems.map(...);
}
```

### 4.3 媒体库扫描

**文件**: `server/lib/scanners/baseScanner.ts:56-755`

三种扫描模式：
1. **全量扫描** (`*-full-scan`) - 每日执行，扫描整个媒体库
2. **最近扫描** (`*-recently-added-scan`) - 每 5 分钟执行，只扫新增内容
3. **Radarr/Sonarr 扫描** (`radarr-scan` / `sonarr-scan`) - 每日执行

处理逻辑 (`processMovie` 示例):
```typescript
protected async processMovie(
  tmdbId: number,
  options: ProcessOptions
): Promise<void> {
  await this.asyncLock.dispatch(tmdbId, async () => {
    const existing = await this.getExisting(tmdbId, MediaType.MOVIE);
    
    if (existing) {
      // 更新现有媒体状态
      if (!processing && hasFile) {
        existing.status = MediaStatus.AVAILABLE;
      }
      // ... 更新其他字段
      await mediaRepository.save(existing);
    } else {
      // 创建新媒体记录
      const newMedia = new Media({ ... });
      await mediaRepository.save(newMedia);
    }
  });
}
```

---

## 五、外部 API 通知机制

### 5.1 通知管理器

**文件**: `server/lib/notifications/index.ts:92-115`

```typescript
class NotificationManager {
  private activeAgents: NotificationAgent[] = [];

  public sendNotification(type: Notification, payload: NotificationPayload): void {
    this.activeAgents.forEach((agent) => {
      if (agent.shouldSend()) {
        agent.send(type, payload);  // 异步发送，不等待结果
      }
    });
  }
}
```

### 5.2 通知类型

**文件**: `server/lib/notifications/index.ts:6-20`

```typescript
export enum Notification {
  NONE = 0,
  MEDIA_PENDING = 2,           // 新请求待审批
  MEDIA_APPROVED = 4,          // 请求已批准
  MEDIA_AVAILABLE = 8,         // 媒体已可用
  MEDIA_FAILED = 16,           // 请求失败
  MEDIA_DECLINED = 64,         // 请求被拒绝
  MEDIA_AUTO_APPROVED = 128,   // 自动批准
  MEDIA_AUTO_REQUESTED = 4096, // 自动请求
}
```

### 5.3 通知触发点

| 触发时机 | 位置 | 通知类型 |
|---------|------|---------|
| 请求创建后 | `MediaRequest.afterInsert()`:629-655 | `MEDIA_PENDING` |
| 自动批准 | `MediaRequest.autoapprovalNotification()`:722-727 | `MEDIA_AUTO_APPROVED` |
| 状态更新 | `MediaRequest.afterUpdate()`:663-720 | `MEDIA_APPROVED` / `MEDIA_DECLINED` |
| 扫描发现可用 | `AvailabilitySync` + `BaseScanner` | `MEDIA_AVAILABLE` |
| 发送到 *arr 失败 | `MediaRequestSubscriber.sendToRadarr()`:398-432 | `MEDIA_FAILED` |

### 5.4 通知 Agent 实现

以 Webhook Agent 为例 (`server/lib/notifications/agents/webhook.ts`):

```typescript
public async send(
  type: Notification,
  payload: NotificationPayload
): Promise<boolean> {
  // 检查是否启用该通知类型
  if (!hasNotificationType(type, settings.types ?? 0)) {
    return true;
  }

  // 构建 payload，支持模板变量替换
  const body = this.buildPayload(type, payload);
  
  // 发送请求
  await axios.post(webhookUrl, body, { headers });
  
  return true;
}
```

**支持的通知 Agent**:
- Discord, Email, Gotify, Ntfy, Pushbullet, Pushover, Slack, Telegram, Webhook, WebPush

### 5.5 10 通知 Agent 扩展机制

#### 5.5.1 架构设计

**文件**: `server/lib/notifications/agents/agent.ts`

采用**抽象基类 + 接口**的双重抽象设计：

```typescript
// 接口定义契约
export interface NotificationAgent {
  shouldSend(): boolean;
  send(type: Notification, payload: NotificationPayload): Promise<boolean>;
}

// 抽象基类提供通用能力
export abstract class BaseAgent<T extends NotificationAgentConfig> {
  protected settings?: T;
  public constructor(settings?: T) {
    this.settings = settings;
  }

  protected abstract getSettings(): T;
}
```

#### 5.5.2 注册机制

**文件**: `server/index.ts:131-143` + `server/lib/notifications/index.ts:95-97`

应用启动时批量注册所有 Agent：

```typescript
// server/index.ts - 启动时注册
notificationManager.registerAgents([
  new DiscordAgent(),
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

// server/lib/notifications/index.ts - 注册实现
public registerAgents = (agents: NotificationAgent[]): void => {
  this.activeAgents = [...this.activeAgents, ...agents];
  logger.info('Registered notification agents', { label: 'Notifications' });
};
```

#### 5.5.3 配置类型系统

**文件**: `server/lib/settings/index.ts:220-347`

每个 Agent 有独立的配置接口，通过 `NotificationAgentKey` 枚举关联：

```typescript
export enum NotificationAgentKey {
  DISCORD = 'discord',
  EMAIL = 'email',
  GOTIFY = 'gotify',
  NTFY = 'ntfy',
  PUSHBULLET = 'pushbullet',
  PUSHOVER = 'pushover',
  SLACK = 'slack',
  TELEGRAM = 'telegram',
  WEBHOOK = 'webhook',
  WEBPUSH = 'webpush',
}

// 基类配置
export interface NotificationAgentConfig {
  enabled: boolean;
  embedPoster: boolean;
  types?: number;        // 位掩码，控制接收哪些通知类型
  options: Record<string, unknown>;
}

// 子类扩展配置
export interface NotificationAgentDiscord extends NotificationAgentConfig {
  options: {
    botUsername?: string;
    botAvatarUrl?: string;
    webhookUrl: string;
    locale: AvailableLocale;
  };
}

export interface NotificationAgentWebhook extends NotificationAgentConfig {
  options: {
    webhookUrl: string;
    jsonPayload: string;  // 支持自定义 JSON 模板
    authHeader?: string;
    method: 'POST' | 'PUT' | 'PATCH';
  };
}
```

#### 5.5.4 新增 Agent 扩展步骤

1. **创建 Agent 类**，继承 `BaseAgent` 并实现 `NotificationAgent` 接口：
   ```typescript
   class NewAgent extends BaseAgent<NotificationAgentNew> 
     implements NotificationAgent 
   {
     protected getSettings(): NotificationAgentNew { ... }
     shouldSend(): boolean { ... }
     async send(type, payload): Promise<boolean> { ... }
   }
   ```

2. **在 settings 中添加配置接口**，扩展 `NotificationAgentConfig`

3. **在 `NotificationAgentKey` 枚举中添加新 key**

4. **在 `NotificationSettings` 接口中添加配置字段**

5. **在 `server/index.ts` 中注册新 Agent**

6. **添加类型位掩码检查**：
   ```typescript
   if (!hasNotificationType(type, settings.types ?? 0)) {
     return true;  // 该类型通知未开启，静默跳过
   }
   ```

#### 5.5.5 位掩码通知类型过滤

**文件**: `server/lib/notifications/index.ts`

```typescript
export const hasNotificationType = (type: Notification, types: number): boolean => {
  return (types & type) === type;  // 位与运算检查
};
```

用户可精细控制每个 Agent 接收哪些通知类型：
- `types = 2 | 4 | 8` 只接收 PENDING、APPROVED、AVAILABLE
- `types = 0` 接收所有类型（默认）

#### 5.5.6 通知 Agent 插件热加载机制

**现状分析** (`server/index.ts:131-143` + `server/lib/notifications/index.ts:95-97`):
当前为**启动时静态注册**，修改 Agent 配置需重启服务：
```typescript
// ❌ 静态注册：启动后无法动态增删
notificationManager.registerAgents([
  new DiscordAgent(),    // 硬编码在 index.ts
  new EmailAgent(),
  // ...
]);
```

**热加载改造架构**:

```
┌──────────────────────────────────────────────────────┐
│                  热加载管理器                          │
│                                                      │
│  ┌────────────┐     fs.watch()    ┌───────────────┐  │
│  │ plugins/   │────────────────▶ │ 配置变更检测   │  │
│  │  discord.ts│                   └──────┬────────┘  │
│  │  webhook.ts│                          │           │
│  │  custom*.ts│                          ▼           │
│  └────────────┘              ┌──────────────────┐    │
│                              │ 模块 HMR 卸载/加载 │    │
│                              │ - clear require   │    │
│                              │ - 重新 import()  │    │
│                              └──────┬───────────┘    │
│                                     ▼                │
│                              ┌──────────────────┐    │
│                              │ Agent 注册表更新 │    │
│                              │ - 旧实例 dispose │    │
│                              │ - 新实例注册     │    │
│                              │ - 发送测试消息   │    │
│                              └──────────────────┘    │
└──────────────────────────────────────────────────────┘
```

**热加载核心实现（建议改造）**:
```typescript
// server/lib/notifications/pluginLoader.ts
import chokidar from 'chokidar';
import { NotificationAgent, NotificationAgentKey } from './agents/agent';

class NotificationPluginManager {
  private agentInstances = new Map<NotificationAgentKey, NotificationAgent & { dispose?: () => void }>();
  private watcher: chokidar.FSWatcher;

  // 启动热加载监听
  public async startHotReload() {
    const pluginDir = path.resolve(__dirname, '../../plugins/notifications');

    this.watcher = chokidar.watch(`${pluginDir}/**/*.ts`, {
      ignoreInitial: false,
      awaitWriteFinish: { stabilityThreshold: 500 },  // 等待文件写入完成
    });

    this.watcher
      .on('add', (filePath) => this.loadPlugin(filePath, 'add'))
      .on('change', (filePath) => this.loadPlugin(filePath, 'change'))
      .on('unlink', (filePath) => this.unloadPlugin(filePath));
  }

  // 动态加载插件
  private async loadPlugin(filePath: string, event: 'add' | 'change') {
    try {
      // 1. 清除 Node 模块缓存（关键）
      delete require.cache[require.resolve(filePath)];

      // 2. 动态导入（支持 ESM 和 CJS）
      const pluginModule = await import(filePath);
      const PluginClass = pluginModule.default || pluginModule.PluginClass;

      // 3. 验证接口契约
      if (!this.validateAgentInterface(PluginClass)) {
        logger.error('Plugin invalid: missing required methods', {
          label: 'Notifications',
          filePath,
          fingerprint: `plugin-validation:${path.basename(filePath)}`,  // Sentry 告警键
        });
        return;
      }

      // 4. 先销毁旧实例（如果存在）
      const agentKey = PluginClass.agentKey as NotificationAgentKey;
      if (event === 'change' && this.agentInstances.has(agentKey)) {
        const oldInstance = this.agentInstances.get(agentKey);
        if (oldInstance?.dispose) {
          await oldInstance.dispose();  // 清理资源（连接、定时器等）
        }
        notificationManager.unregisterAgent(agentKey);
      }

      // 5. 创建新实例并注册
      const settings = getSettings().notifications[agentKey];
      const newInstance = new PluginClass(settings);

      notificationManager.registerAgents([newInstance]);
      this.agentInstances.set(agentKey, newInstance);

      logger.info('Notification plugin loaded', {
        label: 'Notifications',
        agentKey,
        event,
        filePath,
      });
    } catch (e) {
      logger.error('Failed to load notification plugin', {
        label: 'Notifications',
        filePath,
        errorMessage: e.message,
        fingerprint: `plugin-load-fail:${path.basename(filePath)}:${e.code || 'unknown'}`,
      });
    }
  }

  // 插件接口校验（类型守卫）
  private validateAgentInterface(cls: unknown): cls is new (...args: any[]) => NotificationAgent {
    return typeof cls === 'function' &&
      typeof cls.prototype.shouldSend === 'function' &&
      typeof cls.prototype.send === 'function' &&
      NotificationAgentKey[cls.agentKey] !== undefined;
  }

  // 安全停止
  public async stop() {
    await this.watcher?.close();
    for (const instance of this.agentInstances.values()) {
      await instance.dispose?.();
    }
  }
}

// 插件需要导出的约定接口
export interface NotificationPlugin {
  agentKey: NotificationAgentKey;  // 静态属性，标识唯一键
  version: string;                 // 版本号，用于兼容性检查
  new (settings?: NotificationAgentConfig): NotificationAgent & {
    dispose?: () => Promise<void> | void;  // 可选的资源清理方法
  };
}
```

**插件目录结构约定**:
```
server/plugins/notifications/
├── custom-slack/
│   ├── index.ts          # 导出 CustomSlackAgent 类
│   ├── manifest.json     # name, version, author, dependencies
│   └── README.md
├── custom-wechat.ts      # 单文件插件也支持
└── _template.ts          # 插件开发模板
```

**热加载安全机制**:
1. **版本兼容性检查**: `manifest.json` 中声明 `compatibleApiVersion >= 1.0.0`
2. **沙箱隔离**: 使用 `vm.Module` 或 `isolated-vm` 隔离第三方插件（可选）
3. **回滚机制**: 新版本加载失败时，自动恢复上一个可用版本
4. **发送测试消息**: 加载成功后立即触发 `TEST_NOTIFICATION` 验证可用性

---

## 六、回执处理与状态回写

### 6.1 可用性同步 (AvailabilitySync)

**文件**: `server/lib/availabilitySync.ts:22-1180`

定时任务 (`availability-sync`) 定期执行，检查媒体是否仍然存在：

```typescript
async run() {
  for await (const media of this.loadAvailableMediaPaginated(50)) {
    if (media.mediaType === 'movie') {
      // 检查 Radarr + Plex/Jellyfin
      const existsInRadarr = await this.mediaExistsInRadarr(media, false);
      const { existsInPlex } = await this.mediaExistsInPlex(media, false);
      
      if (!existsInRadarr && !existsInPlex && media.status === AVAILABLE) {
        await this.mediaUpdater(media, false, mediaServerType);
      }
    }
    
    if (media.mediaType === 'tv') {
      // 类似逻辑，检查 Sonarr + Plex/Jellyfin
      // 额外检查每季的可用性
      await this.seasonUpdater(media, finalSeasons, false, ...);
    }
  }
}
```

### 6.2 媒体状态流转

**文件**: `server/constants/media.ts`

```typescript
export enum MediaStatus {
  UNKNOWN = 1,        // 初始状态
  PENDING = 2,        // 已请求待处理
  PROCESSING = 3,     // 已发送到 *arr，正在下载
  PARTIALLY_AVAILABLE = 4,  // 部分季可用 (剧集)
  AVAILABLE = 5,      // 完全可用
  BLOCKLISTED = 6,    // 被黑名单
  DELETED = 7,        // 已从媒体服务器删除
}

export enum MediaRequestStatus {
  PENDING = 1,        // 待审批
  APPROVED = 2,       // 已批准
  DECLINED = 3,       // 已拒绝
  FAILED = 4,         // 处理失败
  COMPLETED = 5,      // 已完成 (媒体已可用)
}
```

### 6.3 状态回写触发链

```
扫描器发现媒体已下载
    ↓
BaseScanner.processMovie() / processShow()
    ↓
更新 Media.status = AVAILABLE
    ↓
Media.lastSeasonChange 被更新 (仅剧集)
    ↓
(下次扫描或用户查看时)
    ↓
MediaRequestSubscriber 检查相关请求
    ↓
如果所有请求季都可用 → 标记请求 COMPLETED
    ↓
触发 MEDIA_AVAILABLE 通知给用户
```

### 6.4 4K 与非 4K 分离存储

Media 实体为 4K 和非 4K 维护独立的状态字段：
```typescript
// Media 实体字段
status: MediaStatus;          // 非 4K 状态
status4k: MediaStatus;        // 4K 状态
serviceId: number;            // 非 4K 服务 ID
serviceId4k: number;          // 4K 服务 ID
externalServiceId: number;    // 非 4K 外部 ID
externalServiceId4k: number;  // 4K 外部 ID
ratingKey: string;            // Plex 非 4K ratingKey
ratingKey4k: string;          // Plex 4K ratingKey
```

#### 6.4.1 4K 双轨字段数据库迁移

**文件**: `server/migration/sqlite/1610370640747-Add4kStatusFields.ts`

采用**临时表重建 + 数据迁移**的零停机迁移策略（SQLite 不支持 ADD COLUMN 带 DEFAULT 约束的某些场景）：

```typescript
public async up(queryRunner: QueryRunner): Promise<void> {
  // 1. 创建带 status4k 字段的临时表
  await queryRunner.query(
    `CREATE TABLE "temporary_media" (
      "id" integer PRIMARY KEY AUTOINCREMENT NOT NULL, 
      "mediaType" varchar NOT NULL, 
      "tmdbId" integer NOT NULL, 
      "tvdbId" integer, 
      "imdbId" varchar, 
      "status" integer NOT NULL DEFAULT (1), 
      "createdAt" datetime NOT NULL DEFAULT (datetime('now')), 
      "updatedAt" datetime NOT NULL DEFAULT (datetime('now')), 
      "lastSeasonChange" datetime NOT NULL DEFAULT (CURRENT_TIMESTAMP), 
      "status4k" integer NOT NULL DEFAULT (1)  -- 新增 4K 状态字段
    )`
  );

  // 2. 迁移历史数据（不包含 status4k，使用默认值 1 = UNKNOWN）
  await queryRunner.query(
    `INSERT INTO "temporary_media"("id", "mediaType", "tmdbId", "tvdbId", "imdbId", "status", "createdAt", "updatedAt", "lastSeasonChange") 
     SELECT "id", "mediaType", "tmdbId", "tvdbId", "imdbId", "status", "createdAt", "updatedAt", "lastSeasonChange" 
     FROM "media"`
  );

  // 3. 替换原表
  await queryRunner.query(`DROP TABLE "media"`);
  await queryRunner.query(`ALTER TABLE "temporary_media" RENAME TO "media"`);

  // 4. 重建索引
  await queryRunner.query(`CREATE INDEX "IDX_7157aad07c73f6a6ae3bbd5ef5" ON "media" ("tmdbId") `);
  await queryRunner.query(`CREATE INDEX "IDX_41a289eb1fa489c1bc6f38d9c3" ON "media" ("tvdbId") `);

  // 5. Season 表和 MediaRequest 表同步迁移
  await queryRunner.query(
    `CREATE TABLE "temporary_season" (..., "status4k" integer NOT NULL DEFAULT (1))`
  );
  await queryRunner.query(
    `CREATE TABLE "temporary_media_request" (..., "is4k" boolean NOT NULL DEFAULT (0))`
  );
}
```

**迁移关键设计决策**:

| 决策 | 原因 | 影响 |
|------|------|------|
| 临时表重建 | SQLite 不支持 `ALTER TABLE ADD COLUMN ... DEFAULT` 在某些场景 | 迁移期间只读，约 10-30 秒停机（取决于数据量） |
| `status4k DEFAULT 1` | 所有历史媒体默认为 `UNKNOWN` 状态，由后续扫描重新确认 | 首次升级后需要运行全量扫描重新检测 4K 状态 |
| `is4k DEFAULT 0` | 历史请求默认为非 4K | 不影响现有请求流 |
| 关闭外键约束 | SQLite 临时表替换时外键约束会阻塞 DROP TABLE | 迁移期间无外键检查，需确保数据一致性 |

**迁移回滚方案** (`down` 方法):
```typescript
public async down(queryRunner: QueryRunner): Promise<void> {
  // 反向操作：重建无 status4k 字段的原始表
  // 数据迁移时丢弃 status4k 列
}
```

#### 6.4.2 4K 降回 1080p 中间态字段为空场景

**场景分析**:
用户先请求 4K 版本，系统写入 `status4k=PROCESSING, serviceId4k=<值>, externalServiceId4k=<值>`，但下载失败或被删除后，用户再请求 1080p 版本。此时 `status4k` 相关字段可能残留或被清空，导致状态不一致。

**源码中的保护逻辑** (`server/subscriber/MediaRequestSubscriber.ts:838-856`):

```typescript
public async updateParentStatus(entity: MediaRequest): Promise<void> {
  const statusKey = entity.is4k ? 'status4k' : 'status';

  // 🔒 三重状态保护：只有在特定状态下才允许升级到 PROCESSING
  if (
    entity.status === MediaRequestStatus.APPROVED &&
    media[statusKey] !== MediaStatus.AVAILABLE &&           // 排除：已经可用
    media[statusKey] !== MediaStatus.PARTIALLY_AVAILABLE && // 排除：部分可用
    media[statusKey] !== MediaStatus.PROCESSING             // 排除：正在处理（可能是另一个请求的）
  ) {
    media[statusKey] = MediaStatus.PROCESSING;
    await mediaRepository.save(media);
  }

  // 🚫 拒绝时状态保护：电影拒绝只回退到 UNKNOWN
  if (
    media.mediaType === MediaType.MOVIE &&
    entity.status === MediaRequestStatus.DECLINED &&
    media[statusKey] !== MediaStatus.DELETED  // 不覆盖 DELETED（已被删除的媒体保持标记）
  ) {
    media[statusKey] = MediaStatus.UNKNOWN;
    await mediaRepository.save(media);
  }
```

**AvailabilitySync 中的中间态保护** (`server/lib/availabilitySync.ts:536-560`):

```typescript
private async mediaUpdater(media: Media, is4k: boolean, mediaServerType: MediaServerType) {
  // ... 先检测 isMediaProcessing ...

  // 🎯 核心：三元表达式保护 —— 处理中时保留原值，否则清空
  media[is4k ? 'status4k' : 'status'] = MediaStatus.DELETED;
  media[is4k ? 'serviceId4k' : 'serviceId'] = isMediaProcessing
    ? media[is4k ? 'serviceId4k' : 'serviceId']  // 处理中：保留原有服务 ID（可能是 4K 也可能是 1080p）
    : null;                                        // 未处理：清空，避免残留脏数据

  media[is4k ? 'externalServiceId4k' : 'externalServiceId'] =
    isMediaProcessing
      ? media[is4k ? 'externalServiceId4k' : 'externalServiceId']  // 保留
      : null;                                                       // 清空

  media[is4k ? 'ratingKey4k' : 'ratingKey'] = isMediaProcessing
    ? media[is4k ? 'ratingKey4k' : 'ratingKey']  // 保留 Plex ratingKey
    : null;
}
```

**Scanner 中状态降级保护** (`server/lib/scanners/baseScanner.ts:119-134`):

```typescript
// 多条件状态判定矩阵，避免异常覆盖
existing[statusField] =
  !processing && hasFile
    ? MediaStatus.AVAILABLE                              // ✅ 下载完成且有文件 → AVAILABLE
    : !processing && !hasFile && previousStatus === MediaStatus.PROCESSING
      ? MediaStatus.UNKNOWN                              // ⚠️ 4K 降回 1080p 场景：处理中但找不到文件 → 重置 UNKNOWN
      : processing
        ? previousStatus === MediaStatus.DELETED
          ? MediaStatus.DELETED                          // 🔒 已删除的不重新激活
          : MediaStatus.PROCESSING                       // ✅ 正常标记处理中
        : previousStatus;                                // 🛡️ 其他情况保持不变（关键兜底）
```

**典型中间态场景与行为矩阵**:

| 当前 `status4k` | 当前 `serviceId4k` | `is4k` 请求 | 操作 | 结果 `status4k` | 结果 `serviceId4k` |
|----------------|-------------------|------------|------|----------------|-------------------|
| PROCESSING | 非空（有值） | false（请求 1080p） | 审批通过 | PROCESSING（**不变**） | 非空（**保留**，避免 4K 正在下载时被误清） |
| PROCESSING | 非空 | false | AvailabilitySync 检测到媒体缺失 | DELETED | 非空（**保留**，isMediaProcessing=true） |
| UNKNOWN | null | true（请求 4K） | 审批通过 | PROCESSING | 填入新值 |
| DELETED | null | true | 审批通过 | PROCESSING（从 DELETED 恢复） | 填入新值 |
| PARTIALLY_AVAILABLE | 非空 | false | 审批通过 | PARTIALLY_AVAILABLE（**不变**，避免覆盖部分可用） | 非空（保留） |

**状态机转移图**:
```
UNKNOWN ──请求批准──▶ PROCESSING ──*arr 下载完成──▶ AVAILABLE
   ▲                         │                          │
   │                         │ 找不到文件              │ AvailabilitySync
   │                         ▼                          ▼
   └────拒绝/撤销──────── UNKNOWN                    DELETED
                               ▲                         │
                               │ 有新请求/下载中保留元数据 │
                               └─────────────────────────┘
```

---

### 6.5 异步非阻塞背压控制

#### 6.5.1 扫描器背压机制

**文件**: `server/lib/scanners/baseScanner.ts:687-725` + `:646-681`

三层背压控制防止系统过载：

**第一层 - 会话隔离 (Session ID)**:
```typescript
protected startRun(): string {
  const sessionId = randomUUID();  // 每次运行生成唯一会话
  this.sessionId = sessionId;
  this.running = true;
  return sessionId;
}

protected async loop(..., { sessionId } = {}): Promise<void> {
  if (this.sessionId !== sessionId) {
    throw new Error('New session was started. Old session aborted.');  // 新会话启动时终止旧会话
  }
  // ...
}
```

**第二层 - 分批节流**:
```typescript
// 默认配置
const BUNDLE_SIZE = 20;       // 每批 20 条
const UPDATE_RATE = 4 * 1000;  // 批间间隔 4 秒

await new Promise<void>((resolve) =>
  setTimeout(() => {
    this.loop(processFn, {
      start: start + this.bundleSize,
      sessionId,
    }).then(() => resolve());
  }, this.updateRate)  // 强制等待，防止 CPU/IO 过载
);
```

**第三层 - 可取消标志**:
```typescript
public cancel(): void {
  this.running = false;  // 由外部设置取消标志
}

if (!this.running) {
  throw new Error('Sync was aborted.');  // 每批次开始前检查
}
```

#### 6.5.2 AvailabilitySync 背压控制

**文件**: `server/lib/availabilitySync.ts:38-466`

```typescript
async run() {
  this.running = true;
  try {
    for await (const media of this.loadAvailableMediaPaginated(50)) {
      if (!this.running) {
        throw new Error('Job aborted');  // 每页检查取消标志
      }
      // 处理单条媒体...
    }
  } finally {
    this.running = false;
  }
}

// 异步生成器分页拉取，避免一次性加载所有数据
private async *loadAvailableMediaPaginated(pageSize: number) {
  let offset = 0;
  let mediaPage: Media[];
  do {
    yield* (mediaPage = await mediaRepository.find({
      where: whereOptions,
      skip: offset,
      take: pageSize,  // 每页 50 条
    }));
    offset += pageSize;
  } while (mediaPage.length > 0);
}
```

**背压效果**:
- 内存占用稳定（O(pageSize) 而非 O(total)）
- 数据库查询压力可控
- 可随时中断，重启时无需重头开始

#### 6.5.3 背压熔断水位线设计

**源码中的监控指标** (`server/lib/scanners/baseScanner.ts:15-19, 59-66, 646-681`):

```typescript
// 状态输出接口
export type StatusBase = {
  running: boolean;   // 熔断开关：false 时直接触发熔断
  progress: number;   // 已处理数量
  total: number;      // 总数量
};

// 扫描器状态
protected progress = 0;
protected items: T[] = [];
protected totalSize?: number = 0;
protected sessionId: string;    // 会话 ID：用于检测会话过期
protected running = false;      // 全局熔断标志
```

**四级熔断水位线（建议补充实现）**:

```
│               │  Level 4: FATAL          │  > 50% 超时 │  终止所有批次 + 报警 + 全局降级（跳过本次扫描）
│               │──────────────────────────│────────────│
│               │  Level 3: CRITICAL       │  > 30% 超时 │  updateRate × 3（降速到 12 秒）
│     慢批次   │──────────────────────────│────────────│
│     超时率   │  Level 2: WARNING        │  > 10% 超时 │  updateRate × 1.5（降速到 6 秒）
│               │──────────────────────────│────────────│
│               │  Level 1: INFO           │  < 10% 超时 │  正常速率 4 秒/批
│               │──────────────────────────│────────────│
│               │  Level 0: IDLE           │  running=false │  完全停止，等待 cancel() 释放
```

**熔断机制实现（建议改造 baseScanner.ts）**:
```typescript
abstract class BaseScanner<T> {
  // 水位线配置
  private readonly WATERMARK = {
    WARN_SLOW_BATCH_RATIO: 0.10,     // 10% 批次超过 2 倍平均耗时 → 警告
    CRITICAL_SLOW_BATCH_RATIO: 0.30, // 30% 批次超 2 倍 → 严重
    FATAL_SLOW_BATCH_RATIO: 0.50,    // 50% 批次超 3 倍 → 致命
    BATCH_TIMEOUT_BASELINE: 8000,    // 批次 8 秒基线超时
    DB_ERROR_THRESHOLD: 5,           // 单批次 DB 错误超过 5 次 → 熔断
    API_429_THRESHOLD: 3,            // 单批次 429 超过 3 次 → 降速
  } as const;

  // 运行时统计
  private batchStats = {
    totalBatches: 0,
    slowBatches: 0,
    dbErrors: 0,
    api429Errors: 0,
    avgBatchDurationMs: 0,
    sessionStartAt: 0,
  };

  protected async loop(processFn, { sessionId }) {
    // 会话级熔断：如果 sessionId 过期，直接终止
    if (this.sessionId !== sessionId) {
      throw new Error('New session was started. Old session aborted.');
    }

    // 全局熔断标志检查
    if (!this.running) {
      throw new Error('Sync was aborted.');
    }

    // 处理前计算当前水位
    const currentWatermark = this.calculateWatermark();

    // 动态调整 updateRate（自适应降速）
    const adaptiveDelay = this.updateRate * this.getBackoffMultiplier(currentWatermark);

    // Level 4 致命熔断
    if (currentWatermark === 'FATAL') {
      this.running = false;
      logger.error('Scan FATAL watermark exceeded, triggering global circuit breaker', {
        label: this.scannerName,
        sessionId,
        fingerprint: ['circuit-breaker-fatal', this.scannerName, sessionId],
        stats: { ...this.batchStats },
      });
      throw new Error('Circuit breaker: FATAL watermark exceeded');
    }

    // Level 2/3 降速（通过增大 setTimeout 延时实现）
    await new Promise<void>((resolve) =>
      setTimeout(() => this.loop(...).then(resolve), adaptiveDelay)
    );
  }

  // 计算当前水位
  private calculateWatermark(): 'IDLE' | 'NORMAL' | 'WARN' | 'CRITICAL' | 'FATAL' {
    if (!this.running) return 'IDLE';
    const { totalBatches, slowBatches, dbErrors, api429Errors } = this.batchStats;
    if (totalBatches === 0) return 'NORMAL';

    const slowRatio = slowBatches / totalBatches;

    // 任一硬阈值命中直接升级
    if (dbErrors >= this.WATERMARK.DB_ERROR_THRESHOLD ||
        api429Errors >= this.WATERMARK.API_429_THRESHOLD * 5) {
      return 'FATAL';
    }
    if (slowRatio >= this.WATERMARK.FATAL_SLOW_BATCH_RATIO) return 'FATAL';
    if (slowRatio >= this.WATERMARK.CRITICAL_SLOW_BATCH_RATIO) return 'CRITICAL';
    if (slowRatio >= this.WATERMARK.WARN_SLOW_BATCH_RATIO) return 'WARN';
    return 'NORMAL';
  }

  // 退避乘数
  private getBackoffMultiplier(level: string): number {
    switch (level) {
      case 'CRITICAL': return 3.0;
      case 'WARN': return 1.5;
      default: return 1.0;
    }
  }
}
```

**熔断状态外部暴露（健康检查 API）**:
```typescript
// GET /api/v1/settings/status/jobs
// Response:
{
  "plex-full-scan": {
    "running": true,
    "progress": 1540,
    "total": 8500,
    "watermark": "WARN",
    "backoffMultiplier": 1.5,
    "currentAdaptiveDelayMs": 6000,
    "stats": {
      "slowBatchRatio": 0.12,
      "avgBatchDurationMs": 2300,
      "dbErrors": 0,
      "api429Errors": 1
    },
    "estimatedRemainingTimeSec": 14800
  }
}
```

---

### 6.6 AvailabilitySync 漂移补偿机制

**文件**: `server/lib/availabilitySync.ts:498-617`

#### 6.6.1 漂移检测

漂移场景：媒体在 Plex/Jellyfin 中被删除，但在 Radarr/Sonarr 中仍然存在（或反之）。

```typescript
// 双重校验：必须在 *arr 和媒体服务器中都不存在才标记为删除
const existsInRadarr = await this.mediaExistsInRadarr(media, false);
const { existsInPlex } = await this.mediaExistsInPlex(media, false);

// 逻辑与：只有两边都不存在才认为真正被删除
if (!existsInRadarr && !existsInPlex && media.status === MediaStatus.AVAILABLE) {
  await this.mediaUpdater(media, false, mediaServerType);
}
```

#### 6.6.2 处理中状态保护

当媒体正在下载时，即使暂时在 Plex 中找不到也不能标记为删除：

```typescript
private async mediaUpdater(media, is4k, mediaServerType) {
  // 检查是否有处理中的请求
  const request = await requestRepository
    .createQueryBuilder('request')
    .where('(media.id = :id)', { id: media.id })
    .andWhere('(request.is4k = :is4k AND request.status = :requestStatus)', {
      requestStatus: MediaRequestStatus.APPROVED,  // 处理中
      is4k: is4k,
    })
    .getOne();

  const isMediaProcessing = !!request;

  // 漂移补偿：处理中时保留外部服务元数据
  media[is4k ? 'serviceId4k' : 'serviceId'] = isMediaProcessing
    ? media[is4k ? 'serviceId4k' : 'serviceId']  // 保留原值
    : null;                                        // 否则清空
  
  media[is4k ? 'externalServiceId4k' : 'externalServiceId'] = isMediaProcessing
    ? media[is4k ? 'externalServiceId4k' : 'externalServiceId']
    : null;
  // ... 其他字段同理
  
  media[is4k ? 'status4k' : 'status'] = MediaStatus.DELETED;  // 仅状态标记为删除
}
```

#### 6.6.3 季级粒度补偿

剧集按季检查，避免单季删除影响整部剧：

```typescript
const filteredSeasonsMap: Map<number, boolean> = new Map();
media.seasons
  .filter(season => 
    season.status === AVAILABLE || season.status === PARTIALLY_AVAILABLE
  )
  .forEach(season => filteredSeasonsMap.set(season.seasonNumber, false));

// 交叉比对 Plex 和 Sonarr
const finalSeasons = new Map([
  ...filteredSeasonsMap,
  ...plexSeasonsMap,
  ...sonarrSeasonsMap,
]);

// 只有两边都不存在的季才标记为删除
if ([...finalSeasons.values()].includes(false)) {
  await this.seasonUpdater(media, finalSeasons, false, mediaServerType);
}
```

#### 6.6.4 TMDB 兜底校验

当媒体服务器数据可能不准确时，从 TMDB 获取真实的季信息：

```typescript
try {
  if (media.tmdbId) {
    tvShow = await this.tmdb.getTvShow({ tvId: Number(media.tmdbId) });
  }
} catch (e) {
  // TMDB 失败时跳过该季的检查，避免误删除
}

// 用 TMDB 数据补全可能遗漏的季
if (tvShow) {
  media.seasons.forEach(season => {
    if (season.seasonNumber === 0) return;  // 跳过特别篇
    if (!finalSeasons.has(season.seasonNumber) &&
        tvShow.seasons.find(s => s.season_number === season.seasonNumber)?.episode_count) {
      finalSeasons.set(season.seasonNumber, false);  // 标记需要检查
    }
  });
}
```

#### 6.6.5 漂移超 24h 兜底降级策略

**漂移问题背景**:
AvailabilitySync 定期（`availability-sync` 任务，默认每小时一次）检查媒体是否仍然存在。但以下场景可能导致**状态漂移**超过 24 小时未被纠正：
1. Radarr/Sonarr 实例离线超过 24 小时，恢复后数据库状态丢失
2. Plex/Jellyfin 媒体库迁移，文件被重新扫描但元数据变化
3. 数据库备份恢复后，Media 表与实际磁盘状态不一致
4. 扫描任务挂起（进程阻塞、Session ID 过期未清理）

**源码中的时间戳字段** (`server/lib/scanners/baseScanner.ts:644, availabilitySync.ts:644`):

```typescript
// Season 状态变更时更新 lastSeasonChange
media.lastSeasonChange = new Date();
await mediaRepository.save(media);

// Media 实体：updatedAt、createdAt、lastSeasonChange、mediaAddedAt
```

**超 24h 兜底降级策略（建议补充实现）**:

```typescript
// server/lib/availabilitySync.ts
class AvailabilitySync {
  private readonly DRIFT_THRESHOLD_MS = 24 * 60 * 60 * 1000;  // 24 小时
  private readonly DRIFT_FALLBACK_ENABLED = true;

  async run() {
    this.running = true;
    try {
      // 正常的可用性同步逻辑...

      // 兜底：扫描 PROCESSING/PENDING 状态超 24h 的媒体
      if (this.DRIFT_FALLBACK_ENABLED) {
        await this.runDriftFallback();
      }
    } finally {
      this.running = false;
    }
  }

  private async runDriftFallback() {
    const driftThreshold = new Date(Date.now() - this.DRIFT_THRESHOLD_MS);

    // 场景 1：电影 PROCESSING 超 24h → 降级为 UNKNOWN + 告警
    const stuckProcessingMovies = await mediaRepository
      .createQueryBuilder('media')
      .where('media.mediaType = :movieType', { movieType: MediaType.MOVIE })
      .andWhere('(media.status = :processing OR media.status4k = :processing)', {
        processing: MediaStatus.PROCESSING,
      })
      .andWhere('media.updatedAt < :threshold', { threshold: driftThreshold })
      .getMany();

    for (const media of stuckProcessingMovies) {
      await this.handleDriftedMedia(media, 'PROCESSING_TIMEOUT');
    }

    // 场景 2：剧集 PARTIALLY_AVAILABLE 超 24h → 检查每一季
    const stuckPartialShows = await mediaRepository
      .createQueryBuilder('media')
      .leftJoinAndSelect('media.seasons', 'season')
      .where('media.mediaType = :tvType', { tvType: MediaType.TV })
      .andWhere('(media.status = :partial OR media.status4k = :partial)', {
        partial: MediaStatus.PARTIALLY_AVAILABLE,
      })
      .andWhere('media.lastSeasonChange < :threshold', { threshold: driftThreshold })
      .getMany();

    for (const media of stuckPartialShows) {
      await this.handleDriftedShow(media);
    }

    // 场景 3：DELETED 超 24h 且有新请求 → 清理脏字段
    const deletedWithPending = await mediaRepository
      .createQueryBuilder('media')
      .innerJoinAndSelect('media.requests', 'request')
      .where('(media.status = :deleted OR media.status4k = :deleted)', {
        deleted: MediaStatus.DELETED,
      })
      .andWhere('request.status = :pending', { pending: MediaRequestStatus.PENDING })
      .andWhere('media.updatedAt < :threshold', { threshold: driftThreshold })
      .getMany();

    for (const media of deletedWithPending) {
      await this.cleanupDriftedDeletedMedia(media);
    }
  }

  private async handleDriftedMedia(media: Media, reason: string) {
    const mediaRepository = getRepository(Media);

    // 记录漂移告警
    logger.warn('Media status drift detected, applying fallback', {
      label: 'AvailabilitySync',
      tmdbId: media.tmdbId,
      mediaType: media.mediaType,
      status: media.status,
      status4k: media.status4k,
      updatedAgoMs: Date.now() - (media.updatedAt?.getTime() || 0),
      driftReason: reason,
      fingerprint: [
        'availability-drift',
        reason,
        `tmdb:${media.tmdbId}`,
        `type:${media.mediaType}`,
      ],
    });

    // 降级策略：
    // PROCESSING → UNKNOWN（等待下次扫描重新确认）
    // 但保留外部服务元数据，避免用户手动配置丢失
    if (media.status === MediaStatus.PROCESSING) {
      media.status = MediaStatus.UNKNOWN;
    }
    if (media.status4k === MediaStatus.PROCESSING) {
      media.status4k = MediaStatus.UNKNOWN;
    }

    await mediaRepository.save(media);
  }

  private async handleDriftedShow(media: Media) {
    const seasonRepository = getRepository(Season);
    const driftThreshold = new Date(Date.now() - this.DRIFT_THRESHOLD_MS);

    for (const season of media.seasons) {
      // 只有 24h 内未变更的季才触发兜底
      if (season.updatedAt && season.updatedAt > driftThreshold) continue;

      // 检查该季是否有处理中的请求
      const activeRequest = await getRepository(SeasonRequest)
        .createQueryBuilder('sr')
        .innerJoin('sr.request', 'request')
        .where('sr.seasonId = :seasonId', { seasonId: season.id })
        .andWhere('request.status IN (:...statuses)', {
          statuses: [MediaRequestStatus.APPROVED, MediaRequestStatus.PENDING],
        })
        .getExists();

      if (!activeRequest) {
        // 无活跃请求的漂移季：从 PARTIALLY_AVAILABLE 降级
        if (season.status === MediaStatus.PARTIALLY_AVAILABLE) {
          season.status = MediaStatus.UNKNOWN;
          await seasonRepository.save(season);
        }
        if (season.status4k === MediaStatus.PARTIALLY_AVAILABLE) {
          season.status4k = MediaStatus.UNKNOWN;
          await seasonRepository.save(season);
        }
      }
    }
  }

  private async cleanupDriftedDeletedMedia(media: Media) {
    const mediaRepository = getRepository(Media);

    // DELETED 状态但有待处理请求 → 可能是误标删除
    // 策略：降级为 UNKNOWN，清空外部 ID，触发下次扫描重新拉取
    if (media.status === MediaStatus.DELETED) {
      media.status = MediaStatus.UNKNOWN;
      media.serviceId = null;
      media.externalServiceId = null;
    }
    if (media.status4k === MediaStatus.DELETED) {
      media.status4k = MediaStatus.UNKNOWN;
      media.serviceId4k = null;
      media.externalServiceId4k = null;
    }

    logger.warn('Cleaned up drifted DELETED media with pending requests', {
      label: 'AvailabilitySync',
      tmdbId: media.tmdbId,
      fingerprint: ['availability-drift-deleted-with-pending', `tmdb:${media.tmdbId}`],
    });

    await mediaRepository.save(media);
  }
}
```

**漂移状态降级决策表**:

| 当前状态 | 持续时间 | 活跃请求 | 降级后状态 | 清理外部 ID | 告警级别 |
|---------|---------|---------|-----------|------------|---------|
| PROCESSING | > 24h | ❌ 无 | UNKNOWN | ❌ 保留 | warning |
| PROCESSING | > 24h | ✅ 有 | PROCESSING（不变） | ❌ 保留 | warning（仅告警） |
| PARTIALLY_AVAILABLE | > 24h | ❌ 无 | AVAILABLE / UNKNOWN（根据 *arr 确认） | ❌ 保留 | warning |
| PARTIALLY_AVAILABLE | > 24h | ✅ 有 | 不变 | ❌ 保留 | info |
| DELETED | > 24h | ✅ 有 | UNKNOWN | ✅ 清空 | warning |
| DELETED | > 24h | ❌ 无 | DELETED（不变） | ❌ 保留 | 不告警 |
| UNKNOWN | > 72h | ❌ 无 | UNKNOWN（不变） | ✅ 清理空字段 | info |

**扫描器任务挂起兜底 (Task Heartbeat)**:
```typescript
// server/job/schedule.ts - 任务心跳检查
const HEARTBEAT_TIMEOUT = 30 * 60 * 1000;  // 30 分钟无心跳

setInterval(() => {
  for (const [jobName, scanner] of runningScanners) {
    if (scanner.running && 
        (Date.now() - scanner.lastHeartbeatAt) > HEARTBEAT_TIMEOUT) {
      // 强制取消超时任务
      scanner.cancel();
      logger.error('Scanner stuck, force canceled', {
        label: 'Scheduler',
        jobName,
        stuckDurationMs: Date.now() - scanner.lastHeartbeatAt,
        fingerprint: ['scanner-heartbeat-timeout', jobName],
      });
      // 触发重新调度
      rescheduleJob(jobName);
    }
  }
}, 60000);  // 每分钟检查一次
```

---

## 七、错误回滚机制

### 7.0 5 种回滚场景优先级

回滚操作按**数据一致性影响程度**和**操作不可逆程度**确定优先级：

| 优先级 | 回滚场景 | 触发条件 | 影响范围 | 不可逆程度 |
|-------|---------|---------|---------|-----------|
| 🔴 P0 | **请求删除回滚** | 请求被用户/管理员删除 | 父级 Media 状态 | ⭐⭐⭐⭐⭐ |
| 🟠 P1 | **请求拒绝回滚** | 请求被管理员拒绝 | 父级 Media + Season 状态 | ⭐⭐⭐⭐ |
| 🟡 P2 | **发送到 *arr 失败回滚** | Radarr/Sonarr API 调用失败 | MediaRequest 状态 | ⭐⭐⭐ |
| 🟢 P3 | **连接/配置错误回滚** | 网络错误或配置无效 | MediaRequest 状态 | ⭐⭐ |
| 🔵 P4 | **重试机制** | 用户手动触发重试 | 重置状态重新发送 | ⭐ |

**优先级设计原则**:
1. **P0 > P1**: 删除比拒绝更严重，删除意味着用户请求完全消失
2. **P1 > P2**: 人为操作（拒绝）比系统错误优先级高
3. **P2 > P3**: API 业务失败比连接失败更需要明确标记
4. **P3 > P4**: 错误标记比重试更紧急

**高优先级回滚会覆盖低优先级状态**:
```typescript
// P0 删除回滚可以覆盖任何状态
if (needsStatusUpdate) {
  cleanMedia.status = hadCompleted ? DELETED : UNKNOWN;  // 无条件覆盖
}

// P1 拒绝回滚只在特定条件下执行
if (media.mediaType === MOVIE && entity.status === DECLINED) {
  // 只重置为 UNKNOWN，不设置 DELETED
  media.status = UNKNOWN;
}

// P2/P3 失败回滚有幂等检查
if (entity.status !== FAILED) {  // 防止重复标记
  entity.status = FAILED;
}
```

### 7.0.1 回滚优先级与 Sentry 告警键关联

**现状分析**:
当前项目未集成 Sentry（全项目 grep 无 `@sentry/node` 引用），但 `logger.error/warn` 的结构化日志参数天然适合映射为 Sentry `fingerprint` 和 `tags`。

**Sentry 告警键 (Fingerprint) 设计映射表**:

| 优先级 | 回滚场景 | Sentry Fingerprint 模板 | Sentry Level | 触发位置 |
|-------|---------|------------------------|--------------|---------|
| 🔴 P0 | **请求删除回滚** | `['rollback-p0-delete', 'media:'+mediaId, 'tmdb:'+tmdbId, 'request:'+requestId]` | error | `handleRemoveParentUpdate` (Subscriber.ts:946) |
| 🟠 P1 | **请求拒绝回滚** | `['rollback-p1-decline', 'media:'+mediaId, 'operator:'+modifiedById]` | warning | `updateParentStatus` (Subscriber.ts:849) |
| 🟡 P2 | **API 业务失败回滚** | `['rollback-p2-api-failed', 'radarr:'+serverName, 'error:'+errorCode]` | error | `sendToRadarr` .catch() (Subscriber.ts:398) |
| 🟢 P3 | **连接/配置错误回滚** | `['rollback-p3-connect-fail', hostname, 'error:'+e.code]` | warning | `sendToRadarr` 外层 catch (Subscriber.ts:789) |
| 🔵 P4 | **重试机制** | `['rollback-p4-retry', 'request:'+requestId, 'retry-count:'+count]` | info | `POST /request/:requestId/retry` (request.ts:633) |

**Sentry 集成示例（建议改造）**:
```typescript
// 在 logger 封装层增加 Sentry 桥接
const sentryError = (message: string, context: {
  label: string;
  fingerprint?: string[];
  tags?: Record<string, string>;
  user?: { id: number; email: string };
  extra?: Record<string, unknown>;
}) => {
  logger.error(message, context);

  // 自动根据回滚优先级映射 Sentry 级别
  const level = context.fingerprint?.[0]?.includes('p0') ? 'error'
              : context.fingerprint?.[0]?.includes('p1') ? 'warning'
              : context.fingerprint?.[0]?.includes('p2') ? 'error'
              : context.fingerprint?.[0]?.includes('p3') ? 'warning'
              : 'info';

  Sentry.withScope((scope) => {
    if (context.fingerprint) scope.setFingerprint(context.fingerprint);
    if (context.tags) Object.entries(context.tags).forEach(([k, v]) => scope.setTag(k, v));
    if (context.user) scope.setUser(context.user);
    if (context.extra) scope.setExtras(context.extra);
    scope.setLevel(level as any);
    scope.setTag('component', context.label);  // 用 label 作为组件分类
    Sentry.captureException(new Error(message));
  });
};

// 实际调用示例（P2 失败回滚）
sentryError('Failed to send to Radarr', {
  label: 'Media Request',
  fingerprint: ['rollback-p2-api-failed', `radarr:${radarrSettings.name}`, `error:${e.code}`],
  tags: {
    media_type: entity.type === MediaType.MOVIE ? 'movie' : 'tv',
    is_4k: String(entity.is4k),
    server_id: String(radarrSettings.id),
  },
  user: { id: entity.modifiedBy?.id, email: entity.modifiedBy?.email },
  extra: { requestId: entity.id, mediaId: entity.media.id },
});
```

**Sentry 告警分组策略（Rules）**:
```yaml
# Sentry Issue Alert Rule
conditions:
  - type: event.frequency
    value: 10  # 10 分钟内超过 10 次
    window: 600

# P0/P2 级 issue 直接 PagerDuty 呼叫 oncall
fingerprint_pattern:
  contains:
    - "rollback-p0"
    - "rollback-p2"
```

---

### 7.1 发送到 *arr 失败回滚

**文件**: `server/subscriber/MediaRequestSubscriber.ts:398-432` (Radarr 示例)

```typescript
radarr
  .addMovie(radarrMovieOptions)
  .catch(async () => {
    try {
      const requestRepository = getRepository(MediaRequest);
      
      // 幂等检查：避免重复标记
      if (entity.status !== MediaRequestStatus.FAILED) {
        entity.status = MediaRequestStatus.FAILED;
        await requestRepository.save(entity);
      }
    } catch (saveError) {
      // 二次错误只记录日志
      logger.error('Failed to mark request as FAILED', { ... });
    }

    // 发送失败通知
    MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
  });
```

### 7.2 连接/配置错误回滚

```typescript
} catch (e) {
  // 连接错误或配置错误
  entity.status = MediaRequestStatus.FAILED;
  await requestRepository.save(entity);
  
  MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
}
```

### 7.3 重试机制

**文件**: `server/routes/request.ts:633-661`

用户可通过 API 重试失败的请求：
```typescript
requestRoutes.post('/:requestId/retry', 
  isAuthenticated(Permission.MANAGE_REQUESTS),
  async (req, res, next) => {
    const request = await requestRepository.findOneOrFail(...);
    
    // 将状态重置为 APPROVED，触发 afterUpdate 重新发送
    request.status = MediaRequestStatus.APPROVED;
    request.modifiedBy = req.user;
    await requestRepository.save(request);
    
    return res.status(200).json(request);
  }
);
```

### 7.4 请求删除时的状态回滚

**文件**: `server/subscriber/MediaRequestSubscriber.ts:946-1004`

```typescript
public async handleRemoveParentUpdate(
  manager: EntityManager,
  entity: MediaRequest
): Promise<void> {
  // 检查是否还有其他活跃请求
  const hasActive = fullMedia.requests.some(
    (request) => !request.is4k && 
      request.status !== COMPLETED && 
      request.status !== DECLINED
  );

  // 如果没有活跃请求，重置媒体状态
  if (!hasActive && media.status !== AVAILABLE) {
    // 如果有过完成请求，标记为 DELETED，否则 UNKNOWN
    cleanMedia.status = hadCompleted ? MediaStatus.DELETED : MediaStatus.UNKNOWN;
    await manager.save(cleanMedia);
  }
}
```

### 7.5 请求拒绝时的回滚

**文件**: `server/subscriber/MediaRequestSubscriber.ts:864-932`

```typescript
// 电影：如果唯一请求被拒绝，重置媒体状态
if (media.mediaType === MOVIE && entity.status === DECLINED) {
  media.status = MediaStatus.UNKNOWN;
  await mediaRepository.save(media);
}

// 剧集：检查是否还有其他待处理请求
if (media.mediaType === TV && entity.status === DECLINED) {
  const pendingCount = await requestRepository.count({ ... });
  if (pendingCount === 0) {
    freshMedia.status = MediaStatus.UNKNOWN;
    await mediaRepository.save(freshMedia);
  }
  
  // 重置相关季的状态
  for (const seasonRequest of entity.seasons) {
    seasonRequest.status = DECLINED;
    await seasonRequestRepository.save(seasonRequest);
    
    // 如果该季没有其他活跃请求，重置状态
    if (otherActiveRequests === 0) {
      season.status = MediaStatus.UNKNOWN;
      await seasonRepository.save(season);
    }
  }
}
```

---

### 7.6 TypeORM EventSubscriber 事件回放机制

#### 7.6.1 事件订阅架构

**文件**: `server/subscriber/MediaRequestSubscriber.ts:1-1050` + `server/subscriber/MediaSubscriber.ts`

TypeORM 的 `EntitySubscriberInterface` 提供实体生命周期钩子：

```typescript
@EventSubscriber()
export class MediaRequestSubscriber 
  implements EntitySubscriberInterface<MediaRequest> 
{
  // 监听的实体类型
  listenTo() {
    return MediaRequest;
  }

  // 实体更新后触发
  public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
    if (!event.entity) return;
    
    // 注意：event.entity 只包含变更字段
    // event.databaseEntity 包含完整的数据库状态
    
    try {
      await this.sendToRadarr(event.entity as MediaRequest);
      await this.sendToSonarr(event.entity as MediaRequest);
    } catch (e) {
      logger.error('Error while sending to *arr', { ... });
    }
    
    try {
      await this.updateParentStatus(event.entity as MediaRequest);
    } catch (e) {
      logger.error('Error while updating parent status', { ... });
    }
  }

  // 实体删除前触发
  public async beforeRemove(event: RemoveEvent<MediaRequest>): Promise<void> {
    if (!event.entity) return;
    await this.handleRemoveParentUpdate(event.manager, event.entity);
  }
}
```

#### 7.6.2 事件回放与事务边界

**事件触发链**:
```
HTTP 请求 → Controller → repository.save() 
    ↓
TypeORM 事务 → UPDATE SQL → 提交事务
    ↓
@AfterUpdate 钩子触发 → MediaRequestSubscriber.afterUpdate()
    ↓
sendToRadarr() / sendToSonarr() → 异步调用外部 API
    ↓
外部 API 成功 → update Media.externalServiceId → repository.save()
    ↓
触发另一次 @AfterUpdate → 可能导致循环调用
```

**防循环调用设计**:
```typescript
// sendToRadarr 中先检查状态，避免重复处理
public async sendToRadarr(entity: MediaRequest): Promise<void> {
  if (
    entity.status === MediaRequestStatus.APPROVED &&  // 只有 APPROVED 状态才处理
    entity.type === MediaType.MOVIE
  ) {
    // ... 执行发送
  }
}

// 外部 API 成功后更新 Media，不会触发 MediaRequest 的 afterUpdate
media.externalServiceId = radarrMovie.id;
await mediaRepository.save(media);  // 只更新 Media，不更新 MediaRequest
```

**事务边界问题**:
```typescript
// ❌ 问题：afterUpdate 在事务提交后触发，但如果后续操作失败
public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
  await this.sendToRadarr(event.entity);  // 这个调用可能失败
  // 但数据库事务已经提交，无法回滚
}

// ✅ 解决方案：catch + 状态标记 + 通知
.catch(async () => {
  entity.status = MediaRequestStatus.FAILED;
  await requestRepository.save(entity);  // 二次写入标记失败
  MediaRequest.sendNotification(..., Notification.MEDIA_FAILED);
});
```

#### 7.6.3 事件实体状态对比

`UpdateEvent` 提供两个关键实体对象：

| 对象 | 包含内容 | 用途 |
|------|---------|------|
| `event.entity` | 仅包含被修改的字段 | 获取变更后的值 |
| `event.databaseEntity` | 数据库中的完整实体 | 获取变更前的状态 |

```typescript
// 状态变更检测示例
public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
  const { entity, databaseEntity } = event;
  
  if (entity && databaseEntity) {
    // 检测状态从 PENDING 变为 APPROVED
    if (databaseEntity.status === MediaRequestStatus.PENDING &&
        entity.status === MediaRequestStatus.APPROVED) {
      // 触发审批通知
      MediaRequest.sendNotification(..., Notification.MEDIA_APPROVED);
    }
    
    // 检测状态从 APPROVED 变为 COMPLETED
    if (databaseEntity.status === MediaRequestStatus.APPROVED &&
        entity.status === MediaRequestStatus.COMPLETED) {
      // 触发可用通知
      MediaRequest.sendNotification(..., Notification.MEDIA_AVAILABLE);
    }
  }
}
```

#### 7.6.4 删除事件的特殊处理

删除操作在 `beforeRemove` 中处理，因为 `afterRemove` 时实体已不存在：

```typescript
// server/subscriber/MediaRequestSubscriber.ts:946-1004
public async handleRemoveParentUpdate(
  manager: EntityManager,  // 使用事务内的 manager
  entity: MediaRequest
): Promise<void> {
  // 必须使用传入的 manager，而不是全局 getRepository()
  // 因为删除操作在事务中，需要保证原子性
  
  const fullMedia = await manager.findOneOrFail(Media, {
    where: { id: entity.media.id },
    relations: { requests: true },
  });
  
  // 计算新状态...
  
  // 重新 fetch 不带 relations 的实体，避免级联删除问题
  const cleanMedia = await manager.findOneOrFail(Media, {
    where: { id: entity.media.id },
  });
  
  cleanMedia.status = newStatus;
  await manager.save(cleanMedia);  // 同一事务内保存
}
```

#### 7.6.5 事件回放幂等键设计

**问题背景**:
TypeORM 的 `AfterInsert` / `AfterUpdate` 事件可能因重试机制、Promise 不等待、或数据库集群主从延迟导致**重复触发**。如果幂等保护不足，同一事件可能被处理多次，造成重复通知、重复发送到 *arr、重复状态更新等问题。

**源码中的隐式幂等保护**:

| 幂等场景 | 现有保护键 | 保护位置 |
|---------|-----------|---------|
| 请求审批重复发送到 Radarr | `entity.status === APPROVED` + `media.status !== PROCESSING` | `sendToRadarr():185` + `updateParentStatus():838` |
| 重复通知 | `media[statusKey] !== AVAILABLE && !== PARTIALLY_AVAILABLE` | `updateParentStatus():841` |
| 重复创建 Media 记录 | AsyncLock(tmdbId) + DB UNIQUE 约束 | `processMovie():113` |
| 重复保存失败状态 | `entity.status !== FAILED` | `sendToRadarr().catch():398` |

**显式幂等键设计（建议补充）**:

```typescript
// server/subscriber/MediaRequestSubscriber.ts
class MediaRequestSubscriber {
  // Redis 或内存去重表
  private idempotencyKeys = new Map<string, number>();  // key -> 过期时间戳
  private readonly IDEMPOTENCY_TTL = 60000;  // 60 秒内事件去重

  // 计算幂等键（基于事件唯一标识）
  private getIdempotencyKey(
    eventType: 'afterInsert' | 'afterUpdate' | 'beforeRemove',
    entity: MediaRequest,
    databaseEntity?: MediaRequest
  ): string {
    // 幂等键 = 事件类型 + 实体 ID + 更新时间戳 + 变更状态哈希
    const statusHash = databaseEntity
      ? `${databaseEntity.status}->${entity.status}`
      : 'INSERT';

    return `event:${eventType}:${entity.id}:${entity.updatedAt?.getTime() || 'now'}:${statusHash}`;
  }

  // 去重检查
  private isDuplicateEvent(
    eventType: 'afterInsert' | 'afterUpdate' | 'beforeRemove',
    entity: MediaRequest,
    databaseEntity?: MediaRequest
  ): boolean {
    const key = this.getIdempotencyKey(eventType, entity, databaseEntity);
    const now = Date.now();

    // 清理过期键（惰性删除）
    for (const [k, exp] of this.idempotencyKeys) {
      if (exp < now) this.idempotencyKeys.delete(k);
    }

    if (this.idempotencyKeys.has(key)) {
      logger.debug('Duplicate event skipped by idempotency key', {
        label: 'Media Request',
        eventType,
        requestId: entity.id,
        idempotencyKey: key,
        fingerprint: ['event-deduplicated', eventType, String(entity.id)],
      });
      return true;
    }

    this.idempotencyKeys.set(key, now + this.IDEMPOTENCY_TTL);
    return false;
  }

  // 应用示例：在 afterUpdate 中使用
  public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
    if (!event.entity) return;

    // 幂等检查：同一事件 60 秒内只处理一次
    if (this.isDuplicateEvent('afterUpdate', event.entity, event.databaseEntity)) {
      return;
    }

    // 原逻辑继续...
    await this.sendToRadarr(event.entity);
  }
}
```

**幂等键分层设计（多实例部署需 Redis 实现）**:

```
幂等键层级:
┌─────────────────────────────────────────────────────┐
│  Global Level (跨实例共享) - Redis SETNX            │
│  key: jseer:idem:global:<eventIdempotencyHash>      │
│  TTL: 300s（覆盖扫描周期的 5 倍）                    │
├─────────────────────────────────────────────────────┤
│  Instance Level (实例内) - 内存 Map                  │
│  key: <eventType>:<entityId>:<updatedAtHash>         │
│  TTL: 60s（覆盖同一会话内的重试）                     │
├─────────────────────────────────────────────────────┤
│  DB Level (持久化) - 数据库列约束                    │
│  UNIQUE(tmdbId, mediaType, is4k)                     │
│  + 状态变更前检查 entity.status !== targetStatus      │
└─────────────────────────────────────────────────────┘
```

**状态变更幂等矩阵**:
| databaseEntity.status | entity.status | 允许处理？ | 说明 |
|----------------------|---------------|-----------|------|
| PENDING | APPROVED | ✅ | 正常审批流程 |
| APPROVED | APPROVED | ❌ | 重复事件，跳过 |
| APPROVED | COMPLETED | ✅ | 媒体已下载完成 |
| PENDING | DECLINED | ✅ | 正常拒绝 |
| PENDING | PENDING | ❌ | 无状态变更，跳过 |
| APPROVED | FAILED | ✅ | 处理失败，标记失败状态 |

---

## 八、关键设计模式总结

### 8.1 异步非阻塞设计
- 发送到 *arr 服务使用 Promise.then().catch()，不阻塞请求响应
- 通知发送 fire-and-forget，不等待结果
- 扫描任务分批处理，避免长时间阻塞

### 8.2 事件驱动架构
- TypeORM 的 `@AfterInsert` / `@AfterUpdate` / `@AfterRemove` 钩子
- `EventSubscriber` 监听实体变化，触发后续流程
- **事件回放**: 利用 `event.databaseEntity` 与 `event.entity` 对比检测状态变更
- **事务边界**: 删除操作在 `beforeRemove` 中使用传入的 `manager` 保证原子性
- **回放幂等键**: 事件类型 + 实体 ID + updatedAt 时间戳 + 状态变更哈希，三层去重（Global Redis → Instance Map → DB Constraints）

### 8.3 幂等性设计
- 重复请求检查 (`DuplicateMediaRequestError`)
- 状态变更前检查当前状态，避免重复标记
- AsyncLock 防止同一媒体并发操作
- **防循环调用**: afterUpdate 中先校验状态再处理，避免级联更新触发死循环
- **事件回放幂等**: `getIdempotencyKey()` + `isDuplicateEvent()` 60 秒去重窗口
- **状态变更幂等矩阵**: 6 种典型状态转移的允许/拒绝决策表

### 8.4 容错机制
- 多级错误捕获 (API 调用层 + 实体保存层)
- 失败状态标记 + 通知 + 重试机制
- 缓存失效时返回旧数据 (`getRolling` 策略)
- **5 级回滚优先级**: P0 删除 → P1 拒绝 → P2 API 失败 → P3 连接错误 → P4 重试
- **Sentry 告警键关联**: 每级回滚绑定唯一 fingerprint 模板 + Level 自动映射 + PagerDuty 升级策略
- **4K 降回中间态保护**: 三元表达式条件赋值，处理中保留外部服务 ID 不被误清

### 8.5 扩展性设计
- 通知 Agent 接口抽象，易于新增通知渠道
- 扫描器基类 `BaseScanner` 可扩展新的扫描源
- 媒体服务器类型通过枚举支持 (Plex/Jellyfin/Emby)
- **10 种通知 Agent 标准化扩展**: BaseAgent 抽象 + NotificationAgent 接口 + 位掩码类型过滤
- **Agent 插件热加载**: chokidar 文件监听 + require.cache 清除 + dispose 生命周期 + 接口校验 + 版本兼容性检查

### 8.6 背压与流控
- 扫描器三层背压: Session ID 隔离 → 分批节流 → 可取消标志
- AvailabilitySync 异步生成器分页，内存占用稳定
- axios-rate-limit 控制外部 API 调用频率
- **AsyncLock 死锁预防**: setImmediate 异步释放 + 无界监听器 + 非递归设计
- **四级熔断水位线**: IDLE → NORMAL → WARN(10% 超时, ×1.5) → CRITICAL(30% 超时, ×3) → FATAL(50% 超时, 全局终止)
- **硬阈值快速熔断**: DB 错误 ≥5 次、API 429 ≥15 次直接升级 FATAL
- **健康检查 API**: 暴露 watermark、backoffMultiplier、stats 等实时监控指标

### 8.7 数据一致性
- **4K 双轨迁移**: 临时表重建 + 历史数据无缝迁移 + 回滚方案
- **AvailabilitySync 漂移补偿**: 双重校验 (*arr + 媒体服务器) + 处理中保护 + 季级粒度 + TMDB 兜底
- **状态保护**: 处理中请求保留外部服务元数据，不盲目清空
- **4K→1080p 中间态矩阵**: 5 种典型场景 × 6 种状态机转移路径，全覆盖无盲区
- **漂移超 24h 兜底降级**: 3 类漂移场景（PROCESSING 超时 / PARTIALLY_AVAILABLE 卡壳 / DELETED 误标）+ 7 级状态降级决策表
- **扫描器心跳检查**: 30 分钟无 heartbeat 自动 cancel + 重启调度

### 8.8 分布式考量
- Rate-Limit 当前为内存实现，分布式部署需 Redis 集中限流
- AsyncLock 为进程内锁，多实例部署需分布式锁 (Redis Redlock)
- 通知发送无重复投递保证，需消费方幂等处理
- 扫描任务无分布式协调，多实例部署可能重复执行
- **Redis 令牌桶改造**: Lua 脚本原子令牌获取 + SET NX PX 自动过期 + 本地 50ms 批量同步
- **Redlock 死锁识别**: 30s 锁超时 + 强制释放日志 + Sentry fingerprint 聚合
- **全局幂等键同步**: Redis SETNX 300s TTL 覆盖多实例事件去重
- **跨实例锁诊断 API**: getDeadlockReport() 输出持锁堆栈、预警级别、PID 等信息
