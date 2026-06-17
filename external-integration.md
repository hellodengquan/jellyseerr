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

### 3.3 异步锁 (AsyncLock)

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

### 3.4 缓存机制

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

---

## 七、错误回滚机制

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

## 八、关键设计模式总结

### 8.1 异步非阻塞设计
- 发送到 *arr 服务使用 Promise.then().catch()，不阻塞请求响应
- 通知发送 fire-and-forget，不等待结果
- 扫描任务分批处理，避免长时间阻塞

### 8.2 事件驱动架构
- TypeORM 的 `@AfterInsert` / `@AfterUpdate` / `@AfterRemove` 钩子
- `EventSubscriber` 监听实体变化，触发后续流程

### 8.3 幂等性设计
- 重复请求检查 (`DuplicateMediaRequestError`)
- 状态变更前检查当前状态，避免重复标记
- AsyncLock 防止同一媒体并发操作

### 8.4 容错机制
- 多级错误捕获 (API 调用层 + 实体保存层)
- 失败状态标记 + 通知 + 重试机制
- 缓存失效时返回旧数据 (`getRolling` 策略)

### 8.5 扩展性设计
- 通知 Agent 接口抽象，易于新增通知渠道
- 扫描器基类 `BaseScanner` 可扩展新的扫描源
- 媒体服务器类型通过枚举支持 (Plex/Jellyfin/Emby)
