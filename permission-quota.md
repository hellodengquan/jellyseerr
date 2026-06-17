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
