# Plex 与 Jellyfin 媒体库扫描：扫描调用、媒体匹配与入队判定

## 一、整体架构概览

系统中存在四种扫描器，职责分工如下：

| 扫描器 | 职责 | 运行频率 |
|--------|------|----------|
| PlexScanner | 扫描 Plex 媒体库，检测已存在的媒体文件 | 最近5分钟增量 + 每日全量 |
| JellyfinScanner | 扫描 Jellyfin/Emby 媒体库，检测已存在的媒体文件 | 最近5分钟增量 + 每日全量 |
| RadarrScanner | 扫描 Radarr 下载队列，跟踪电影下载状态 | 每日全量 |
| SonarrScanner | 扫描 Sonarr 下载队列，跟踪剧集下载状态 | 每日全量 |

四种扫描器均继承自 `BaseScanner<T>`（`server/lib/scanners/baseScanner.ts:56`），共享核心的处理逻辑框架。

核心数据模型关系：
```
Media (tmdbId + mediaType 唯一标识)
  ├─ status / status4k    → 媒体可用性状态
  ├─ ratingKey / jellyfinMediaId  → Plex/Jellyfin 索引键
  ├─ externalServiceId     → Radarr/Sonarr 内部 ID
  └─ requests: MediaRequest[]

MediaRequest (请求队列项)
  ├─ status: PENDING → APPROVED → COMPLETED
  ├─ is4k: boolean
  ├─ seasons: SeasonRequest[]     (仅剧集)
  └─ media: Media                 (外键关联)
```

---

## 二、扫描器调用触发机制

### 2.1 调度系统：定时触发

所有扫描任务通过 `node-schedule` 在 `server/job/schedule.ts:33` 中注册 cron 任务：

```
startJobs()
  ├─ Plex 场景 (MediaServerType.PLEX)
  │   ├─ plexRecentScanner.run()   → 默认每 5 分钟 (cron 可配置)
  │   └─ plexFullScanner.run()     → 默认每 24 小时 (cron 可配置)
  │
  ├─ Jellyfin/Emby 场景 (MediaServerType.JELLYFIN/EMBY)
  │   ├─ jellyfinRecentScanner.run()  → 默认每 5 分钟
  │   └─ jellyfinFullScanner.run()    → 默认每 24 小时
  │
  ├─ radarrScanner.run()           → 每日全量
  ├─ sonarrScanner.run()           → 每日全量
  └─ availabilitySync.run()        → 媒体可用性反向校验 (防假删除)
```

扫描器生命周期钩子：
- `startRun()`（`baseScanner.ts:646`）：初始化会话 sessionId、检测 4K 服务是否启用、设置 `running = true`
- `endRun(sessionId)`（`baseScanner.ts:677`）：校验 session 一致性后标记 `running = false`
- `cancel()`：支持外部中断

### 2.2 Plex 扫描流程细节

**最近增量扫描** (`server/lib/scanners/plex/index.ts:90-139`)：
```typescript
for (const library of libraries) {
  // 使用 library.lastScan - 10min 缓冲时间查询新增
  libraryItems = plexClient.getRecentlyAdded(library.id, { addedAt })
  
  // 关键：按 ratingKey 维度去重聚合
  // - grandparentRatingKey (剧集)
  // - parentRatingKey (季)
  // - ratingKey (单集/电影)
  items = uniqWith(libraryItems, ...)
  
  // 分批处理 (bundleSize=50, 每批次间隔 UPDATE_RATE=4s)
  loop(processItem, { sessionId })
  
  // 更新 library.lastScan = Date.now()
}
```

**全量扫描** (`plex/index.ts:141-145`)：
```typescript
for (const library of libraries) {
  paginateLibrary(library)
    // 递归分页：getLibraryContents(libId, { size:50, offset:start })
    // 每批 50 条 → Promise.all(processItem) → setTimeout 4s → 下一批
}
```

### 2.3 Jellyfin 扫描流程细节

与 Plex 流程对称，但数据聚合键不同：
- Plex 使用 `ratingKey / parentRatingKey / grandparentRatingKey`
- Jellyfin 使用 `Id / SeasonId / SeriesId`（`server/lib/scanners/jellyfin/index.ts:504-514`）

额外特性：
- `processedAnidbSeason: Map<tmdbId, Map<season, count>>` 用于 AniDB 动漫多季合并统计（`jellyfin/index.ts:41`）

---

## 三、媒体匹配逻辑：从扫描结果到 TMDB ID

匹配的核心目标：**将 Plex/Jellyfin 中的 LibraryItem 通过多源 ID 解析，最终收敛到唯一的 `tmdbId`**。

### 3.1 Plex ID 解析策略 (`plex/index.ts:382 getMediaIds()`)

支持 7 种 agent 格式，按优先级匹配：

| 类型 | 正则 | 解析逻辑 |
|------|------|----------|
| 新 Plex Agent | `plex://` | 读取 `Guid[]` 数组，同时提取 imdb/tmdb/tvdb，然后互相补全 |
| IMDb Agent | `imdb://tt\d+` | IMDb ID → TMDB API `getMediaByImdbId()` |
| TMDB Agent (电影) | `tmdb://\d+` | 直接取 TMDB ID |
| TVDB Agent | `tvdb://\d+` | TVDB ID → TMDB API `getShowByTvdbId()` |
| TMDB Agent (剧集) | `themoviedb://\d+` | 直接取 TMDB ID |
| HAMA (TVDB) | `hama://tvdb?-(\d+)` | 同上，标记 `isHama=true` |
| HAMA (AniDB) | `hama://anidb?-(\d+)` | anidbId → animeList 映射表 → tmdb/imdb/tvdb |

**缓存优化**：使用 `cacheManager.getCache('plexguid')` 按 `ratingKey` 缓存解析结果，避免重复 API 调用。

**ID 补全链**：
```
imdbId 存在 && tmdbId 缺失
  → tmdb.getMediaByImdbId() → 填充 tmdbId

tvdbId 存在 && tmdbId 缺失
  → tmdb.getShowByTvdbId() → 填充 tmdbId
```

最终必须确保 `tmdbId` 存在，否则抛出 `Error('Unable to find TMDB ID')`。

### 3.2 Jellyfin ID 解析策略

**电影** (`jellyfin/index.ts:48 extractMovieIds()`)：
```
ProviderIds.Tmdb / TheMovieDb  → 直接使用
ProviderIds.Imdb               → tmdb.getMediaByImdbId() 补全
ProviderIds.AniDB              → animeList 查映射表 (仅当 tmdb/imdb 都缺失时)
```

**剧集** (`jellyfin/index.ts:217 processJellyfinShow()`)：
```
ProviderIds.Tmdb / TheMovieDb → tmdb.getTvShow()
ProviderIds.Tvdb              → tmdb.getShowByTvdbId()
ProviderIds.AniDB             → animeList → tvdbId/tmdbId
  ├─ 查到 tvdbId → getShowByTvdbId()
  └─ 只有 imdb/tmdb → 按动画电影处理，转调 processJellyfinMovie()
```

### 3.3 跨服务匹配：getExisting 判定

所有扫描器在处理时都调用 `BaseScanner.getExisting(tmdbId, mediaType)`（`baseScanner.ts:85`）：

```sql
SELECT * FROM Media WHERE tmdbId = ? AND mediaType = ?  -- 单条查找
```

如果找到 existing：更新字段（status、ratingKey、jellyfinMediaId 等）
如果没找到：**仅当满足 `hasFile || processing` 条件时才创建新 Media 记录**（`baseScanner.ts:210-212`）

**剧集级匹配** 使用 `seasons.find(es => es.seasonNumber === season.seasonNumber)` 在已加载的 seasons 数组内匹配。

---

## 四、入队判定：扫描结果与请求队列的交互机制

### 4.1 关键澄清：扫描器本身不"创建请求"

这是理解系统最重要的一点：

```
Plex/Jellyfin 扫描器 ──→ 只更新 Media 表的 status/status4k
                              不操作 MediaRequest (请求队列)

请求队列 (MediaRequest) ──→ 由以下路径创建:
                              1. 用户 POST /api/request (手动)
                              2. watchlistsync.ts (Plex 监视列表自动)
                              3. 其他自动请求逻辑
```

所以"入队判定"实际上是**双向的两阶段判定**：

### 4.2 阶段一：请求创建时的重复检测 (入队门槛)

当请求被创建时 (`MediaRequest.request()`, `entity/MediaRequest.ts:47`)，会做以下入队判定：

**① 通过 tmdbId 找（或创建）Media**：
```typescript
let media = mediaRepository.findOne({ tmdbId, mediaType, relations:['requests'] })

if (!media) {
  // 先创建占位 Media: status = PENDING, status4k = UNKNOWN (或反之)
  media = new Media({ tmdbId, status: PENDING, status4k: UNKNOWN, ... })
} else {
  // 已有 Media，检测黑名单
  if (media.status === BLOCKLISTED) → 抛出 BlocklistedMediaError
  
  // 根据 is4k 将 UNKNOWN/DELETED 升级为 PENDING
  media[is4k ? 'status4k' : 'status'] = PENDING
}
```

**② 同模式 (is4k) 下的重复请求检测**：
```typescript
existing = requestRepository
  .join('request.media')
  .where('is4k = :is4k AND tmdbId = :tmdbId AND mediaType = :mediaType')
  .getMany()

// 电影: 非 DECLINED/COMPLETED 的请求一律拦截
if (movie && existing[0].status !== DECLINED && !== COMPLETED)
  → 抛出 DuplicateMediaRequestError

// 自动请求: 同用户 + 同模式且非 DELETED 的自动请求拦截
if (existing.find(r => r.requestedBy.id === user.id && r.isAutoRequest && media[statusKey] !== DELETED))
  → 抛出 DuplicateMediaRequestError
```

**③ 剧集季去重 (防止同季重复入队)**：
```typescript
existingSeasons = media.requests
  .filter(r => r.is4k === body.is4k && r.status ∉ [DECLINED, COMPLETED])
  .flatMap(r => r.seasons.map(s => s.seasonNumber))

// 合并已扫描到（有文件）但无请求的季
existingSeasons += media.seasons
  .filter(s => s[statusKey] ∉ [UNKNOWN, DELETED])
  .map(s => s.seasonNumber)

finalSeasons = requestedSeasons - existingSeasons
if (finalSeasons.length === 0)
  → 抛出 NoSeasonsAvailableError
```

### 4.3 阶段二：请求批准后的 Radarr/Sonarr 入队 (真正的下载队列)

当 `MediaRequest.status` 变为 `APPROVED` 时，通过 TypeORM 的 EventSubscriber 触发入队：

**触发点**：
- `MediaRequestSubscriber.afterInsert()` → 新请求创建后（自动批准场景）
- `MediaRequestSubscriber.afterUpdate()` → 请求状态更新为 APPROVED 后

**电影入队 sendToRadarr()** (`subscriber/MediaRequestSubscriber.ts:183`)：
```typescript
if (entity.status === APPROVED && entity.type === MOVIE) {
  
  // ① 选服务器: is4k + isDefault 的 Radarr，或 serverId 覆盖
  radarrSettings = settings.radarr.find(isDefault && is4k === entity.is4k)
  
  // ② 先查 Media 状态，如已 AVAILABLE 直接 COMPLETED，不入队
  if (media[statusKey] === AVAILABLE) {
    entity.status = COMPLETED
    return  // ← 关键：扫描器先标记可用 → 批准时跳过入队
  }
  
  // ③ 异步调用 Radarr API
  radarr.addMovie({
    tmdbId, title, year,
    profileId, rootFolderPath, minimumAvailability,
    tags, searchNow: !preventSearch
  })
    .then(radarrMovie => {
      // 成功: 回填关联字段供后续扫描匹配
      media.externalServiceId4k/normal = radarrMovie.id
      media.externalServiceSlug = radarrMovie.titleSlug
      media.serviceId = radarrSettings.id
    })
    .catch(() => entity.status = FAILED)  // 入队失败
}
```

**剧集入队 sendToSonarr()** (`subscriber/MediaRequestSubscriber.ts:477`)：
与电影同构，增加：
- 通过 `ANIME_KEYWORD_ID` 判断动漫类型 → 切换 anime 专属 profile/rootFolder
- `seasons: entity.seasons.map(s => s.seasonNumber)` 传入目标季列表

### 4.4 阶段三：扫描器回填 → 触发请求完成

扫描周期与请求的时序图：
```
用户请求 → MediaRequest[PENDING→APPROVED]
              ↓ (Subscriber)
         Radarr/Sonarr.add*() → 进入下载队列
              ↓ (同时)
         Media.status = PROCESSING

  [时间流逝... Radarr/Sonarr 下载完成]

RadarrScanner 24h 扫描:
  radarrMovie.hasFile = true
    → processMovie(tmdbId, { processing:false, hasFile:true })
      → Media.status = AVAILABLE

 或 SonarrScanner:
    season.statistics.episodeFileCount === totalEpisodes
      → processShow(..., { processing:false })
        → Season.status = AVAILABLE
        → (rollup) Media.status = AVAILABLE/PARTIALLY_AVAILABLE

 或 Plex/JellyfinScanner (5min 增量或 24h 全量):
    检测到媒体文件实际存在
      → processMovie/processShow()
        → Media.status = AVAILABLE
        + 回填 ratingKey / jellyfinMediaId 等字段
```

当 `Media.status` 变为 `AVAILABLE` 后，**与请求队列的关联再次通过 Subscriber + 请求查询建立**：

`MediaRequestSubscriber.notifyAvailableMovie/Series()` 检查：
- 如果 MediaRequest 状态为 COMPLETED 且对应的 Media[statusKey] === AVAILABLE
- 则发送 `MEDIA_AVAILABLE` 通知给请求用户

（注：由于扫描器直接更新 Media 而非 MediaRequest，COMPLETED 标记实际上是通过以下路径间接触发的：扫描完成后 Radarr/Sonarr 记录 hasFile=true → 下次 `availabilitySync` 或下次请求查询时对比触发。实际系统中 COMPLETED 主要依赖于 Sonarr/Radarr 扫描与 Plex 扫描的双重确认。）

### 4.5 processMovie 核心判定矩阵

`BaseScanner.processMovie()` (`baseScanner.ts:95-258`) 是所有扫描器共有的状态判定核心：

| 场景 | existing? | processing | hasFile | 结果 status |
|------|-----------|------------|---------|-------------|
| 发现新文件，非 4K | 否 | false | true | AVAILABLE |
| 发现新文件，4K (且 enable4kMovie) | 否 | false | true | status4k = AVAILABLE |
| Radarr 中但没文件 (监控中) | 否 | true | false | PROCESSING |
| Radarr 未监控 + 无文件 | 否 | false | false | 跳过，不创建 Media |
| 已有记录，原非 AVAILABLE，现在有文件 | 是 | false | true | AVAILABLE |
| 已有记录，原 PROCESSING，现在 Radarr 也没了 | 是 | false | false | UNKNOWN |
| 已有记录，状态为 DELETED，但仍在处理 | 是 | true | false | 保持 DELETED |
| 其他情况 | 是 | - | - | 保持原状态不变 |

字段更新规则：
- `ratingKey / ratingKey4k`: Plex 专属，写入时检测 is4k 对应字段
- `jellyfinMediaId / jellyfinMediaId4k`: Jellyfin 专属，同上
- `serviceId / externalServiceId / externalServiceSlug`: Radarr/Sonarr 专属
- `mediaAddedAt`: 首次检测到文件时写入（Plex addedAt 或 Jellyfin DateCreated）

---

## 五、可用性反向校验与异常修正

`availabilitySync.ts` 提供**反向扫描**机制作为正向扫描的纠错层：

**工作原理**（`availabilitySync.ts:38 run()`）：
1. 分页加载所有当前标记为 AVAILABLE / PARTIALLY_AVAILABLE 的 Media
2. 对每条记录同时查 Plex/Jellyfin + Radarr/Sonarr
3. 只要**任一**来源确认存在 → 保留状态
4. **所有**来源都查不到 → 降级为 DELETED

```typescript
movieExists = (existsInPlex || existsInRadarr)
movieExists4k = (existsInPlex4k || existsInRadarr4k)

if (!movieExists && media.status === AVAILABLE)
  → mediaUpdater(media, is4k=false) → status = DELETED
```

**防误删保护**：
- API 返回 404 以外的错误 → 视为"仍可能存在"，不删
- 剧集级 `preventSeasonSearch`：API 异常时保留全剧状态
- 下载状态检查：如 MediaRequest 存在 APPROVED 的请求，DELETED 时不清除 externalServiceId，便于重试恢复

---

## 六、异常与边界场景

### 6.1 竞态保护：AsyncLock

`BaseScanner.asyncLock.dispatch(tmdbId, async () => ...)` 保证同一 `tmdbId` 不会被并行处理：
- Plex/Jellyfin/Radarr/Sonarr 四种扫描器可能几乎同时运行
- 都调用同一 processMovie() 时会排队执行，防止数据库脏写

### 6.2 HAMA 动漫特殊处理

Plex 上 HAMA 代理的条目经常是 "剧集中的电影" 或 "AniDB 风格"：

**Hama 无 TVDB ID 的电影** (`plex/index.ts:307-310`)：
```
processHamaMovie() → 取第一个 Season 的第一个 Episode
  → 按 episode 为单位调用 processPlexMovieByTmdbId()
```

**Hama 有 TVDB ID 的特别篇** (`plex/index.ts:314-316`)：
```
processHamaSpecials() → 遍历 S00 的每一集
  → animeList.getSpecialEpisode(tvdbId, epIndex)
    → 查到 tmdbId 或 imdbId → processPlexMovieByTmdbId()
```

### 6.3 AniDB 动漫季合并 (Jellyfin)

Jellyfin 中 AniDB 条目常把一部动漫的多季拆成多个 Series。`processedAnidbSeason` 跨 Jellyfin LibraryItem 聚合 episode 计数，防止 TMDB 上同一季的 episode count 被反复覆盖为不完整数字。

---

## 七、关键文件索引

| 文件路径 | 职责 |
|----------|------|
| `server/lib/scanners/baseScanner.ts` | 抽象基类，processMovie/processShow 判定矩阵 |
| `server/lib/scanners/plex/index.ts` | Plex 扫描、7 种 Agent ID 解析 |
| `server/lib/scanners/jellyfin/index.ts` | Jellyfin/Emby 扫描、AniDB 聚合 |
| `server/lib/scanners/radarr/index.ts` | Radarr 扫描、orphan 清理 |
| `server/lib/scanners/sonarr/index.ts` | Sonarr 扫描、orphan 清理 |
| `server/job/schedule.ts` | cron 调度器注册 |
| `server/entity/MediaRequest.ts` | 请求创建 (MediaRequest.request)、重复检测 |
| `server/subscriber/MediaRequestSubscriber.ts` | 批准后 Radarr/Sonarr 入队 + 状态回填 |
| `server/lib/availabilitySync.ts` | 反向可用性校验防误删 |
| `server/routes/request.ts` | 请求 CRUD API 路由 |
