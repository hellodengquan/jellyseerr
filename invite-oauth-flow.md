# 多用户邀请与 OAuth 协作流程梳理

## 一、核心概念与数据模型

### 1.1 用户类型（UserType）
定义于 `server/constants/user.ts:1-6`：

| 值 | 类型 | 说明 |
|---|---|---|
| 1 | PLEX | Plex 媒体服务器用户 |
| 2 | LOCAL | 本地注册用户（邮箱+密码） |
| 3 | JELLYFIN | Jellyfin 媒体服务器用户 |
| 4 | EMBY | Emby 媒体服务器用户 |

### 1.2 用户实体关键字段
定义于 `server/entity/User.ts`：

- `id`: 主键，自增
- `email`: 用户邮箱（唯一，大小写不敏感）
- `username`: 本地用户名
- `plexId` / `plexUsername` / `plexToken`: Plex 关联信息
- `jellyfinUserId` / `jellyfinUsername` / `jellyfinDeviceId` / `jellyfinAuthToken`: Jellyfin/Emby 关联信息
- `password`: 本地密码哈希（bcrypt）
- `permissions`: 权限位掩码
- `userType`: 用户类型枚举

---

## 二、用户"邀请"（创建）的三种途径

> 注：该项目并无传统的"邀请码"机制，用户通过以下方式加入系统：

### 2.1 管理员创建本地用户

**前端入口**：`src/components/UserList/index.tsx:384-561`（创建用户弹窗）

**后端接口**：`POST /api/v1/user`，实现于 `server/routes/user/index.ts:170-228`

流程：
1. 管理员在用户列表页点击"Create Local User"
2. 填写用户名、邮箱、密码（或勾选自动生成）
3. 后端校验邮箱唯一性
4. 若未显式提供密码：调用 `User.generatePassword()` 生成随机密码并发送邮件
5. 新用户以 `UserType.LOCAL` 类型入库，权限为 `settings.main.defaultPermissions`

### 2.2 管理员从媒体服务器批量导入

**Plex 导入**：
- 接口：`POST /api/v1/user/import-from-plex`，`server/routes/user/index.ts:657-722`
- 流程：用主用户（id=1）的 plexToken 调用 Plex.tv API 获取共享用户列表，逐个校验邮箱/plexId 是否存在，不存在则以 `UserType.PLEX` 新建

**Jellyfin/Emby 导入**：
- 接口：`POST /api/v1/user/import-from-jellyfin`，`server/routes/user/index.ts:724-799`
- 流程：用管理员身份调用 Jellyfin API 获取用户列表，按选中的 userId 批量创建，类型为 `UserType.JELLYFIN` 或 `UserType.EMBY`

### 2.3 外部用户自助登录（首次登录即"接受邀请"）

当 `settings.main.newPlexLogin` 为 `true` 时，媒体服务器的合法用户首次登录即自动创建账号。

- Plex：`server/routes/auth.ts:147-183`
- Jellyfin：`server/routes/auth.ts:440-483`

### 2.4 密码重置（类邀请）链接的过期与失效机制

> 注：该项目**没有传统的一次性邀请码/邀请链接机制**。与"链接过期"相关的代码触发只有密码重置链接 + Plex PIN + Session 过期三条链路。

#### 2.4.1 密码重置链接（最接近邀请语义）

**生成**：`server/entity/User.ts:229-265` `User.resetPassword()`
```typescript
this.resetPasswordGuid = randomUUID();              // 随机 GUID 作为链接 token
// 24 hours into the future
const targetDate = new Date();
targetDate.setDate(targetDate.getDate() + 1);
this.recoveryLinkExpirationDate = targetDate;       // 写入过期时间 = 现在+24h
```
触发入口：`POST /api/v1/auth/reset-password`（`server/routes/auth.ts:725-758`），任何用户（登录与否都可）提供自己的 email 即可触发（邮箱不存在也返回 200，避免枚举）。

**校验（过期/失效判定）**：`POST /api/v1/auth/reset-password/:guid`，`server/routes/auth.ts:760-817`

三层判定按顺序触发：
1. **Token 不存在**（`resetPasswordGuid` 数据库无匹配）→ 500 `"Invalid password reset link."`
2. **过期**（`!recoveryLinkExpirationDate || recoveryLinkExpirationDate <= new Date()`）→ 500 `"Invalid password reset link."`
3. **一次性消费**：通过校验后立即 `user.recoveryLinkExpirationDate = null`，再 `setPassword()` 保存——同一 guid 不能重复使用

#### 2.4.2 Plex OAuth PIN 码（登录授权的临时码）

- 服务端无显式代码控制过期，完全由 Plex.tv API 本身控制
- 前端 `src/utils/plex.ts` 的 `pinPoll()` 每秒轮询一次 `plex.tv/api/v2/pins/{id}`，直到拿到 `authToken` 就停止
- 实际上 Plex 的 PIN 码有自己的过期时间（约 15 分钟），轮询不到时用户关闭弹窗即流程终止

#### 2.4.3 登录 Session 过期（相当于"登录凭证失效"）

配置于 `server/index.ts:207-226`：
```
cookie.maxAge        = 30 天（毫秒）
TypeormStore.ttl     = 30 天（秒）
Session.expiredAt    = connect-typeorm 每次访问会刷新为 Date.now() + ttl
```
- 清理策略：`cleanupLimit: 2` —— 每 2 次 session 写入触发一次过期数据清理（connect-typeorm 内置）
- 手动失效：`POST /api/v1/auth/logout` 调用 `req.session.destroy()` 并在 Jellyfin 侧调用 `DELETE /Devices` 注销对应的 jellyfinDeviceId

---

## 三、OAuth / 外部登录流程

### 3.1 Plex OAuth 登录（弹窗 PIN 码模式）

#### 前端链路
```
src/components/Login/PlexLoginButton.tsx
  └─> src/hooks/usePlexLogin.ts:login()
        └─> src/utils/plex.ts:PlexOAuth.login()
              ├─ initializeHeaders()     构造 X-Plex-* 请求头
              ├─ getPin()               POST plex.tv/api/v2/pins 获取 PIN(id, code)
              ├─ preparePopup()         打开 /login/plex/loading 弹窗
              ├─ 弹窗跳转 app.plex.tv/auth/#?code=xxx
              └─ pinPoll()              每秒轮询 plex.tv/api/v2/pins/{id}，拿到 authToken 后回调
```

**回调处理**：`src/components/Login/index.tsx:47-68`
拿到 `authToken` 后调用 `POST /api/v1/auth/plex`。

#### 后端链路：`POST /api/v1/auth/plex`
实现于 `server/routes/auth.ts:49-220`

```
1. 校验 mediaServerType 和 mediaServerLogin 开关
2. 用 authToken 调 PlexTvAPI.getUser() 拿到 Plex 账户信息（id, email, username, thumb）
3. 用户查找（匹配逻辑）：
   WHERE user.plexId = account.id OR user.email = account.email
4. 分支判断：
   A) 数据库无任何用户 → 自动将该用户设为管理员（Owner），保存 Plex 配置，启动定时任务
   B) 用户已存在 → 更新 plexToken、头像、邮箱等信息
   C) 用户不存在 + newPlexLogin=true + 有权限 → 自动创建新用户（默认权限）
   D) 用户不存在 + newPlexLogin=false → 返回 403 Access denied
5. 关键：调用 PlexTvAPI.checkUserAccess(account.id) 验证该用户是否在主用户的媒体服务器共享列表中
6. 写入 session: req.session.userId = user.id
```

### 3.2 Jellyfin / Emby 登录（用户名密码模式）

**接口**：`POST /api/v1/auth/jellyfin`，`server/routes/auth.ts:226-594`

```
1. 校验登录开关及主机配置
2. 查找已存在的设备 ID（jellyfinDeviceId），否则基于用户名 base64 生成
3. 调 JellyfinAPI.login(username, password, clientIp) 获取 AccessToken 及 User 信息
4. 按 jellyfinUserId 查找已存在用户
5. 分支：
   A) 无任何用户 + 当前 Jellyfin 用户是管理员 → 创建初始管理员（id=1），并在 Jellyfin 侧创建 API Key
   B) 用户已存在 → 更新用户名、头像
   C) 用户不存在 + newPlexLogin=true → 自动创建新用户
   D) 用户不存在 + newPlexLogin=false → 返回 403
6. 初始化本地密码（若显式传入 password）
7. 检查头像变更
8. 写入 session
```

### 3.3 本地账号登录

**接口**：`POST /api/v1/auth/local`，`server/routes/auth.ts:596-646`

- 校验 `localLogin` 开关
- 按 email 查找用户
- `bcrypt.compare()` 验证密码
- 写入 session

### 3.4 多 OAuth Provider 共存时的账号合并与冲突处理

三种登录方式使用**不同的用户匹配键**，由此决定了合并与冲突的行为差异——这是多 Provider 共存时最核心的代码特点。

#### 3.4.1 各 Provider 的匹配键对比

| 登录方式 | 匹配条件（WHERE） | 匹配到后的合并行为 |
|---|---|---|
| Plex `POST /auth/plex` | `plexId = account.id` **OR** `email = account.email` | 无条件覆盖：`plexId/plexUsername/plexToken/email/avatar/userType` 全改为 Plex 提供的值 |
| Jellyfin `POST /auth/jellyfin` | `jellyfinUserId = account.User.Id`（不看 email） | 只更新 `jellyfinUsername/avatar`，不动 email 与其他 Provider 字段 |
| Local `POST /auth/local` | `email = body.email`（精确唯一） | 不合并，纯密码校验 |

#### 3.4.2 自动合并的成功场景（隐式 Link）

**场景 1：本地用户 → Plex 登录**
> 用户 A 先由管理员创建为 Local（`email=foo@bar.com, userType=LOCAL`），后来用同邮箱的 Plex 账号点 Plex 登录

- `server/routes/auth.ts:74-80` 走的是 `OR email` 分支，A 用户被找到
- `auth.ts:139-144`：**静默升级**为 PLEX 类型，`plexId/plexToken/...` 写入，相当于**隐式关联了 Plex 账号**
- 关键：之后即使邮箱变化，下一次登录仍按 `plexId` 匹配回该用户

**场景 2：Jellyfin 用户 → 继续用 Jellyfin 登录**
> 用 Jellyfin UserId 精确匹配，不会产生合并歧义。

#### 3.4.3 产生冲突/重复账号的场景（代码未做合并）

**场景 3：本地用户 + 同邮箱的 Jellyfin 登录（新建独立账号！）**
> 本地用户 A（email=foo@bar.com, userType=LOCAL）。同邮箱在 Jellyfin 里也有账号 foo。配置 newPlexLogin=true 后，foo 用 Jellyfin 用户名密码登录 Seerr

- Jellyfin 登录**完全不匹配 email**，只看 `jellyfinUserId`
- 结果：创建全新的用户 B（jellyfinUserId=xxx, email 为 undefined 或 body.email）
- **两个独立的 Seerr 用户记录 A 和 B，邮箱理论上可能冲突**（但因为 Jellyfin 登录创建时 `body.email` 可不传，实际不冲突；如果 body.email 与 A 相同，数据库唯一索引会在 `save()` 时抛错）

**场景 4：Plex 用户 + 同人的 Jellyfin 用户（永远不会自动合并）**
> 同一个人同时有 Plex 和 Jellyfin 账号，先后用两种方式登录 Seerr

- Plex 匹配的是 `plexId/email`，Jellyfin 匹配的是 `jellyfinUserId`
- 两套键完全正交 → **必然创建两个 Seerr 用户**
- 解决方式：**只能走手动关联接口 linked-accounts**（见第五节）

**场景 5：两个不同 Plex 账号共用邮箱（后登录者被合并到先存在的用户）**
> Plex 账号 X（email=foo）和 Plex 账号 Y（email=foo）。X 先登录创建 Seerr 用户，Y 再来。

- Y 登录时按 `OR email` 匹配到用户 → 覆盖写 Y 的 plexId/plexToken 到同一个记录
- X 再次登录时按自己的 plexId 找不到了，但 `OR email` 仍匹配 → **又覆盖回来**
- 本质：**邮箱在 Plex 登录中是主键级别的匹配键**，两个外部账号同邮箱会出现"抢用户"的乒乓现象

#### 3.4.4 手动关联（linked-accounts）的冲突保护

当用户通过 `POST /settings/linked-accounts/*` 显式关联时，代码做了严格的防占用检查：

**Plex 关联**（`server/routes/user/usersettings.ts:264-309`）三重校验：
1. `userRepository.exist({ where: { plexId: account.id } })` → 422 `"This Plex account is already linked to a Seerr user"`
2. `user.email !== account.email` → 422 `"This Plex account is registered under a different email address."`（**邮箱必须完全一致**）
3. 归属校验：用传入的 authToken 成功拉到 Plex 用户信息即隐含通过

**Jellyfin 关联**（`server/routes/user/usersettings.ts:362-458`）三重校验：
1. 登录前按 `jellyfinUsername` 查存在 → 422
2. 用 Jellyfin 凭证登录成功拿到 `account.User.Id`
3. 登录后按 `jellyfinUserId = account.User.Id` 再查一遍 → 422（防止用户名改动后 ID 已绑定他人）

注意 Jellyfin 关联**不检查 email**，意味着同一个 Seerr 用户可以"先关联 Plex 再解绑再关联 Jellyfin"逐步切换 Provider，但 Plex→Jellyfin 的 email 相同不代表会自动匹配。

#### 3.4.5 解除关联时的字段回退

`DELETE /linked-accounts/plex` 或 `jellyfin`：
- `userType` 强制回写为 `UserType.LOCAL`
- 对应 Provider 字段全部置空（plexId/plexUsername/plexToken 或 jellyfinUserId/jellyfinUsername/jellyfinAuthToken/jellyfinDeviceId）
- **硬性限制 1**：id=1（主管理员）禁止解绑 Provider（`usersettings.ts:336-341`）
- **硬性限制 2**：`!user.email || !user.password` → 400 `"User does not have a local email or password set."`（解绑后必须还能用 Local 方式登录）

---

## 四、用户归属判断逻辑

用户归属即"该外部账号是否属于本 Seerr 实例的合法用户"，核心校验在登录接口与导入接口中。

### 4.1 Plex 归属校验

**核心函数**：`server/api/plextv.ts:230-257` `PlexTvAPI.checkUserAccess(userId)`

```
1. 确保 Plex 已配置（settings.plex.machineId 存在）
2. 调用 getUsers() 获取主账号的所有共享用户
3. 在共享列表中找到 userId 对应的用户
4. 检查该用户是否被授予了当前媒体服务器（machineIdentifier）的访问权
   即 user.Server[] 中存在与 settings.plex.machineId 匹配的记录
```

在 `server/routes/auth.ts:119-123` 中的完整判定：
```typescript
if (
  account.id === mainUser.plexId ||                      // 本人就是主用户
  (account.email === mainUser.email && !mainUser.plexId) || // 邮箱匹配且主用户还未绑定 Plex ID
  (await mainPlexTv.checkUserAccess(account.id))           // 在共享列表中且有权限
) { /* 通过校验 */ }
```

### 4.2 Jellyfin / Emby 归属校验

Jellyfin 本身就是一个账号系统，因此校验较简单：
- 能用给定用户名密码成功登录 Jellyfin 服务即视为归属合法
- 登录返回的 `account.User.Policy.IsAdministrator` 决定是否为管理员

### 4.3 本地账号归属校验

- email 唯一匹配 + bcrypt 密码验证
- 见 `server/routes/auth.ts:609-626`

---

## 五、账户关联（Link）与解除

本地用户可与媒体服务器账号绑定/解绑。

### 5.1 关联 Plex

**接口**：`POST /api/v1/user/:id/settings/linked-accounts/plex`，`server/routes/user/usersettings.ts:264-309`

校验条件：
1. Plex 登录已启用
2. 该 Plex 账号（plexId）未被其他用户绑定
3. Plex 账号的 email 与 Seerr 用户的 email **必须完全一致**

成功后将 userType 改为 `UserType.PLEX`，写入 plexId、plexUsername、plexToken。

### 5.2 关联 Jellyfin / Emby

**接口**：`POST /api/v1/user/:id/settings/linked-accounts/jellyfin`，`server/routes/user/usersettings.ts:362-458`

校验条件：
1. Jellyfin/Emby 登录已启用
2. jellyfinUsername 未被绑定
3. 能用 Jellyfin 凭证登录成功
4. 返回的 jellyfinUserId 未被绑定

成功后更新 userType 及相关字段。

### 5.3 解除关联

**接口**：`DELETE /api/v1/user/:id/settings/linked-accounts/{plex|jellyfin}`

限制：
- 主管理员（id=1）**不得**解除媒体服务器关联
- 用户必须已设置本地 email 和密码，否则解除后无法登录
- 解除后 userType 重置为 `UserType.LOCAL`，清空对应媒体服务器字段

---

## 六、用户配额（Quota）系统

配额系统限制用户在一定周期内的请求数量（电影按"部"计，剧集按"季"计），与邀请流程的关联在于：新建用户时即按全局默认或管理员指定写入配额参数。

### 6.1 数据模型：User 实体中的配额字段

定义于 `server/entity/User.ts:125-135`：

| 字段 | 含义 | null 语义 |
|---|---|---|
| `movieQuotaLimit` | 周期内允许请求的电影数量（整数） | 用全局默认值 |
| `movieQuotaDays` | 电影配额的统计周期（天数） | 用全局默认值 |
| `tvQuotaLimit` | 周期内允许请求的电视剧**季**数 | 用全局默认值 |
| `tvQuotaDays` | 电视剧配额的统计周期（天数） | 用全局默认值 |

**全局默认配置**：`settings.main.defaultQuotas.movie.{quotaLimit, quotaDays}` 与 `settings.main.defaultQuotas.tv.{quotaLimit, quotaDays}`。

> 配额字段可在管理员创建用户（`POST /api/v1/user`）或编辑用户（`PUT /api/v1/user/:id`）时单独指定，覆盖全局默认。

### 6.2 配额计算：`User.getQuota()`

实现于 `server/entity/User.ts:273-372`，返回 `QuotaResponse` 结构：

```typescript
{
  movie: { days, limit, used, remaining, restricted },
  tv:    { days, limit, used, remaining, restricted }
}
```

#### 计算规则（代码逐点）

1. **权限豁免**（`User.ts:278-280, 306-308`）
   ```typescript
   const canBypass = this.hasPermission([Permission.MANAGE_USERS], { type: 'or' });
   const movieQuotaLimit = !canBypass ? (this.movieQuotaLimit ?? defaultQuotas.movie.quotaLimit) : 0;
   ```
   有 `MANAGE_USERS` 权限者 limit 强制置 0（表示无限制，后续判断以 `!!limit && restricted` 逻辑绕过）。

2. **字段优先级**：`this.xxx ?? defaultQuotas.xxx.quotaLimit/Days`（用户字段未设则用全局默认）。

3. **电影配额计数**（`User.ts:293-304`）
   - 统计条件：`requestedBy = 当前用户` + `type = MOVIE` + `status != DECLINED`
   - 若设置了 `movieQuotaDays`：附加 `createdAt > (now - N 天)` 滚动窗口
   - 注意：**status=DECLINED 的请求不占配额**，FAILED/PENDING/APPROVED/COMPLETED 都占

4. **电视剧配额计数**（`User.ts:317-348`）
   - 不是按"部"计，而是按 **季数总和** 计
   - 用子查询统计每个 TV 请求的 `SeasonRequest` 条数，再 `.reduce(sum, seasonCount)` 叠加
   - 同样排除 DECLINED 请求，支持滚动窗口

5. **restricted 判定**（`User.ts:358-360, 369`）
   ```typescript
   restricted: !!(movieQuotaLimit && movieQuotaLimit - movieQuotaUsed <= 0)
   ```
   limit 非零 且 used ≥ limit → 真正受限。limit=0（管理员或未设配额）永远返回 `restricted=false`。

### 6.3 请求拦截点：`MediaRequest.request()`

配额拒绝发生在**创建请求前立即拦截**，贯穿两处代码：

#### 通用拦截：`server/entity/MediaRequest.ts:114-120`
```typescript
const quotas = await requestUser.getQuota();
if (requestBody.mediaType === MediaType.MOVIE && quotas.movie.restricted)
  throw new QuotaRestrictedError('Movie Quota exceeded.');
if (requestBody.mediaType === MediaType.TV && quotas.tv.restricted)
  throw new QuotaRestrictedError('Series Quota exceeded.');
```

#### TV 细粒度拦截：`server/entity/MediaRequest.ts:451-455`
即使 `quotas.tv.restricted` 为 false（剩余 > 0），若**本次请求的季数超过剩余额度**也拒绝：
```typescript
if (quotas.tv.limit && finalSeasons.length > (quotas.tv.remaining ?? 0))
  throw new QuotaRestrictedError('Series Quota exceeded.');
```
例如：剩余 2 季，但用户一次性请求包含 S1+S2+S3 共 3 季 → 拒绝。

### 6.4 HTTP 层的错误传播

`server/routes/request.ts:321-324` 按错误类型映射状态码：
```typescript
case QuotaRestrictedError:
  return next({ status: 403, message: error.message });
```
前端可据此展示"配额超限"提示。

### 6.5 与邀请流程的配合

邀请/创建用户时的配额写入：
- 管理员点"Create Local User"创建：前端 `src/components/UserList/index.tsx` 弹窗中的配额字段传入后端
- 后端 `POST /api/v1/user`（`server/routes/user/index.ts:170-228`）按 body 赋值，未传时使用数据库列默认（null），真正生效时由 `getQuota()` 回退到全局默认
- 批量从 Plex/Jellyfin 导入的用户：配额字段全部不写，统一走全局默认 `defaultQuotas`
- 外部用户自助登录首次创建账号（`auth.ts` 中 `new User({...})`）：配额字段同样留空，按全局默认执行

---

## 七、会话与鉴权中间件

### 7.1 用户注入中间件

`server/middleware/auth.ts:9-41` `checkUser`

```
1. 若请求头 X-API-Key 匹配全局 API Key：
   - 以 id=1（主管理员）身份操作，或按 X-API-User 指定用户
2. 否则若 req.session.userId 存在：
   - 从数据库加载用户，挂到 req.user
3. 设置 req.locale（用户设置或全局默认）
```

### 7.2 权限校验中间件

`server/middleware/auth.ts:43-58` `isAuthenticated(permissions?, options?)`

- 未登录 → 403
- 缺少指定权限 → 403
- 位运算：`hasPermission(permissions, this.permissions, options)`

---

## 八、完整协作流程图

```
管理员视角                           新用户视角
───────────                          ───────────
创建本地用户 ────────────────────────────┐
   │                                     │
   │                                     ▼
   │                              [收到邮件/初始密码]
   │                                     │
   ▼                                     ▼
从 Plex/Jellyfin 导入用户 ────────► 用户存在于 Seerr 数据库
   │                                     │
   │                                     │ 若 newPlexLogin = true
   │                                     ▼
   │                              用户自助打开登录页
   │                                     │
   │                    ┌────────────────┴────────────────┐
   │                    │                                 │
   │                    ▼                                 ▼
   │           [点击 Plex 登录]                   [输入 Jellyfin 账号]
   │                    │                                 │
   │                    ▼                                 ▼
   │           Plex PIN 弹窗授权                   POST /auth/jellyfin
   │                    │                                 │
   │                    ▼                                 │
   │           POST /auth/plex  ◄─────────────────────────┘
   │                    │
   │                    ▼
   │          1. 用 Token 获取 Plex/Jellyfin 账户资料
   │          2. 按 plexId / email / jellyfinUserId 查找用户
   │          3. 归属校验：
   │             - Plex: 是主用户 or 在共享列表且有服务器权限
   │             - Jellyfin: 登录成功即合法
   │          4. 用户已存在？───是───► 更新资料并登录
   │                    │否
   │                    ▼
   │          newPlexLogin 开启？───否───► 403 Access Denied
   │                    │是
   │                    ▼
   │              自动创建新用户（默认权限 + 对应 UserType）
   │                    │
   ▼                    ▼
[用户列表可见]        登录成功，session 写入 userId
                          │
                          ▼
              用户可在设置页：关联/解绑媒体服务器账号
                          │
                          ▼
              关联要求：邮箱一致(Plex) 或 凭证正确(Jellyfin)
              解绑要求：已设置本地邮箱+密码，且非主管理员
```

---

## 九、关键代码文件索引

| 功能 | 文件路径 |
|---|---|
| 用户类型常量 | `server/constants/user.ts` |
| 用户实体（含 getQuota、resetPassword、密码哈希） | `server/entity/User.ts` |
| 会话实体（Session expiredAt 字段） | `server/entity/Session.ts` |
| 请求实体（含 QuotaRestrictedError、配额拦截逻辑） | `server/entity/MediaRequest.ts` |
| 认证路由（登录/登出/重置密码过期判定） | `server/routes/auth.ts` |
| 请求路由（QuotaRestrictedError → 403 映射） | `server/routes/request.ts` |
| 用户管理路由（CRUD/导入/配额字段写入） | `server/routes/user/index.ts` |
| 用户设置路由（密码/关联冲突校验/权限） | `server/routes/user/usersettings.ts` |
| 鉴权中间件（Session 读取、权限校验） | `server/middleware/auth.ts` |
| Plex.tv API 封装（含 checkUserAccess） | `server/api/plextv.ts` |
| Jellyfin API 封装 | `server/api/jellyfin.ts` |
| 服务器启动（Session TTL / maxAge 配置） | `server/index.ts` |
| Plex OAuth 前端逻辑（PIN 轮询） | `src/utils/plex.ts` |
| Plex 登录 Hook | `src/hooks/usePlexLogin.ts` |
| 前端登录页 | `src/components/Login/index.tsx` |
| 前端用户列表（创建/导入/配额表单） | `src/components/UserList/index.tsx` |
