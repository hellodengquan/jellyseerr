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
| `server/entity/MediaRequest.ts` | 请求实体 + 静态创建方法 `request()` + `@AfterInsert`/`@AfterUpdate` 通知钩子 |
| `server/entity/SeasonRequest.ts` | 季请求实体 |
| `server/routes/request.ts` | REST API：创建/审批/拒绝/重试/删除请求 |
| `server/subscriber/MediaRequestSubscriber.ts` | 请求实体事件订阅器：推送到 *arr + 更新父 Media 状态 |
| `server/subscriber/MediaSubscriber.ts` | 媒体实体事件订阅器：反向驱动请求状态 → COMPLETED |
| `server/lib/availabilitySync.ts` | 定时同步：校验媒体是否仍存在于媒体服务器 |

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
