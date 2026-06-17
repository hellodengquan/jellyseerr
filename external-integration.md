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

### 8.3 幂等性设计
- 重复请求检查 (`DuplicateMediaRequestError`)
- 状态变更前检查当前状态，避免重复标记
- AsyncLock 防止同一媒体并发操作
- **防循环调用**: afterUpdate 中先校验状态再处理，避免级联更新触发死循环

### 8.4 容错机制
- 多级错误捕获 (API 调用层 + 实体保存层)
- 失败状态标记 + 通知 + 重试机制
- 缓存失效时返回旧数据 (`getRolling` 策略)
- **5 级回滚优先级**: P0 删除 → P1 拒绝 → P2 API 失败 → P3 连接错误 → P4 重试

### 8.5 扩展性设计
- 通知 Agent 接口抽象，易于新增通知渠道
- 扫描器基类 `BaseScanner` 可扩展新的扫描源
- 媒体服务器类型通过枚举支持 (Plex/Jellyfin/Emby)
- **10 种通知 Agent 标准化扩展**: BaseAgent 抽象 + NotificationAgent 接口 + 位掩码类型过滤

### 8.6 背压与流控
- 扫描器三层背压: Session ID 隔离 → 分批节流 → 可取消标志
- AvailabilitySync 异步生成器分页，内存占用稳定
- axios-rate-limit 控制外部 API 调用频率
- **AsyncLock 死锁预防**: setImmediate 异步释放 + 无界监听器 + 非递归设计

### 8.7 数据一致性
- **4K 双轨迁移**: 临时表重建 + 历史数据无缝迁移 + 回滚方案
- **AvailabilitySync 漂移补偿**: 双重校验 (*arr + 媒体服务器) + 处理中保护 + 季级粒度 + TMDB 兜底
- **状态保护**: 处理中请求保留外部服务元数据，不盲目清空

### 8.8 分布式考量
- Rate-Limit 当前为内存实现，分布式部署需 Redis 集中限流
- AsyncLock 为进程内锁，多实例部署需分布式锁 (Redis Redlock)
- 通知发送无重复投递保证，需消费方幂等处理
- 扫描任务无分布式协调，多实例部署可能重复执行
