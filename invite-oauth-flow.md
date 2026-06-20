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

### 2.4 邀请链接共享与撤回（不存在该机制）

> 重要澄清：该项目**完全没有"邀请链接"或"共享链接"的概念**。经过全代码库检索，没有任何 invite link、share link、public link、邀请码 token 等相关实现。

#### 2.4.1 为什么不需要邀请链接

三种用户加入途径均不需要"链接"：
1. **管理员创建本地用户** → 直接填表单入库，可选邮件发送初始密码
2. **从媒体服务器批量导入** → 管理员后台操作，用户无需点击链接
3. **自助登录首次创建** → 用户直接通过 Plex/Jellyfin OAuth 登录即创建，不需要邀请链接

#### 2.4.2 与"邀请"最接近的功能对比

| 功能 | 是否生成链接 | 能否撤回 | 代码位置 |
|---|---|---|---|
| 密码重置链接 | ✅ 是（`resetPasswordGuid`） | ✅ 是（用后置空或过期） | `server/entity/User.ts:229-265` |
| Plex OAuth PIN 码 | ❌ 不是链接，是弹窗 PIN | ❌ 不能撤回，Plex.tv 侧控制过期 | `src/utils/plex.ts` |
| 登录 Session | ❌ 是 Cookie-Session | ✅ 是（logout 销毁） | `server/routes/auth.ts:648-716` |
| 邀请链接 | ❌ 不存在 | ❌ 不存在 | 无 |

#### 2.4.3 所谓"撤回邀请"的等效操作

如果管理员想"撤回"一个用户的访问权，只能通过**硬删除用户**实现：
- 接口：`DELETE /api/v1/user/:id`（`server/routes/user/index.ts:593-655`）
- 副作用：用户的所有请求、关注清单、Issue 等会被 CASCADE 删除
- 没有"禁用/冻结"的软删除概念

### 2.5 密码重置（类邀请）链接的过期与失效机制

> 注：该项目**没有传统的一次性邀请码/邀请链接机制**。与"链接过期"相关的代码触发只有密码重置链接 + Plex PIN + Session 过期三条链路。

#### 2.5.1 密码重置链接（最接近邀请语义）

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

## 六、用户删除后的关联清理（没有"禁用"功能）

> 澄清：该项目**没有"禁用用户"功能**（没有 active/disabled 状态字段），只有**硬删除**。用户被删时的关联数据清理由「数据库级 ON DELETE 约束 + 应用层手动批量删除」共同完成。

### 6.1 删除权限与前置校验

接口：`DELETE /api/v1/user/:id`，实现于 `server/routes/user/index.ts:593-655`

三层校验按顺序：
1. `user.id === 1` → 405 `"This account cannot be deleted."`（主管理员永远保留）
2. `user.hasPermission(Permission.ADMIN) && req.user?.id !== 1` → 405 `"You cannot delete users with administrative privileges."`（非 Owner 不能删管理员）
3. 必须有 `Permission.MANAGE_USERS` 权限（由路由层 `isAuthenticated` 保证）

### 6.2 关联实体的级联策略总览

TypeORM 实体上的 `onDelete` 配置决定了大部分清理行为，定义在各实体的 `@ManyToOne` / `@OneToOne` 装饰器：

| 关联实体 | 关联字段 | onDelete 策略 | 清理时机 |
|---|---|---|---|
| MediaRequest | requestedBy | **CASCADE** | 数据库级，删用户时自动删请求 |
| MediaRequest | modifiedBy | **SET NULL** | 数据库级，删用户时 modifiedBy 字段置空（保留历史审计） |
| Watchlist | requestedBy | **CASCADE** | 数据库级 |
| Watchlist | user | **CASCADE** | 数据库级 |
| UserSettings | user | **CASCADE** | 数据库级 |
| UserPushSubscription | user | **CASCADE** | 数据库级 |
| Issue | createdBy | **CASCADE** | 数据库级 |
| IssueComment | commentBy | **CASCADE** | 数据库级 |
| SeasonRequest | request | **CASCADE** | 随 MediaRequest 间接级联 |
| Issue | media | **CASCADE** | 随 Media 级联 |

> 完整 onDelete 定义见 `server/entity/MediaRequest.ts:535-550`、`server/entity/Watchlist.ts:49-56`、`server/entity/UserSettings.ts:39` 等。

### 6.3 应用层手动清理（覆盖 CASCADE 的特殊处理）

关键代码在 `server/routes/user/index.ts:623-639`，这段代码**故意绕过**数据库级 CASCADE：

```typescript
/**
 * Requests are usually deleted through a cascade constraint. Those however, do
 * not trigger the removal event so listeners to not run and the parent Media
 * will not be updated back to unknown for titles that were still pending. So
 * we manually remove all requests from the user here so the parent media's
 * properly reflect the change.
 */
await requestRepository.remove(user.requests, {
  chunk: user.requests.length / 1000, // 避免 SQLite Expression tree is too large
});
await userRepository.delete(user.id);
```

**为什么不直接靠 CASCADE？**
- CASCADE 只在数据库层执行，不触发 TypeORM 的 `@BeforeRemove` / `@AfterRemove` 事件监听器
- MediaRequest 上挂了 `@AfterRemove` 事件（`MediaRequestSubscriber` 或实体内部）会更新父 Media 的 `status` 回到 `UNKNOWN`（当该媒体最后一个待处理请求被删除时）
- 如果走 CASCADE，Media 状态会停留在 `PENDING`，变成"孤儿"状态，永远不会被下载

### 6.4 Jellyfin 侧的设备清理（登出）

删除用户并不会自动清理 Jellyfin 上的登录设备。Jellyfin 侧的设备注销只在**用户主动登出**时发生：

`server/routes/auth.ts:660-698`：
```typescript
if (isJellyfinOrEmby) {
  // 先在 Jellyfin 侧 DELETE /Devices 删除对应 jellyfinDeviceId
  await axios.delete(`${baseUrl}/Devices`, { params: { Id: user.jellyfinDeviceId } });
}
req.session?.destroy(() => { ... });
```
删除用户时只删 Seerr 数据库记录，Jellyfin 上的 `jellyfinDeviceId` 会作为"僵尸设备"保留，直到 Jellyfin 自身超时清理。

---

## 七、邀请链接追踪与滥用检测（几乎为空）

> 澄清：该项目**没有邀请链接（invite link）机制**，也没有 referrer/UTM/affiliate 追踪代码。与"滥用检测"相关的代码只有三处极简实现。

### 7.1 不存在的特性

| 特性 | 代码现状 |
|---|---|
| 邀请链接生成（邀请码 token） | 无 |
| 邀请人追踪（referrerUserId） | 无 |
| UTM 参数记录（utm_source/utm_medium） | 无 |
| 邀请链接点击统计 | 无 |
| Referer 请求头记录 | 无（代码中无任何 `req.get('Referer')` 或 `req.headers.referer` 调用） |

### 7.2 已有的滥用防护

#### 7.2.1 Rate Limit（设置接口）

`server/routes/settings/index.ts:35` 引入 + `routes/settings/index.ts:540` 使用：
```typescript
import rateLimit from 'express-rate-limit';
rateLimit({ windowMs: 60 * 1000, max: 50 }) // 1分钟最多50次
```
仅用于设置接口，登录、重置密码等敏感接口**没有** rate limit。

#### 7.2.2 重置密码邮箱防枚举

`server/routes/auth.ts:736-755`：
```typescript
const user = await userRepository.findOne({ where: { email: body.email.toLowerCase() } });
if (user) {
  await user.resetPassword();
  await userRepository.save(user);
  logger.info('Successfully sent password reset link', { ... });
} else {
  logger.error('Something went wrong sending password reset link', { ... });
}
return res.status(200).json({ status: 'ok' }); // 不管邮箱是否存在都返回 200
```
> 攻击者无法通过返回值判断哪些邮箱已注册。

#### 7.2.3 登录失败日志（无封禁）

三种登录接口（Plex/Jellyfin/Local）在失败时都会用 `logger.warn` 记录：
- `server/routes/auth.ts:148-159`（Plex 未导入用户警告）
- `server/routes/auth.ts:186-195`（Plex 无权限警告）
- `server/routes/auth.ts:615-626`（本地密码错误警告）

记录字段：`ip`, `email`, `plexId`, `plexUsername`, `userId`。

**但没有**：
- 失败计数
- 账号锁定（N 次失败后锁定 X 分钟）
- IP 临时封禁
- CAPTCHA

### 7.3 间接追踪信息（仅用于 Push 订阅清理）

唯一记录 client 信息的地方是 Web Push 订阅：
`server/routes/user/index.ts:237-310` 中的 `registerPushSubscription` 接口会存 `userAgent` 字段，用于检测 iOS 静默刷新 endpoint 时清理"僵尸订阅"——不用于滥用检测。

`req.ip` 在登录、重置密码中广泛用于日志，但只做审计，不做拦截。IP 解析逻辑在 `server/index.ts:168-184` 用 `@supercharge/request-ip` 库处理 X-Forwarded-For。

---

## 八、媒体库权限传递与 Plex/Jellyfin 权限映射

> 核心结论：**Seerr 自身不做媒体库级别的访问控制**，权限完全由 Plex/Jellyfin 媒体服务器侧定义。Seerr 在邀请/登录流程中只做"用户是否有权访问媒体服务器"的二元判断，不传递、不映射具体库级权限。

### 8.1 Plex 侧权限的 Seerr 侧判定

**仅检查服务器访问权，不检查具体库。**

核心代码：`server/api/plextv.ts:230-257` `checkUserAccess(userId)`
```typescript
const user = users.find(u => parseInt(u.$.id) === userId);
return !!user.Server?.find(
  server => server.$.machineIdentifier === settings.plex.machineId
);
```

Plex.tv API 返回的 `User.Server[]` 是该用户被共享的服务器列表，每个 `Server.$` 包含：
- `id` / `serverId`
- `machineIdentifier`（UUID，与 Seerr 配置的 `settings.plex.machineId` 匹配即表示有权）
- `name` / `numLibraries` / `owned`

**注意**：`numLibraries` 只是展示字段，**Seerr 不检查具体哪些库被共享**——只要用户能访问该 Plex 服务器，就认为其有权通过 Seerr 请求**所有 Seerr 侧已启用的库**。

### 8.2 Seerr 侧的库过滤（全局配置，非用户级）

Seerr 有库的启用/禁用开关，但这是**全局配置**，不是用户级权限：

`server/routes/settings/index.ts:232-254`（Plex）与 `routes/settings/index.ts:333-393`（Jellyfin）：
```typescript
settings.plex.libraries = settings.plex.libraries.map(library => ({
  ...library,
  enabled: enabledLibraries.includes(library.id), // 管理员在设置页勾选
}));
```

库过滤仅作用于**扫描器**（`server/lib/scanners/plex/index.ts:81-82`）：
```typescript
this.libraries = settings.plex.libraries.filter(
  library => library.enabled
);
```
即：禁用的库不会被扫描入库，所有用户都看不到这些库的内容。这是全局开关，不是按用户分配权限。

### 8.3 Jellyfin 侧权限的 Seerr 侧判定

**仅检查是否是管理员，忽略其他 Policy 字段。**

Jellyfin 登录返回的 `account.User.Policy` 定义在 `server/api/jellyfin.ts:19-22`：
```typescript
Policy: {
  IsAdministrator: boolean;
  // Jellyfin 实际返回的还有：EnableContentDownloading、EnableMediaPlayback、
  // EnableSync、EnableAllLibraries、EnabledFolders、ExcludeFolders 等
  // 但 Seerr 的 interface 里只声明了 IsAdministrator，其他字段直接忽略
}
```

仅在**首次初始化**时判断：`server/routes/auth.ts:320-322`
```typescript
if (account.User.Policy.IsAdministrator === false) {
  throw new ApiError(403, ApiErrorCode.NotAdmin);
}
```
日常登录和请求流程中，**完全不检查 Jellyfin Policy 的任何字段**。

### 8.4 邀请时的权限传递（实际上没有传递）

三种邀请途径在创建用户时，都**不会从媒体服务器侧拉取权限映射到 Seerr**：

1. **管理员创建本地用户**（`POST /api/v1/user`）：权限固定为 `settings.main.defaultPermissions`，与媒体服务器无关
2. **批量从 Plex 导入**（`POST /api/v1/user/import-from-plex`）：同样写入 `settings.main.defaultPermissions`，不考虑 Plex 侧该用户是否有"邀请其他人"或"管理库"的权限
3. **自助登录自动创建**（`auth.ts:163-183` / `auth.ts:454-483`）：权限也是 `settings.main.defaultPermissions`

> 唯一例外：**首个用户**（数据库为空时登录）会被授予 `Permission.ADMIN`，这是基于"首次配置"的约定，不是从媒体服务器映射而来。

### 8.5 权限模型总结图

```
Plex 侧权限                      Seerr 侧权限
─────────────                   ─────────────
共享某台服务器 ────────────────► checkUserAccess() = true
  │                                    │
  ├─ 共享哪些库（Seerr 不读）           ├─ 全局 enabled libraries（管理员设）
  ├─ 是否能邀请他人（Seerr 不读）        ├─ defaultPermissions（全局默认）
  └─ Plex Pass 状态（Seerr 不读）        └─ 管理员手动调整 permissions 位掩码

Jellyfin 侧权限                   Seerr 侧权限
────────────────                   ─────────────
IsAdministrator ────────────────► 仅首次初始化用（是否能成为 Owner）
EnableAllLibraries ────────────► 完全忽略
EnabledFolders ────────────────► 完全忽略
EnableContentDownloading ───────► 完全忽略
EnableMediaPlayback ────────────► 完全忽略
```

---

## 九、用户配额（Quota）系统

配额系统限制用户在一定周期内的请求数量（电影按"部"计，剧集按"季"计），与邀请流程的关联在于：新建用户时即按全局默认或管理员指定写入配额参数。

### 9.1 数据模型：User 实体中的配额字段

定义于 `server/entity/User.ts:125-135`：

| 字段 | 含义 | null 语义 |
|---|---|---|
| `movieQuotaLimit` | 周期内允许请求的电影数量（整数） | 用全局默认值 |
| `movieQuotaDays` | 电影配额的统计周期（天数） | 用全局默认值 |
| `tvQuotaLimit` | 周期内允许请求的电视剧**季**数 | 用全局默认值 |
| `tvQuotaDays` | 电视剧配额的统计周期（天数） | 用全局默认值 |

**全局默认配置**：`settings.main.defaultQuotas.movie.{quotaLimit, quotaDays}` 与 `settings.main.defaultQuotas.tv.{quotaLimit, quotaDays}`。

> 配额字段可在管理员创建用户（`POST /api/v1/user`）或编辑用户（`PUT /api/v1/user/:id`）时单独指定，覆盖全局默认。

### 9.2 配额计算：`User.getQuota()`

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

#### 9.2.1 配额预占用机制（核心设计）

**"预占用"就是 `status != DECLINED` 这条规则本身**——只要请求没被明确拒绝，就会计入配额占用，这是该项目配额系统最核心的设计。

完整状态对应表（`server/entity/MediaRequest.ts:529-531` 的 `MediaRequestStatus` 枚举）：

| 状态 | 占配额？ | 说明 |
|---|---|---|
| `PENDING` (1) | ✅ 是 | 用户提交，等待审批 |
| `APPROVED` (2) | ✅ 是 | 已批准，等待下载 |
| `DECLINED` (3) | ❌ 否 | 被拒绝，配额释放 |
| `FAILED` (4) | ✅ 是 | 下载失败 |
| `PROCESSING` (5) | ✅ 是 | 正在下载 |
| `COMPLETED` (6) | ✅ 是 | 已完成 |

**关键行为**：
1. **用户提交即占用**（`MediaRequest.ts:459-521` `request()` 方法中 new 出来就是 PENDING 或 APPROVED 状态，立即计入配额）
2. **拒绝才归还**（只有管理员点"Decline"将状态改为 DECLINED，配额才释放）
3. **滚动窗口过期自然释放**（若设置了 quotaDays，超过 N 天的请求自动退出统计窗口）
4. **删除请求也释放**（`MediaRequestSubscriber.afterRemove()` 触发父状态更新，但配额归还靠的是记录从数据库消失，不在统计范围内了）

### 9.3 请求拦截点：`MediaRequest.request()`

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

### 9.4 HTTP 层的错误传播

`server/routes/request.ts:321-324` 按错误类型映射状态码：
```typescript
case QuotaRestrictedError:
  return next({ status: 403, message: error.message });
```
前端可据此展示"配额超限"提示。

#### 9.4.1 状态变化时的配额自动归还

当请求状态被管理员修改时，配额会自动重算（因为 `getQuota()` 是实时查询，不是缓存）：

**管理员拒绝请求**：`POST /api/v1/request/:id/decline`（`server/routes/request.ts:672-694`）
- 将 `request.status` 改为 `MediaRequestStatus.DECLINED`
- 触发 `MediaRequestSubscriber.afterUpdate()` → `updateParentStatus()`（`MediaRequestSubscriber.ts:820-904`）
- 拒绝的同时将所有 `SeasonRequest.status` 也改为 DECLINED（`MediaRequestSubscriber.ts:890-904`）
- 下次 `getQuota()` 查询时，`status != DECLINED` 条件自动排除该请求 → **配额自动归还**

**管理员批准请求**：状态改为 APPROVED，配额仍被占用（因为 `status != DECLINED` 仍然成立），没有归还。

**删除用户时**：`requestRepository.remove(user.requests)` 批量删除请求，`afterRemove` 事件触发父 Media 状态更新，但配额归还是因为记录消失了。

### 9.5 与邀请流程的配合

邀请/创建用户时的配额写入：
- 管理员点"Create Local User"创建：前端 `src/components/UserList/index.tsx` 弹窗中的配额字段传入后端
- 后端 `POST /api/v1/user`（`server/routes/user/index.ts:170-228`）按 body 赋值，未传时使用数据库列默认（null），真正生效时由 `getQuota()` 回退到全局默认
- 批量从 Plex/Jellyfin 导入的用户：配额字段全部不写，统一走全局默认 `defaultQuotas`
- 外部用户自助登录首次创建账号（`auth.ts` 中 `new User({...})`）：配额字段同样留空，按全局默认执行

---

## 十、邀请生命周期事件审计

> 澄清：该项目**没有独立的审计日志数据库表**（没有 `audit_log` / `event_log` 表），事件审计完全通过 **winston logger 写入日志文件** 实现。

### 10.1 日志基础设施

定义于 `server/logger.ts:1-75`，winston 配置双通道输出：

| 日志通道 | 文件名 | 格式 | 保留策略 | 用途 |
|---|---|---|---|---|
| seerr | `seerr-%DATE%.log` | 人类可读文本 | 7天轮转，单文件20MB | 业务日志 |
| machine | `.machinelogs-%DATE%.json` | JSON | 1天轮转，单文件20MB | 机器解析日志 |

日志级别默认 `debug`，可通过 `LOG_LEVEL` 环境变量覆盖。

### 10.2 邀请相关的关键审计事件

以下是邀请/用户生命周期各阶段的日志触发点：

#### 10.2.1 用户创建阶段

| 事件 | 代码位置 | 日志级别 | 关键字段 |
|---|---|---|---|
| 管理员创建本地用户成功 | `server/routes/user/index.ts:222` 前后（无显式日志） | - | 无 |
| 批量导入 Plex 用户成功 | `server/routes/user/index.ts:711` 前后（无显式日志） | - | 无 |
| 批量导入 Jellyfin 用户成功 | `server/routes/user/index.ts:791` 前后（无显式日志） | - | 无 |
| 自助登录首次创建 Plex 用户 | `server/routes/auth.ts:147-183`（创建成功无日志） | - | 无 |
| 自助登录首次创建 Jellyfin 用户 | `server/routes/auth.ts:454-483`（创建成功无日志） | - | 无 |
| 重置密码链接发送成功 | `server/routes/auth.ts:739` | `info` | `email, label: 'Auth API'` |
| 重置密码链接发送失败（邮箱不存在） | `server/routes/auth.ts:749` | `error` | `email, label: 'Auth API'` |

> **重要发现**：用户创建（邀请）成功本身**没有日志记录**，只有失败/异常路径有日志。这是设计缺陷，审计能力较弱。

#### 10.2.2 用户登录阶段

| 事件 | 代码位置 | 日志级别 | 关键字段 |
|---|---|---|---|
| Plex 登录未导入 + newPlexLogin=false | `server/routes/auth.ts:149-158` | `warn` | `ip, plexId, plexUsername, email, label: 'Auth API'` |
| Plex 登录但无服务器访问权限 | `server/routes/auth.ts:187-195` | `warn` | `ip, plexId, plexUsername, email, label: 'Auth API'` |
| 本地登录密码错误 | `server/routes/auth.ts:615-624` | `warn` | `ip, email, userId, label: 'Auth API'` |
| 本地登录成功 | `server/routes/auth.ts:636` 前后（无显式日志） | - | 无 |

#### 10.2.3 请求审批阶段（邀请后用户行为）

| 事件 | 代码位置 | 日志级别 | 关键字段 |
|---|---|---|---|
| 请求被拉黑媒体拦截 | `server/entity/MediaRequest.ts:144-151` | `warn` | `tmdbId, mediaType, label: 'Media Request'` |
| 重复请求拦截 | `server/entity/MediaRequest.ts:188-197` | `warn` | `tmdbId, mediaType, is4k, label: 'Media Request'` |
| Plex 设备获取失败 | `server/api/plextv.ts:206-210` | `error` | `errorMessage, label: 'Plex.tv API'` |
| 删除用户失败 | `server/routes/user/index.ts:643-647` | `error` | `userId, message, label: 'User API'` |

#### 10.2.4 账户关联/解绑阶段

| 事件 | 代码位置 | 日志级别 | 关键字段 |
|---|---|---|---|
| 关联 Plex 失败（邮箱不匹配/已绑定） | `server/routes/user/usersettings.ts:276-300` | 无显式日志（直接抛错） | - |
| 关联 Jellyfin 失败 | `server/routes/user/usersettings.ts:375-445` | 无显式日志（直接抛错） | - |
| 关联 Plex/Jellyfin 成功 | 无日志 | - | - |
| 解绑成功 | 无日志 | - | - |

### 10.3 审计缺陷总结

1. **无持久化审计表**：日志文件轮转 7 天后丢失，无法做长期审计
2. **成功路径几乎无日志**：创建用户、登录成功、关联账户等正常操作均不记录
3. **无操作人记录**：管理员创建用户、审批请求等操作未记录操作人（modifiedBy 字段只存于 MediaRequest，用户操作没有）
4. **无结构化事件类型**：日志 message 是自由文本，难以做聚合分析
5. **无 IP 地址全局记录**：仅登录失败和少量错误路径记录了 ip，大部分操作无 IP

---

## 十一、Plex Home 子账号与主账号的关联

> 澄清：该项目**在数据模型中识别了 Plex Home 关系，但在业务逻辑中完全没有使用它**。Plex Home 子账号与主账号在 Seerr 中是**完全独立的两个用户**，没有任何关联逻辑。

### 11.1 数据层识别：PlexDevice 中的 home / ownerID 字段

Plex.tv API 返回的设备列表（`/api/resources`）中，每个 Device 包含 `home` 和 `ownerID` 字段：

**接口定义**：
- `server/interfaces/api/plexInterfaces.ts:40-41`：`ownerID?: string; home?: boolean;`
- `server/api/plextv.ts:67-68`（原始 XML 字段）：`ownerID?: string; home?: string;`

**解析代码**（`server/api/plextv.ts:194-195`）：
```typescript
ownerID: pxml.$?.ownerID,
home: pxml.$?.home == '1' ? true : false,
```

字段含义（Plex 官方语义）：
- `home: true`：该设备属于 Plex Home 网络
- `ownerID`：Plex Home 主账号的用户 ID（子账号的设备会带上主账号的 ownerID）

### 11.2 Plex Home 字段的实际使用情况

在整个代码库中搜索 `\.home` 和 `ownerID`：
- `server/interfaces/api/plexInterfaces.ts:40-41`：接口声明
- `server/api/plextv.ts:67-68`：XML 字段声明
- `server/api/plextv.ts:194-195`：解析赋值
- **没有任何地方读取这两个字段用于业务判断**

**前端也不展示**：
- `src/components/UserList/index.tsx`：用户列表只展示 username、email、userType、permissions
- `src/pages/users/[userId]/settings/linked-accounts.tsx`：关联账户页只展示 plexUsername / jellyfinUsername

### 11.3 实际行为：Plex Home 子账号与主账号完全独立

假设 Plex Home 配置如下：
- 主账号 A（plexId=100，ownerID=100，home=true）
- 子账号 B（plexId=200，ownerID=100，home=true）

在 Seerr 中的表现：

| 场景 | 行为 |
|---|---|
| A 先登录 | 创建用户 User1（plexId=100） |
| B 再登录（用 B 自己的 PIN 授权） | 创建**独立**的 User2（plexId=200），**不会**关联到 User1 |
| 管理员给 User1 管理员权限 | User2 仍是普通用户，权限不共享 |
| User1 有 10 部电影配额 | User2 配额独立计算 |
| 删除 User1 | User2 不受影响 |
| User1 请求了电影 X | User2 仍可请求电影 X（不会被判定为重复） |

### 11.4 为什么没有关联逻辑

从代码设计推测的原因：
1. Seerr 的权限模型是按 Seerr 用户独立管理的，不继承 Plex 的权限关系
2. Plex Home 子账号本身有独立的 Plex ID 和邮箱，可以独立登录 Seerr
3. 没有"家庭组共享配额"、"主账号代子账号审批"等需求场景
4. 识别 home/ownerID 可能是为了未来扩展，目前只是"透传但不使用"

---

## 十二、会话与鉴权中间件

### 12.1 用户注入中间件

`server/middleware/auth.ts:9-41` `checkUser`

```
1. 若请求头 X-API-Key 匹配全局 API Key：
   - 以 id=1（主管理员）身份操作，或按 X-API-User 指定用户
2. 否则若 req.session.userId 存在：
   - 从数据库加载用户，挂到 req.user
3. 设置 req.locale（用户设置或全局默认）
```

### 12.2 权限校验中间件

`server/middleware/auth.ts:43-58` `isAuthenticated(permissions?, options?)`

- 未登录 → 403
- 缺少指定权限 → 403
- 位运算：`hasPermission(permissions, this.permissions, options)`

### 12.3 单点登出（SLO）与邀请会话清理

> 澄清：该项目**没有 SSO 单点登出（Single Logout）机制**。没有 SAML/OIDC 全局登出回调，也没有"一处登出，所有设备失效"的功能。

#### 12.3.1 登出接口行为

接口：`POST /api/v1/auth/logout`，实现于 `server/routes/auth.ts:648-716`

登出时按媒体服务器类型做不同清理：

| 登录方式 | Session 销毁 | 媒体服务器侧清理 |
|---|---|---|
| Plex OAuth | ✅ `req.session.destroy()` | ❌ 不通知 Plex.tv 失效 Token |
| Jellyfin / Emby | ✅ `req.session.destroy()` | ✅ 调用 `DELETE /Devices` 删除 `jellyfinDeviceId` 对应的设备 |
| Local | ✅ `req.session.destroy()` | ❌ 无外部服务 |

#### 12.3.2 Jellyfin 设备注销详细流程

代码段（`server/routes/auth.ts:660-698`）：

```
1. 判断是否 Jellyfin/Emby 媒体服务器
2. 读取用户的 jellyfinUserId 和 jellyfinDeviceId
3. 构造 X-Emby-Authorization 头（用管理员 API Key）
4. axios.delete(`${baseUrl}/Devices?Id=${jellyfinDeviceId}`)
5. 无论成功失败都继续销毁 Session
6. 失败时记录 error 日志
```

**注意**：
- Jellyfin 的 `jellyfinAuthToken`（即用户登录后拿到的 AccessToken）**不会**被显式失效，只是删除了设备记录
- 同一用户在多个浏览器/设备上登录会产生多个 Session，登出一个**不会**影响其他 Session
- 没有"全局强制下线"管理员功能

#### 12.3.3 对"邀请会话"的影响

> 不存在"邀请会话"这个概念。邀请（用户创建）和会话（登录状态）是完全独立的两件事：

1. **邀请不会产生 Session**：用户被创建后只是数据库中有一条记录，不自动登录
2. **登出不会撤回邀请**：用户登出只是销毁当前 Session，用户记录仍在数据库，下次仍可登录
3. **用户删除 ≠ 登出**：删除用户时 CASCADE 掉所有关联数据，但 Session 表中可能残留该用户的 session（TTL 到期后由 TypeormStore 清理）

#### 12.3.4 Plex Token 的"隐式失效"

Plex 登录拿到的 `plexToken` 存储在用户表中，用于后续 Watchlist 同步等后台任务。这个 token **不会**因为用户登出而失效：

- 用户登出 Seerr → 本地 Session 销毁 → 但 `user.plexToken` 仍在数据库
- 下次登录时，Plex OAuth 流程会用新的 authToken 重新获取用户信息，然后**覆盖更新** `plexToken`
- 如果用户在 Plex.tv 侧撤销了 Seerr 的授权，后台的 Watchlist 同步任务会失败，但代码不会自动禁用该用户

---

## 十三、Plex Watchlist 集成与邀请触发同步

Plex Watchlist 同步是该项目中**唯一与"邀请后自动触发请求"相关的机制**。新用户被邀请后，如果开启了 Watchlist 同步，其 Plex Watchlist 中的内容会被自动请求，相当于"邀请触发了一批媒体请求"。

### 13.1 整体架构：定时任务 + 逐用户同步

**任务调度**：`server/job/schedule.ts:89-107`
```
Job ID: plex-watchlist-sync
类型: process
间隔: 可配置（默认短间隔秒级）
触发: node-schedule cron
执行: watchlistSync.syncWatchlist()
```
仅在 `mediaServerType === PLEX` 时注册该任务，Jellyfin/Emby 模式下不运行。

### 13.2 同步入口：`WatchlistSync.syncWatchlist()`

实现于 `server/lib/watchlistsync.ts:17-32`

```
1. 查找所有 plexToken 不为空的用户
   （即所有通过 Plex 登录或关联了 Plex 账号的用户）
2. 逐用户调用 syncUserWatchlist(user)
```

> 注意：这里遍历的是**全部有 Plex token 的用户**，包括管理员和普通用户。没有白名单或"只同步被邀请用户"的限制。

### 13.3 单用户同步逻辑：`syncUserWatchlist()`

实现于 `server/lib/watchlistsync.ts:34-197`

#### 13.3.1 前置检查（按顺序跳过）

| 检查项 | 代码位置 | 跳过条件 |
|---|---|---|
| 无 Plex Token | 第 35-41 行 | `!user.plexToken` → warn 日志 |
| 无 Auto-Request 权限 | 第 43-54 行 | 没有 `AUTO_REQUEST / AUTO_REQUEST_MOVIE / AUTO_REQUEST_TV` |
| 未开启同步开关 | 第 56-62 行 | `watchlistSyncMovies` 和 `watchlistSyncTv` 都为 false |

> **与邀请的关系**：新用户被创建时，`settings.watchlistSyncMovies/Tv` 默认为 false，需要用户自己在设置页打开。管理员导入用户时**不会**自动开启。

#### 13.3.2 数据拉取与比对

```typescript
// 1. 拉取 Plex Watchlist 前 20 条
const response = await plexTvApi.getWatchlist({ size: 20 });

// 2. 在 Seerr 本地库中匹配已存在的媒体
const mediaItems = await Media.getRelatedMedia(user, response.items.map(...));

// 3. 查出该用户已有的 auto-request（避免重复）
const existingAutoRequests = await requestRepository
  .createQueryBuilder('request')
  .where('request.requestedBy = :userId', { userId: user.id })
  .andWhere('request.isAutoRequest = true')
  .andWhere('media.tmdbId IN (:...tmdbIds)', { tmdbIds: watchlistTmdbIds })
  .getMany();

// 4. 过滤出"未请求 + 未入库 + 未拉黑"的条目
const unavailableItems = response.items.filter(...);
```

#### 13.3.3 自动创建请求

对每个 `unavailableItems` 中的条目：
```typescript
await MediaRequest.request(
  {
    mediaId: mediaItem.tmdbId,
    mediaType: movie/show 对应 MOVIE/TV,
    seasons: 'all',           // TV 默认请求全部季
    tvdbId: mediaItem.tvdbId,
    is4k: false,
  },
  user,
  { isAutoRequest: true }     // 标记为自动请求
);
```

**标记 `isAutoRequest: true` 的意义**：
- 去重：同步时只检查 `isAutoRequest=true` 的请求，避免和用户手动请求混淆
- 前端展示：用户可以在 Watchlist 页区分哪些是自动的
- 错误分级：同步时遇到配额超限、重复请求等错误只打 debug 日志，不中断流程

### 13.4 配额交互：邀请触发时的预占用

Watchlist 同步调用的是标准的 `MediaRequest.request()` 方法，**配额计算完全一致**：

1. 同步时每个自动请求都会调用 `getQuota()` 检查
2. 如果配额超限 → 抛出 `QuotaRestrictedError` → 被 catch 打 debug 日志 → **跳过该条继续下一条**
3. 成功创建的请求 → `status=PENDING 或 APPROVED` → 立即占用配额（预占用机制）
4. 如果管理员后续拒绝 → 状态变 DECLINED → 配额释放

**批量场景特点**：
- 假设用户剩余配额 3 条，Watchlist 有 10 条 → 前 3 条成功，后 7 条因配额超限跳过
- 下次同步时，如果配额还没恢复（滚动窗口没到或没被拒绝）→ 继续跳过
- 配额恢复后 → 下次同步时继续自动请求

### 13.5 本地 Watchlist 与 Plex Watchlist 的区别

该项目有**两个独立的 Watchlist 概念**，容易混淆：

| 维度 | Plex Watchlist | 本地 Seerr Watchlist |
|---|---|---|
| 数据存储 | Plex.tv 云端 | 本地数据库 `watchlist` 表 |
| 实体 | 无本地表 | `server/entity/Watchlist.ts` |
| 路由 | 无直接路由（通过同步间接操作） | `server/routes/watchlist.ts`（POST/DELETE） |
| 自动请求 | ✅ 是（同步后自动请求） | ❌ 否（只是收藏，不自动请求） |
| 与邀请关系 | 邀请后开启同步会自动触发一批请求 | 邀请本身不影响，用户手动维护 |
| 删除用户时 | 不影响 Plex 侧数据 | CASCADE 级联删除 |

> 注意：`MediaRequest.isAutoRequest = true` 的请求**来源于** Plex Watchlist 同步，但请求本身和本地 Watchlist 表没有直接关联。

### 13.6 邀请 → Watchlist 同步的完整触发链路

```
用户被邀请（三种途径之一）
   │
   ▼
用户首次 Plex 登录 / 关联 Plex 账号
   │
   ├─► user.plexToken 被写入数据库
   │
   ▼
用户在设置页打开 "Auto-request from Plex Watchlist"
   │
   ├─► user.settings.watchlistSyncMovies = true
   └─► user.settings.watchlistSyncTv = true
   │
   ▼
下一次定时任务触发（plex-watchlist-sync）
   │
   ├─► 遍历所有有 plexToken 的用户
   ├─► 检查权限和开关
   ├─► 拉取 Plex Watchlist 前 20 条
   ├─► 比对已存在/已请求/已拉黑
   └─► 对未处理的条目调用 MediaRequest.request({ isAutoRequest: true })
         │
         ├─► 配额检查（预占用）
         ├─► 成功 → PENDING 或 APPROVED，占用配额
         └─► 失败（配额超/重复/拉黑）→ 跳过，记日志
```

### 13.7 相关开关与权限

| 配置项 | 位置 | 默认值 | 说明 |
|---|---|---|
| `jobs['plex-watchlist-sync'].schedule` | 设置页 Jobs | 可配置 | cron 表达式 |
| `user.settings.watchlistSyncMovies` | 用户设置 | false | 电影自动同步开关 |
| `user.settings.watchlistSyncTv` | 用户设置 | false | 剧集自动同步开关 |
| `Permission.AUTO_REQUEST` | 权限位 | 含在 defaultPermissions 中 | 总开关 |
| `Permission.AUTO_REQUEST_MOVIE` | 权限位 | 含在 defaultPermissions 中 | 电影细分 |
| `Permission.AUTO_REQUEST_TV` | 权限位 | 含在 defaultPermissions 中 | 剧集细分 |

---

## 十四、完整协作流程图

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

## 十五、关键代码文件索引

| 功能 | 文件路径 |
|---|---|
| 用户类型常量 | `server/constants/user.ts` |
| 权限位枚举与 hasPermission 算法 | `server/lib/permissions.ts` |
| 日志系统（winston 双通道配置） | `server/logger.ts` |
| 定时任务调度（所有 cron job） | `server/job/schedule.ts` |
| Plex Watchlist 同步逻辑（核心） | `server/lib/watchlistsync.ts` |
| Watchlist 同步单测 | `server/lib/watchlistsync.test.ts` |
| 用户实体（含 getQuota、resetPassword、密码哈希、plexToken） | `server/entity/User.ts` |
| 会话实体（Session expiredAt 字段） | `server/entity/Session.ts` |
| 请求实体（含 QuotaRestrictedError、isAutoRequest、onDelete 级联） | `server/entity/MediaRequest.ts` |
| 本地 Watchlist 实体（与 Plex Watchlist 是两个概念） | `server/entity/Watchlist.ts` |
| 关注清单实体（onDelete: CASCADE） | `server/entity/Watchlist.ts` |
| 用户设置实体（watchlistSyncMovies/Tv 开关） | `server/entity/UserSettings.ts` |
| Issue/评论实体（onDelete 级联配置） | `server/entity/Issue.ts` |
| MediaRequest 事件订阅（afterRemove/updateParentStatus/状态级联） | `server/subscriber/MediaRequestSubscriber.ts` |
| Media 事件订阅（子请求状态同步） | `server/subscriber/MediaSubscriber.ts` |
| IssueComment 事件订阅（通知） | `server/subscriber/IssueCommentSubscriber.ts` |
| Plex 设备接口（home/ownerID 字段定义） | `server/interfaces/api/plexInterfaces.ts` |
| Watchlist 创建接口 Zod Schema | `server/interfaces/api/watchlistCreate.ts` |
| 认证路由（登录/登出/重置密码/Jellyfin 设备注销） | `server/routes/auth.ts` |
| 请求路由（QuotaRestrictedError → 403 映射、审批/拒绝） | `server/routes/request.ts` |
| 本地 Watchlist 路由（POST/DELETE） | `server/routes/watchlist.ts` |
| 用户管理路由（CRUD/导入/删除/配额字段） | `server/routes/user/index.ts` |
| 用户设置路由（密码/关联冲突校验/Watchlist 开关） | `server/routes/user/usersettings.ts` |
| 设置路由（libraries 全局启用/禁用、rate-limit、Jobs 配置） | `server/routes/settings/index.ts` |
| 鉴权中间件（Session 读取、权限校验） | `server/middleware/auth.ts` |
| Plex.tv API 封装（含 checkUserAccess、getWatchlist、home/ownerID） | `server/api/plextv.ts` |
| Jellyfin API 封装（Policy 接口定义、登录、Devices 删除） | `server/api/jellyfin.ts` |
| 服务器启动（Session TTL / maxAge 配置、clientIp 解析） | `server/index.ts` |
| Plex 扫描器（libraries enabled 过滤逻辑） | `server/lib/scanners/plex/index.ts` |
| Plex OAuth 前端逻辑（PIN 轮询） | `src/utils/plex.ts` |
| Plex 登录 Hook | `src/hooks/usePlexLogin.ts` |
| 前端登录页 | `src/components/Login/index.tsx` |
| 前端用户列表（创建/导入/配额表单） | `src/components/UserList/index.tsx` |
| 前端 Watchlist 页面 | `src/pages/users/[userId]/watchlist.tsx` |
