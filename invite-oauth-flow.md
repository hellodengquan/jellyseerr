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

## 六、会话与鉴权中间件

### 6.1 用户注入中间件

`server/middleware/auth.ts:9-41` `checkUser`

```
1. 若请求头 X-API-Key 匹配全局 API Key：
   - 以 id=1（主管理员）身份操作，或按 X-API-User 指定用户
2. 否则若 req.session.userId 存在：
   - 从数据库加载用户，挂到 req.user
3. 设置 req.locale（用户设置或全局默认）
```

### 6.2 权限校验中间件

`server/middleware/auth.ts:43-58` `isAuthenticated(permissions?, options?)`

- 未登录 → 403
- 缺少指定权限 → 403
- 位运算：`hasPermission(permissions, this.permissions, options)`

---

## 七、完整协作流程图

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

## 八、关键代码文件索引

| 功能 | 文件路径 |
|---|---|
| 用户类型常量 | `server/constants/user.ts` |
| 用户实体 | `server/entity/User.ts` |
| 认证路由（登录/登出/重置密码） | `server/routes/auth.ts` |
| 用户管理路由（CRUD/导入/配额） | `server/routes/user/index.ts` |
| 用户设置路由（密码/关联/通知/权限） | `server/routes/user/usersettings.ts` |
| 鉴权中间件 | `server/middleware/auth.ts` |
| Plex.tv API 封装（含 checkUserAccess） | `server/api/plextv.ts` |
| Jellyfin API 封装 | `server/api/jellyfin.ts` |
| Plex OAuth 前端逻辑 | `src/utils/plex.ts` |
| Plex 登录 Hook | `src/hooks/usePlexLogin.ts` |
| 前端登录页 | `src/components/Login/index.tsx` |
| 前端用户列表（创建/导入） | `src/components/UserList/index.tsx` |
