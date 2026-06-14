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

