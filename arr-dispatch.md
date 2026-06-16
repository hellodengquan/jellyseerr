# Jellyseerr 请求分流至 Sonarr/Radarr 代码链路分析

## 一、整体架构概览

```
用户审批请求 (API PUT /:requestId/approve)
        │
        ▼
  MediaRequest.status = APPROVED  (保存至 DB)
        │
        ▼
  TypeORM EntitySubscriber 触发 afterUpdate / afterInsert
        │
        ├─► sendToRadarr()  ── 电影类请求
        │
        └─► sendToSonarr()  ── 剧集类请求
                    │
                    ▼
            异步调用 *arr API
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
    成功：写回 Media         失败：标记 FAILED
    (externalServiceId)     + 发送失败通知
```

核心文件：
- `server/routes/request.ts` — 请求审批 API 入口
- `server/entity/MediaRequest.ts` — 请求实体，含静态 `request()` 工厂方法
- `server/subscriber/MediaRequestSubscriber.ts` — **分流逻辑的核心所在地**
- `server/api/servarr/base.ts` — ServarrBase 基类（通用能力）
- `server/api/servarr/radarr.ts` — Radarr API 封装
- `server/api/servarr/sonarr.ts` — Sonarr API 封装
- `server/entity/Media.ts` — 媒体实体（父级状态）
- `server/constants/media.ts` — 状态枚举定义

---

## 二、审批通过后的请求分流逻辑

### 2.1 触发入口

审批通过有三条路径，最终都会落到 `MediaRequest.status = APPROVED` 并触发 TypeORM 生命周期钩子：

| 路径 | 代码位置 | 说明 |
|---|---|---|
| 手动审批 | `server/routes/request.ts:663-705` | `POST /:requestId/approve`，管理员操作 |
| 自动审批 | `server/entity/MediaRequest.ts:354-367` | 创建请求时若用户有 AUTO_APPROVE 权限，直接设为 APPROVED |
| 重试失败 | `server/routes/request.ts:633-661` | `POST /:requestId/retry`，将 FAILED 重置为 APPROVED |

### 2.2 分流判断条件（TypeORM Subscriber）

`server/subscriber/MediaRequestSubscriber.ts` 通过 `@EventSubscriber()` 注册监听 `MediaRequest` 实体：

- `afterInsert(event)` — 新建请求入库后触发（覆盖自动审批场景）
- `afterUpdate(event)` — 请求更新后触发（覆盖手动审批、重试场景）

两个钩子内部**都会同时调用**：

```typescript
await this.sendToRadarr(event.entity as MediaRequest);
await this.sendToSonarr(event.entity as MediaRequest);
```

而在每个方法内部做二次过滤：

**sendToRadarr 入口守卫** (`server/subscriber/MediaRequestSubscriber.ts:183-187`)：
```typescript
if (
  entity.status === MediaRequestStatus.APPROVED &&
  entity.type === MediaType.MOVIE
) { ... }
```

**sendToSonarr 入口守卫** (`server/subscriber/MediaRequestSubscriber.ts:477-481`)：
```typescript
if (
  entity.status === MediaRequestStatus.APPROVED &&
  entity.type === MediaType.TV
) { ... }
```

即：分流的核心判断条件是 `entity.type`（`movie` vs `tv`），两条方法总是被同时调用但内部互斥。

### 2.4 entity.type 守卫的异常类型与静默失败

守卫逻辑是**纯条件判断，不抛出任何异常**，不匹配时直接 `return` 静默退出。这意味着：

| 异常场景 | 行为 | 代码位置 |
|---|---|---|
| `entity.type` 既非 `movie` 也非 `tv`（如 DB 脏数据） | 两个守卫都不匹配，请求**永远停留在 APPROVED 状态** | Radarr:183-187, Sonarr:477-481 |
| `entity.type` 正确但 status 不是 APPROVED | 守卫不匹配，无动作 | 同上 |
| TypeORM 钩子触发时 entity 部分字段为 undefined | `entity.type` 为 undefined 时守卫不匹配 | 同上 |

**风险**：若 DB 中存在 `MediaRequest.type` 非法值（如被手动修改），请求会卡在 APPROVED 状态且无任何日志。需通过 `POST /:requestId/retry` 手动触发重试（重置为 APPROVED 会再次触发钩子）。

---

## 2.5 字段映射的部分覆盖机制

字段映射采用 **"请求级非空且不等于默认则覆盖"** 的策略，而非全量覆盖。以 Radarr 为例 (`server/subscriber/MediaRequestSubscriber.ts:241-270`)：

```typescript
// 初始值 = 服务器默认配置
let rootFolder = radarrSettings.activeDirectory;
let qualityProfile = radarrSettings.activeProfileId;
let tags = radarrSettings.tags ? [...radarrSettings.tags] : [];

// 仅当请求级字段非空且与默认不同时才覆盖
if (entity.rootFolder && entity.rootFolder !== ''
    && entity.rootFolder !== radarrSettings.activeDirectory) {
  rootFolder = entity.rootFolder;
}

if (entity.profileId && entity.profileId !== radarrSettings.activeProfileId) {
  qualityProfile = entity.profileId;
}

if (entity.tags && entity.tags.length > 0
    && JSON.stringify(entity.tags.sort()) !== JSON.stringify(radarrSettings.tags?.sort() || [])) {
  tags = entity.tags;
}
```

Sonarr 遵循同样模式，但额外增加动漫分支判断 (`server/subscriber/MediaRequestSubscriber.ts:586-701`)：

```typescript
let qualityProfile = seriesType === 'anime' && sonarrSettings.activeAnimeProfileId
  ? sonarrSettings.activeAnimeProfileId
  : sonarrSettings.activeProfileId;
// 然后再检查 entity.profileId 是否覆盖
```

**覆盖优先级**（从高到低）：
1. `entity.profileId` / `entity.rootFolder` / `entity.tags`（请求级，非空且不同时生效）
2. `activeAnimeProfileId` / `activeAnimeDirectory` / `animeTags`（动漫专属，仅 seriesType=anime 时）
3. `activeProfileId` / `activeDirectory` / `tags`（服务器默认）

---

## 2.6 ANIME_KEYWORD_ID 的多关键词识别

### 2.6.1 单关键词匹配机制

`ANIME_KEYWORD_ID = 210024` 定义于 `server/api/themoviedb/constants.ts:1`。识别逻辑使用 `Array.some()` 匹配 TMDB 返回的关键词列表：

```typescript
// server/subscriber/MediaRequestSubscriber.ts:579-584
if (series.keywords.results.some(keyword => keyword.id === ANIME_KEYWORD_ID)) {
  seriesType = sonarrSettings.animeSeriesType ?? 'anime';
}
```

这是**单关键词精确匹配**，而非多关键词组合。只要任一关键词 id 等于 210024，即判定为动漫。

### 2.6.2 TMDB API 返回格式兼容

TMDB 有两种不同的关键词返回结构，代码在 `OverrideRule` 匹配逻辑中做了兼容处理 (`server/entity/MediaRequest.ts:294-298`)：

```typescript
if ('keywords' in tmdbMedia.keywords) {
  keywordList = tmdbMedia.keywords.keywords;    // 格式 A: { keywords: [...] }
} else if ('results' in tmdbMedia.keywords) {
  keywordList = tmdbMedia.keywords.results;     // 格式 B: { results: [...] }
}
```

这是因为 TMDB 的 `/movie/{id}/keywords` 和 `/tv/{id}/keywords` 接口返回格式不同。

### 2.6.3 OverrideRule 中的动漫关键词排除

在 `MediaRequest.request()` 工厂方法中 (`server/entity/MediaRequest.ts:244-259`)，动漫类型会**排除普通 OverrideRule**，除非规则显式包含动漫关键词：

```typescript
if (requestBody.mediaType === MediaType.TV && hasAnimeKeyword
    && (!rule.keywords || !rule.keywords.split(',').map(Number).includes(ANIME_KEYWORD_ID))) {
  return false;  // 跳过不包含动漫关键词的规则
}
```

即：动漫媒体只能匹配**显式包含 210024 关键词 ID** 的 OverrideRule。

### 2.3 服务器实例选择

请求可以指定 `serverId` 覆盖默认服务器，选择逻辑如下：

**Radarr 服务器选择** (`server/subscriber/MediaRequestSubscriber.ts:203-239`)：
1. 从 `settings.radarr[]` 中找 `isDefault === true && is4k === entity.is4k` 的服务器作为默认
2. 若 `entity.serverId !== null && entity.serverId >= 0` 且与默认不同，则按 `serverId` 精确查找覆盖
3. 找不到则打 warn 日志并 return，请求停留在 APPROVED 状态

**Sonarr 服务器选择** (`server/subscriber/MediaRequestSubscriber.ts:497-533`)：逻辑同上，从 `settings.sonarr[]` 中选择。

---

## 三、字段映射关系

### 3.1 MediaRequest → RadarrMovieOptions

映射代码位于 `server/subscriber/MediaRequestSubscriber.ts:363-374`。

| Jellyseerr 字段来源 | Radarr API 字段 | 优先级 / 说明 |
|---|---|---|
| `entity.profileId` 或 `radarrSettings.activeProfileId` | `qualityProfileId`, `profileId` | 请求级 profileId 非空且与默认不同则覆盖 |
| `entity.rootFolder` 或 `radarrSettings.activeDirectory` | `rootFolderPath` | 同上 |
| `entity.tags` 或 `radarrSettings.tags` | `tags` | 同上 |
| `radarrSettings.minimumAvailability` | `minimumAvailability` | 仅来自服务器配置（如 `announced`, `inCinemas`, `released`） |
| TMDB `movie.title` | `title` | |
| TMDB `movie.id` | `tmdbId` | |
| TMDB `movie.release_date.slice(0, 4)` 转 Number | `year` | |
| 常量 `true` | `monitored` | |
| `!radarrSettings.preventSearch` | `searchNow` | |
| `tagRequests` 开启时自动创建的用户 tag | 追加到 `tags` | 标签格式 `{userId}-{sanitizedDisplayName}`，若不存在则调用 `radarr.createTag()` |

RadarrMovieOptions 完整定义见 `server/api/servarr/radarr.ts:4-15`。

### 3.2 MediaRequest → AddSeriesOptions (Sonarr)

映射代码位于 `server/subscriber/MediaRequestSubscriber.ts:703-716`。

| Jellyseerr 字段来源 | Sonarr API 字段 | 优先级 / 说明 |
|---|---|---|
| `entity.profileId` 或 `activeProfileId` / `activeAnimeProfileId` | `profileId` | 动漫类型走 `activeAnime*` 配置 |
| `entity.languageProfileId` 或 `activeLanguageProfileId` / `activeAnimeLanguageProfileId` | `languageProfileId` | 同上 |
| `entity.rootFolder` 或 `activeDirectory` / `activeAnimeDirectory` | `rootFolderPath` | 同上 |
| `entity.tags` 或 `sonarrSettings.tags` / `animeTags` | `tags` | 同上 |
| TMDB `series.name` | `title` | |
| TMDB `external_ids.tvdb_id` 或 `media.tvdbId` | `tvdbid` | 若两者皆空则删除 media 和 request 并抛出 |
| `entity.seasons.map(s => s.seasonNumber)` | `seasons: number[]` | 季号数组 |
| `sonarrSettings.enableSeasonFolders` | `seasonFolder: boolean` | |
| TMDB 含 `ANIME_KEYWORD_ID (210024)` → `animeSeriesType ?? 'anime'`，否则 `'standard'` | `seriesType` | `'standard' \| 'daily' \| 'anime'` |
| 常量 `true` | `monitored` | |
| `sonarrSettings.monitorNewItems` | `monitorNewItems` | `'all' \| 'none'` |
| `!sonarrSettings.preventSearch` | `searchNow` | |
| `tagRequests` 开启时自动创建的用户 tag | 追加到 `tags` | 同 Radarr 规则 |

AddSeriesOptions 完整定义见 `server/api/servarr/sonarr.ts:91-104`。

### 3.3 动漫类型的特殊分支

Sonarr 对动漫有一整套独立配置链路：

```
TMDB series.keywords.results
    │ 是否包含 id === 210024 (ANIME_KEYWORD_ID)
    ▼
seriesType = sonarrSettings.animeSeriesType ?? 'anime'
    │
    ├─► activeAnimeProfileId        (替代 activeProfileId)
    ├─► activeAnimeDirectory        (替代 activeDirectory)
    ├─► activeAnimeLanguageProfileId (替代 activeLanguageProfileId)
    └─► animeTags                   (替代 tags)
```

动漫关键词 ID 常量定义在 `server/api/themoviedb/constants.ts`。

---

## 四、错误回退流程

错误处理分两层：**同步 try/catch**（调用前检查阶段）和 **异步 Promise.catch**（API 调用阶段）。

### 4.1 同步 try/catch 阶段

发生在 `sendToRadarr` / `sendToSonarr` 的主函数 try 块内：

| 场景 | 处理方式 | 代码位置 |
|---|---|---|
| 无 *arr 服务器配置（数组为空） | info/warn 日志 → return，状态不变 | Radarr:191-201, Sonarr:485-495 |
| 找不到默认（或指定 serverId 的）服务器 | warn 日志 → return，状态不变 | Radarr:225-239, Sonarr:519-533 |
| Media 实体不存在 | error 日志 → return | Radarr:294-301, Sonarr:539-541 |
| TVDB ID 缺失（仅 Sonarr） | 删除 media 和 request → throw Error | Sonarr:569-574 |
| 其他异常（API 连接错误、配置错误等） | `entity.status = FAILED` → 保存 → 发送 `MEDIA_FAILED` 通知 | Radarr:446-473, Sonarr:789-817 |

### 4.1.1 profileId missing fallback

当 `entity.profileId` 为 **0 / null / undefined / 空字符串** 时，会自动 fallback 到服务器默认配置。

**Radarr fallback 逻辑** (`server/subscriber/MediaRequestSubscriber.ts:242,258-270`)：
```typescript
// 初始值 = 服务器默认
let qualityProfile = radarrSettings.activeProfileId;

// 仅当 entity.profileId 非空且与默认不同时才覆盖
if (entity.profileId && entity.profileId !== radarrSettings.activeProfileId) {
  qualityProfile = entity.profileId;
}
```

利用 JavaScript 真值判断，`entity.profileId` 为 `0 | null | undefined | ''` 时条件为 false，直接使用 `radarrSettings.activeProfileId` 作为 fallback。

**Sonarr fallback 逻辑** (`server/subscriber/MediaRequestSubscriber.ts:591-622`)：
```typescript
// 第一步：根据 seriesType 选择动漫或普通默认
let qualityProfile = seriesType === 'anime' && sonarrSettings.activeAnimeProfileId
  ? sonarrSettings.activeAnimeProfileId
  : sonarrSettings.activeProfileId;

// 第二步：请求级覆盖（同样带真值检查）
if (entity.profileId && entity.profileId !== qualityProfile) {
  qualityProfile = entity.profileId;
}
```

**风险**：若 `radarrSettings.activeProfileId` 本身为 0（未正确配置），则会向 *arr API 发送 `profileId: 0`，导致 400 Bad Request 错误。

### 4.2 异步 Promise.catch 阶段

`radarr.addMovie()` 和 `sonarr.addSeries()` 均为**异步 fire-and-forget** 调用（不 await），错误在 `.catch()` 中处理：

```typescript
radarr
  .addMovie(radarrMovieOptions)
  .then(/* 成功写回 Media */)
  .catch(async () => {
    // 1. 将请求标记为 FAILED（幂等判断）
    if (entity.status !== MediaRequestStatus.FAILED) {
      entity.status = MediaRequestStatus.FAILED;
      await requestRepository.save(entity);
    }
    // 2. 发送失败通知
    MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
  })
  .finally(() => {
    // 3. 清除相关 API 缓存
    radarr.clearCache({ tmdbId, externalId });
  });
```

Sonarr 的 catch 逻辑完全对称，见 `server/subscriber/MediaRequestSubscriber.ts:740-783`。

关键细节：
- `.catch()` 内部再包一层 try/catch 防止保存 FAILED 状态时再次报错
- 使用 `entity.status !== MediaRequestStatus.FAILED` 判断保证幂等，避免重复写入

### 4.2.1 fire-and-forget 失败的补偿回滚

由于 API 调用是 fire-and-forget 模式，**没有数据库事务回滚**。失败时仅做以下补偿：

1. **状态标记**：将 `MediaRequest.status` 设为 `FAILED`（幂等）
2. **通知发送**：调用 `MediaRequest.sendNotification()` 发送失败通知
3. **缓存清理**：在 `.finally()` 中调用 `radarr.clearCache()` / `sonarr.clearCache()`，避免缓存脏数据
4. **无数据回滚**：
   - 已写入的 `media.status = PROCESSING` **不会回滚**
   - 若 *arr API 部分成功（如条目已创建但监控失败），`externalServiceId` **不会被清理**

这种设计意味着：
- `Media.status = PROCESSING` 但 `MediaRequest.status = FAILED` 是合法状态组合
- 重试（POST /retry）会重新尝试整个流程，*arr API 内部的幂等逻辑（先查后更）会处理重复请求

### 4.3 同步异步双层错误的重试策略

**重试入口**：`POST /request/:requestId/retry` (`server/routes/request.ts:633-661`)

```typescript
mediaRequest.status = MediaRequestStatus.APPROVED;
mediaRequest.modifiedBy = user;
await requestRepository.save(mediaRequest);
```

重试的本质是**将 FAILED 重置为 APPROVED**，触发 TypeORM `afterUpdate` 钩子，重新走完整的分流流程。

**双层错误的重试差异**：

| 错误类型 | 重试时是否重新执行 | 说明 |
|---|---|---|
| 同步检查错误（无服务器、无 Media 等） | 是 | 每次重试都会重新检查前置条件 |
| TVDB ID 缺失错误 | 否 | 会删除 media 和 request，无法重试 |
| 异步 API 调用错误 | 是 | 从服务器选择开始完全重新执行 |
| *arr API 内部幂等逻辑 | 自动处理 | 已存在的条目会走更新路径而非创建 |

**重试次数限制**：代码中**无重试次数限制**，用户可无限次点击重试。

### 4.4 Radarr/Sonarr API 内部容错

`server/api/servarr/radarr.ts:118-249` 的 `addMovie` 方法存在分级容错：

1. **已存在且 hasFile=true**：直接返回成功（不重复添加）
2. **已存在但 id 存在且未 monitored**：PUT 更新为 monitored，按需触发 search
3. **已存在且已 monitored**：直接返回，按需触发 search
4. **全新条目**：POST `/movie` 创建
5. **全部失败**：throw Error 交由上层 catch 处理

Sonarr `addSeries` (`server/api/servarr/sonarr.ts:191-313`) 遵循同样模式：
- 已存在则 PUT 更新 seasons monitored 状态，并重新监控未监控的 episode
- 不存在则 POST 创建

### 4.4.1 searchNow 触发逻辑

`searchNow` 字段由 `!radarrSettings.preventSearch` / `!sonarrSettings.preventSearch` 决定，在 *arr API 内部的三个时机触发：

1. **新建条目时**：通过 `addOptions.searchForMovie` / `addOptions.searchForMissingEpisodes` 传递
2. **更新已存在未监控条目时**：PUT 更新后单独调用 `searchMovie()` / `searchSeries()`
3. **已存在已监控但无文件时**：单独调用 `searchMovie()` / `searchSeries()`

`searchMovie()` 和 `searchSeries()` 本身也是 fire-and-forget 调用（无 await）。

---

## 五、状态写回流程

### 5.1 状态枚举回顾

`server/constants/media.ts`：

```typescript
enum MediaRequestStatus {
  PENDING = 1,
  APPROVED,     // 2
  DECLINED,     // 3
  FAILED,       // 4
  COMPLETED,    // 5
}

enum MediaStatus {
  UNKNOWN = 1,
  PENDING,           // 2
  PROCESSING,        // 3
  PARTIALLY_AVAILABLE, // 4
  AVAILABLE,         // 5
  BLOCKLISTED,       // 6
  DELETED,           // 7
}
```

Media 实体同时维护 `status`（普通画质）和 `status4k`（4K 画质）两套状态，通过 `entity.is4k` 选择读写目标。

### 5.1.1 status 与 status4k 一致性窗口

两套状态**独立更新，互不影响**，通过动态 key `statusKey = entity.is4k ? 'status4k' : 'status'` 选择读写目标。

**典型读写模式** (`server/subscriber/MediaRequestSubscriber.ts:834`)：
```typescript
const statusKey = entity.is4k ? 'status4k' : 'status';
const externalIdKey = entity.is4k ? 'externalServiceId4k' : 'externalServiceId';

media[statusKey] = MediaStatus.PROCESSING;
media[externalIdKey] = radarrMovie.id;
```

**一致性窗口风险**：

| 场景 | 不一致状态 | 持续时间 | 代码位置 |
|---|---|---|---|
| 先写 status=PROCESSING，API 调用失败 | `status=PROCESSING` 但 `MediaRequest.status=FAILED` | 直到重试成功或请求被删除 | `updateParentStatus` vs `.catch()` |
| 先写 externalServiceId，再写 status | 短暂存在 `externalServiceId` 已设置但 `status` 仍为 PENDING | 毫秒级（两次 DB 写入之间） | `.then()` 回调中 |
| 普通和 4K 请求同时审批 | `status` 和 `status4k` 各自独立变迁，无同步机制 | 永久独立 | 全程 |
| MediaSubscriber 异步更新子请求 | `Media.status=AVAILABLE` 但 `MediaRequest.status=APPROVED` | 直到 `updateRelatedMediaRequest` 执行完毕 | `server/subscriber/MediaSubscriber.ts:34-121` |

**双状态一致性保证**：仅在 `updateParentStatus()` 中有前置检查：
```typescript
if (entity.status === MediaRequestStatus.APPROVED) {
  if (media[statusKey] !== MediaStatus.AVAILABLE
      && media[statusKey] !== MediaStatus.PARTIALLY_AVAILABLE
      && media[statusKey] !== MediaStatus.PROCESSING) {
    media[statusKey] = MediaStatus.PROCESSING;
  }
}
```
即不会覆盖已处于终态（AVAILABLE/PARTIALLY_AVAILABLE）的状态，但 PROCESSING 可被多次写入。

### 5.2 审批通过 → 父 Media 状态更新

`server/subscriber/MediaRequestSubscriber.ts:820-944` 的 `updateParentStatus()` 在 afterInsert/afterUpdate 中被调用：

| 请求状态 | 媒体类型 | 条件 | 写入 Media 状态 |
|---|---|---|---|
| APPROVED | MOVIE / TV | 当前 status 非 AVAILABLE / PARTIALLY_AVAILABLE / PROCESSING | **PROCESSING** |
| DECLINED | MOVIE | status != DELETED | **UNKNOWN** |
| DECLINED | TV | status === PENDING 且无其他同 is4k 的 PENDING 请求 | **UNKNOWN** |
| DECLINED | TV | 每个 seasonRequest | 标记为 DECLINED；若对应 Season.status === PENDING 且无其他活动请求，则该季 UNKNOWN |
| APPROVED | TV | 每个关联 seasonRequest | 标记为 **APPROVED** |

### 5.3 *arr API 调用成功 → 写回 Media 外部服务 ID

成功回调 `then()` 中写回 `server/entity/Media.ts` 的三个字段（均有 is4k 双份）：

| Media 字段 | 来源 | 作用 |
|---|---|---|
| `externalServiceId` / `externalServiceId4k` | `radarrMovie.id` / `sonarrSeries.id` | *arr 内部主键，用于后续扫描匹配 |
| `externalServiceSlug` / `externalServiceSlug4k` | `radarrMovie.titleSlug` / `sonarrSeries.titleSlug` | 用于拼接服务端跳转 URL |
| `serviceId` / `serviceId4k` | `radarrSettings.id` / `sonarrSettings.id` | 指向 Jellyseerr 配置中的服务器 ID |

写回后，`Media.afterLoad` 钩子会根据这三个字段组装出 `serviceUrl` 供前端跳转使用。

**写回顺序风险**：三个字段是**分别赋值后一次性保存**，如果保存前进程崩溃，可能出现部分写入。

### 5.4 快捷路径：媒体已可用

若审批时检测到 `media.status(status4k) === MediaStatus.AVAILABLE`，则跳过调用 *arr：

- Radarr 分支 (`server/subscriber/MediaRequestSubscriber.ts:348-361`)：直接 `entity.status = COMPLETED` 并保存
- Sonarr 分支 (`server/subscriber/MediaRequestSubscriber.ts:543-559`)：`entity.status = COMPLETED` 且所有 `seasons[].status = COMPLETED`

### 5.5 请求被删除 → 父 Media 状态回退

`afterRemove` 钩子 → `handleRemoveParentUpdate()` (`server/subscriber/MediaRequestSubscriber.ts:946-1004`)：

1. 统计该 media 下所有请求，判断是否还有**活动请求**（status 非 COMPLETED/DECLINED 且 is4k 匹配）
2. 若无活动请求：
   - 曾有 COMPLETED 请求 → `media.status = DELETED`（表示曾在库里后被删）
   - 否则 → `media.status = UNKNOWN`（回到初始态）

### 5.5.1 DELETED / UNKNOWN 删除请求的审计

`DELETED` vs `UNKNOWN` 的语义差异是 Jellyseerr 的**软删除审计机制**：

**判断逻辑** (`server/subscriber/MediaRequestSubscriber.ts:973-999`)：
```typescript
// 检查是否有过 COMPLETED 状态的请求（is4k 匹配）
const hadCompleted = allRequests.some(
  (req) => req.is4k === is4k && req.status === MediaRequestStatus.COMPLETED
);

if (hadCompleted) {
  cleanMedia[statusKey] = MediaStatus.DELETED;    // 曾同步过，后被删
} else {
  cleanMedia[statusKey] = MediaStatus.UNKNOWN;    // 从未成功同步
}
```

**审计含义**：

| 最终状态 | 含义 | 可追溯性 |
|---|---|---|
| `UNKNOWN` | 媒体从未成功进入 *arr 库，所有请求都在审批前被取消或失败 | 无外部系统痕迹 |
| `DELETED` | 媒体曾成功进入 *arr 库（至少有过一次 COMPLETED 请求），后所有关联请求被删除 | `externalServiceId` 仍保留，可追溯曾在 *arr 中的 ID |

**关键细节**：
- `externalServiceId` / `externalServiceSlug` / `serviceId` 在状态变为 `DELETED` 时**不会被清空**，保留审计线索
- 状态变为 `UNKNOWN` 时，这些字段同样**保留**（因为代码没有清理逻辑）
- 测试用例 `server/routes/request.test.ts:269-452` 完整覆盖了这一状态机的各种边界场景

**测试用例验证的场景**：
1. 删除重新请求但保留 stale COMPLETED 请求 → 恢复 `DELETED`
2. 删除 stale COMPLETED 请求 → 恢复 `UNKNOWN`
3. 仍有其他活动请求时 → 不修改状态
4. `PARTIALLY_AVAILABLE` 时删除 COMPLETED 请求 → 保留 `PARTIALLY_AVAILABLE`

### 5.6 可用通知触发

在 `afterUpdate` 中检测到 `entity.status === COMPLETED` 时：
- 电影 → `notifyAvailableMovie()`：再次读取最新 media 状态确认 AVAILABLE 后发送 `MEDIA_AVAILABLE` 通知
- 剧集 → `notifyAvailableSeries()`：检查所有申请的季均为 AVAILABLE 后发送通知

### 5.7 MediaSubscriber 反向同步

`server/subscriber/MediaSubscriber.ts` 监听 Media 实体变化，**反向更新关联的 MediaRequest**：

- `beforeUpdate`：当 `Media.status` 从 `PENDING` → `AVAILABLE` 时，自动 approve 所有同 is4k 的 PENDING 请求
- `afterUpdate`：当 `Media.status` 变为 `AVAILABLE` / `PARTIALLY_AVAILABLE` / `DELETED` 时，将关联的 APPROVED/FAILED 请求标记为 `COMPLETED`

这形成了一个**双向状态同步闭环**：
```
MediaRequest.APPROVED → Media.PROCESSING → *arr 下载 → Media.AVAILABLE → MediaRequest.COMPLETED
```

---

## 六、完整时序图（电影审批通过为例）

```
Admin                       API              MediaRequest         Subscriber          RadarrAPI         DB
  │  POST /request/42/approve │                   │                   │                  │              │
  ├──────────────────────────►│                   │                   │                  │              │
  │                           │  status=APPROVED  │                   │                  │              │
  │                           │──save(request)───►│                   │                  │              │
  │                           │                   │──afterUpdate()───►│                  │              │
  │                           │                   │                   │                  │              │
  │                           │                   │                   │──updateParentStatus()          │
  │                           │                   │                   │                  │              │
  │                           │                   │                   │  media.status=PROCESSING       │
  │                           │                   │                   │──────────────────────────────►│
  │                           │                   │                   │                  │              │
  │                           │                   │                   │──sendToRadarr()               │
  │                           │                   │                   │  (类型/状态校验通过)           │
  │                           │                   │                   │                  │              │
  │                           │                   │                   │  组装 RadarrMovieOptions       │
  │                           │                   │                   │                  │              │
  │                           │                   │                   │──POST /api/v3/movie──────────►│
  │                           │                   │                   │                  │              │
  │                           │                   │                   │◄──{id, titleSlug...}─────────│
  │                           │                   │                   │                  │              │
  │                           │                   │                   │  then(): 写回 Media            │
  │                           │                   │                   │  externalServiceId=...        │
  │                           │                   │                   │──────────────────────────────►│
  │                           │                   │                   │                  │              │
  │                           │◄── 200 OK ────────│                   │                  │              │
  │◄──────────────────────────┤                   │                   │                  │              │
  │                           │                   │                   │                  │              │
```

注：调用 Radarr API 是**异步 fire-and-forget**，HTTP 响应在 API 调用完成前就已返回给客户端。
