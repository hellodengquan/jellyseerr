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
- `cancel()`：支持外部中断（设置 `running = false`，下一批次开始前检测并抛出中断错误）

### 2.2 Plex 扫描流程细节

**最近增量扫描** (`server/lib/scanners/plex/index.ts:90-139`)：
```typescript
for (const library of libraries) {
  // 使用 library.lastScan - 10min 缓冲时间查询新增
  libraryItems = plexClient.getRecentlyAdded(library.id, { addedAt }, library.type)
  
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

### 2.4 增量扫描 since timestamp 处理路径

**Plex 增量扫描时间戳策略** (`plex/index.ts:90-107, 126-138`)：

```
初次扫描 (lastScan 为空):
  → 不传 addedAt 参数 → Plex API 默认返回 1 小时内新增
  → 相当于保守的"最近一批"保底

非初次扫描:
  → addedAt = library.lastScan - 10 * 60 * 1000
  → 即：从上一次扫描时间往前推 10 分钟作为缓冲
  → 避免 Plex 端入库延迟 / 时钟偏差导致漏扫
```

**时间戳持久化**：
- 每个 library 独立维护 `lastScan` 字段（存在 `settings.json` 的 `plex.libraries[]` 中）
- 增量扫描每完成一个 library 就立刻 `settings.save()` 写入
- 全量扫描不更新 lastScan（全量走分页，不依赖时间窗口）

**Jellyfin 增量扫描的差异** (`jellyfin/index.ts:492-517`)：

Jellyfin 增量扫描**不使用时间戳**，而是调用 `/Items/Latest?Limit=12` 接口：
- 固定拉取最近 12 条新增
- 无 lastScan 持久化，也无时间缓冲
- 依赖扫描频率（每 5 分钟）来弥补数量限制
- 对于大批量入库场景，必须等全量扫描才能完整覆盖

**Plex API 时间参数细节** (`plexapi.ts:217-237`)：
```typescript
// addedAt 以秒为单位（Plex 用 Unix timestamp）
`&sort=addedAt%3Adesc&addedAt>>=${Math.floor(options.addedAt / 1000)}`

// 默认 1 小时兜底
options: { addedAt: Date.now() - 1000 * 60 * 60 }
```

**时间回溯与 since 复位场景**：

`lastScan` 字段的设计隐含了几个边界场景的处理：

- **系统时间回拨 / NTP 同步**：如果系统时间在两次扫描之间回拨（例如 NTP 修正时钟偏差），`library.lastScan` 会大于当前时间，导致 `addedAt = library.lastScan - 10min` 变成未来时间。Plex API 对未来的 `addedAt>>` 参数会返回空结果（无新增），但下次扫描时 `lastScan` 会被更新为正确的当前时间，恢复正常。
- **手动重置 since**：没有提供"从头扫描"的 API，但用户可以通过 UI 触发全量扫描 (`POST /api/v1/settings/plex/sync?start=true`)，全量扫描走 pagination 路径，不依赖 `lastScan`，相当于隐式回溯整个库。
- **library.lastScan 丢失**：初次运行或配置损坏时 `lastScan` 为 `undefined`，此时增量扫描走"最近 1 小时兜底"，之后每次扫描更新，逐步收敛到正确的时间窗口。
- **删除 library 后重新添加**：lastScan 字段是 library 对象的一部分，library 被移除后重建，时间戳从零开始，不会继承旧 library 的扫描进度。

### 2.5 PlexAPI 与 JellyfinAPI 统一适配 schema mapping

虽然 Plex 和 Jellyfin 的返回结构差异巨大，但扫描器内部通过字段映射在 `processItem` 层统一处理。以下是核心 schema 的对照关系：

| 语义字段 | Plex LibraryItem | Jellyfin LibraryItem | 说明 |
|----------|------------------|----------------------|------|
| 唯一标识 | `ratingKey` | `Id` | 媒体库内主键，字符串 |
| 上级（季）标识 | `parentRatingKey` | `SeasonId` | 剧集场景 |
| 上级（剧集）标识 | `grandparentRatingKey` | `SeriesId` | 剧集场景 |
| 标题 | `title` | `Name` | - |
| 类型 | `type: 'movie'/'show'` | `Type: 'Movie'/'Series'` | 大小写 + 单词差异 |
| ID 来源 | `guid` 字符串 + `Guid[]` 数组 | `ProviderIds: {}` 对象 | 见下详述 |
| 添加时间 | `addedAt` (秒级 unix) | `DateCreated` (ISO 字符串) | 首次入库时间 |
| 媒体质量 | `Media[].videoResolution` | `Width/Height` + `IsHD` | 4K 判断依据 |

**ID 解析层的结构性差异**：

Plex 有两种 ID 提供方式：
1. 旧代理：单个 `guid` 字符串，形如 `com.plexapp.agents.imdb://tt12345?lang=en`
2. 新 Plex Agent：`Guid[]` 数组，每项形如 `{ id: "imdb://tt12345" }` 或 `"tmdb://12345"`

Jellyfin 则是 `ProviderIds` 对象：
```typescript
{ Tmdb: "123", TheMovieDb: "123", Imdb: "tt123", Tvdb: "456", AniDB: "789" }
```

**扫描器内部分层解耦**：
- `BaseScanner<T>` 不感知 API 差异，只处理 `tmdbId + 状态参数`
- PlexScanner / JellyfinScanner 各自的 `processItem()` 负责把 API 数据转换成 `processMovie(tmdbId, { hasFile, processing, ... })` 标准调用
- 这种适配模式保证了 processMovie/processShow 判定逻辑的复用

### 2.6 自定义 Server Type 扩展点

当前支持 3 种媒体服务器类型，枚举定义在 `server/constants/server.ts:1-6`：
```typescript
export enum MediaServerType {
  PLEX = 1,
  JELLYFIN,
  EMBY,
  NOT_CONFIGURED,
}
```

**扩展新的媒体服务器类型需要修改以下关键点**：

1. **枚举扩展**：在 `MediaServerType` 中增加新值（如 `KODI = 5`）
2. **调度分支**：在 `server/job/schedule.ts:37-111` 的 `if/else if` 链中增加新的扫描任务注册分支
3. **UI 选择**：在 `src/components/Setup/index.tsx:77-79` 的 Setup 页面增加类型映射和路由
4. **设置 API**：在 `server/routes/settings/` 下增加对应服务器的设置路由（参考 `settings/jellyfin/`）
5. **认证路由**：在 `server/routes/auth.ts` 中增加对应的 OAuth 登录分支
6. **用户类型**：在 `server/constants/user.ts` 中增加对应 `UserType` 枚举值
7. **扫描器实现**：继承 `BaseScanner<T>`，实现 `run()` + `status()` + processItem 适配

**架构约束**：
- 全局只能同时启用一种媒体服务器类型（全局单例 `settings.main.mediaServerType`）
- `NOT_CONFIGURED` 是初始状态，完成 setup 流程后切换到实际类型
- 切换媒体服务器类型需要重新 setup，现有 Media 数据会保留但 ratingKey 等索引键失效

**Jellyfin/Emby 代码复用的设计启示**：
Emby 与 Jellyfin API 高度兼容，因此**共用同一套扫描器代码**，仅在 API 基础 URL 和品牌文案上有差异：
- `MediaServerType.JELLYFIN` 和 `MediaServerType.EMBY` 走同一代码分支 (`schedule.ts:108-111`)
- 差异通过 `ServerType` 枚举字符串区分（`server/constants/server.ts:8-11`）
- 这种 "兼容服务器共享扫描器" 的模式是扩展新类型时的重要参考

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

### 3.4 无外部 ID 时的 fallback 行为

**系统没有做基于标题/文件名的模糊匹配** —— 这是一个重要的设计决策：

- **Plex 侧**：`getMediaIds()` 解析失败时直接 `throw new Error('Unable to find TMDB ID')`
  - 上层 `processItem()` catch 住错误 → `this.log('Failed to process Plex media', ...)`
  - 该条目被**静默跳过**，不进入数据库，不计入 mediaAddedAt
  - 常见于 `com.plexapp.agents.none` 代理的个人视频库（在 syncLibraries 阶段就被过滤掉）

- **Jellyfin 侧**：`extractMovieIds()` / `processJellyfinShow()` 如果 ProviderIds 全空
  - 直接 `return` 跳过，不抛出错误
  - 不创建 Media 记录，不做任何本地映射

**设计意图**：
- 系统的核心是"请求 → 下载 → 入库"闭环，TMDB ID 是贯穿全链路的主键
- 没有外部 ID 的媒体（家庭录像、个人视频等）不在系统的管理范围内
- 避免基于标题的模糊匹配带来的误匹配风险（同名电影、重制版等）

### 3.5 文件名指纹与同名歧义解决

**扫描器层面不做文件指纹匹配**：
- 系统没有计算文件 hash 的逻辑
- 没有基于文件名 / 文件大小的模糊匹配
- `wink-jaro-distance` 包被引入，但**仅用于 Rotten Tomatoes 评分的标题匹配**，不用于扫描器

**RT 评分的相似度匹配参考** (`server/api/rating/rottentomatoes.ts:51-90`)：
虽然不是扫描器的功能，但 RT 评分的匹配算法展示了系统如何处理"同名不同内容"的歧义：

```typescript
// 标题归一化：小写 + 移除非字母数字字符
const norm = (s: string) => s.toLowerCase().replace(/[^\p{L}\p{N} ]/gu, '');

// Jaro 相似度计算：完全匹配 = 1，否则 jaro * 0.25 惩罚
const similarity = (a, b) => a === b ? 1 : jaro(a, b).similarity * 0.25;

// 综合评分 = 标题相似度 * 年份差惩罚 * 评分存在奖励
const score = (result, name, year) => 
  t_score(result, name) * y_score(result, year) * extra_score(result);

// 年份差惩罚：0 年 = 1.0，每差 1 年 -0.4，3 年以上归零
const y_score = (r, y) => y ? Math.max(0, 1 - Math.abs(r.releaseYear - y) * 0.4) : 1;

// 阈值过滤：score > 0.175 才接受匹配
const best = (results, name, year) => 
  results.map(...).filter(({ score }) => score > MINIMUM_SCORE)[0]?.result;
```

**扫描器为什么不采用类似算法**：
1. **准确性 vs 可用性**：RT 评分是辅助信息，匹配错了影响不大；但媒体匹配错了会导致错误标记 AVAILABLE，用户收不到通知，后果严重
2. **ID 可用性**：Plex/Jellyfin 作为元数据服务器，入库时已经做过 ID 匹配，扫描器信任上游结果
3. **误匹配的风险**：《蝙蝠侠》(1989) vs 《蝙蝠侠》(2022)，标题 + 年份模糊匹配也可能出错，特别是重制版、导演剪辑版、同名不同语言版本
4. **人工兜底**：扫描不到的媒体，用户仍可手动标记为可用（通过 `PUT /api/v1/media/:id` 设置 status）

**唯一的"歧义解决"字段**：`mediaAddedAt`
- Plex 扫描时写入 `plexitem.addedAt`（媒体库实际入库时间）
- Jellyfin 扫描时写入 `DateCreated`
- 两个条目同 tmdbId 但不同 mediaAddedAt 时，后扫描的会覆盖（因为 tmdbId 是唯一键）
- 这实际上是"同一内容不同版本"的最终一致性解决策略：以最新入库的文件为准

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

### 4.6 请求终态归并：DECLINED 与 FAILED 的语义边界

**两种终态的触发路径**：

| 状态 | 触发时机 | 触发者 | 语义 |
|------|----------|--------|------|
| `DECLINED` | 管理员拒绝请求 | 人工 / API `POST /api/request/:id/approve` | 主观拒绝，不重试 |
| `FAILED` | 入队 Radarr/Sonarr 失败 | Subscriber 异步回调 | 客观失败，可重试 |

**在重复检测中的归并逻辑**（`MediaRequest.request()` 去重判断）：

```typescript
// 电影: DECLINED 和 COMPLETED 一样，都不算"活跃请求"
// 也就是说：被拒绝过的请求可以重新提交
if (existing[0].status !== MediaRequestStatus.DECLINED 
    && existing[0].status !== MediaRequestStatus.COMPLETED) {
  throw new DuplicateMediaRequestError();
}
```

**关键结论：DECLINED 和 COMPLETED 都被视为"终态"，不阻碍新请求创建**。
- 用户请求被拒绝后，可以再次发起相同请求（给管理员再次审核的机会）
- FAILED 状态的请求**仍然会被去重拦截**，因为它本质上是 APPROVED 的失败版本

**FAILED 请求的重试机制**：
- 系统没有自动重试 FAILED 请求的逻辑
- 管理员可以通过 UI 手动重新批准（状态从 FAILED → APPROVED，会重新触发入队）
- 这是与 DECLINED 的本质区别：FAILED 代表"批准了但入队失败"，DECLINED 代表"根本不批准"

**通知系统中的归并**（`MediaRequestSubscriber`）：
- `MEDIA_DECLA`对应`MediaRequestStatus.DECLINED`
- `MEDIA_FAILED` 对应 `MediaRequestStatus.FAILED`
- 两者触发不同的通知事件和文案

### 4.7 DECLINED 与 FAILED 的重审窗口期

系统**没有硬编码的时间窗口限制** —— DECLINED 和 FAILED 请求的"重审"是随时允许的，但两者的触发路径完全不同：

**DECLINED 的重审路径**：
- **创建新请求**：DECLINED 被视为终态，不参与去重拦截。用户可以随时发起新请求，走完整的 PENDING → APPROVED/DECLINED 流程
- **编辑现有请求**：`PUT /api/v1/request/:id` 可以修改请求的 seasons、rootFolder 等字段，但不会自动重置 status 为 PENDING
- **管理员手动重置**：`POST /api/v1/request/:id/pending` 可以将 DECLINED 请求重置为 PENDING，需要 `MANAGE_REQUESTS` 权限

**FAILED 的重审路径** (`server/routes/request.ts:636-661`)：
```typescript
// /:requestId/retry
request.status = MediaRequestStatus.APPROVED;  // 直接恢复到 APPROVED
request.modifiedBy = req.user;
await requestRepository.save(request);
// save 会触发 afterUpdate Subscriber，重新调用 sendToRadarr/Sonarr
```

- **无需再次审批**：FAILED 是"已经批准过但执行失败"，retry 直接回到 APPROVED 状态，跳过审批流程
- **不限次数**：retry 接口没有调用次数限制，也没有冷却时间
- **权限要求**：需 `MANAGE_REQUESTS` 权限，用户不能自己重试失败的请求

**两个关键的无窗口设计决策**：
1. **无时间窗口**：1 年前被拒绝的请求，今天仍然可以重新提交。设计假设是"用户需求可能会变化，管理员的决策也可能变化"
2. **无冷却时间**：同一请求 retry 失败后可以立即再次 retry。考虑到 FAILED 是客观技术原因（Radarr 宕机、API 超时等），快速重试是合理的

**隐式窗口：重复检测的存在周期**：
- 对于 **FAILED** 请求，只要它保持 FAILED 状态，就会阻止同 tmdbId+is4k 下的新请求创建
- 只有以下两种方式解除：管理员 retry（回到 APPROVED），或管理员删除该 FAILED 请求
- 这形成了一个"隐性窗口期"：FAILED 状态存在期间 = 同内容不可重新请求

---

## 五、多 Server 实例冲突合并与去重

### 5.1 Radarr/Sonarr 多实例架构

系统支持配置多个 Radarr / Sonarr 服务器（`settings.radarr: RadarrSettings[]`），常见使用场景：
- 标准画质 + 4K 画质分离
- 不同语言 / 地区的内容分离
- 动漫与普通内容分离

每个实例有独立的：
- `is4k`：是否 4K 服务器
- `isDefault`：是否默认服务器（决定入队时选哪台）
- `syncEnabled`：是否参与扫描同步
- `activeProfileId / activeDirectory / tags` 等配置

### 5.2 多服务器扫描时的去重策略

**扫描器启动时的去重** (`radarr/index.ts:55-61`, `downloadtracker.ts:71-77`)：

```typescript
this.servers = uniqWith(settings.radarr, (radarrA, radarrB) => {
  return (
    radarrA.hostname === radarrB.hostname &&
    radarrA.port === radarrB.port &&
    radarrA.baseUrl === radarrB.baseUrl
  );
});
```

- 按 **hostname + port + baseUrl** 三元组去重
- 即使在设置里配置了多条记录，只要指向同一台物理服务器，只扫描一次
- 避免重复 API 调用和重复处理

**下载追踪器的"数据复制"策略** (`downloadtracker.ts:119-139`)：

DownloadTracker 针对同一物理服务器的多个逻辑配置做了数据同步：
```typescript
// 从其中一个配置拉取队列数据
const queueItems = await radarr.getQueue();
this.radarrServers[server.id] = queueItems.map(...);

// 复制给同服务器的其他逻辑配置
matchingServers.forEach(ms => {
  if (ms.syncEnabled) {
    this.radarrServers[ms.id] = this.radarrServers[server.id];
  }
});
```

### 5.3 Orphan 清理的保守策略

**多实例场景下不能随意把"某台服务器上没有的媒体"判定为孤儿**（`radarr/index.ts:91-107`）：

```typescript
// 只有同类型的所有服务器都参与了扫描，才做 orphan 清理
// 只要有一台没开 syncEnabled，就跳过清理

const allStandardScanned = this.servers
  .filter(s => !this.enable4kMovie || !s.is4k)
  .every(s => s.syncEnabled);

const all4kScanned = this.servers
  .filter(s => this.enable4kMovie && s.is4k)
  .every(s => s.syncEnabled);

if (!allStandardScanned) {
  this.didScanStandard = false; // 标记不清理
}
```

**设计意图**：如果用户配置了多台 Radarr（如动画 + 电影），但只开了其中一台的同步，清理逻辑会误把另一台上存在的媒体当成孤儿。宁可漏清理，不可误删。

### 5.4 入队时的服务器选择

请求批准后入队的服务器选择逻辑（`MediaRequestSubscriber.ts`）：

```typescript
// 优先级：
// 1. 请求上显式指定的 serverId (body.serverId)
// 2. 同类型 (is4k) 下 isDefault = true 的服务器
// 3. 同类型下的第一台服务器

const radarrSettings = settings.radarr.find(r => r.isDefault && r.is4k === entity.is4k)
  ?? settings.radarr.find(r => r.is4k === entity.is4k);
```

**多服务器下的状态写入**：
- 每台服务器扫描时，只更新自己对应 `serviceId` 匹配的记录？
- 实际上：processMovie 不校验 serviceId，直接根据 is4k 写入
- 如果两台非 4K Radarr 都配置了同一部电影，后扫描的会覆盖 serviceId/externalServiceId
- 这是合理的，因为媒体是否存在是"或"的关系，指向哪个服务器不重要

### 5.5 多 Server 优先级动态调整

**静态优先级：isDefault 标志**：
- 每个服务器有一个 `isDefault: boolean` 字段
- 同类型 (is4k) 下只能有一台 `isDefault = true`
- 当请求没有指定 `serverId` 时，优先选择 `isDefault = true` 的服务器
- 如果没有 isDefault 服务器，则 `find()` 返回 undefined，回退到同类型下的第一台服务器

**动态优先级：请求级覆盖** (`server/routes/request.ts:509, 518`)：

请求创建和编辑时可以显式指定 `serverId`，优先级高于 isDefault：
```typescript
// 创建请求时
if (req.body.serverId) {
  request.serverId = req.body.serverId;
}

// 编辑请求时
if (req.body.serverId !== undefined) {
  request.serverId = req.body.serverId;
}
```

**入队时的优先级解析** (`MediaRequestSubscriber.ts:203-223`)：
```typescript
// 第一步：找 isDefault 的服务器
let radarrSettings = settings.radarr.find(
  r => r.isDefault && r.is4k === entity.is4k
);

// 第二步：如果请求有 serverId 覆盖，且与默认不同，按 serverId 精确查找
if (entity.serverId !== null && entity.serverId >= 0 
    && radarrSettings?.id !== entity.serverId) {
  radarrSettings = settings.radarr.find(r => r.id === entity.serverId);
}

// 第三步：都没找到 → 记 warn 日志，不入队
if (!radarrSettings) {
  logger.warn('No default server configured');
  return;
}
```

**rootFolder 与 profileId 的次级覆盖**：
请求上还可以覆盖 `rootFolder` 和 `profileId`，优先级高于服务器默认配置：
```typescript
if (entity.rootFolder && entity.rootFolder !== radarrSettings.activeDirectory) {
  rootFolder = entity.rootFolder;  // 请求级覆盖
}

if (entity.profileId && entity.profileId !== radarrSettings.activeProfileId) {
  qualityProfile = entity.profileId;  // 请求级覆盖
}
```

**优先级总表（从高到低）**：
1. `request.serverId`（请求创建时指定，可覆盖）
2. `request.rootFolder` / `request.profileId`（请求级参数覆盖）
3. `server.isDefault && server.is4k === request.is4k`（同类型默认服务器）
4. `server.is4k === request.is4k`（同类型第一台服务器）

**运行时动态调整**：
- 修改服务器的 `isDefault` 属性会立即生效（下一次入队时使用）
- 但**已创建的请求**不会因为默认服务器变更而改变 serverId
- 已存在的请求的 serverId 需要通过 `PUT /api/v1/request/:id` 编辑修改
- 没有批量迁移请求 serverId 的 API，需要逐条编辑

---

## 六、扫描进度反馈与 UI 实时更新

### 6.1 后端 status 接口

所有扫描器都实现了 `status()` 方法，返回标准化的进度信息：

**Base 状态** (`baseScanner.ts:14-19`)：
```typescript
interface StatusBase {
  running: boolean;   // 是否正在运行
  progress: number;  // 已处理数量 (items 数组的索引位置)
  total: number;     // 总数量
}
```

**Plex 扩展状态** (plex/index.ts:54-59)：
```typescript
interface PlexSyncStatus extends StatusBase {
  currentLibrary: Library;
  libraries: Library[];
}
```

**Radarr 扩展状态** (radarr/index.ts:15-18)：
```typescript
interface SyncStatus extends StatusBase {
  currentServer: RadarrSettings;
  servers: RadarrSettings[];
}
```

**HTTP 接口**（`routes/settings/index.ts:257-268`）：
```
GET  /api/v1/settings/plex/sync     → 获取扫描状态
POST /api/v1/settings/plex/sync     → 启动 / 取消扫描
  body: { cancel?: boolean; start?: boolean }
```

### 6.2 前端轮询机制

前端使用 **SWR (stale-while-revalidate)** 库做轮询，不是 WebSocket 推送：

**Plex 设置页** (`src/components/Settings/SettingsPlex.tsx:127-132`)：
```typescript
const { data: dataSync, mutate: revalidateSync } = useSWR<SyncStatus>(
  '/api/v1/settings/plex/sync',
  {
    refreshInterval: 1000,  // 每 1 秒轮询一次
  }
);
```

**Jobs & Cache 页面** (`src/components/Settings/SettingsJobsCache/index.tsx`)：
- 显示所有定时任务的状态
- 同样使用 SWR 轮询（取决于具体实现）

**进度显示内容**：
- 运行中 / 未运行
- 当前正在处理的 library / server
- 已处理数量 / 总数量（可计算百分比）
- 剩余 library 数量

### 6.3 取消扫描的实现

取消是**协作式中断**，不是强杀：
- `cancel()` 只是设置 `this.running = false`
- `loop()` 方法在每一批次开始前检查 `if (!this.running) throw Error('Sync was aborted.')`
- 已在进行中的 Promise.all 批次会跑完当前批

### 6.4 Download Tracker 的高频更新

下载进度（队列中正在下载的项目）有独立的高频更新通道：

- **频率**：每 1 分钟（`download-sync` cron: `0 * * * * *`）
- **数据来源**：Radarr/Sonarr 的 queue API
- **粒度**：精确到每个下载项的 sizeLeft / timeLeft / estimatedCompletionTime
- **用途**：UI 上显示"正在下载"进度条

与扫描器的关系：
- Download Tracker 是轻量队列查询，不修改 Media 状态
- 真正的状态变更（PROCESSING → AVAILABLE）还是靠每日 Radarr/Sonarr 全量扫描

### 6.5 WebPush 实时通知与断开退路

系统没有使用 WebSocket 或 SSE (Server-Sent Events) 做实时推送，而是通过 **Web Push API (RFC 8030)** 实现浏览器级通知：

**WebPush 代理实现** (`server/lib/notifications/agents/webpush.ts`)：
```typescript
// 发送推送（请求级别事件：可用、批准、拒绝、失败等）
await webpush.sendNotification(
  { endpoint, keys: { auth, p256dh } },  // 浏览器订阅信息
  notificationPayload  // 消息体 + pendingRequestsCount badge
);

// 错误处理：410/404 是永久失效，清理订阅
const isPermanentFailure = statusCode === 410 || statusCode === 404;
if (isPermanentFailure) {
  await userPushSubRepository.remove(pushSub);  // 清理无效订阅
}
// 其他状态码（429 限流、5xx 服务端错误）视为临时失败，保留订阅下次重发
```

**没有 WebSocket 的架构选择原因**：
1. **规模化友好**：Web Push 是无状态协议，不需要维持长连接，适合多实例部署
2. **浏览器关闭也能收到**：Web Push 可以在浏览器未打开时触发系统级通知
3. **更新频率不高**：扫描器几分钟跑一次，事件频率秒级轮询足够
4. **后退兼容性**：Web Push 是浏览器标准，比 WebSocket 更稳定

**前端的"断开退路" —— SWR 轮询兜底**：
即使 Web Push 失败（用户未授权通知、浏览器不支持、网络限制），SWR 的 1 秒轮询仍然保证 UI 能在 1 秒内看到最新状态：
- Web Push = 毫秒级通知（优化体验）
- SWR 轮询 = 秒级保底（确保正确性）

**WebPush 的可用性衰减策略**：
- 用户首次访问时浏览器询问是否允许通知
- 用户拒绝后，系统不会重试请求权限，仅依靠 SWR 轮询
- 用户允许后，系统存储 `UserPushSubscription`，包括 `endpoint`（推送服务 URL）和加密密钥
- 如果推送返回 410/404，自动删除订阅记录（用户取消了通知权限）
- 如果推送返回其他错误（429 限流、5xx），保留订阅，下次事件触发时重试（隐式重试）

---

## 七、定时任务频率与调度策略

### 7.1 默认 cron 表达式一览

所有任务的默认频率定义在 `server/lib/settings/index.ts:569-608`：

| Job ID | 默认 cron | 频率 | 说明 |
|--------|-----------|------|------|
| `plex-recently-added-scan` | `0 */5 * * * *` | 每 5 分钟 | Plex 增量扫描 |
| `jellyfin-recently-added-scan` | `0 */5 * * * *` | 每 5 分钟 | Jellyfin 增量扫描 |
| `download-sync` | `0 * * * * *` | 每 1 分钟 | 下载队列同步 |
| `download-sync-reset` | `0 0 1 * * *` | 每日 01:00 | 重置下载追踪器 |
| `plex-full-scan` | `0 0 3 * * *` | 每日 03:00 | Plex 全库扫描 |
| `jellyfin-full-scan` | `0 0 3 * * *` | 每日 03:00 | Jellyfin 全库扫描 |
| `radarr-scan` | `0 0 4 * * *` | 每日 04:00 | Radarr 全量扫描 |
| `sonarr-scan` | `0 30 4 * * *` | 每日 04:30 | Sonarr 全量扫描 |
| `availability-sync` | `0 0 5 * * *` | 每日 05:00 | 可用性反向校验 |
| `plex-refresh-token` | `0 0 5 * * *` | 每日 05:00 | Plex token 刷新 |
| `image-cache-cleanup` | `0 0 5 * * *` | 每日 05:00 | 图片缓存清理 |
| `plex-watchlist-sync` | `0 */3 * * * *` | 每 3 分钟 | Plex 监视列表同步 |
| `process-blocklisted-tags` | `0 30 1 */7 * *` | 每 7 天 01:30 | 黑名单标签处理 |

**调度时间分布设计**：
- 凌晨 3:00 开始媒体库全量扫描
- 4:00 开始 Radarr 扫描（媒体库扫完后再扫下载器，时间上错开）
- 4:30 Sonarr 扫描（与 Radarr 也错开，避免同时打满 IO）
- 5:00 轻量级维护任务（token 刷新、缓存清理、可用性校验）

### 7.2 调度的可配置性

用户可以在 UI 的 Settings → Jobs 页面修改每个任务的频率：

- 修改后调用 `rescheduleJob()` 立即生效（`routes/settings/index.ts` 中导入）
- 频率通过下拉框选择（秒/分/时/天），不是手动写 cron
- 前端根据选择动态拼出 cron 表达式

### 7.3 无 backoff / 重试策略

**重要：系统没有失败重试 / backoff 机制**：

- 扫描失败了就是失败了，等下一个调度周期自动重跑
- 没有指数退避，没有失败计数，没有告警
- 设计假设：扫描是幂等的，漏掉一次不会有严重后果，下次补上即可

**sessionId 防重入**：
- `startRun()` 生成新的 sessionId
- `loop()` 每批次校验 `this.sessionId !== sessionId`，不一致则抛出
- 防止上一次还没跑完下一次调度又启动了

### 7.4 node-cron 的 backoff 行为与最大上限

**node-schedule 库的原生特性**：
- 使用的是 `node-schedule@2.1.1`（`package.json:76`），不是 `node-cron`
- `node-schedule` 是**基于时间点**的调度，不是基于间隔的
- 这意味着：如果一个任务执行时间超过了 cron 间隔，下一次调度**不会重叠**，而是等当前任务结束后再调度下一次
- 没有指数退避，没有失败重试，没有错误计数

**实际的"隐式 backoff"**：
扫描器的 `run()` 方法本身是同步触发 + 异步执行的模式：
```typescript
// schedule.ts:51
plexRecentScanner.run();  // 立即返回，不等待完成

// baseScanner.ts:646
async run() {
  const sessionId = uuid();
  this.startRun(sessionId);  // 设置 running = true
  
  // ... 实际扫描逻辑 ...
  
  this.endRun(sessionId);  // 设置 running = false
}
```

**最大并发上限 = 1**：
这是整个扫描系统最关键的隐式限制：
- `BaseScanner` 是单例模式（`export const plexRecentScanner = new PlexScanner()`）
- `startRun()` 开头检查 `if (this.running) return;`
- 所以即使 cron 每 5 分钟触发一次，但只要上一次扫描还在跑，本次触发直接返回
- 这形成了一个天然的背压机制：扫描越慢，实际运行频率越低
- 对于超大型媒体库，一次全量扫描可能跑几小时，cron 触发会被持续跳过，直到本次完成

**与真正 exponential backoff 的差异**：

| 特性 | 系统当前行为 | 标准 exponential backoff |
|------|-------------|--------------------------|
| 失败重试 | 无，等下一个调度周期 | 有，延迟递增重试 |
| 最大重试次数 | 无，无限期重试 | 通常 3~5 次上限 |
| 最大延迟 | = cron 间隔（固定） | 通常 30s~1h 指数增长 |
| 成功后重置 | 每次调度都是独立的 | 成功后重置重试计数 |

**API 层面的 rate limit** (`server/api/externalapi.ts:17-50`)：
虽然扫描器本身没有 backoff，但底层 API 调用有硬限流：
```typescript
// 通过 axios-rate-limit 包装
this.axios = rateLimit(this.axios, {
  maxRequests: options.rateLimit.maxRequests,  // 窗口内最大请求数
  maxRPS: options.rateLimit.maxRPS,             // 每秒最大请求数
});

// TMDB 配置: maxRPS=40
// TVDB 配置: maxRequests=30, maxRPS=1
// Plex/Jellyfin/Radarr/Sonarr 无内置限流，靠批次间 4s 间隔限制
```

---

## 八、扫描资源占用与并发控制

### 8.1 批处理与节流

**BaseScanner 的批处理循环** (`baseScanner.ts:687-725`)：

```
每批次:
  ├─ bundleSize = 50 条 (Plex/Radarr/Sonarr 都是这个默认值)
  ├─ 批次内: Promise.all 并行处理 50 条
  └─ 批次间: setTimeout 间隔 4 秒 (UPDATE_RATE)
```

**两个关键参数**：
- `bundleSize`（`baseScanner.ts:57`）：每批处理条数
- `updateRate`（`baseScanner.ts:13, 82`）：批次间隔毫秒数，默认 4000ms = 4s

**设计意图**：
- 避免一次性发大量请求打垮 Plex/Jellyfin/Radarr/Sonarr 服务器
- 4 秒间隔给外部服务器喘息时间，也避免触发 API 限流

### 8.2 并发层级分析

整个扫描系统有三层并发控制：

**第一层：单媒体串行 (AsyncLock)**
- 锁粒度：`tmdbId` 维度
- 位置：`processMovie()` 和 `processShow()` 内部
- 作用：四种扫描器可能同时处理同一部电影，串行化保证不会重复创建 Media 记录

**第二层：批次内并行 (Promise.all)**
- 粒度：每批次 bundleSize 条（50）
- 位置：`processItems()` → `Promise.all(items.map(processFn))`
- 作用：同一批次内的不同媒体可以并行处理（主要是并行发 API 请求查 TMDB）

**第三层：批次间串行 (setTimeout)**
- 粒度：批次之间
- 位置：`loop()` 递归调用前的 setTimeout
- 作用：整体流量整形，避免持续高压

### 8.3 资源占用评估

**内存占用**：
- 全量扫描时 `this.items` 会加载整个库的所有条目到内存
- Plex 库有几千部到几万部都很常见
- Jellyfin 通过 getLibraryContents 一次性拉取（非分页）

**数据库压力**：
- 每条媒体至少一次 findOne + 可能一次 save
- 50 条并发 → 数据库连接池压力可控
- SQLite 场景下因为是文件锁，并发写入实际上也是串行的

**API 请求量**：
- ID 解析阶段如果缓存 miss，每条会触发 1-N 次 TMDB API 调用
- 全量扫描首次运行时 API 调用量最大
- 有 plexguid 缓存后，增量扫描基本不触发 TMDB API

### 8.4 Jellyfin 增量扫描的特殊限制

Jellyfin `getRecentlyAdded` 只返回 12 条（`Limit=12`），不是时间窗口：
- 意味着如果 5 分钟内入库超过 12 条，增量扫描会漏掉
- 只能等每日全量扫描补上
- 这是 Jellyfin API 的限制（Latest 接口没有时间筛选参数）

### 8.5 p-limit 并发控制与配置暴露

**重要澄清：系统没有使用 p-limit 包**：
- `package.json` 中没有 `p-limit` 依赖
- 扫描器的并发控制全部通过**"批次大小 + 批次间 sleep"** 的手动方式实现
- `p-limit@2.3.0` 和 `p-limit@3.1.0` 存在于 `pnpm-lock.yaml` 中，但是**其他依赖的传递依赖**（如 `typeorm` 内部可能用到），不是本项目直接使用

**实际的并发控制机制**：

| 控制层面 | 实现方式 | 可配置性 |
|----------|---------|----------|
| 同媒体串行 | `AsyncLock` 按 tmdbId 排队 | 不可配置 |
| 批次大小 | `BaseScanner.bundleSize = 50` | 硬编码，不可配置 |
| 批次间隔 | `BaseScanner.updateRate = 4000ms` | 硬编码，不可配置 |
| 单扫描器并行 | `BaseScanner.running` 标志，最大 1 | 硬编码，不可配置 |
| API 请求限流 | `axios-rate-limit`（TMDB maxRPS=40, TVDB maxRPS=1） | 硬编码，不可配置 |

**所有并发参数都是硬编码**：
```typescript
// baseScanner.ts:57 - bundleSize
protected bundleSize = 50;

// baseScanner.ts:13,82 - updateRate
protected updateRate = 4000;

// server/api/themoviedb/index.ts:142 - TMDB 限流
rateLimit: {
  maxRPS: 40,
  maxRequests: 100,
}

// server/api/tvdb/index.ts:57 - TVDB 限流
rateLimit: {
  maxRequests: 30,
  maxRPS: 1,
}
```

**Session store 的 concurrency 控制**：
唯一"暴露"的并发相关配置是 TypeORM session store 的 `cleanupLimit`（`server/index.ts:222`）：
```typescript
store: new TypeormStore({
  cleanupLimit: 2,  // 每次清理过期 session 的数量
  ttl: 60 * 60 * 24 * 30,
}).connect(sessionRespository)
```
这不是扫描器的配置，但体现了系统对并发写入的保守态度。

**为什么不做配置暴露**：
1. 扫描器是后台任务，调优价值不大
2. 硬编码的 50 + 4s 组合对大多数媒体库已经足够
3. 暴露配置会增加测试负担和用户困惑
4. 真需要调优的用户可以 fork 代码修改

---

## 九、可用性反向校验与异常修正

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

## 十、异常与边界场景

### 10.1 竞态保护：AsyncLock

`AsyncLock.dispatch(tmdbId, async () => ...)` 保证同一 `tmdbId` 不会被并行处理：
- Plex/Jellyfin/Radarr/Sonarr 四种扫描器可能几乎同时运行
- 都调用同一 processMovie() 时会排队执行，防止数据库脏写

实现细节 (`utils/asyncLock.ts`)：
- 基于 EventEmitter 的等待队列
- 每个 key 一个布尔锁 + 事件监听
- `setMaxListeners(0)` 取消监听器数量限制
- 用 `setImmediate` 释放事件，避免同步释放导致的栈溢出

### 10.2 HAMA 动漫特殊处理

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

### 10.3 AniDB 动漫季合并 (Jellyfin)

Jellyfin 中 AniDB 条目常把一部动漫的多季拆成多个 Series。`processedAnidbSeason` 跨 Jellyfin LibraryItem 聚合 episode 计数，防止 TMDB 上同一季的 episode count 被反复覆盖为不完整数字。

---

## 十一、关键文件索引

| 文件路径 | 职责 |
|----------|------|
| `server/lib/scanners/baseScanner.ts` | 抽象基类，processMovie/processShow 判定矩阵 |
| `server/lib/scanners/plex/index.ts` | Plex 扫描、7 种 Agent ID 解析 |
| `server/lib/scanners/jellyfin/index.ts` | Jellyfin/Emby 扫描、AniDB 聚合 |
| `server/lib/scanners/radarr/index.ts` | Radarr 扫描、多实例去重、orphan 清理 |
| `server/lib/scanners/sonarr/index.ts` | Sonarr 扫描、多实例去重、orphan 清理 |
| `server/job/schedule.ts` | cron 调度器注册、所有定时任务入口 |
| `server/lib/settings/index.ts` | 默认配置、jobs cron 表达式定义 |
| `server/api/plexapi.ts` | Plex HTTP API 封装 |
| `server/api/jellyfin.ts` | Jellyfin/Emby HTTP API 封装 |
| `server/api/externalapi.ts` | 外部 API 基类、axios-rate-limit 限流 |
| `server/api/rating/rottentomatoes.ts` | Jaro 相似度匹配、同名歧义解决参考 |
| `server/constants/server.ts` | MediaServerType 枚举、自定义 server type 扩展入口 |
| `server/entity/MediaRequest.ts` | 请求创建 (MediaRequest.request)、重复检测 |
| `server/subscriber/MediaRequestSubscriber.ts` | 批准后 Radarr/Sonarr 入队 + 状态回填 |
| `server/lib/availabilitySync.ts` | 反向可用性校验防误删 |
| `server/lib/downloadtracker.ts` | 下载队列实时追踪（每分钟） |
| `server/lib/notifications/index.ts` | Notification 枚举与管理器 |
| `server/lib/notifications/agents/webpush.ts` | WebPush 推送实现、410/404 永久失效清理 |
| `server/lib/notifications/agents/agent.ts` | NotificationAgent 抽象基类 |
| `server/utils/asyncLock.ts` | 单媒体维度的并发锁 |
| `server/routes/settings/index.ts` | 扫描状态查询 / 启动 / 取消 API |
| `server/routes/request.ts` | 请求 CRUD / retry / decline API 路由 |
| `src/components/Settings/SettingsPlex.tsx` | Plex 设置页 + 扫描进度 UI (SWR 1s 轮询) |
| `src/components/Settings/SettingsJobsCache/index.tsx` | 定时任务管理页 |
| `src/components/Setup/index.tsx` | Setup 页面、server type 选择 UI |
