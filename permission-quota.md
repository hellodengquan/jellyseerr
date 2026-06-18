# 媒体请求服务：权限位与配额联动机制分析

## 一、权限位系统

### 1.1 权限位定义（位掩码设计）

权限位定义在 `server/lib/permissions.ts:1-32`，采用二进制位掩码（Bitmask）方式存储，每个权限占用一个独立的二进制位。这种设计允许在单个整数值中高效存储多个权限状态。

| 权限位 | 数值 | 说明 | 层级 |
|--------|------|------|------|
| NONE | 0 | 无权限 | - |
| ADMIN | 2 | 管理员，**绕过所有其他权限检查** | 最高 |
| MANAGE_SETTINGS | 4 | 管理设置 | 系统级 |
| MANAGE_USERS | 8 | 管理用户（配额豁免） | 系统级 |
| MANAGE_REQUESTS | 16 | 管理请求（自动批准 + 管理所有请求） | 请求管理级 |
| REQUEST | 32 | 提交非4K媒体请求（通用） | 请求基础级 |
| REQUEST_MOVIE | 262144 | 提交非4K电影请求 | 请求细分级 |
| REQUEST_TV | 524288 | 提交非4K剧集请求 | 请求细分级 |
| REQUEST_4K | 1024 | 提交4K媒体请求（通用） | 请求基础级 |
| REQUEST_4K_MOVIE | 2048 | 提交4K电影请求 | 请求细分级 |
| REQUEST_4K_TV | 4096 | 提交4K剧集请求 | 请求细分级 |
| AUTO_APPROVE | 128 | 自动批准所有非4K请求 | 自动批准级 |
| AUTO_APPROVE_MOVIE | 256 | 自动批准非4K电影 | 自动批准细分级 |
| AUTO_APPROVE_TV | 512 | 自动批准非4K剧集 | 自动批准细分级 |
| AUTO_APPROVE_4K | 32768 | 自动批准所有4K请求 | 自动批准级 |
| AUTO_APPROVE_4K_MOVIE | 65536 | 自动批准4K电影 | 自动批准细分级 |
| AUTO_APPROVE_4K_TV | 131072 | 自动批准4K剧集 | 自动批准细分级 |
| REQUEST_ADVANCED | 8192 | 高级请求选项（指定服务器/配置） | 扩展功能级 |
| REQUEST_VIEW | 16384 | 查看其他用户的请求 | 视图级 |

### 1.2 权限层级关系

权限检查采用「通用权限 + 细分权限」的 OR 逻辑（满足其一即可）：

```
请求权限层级（以非4K电影为例）：
├── ADMIN (2)                → 无条件通过
├── REQUEST (32)             → 通用请求权限（覆盖电影和剧集）
└── REQUEST_MOVIE (262144)   → 仅电影请求权限

4K剧集请求层级：
├── ADMIN (2)
├── REQUEST_4K (1024)        → 通用4K请求权限
└── REQUEST_4K_TV (4096)     → 仅4K剧集请求权限
```

### 1.3 权限检查时机（请求处理流程）

所有权限检查集中在 `server/entity/MediaRequest.ts:47-120` 的 `MediaRequest.request()` 静态方法中，按以下顺序执行：

**检查顺序（先到先失败原则）：**

1. **请求用户覆盖检查**（第 60-74 行）
   - 时机：请求提交后，最先检查
   - 检查：如果指定了 `userId`（代用户提交），必须拥有 `MANAGE_USERS` 或 `MANAGE_REQUESTS` 权限
   - 失败：抛出 `RequestPermissionError`

2. **电影请求权限检查**（第 80-95 行）
   - 时机：用户覆盖检查通过后
   - 非4K：检查 `[REQUEST, REQUEST_MOVIE]`（OR逻辑）
   - 4K：检查 `[REQUEST_4K, REQUEST_4K_MOVIE]`（OR逻辑）
   - 失败：抛出 `RequestPermissionError`

3. **剧集请求权限检查**（第 96-112 行）
   - 时机：电影类型检查通过或非电影类型
   - 非4K：检查 `[REQUEST, REQUEST_TV]`（OR逻辑）
   - 4K：检查 `[REQUEST_4K, REQUEST_4K_TV]`（OR逻辑）
   - 失败：抛出 `RequestPermissionError`

4. **配额检查**（第 114-120 行）
   - 时机：**所有权限检查通过之后**，实际创建请求之前
   - 失败：抛出 `QuotaRestrictedError`

5. **自动批准权限检查**（第 348-375 行）
   - 时机：请求对象构建时（决定 `status` 字段）
   - 检查权限：`[AUTO_APPROVE_*, AUTO_APPROVE_*_MOVIE/TV, MANAGE_REQUESTS]`（OR逻辑）
   - 效果：通过则状态为 `APPROVED`，否则为 `PENDING`

权限检查核心实现在 `server/lib/permissions.ts:47-74`：
- ADMIN 权限（值为 2）会**无条件通过所有检查**
- 支持 AND（必须全部拥有）和 OR（拥有其一即可）两种组合模式

---

## 二、配额计量机制

### 2.1 计量维度：按用户（而非请求队列）

配额计量完全基于**发起请求的用户个体**，与请求队列（待处理/处理中）无关。核心逻辑在 `server/entity/User.ts:273-372` 的 `getQuota()` 方法中。

**关键代码定位：**
- 电影配额统计：`User.ts:293-304`
- 剧集配额统计：`User.ts:336-348`

### 2.2 配额配置

配额有两层配置，用户级覆盖全局级：

| 配置层级 | 字段 | 说明 | 默认值 |
|---------|------|------|--------|
| 全局默认 | `settings.main.defaultQuotas.movie.quotaLimit` | 电影请求数量上限 | 0（无限制） |
| 全局默认 | `settings.main.defaultQuotas.movie.quotaDays` | 电影配额统计周期（天） | 7 |
| 全局默认 | `settings.main.defaultQuotas.tv.quotaLimit` | 剧集请求数量上限 | 0（无限制） |
| 全局默认 | `settings.main.defaultQuotas.tv.quotaDays` | 剧集配额统计周期（天） | 7 |
| 用户覆盖 | `User.movieQuotaLimit` | 用户级电影上限 | `null`（使用全局） |
| 用户覆盖 | `User.movieQuotaDays` | 用户级电影周期 | `null`（使用全局） |
| 用户覆盖 | `User.tvQuotaLimit` | 用户级剧集上限 | `null`（使用全局） |
| 用户覆盖 | `User.tvQuotaDays` | 用户级剧集周期 | `null`（使用全局） |

**豁免规则**（`User.ts:278-280`）：
- 拥有 `MANAGE_USERS` 权限的用户，配额限制强制设为 0（无限制），即**不受配额约束**

### 2.3 计量算法

**电影配额（按请求数计数）：**

```sql
SELECT COUNT(*) FROM media_request
WHERE requestedBy_id = :userId
  AND type = 'MOVIE'
  AND status != 'DECLINED'
  [AND createdAt > :quotaStartDate]  -- 如果quotaDays > 0
```

统计条件：
- 按 `requestedBy.id` 过滤当前用户
- 排除状态为 DECLINED（已拒绝）的请求
- 仅在配置了 `quotaDays` 时才限制时间窗口
- 一部电影 = 1 次计数（不论季数，电影没有多季概念）

**剧集配额（按季数计数）：**

```sql
SELECT request.*, (
  SELECT COUNT(season.id) FROM season_request season
  WHERE season.request_id = request.id
) as seasonCount
FROM media_request request
WHERE request.requestedBy_id = :userId
  AND request.type = 'TV'
  AND request.status != 'DECLINED'
  [AND request.createdAt > :quotaStartDate]
-- 然后对所有请求的 seasonCount 求和
```

统计条件：
- 同样按用户过滤，排除已拒绝请求
- **关键差异**：剧集不是按「请求次数」计数，而是按「请求的季数总和」计数
  - 例：一次请求 3 季 = 计数 3，而非 1
- 这样设计更公平，因为一部多季剧集占用的资源远多于单季

### 2.4 配额状态计算

配额计算返回 `QuotaResponse`（定义在 `server/interfaces/api/userInterfaces.ts:15-28`）：

```typescript
interface QuotaStatus {
  days?: number;        // 统计周期（天数），0 = 无时间限制
  limit?: number;       // 上限数量，0 = 无限制
  used: number;         // 已使用数量
  remaining?: number;   // 剩余数量 = max(0, limit - used) （limit为0时不返回）
  restricted: boolean;  // 是否已受限（limit > 0 && remaining <= 0）
}
```

`restricted` 是后端检查的关键字段：当 `restricted === true` 时，拒绝创建新请求。

---

## 三、权限不足 vs 配额超出：用户反馈差异

### 3.1 后端响应差异

两种错误在 `server/routes/request.ts:321-324` 被统一映射，但错误信息有明确区分：

| 场景 | 异常类 | HTTP状态码 | 错误消息示例 | 语义定位 |
|------|--------|-----------|-------------|---------|
| 无请求权限 | `RequestPermissionError` | 403 Forbidden | `You do not have permission to make 4K movie requests.` | **认证/授权失败** — 用户根本不允许执行该操作 |
| 电影配额耗尽 | `QuotaRestrictedError` | 403 Forbidden | `Movie Quota exceeded.` | **资源限制** — 用户有权限但额度用完 |
| 剧集配额耗尽 | `QuotaRestrictedError` | 403 Forbidden | `Series Quota exceeded.` | **资源限制** — 用户有权限但额度用完 |
| 代用户提交无权限 | `RequestPermissionError` | 403 Forbidden | `You do not have permission to modify the request user.` | **认证/授权失败** |
| 被拉黑媒体 | `BlocklistedMediaError` | 403 Forbidden | `This media is blocklisted.` | **媒体级限制** |
| 重复请求 | `DuplicateMediaRequestError` | 409 Conflict | `Request for this media already exists.` | **状态冲突** |

**注意**：虽然权限不足和配额超出都返回 403，但异常类和消息文本不同，前端可据此区分处理。

### 3.2 前端反馈差异

**（1）配额 — 预先可见、渐进式提示**

前端在请求模态框打开时就会通过 SWR 拉取 `/api/v1/user/{id}/quota` 接口（`TvRequestModal.tsx:60-68`，`CollectionRequestModal.tsx:60-67`），并渲染 `QuotaDisplay` 组件：

- **进度条展示**：`QuotaDisplay/index.tsx:61-67` 用圆形进度条显示百分比，使用热度颜色（越接近0越红）
- **实时剩余数量**：显示「还剩 X 个电影/季请求」（第 76-84 行）
- **超量预选预警**：当选择的季数/电影数超过剩余额度时，显示红色的「Not enough season requests remaining」提示，同时展开详细说明需要多少额度（第 76-108 行）
- **全选按钮禁用**：配额不足时自动禁用「全选」功能（`TvRequestModal.tsx:305-327`）
- **详情面板**：点击可展开查看配额规则（X 部电影/季 每 Y 天）和个人资料页链接

**（2）权限不足 — 静默隐藏、按钮级控制**

权限不足的前端反馈主要体现在 UI 层面的功能隐藏：
- 请求按钮是否可点击取决于权限检查
- 4K 相关选项仅在拥有 4K 请求权限时显示
- 高级请求选项（服务器选择、配置文件等）需要 `REQUEST_ADVANCED` 权限
- 权限不足通常不会弹出错误提示，而是对应功能直接不可见/不可用

**（3）提交失败 — 统一 Toast 但信息丢失**

如果绕过前端预检查直接提交（或后端状态变化），前端在 `TvRequestModal.tsx:225-229` 和类似位置捕获异常：

```typescript
catch {
  addToast(intl.formatMessage(messages.requesterror), {
    appearance: 'error',
    autoDismiss: true,
  });
}
```

**当前局限**：前端 catch 块直接丢弃了原始错误信息，统一显示「Something went wrong while submitting the request.」，用户无法区分是权限问题、配额问题还是其他错误。后端返回的详细错误消息（如 `Movie Quota exceeded.`）在这一层丢失了。

### 3.3 检查时序上的用户体验差异

| 检查类型 | 用户何时感知 | 交互方式 | 体验评价 |
|---------|-------------|---------|---------|
| 配额检查 | 打开模态框时立即看到 | 进度条、剩余数、禁用全选 | ✅ 主动、提前、可规划 |
| 权限检查（按钮级） | 交互前 — 按钮/选项不存在或灰掉 | UI 隐藏/禁用 | ✅ 符合预期 |
| 权限检查（提交级） | 提交后 Toast 报错 | 统一错误提示 | ⚠️ 需改进：可显示具体原因 |
| 配额检查（提交级兜底） | 提交后 Toast 报错 | 统一错误提示 | ⚠️ 需改进：可显示「配额已用完，请等待X天」 |

---

## 四、权限与配额的联动关系总结

### 4.1 执行顺序流程图

```
用户发起请求
    │
    ▼
┌─────────────────────────────┐
│ 1. 请求用户覆盖权限检查      │  是否有 MANAGE_USERS / MANAGE_REQUESTS
│    (RequestPermissionError) │
└─────────────────────────────┘
    │ 通过
    ▼
┌─────────────────────────────┐
│ 2. 媒体类型请求权限检查      │  REQUEST + REQUEST_MOVIE/TV（OR）
│    (RequestPermissionError) │  REQUEST_4K + REQUEST_4K_MOVIE/TV（OR）
└─────────────────────────────┘
    │ 通过
    ▼
┌─────────────────────────────┐
│ 3. 配额检查                  │  User.getQuota() → restricted === true?
│    (QuotaRestrictedError)    │  注意：MANAGE_USERS 用户豁免配额
└─────────────────────────────┘
    │ 通过
    ▼
┌─────────────────────────────┐
│ 4. 自动批准权限检查          │  AUTO_APPROVE + MANAGE_REQUESTS 等
│    (决定请求状态)            │  通过 → APPROVED / 不通过 → PENDING
└─────────────────────────────┘
    │
    ▼
 创建 MediaRequest 实体
```

### 4.2 关键联动点

1. **MANAGE_USERS 双重豁免**：
   - 权限检查层面：可以为其他用户代提交请求
   - 配额检查层面：强制 `quotaLimit = 0`（无限制），不受配额约束

2. **MANAGE_REQUESTS 双重作用**：
   - 权限检查层面：可以管理（修改/删除）所有用户的请求，也可代用户提交
   - 自动批准层面：拥有此权限的用户，自己的所有请求会被**自动批准**

3. **通用权限与细分权限的 OR 组合**：
   - 权限设计采用「宽权限 + 细权限」策略，方便管理员灵活分配
   - 例：授予 `REQUEST` 权限 = 同时授予电影和剧集的请求权限，无需分别设置

4. **配额的「已拒绝请求不计入」策略**：
   - DECLINED 状态的请求不占用配额
   - PENDING、APPROVED、PROCESSING、AVAILABLE、COMPLETED 均占用配额
   - 这意味着管理员拒绝请求后，用户的配额会自动「退还」

---

## 五、家庭组共享场景下的配额边界分析

### 5.1 Seerr 无「家庭组配额汇总」概念：完全按独立用户计量

**关键结论：Seerr 不实现任何 Plex Home / Jellyfin 家庭组的配额层级汇总逻辑。**

用户类型定义在 `server/constants/user.ts:1-6`，仅有四种独立账户类型：

```typescript
enum UserType {
  PLEX = 1,      // Plex 登录用户（含 Plex Home 成员）
  LOCAL = 2,     // 本地创建用户
  JELLYFIN = 3,  // Jellyfin 登录用户
  EMBY = 4,      // Emby 登录用户
}
```

配额计量逻辑 `User.getQuota()` 完全基于 `requestedBy.id`（当前用户的 Seerr 内部 ID）进行统计，与 Plex/Jellyfin 服务端的家庭组层级、家长账户 ID、成员归属关系**完全解耦**。

**证据链：**
1. `User.ts:293-304` 电影配额查询的 WHERE 条件只有 `requestedBy: { id: this.id }`
2. `User.ts:317-348` 剧集配额查询同样只过滤 `requestedBy.id = :userId`
3. `plextv.ts:67-68` 中虽然 Plex API 返回了 `home` 字段标识用户是否属于 Plex Home，但该字段**仅用于服务端连接检测**，从未参与配额或权限计算
4. Plex 用户导入逻辑 `routes/user/index.ts:657-722` 中，每个 Plex 用户（无论家庭组成员还是好友共享）均被创建为拥有独立 `id` 的 Seerr 用户行

**实际效果示例：**
- Plex Home 家长账户（User ID: 1）配额 = 统计 `requestedBy.id = 1` 的请求
- Plex Home 孩子账户（User ID: 5）配额 = 统计 `requestedBy.id = 5` 的请求
- 两者**互不影响、各自独立**，不存在汇总到家长或家庭共享配额池的机制

### 5.2 家长撤销成员资格时的配额处理

Seerr 中有三种与「成员资格撤销」相关的操作，配额处理行为各不同：

#### 场景 A：解除 Plex/Jellyfin 账户关联（Unlink Linked Account）

代码位置：`server/routes/user/usersettings.ts:311-360`（Plex）、`460-515`（Jellyfin）

```typescript
// Plex 解除关联核心逻辑
user.userType = UserType.LOCAL;    // 转换为本地用户
user.plexId = null;                // 清除 Plex ID
user.plexUsername = null;
user.plexToken = null;
await userRepository.save(user);
```

**配额处理：完全不触动已用配额。**
- 用户的 `id`（主键）保持不变
- `movieQuotaLimit / tvQuotaLimit / Days` 等配额字段保持原值
- 历史 `MediaRequest` 记录的 `requestedBy.id` 关联不变
- 效果：用户「降级」为本地账户后，配额的已用值、剩余值与解除前**完全一致**，配额额度继续在原时间窗口内累计

**前置条件保护**（`usersettings.ts:343-346`）：
- 要求用户已设置 `email` 和 `password`，防止解除后无法登录
- id = 1 的主管理员禁止解除媒体服务器关联

#### 场景 B：从用户列表删除用户（Delete User）

代码位置：`server/routes/user/index.ts:593-655`

```typescript
// 先手动删除所有请求（触发 Subscriber 级联更新 Media 状态）
await requestRepository.remove(user.requests, { chunk: user.requests.length / 1000 });
// 再删除用户行
await userRepository.delete(user.id);
```

**配额处理：已用配额随用户记录一起被删除。**
- 用户所有的 `MediaRequest` 记录被显式删除（而非依赖数据库 CASCADE，因为 CASCADE 不会触发 TypeORM 的 `afterRemove` Subscriber）
- 用户行被 DELETE，配额字段一并消失
- 对其他用户的配额无任何影响（因为按 id 独立计量）
- `MediaRequestSubscriber.afterRemove`（`subscriber/MediaRequestSubscriber.ts:1075-1084`）会将对应 Media 状态重置为 UNKNOWN/DELETED，但不触及任何配额统计

**前置保护**（`routes/user/index.ts:609-621`）：
- id = 1 的主管理员不可删除
- 非 Owner（id ≠ 1）不能删除 ADMIN 用户

#### 场景 C：Plex/Jellyfin 服务端撤销共享但用户未在 Seerr 中被删除

这是最常见的「软撤销」场景：家长在 plex.tv 上移除了家庭成员的服务器访问权限，但 Seerr 数据库中仍然保留该用户行。

**配额处理：无任何变化，配额计量继续独立进行。**
- Seerr 不主动与 Plex/Jellyfin 同步成员资格状态
- 该用户的后续请求在认证阶段（auth middleware）会因 Plex 令牌失效或无服务器访问权限而失败
- 但已统计的配额不会被清零、退还或转移
- 管理员需要手动执行「删除用户」或「禁用用户」操作

### 5.3 配额边界总结表

| 操作场景 | 用户 Seerr ID | 历史请求记录 | 已用配额值 | 剩余配额值 |
|---------|-------------|------------|----------|----------|
| Plex Home 成员提交请求 | ✅ 保留（独立 id） | ✅ 计入该用户 id | ✅ 独立累计，不汇总家长 | ✅ 独立扣减 |
| 解除 Plex 关联（转本地用户） | ✅ 不变 | ✅ 保留，关联不变 | ✅ 不变，继续累计 | ✅ 不变 |
| 删除用户 | ❌ 被删除 | ❌ 被级联删除 | ❌ 随用户消失 | ❌ 随用户消失 |
| Plex 服务端撤销共享 | ✅ 保留 | ✅ 保留 | ✅ 保留（但用户可能无法再提交新请求） | ✅ 保留 |
| 管理员拒绝某条请求 | ✅ 不变 | ✅ 保留（状态 DECLINED） | ⬇️ 自动退还（DECLINED 不计入） | ⬆️ 自动增加 |

---

## 六、上游故障容错：前端对配额 API 错误的处理行为

### 6.1 配额 API 可能失败的场景

后端配额接口 `GET /api/v1/user/:id/quota`（`routes/user/index.ts:801-832`）可能返回的错误：

| HTTP 状态 | 触发条件 |
|----------|---------|
| 403 Forbidden | 查看他人配额但不同时拥有 `MANAGE_USERS` **且** `MANAGE_REQUESTS` 权限 |
| 404 Not Found | 用户 id 不存在 |
| 500 Internal Server Error | 数据库查询异常、TypeORM 错误等 |

此外还包括：网络超时、DNS 失败、服务端重启导致的连接失败。

### 6.2 各组件的容错行为分析

#### （1）MiniQuotaDisplay（用户下拉菜单中的配额预览）

代码位置：`src/components/Layout/UserDropdown/MiniQuotaDisplay/index.tsx:21-31`

```typescript
const { data, error } = useSWR<QuotaResponse>(`/api/v1/user/${userId}/quota`);

if (error) {
  return null;  // 静默降级：有错误直接不渲染
}

if (!data && !error) {
  return <SmallLoadingSpinner />;  // 加载中显示旋转指示器
}
```

**容错策略：静默隐藏（Silent Degradation）**
- ✅ 403 / 404 / 500 / 网络错误：**直接返回 null，不显示任何内容**，用户看到的是下拉菜单中缺失了配额区块，没有 Toast、没有错误提示
- ✅ 加载中：显示小型加载 Spinner
- ✅ 无配额限制（limit 均为 0）：同样不渲染（第 35 行判断）
- **无本地缓存回退**：SWR 默认配置下如果有缓存数据会显示缓存，但首次加载失败就什么都不显示

#### （2）MovieRequestModal / CollectionRequestModal（电影请求模态框）

代码位置：
- `MovieRequestModal.tsx:66-71`（SWR 配额获取）
- `MovieRequestModal.tsx:319`（Modal loading 判断）
- `MovieRequestModal.tsx:323`（okDisabled 判断）

```typescript
const { data: quota } = useSWR<QuotaResponse>(
  user && (!requestOverrides?.user?.id || hasPermission(Permission.MANAGE_USERS))
    ? `/api/v1/user/${requestOverrides?.user?.id ?? user.id}/quota`
    : null
);

// Modal 级别的加载判断
<Modal
  loading={(!data && !error) || !quota}  // 配额没拿到 → 整个 Modal 保持加载状态
  ...
  okDisabled={isUpdating || quota?.movie.restricted}  // quota 为 undefined 时不限制
>
```

**容错策略：持续加载（Blocking Loading）**
- ✅ 加载中（`!quota` 且无 error）：Modal 持续显示 loading 遮罩，用户无法交互
- ⚠️ 403 / 500 等错误：SWR 返回 `error`，但组件**只解构了 `data: quota`**，未使用 `error` 变量
  - 结果：`quota` 为 `undefined`
  - Modal 的 `loading={(!data && !error) || !quota}` → `!quota = true` → **持续卡在加载状态**
  - `okDisabled={... || quota?.movie.restricted}` → `undefined?.restricted = undefined` → falsy → 按钮可点击
  - 用户可绕过前端配额限制直接提交（后端仍会检查配额并返回 403，最终触发统一 Toast 报错）
- **无本地缓存回退**：如果之前没加载成功过，SWR 没有缓存数据

CollectionRequestModal 的行为完全一致（`CollectionRequestModal.tsx:267`）。

#### （3）TvRequestModal（剧集请求模态框）

代码位置：`TvRequestModal.tsx:91-96`、`TvRequestModal.tsx:392-393`

```typescript
const { data: quota } = useSWR<QuotaResponse>(
  user && (!requestOverrides?.user?.id || hasPermission(Permission.MANAGE_USERS))
    ? `/api/v1/user/${requestOverrides?.user?.id ?? user.id}/quota`
    : null
);

<Modal
  loading={!data && !error}  // 只等媒体详情加载，不等 quota
  ...
>
```

**容错策略：不阻塞加载 + 静默降级（Non-blocking + Silent）**
- ✅ 加载中：Modal 只等待媒体详情（`!data && !error`），**不等待配额数据**
- ✅ 配额 API 失败：`quota` 为 `undefined`
  - 配额 UI 区块因 `quota` 为空不渲染（QuotaDisplay 组件不显示）
  - `currentlyRemaining`（`TvRequestModal.tsx:98-101`）退化为 `(0) - selected + editingLength`，可能显示负数
  - 「全选」按钮（`TvRequestModal.tsx:305-327`）因 `quota?.tv.limit` 为 undefined，跳过配额限制逻辑
  - 用户仍然可以提交，由后端做最终检查
- ✅ 不会卡住 UI，用户可以继续操作（但没有配额提示）

#### （4）UserProfile（用户资料页配额展示）

代码位置：`src/components/UserProfile/index.tsx:66-75`、`148-153`

```typescript
const { data: quota } = useSWR<QuotaResponse>(
  user && (user.id === currentUser?.id || currentHasPermission(..., { type: 'and' }))
    ? `/api/v1/user/${user.id}/quota`
    : null
);

// 渲染时做短路判断
{quota && (user.id === currentUser?.id || ...) && (
  <QuotaSection ... />
)}
```

**容错策略：静默降级（Silent Degradation）**
- ✅ 403 / 404 / 500：`quota` 为 undefined，`{quota && ...}` 判断直接短路，配额卡片不渲染
- ✅ 无 Toast，无错误提示
- ✅ 用户资料页其他内容（请求历史、通知设置等）正常显示

#### （5）QuotaDisplay 组件本身（纯展示层）

代码位置：`src/components/RequestModal/QuotaDisplay/index.tsx:39-147`

QuotaDisplay 是受控组件，只接收 props，不自己发请求：

```typescript
const QuotaDisplay = ({ quota, mediaType, userOverride, remaining, overLimit }) => {
  // 计算进度时使用多重 nullish fallback
  progress={Math.round(
    ((remaining ?? quota?.remaining ?? 0) / (quota?.limit ?? 1)) * 100
  )}
```

- `quota` 为 `undefined` 时：`remaining ?? undefined ?? 0 = 0`，`limit ?? 1 = 1`，进度显示 0%
- 文本也安全 fallback 为 0，但如果父组件没传 `overLimit`，不会显示红色超量提示

### 6.3 提交阶段的后端兜底与前端 Toast

无论前端配额加载状态如何，提交请求时后端 `MediaRequest.request()`（`entity/MediaRequest.ts:114-120`）始终会做配额检查：

```typescript
const quotas = await requestUser.getQuota();
if (requestBody.mediaType === MediaType.MOVIE && quotas.movie.restricted) {
  throw new QuotaRestrictedError('Movie Quota exceeded.');
}
```

前端在 `MovieRequestModal.tsx:127-131`、`TvRequestModal.tsx:225-229` 的提交 catch 中：

```typescript
catch {
  addToast(intl.formatMessage(messages.requesterror), {
    appearance: 'error',
    autoDismiss: true,
  });
}
```

**问题**：catch 块没有捕获 `error` 参数，后端返回的具体错误消息（如 `Movie Quota exceeded.` vs `You do not have permission...`）被完全丢弃，用户看到的永远是「Something went wrong while submitting the request.」。

### 6.4 容错行为汇总对比

| 组件 | API 失败时的行为 | 是否阻塞 UI | 是否显示 Toast | 是否降级使用缓存 | 后端兜底 |
|-----|----------------|-----------|--------------|----------------|---------|
| MiniQuotaDisplay | 静默不渲染（return null） | ❌ 不阻塞 | ❌ 无 | SWR 有缓存则用缓存 | 不涉及 |
| MovieRequestModal | **持续卡在 loading 状态**，按钮可点击 | ✅ 阻塞 Modal | ❌ 无 | 首次失败无缓存 | ✅ 提交时检查 |
| CollectionRequestModal | **持续卡在 loading 状态**，按钮可点击 | ✅ 阻塞 Modal | ❌ 无 | 首次失败无缓存 | ✅ 提交时检查 |
| TvRequestModal | 不阻塞，配额 UI 隐藏，全选跳过配额检查 | ❌ 不阻塞 | ❌ 无 | SWR 有缓存则用缓存 | ✅ 提交时检查 |
| UserProfile | 配额卡片不渲染，其他内容正常 | ❌ 不阻塞 | ❌ 无 | SWR 有缓存则用缓存 | 不涉及 |
| 提交 catch（所有 Modal） | 显示通用错误 Toast | ❌ 不阻塞 | ✅ 通用错误 Toast | 不涉及 | ✅（已生效） |

### 6.5 当前容错设计的风险点

1. **MovieRequestModal / CollectionRequestModal 的潜在死锁**：
   - 配额 API 返回 403/500 时，Modal 的 `loading` prop 恒为 true（因为 error 被忽略、`!quota` 成立）
   - 用户无法关闭 Modal（除非刷新页面或点击背景，但 `loading={true}` 可能禁用背景点击）
   - 应改为解构 `error` 并在出错时终止加载状态

2. **前端配额预检查失效导致的用户困惑**：
   - TvRequestModal 在配额 API 失败时允许用户点击「全选」和「提交」，但后端仍会返回配额超限 403
   - 用户看到的是通用「提交失败」Toast，而非具体的配额提示，体验不一致

3. **配额错误与权限错误同质化**：
   - 403 可能是权限不足也可能是配额超限，但前端 catch 块未区分
   - 建议：catch 中解构 `error.response?.data?.message`，按后端消息显示具体原因，或根据状态码 + 消息关键词展示「配额已用完，请 X 天后再试」vs「无操作权限」

---

## 七、三条 Bug 的代码级修复路径深度分析

### 7.1 Bug 1：SWR 补解构 error 后，loading 能否在 useEffect 里正确翻 false

#### 7.1.1 Modal 组件的 loading 工作机制

Modal 组件（`src/components/Common/Modal/index.tsx:44-250`）接收 `loading` prop 后，内部通过两个互斥的 `Transition` 控制显示：

```tsx
// 第 112-117 行：loading=true 时显示 Spinner
<Transition show={loading}>
  <div style={{ position: 'absolute' }}>
    <LoadingSpinner />
  </div>
</Transition>

// 第 118-135 行：loading=false 时显示表单内容
<Transition show={!loading} ref={modalRef}>
  {/* 实际 Modal 内容 + 按钮 */}
</Transition>
```

**关键细节**：
- 两个 Transition 通过 `show={loading}` vs `show={!loading}` 实现切换
- `useClickOutside`（第 83-87 行）始终挂载，但点击背景时 `backgroundClickable` 为 true 才触发 `onCancel`。**`loading` 不影响背景点击**，所以用户即使在死锁状态也能点背景关闭 Modal（只是看不到内容）
- 按钮区域（第 193-244 行）在 `<Transition show={!loading}>` 内部，loading=true 时 DOM 不渲染，**确认按钮不可点击**，但 `okDisabled` 的判断仍正确执行（只是渲染层被覆盖）

#### 7.1.2 当前 MovieRequestModal 的死锁逻辑溯源

代码位置：`src/components/RequestModal/MovieRequestModal.tsx:61-71` 和 `317-323`

```tsx
// 第 61-71 行：两个 SWR
const { data, error } = useSWR<MovieDetails>(`/api/v1/movie/${tmdbId}`, {
  revalidateOnMount: true,
});
// ⚠️ 配额 SWR 只解构 data，不解构 error
const { data: quota } = useSWR<QuotaResponse>(
  user && ... ? `/api/v1/user/${...}/quota` : null
);

// 第 319 行：loading 计算
loading={(!data && !error) || !quota}
```

**死锁真值表演绎**（配额 API 返回 403 的场景）：

| 变量 | 值 | 原因 |
|------|----|------|
| `data`（电影详情） | 非 null | 电影详情 API 成功 |
| `error`（电影详情） | undefined | 电影详情无错 |
| `quota` | undefined | 配额 SWR 失败，data 为 undefined |
| `quotaError`（未被解构） | Error 对象 | 实际存在但无法访问 |
| `(!data && !error)` | `false` | 因为 data 存在 |
| `!quota` | `true` | quota 为 undefined |
| **最终 loading** | **`true`** | OR 运算结果恒真 |

CollectionRequestModal 完全相同（`CollectionRequestModal.tsx:54-64`、`267`）。

TvRequestModal 无此问题（`TvRequestModal.tsx:392-393`）：`loading={!data && !error}` 只等媒体详情，不等配额。

#### 7.1.3 修复方案：不需要 useEffect，直接在 loading 表达式中加入 error

**正确做法（推荐）**：解构 error 并在 loading 表达式中短路：

```tsx
// MovieRequestModal.tsx / CollectionRequestModal.tsx
// Step 1: 补解构 error
const { data: quota, error: quotaError } = useSWR<QuotaResponse>(
  user && ... ? `/api/v1/user/${...}/quota` : null
);

// Step 2: 修改 loading 表达式（仅在 既没数据 也 没错误 时才显示 loading）
loading={(!data && !error) || (!quota && !quotaError)}
//                                    ^^^^^^^^^^^^^^^^^^^^
//                                    新增：配额有错误时不再卡 loading
```

**为什么不需要 useEffect？**
- loading 是纯派生值（derived state），不是独立 state，无需通过 useEffect 监听 error 变化再 setState
- SWR 的 `data` 和 `error` 本身是响应式的，它们变化会触发组件重渲染，重新计算 loading prop
- 如果引入 `useState + useEffect` 反而增加不必要的重渲染和状态同步问题

**修复后的真值表**：

| 场景 | quota | quotaError | `!quota && !quotaError` | loading 最终值 |
|------|-------|-----------|------------------------|--------------|
| 加载中 | undefined | undefined | true | true（显示 Spinner，正常） |
| 成功 | 非 null | undefined | false | false（显示内容） |
| 403 失败 | undefined | Error 对象 | false | false（显示内容，后端兜底） |
| 500 失败 | undefined | Error 对象 | false | false（显示内容，后端兜底） |

**需要同步修改的文件**：
- `src/components/RequestModal/MovieRequestModal.tsx:66,319`（两处：第 66 行解构、第 319 行 loading 表达式）
- `src/components/RequestModal/CollectionRequestModal.tsx:59,267`（同上两处）
- TvRequestModal 不受影响（不等待 quota），但建议也加上错误解构以保持代码一致性

---

### 7.2 Bug 2：catch 块保留后端错误 detail 的实现路径与接口兼容性

#### 7.2.1 后端错误响应结构（已稳定，无需兼容改造）

后端全局错误处理中间件定义在 `server/index.ts:255-269`：

```tsx
server.use(
  (
    err: { status: number; message: string; errors: string[] },
    _req: Request,
    res: Response,
    _next: NextFunction
  ) => {
    res.status(err.status || 500).json({
      message: err.message,   // 后端抛出的人类可读消息
      errors: err.errors,     // 可选：字段级校验错误数组
    });
  }
);
```

请求路由层的具体错误映射（`server/routes/request.ts:316-334`）：

```tsx
switch (error.constructor) {
  case RequestPermissionError:
  case QuotaRestrictedError:
    return next({ status: 403, message: error.message });
    //                               ^^^^^^^^^^^^^^^^^^^^
    // 例如："Movie Quota exceeded." / "You do not have permission to make 4K movie requests."
  case DuplicateMediaRequestError:
    return next({ status: 409, message: error.message });
  ...
}
```

**结论：后端响应格式已稳定为 `{ message: string, errors?: string[] }`**，且 message 是有语义的英文原文，前端可直接解析关键词或直接展示。

测试文件也验证了此格式（`server/routes/request.test.ts:44-56`、`issue.test.ts:45-56`）。

#### 7.2.2 前端 catch 中错误对象的实际结构

项目使用 `axios`，错误通过 `axios.isAxiosError(e)` 识别。参考 `TitleCard/ErrorCard.tsx:33`、`SettingsMetadata.tsx:118`、`Login/LocalLogin.tsx:71` 等已有的正确用法：

```typescript
// axios 错误对象结构
e.response?.status    // HTTP 状态码，如 403
e.response?.data?.message  // 后端返回的 { message } 字段内容
e.message             // axios 本地消息（如 "Request failed with status code 403"）
```

#### 7.2.3 修复方案：分层展示，优先后端消息，fallback 到通用文案

**当前代码**（MovieRequestModal.tsx:127-131，TvRequestModal.tsx:225-229，CollectionRequestModal 类似）：

```tsx
catch {
  addToast(intl.formatMessage(messages.requesterror), {
    appearance: 'error',
    autoDismiss: true,
  });
}
```

**修复后代码**（三个 Modal 均需修改）：

```tsx
catch (e) {
  let errorMessage = intl.formatMessage(messages.requesterror); // 默认 fallback

  if (axios.isAxiosError(e) && e.response?.data?.message) {
    const backendMessage = e.response.data.message;
    // 可选：根据关键词映射为 i18n 消息（如果需要多语言）
    // 例如：
    // if (backendMessage.includes('Quota exceeded')) {
    //   errorMessage = intl.formatMessage(messages.quotaExceeded);
    // } else if (backendMessage.includes('permission')) {
    //   errorMessage = intl.formatMessage(messages.noPermission);
    // } else {
    //   errorMessage = backendMessage; // 兜底直接显示后端英文
    // }

    // 简单方案：直接显示后端 message（当前后端都是英文，需新增 i18n key 才完美）
    errorMessage = backendMessage;
  }

  addToast(errorMessage, {
    appearance: 'error',
    autoDismiss: true,
  });
}
```

#### 7.2.4 兼容性与 i18n 分析

**需要修改的文件清单**（共 3 个 Modal，9 处 catch 块）：

| 文件 | catch 位置 | 当前消息 | 用途 |
|------|----------|---------|------|
| MovieRequestModal.tsx | 127 行 | `requesterror` | 提交新请求失败 |
| MovieRequestModal.tsx | 170 行 | 无 Toast | 取消请求失败（静默） |
| MovieRequestModal.tsx | 215 行 | `errorediting` | 编辑请求失败 |
| TvRequestModal.tsx | 225 行（搜索 catch） | `requesterror` | 提交新请求失败 |
| TvRequestModal.tsx | 搜索 edit catch | `errorediting` | 编辑请求失败 |
| CollectionRequestModal.tsx | 搜索 sendRequest catch | `requesterror` | 提交合集请求失败 |

**i18n 兼容性评估**：
- 当前 `messages.requesterror` 在 `defineMessages` 中有 4 种语言的翻译（`src/i18n/locale/*`）
- 后端返回的 `message` 是纯英文，若直接显示会导致非英文用户看到英文错误
- **推荐做法**：新增 i18n key（`quotaExceededMovie`、`quotaExceededSeries`、`noRequestPermission` 等），在 catch 中按关键词匹配
- **最小侵入做法**：先判断是否为 AxiosError，若状态码为 403 且 message 包含 "Quota"，显示新的 `quotaExceeded` i18n 消息；其余仍用 `requesterror`

---

### 7.3 Bug 3：首次加载失败的降级回退（缓存 vs 默认值）修改位置

#### 7.3.1 当前 SWR 全局配置与缓存机制

全局 SWR 配置在 `src/pages/_app.tsx:191-198`：

```tsx
<SWRConfig
  value={{
    fetcher: (url) => axios.get(url).then((res) => res.data),
    fallback: {
      '/api/v1/auth/me': user, // SSR 注入的初始用户数据
    },
    // 无 onErrorRetry、无 errorRetryCount，使用 SWR 默认重试策略
  }}
>
```

SWR 默认重试策略：
- 首次失败后指数退避重试（最多重试 5 次）
- 重试会继续更新 `error`，但不会更新 `data`
- 只有**同一 key 之前成功过**的数据才会保留在 SWR 缓存中并通过 `data` 返回

#### 7.3.2 各组件当前的降级行为分析

| 组件 | 配额数据不可用时的行为 | 使用 SWR fallback？ |
|------|----------------------|-------------------|
| MiniQuotaDisplay | `if (error) return null`（`MiniQuotaDisplay/index.tsx:25-27`） | 无，静默隐藏 |
| MovieRequestModal | `!quota` → 卡 loading 死锁（Bug 1） | 无 fallbackData |
| CollectionRequestModal | 同上死锁 | 无 fallbackData |
| TvRequestModal | `quota?.tv.remaining ?? 0` → 退化为 0 | 无 fallbackData，但代码用了 `??` 兜底 |
| UserProfile | `{quota && ...}` → 配额卡片不渲染 | 无 fallbackData |
| QuotaDisplay（子组件） | `remaining ?? quota?.remaining ?? 0`、`quota?.limit ?? 1` | 无，靠 props 传值的 `??` 兜底 |

#### 7.3.3 降级回退方案对比

**方案 A：使用 SWR `fallbackData`（推荐）**

SWR 原生支持 `fallbackData`，在**首次加载且无缓存**时作为初始 data 展示。参考已有的 `useUser.ts:68-69`：

```tsx
const { data, error } = useSWR<User>(url, {
  fallbackData: initialData,  // SSR 注入的初始用户
  ...
});
```

在三个 RequestModal 中的应用：

```tsx
// 默认配额：视为无限制（restricted=false, limit=0），避免阻塞请求
const defaultQuota: QuotaResponse = {
  movie: { used: 0, restricted: false },
  tv: { used: 0, restricted: false },
};

const { data: quota, error: quotaError } = useSWR<QuotaResponse>(
  user && ... ? `/api/v1/user/${...}/quota` : null,
  {
    fallbackData: defaultQuota,
    // 可选：shouldRetryOnError: false  // 避免失败后反复重试
  }
);
```

**方案 B：通过 JSX 中 `??` 运算符内联兜底**

当前 TvRequestModal 的 `currentlyRemaining` 已部分采用（`TvRequestModal.tsx:98-101`）：

```tsx
const currentlyRemaining =
  (quota?.tv.remaining ?? 0) - selectedSeasons.length + ...;
```

但这不够：`quota?.movie.restricted` 在 quota undefined 时为 undefined，导致 `okDisabled` 判断逻辑不完整。

**方案 C：SWR 全局 `fallback` 中预填所有用户配额 key**

不推荐：用户 id 是动态的，无法在 `_app.tsx` 的静态 fallback 中预知。

#### 7.3.4 推荐的修改位置与具体代码

**优先级最高：修复 MovieRequestModal 和 CollectionRequestModal（消除死锁 + 合理降级）**

```tsx
// MovieRequestModal.tsx:66-71 修改后
const defaultQuota: QuotaResponse = {
  movie: { used: 0, restricted: false },
  tv: { used: 0, restricted: false },
};

const { data: quota, error: quotaError } = useSWR<QuotaResponse>(
  user &&
    (!requestOverrides?.user?.id || hasPermission(Permission.MANAGE_USERS))
    ? `/api/v1/user/${requestOverrides?.user?.id ?? user.id}/quota`
    : null,
  { fallbackData: defaultQuota }  // 新增
);

// MovieRequestModal.tsx:319 修改后
loading={(!data && !error) || (!quota && !quotaError && !defaultQuota)}
// 注：用了 fallbackData 后 quota 至少是 defaultQuota，!quota 永远为 false，
// 所以可以简化为：loading={!data && !error}
```

**TvRequestModal.tsx:91-96**（一致性优化，原无死锁，但可加 fallbackData 让 UI 更平滑）：

```tsx
const defaultQuota: QuotaResponse = { ... };
const { data: quota } = useSWR<QuotaResponse>(url, { fallbackData: defaultQuota });
```

**MiniQuotaDisplay（一致性优化）**：
- 当前 `if (error) return null` 行为合理，无需改动
- 若需体验更好，可改为显示「配额信息暂时不可用」提示，但当前静默降级符合 Seerr 整体 UX

**UserProfile**：
- 当前 `{quota && ...}` 短路行为合理，资料页不强求显示配额
- 无需改动

#### 7.3.5 降级为默认值的安全性考量

将 `defaultQuota.restricted` 设为 `false`（无限制）是安全的，因为：

1. 后端 `MediaRequest.request()`（`MediaRequest.ts:114-120`）始终会再次调用 `getQuota()` 并检查 `restricted`，前端降级不会绕过配额限制
2. 降级场景（配额 API 失败）下，用户体验优先于前端预检查；后端兜底足以防止超限
3. 如果 `restricted` 设为 `true`，会导致配额 API 暂时故障时所有用户都无法提交请求，属于误杀

---

### 7.4 三条 Bug 的修改总览

| Bug | 需改文件 | 修改点 | 风险等级 |
|-----|---------|-------|---------|
| 1. loading 死锁 | MovieRequestModal.tsx、CollectionRequestModal.tsx | 解构 `error: quotaError` + 修改 loading 表达式 | 低（纯派生值，无副作用） |
| 2. catch 丢失错误 detail | MovieRequestModal.tsx、TvRequestModal.tsx、CollectionRequestModal.tsx（多个 catch） | 补 `catch (e)` 参数 + `axios.isAxiosError(e)` 判断 + 读取 `e.response.data.message` | 中（需决定是显示后端原文还是新增 i18n key） |
| 3. 首次加载无降级 | MovieRequestModal.tsx、CollectionRequestModal.tsx（可选 TvRequestModal） | 增加 `fallbackData: { movie: {used:0, restricted:false}, tv: {...} }` | 低（后端有兜底检查） |

**推荐修复顺序**：Bug 1（死锁最严重）→ Bug 3（fallbackData 顺带消除死锁）→ Bug 2（i18n 工作量较大，可延后）。
