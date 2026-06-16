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

### 4.3.1 FAILED 重试的指数退避策略

**结论：Jellyseerr 没有内置指数退避机制。**

重试完全是**手动触发**的，每次点击「重试」按钮都会：
1. 将 `status` 从 `FAILED` 重置为 `APPROVED`
2. 更新 `modifiedBy` 和 `updatedAt`
3. 触发 TypeORM `afterUpdate` 钩子，从头执行完整分流流程

代码位于 `server/routes/request.ts:633-661`：
```typescript
mediaRequest.status = MediaRequestStatus.APPROVED;
mediaRequest.modifiedBy = req.user;
await requestRepository.save(mediaRequest);
```

**退避策略对比**：

| 策略类型 | 是否实现 | 说明 |
|---|---|---|
| 自动指数退避 | ❌ 无 | 没有定时器或后台 job 自动重试 |
| 重试次数限制 | ❌ 无 | 可无限次手动重试 |
| 失败历史记录 | ❌ 无 | 不记录失败次数或上次失败时间 |
| 手动重试 | ✅ 有 | 管理员点击重试按钮触发 |

**隐含的"退避"**：由于每次重试都需要管理员手动点击，人为操作间隔起到了自然退避效果。如果未来需要自动退避，可基于 `updatedAt` 字段计算距上次失败的时间。

### 4.3.2 TVDB 缺失错误的手动恢复路径

当 Sonarr 分流时检测不到 TVDB ID，会执行**破坏性处理**——直接删除 media 和 request 实体 (`server/subscriber/MediaRequestSubscriber.ts:569-574`)：

```typescript
if (!tvdbId) {
  const requestRepository = getRepository(MediaRequest);
  await mediaRepository.remove(media);
  await requestRepository.remove(entity);
  throw new Error('TVDB ID not found');
}
```

**TVDB ID 查找顺序**：
1. `series.external_ids.tvdb_id`（TMDB 实时查询结果）
2. `media.tvdbId`（数据库中已缓存的）
3. 两者皆空 → 删除并报错

**手动恢复路径**：

由于实体已被删除，无法通过「重试」按钮恢复。必须：

| 恢复方式 | 操作路径 | 前提条件 |
|---|---|---|
| 重新请求 | 用户在搜索结果中重新发起请求 | TMDB 已更新 tvdb_id 映射 |
| 手动补 tvdbId | 直接修改数据库 `media.tvdbId` 字段 | 知道正确的 TVDB ID |
| 等待扫描 | 等待 *arr 扫描器同步时自动创建 Media 实体 | Sonarr 中已存在该剧 |

**相关删除 API**：`DELETE /media/:mediaId` (`server/routes/media.ts:263-283`) 也会遇到同样的 TVDB ID 问题，但它是**真正的删除**（调用 `sonarr.removeSeries(tvdbId)` 从 Sonarr 中删除剧集），若 tvdbId 缺失则直接抛错不执行删除。

### 4.3.3 DELETE 物理删除的异步清理超时

`DELETE /media/:mediaId` 是同步 await *arr API 的 DELETE 调用，但 *arr 内部的文件清理是异步的。

**调用链**：
```
DELETE /media/:mediaId
  → radarr.removeMovie(tmdbId) / sonarr.removeSeries(tvdbId)
    → this.axios.delete(`/movie/${id}`, { params: { deleteFiles: true, addImportExclusion: false } })
```

代码位于 `server/api/servarr/radarr.ts:270-285` 和 `server/api/servarr/sonarr.ts:413-428`。

**超时配置**：

| 层级 | 超时值 | 配置位置 |
|---|---|---|
| HTTP 代理 keepAliveTimeout | 5000ms | `server/utils/customProxyAgent.ts:18,74` |
| HTTP 代理 socket timeout | 5000ms | `server/utils/customProxyAgent.ts:86` |
| Plex API 调用超时 | 5000ms | `server/routes/settings/index.ts:203` |
| axios 默认超时 | 未显式设置（Node.js 默认无超时） | — |

**风险**：
1. *arr DELETE API 本身是同步返回的，但它触发的磁盘文件删除是 *arr 内部的异步任务，Jellyseerr **不等待文件删除完成**
2. HTTP 层面仅有 5 秒超时，若 *arr 无响应，`await` 会在 5 秒后抛错，但此时 *arr 可能已接收请求，文件删除仍在进行
3. 删除 API 出错时，`server/routes/media.ts:286-292` 只打日志并返回 404，**不做任何回滚**，可能出现 "*arr 中已删但 Jellyseerr 还显示" 或 "Jellyseerr 认为删了但 *arr 中还在"

### 4.3.4 状态守卫下 FAILED 重试退出

重试将 `FAILED` 重置为 `APPROVED` 后，经过三层守卫可能会**静默退出**而不推进：

```
FAILED →(retry)→ APPROVED
                    │
                    ├─► 守卫 1: entity.type 是否正确？
                    │     不是 movie/tv → 静默 return
                    │
                    ├─► 守卫 2: 是否能找到 *arr 服务器？
                    │     isDefault 不匹配 / serverId 不存在 → 静默 return
                    │
                    └─► 守卫 3: Media 实体是否还存在？
                          已被 TVDB 缺失清理 → 静默 return
```

**静默退出的共同特征**：
- 不修改 `entity.status`（保持 APPROVED）
- 只打 warn/info 日志
- 不发送通知

**结果**：用户点击重试后，请求会**卡在 APPROVED 状态但没有任何实际动作**，需要查看日志才能发现原因。这与「卡在 APPROVED 但类型错误」是同一种静默失败模式。

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

### 5.1.2 四种不一致场景的优先级

按**影响程度从高到低**排序：

| 优先级 | 场景 | 影响 | 恢复方式 |
|---|---|---|---|
| 🔴 P0 | `Media.status=PROCESSING` 但 `MediaRequest.status=FAILED` | 用户看到"处理中"但实际已失败，产生误导 | 重试或删除请求 |
| 🟠 P1 | `externalServiceId` 已设置但 `status` 未同步 | 前端可能显示错误链接 | 下次扫描或请求时自动修正 |
| 🟡 P2 | 普通/4K 状态长期不一致（如一个 AVAILABLE 一个 PROCESSING） | 双画质状态不一致，影响用户判断 | 各自独立收敛，无强制同步 |
| 🟢 P3 | Media 变 AVAILABLE 但 MediaRequest 仍为 APPROVED | 短暂延迟，最终会被 MediaSubscriber 同步 | `updateRelatedMediaRequest` 异步执行 |

**为什么 P0 优先级最高**：
- 处于 PROCESSING 状态的媒体在前端会显示"处理中"
- 用户可能误以为系统正在工作，但实际已失败且不会自动恢复
- 需要管理员手动点击重试才能推进

### 5.1.3 status 与 status4k 回看一致性

**回看一致性**：即从任一画质视角看，状态迁移路径是否与单画质系统一致。

**单画质迁移路径**（正常流程）：
```
UNKNOWN → PENDING → PROCESSING → AVAILABLE
                       ↓
                     FAILED → APPROVED → PROCESSING → ...
```

**双画质并行时的一致性特点**：

1. **完全隔离**：`status` 和 `status4k` 是两个独立的状态机，各自拥有完整的状态枚举
2. **互不干扰**：一套状态的变迁不会触发另一套状态的变化
3. **各自终态**：普通版可以是 `AVAILABLE`，4K 版可以是 `DELETED`，两者共存

**代码佐证** (`server/subscriber/MediaRequestSubscriber.ts:834`)：
```typescript
const statusKey = entity.is4k ? 'status4k' : 'status';
// 只操作 statusKey 对应的那一套，不碰另一套
```

**回看验证方式**：
- 对于普通画质请求，只看 `media.status` 和 `serviceId` 等字段
- 对于 4K 请求，只看 `media.status4k` 和 `serviceId4k` 等字段
- 各自的状态迁移都是完整且自洽的

**潜在不一致点**：`media.mediaAddedAt` 只有一个字段，无法区分普通版和 4K 版的入库时间。

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

### 5.5.2 DELETED 软删除的真正删除时机

`MediaStatus.DELETED` 是**软删除标记**，而非真正的物理删除。它只表示：
> "这个媒体曾经在 *arr 库中存在过，现在已被移除"

**真正的物理删除发生在以下时机**：

| 删除方式 | 触发者 | 代码位置 | 后果 |
|---|---|---|---|
| 请求级删除 | 用户删除自己的请求 / 管理员删除请求 | `DELETE /request/:requestId` (`server/routes/request.ts`) | 仅删除 MediaRequest，Media 状态变为 DELETED/UNKNOWN |
| 媒体级删除 | 管理员从媒体详情页删除 | `DELETE /media/:mediaId` (`server/routes/media.ts:263-283`) | 同时删除 *arr 中的条目 + Jellyseerr 的 Media 实体 |
| 扫描器清理 | *arr 扫描时发现条目已不存在 | `baseScanner.ts` | 只更新状态为 DELETED，不物理删除 |
| 级联删除 | 删除用户时级联删除其请求 | `server/routes/user/index.ts:626-635` | 请求被级联删除，但 Media 状态不会更新（因为级联不触发 afterRemove） |

**`DELETE /media/:mediaId` 是唯一会真正删除 *arr 条目的操作**，流程：
```typescript
// server/routes/media.ts:273-282
if (isMovie) {
  await (service as RadarrAPI).removeMovie(media.tmdbId);
} else {
  const tvdbId = series.external_ids.tvdb_id ?? media.tvdbId;
  await (service as SonarrAPI).removeSeries(tvdbId);
}
```

`removeMovie` / `removeSeries` 会调用 *arr 的 DELETE API，并带参数：
- `deleteFiles: true` — 删除媒体文件
- `addImportExclusion: false` — 不加入排除列表（以后还能重新添加）

### 5.5.3 externalServiceId 迁移

`externalServiceId` / `externalServiceId4k` 是 *arr 系统内的主键 ID。它的变更（迁移）主要有两个途径：

#### 途径 1：请求时写入（首次同步）
`server/subscriber/MediaRequestSubscriber.ts:389,731`
```typescript
media[entity.is4k ? 'externalServiceId4k' : 'externalServiceId'] = radarrMovie.id;
```
这是最常见的写入时机：审批通过 → 调用 *arr API 创建条目 → 写回 ID。

#### 途径 2：扫描器同步（批量校正）
`server/lib/scanners/baseScanner.ts:180-187`
```typescript
if (
  externalServiceId !== undefined &&
  existing[is4k ? 'externalServiceId4k' : 'externalServiceId'] !== externalServiceId
) {
  existing[is4k ? 'externalServiceId4k' : 'externalServiceId'] = externalServiceId;
  changedExisting = true;
}
```
扫描器每次运行时，用 *arr 返回的 ID 覆盖本地 ID。

#### 5.5.4 externalServiceId 双写入冲突

1080p 和 4K 可能**使用不同的 *arr 服务器，因此理论上 `externalServiceId` 和 `externalServiceId4k` 可能**可能相同（同一个 *arr 实例中的不同条目）或不同（两个独立服务器）。

**AsyncLock 仅扫描器内部用 tmdbId 为 key 串行化处理 (`server/lib/scanners/baseScanner.ts:113`)：

```typescript
await this.asyncLock.dispatch(tmdbId, async () => {
  // getExisting → 条件判断 → save
});
```

AsyncLock 实现位于 `server/utils/asyncLock.ts`，确保同一个 tmdbId 的代码不会并发执行。

**冲突场景**：

| 场景 | 是否有冲突 | 说明 |
|---|---|---|
| 同时审批普通和 4K 请求 | ✅ 无冲突 | 两者写不同的字段 (`externalServiceId` vs `externalServiceId4k`)，互不干扰 |
| 扫描器同时扫描普通和 4K | ✅ 无冲突 | AsyncLock 按 tmdbId 串行化 |
| 审批请求和扫描器同时写同一画质 | ✅ 无冲突 | AsyncLock 按 tmdbId 串行化 |
| 审批请求和扫描器同时写不同画质 | ⚠️ 理论冲突 | 实际不冲突：字段独立 |
| 两个审批请求同时写同一画质的同一字段 | ❌ 存在竞态 | 罕见场景：一个可能覆盖另一个，但值相同，最终一致 |

**真实风险**：AsyncLock 仅扫描器内部使用，**MediaRequestSubscriber 分流时不使用 AsyncLock**。因此当两个管理员在毫秒级同时审批同一张电影的两个普通画质请求时，两个 `radarr.addMovie()` 是 fire-and-forget，它们的 `.then()` 回调可能同时写 `media.externalServiceId`。但由于 radarr.addMovie 本身是幂等的（查后更），两个请求拿到的是同一个 id，最终值一致，不构成实质冲突。

### 5.5.5 1080p 与 4K 同源条目下载冲突

1080p 和 4K 使用**独立的 *arr 服务器实例**（通过 `isDefault + is4k 选择），因此在 Jellyseerr 中互不干扰。但在 *arr 和媒体服务器层面可能冲突：

**Jellyseerr 内部完全隔离**：
- 两套独立字段：status/status4k、externalServiceId/externalServiceId4k、ratingKey/ratingKey4k
- 独立服务器配置：`is4k=true 走独立的 `isDefault && is4k` 的服务器
- AsyncLock 同一 tmdbId 串行化

**外部系统可能的潜在冲突**：

| 系统 | 冲突点 | Jellyseerr 防护 |
|---|---|---|
| Radarr/Sonarr | 1080p 和 4K 条目使用相同 tmdbId → *arr 是否认为是不同的条目 | 由 *arr 内部处理，Jellyseerr 不干预 |
| Plex/Jellyfin | 同一媒体的 1080p 和 4K 可能是两个独立的库条目 | 各自有独立的 ratingKey |

**配置约束**：Jellyseerr 的 `mediaAddedAt 只有一个字段，区分 1080p 和 4K 共用同一个 mediaAddedAt。

### 5.5.6 扫描器同步校正的窗口期

扫描器校正 `processMovie()` / `processShow()` 是 Jellyseerr 的**最终一致性保障机制**，但存在明显的校正窗口期。

**扫描器的执行时机**：
- Plex 扫描：定时任务，BUNDLE_SIZE = 20, UPDATE_RATE = 4000ms (`server/lib/scanners/baseScanner.ts:12-13`)
- Radarr/Sonarr 扫描：同样定时任务
- 手动点击「立即扫描」按钮

**校正窗口期**（从状态不一致的持续时间）：

| 不一致场景 | 持续时间 | 校正时机 |
|---|---|---|
| 请求审批通过 → 状态 PROCESSING → externalServiceId 已写回但 status 未变 | 毫秒级（fire-and-forget 回调内的两个字段分别保存） | .then() 回调 |
| 请求 fire-and-forget API 失败 → PROCESSING + FAILED | 直到下次扫描或手动重试 | 扫描器检测到 *arr 条目不存在或有文件 |
| *arr 下载完成 → 文件到位 | 到下次扫描运行时（分钟～小时级，取决于扫描间隔） | 扫描器运行时 |
| 删除用户级联删除请求 → 父 Media 状态停在 PROCESSING | 永远不会自动修复（见 5.5.7） | 需要手动删除 Media |

**扫描器状态机校正逻辑 (`server/lib/scanners/baseScanner.ts:119-134`：
```typescript
existing[statusField] =
  !processing && hasFile
    ? MediaStatus.AVAILABLE           // 有文件 → AVAILABLE
    : !processing && !hasFile && previousStatus === MediaStatus.PROCESSING
      ? MediaStatus.UNKNOWN             // 无文件且之前在处理中 → UNKNOWN
      : processing
        ? previousStatus === MediaStatus.DELETED
          ? MediaStatus.DELETED       // 处理中且之前被删 → 保持 DELETED
          : MediaStatus.PROCESSING    // 处理中 → PROCESSING
        : previousStatus;                // 其他情况保持原状态
```

关键：已 AVAILABLE 一旦设置后不再修改（守卫 `existing[status] !== AVAILABLE`），即扫描器不会把 AVAILABLE 降级。

### 5.5.8 三层守卫的异常 case 漏处理

闭环抑制依赖三层守卫的状态判断，但存在若干漏处理的边缘 case：

**三层守卫回顾**：

| 守卫层 | 位置 | 检查条件 |
|---|---|---|
| L1 | `sendToRadarr/Sonarr` 入口 | `status === APPROVED` + `type === MOVIE/TV` |
| L2 | `updateParentStatus` | `status === APPROVED \|\| DECLINED` 时才修改 Media |
| L3 | `MediaSubscriber.updateRelatedMediaRequest` | 只处理 `APPROVED / FAILED` 状态的请求 |

**已知漏处理 case**：

| 异常场景 | 经过守卫结果 | 后果 |
|---|---|---|
| `MediaRequest.status = DECLINED` 但 `Media.status = PROCESSING` | L1 不进；L2 对 MOVIE 重置 UNKNOWN，TV 需判断其他 PENDING；L3 不处理 | TV 可能残留 PROCESSING（若同画质无其他 PENDING） |
| `MediaRequest.status = COMPLETED` 但 `Media.status = PROCESSING` | L1 不进；L2 不处理 COMPLETED；L3 不处理 COMPLETED | 停留在 PROCESSING，直到扫描器校正 |
| `MediaRequest.status = PENDING` 但 `Media.status = PROCESSING`（审批流程回退） | L1 不进；L2 不处理 PENDING；L3 不处理 PENDING | PROCESSING 残留 |
| 批量删除请求时 `handleRemoveParentUpdate` 判断 `allRequests.some(req => req.status === COMPLETED)` | 可能误判曾有 COMPLETED 而写 DELETED | 审计语义失真（实际删除的请求未 COMPLETED 过） |
| 审批通过后，用户立即删除请求（PROCESSING 时删除） | `afterRemove` 触发 → 若无其他活动请求，判断是否有 COMPLETED 过 → 写 UNKNOWN/DELETED | 一般正确 |

**状态转换矩阵守卫缺口**：

```
PENDING  → APPROVED  ✅ L1 L2 都处理
APPROVED → COMPLETED ✅ L2(L3反向同步) 都处理
APPROVED → FAILED    ✅ L1 失败时标记 FAILED
FAILED   → APPROVED  ✅ 重试触发
APPROVED → DECLINED  ⚠️ L2 只对 MOVIE 重置 UNKNOWN，TV 需判断其他请求
DECLINED → APPROVED  ✅ 重新审批触发
任何     → COMPLETED ⚠️ L2 L3 都不主动修正 Media 状态，靠扫描器
任何     → 被删除    ✅ afterRemove 处理
```

### 5.5.9 ratingKey 双份在 Plex 与 Jellyfin 的差异

`ratingKey` 和 `jellyfinMediaId` 是两套独立的媒体服务器 ID 字段，各有 1080p / 4K 双份：

```typescript
// Media 实体字段（server/migration/sqlite 建表语句）
"ratingKey" varchar, "ratingKey4k" varchar,
"jellyfinMediaId" varchar, "jellyfinMediaId4k" varchar
```

**差异对比**：

| 维度 | Plex (ratingKey) | Jellyfin (jellyfinMediaId) |
|---|---|---|
| ID 格式 | 数字字符串（如 `"12345"`） | GUID 字符串（如 `"a1b2c3d4..."`） |
| 扫描器写入 | `server/lib/scanners/plex/index.ts:237,253` | `server/lib/scanners/jellyfin/`（对应 jellyfin scanner） |
| Season 级写入 | 在 `processShow` 内按季写入：有剧集则更新 `media.ratingKey` (`baseScanner.ts:310-321`) | 同左，`jellyfinMediaId` 同理 (`baseScanner.ts:323-338`) |
| Tautulli 查询 | 用 ratingKey 查询观看统计 (`server/routes/media.ts:351-356`) | — |
| 用户观看历史匹配 | 用 ratingKey4k 同时匹配两种画质的观看记录 (`server/routes/user/index.ts:863-910`) | — |
| 1080p / 4K 隔离 | 完全独立：`ratingKey` vs `ratingKey4k` | 完全独立：`jellyfinMediaId` vs `jellyfinMediaId4k` |
| 季级双份字段 | 无（Season 实体没有 ratingKey4k） | 无（Season 实体也没有 jellyfinMediaId4k） |

**Season 级别注意事项**：
`Season` 实体只有 `ratingKey` 和 `jellyfinMediaId`（单份），没有 4K 版本。在 `baseScanner.ts:310-338` 的 `processShow` 中，判断是根据 `season.episodes > 0`（普通）或 `season.episodes4k > 0`（4K）来更新 `media.ratingKey` / `media.ratingKey4k` 的，即 ratingKey 的双份是**媒体级**而非季级。

**观看历史查询**（`server/routes/user/index.ts:863-910`）：
```typescript
// 同时查询 ratingKey 和 ratingKey4k，任一命中就算看过
(!!media.ratingKey && parseInt(media.ratingKey) === record.rating_key) ||
(!!media.ratingKey4k && parseInt(media.ratingKey4k) === record.rating_key)
```
只要任一画质的 ratingKey 匹配 Tautulli 记录，即认为用户看过该媒体。

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

### 5.7.1 状态闭环的环路检测

双向 Subscriber 形成了一个理论上的状态闭环，但**实际上不会发生无限循环**，因为各层都有状态守卫：

```
┌─────────────────────────────────────────────────────────────┐
│  MediaRequestSubscriber.afterUpdate                         │
│    → 检测到 APPROVED → 更新 Media.status = PROCESSING       │
│    → 调用 *arr API（异步）→ 写回 externalServiceId          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
              Media.status = PROCESSING
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  MediaSubscriber.afterUpdate (Media 状态变化)               │
│    → 只有 status 变为 AVAILABLE / PARTIALLY_AVAILABLE /     │
│      DELETED 时才触发 updateRelatedMediaRequest              │
│    → 将关联的 APPROVED/FAILED 请求标记为 COMPLETED           │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
              MediaRequest.status = COMPLETED
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  MediaRequestSubscriber.afterUpdate (再次触发)               │
│    → sendToRadarr/Sonarr 守卫：status 不是 APPROVED，跳过    │
│    → updateParentStatus：请求是 COMPLETED，不修改 Media       │
│    → notifyAvailableMovie/Series：发送通知（仅一次）          │
└─────────────────────────────────────────────────────────────┘
```

**环路抑制机制**（三层守卫）：

| 守卫位置 | 检查条件 | 作用 |
|---|---|---|
| `sendToRadarr` / `sendToSonarr` 入口 | `entity.status === APPROVED` | COMPLETED 状态不会触发 *arr API 调用 |
| `updateParentStatus` | 仅 APPROVED / DECLINED 状态才修改 Media | COMPLETED 状态不会回写 Media |
| `MediaSubscriber.updateRelatedMediaRequest` | 只处理 APPROVED / FAILED 状态的请求 | COMPLETED 状态不会被重复处理 |

**潜在的小循环**：
- Media AVAILABLE → MediaRequest COMPLETED → notifyAvailable 再次读取 Media 确认 → 无写操作 → 终结
- 这个"读取确认"步骤是安全的，因为它只有读操作没有写操作，不会触发下一轮更新

**测试验证**：`server/routes/request.test.ts` 中的审批测试没有出现无限循环或堆栈溢出，证明环路抑制有效。

### 5.8 1080p 与 4K 切换

**结论：Jellyseerr 没有"切换"概念，1080p 和 4K 是完全独立的两套体系。**

#### 双轨并行设计

| 维度 | 1080p（普通） | 4K |
|---|---|---|
| 状态字段 | `media.status` | `media.status4k` |
| 外部服务 ID | `media.externalServiceId` | `media.externalServiceId4k` |
| 服务跳转 slug | `media.externalServiceSlug` | `media.externalServiceSlug4k` |
| 服务器配置 ID | `media.serviceId` | `media.serviceId4k` |
| Plex/Jellyfin ID | `media.ratingKey` | `media.ratingKey4k` |
| 请求字段 | `request.is4k = false` | `request.is4k = true` |
| 默认服务器选择 | `isDefault && !is4k` | `isDefault && is4k` |

#### "切换"的本质

用户不能直接把一个 1080p 请求"切换"成 4K 请求。所谓的"切换"实际上是：

1. **创建新请求**：用户为同一媒体提交一个 4K 请求（`is4k = true`）
2. **各自独立**：1080p 请求和 4K 请求并行存在，各自有独立的状态机
3. **分别同步**：两套状态分别与各自的 *arr 服务器同步
4. **删除旧请求**：用户可以删除 1080p 请求（只删除请求，不删除媒体文件）

#### 状态交互

1080p 和 4K 之间的**唯一交互**发生在 `handleRemoveParentUpdate` 删除请求时的状态判断中 (`server/subscriber/MediaRequestSubscriber.ts:955-999`)：

- 删除 1080p 请求 → 只检查 `hasActive`（非 4K 的活动请求） → 只修改 `media.status`
- 删除 4K 请求 → 只检查 `hasActive4k`（4K 的活动请求） → 只修改 `media.status4k`

两套状态完全独立，互不影响。

#### 查询时的关联

在请求列表 API 中，通过 SQL 条件将两套状态关联起来 (`server/routes/request.ts:134-138`)：

```sql
((request.is4k = false AND media.status IN (:...mediaStatus))
  OR
 (request.is4k = true AND media.status4k IN (:...mediaStatus)))
```

即：查询请求时，用 `request.is4k` 决定匹配 `media.status` 还是 `media.status4k`。

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
