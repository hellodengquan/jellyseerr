# 媒体请求生命周期与状态流转

本文档梳理 Jellyseerr 中一条媒体请求（MediaRequest）从提交到批准再到拉取到位的完整状态流转，结合源码分析每个状态切换的触发条件与驱动机制。

---

## 一、核心状态枚举

### 1. 请求状态 — `MediaRequestStatus`

> 定义位置：`server/constants/media.ts:1-7`

| 值 | 名称 | 含义 |
|---|---|---|
| 1 | `PENDING` | 待审批 |
| 2 | `APPROVED` | 已批准，等待服务端拉取 |
| 3 | `DECLINED` | 已拒绝 |
| 4 | `FAILED` | 推送到 Radarr/Sonarr 失败 |
| 5 | `COMPLETED` | 媒体已拉取到位 |

### 2. 媒体状态 — `MediaStatus`

> 定义位置：`server/constants/media.ts:14-22`

| 值 | 名称 | 含义 |
|---|---|---|
| 1 | `UNKNOWN` | 未知/无请求关联 |
| 2 | `PENDING` | 有待审批请求 |
| 3 | `PROCESSING` | 请求已批准，正在拉取 |
| 4 | `PARTIALLY_AVAILABLE` | 部分可用（TV 部分季已到位） |
| 5 | `AVAILABLE` | 完全可用 |
| 6 | `BLOCKLISTED` | 已加入黑名单 |
| 7 | `DELETED` | 已从媒体服务器删除 |

---

## 二、完整状态流转图

```
                          ┌──────────────────────────────────────────────────┐
                          │            用户提交请求 (POST /api/v1/request)    │
                          └────────────────────┬─────────────────────────────┘
                                               │
                              ┌────────────────┴────────────────┐
                              │                                 │
                     有自动审批权限                    无自动审批权限
                              │                                 │
                              ▼                                 ▼
                       ┌──────────┐                       ┌──────────┐
                       │ APPROVED │                       │ PENDING  │
                       └────┬─────┘                       └────┬─────┘
                            │                                  │
                 ┌──────────┴──────────┐            ┌─────────┴─────────┐
                 │                     │            │                   │
         媒体已 AVAILABLE        推送到 *arr       管理员批准        管理员拒绝
                 │              (Radarr/Sonarr)   (PUT /:id/approve)  (PUT /:id/decline)
                 ▼                     │            │                   │
          ┌───────────┐        ┌──────┴──────┐     ▼                   ▼
          │ COMPLETED │        │             │  ┌──────────┐      ┌──────────┐
          └───────────┘        ▼             ▼  │ APPROVED │      │ DECLINED │
                          推送成功        推送失败  └────┬─────┘      └──────────┘
                               │             │         │
                               │             ▼         │ 同上后续流程
                               │      ┌──────────┐    │
                               │      │  FAILED  │    │
                               │      └────┬─────┘    │
                               │           │          │
                               │      管理员重试       │
                               │   (POST /:id/retry)  │
                               │           │          │
                               │           ▼          │
                               │      ┌──────────┐   │
                               │      │ APPROVED │───┘
                               │      └──────────┘
                               │
                        媒体服务器同步
                     (Plex/Jellyfin 扫描回调)
                               │
                               ▼
                        ┌───────────┐
                        │ COMPLETED │
                        └───────────┘
```

---

## 三、各阶段详细分析

### 阶段 1：请求提交（创建）

**入口**：`POST /api/v1/request` → `server/routes/request.ts:303-336`

**核心方法**：`MediaRequest.request()` → `server/entity/MediaRequest.ts:47-523`

#### 1.1 初始状态决定

请求创建时的 `status` 取决于提交用户是否拥有自动审批权限：

```typescript
// server/entity/MediaRequest.ts:354-367 (电影)
status: user.hasPermission(
  [
    requestBody.is4k
      ? Permission.AUTO_APPROVE_4K
      : Permission.AUTO_APPROVE,
    requestBody.is4k
      ? Permission.AUTO_APPROVE_4K_MOVIE
      : Permission.AUTO_APPROVE_MOVIE,
    Permission.MANAGE_REQUESTS,
  ],
  { type: 'or' }
)
  ? MediaRequestStatus.APPROVED
  : MediaRequestStatus.PENDING,
```

**触发条件**：
- 用户拥有 `AUTO_APPROVE` / `AUTO_APPROVE_4K` / `AUTO_APPROVE_MOVIE` / `AUTO_APPROVE_4K_MOVIE` / `MANAGE_REQUESTS` 中任一权限 → 初始状态为 **APPROVED**
- 否则 → 初始状态为 **PENDING**

TV 请求逻辑相同（`server/entity/MediaRequest.ts:464-477`），权限替换为 TV 相关的 `AUTO_APPROVE_TV` / `AUTO_APPROVE_4K_TV`。

#### 1.2 关联的 Media 状态初始化

创建请求时同步创建/更新 `Media` 记录：

```typescript
// server/entity/MediaRequest.ts:136-168
if (!media) {
  media = new Media({
    status: !requestBody.is4k ? MediaStatus.PENDING : MediaStatus.UNKNOWN,
    status4k: requestBody.is4k ? MediaStatus.PENDING : MediaStatus.UNKNOWN,
    ...
  });
} else {
  if ((media.status === MediaStatus.UNKNOWN || media.status === MediaStatus.DELETED) && !requestBody.is4k) {
    media.status = MediaStatus.PENDING;
  }
  // 4k 同理
}
```

**规则**：Media 的 `status`/`status4k` 仅在 `UNKNOWN` 或 `DELETED` 时才被提升为 `PENDING`，已处于更高级状态的不会被降级。

#### 1.3 前置校验（可能阻断创建）

| 校验项 | 异常类型 | HTTP 状态码 |
|---|---|---|
| 无请求权限 | `RequestPermissionError` | 403 |
| 配额超限 | `QuotaRestrictedError` | 403 |
| 重复请求（电影未 DECLINED/COMPLETED） | `DuplicateMediaRequestError` | 409 |
| 无可请求的季 | `NoSeasonsAvailableError` | 202 |
| 媒体被拉黑 | `BlocklistedMediaError` | 403 |

#### 1.4 创建后的自动动作

通过 TypeORM `@AfterInsert` 钩子（`server/entity/MediaRequest.ts:629-655`）：
- **PENDING** → 发送 `MEDIA_PENDING` 通知给管理员
- **APPROVED**（自动审批）→ 调用 `notifyApprovedOrDeclined(true)` 发送 `MEDIA_AUTO_APPROVED` 通知

通过 `MediaRequestSubscriber.afterInsert()`（`server/subscriber/MediaRequestSubscriber.ts:1045-1073`）：
- 调用 `sendToRadarr()` / `sendToSonarr()` — 若状态为 APPROVED，推送到对应 *arr
- 调用 `updateParentStatus()` — 更新父 Media 状态

---

### 阶段 2：审批 / 拒绝

**入口**：`PUT /api/v1/request/:requestId/:status` → `server/routes/request.ts:663-705`

需要 `MANAGE_REQUESTS` 权限。

```typescript
// server/routes/request.ts:680-694
switch (req.params.status) {
  case 'pending':  newStatus = MediaRequestStatus.PENDING;   break;
  case 'approve':  newStatus = MediaRequestStatus.APPROVED;  break;
  case 'decline':  newStatus = MediaRequestStatus.DECLINED;  break;
}
request.status = newStatus;
request.modifiedBy = req.user;
await requestRepository.save(request);
```

#### 2.1 PENDING → APPROVED

**触发条件**：管理员调用 `PUT /:requestId/approve`

**后续动作**（由 `MediaRequestSubscriber.afterUpdate` 驱动）：

1. **`sendToRadarr()` / `sendToSonarr()`**（`MediaRequestSubscriber.ts:183-474, 477-817`）
   - 前置检查：请求状态为 `APPROVED` 且类型匹配
   - 查找默认 *arr 服务器配置
   - 若媒体已 `AVAILABLE` → 直接标记请求为 `COMPLETED`（短路）
   - 调用 `radarr.addMovie()` / `sonarr.addSeries()` 推送请求
   - 推送成功 → 更新 Media 的 `externalServiceId`、`serviceId` 等
   - 推送失败 → 请求状态设为 `FAILED`，发送 `MEDIA_FAILED` 通知

2. **`updateParentStatus()`**（`MediaRequestSubscriber.ts:820-943`）
   - 请求 APPROVED 且 Media 非 AVAILABLE/PARTIALLY_AVAILABLE/PROCESSING → Media 状态设为 `PROCESSING`
   - TV 类型：子季 `SeasonRequest.status` 设为 `APPROVED`

#### 2.2 PENDING → DECLINED

**触发条件**：管理员调用 `PUT /:requestId/decline`

**后续动作**（由 `updateParentStatus()` 驱动）：

1. **电影**：Media 状态 → `UNKNOWN`（如果非 DELETED）
2. **TV**：
   - 若无其他 PENDING 请求 → Media 状态 → `UNKNOWN`
   - 所有 `SeasonRequest.status` → `DECLINED`
   - 若无其他活跃请求的季 → Season 状态 → `UNKNOWN`

#### 2.3 通知机制

通过 TypeORM `@AfterUpdate` 钩子（`server/entity/MediaRequest.ts:663-719`）：
- APPROVED → 发送 `MEDIA_APPROVED` 通知（或自动审批时发送 `MEDIA_AUTO_APPROVED`）
- DECLINED → 发送 `MEDIA_DECLINED` 通知
- 若 APPROVED 时媒体已 AVAILABLE → 改发 `MEDIA_AVAILABLE`

---

### 阶段 3：推送到 *arr 服务

由 `MediaRequestSubscriber` 在 `afterInsert` 和 `afterUpdate` 中自动驱动。

#### 3.1 电影 → Radarr

**方法**：`MediaRequestSubscriber.sendToRadarr()`（`MediaRequestSubscriber.ts:183-474`）

```
APPROVED  →  检查 Radarr 配置
              │
              ├─ 无默认服务器 → 仅日志警告，状态不变
              ├─ 媒体已 AVAILABLE → COMPLETED（短路）
              └─ 调用 radarr.addMovie()
                   │
                   ├─ 成功 → 更新 Media.externalServiceId 等
                   └─ 失败 → 请求状态 → FAILED + 发送 MEDIA_FAILED 通知
```

#### 3.2 TV → Sonarr

**方法**：`MediaRequestSubscriber.sendToSonarr()`（`MediaRequestSubscriber.ts:477-817`）

逻辑与 Radarr 对称，额外处理：
- TVDB ID 缺失 → 删除请求和媒体记录
- 动漫关键词检测 → 使用动漫专用配置
- 季级别请求映射

---

### 阶段 4：媒体拉取到位（APPROVED → COMPLETED）

这是最关键的状态闭环：**请求本身不会主动轮询 *arr 来判断是否下载完成**，而是由 **Media 状态变更** 反向驱动请求状态更新。

#### 4.1 Media 状态如何变为 AVAILABLE

有两个来源：

**来源 A：媒体服务器扫描（Plex/Jellyfin 同步）**

媒体服务器（Plex/Jellyfin）定期扫描库，发现新文件后通过 webhook 或定时同步任务更新 `Media` 记录的 `status`/`status4k` 为 `AVAILABLE`。

**来源 B：AvailabilitySync 定时任务**

`server/lib/availabilitySync.ts` — 定期检查所有标记为 AVAILABLE 的媒体是否仍然存在于媒体服务器/Radarr/Sonarr 中，若不存在则降级为 `DELETED`。

#### 4.2 Media 变更驱动请求状态更新

**核心驱动**：`MediaSubscriber`（`server/subscriber/MediaSubscriber.ts`）

当 `Media` 实体被更新时，TypeORM 触发 `beforeUpdate` 和 `afterUpdate` 钩子：

**`beforeUpdate`**（`MediaSubscriber.ts:124-153`）：
```typescript
// 媒体从 PENDING → AVAILABLE 时，自动将关联的 PENDING 请求提升为 APPROVED
if (event.entity.status === MediaStatus.AVAILABLE
    && event.databaseEntity.status === MediaStatus.PENDING) {
  this.updateChildRequestStatus(event.entity, false);
}
```
这处理了一种场景：**媒体已经存在于 Plex/Jellyfin 中，但请求还是 PENDING**。此时请求直接变为 APPROVED。

**`afterUpdate`**（`MediaSubscriber.ts:155-200`）：
```typescript
// 媒体变为 AVAILABLE/PARTIALLY_AVAILABLE/DELETED 时，将关联的 APPROVED/FAILED 请求标记为 COMPLETED
if (validStatuses.includes(event.entity.status)) {
  this.updateRelatedMediaRequest(event.entity, event.databaseEntity, false);
}
```

**`updateRelatedMediaRequest()`** 详细逻辑（`MediaSubscriber.ts:34-121`）：

- 查找该 Media 下所有 `APPROVED` 或 `FAILED` 状态的请求
- **电影**：Media 变为 `AVAILABLE` 或 `DELETED` → 请求状态 → `COMPLETED`
- **TV**：逐一检查请求中每个季的状态：
  - 季变为 `AVAILABLE` 或 `DELETED` → 对应 `SeasonRequest.status` → `COMPLETED`
  - 所有请求的季都到位 → 请求整体状态 → `COMPLETED`

#### 4.3 APPROVED → COMPLETED 的另一条路径

在 `sendToRadarr()` / `sendToSonarr()` 中，如果推送前发现媒体已经是 `AVAILABLE`，则直接短路：

```typescript
// MediaRequestSubscriber.ts:348-360
if (media[entity.is4k ? 'status4k' : 'status'] === MediaStatus.AVAILABLE) {
  entity.status = MediaRequestStatus.COMPLETED;
  await requestRepository.save(entity);
  return;
}
```

---

### 阶段 5：失败重试

**入口**：`POST /api/v1/request/:requestId/retry` → `server/routes/request.ts:633-661`

需要 `MANAGE_REQUESTS` 权限。

```typescript
request.status = MediaRequestStatus.APPROVED;
request.modifiedBy = req.user;
await requestRepository.save(request);
```

将 `FAILED` 状态重新设为 `APPROVED`，触发 `afterUpdate` 钩子，重新走推送 *arr 流程。

---

### 阶段 6：请求删除

**入口**：`DELETE /api/v1/request/:requestId` → `server/routes/request.ts:601-631`

**权限**：
- 管理员（`MANAGE_REQUESTS`）可删除任何请求
- 普通用户只能删除自己且状态为 `PENDING` 的请求

**后续动作**（`MediaRequestSubscriber.afterRemove()` → `handleRemoveParentUpdate()`，`MediaRequestSubscriber.ts:946-1004`）：

- 检查 Media 下是否还有活跃请求（非 COMPLETED/DECLINED）
- 无活跃请求 → Media 状态重置：
  - 有过 COMPLETED 请求 → `DELETED`
  - 无 COMPLETED 请求 → `UNKNOWN`

---

## 四、双重状态体系交互总结

Jellyseerr 维护了两套独立但联动的状态：

| 维度 | 实体 | 枚举 | 侧重点 |
|---|---|---|---|
| 请求状态 | `MediaRequest` / `SeasonRequest` | `MediaRequestStatus` | 审批流程 |
| 媒体状态 | `Media` / `Season` | `MediaStatus` | 可用性 |

**联动规则**：

```
请求 PENDING    → Media PENDING
请求 APPROVED   → Media PROCESSING（若 Media 未达更高级状态）
请求 DECLINED   → Media UNKNOWN（若无其他活跃请求）
Media AVAILABLE → 请求 COMPLETED（反向驱动）
```

---

## 五、关键代码文件索引

| 文件 | 职责 |
|---|---|
| `server/constants/media.ts` | 状态枚举定义 |
| `server/entity/MediaRequest.ts` | 请求实体 + 静态创建方法 `request()` + `@AfterInsert`/`@AfterUpdate` 通知钩子 + 重复请求检测 |
| `server/entity/SeasonRequest.ts` | 季请求实体 |
| `server/entity/Media.ts` | 媒体实体 + 状态管理 |
| `server/routes/request.ts` | REST API：创建/审批/拒绝/重试/删除请求 |
| `server/subscriber/MediaRequestSubscriber.ts` | 请求实体事件订阅器：推送到 *arr + 更新父 Media 状态 + 删除回收 |
| `server/subscriber/MediaSubscriber.ts` | 媒体实体事件订阅器：反向驱动请求状态 → COMPLETED |
| `server/lib/availabilitySync.ts` | 定时同步：校验媒体是否仍存在于媒体服务器 |
| `server/utils/asyncLock.ts` | 进程内互斥锁：防止单实例内重复创建 |
| `server/api/servarr/radarr.ts` | Radarr API 封装：幂等添加电影 |
| `server/api/servarr/sonarr.ts` | Sonarr API 封装：幂等添加剧集 |
| `server/api/servarr/base.ts` | *arr API 基类：超时配置、缓存管理 |
| `server/lib/scanners/baseScanner.ts` | 扫描器基类：AsyncLock 使用示例 |
| `server/entity/OverrideRule.ts` | 覆盖规则实体：条件匹配 + 参数覆盖 |
| `server/entity/User.ts` | 用户实体：配额字段、`getQuota()` 计算 |
| `server/lib/permissions.ts` | 权限枚举与 `hasPermission()` 位运算检查 |
| `server/lib/watchlistsync.ts` | Watchlist 同步：自动请求创建流程 |
| `server/job/schedule.ts` | 定时任务调度：所有扫描/同步/清理任务的注册与 cron 配置 |
| `server/job/blocklistedTagsProcessor.ts` | 拉黑标签处理器：唯一批量清理 Media + Request 的组件 |
| `server/lib/scanners/baseScanner.ts` | 扫描器基类：processMovie / processShow 状态更新逻辑 |
| `server/lib/scanners/radarr/index.ts` | Radarr 扫描器：正向同步 + 孤儿清理 |
| `server/lib/scanners/sonarr/index.ts` | Sonarr 扫描器：正向同步 + 孤儿清理 |
| `server/lib/settings/index.ts` | 全局设置：JobId / JobSettings / 默认 cron 表达式 |

---

## 六、状态流转速查表

| 当前状态 | 目标状态 | 触发方式 | 触发位置 |
|---|---|---|---|
| — | PENDING | 用户提交请求（无自动审批权限） | `MediaRequest.request()` |
| — | APPROVED | 用户提交请求（有自动审批权限） | `MediaRequest.request()` |
| PENDING | APPROVED | 管理员批准 / 媒体变为 AVAILABLE 自动提升 | `PUT /:id/approve` / `MediaSubscriber.beforeUpdate` |
| PENDING | DECLINED | 管理员拒绝 | `PUT /:id/decline` |
| APPROVED | COMPLETED | 媒体已 AVAILABLE（推送前短路）/ 媒体服务器同步到位 | `sendToRadarr()`/`sendToSonarr()` / `MediaSubscriber.afterUpdate` |
| APPROVED | FAILED | 推送到 *arr 失败 | `sendToRadarr()`/`sendToSonarr()` catch 块 |
| FAILED | APPROVED | 管理员重试 | `POST /:id/retry` |
| 任意 | (删除) | 管理员/用户删除请求 | `DELETE /:id` |

---

## 七、请求被驳回后的回收流程

当请求被管理员拒绝（`PENDING → DECLINED`）或被删除时，系统需要执行一系列清理操作，回滚 Media 和 Season 的状态，避免资源泄漏。

### 7.1 拒绝请求的回收逻辑

**入口**：`MediaRequestSubscriber.updateParentStatus()` → `server/subscriber/MediaRequestSubscriber.ts:820-943`

#### 电影请求拒绝

```typescript
// MediaRequestSubscriber.ts:849-856
if (
  media.mediaType === MediaType.MOVIE &&
  entity.status === MediaRequestStatus.DECLINED &&
  media[statusKey] !== MediaStatus.DELETED
) {
  media[statusKey] = MediaStatus.UNKNOWN;
  await mediaRepository.save(media);
}
```

**规则**：
- Media 状态从 `PENDING` / `PROCESSING` 回滚为 `UNKNOWN`
- 但如果 Media 已经是 `DELETED`（已从媒体服务器移除），则不做修改

#### TV 请求拒绝

TV 的回收逻辑更复杂，需要同时考虑：
1. 是否还有其他 PENDING 请求
2. 各季的状态回滚
3. 其他活跃请求对季状态的影响

```typescript
// MediaRequestSubscriber.ts:864-888
if (media.mediaType === MediaType.TV &&
    entity.status === MediaRequestStatus.DECLINED &&
    media[statusKey] === MediaStatus.PENDING) {
  const pendingCount = await requestRepository.count({ ... });
  if (pendingCount === 0) {
    freshMedia[statusKey] = MediaStatus.UNKNOWN;
  }
}
```

**季级别的回收**：
```typescript
// MediaRequestSubscriber.ts:890-932
for (const seasonRequest of entity.seasons) {
  seasonRequest.status = MediaRequestStatus.DECLINED;
  
  // 若该季没有其他活跃请求，则回滚 Season 状态
  if (season && season[statusKey] === MediaStatus.PENDING) {
    const otherActiveRequests = await requestRepository.createQueryBuilder(...)
      .where('request.id != :requestId', { requestId: entity.id })
      .andWhere('request.status NOT IN (:...statuses)', {
        statuses: [MediaRequestStatus.DECLINED, MediaRequestStatus.COMPLETED]
      })
      .andWhere('season.seasonNumber = :seasonNumber', { ... })
      .getCount();
    
    if (otherActiveRequests === 0) {
      season[statusKey] = MediaStatus.UNKNOWN;
    }
  }
}
```

**回收规则总结**：

| 对象 | 条件 | 回收后状态 |
|---|---|---|
| 电影 Media | 非 DELETED | `UNKNOWN` |
| TV Media | 无其他 PENDING 请求 | `UNKNOWN` |
| SeasonRequest | 父请求 DECLINED | `DECLINED` |
| Season | 无其他活跃请求且当前为 PENDING | `UNKNOWN` |

### 7.2 删除请求的回收逻辑

**入口**：`MediaRequestSubscriber.handleRemoveParentUpdate()` → `server/subscriber/MediaRequestSubscriber.ts:946-1004`

```typescript
// 检查是否还有活跃请求（非 COMPLETED / DECLINED）
const hasActive = fullMedia.requests.some(
  (request) => !request.is4k &&
    request.status !== MediaRequestStatus.COMPLETED &&
    request.status !== MediaRequestStatus.DECLINED
);

if (needsStatusUpdate) {
  const hadCompleted = fullMedia.requests.some(
    (r) => !r.is4k && r.status === MediaRequestStatus.COMPLETED
  );
  cleanMedia.status = hadCompleted
    ? MediaStatus.DELETED    // 有过已完成请求 → 标记为已删除
    : MediaStatus.UNKNOWN;   // 无已完成请求 → 重置为未知
}
```

**删除回收规则**：
- 无活跃请求时才触发 Media 状态重置
- 若该 Media 曾有过 `COMPLETED` 请求 → 标记为 `DELETED`（表示曾经存在过但现已移除）
- 若该 Media 从未有过 `COMPLETED` 请求 → 标记为 `UNKNOWN`（干净重置）

---

## 八、多实例部署时的去重和互斥机制

Jellyseerr 通过**三层防护**确保多实例部署或高并发场景下不会产生重复请求：

### 8.1 第一层：应用层互斥锁（AsyncLock）

**实现**：`server/utils/asyncLock.ts`

```typescript
// 使用方式：将"检查-创建"的临界区包裹在 asyncLock.dispatch 中
await this.asyncLock.dispatch(tmdbId, async () => {
  const existing = await this.getExisting(tmdbId, mediaType);
  if (existing) {
    // 更新现有记录
  } else {
    // 创建新记录
  }
});
```

**工作原理**：
- 基于 `tmdbId` 为 key 的内存级互斥锁
- 同一 `tmdbId` 的操作串行化执行
- 使用 EventEmitter 实现等待队列，避免 busy-wait
- 单个进程内保证不会创建重复 Media 记录

**使用场景**：
- `BaseScanner.processMovie()` / `processShow()` → 扫描器处理媒体时
- 所有"查找或创建"模式的数据库操作

**局限性**：仅在单进程内有效，多实例部署时需依赖数据库层约束。

### 8.2 第二层：业务逻辑去重检查

**实现**：`MediaRequest.request()` → `server/entity/MediaRequest.ts:171-216`

```typescript
// 检查是否已有相同媒体的请求
const existing = await requestRepository
  .createQueryBuilder('request')
  .leftJoinAndSelect('request.media', 'media')
  .where('request.is4k = :is4k', { is4k: requestBody.is4k })
  .andWhere('media.tmdbId = :tmdbId', { tmdbId: tmdbMedia.id })
  .andWhere('media.mediaType = :mediaType', { mediaType: requestBody.mediaType })
  .getMany();

// 电影去重规则
if (requestBody.mediaType === MediaType.MOVIE &&
    existing[0].status !== MediaRequestStatus.DECLINED &&
    existing[0].status !== MediaRequestStatus.COMPLETED) {
  throw new DuplicateMediaRequestError('Request for this media already exists.');
}

// 自动请求去重
const statusKey = requestBody.is4k ? 'status4k' : 'status';
if (existing.find(
  (r) => r.requestedBy.id === requestUser.id &&
    r.isAutoRequest &&
    r.media?.[statusKey] !== MediaStatus.DELETED
)) {
  throw new DuplicateMediaRequestError('Auto-request for this media and user already exists.');
}
```

**去重规则**：

| 场景 | 去重条件 | 异常 |
|---|---|---|
| 电影请求 | 存在非 DECLINED / 非 COMPLETED 的同类型请求 | `DuplicateMediaRequestError` (409) |
| 自动请求 | 同一用户 + 同一媒体 + 非 DELETED 状态 | `DuplicateMediaRequestError` (409) |
| TV 请求季去重 | 过滤掉已有请求或已可用的季（`NoSeasonsAvailableError`） | `NoSeasonsAvailableError` (202) |

### 8.3 第三层：数据库唯一约束

通过数据库级别的唯一索引作为最后一道防线：

| 实体 | 唯一约束 | 作用 | 定义位置 |
|---|---|---|---|
| `Media` | `(tmdbId, mediaType)` 联合索引 | 防止同一媒体创建多条 Media 记录 | `server/entity/Media.ts:30` |
| `Blocklist` | `(tmdbId, mediaType)` 唯一约束 | 防止同一媒体重复拉黑 | `server/entity/Blocklist.ts:21` |
| `Watchlist` | `(tmdbId, mediaType, requestedBy)` 唯一约束 | 同一用户不能重复添加同一媒体到监视列表 | `server/entity/Watchlist.ts:29` |
| `Season` | `(mediaId, seasonNumber)` 隐式约束 | 通过 `@OneToMany` cascade 保证 | `server/entity/Media.ts:118` |

**迁移历史**：`server/migration/sqlite/1772047972752-AddMediaTypeToUniqueConstraints.ts` 中专门为 `Blocklist` 和 `Watchlist` 添加了 `mediaType` 到唯一约束中，确保同一 TMDB ID 作为电影和剧集时互不影响。

### 8.4 多实例部署注意事项

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Instance A │     │  Instance B │     │  Instance C │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           ▼
                    ┌─────────────┐
                    │  SQLite/PG  │   ◄─── 数据库唯一约束是唯一跨实例的互斥机制
                    └─────────────┘
```

**风险与限制**：
- `AsyncLock` 是内存级别的，多实例部署时无效
- 业务层去重检查（`SELECT` + `INSERT`）存在 TOCTOU 竞态条件
- **唯一依赖数据库唯一约束保证最终一致性**
- 建议：生产环境多实例部署时使用 PostgreSQL，并配合 `SERIALIZABLE` 事务隔离级别或悲观锁（`SELECT ... FOR UPDATE`）

---

## 九、Sonarr / Radarr 集成失败的重试和降级策略

### 9.1 集成调用的幂等设计

`sendToRadarr()` 和 `sendToSonarr()` 在推送前会先检查媒体是否已存在于 *arr 中，避免重复添加：

**Radarr 幂等逻辑** → `server/api/servarr/radarr.ts:118-248`
```typescript
public addMovie = async (options: RadarrMovieOptions): Promise<RadarrMovie> => {
  const movie = await this.getMovieByTmdbId(options.tmdbId);
  
  // 1. 已有文件 → 直接返回，跳过添加
  if (movie.hasFile) return movie;
  
  // 2. 已存在但未监控 → 更新为监控状态
  if (movie.id && !movie.monitored) {
    const response = await this.axios.put('/movie', { ... });
    if (options.searchNow) this.searchMovie(response.data.id);
    return response.data;
  }
  
  // 3. 已存在且已监控 → 直接返回，按需触发搜索
  if (movie.id) {
    if (options.searchNow && !movie.hasFile) {
      this.searchMovie(movie.id);
    }
    return movie;
  }
  
  // 4. 真正不存在 → 创建新记录
  const response = await this.axios.post('/movie', { ... });
  return response.data;
};
```

**Sonarr 幂等逻辑** → `server/api/servarr/sonarr.ts:191-310`

```typescript
public async addSeries(options: AddSeriesOptions): Promise<SonarrSeries> {
  const series = await this.getSeriesByTvdbId(options.tvdbid);
  
  // 1. 已存在 → 更新监控状态和季列表
  if (series.id) {
    series.monitored = options.monitored ?? series.monitored;
    series.seasons = this.buildSeasonList(options.seasons, series.seasons);
    const response = await this.axios.put('/series', series);
    
    // 重新监控缺失的集
    const episodes = await this.getEpisodes(response.data.id);
    const episodeIdsToMonitor = episodes
      .filter(ep => options.seasons.includes(ep.seasonNumber) && !ep.monitored)
      .map(ep => ep.id);
    if (episodeIdsToMonitor.length > 0) {
      await this.monitorEpisodes(episodeIdsToMonitor);
    }
    return response.data;
  }
  
  // 2. 不存在 → 创建
  const createdSeriesResponse = await this.axios.post('/series', { ... });
  return createdSeriesResponse.data;
}
```

### 9.2 失败处理流程

**入口**：`MediaRequestSubscriber.sendToRadarr()` catch 块 → `MediaRequestSubscriber.ts:398-473`

```typescript
radarr.addMovie(radarrMovieOptions)
  .then(async (radarrMovie) => {
    // 成功：更新 Media 的 externalServiceId 等
  })
  .catch(async () => {
    try {
      const requestRepository = getRepository(MediaRequest);
      // 防重复标记：避免重复 FAILED 状态写入
      if (entity.status !== MediaRequestStatus.FAILED) {
        entity.status = MediaRequestStatus.FAILED;
        await requestRepository.save(entity);
      }
    } catch (saveError) { ... }
    
    MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
  })
  . .finally(() => {
    // 无论成功失败，清理缓存
    .finally(() => {
      radarr.clearCache({
        tmdbId: movie.id,
        externalId: entity.is4k ? media.externalServiceId4k : media.externalServiceId,
      });
    })
```

**失败分级处理**：

| 失败场景 | 处理方式 | 请求状态 |
|---|---|---|
| 网络连接错误 / 配置错误 | catch 块捕获，标记为 FAILED | `FAILED` |
| API 返回非 2xx | catch 块捕获，标记为 FAILED | `FAILED` |
| 推送中途异常（已提交但更新 Media 失败） | 依赖 *arr 端幂等性，下次重试会走更新逻辑 | `FAILED`（取决于失败点） |
| 状态已为 FAILED（并发） | 跳过重复保存 | 保持 `FAILED` |

### 9.3 重试机制

**手动重试**（唯一支持的重试方式）：
```typescript
// server/routes/request.ts:633-661
requestRoutes.post('/:requestId/retry', ..., async (req, res) => {
  const request = await requestRepository.findOneOrFail({ ... });
  // 将 FAILED 重新设为 APPROVED，触发 afterUpdate 钩子
  request.status = MediaRequestStatus.APPROVED;
  request.modifiedBy = req.user;
  await requestRepository.save(request);
});
```

**重试触发的流程**：
1. `FAILED → APPROVED` 状态变更
2. `MediaRequestSubscriber.afterUpdate` 被触发
3. 重新调用 `sendToRadarr()` / `sendToSonarr()`
4. 由于 *arr API 是幂等的，会执行"查找 → 更新"而非重复创建

**无自动重试**：
- 系统不提供自动重试机制（无指数退避、无定时重试）
- 所有重试必须由管理员手动触发
- 设计考量：避免无效请求打爆 *arr 服务，让管理员介入判断失败原因

### 9.4 降级策略

| 降级配置 | 位置 | 默认值 | 作用 |
|---|---|---|---|
| `preventSearch` | `DVRSettings` | `false` | 为 `true` 时，添加媒体时不立即触发搜索（`searchNow: false`），由 *arr 后台索引器定时搜索 |
| `minimumAvailability` | `RadarrSettings` | - | Radarr 接受请求时的最小可用标准（`announced` / `inCinemas` / `released` / `preDB`） |
| `apiRequestTimeout` | `NetworkSettings` | `10000` ms | 所有 *arr API 请求的超时时间，避免长时间挂起 |
| `tagRequests` | `DVRSettings` | - | 为请求添加用户标签，便于在 *arr 中追溯来源 |

**降级工作流**：
```
管理员启用 preventSearch
    │
    ▼
请求批准 → sendToRadarr(searchNow: false)
    │
    ▼
Radarr 添加媒体但不立即搜索
    │
    ▼
Radarr 索引器按计划自动搜索
    │
    ▼
搜索命中 → 下载 → 媒体服务器扫描 → Media AVAILABLE → 请求 COMPLETED
```

**防止状态标记竞态**：
```typescript
// MediaRequestSubscriber.ts:402-405
if (entity.status !== MediaRequestStatus.FAILED) {
  entity.status = MediaRequestStatus.FAILED;
  await requestRepository.save(entity);
}
```
- 避免并发失败场景下重复写入 FAILED 状态
- 配合 `finally` 块中的缓存清理，确保下次重试不会命中脏缓存

### 9.5 故障模式与恢复

| 故障模式 | 对请求状态的影响 | 恢复方式 |
|---|---|---|
| *arr 服务宕机 | 请求 → `FAILED` | 服务恢复后管理员手动重试 |
| 配置错误（API Key 错误） | 请求 → `FAILED` | 修正配置后重试 |
| 磁盘空间不足 | *arr 拒绝 → `FAILED` | 清理空间后重试 |
| 媒体被 *arr 拒绝（版权/违规） | `FAILED` | 无法恢复，只能拒绝或删除请求 |
| 网络超时（API 实际成功了） | `FAILED` 但媒体实际已添加 | 重试会走幂等更新路径，自动恢复 |
| 媒体服务器未扫描到文件 | Media 长期 `PROCESSING` | 手动触发媒体库扫描 |

---

## 十、请求配额管理

配额机制在请求创建阶段进行拦截，限制用户在指定时间窗口内提交的请求数量，防止滥用。

### 10.1 配额模型

**配额维度**：电影和 TV 独立计算。

| 维度 | 电影 | TV |
|---|---|---|
| 计量单位 | 请求条数 | 请求涉及的季数（`SeasonRequest` 数量） |
| 限制字段 | `movieQuotaLimit` | `tvQuotaLimit` |
| 时间窗口 | `movieQuotaDays` | `tvQuotaDays` |
| 已用量 | `movieQuotaUsed` | `tvQuotaUsed` |

**配额来源优先级**：用户级配额 > 全局默认配额。

```typescript
// server/entity/User.ts:282-284
const movieQuotaLimit = !canBypass
  ? (this.movieQuotaLimit ?? defaultQuotas.movie.quotaLimit)
  : 0;
const movieQuotaDays = this.movieQuotaDays ?? defaultQuotas.movie.quotaDays;
```

- 用户字段 `movieQuotaLimit` / `tvQuotaLimit` 为 `null` 时回退到全局设置 `defaultQuotas`
- 拥有 `MANAGE_USERS` 权限的用户 `canBypass = true`，配额限制设为 0（即无限制）

### 10.2 配额检查时机

配额检查发生在 `MediaRequest.request()` 的早期阶段（`server/entity/MediaRequest.ts:114-120`），在去重检查和 *arr 推送之前：

```typescript
const quotas = await requestUser.getQuota();

if (requestBody.mediaType === MediaType.MOVIE && quotas.movie.restricted) {
  throw new QuotaRestrictedError('Movie Quota exceeded.');
} else if (requestBody.mediaType === MediaType.TV && quotas.tv.restricted) {
  throw new QuotaRestrictedError('Series Quota exceeded.');
}
```

TV 请求还额外检查剩余配额是否足够覆盖本次请求的季数（`server/entity/MediaRequest.ts:451-455`）：

```typescript
if (quotas.tv.limit && finalSeasons.length > (quotas.tv.remaining ?? 0)) {
  throw new QuotaRestrictedError('Series Quota exceeded.');
}
```

### 10.3 配额计算逻辑

**方法**：`User.getQuota()` → `server/entity/User.ts:273-372`

#### 电影配额计算

```typescript
// 统计时间窗口内非 DECLINED 的电影请求数
const movieQuotaUsed = await requestRepository.count({
  where: {
    requestedBy: { id: this.id },
    ...(movieQuotaDays ? { createdAt: AfterDate(movieDate) } : {}),
    type: MediaType.MOVIE,
    status: Not(MediaRequestStatus.DECLINED),
  },
});
```

**计数规则**：
- 仅统计非 DECLINED 状态的请求（PENDING / APPROVED / FAILED / COMPLETED 都算）
- `movieQuotaDays` 为空时统计全部历史，非空时仅统计最近 N 天

#### TV 配额计算

```typescript
// 统计时间窗口内非 DECLINED 的 TV 请求涉及的季数
const tvQuotaUsed = (
  await tvQuotaUsedQuery
    .addSelect((subQuery) => {
      return subQuery
        .select('COUNT(season.id)', 'seasonCount')
        .from(SeasonRequest, 'season')
        .leftJoin('season.request', 'parentRequest')
        .where('parentRequest.id = request.id');
    }, 'seasonCount')
    .getMany()
).reduce((sum, req) => sum + req.seasonCount, 0);
```

**计数规则**：
- 以季为计量单位（不是以请求为单位）
- 使用子查询统计每个请求关联的 `SeasonRequest` 数量
- 同样排除 DECLINED 状态

### 10.4 配额响应结构

```typescript
// GET /api/v1/user/:id/quota → QuotaResponse
{
  movie: {
    days: 7,           // 配额时间窗口（天）
    limit: 5,          // 配额上限（0 = 无限制）
    used: 3,           // 已使用量
    remaining: 2,      // 剩余量
    restricted: false  // 是否已超额
  },
  tv: {
    days: 7,
    limit: 10,
    used: 8,
    remaining: 2,
    restricted: false
  }
}
```

`restricted` 为 `true` 时触发 `QuotaRestrictedError`（HTTP 403）。

### 10.5 配额与请求状态的关系

```
用户提交请求
    │
    ├─ getQuota() 计算当前配额
    │
    ├─ restricted = true → QuotaRestrictedError (403) ← 阻断
    │
    ├─ TV: remaining < finalSeasons.length → QuotaRestrictedError (403) ← 阻断
    │
    └─ 通过配额检查 → 继续后续流程（去重、自动审批、推送 *arr）
```

**注意**：DECLINED 请求不计入配额，这意味着被拒绝的请求不占用配额空间；COMPLETED 请求仍计入配额（直到超出时间窗口）。

---

## 十一、自动批准规则的匹配链路

Jellyseerr 的"自动批准"由两部分组成：**权限驱动的自动审批** 和 **覆盖规则（OverrideRule）驱动的参数覆盖**。

### 11.1 权限驱动的自动审批

**核心逻辑**：用户提交请求时，若拥有特定权限，请求直接进入 `APPROVED` 状态，跳过管理员审批。

**权限体系**（`server/lib/permissions.ts`）：

| 权限 | 值 | 含义 |
|---|---|---|
| `AUTO_APPROVE` | 128 | 自动批准所有类型 |
| `AUTO_APPROVE_MOVIE` | 256 | 自动批准电影 |
| `AUTO_APPROVE_TV` | 512 | 自动批准 TV |
| `AUTO_APPROVE_4K` | 32768 | 自动批准所有 4K 类型 |
| `AUTO_APPROVE_4K_MOVIE` | 65536 | 自动批准 4K 电影 |
| `AUTO_APPROVE_4K_TV` | 131072 | 自动批准 4K TV |
| `MANAGE_REQUESTS` | 16 | 管理请求（隐含自动批准） |

**匹配链路**（`server/entity/MediaRequest.ts:354-367`）：

```
请求提交
    │
    ├─ is4k = false + mediaType = MOVIE
    │   └─ hasPermission([AUTO_APPROVE, AUTO_APPROVE_MOVIE, MANAGE_REQUESTS], 'or')
    │       → true: status = APPROVED, modifiedBy = user
    │       → false: status = PENDING
    │
    ├─ is4k = true + mediaType = MOVIE
    │   └─ hasPermission([AUTO_APPROVE_4K, AUTO_APPROVE_4K_MOVIE, MANAGE_REQUESTS], 'or')
    │
    ├─ is4k = false + mediaType = TV
    │   └─ hasPermission([AUTO_APPROVE, AUTO_APPROVE_TV, MANAGE_REQUESTS], 'or')
    │
    └─ is4k = true + mediaType = TV
        └─ hasPermission([AUTO_APPROVE_4K, AUTO_APPROVE_4K_TV, MANAGE_REQUESTS], 'or')
```

**权限匹配特性**：
- 使用 `type: 'or'` — 满足任一权限即可
- `ADMIN` 权限隐含所有权限（`hasPermission` 中 `value & Permission.ADMIN` 直接返回 true）
- 自动审批时 `modifiedBy = user`（非自动审批时 `modifiedBy = undefined`）

### 11.2 覆盖规则（OverrideRule）匹配链路

OverrideRule 不影响审批决策，但会覆盖请求推送到 *arr 时的参数（rootFolder、profileId、tags）。

**实体定义**（`server/entity/OverrideRule.ts`）：

| 字段 | 类型 | 含义 |
|---|---|---|
| `radarrServiceId` | int | 关联的 Radarr 服务 ID |
| `sonarrServiceId` | int | 关联的 Sonarr 服务 ID |
| `users` | string (逗号分隔) | 适用的用户 ID 列表 |
| `genre` | string (逗号分隔) | 匹配的类型 ID 列表 |
| `language` | string (\|分隔) | 匹配的原始语言代码 |
| `keywords` | string (逗号分隔) | 匹配的 TMDB 关键词 ID 列表 |
| `profileId` | int | 覆盖的质量配置 ID |
| `rootFolder` | string | 覆盖的根文件夹路径 |
| `tags` | string (逗号分隔) | 追加的标签 ID 列表 |

**匹配流程**（`server/entity/MediaRequest.ts:218-343`）：

```
请求提交（非管理员 / 非 REQUEST_ADVANCED 用户）
    │
    ├─ 确定默认 *arr 服务
    │   └─ is4k ? 找 4k 默认服务器 : 找非 4k 默认服务器
    │
    ├─ 查找匹配该服务的所有 OverrideRule
    │
    ├─ 逐条过滤规则（所有条件 AND 逻辑）
    │   ├─ users: 规则中的用户列表包含当前用户 ID
    │   ├─ genre: 规则中的类型 ID 与媒体类型有交集
    │   ├─ language: 规则中的语言代码与媒体原始语言匹配
    │   └─ keywords: 规则中的关键词 ID 与媒体关键词有交集
    │
    ├─ 特殊处理：动漫关键词
    │   └─ TV + 动漫关键词 + 规则不含动漫关键词 → 跳过该规则
    │
    ├─ 按"特异度"排序匹配的规则
    │   └─ specificity = [genre, language, keywords] 中非 null 的数量
    │   └─ 特异度最高的规则胜出
    │
    └─ 应用胜出规则的覆盖
        ├─ rootFolder → 覆盖
        ├─ profileId → 覆盖
        └─ tags → 合并（去重）
```

**关键代码**：

```typescript
// server/entity/MediaRequest.ts:311-321
const prioritizedRule = appliedOverrideRules.sort((a, b) => {
  const keys: (keyof OverrideRule)[] = ['genre', 'language', 'keywords'];
  const aSpecificity = keys.filter((key) => a[key] !== null).length;
  const bSpecificity = keys.filter((key) => b[key] !== null).length;
  return bSpecificity - aSpecificity;  // 高特异度优先
})[0];
```

**OverrideRule 不适用的场景**：
- 用户拥有 `MANAGE_REQUESTS` 权限 → `useOverrides = false`
- 此时用户可以自行指定 rootFolder / profileId / tags

### 11.3 自动请求（Auto Request）

Watchlist 同步和自动请求是另一种自动审批路径。

**权限**：`AUTO_REQUEST` / `AUTO_REQUEST_MOVIE` / `AUTO_REQUEST_TV`

**流程**（`server/lib/watchlistsync.ts`）：

```
Plex Watchlist 同步定时任务
    │
    ├─ 遍历所有有 Plex Token 的用户
    │
    ├─ 获取用户 Plex Watchlist
    │
    ├─ 过滤：跳过已拉黑、已可用、已有自动请求的媒体
    │
    └─ 对不可用媒体调用 MediaRequest.request({ isAutoRequest: true })
        │
        ├─ 走正常配额检查、去重检查
        ├─ 拥有 AUTO_APPROVE 权限 → 直接 APPROVED
        └─ 无 AUTO_APPROVE 权限 → PENDING（等待管理员审批）
```

**自动请求与普通请求的区别**：
- `isAutoRequest = true` 标记
- 自动请求的去重检查更严格：同一用户 + 同一媒体只能有一条非 DELETED 的自动请求
- Watchlist 同步中，`DuplicateMediaRequestError` / `QuotaRestrictedError` / `NoSeasonsAvailableError` 等异常被降级为 debug 日志，不打断同步流程

---

## 十二、请求统计与运维仪表盘

### 12.1 请求统计 API

**入口**：`GET /api/v1/request/count` → `server/routes/request.ts:338-426`

该接口返回按维度分类的请求数量，供前端仪表盘使用：

```typescript
return res.status(200).json({
  total: totalCount,         // 全部请求总数
  movie: movieCount,         // 电影请求数
  tv: tvCount,               // TV 请求数
  pending: pendingCount,     // PENDING 状态数
  approved: approvedCount,   // APPROVED 状态数
  declined: declinedCount,   // DECLINED 状态数
  processing: processingCount, // 处理中（APPROVED 且 Media 非 AVAILABLE）
  available: availableCount,   // 已可用（APPROVED 且 Media AVAILABLE）
  completed: completedCount,   // COMPLETED 状态数
});
```

**统计维度说明**：

| 维度 | 统计逻辑 | 含义 |
|---|---|---|
| `pending` | `request.status = PENDING` | 待审批 |
| `approved` | `request.status = APPROVED` | 已批准（含处理中和已可用） |
| `declined` | `request.status = DECLINED` | 已拒绝 |
| `processing` | `request.status = APPROVED AND media.status != AVAILABLE` | 已批准但媒体未到位 |
| `available` | `request.status = APPROVED AND media.status = AVAILABLE` | 已批准且媒体已到位（但请求尚未 COMPLETED） |
| `completed` | `request.status = COMPLETED` | 已完成 |

**processing 和 available 的区分**：这两个维度不是独立的请求状态，而是 `APPROVED` 状态的细分：

```typescript
// server/routes/request.ts:378-388
const processingCount = await query
  .where('request.status = :requestStatus', {
    requestStatus: MediaRequestStatus.APPROVED,
  })
  .andWhere(
    '((request.is4k = false AND media.status != :availableStatus) OR (request.is4k = true AND media.status4k != :availableStatus))',
    { availableStatus: MediaStatus.AVAILABLE }
  )
  .getCount();
```

### 12.2 请求列表查询 API

**入口**：`GET /api/v1/request` → `server/routes/request.ts:32-276`

支持多维筛选：

| 参数 | 值 | 作用 |
|---|---|---|
| `filter` | `pending` / `approved` / `processing` / `unavailable` / `available` / `completed` / `failed` / `deleted` | 按请求状态筛选 |
| `mediaType` | `movie` / `tv` / `all` | 按媒体类型筛选 |
| `requestedBy` | 用户 ID | 按请求人筛选 |
| `take` / `skip` | 分页参数 | 分页查询 |

**筛选条件与状态映射**：

| filter 值 | 请求状态 | 媒体状态 |
|---|---|---|
| `pending` | PENDING | — |
| `approved` / `processing` | APPROVED | — |
| `unavailable` | PENDING + APPROVED | UNKNOWN + PENDING + PROCESSING + PARTIALLY_AVAILABLE |
| `available` | COMPLETED | AVAILABLE |
| `completed` | COMPLETED | — |
| `failed` | FAILED | — |
| `deleted` | COMPLETED | DELETED |

### 12.3 用户配额查询 API

**入口**：`GET /api/v1/user/:id/quota` → `server/routes/user/index.ts:802-827`

需要 `MANAGE_USERS` 权限或本人查询。返回当前用户的配额使用情况，供前端显示配额进度条。

### 12.4 运维关注指标

基于以上 API，运维仪表盘通常关注以下指标：

| 指标 | 数据来源 | 告警阈值建议 |
|---|---|---|
| PENDING 请求堆积数 | `GET /request/count` → `pending` | > 50 |
| FAILED 请求数 | `GET /request/count` → `failed` (需额外查询) | > 0 |
| processing 停滞时间 | 请求 `createdAt` 与当前时间差 | > 24h |
| 用户配额命中率 | `GET /user/:id/quota` → `restricted` | 持续 true |
| *arr 推送失败率 | FAILED / (APPROVED + FAILED) | > 5% |
| 媒体同步延迟 | `lastSeasonChange` 与 `mediaAddedAt` 差值 | > 1h |
| 自动请求异常 | Watchlist Sync 日志中的 error 级别消息 | 任何 error |

### 12.5 请求优先级

Jellyseerr **不提供显式的请求优先级机制**。请求处理顺序遵循以下隐式规则：

1. **审批顺序**：PENDING 请求按 `createdAt` 排序，管理员手动选择批准顺序
2. **推送顺序**：`MediaRequestSubscriber` 在 `afterInsert` / `afterUpdate` 中同步推送，取决于数据库事件触发顺序
3. **OverrideRule 优先级**：当多条规则匹配时，按特异度排序选择（见第十一章），这是代码中唯一的"优先级"概念

**源码注释**（`server/entity/MediaRequest.ts:311-312`）：
```
// hacky way to prioritize rules
// TODO: make this better
```
表明当前的优先级排序是临时方案，未来可能改进。

---

## 十三、请求过期清理与归档策略

### 13.1 请求生命周期中的数据保留策略

Jellyseerr **不提供请求过期自动清理或归档机制**。所有请求记录从创建起永久保留在数据库中，除非被管理员手动删除。

**各终态请求的保留行为**：

| 请求终态 | 数据保留 | 对配额的影响 | 备注 |
|---|---|---|---|
| `COMPLETED` | 永久保留 | 计入配额（直到超出时间窗口） | 关联的 Media 状态可能因 AvailabilitySync 变为 `DELETED`，但请求记录不变 |
| `DECLINED` | 永久保留 | 不计入配额 | 可被相同媒体的后续新请求覆盖（去重检查跳过 DECLINED） |
| `FAILED` | 永久保留 | 计入配额 | 可通过 `/retry` 重新激活 |
| 已删除 | 物理删除 | 不再计入 | 删除后 Media 状态可能重置（见第七章） |

### 13.2 定时清理任务

Jellyseerr 的定时任务（`server/job/schedule.ts`）中，**没有任何任务负责清理请求记录**。所有定时清理任务均针对其他维度：

| 定时任务 | 清理对象 | 默认周期 | 定义位置 |
|---|---|---|---|
| `image-cache-cleanup` | TMDB 图片缓存 + 用户头像缓存 | 每天 05:00 | `server/job/schedule.ts:228-244` |
| `process-blocklisted-tags` | 基于标签的拉黑记录 + 关联 Media 记录 | 每 7 天 01:30 | `server/job/blocklistedTagsProcessor.ts:207-224` |
| `availability-sync` | 标记为 AVAILABLE 但实际不存在的 Media | 每天 05:00 | `server/lib/availabilitySync.ts` |
| `download-sync-reset` | 下载追踪器内存数据 | 每天 01:00 | `server/job/schedule.ts:213-225` |

### 13.3 BlocklistedTagsProcessor — 唯一的批量清理逻辑

`server/job/blocklistedTagsProcessor.ts` 是代码中唯一执行批量删除请求关联数据的组件：

```
BlocklistedTagsProcessor.run()
    │
    ├─ cleanBlocklist() — 事务内执行
    │   └─ 查找所有 blocklistedTags IS NOT NULL 的 Media
    │   └─ 批量删除（每批 500 条）
    │       └─ Media 删除 → 级联删除 Blocklist + 关联的 Season + MediaRequest
    │
    └─ createBlocklistEntries() — 事务内执行
        └─ 遍历 blocklistedTags 配置中的关键词
        └─ 调用 TMDB Discover API 搜索匹配的媒体
        └─ 将结果批量加入 Blocklist
```

**注意**：整个流程在一个数据库事务中执行。如果 `createBlocklistEntries` 阶段因 `AbortTransaction` 中断（`this.running = false`），事务会回滚，`cleanBlocklist` 的删除也不生效。

### 13.4 Session 过期清理

Session 实体（`server/entity/Session.ts`）是唯一有过期字段的实体：

```typescript
public expiredAt = Date.now();
```

但 Session 清理不通过定时任务，而是通过 TypeORM 的 `synchronize` 机制或手动清理。Session 的过期与请求生命周期无关。

### 13.5 运维建议

由于 Jellyseerr 不内置请求归档功能，生产环境中建议：

| 场景 | 建议 |
|---|---|
| 请求表过大 | 定期手动导出 + 归档 COMPLETED/DECLINED 请求，然后通过 API 或 SQL 删除 |
| 配额计算变慢 | 配额查询（`User.getQuota()`）使用 `COUNT` 查询，大表需添加 `createdAt` 索引 |
| 数据库膨胀 | SQLite 场景下定期 `VACUUM`；PostgreSQL 场景下定期 `VACUUM ANALYZE` |
| 历史审计 | 通过 `GET /api/v1/request` 分页导出，外部归档 |

---

## 十四、用户请求历史导出

### 14.1 API 层面的导出能力

Jellyseerr **不提供专用的请求导出 API**（无 CSV/JSON 批量下载端点）。但可以通过现有的请求列表 API 实现分页导出：

**可用接口**：

| 接口 | 方法 | 用途 |
|---|---|---|
| `GET /api/v1/request` | 分页查询 | 按状态/类型/用户筛选，返回完整请求对象 |
| `GET /api/v1/request/count` | 聚合统计 | 返回各状态计数 |
| `GET /api/v1/user/:id/requests` | 用户维度 | 查询特定用户的请求列表 |

**分页查询参数**（`server/routes/request.ts:32-276`）：

```
GET /api/v1/request?take=20&skip=0&filter=all&mediaType=all&sort=added
```

| 参数 | 默认值 | 说明 |
|---|---|---|
| `take` | 20 | 每页条数 |
| `skip` | 0 | 偏移量 |
| `filter` | all | 状态筛选 |
| `mediaType` | all | 媒体类型 |
| `requestedBy` | — | 按用户筛选 |
| `sort` | added | 排序方式 |

**返回的请求对象包含以下可导出字段**：

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | int | 请求 ID |
| `status` | int (enum) | 请求状态 |
| `media.mediaType` | string | 媒体类型 (movie/tv) |
| `media.tmdbId` | int | TMDB ID |
| `media.tvdbId` | int | TVDB ID (TV) |
| `requestedBy.id` | int | 请求用户 ID |
| `requestedBy.displayName` | string | 请求用户名 |
| `createdAt` | datetime | 创建时间 |
| `updatedAt` | datetime | 更新时间 |
| `modifiedBy` | int | 最后修改人 |
| `is4k` | boolean | 是否 4K |
| `isAutoRequest` | boolean | 是否自动请求 |
| `seasons` | array | 关联季信息 (TV) |

### 14.2 数据库层面的导出

对于大规模导出，可直接操作数据库：

**SQLite**：
```bash
sqlite3 config/db/jellyseerr.db \
  "SELECT id, status, mediaType, tmdbId, requestedBy, createdAt, updatedAt FROM media_request;" \
  -csv -header > requests_export.csv
```

**PostgreSQL**：
```sql
COPY (
  SELECT r.id, r.status, m.media_type, m.tmdb_id, u.username, r.created_at, r.updated_at
  FROM media_request r
  JOIN media m ON r.media_id = m.id
  JOIN "user" u ON r.requested_by_id = u.id
) TO STDOUT WITH CSV HEADER;
```

### 14.3 导出数据的状态映射

导出时需将数值状态码映射为可读文本：

| MediaRequestStatus 值 | 文本 | MediaStatus 值 | 文本 |
|---|---|---|---|
| 1 | PENDING | 1 | UNKNOWN |
| 2 | APPROVED | 2 | PENDING |
| 3 | DECLINED | 3 | PROCESSING |
| 4 | FAILED | 4 | PARTIALLY_AVAILABLE |
| 5 | COMPLETED | 5 | AVAILABLE |
| — | — | 6 | BLOCKLISTED |
| — | — | 7 | DELETED |

---

## 十五、请求与媒体库同步状态的一致性核对

一致性核对是 Jellyseerr 保证"请求状态"与"实际媒体可用性"不漂移的核心机制。通过多层定时任务实现：

### 15.1 一致性核对体系总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                     一致性核对体系                                    │
│                                                                     │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────────────────┐ │
│  │ 媒体服务器扫描 │   │ *arr 扫描    │   │ AvailabilitySync        │ │
│  │ (正向同步)    │   │ (正向同步)    │   │ (反向核对)               │ │
│  │             │   │              │   │                          │ │
│  │ Plex Recent │   │ Radarr Scan  │   │ 检查 AVAILABLE 媒体      │ │
│  │ Plex Full   │   │ Sonarr Scan  │   │ 是否仍然存在              │ │
│  │ Jellyfin    │   │              │   │                          │ │
│  │ Recent/Full │   │              │   │ 不存在 → DELETED          │ │
│  └─────────────┘   └──────────────┘   └──────────────────────────┘ │
│        │                  │                     │                   │
│        ▼                  ▼                     ▼                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Media / Season 状态                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│        │                                                           │
│        ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              MediaSubscriber (状态变更驱动器)                   │  │
│  │     Media AVAILABLE → 关联请求 COMPLETED                       │  │
│  │     Media PENDING → 关联 PENDING 请求 APPROVED                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 15.2 正向同步：媒体服务器扫描

媒体服务器扫描将 Plex/Jellyfin 的实际库状态同步到 `Media` / `Season` 实体。

**Plex 扫描**：

| 任务 | 默认周期 | 扫描范围 | 定义位置 |
|---|---|---|---|
| `plex-recently-added-scan` | 每 5 分钟 | 最近新增的媒体 | `server/lib/scanners/plex/` |
| `plex-full-scan` | 每天 03:00 | 全量库 | `server/lib/scanners/plex/` |

**Jellyfin 扫描**：

| 任务 | 默认周期 | 扫描范围 |
|---|---|---|
| `jellyfin-recently-added-scan` | 每 5 分钟 | 最近新增的媒体 |
| `jellyfin-full-scan` | 每天 03:00 | 全量库 |

**状态更新逻辑**（`BaseScanner.processMovie()` → `server/lib/scanners/baseScanner.ts:95-258`）：

```
扫描到电影（hasFile + 非processing）
    └─ Media.status = AVAILABLE
    └─ 触发 MediaSubscriber → 关联请求 COMPLETED

扫描到电影（processing + 无文件）
    └─ Media.status = PROCESSING（仅从 DELETED 以外的状态升级）

未扫描到的已有记录
    └─ 状态不变（交由 AvailabilitySync 处理）
```

**TV 季级别更新逻辑**（`BaseScanner.processShow()` → `server/lib/scanners/baseScanner.ts:270-450`）：

```
季的可用集数 == 总集数 且 > 0  → Season.status = AVAILABLE
季的可用集数 > 0 但 < 总集数    → Season.status = PARTIALLY_AVAILABLE
季标记为 processing 且无可用集   → Season.status = PROCESSING（不从 DELETED 升级）
季非 processing 且无可用集
  且原状态为 PROCESSING          → Season.status = UNKNOWN
其他情况                         → 保持原状态
```

**关键防护**：已为 `AVAILABLE` 的季不会被降级（`existingSeason.status === MediaStatus.AVAILABLE` 强制保持），避免多个扫描器竞争导致状态抖动。

### 15.3 正向同步：*arr 扫描

Radarr/Sonarr 扫描将 *arr 的实际媒体记录同步到 `Media` 实体。

**Radarr 扫描**（`server/lib/scanners/radarr/index.ts`）：

| 任务 | 默认周期 |
|---|---|
| `radarr-scan` | 每天 04:00 |

**扫描逻辑**：

```
遍历所有 Radarr 服务器（syncEnabled = true）
    │
    ├─ 获取所有 movie 列表
    │
    ├─ 逐条 processRadarrMovie()
    │   └─ hasFile + monitored → 填入 hasFile: true, processing: false
    │   └─ !hasFile + monitored → 填入 hasFile: false, processing: true
    │   └─ !monitored → 填入 processing: false
    │
    └─ cleanupOrphanedMovies() — 扫描后清理孤儿记录
```

**孤儿清理逻辑**（`server/lib/scanners/radarr/index.ts:144-193`）：

```
所有标准 Radarr 服务器均已扫描（syncEnabled = true）
    │
    ├─ 查找 status = PROCESSING 的电影
    │
    └─ 若 tmdbId 不在本次扫描结果中
        └─ Media.status = UNKNOWN
        └─ 日志："Movie {tmdbId} not found in any Radarr server"
```

**安全条件**：如果任何一个同类型（标准/4K）服务器未启用同步，则**跳过孤儿清理**，避免误删存在于未扫描服务器上的媒体：

```typescript
// server/lib/scanners/radarr/index.ts:95-107
const allStandardScanned = this.servers
  .filter((s) => !this.enable4kMovie || !s.is4k)
  .every((s) => s.syncEnabled);
if (!allStandardScanned) {
  this.didScanStandard = false; // 标记为未完成全量扫描，跳过孤儿清理
}
```

**Sonarr 扫描**（`server/lib/scanners/sonarr/index.ts`）：逻辑与 Radarr 对称，额外处理：
- 通过 TVDB ID 匹配（非 TMDB ID）
- 孤儿清理时同时重置关联 `Season.status` 为 `UNKNOWN`

### 15.4 反向核对：AvailabilitySync

AvailabilitySync 是唯一执行**反向核对**的定时任务，检查标记为 AVAILABLE 的媒体是否仍然实际存在。

**任务配置**：

| 属性 | 值 |
|---|---|
| Job ID | `availability-sync` |
| 默认周期 | 每天 05:00 |
| 定义位置 | `server/lib/availabilitySync.ts` |

**核对流程**：

```
遍历所有 status IN (AVAILABLE, PARTIALLY_AVAILABLE) 的 Media
    │
    ├─ 电影核对
    │   ├─ 检查媒体服务器（Plex/Jellyfin）
    │   ├─ 检查 *arr（Radarr/Sonarr）
    │   │
    │   ├─ 任一来源确认存在 → 保留 AVAILABLE
    │   └─ 全部来源确认不存在 → mediaUpdater()
    │       ├─ Media.status = DELETED
    │       ├─ 若有 APPROVED 请求正在处理 → 保留 externalServiceId 等字段
    │       └─ 若无处理中请求 → 清空所有外部关联字段
    │
    └─ TV 季核对
        ├─ 交叉比对：媒体服务器季数据 + Sonarr 季数据 → finalSeasons Map
        ├─ 对比 TMDB 的集数信息补充缺失的季
        └─ seasonUpdater()
            ├─ 不存在的季 → Season.status = DELETED
            └─ 若部分季不存在 → Media.status 降级为 PARTIALLY_AVAILABLE
```

**容错策略**：

| 异常场景 | 处理方式 | 原因 |
|---|---|---|
| *arr API 返回 404 | 视为不存在 | 媒体已被从 *arr 中删除 |
| *arr API 返回非 404 错误 | **视为存在** | 避免网络抖动导致误删 |
| Plex/Jellyfin API 返回 404 | 视为不存在 | 媒体已从媒体服务器删除 |
| Plex/Jellyfin API 返回非 404 错误 | **视为存在** + 阻止季级别搜索 | 保守策略，宁可漏删不可误删 |
| 获取季集数失败 | 假设季存在 | `seasonExistsInJellyfin/Plex` catch 块返回 `true` |

**处理中请求保护**（`server/lib/availabilitySync.ts:506-543`）：

```typescript
// TV 类型：若有 APPROVED 请求正在处理，保留外部服务关联字段
if (media.mediaType === 'tv') {
  const request = await requestRepository
    .createQueryBuilder('request')
    .where('media.id = :id', { id: media.id })
    .andWhere('request.is4k = :is4k AND request.status = :requestStatus', {
      requestStatus: MediaRequestStatus.APPROVED,
      is4k: is4k,
    })
    .getOne();
  if (request) isMediaProcessing = true;
}

// 处理中的请求保留 serviceId、externalServiceId 等字段
media.serviceId = isMediaProcessing ? media.serviceId : null;
media.externalServiceId = isMediaProcessing ? media.externalServiceId : null;
```

### 15.5 一致性核对的完整时间线

```
00:00  ┌──────────────────────────┐
       │  Plex/Jellyfin Recent    │ ← 每 5 分钟：增量同步媒体服务器
       │  (*/5 * * * * *)         │
       ├──────────────────────────┤
03:00  │  Plex/Jellyfin Full      │ ← 每天：全量同步媒体服务器
       │  (0 0 3 * * *)          │
       ├──────────────────────────┤
04:00  │  Radarr Scan             │ ← 每天：全量同步 Radarr + 孤儿清理
       │  (0 0 4 * * *)          │
04:30  │  Sonarr Scan             │ ← 每天：全量同步 Sonarr + 孤儿清理
       │  (0 30 4 * * *)         │
       ├──────────────────────────┤
05:00  │  AvailabilitySync        │ ← 每天：反向核对 AVAILABLE 媒体
       │  (0 0 5 * * *)          │
       │  Image Cache Cleanup     │
       ├──────────────────────────┤
01:00  │  Download Sync Reset     │ ← 每天：重置下载追踪器
       ├──────────────────────────┤
01:30  │  Blocklisted Tags        │ ← 每 7 天：拉黑标签处理 + 批量清理
       │  (0 30 1 */7 * *)       │
       └──────────────────────────┘
```

### 15.6 潜在的一致性漂移场景

| 场景 | 漂移方向 | 检测机制 | 修复时间 |
|---|---|---|---|
| 媒体在 Plex 中删除但 *arr 仍存在 | 无漂移（*arr 扫描确认存在） | Radarr/Sonarr Scan | ≤ 24h |
| 媒体从 *arr 删除但 Plex 仍存在 | 无漂移（Plex 扫描确认存在） | Plex Recent Scan | ≤ 5min |
| 媒体从所有来源删除 | Media 仍标记为 AVAILABLE | AvailabilitySync | ≤ 24h |
| 请求 APPROVED 但 *arr 未收到推送 | 请求 APPROVED + Media PROCESSING | 无自动检测 | 需手动 retry |
| Media AVAILABLE 但请求未 COMPLETED | 请求卡在 APPROVED | MediaSubscriber 需 Media 实体变更触发 | 需手动触发 Media 更新 |
| 季部分集缺失 | Season 仍标记为 AVAILABLE | AvailabilitySync 季级别核对 | ≤ 24h |
| *arr 服务器 syncEnabled=false | 孤儿清理被跳过 | 日志提示 "Skipping orphaned cleanup" | 不修复（设计预期） |


