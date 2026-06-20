# Issue 报告与备注流转分析

## 一、核心实体关系

### 1. 数据模型定义

**Issue 实体** (`server/entity/Issue.ts`)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | number | 主键自增 |
| issueType | IssueType | 问题类型：VIDEO(1)/AUDIO(2)/SUBTITLES(3)/OTHER(4) |
| status | IssueStatus | 状态：OPEN(1)/RESOLVED(2)，默认 OPEN |
| problemSeason | number | 有问题的季（TV专用） |
| problemEpisode | number | 有问题的集（TV专用） |
| media | Media | 关联的媒体资源（多对一，eager加载） |
| createdBy | User | 创建者（多对一，eager加载） |
| modifiedBy | User | 最后修改者（可空） |
| comments | IssueComment[] | 备注列表（一对多，cascade + eager） |
| createdAt | Date | 创建时间 |
| updatedAt | Date | 更新时间 |

**IssueComment 实体** (`server/entity/IssueComment.ts`)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | number | 主键自增 |
| user | User | 评论用户（多对一，eager加载） |
| issue | Issue | 所属Issue（多对一） |
| message | string | 评论内容（text类型） |
| createdAt | Date | 创建时间 |
| updatedAt | Date | 更新时间 |

### 2. 实体关联图

```
User (1) ──── createdIssues ────< (N) Issue (N) ──── comments ────< (1) IssueComment (N) ──── user ────> (1) User
  │                                      │                                                      │
  │                                      │                                                      │
  └── modifiedBy (可空) ────────────────┘                                                      ┘

  Media (1) ─── issues (OneToMany, cascade: true) ───< (N) Issue (N) ─── media (ManyToOne, eager) ───> (1) Media
```

### 3. 数据库索引与外键

**Issue 表索引**（迁移文件 `server/migration/sqlite/1770627968781-AddPerformanceIndexes.ts`）

| 索引字段 | 说明 |
|----------|------|
| `issueType` | 无显式独立索引，依赖 status 过滤 |
| `status` | 无显式独立索引，列表查询用 |
| `mediaId` | 外键 + 隐式索引（SQLite 外键自动建索引） |
| `createdById` | 外键 + 隐式索引 |
| `modifiedById` | 外键 + 隐式索引 |

**IssueComment 表索引**

| 索引字段 | 说明 |
|----------|------|
| `issueId` | 外键 + 隐式索引，按 Issue 查评论用 |
| `userId` | 外键 + 隐式索引 |

**Media 表索引**

| 索引字段 | 说明 |
|----------|------|
| `tmdbId` + `mediaType` | 联合唯一索引，核心查询路径 |
| `status` / `status4k` | 独立索引 |
| `tvdbId` / `imdbId` | 独立索引 |

> 注：Issue 表没有独立的 `status` 单列索引（迁移文件里没看到），查询时依赖 `status IN (...)` 做过滤。

---

## 二、媒体关联与双向反查

### 1. 关联定义

**Media → Issue（一对多）**：`server/entity/Media.ts:124-125`
```typescript
@OneToMany(() => Issue, (issue) => issue.media, { cascade: true })
public issues: Issue[];
```
- `cascade: true`：保存/删除 Media 时级联操作关联的 Issues
- 非 eager 加载，需要显式 `relations: { issues: true }` 才会加载

**Issue → Media（多对一）**：`server/entity/Issue.ts:36-41`
```typescript
@ManyToOne(() => Media, (media) => media.issues, {
  eager: true,
  onDelete: 'CASCADE',
})
@Index()
public media: Media;
```
- `eager: true`：每次查 Issue 自动带出 Media
- `onDelete: 'CASCADE'`：Media 删了，关联的 Issue 也被删
- `@Index()`：在 mediaId 上加索引

### 2. 反查路径：从媒体查关联 Issue

**路径 A：Media.getMedia() 静态方法**（`server/entity/Media.ts:65-82`）
```typescript
static async getMedia(id: number, mediaType: MediaType): Promise<Media | undefined> {
  const media = await mediaRepository.findOne({
    where: { tmdbId: id, mediaType: mediaType },
    relations: { requests: true, issues: true },  // ← 主动加载 issues
  });
  return media ?? undefined;
}
```
这是前端详情页的标准查询入口——查电影/剧集详情时，一并带出所有关联的 issues。

**路径 B：推荐列表批量反查**（`server/entity/Media.ts:32-63`）
`Media.getRelatedMedia()` 批量查媒体数据，**不带 issues**，只用于列表展示。

**路径 C：Issue 列表按 createdBy 过滤**（`server/routes/issue.ts:56-64`）
Issue 查询页走 `issueRepository.createQueryBuilder('issue')`，关联 media 用于展示，不依赖 Media 反查。

### 3. 实际使用场景

| 页面/接口 | 方向 | 加载方式 |
|----------|------|----------|
| 电影/剧集详情页 | Media → Issues | `relations: { issues: true }` 一次查出 |
| Issue 列表页 | Issue → Media | `eager: true` 自动带出 |
| Issue 详情页 | Issue → Media | `eager: true` 自动带出 |

> 注意：Media 上的 `issues` 属性默认**不加载**，只有显式调用 `relations` 或 `leftJoinAndSelect` 才会返回。这也是为什么从 Issue 侧看总是有 media（eager），但从 Media 侧看不一定有 issues。

---

## 三、Issue 创建流程详解

### 1. 创建入口：POST /issue

**路由位置**：`server/routes/issue.ts:102-165`

#### 流程步骤：

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 权限校验                                                      │
│    需要: MANAGE_ISSUES 或 CREATE_ISSUES (任一即可)               │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 确定创建者 (createdBy)                                        │
│    ├─ 请求体含 userId 且 ≠ 当前用户id                            │
│    │   └─ 需要 MANAGE_ISSUES 权限，否则 403                       │
│    │   └─ 查库验证 userId 对应用户是否存在，否则 404              │
│    └─ 否则使用当前登录用户作为 createdBy                          │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 验证媒体存在性                                                │
│    根据 mediaId 查询 Media，不存在则返回 404                       │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 构造 Issue + 第一条 Comment（级联保存）                       │
│    Issue {                                                        │
│      createdBy, issueType, problemSeason, problemEpisode, media, │
│      comments: [                                                 │
│        IssueComment { user: createdBy, message: req.body.message }│
│      ]                                                           │
│    }                                                             │
│    ⚠️ 关键点：创建Issue时，消息内容作为**第一条Comment**保存        │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. issueRepository.save(issue)                                   │
│    级联插入 Issue + IssueComment 两条记录                         │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. TypeORM afterInsert 钩子触发                                  │
│    IssueSubscriber.afterInsert() → 发送 ISSUE_CREATED 通知       │
└─────────────────────────────────────────────────────────────────┘
```

### 2. 创建时的关键设计：Issue 与第一条 Comment 强绑定

用户提交 Issue 时的 `message` 字段并不会直接存在 Issue 上，而是**立即转化为 Issue 的第一条 Comment**。这意味着：

- **Issue 本身不存储描述文本**，描述信息全在 comments[0]
- 删除 Issue 时会级联删除所有 Comments
- 后续发送通知时，取 `sortBy(entity.comments, 'id')[0]` 作为问题描述

---

## 四、备注（Comment）流转全路径

### 1. Comment 的三种创建/修改/删除方式

#### 方式 A：Issue 创建时隐式创建（上文已述）
- 路由：`POST /issue`
- Comment.user = Issue.createdBy
- 不会触发 ISSUE_COMMENT 通知（见下文通知过滤逻辑）

#### 方式 B：在已有 Issue 下新增 Comment
- **路由**：`POST /issue/:issueId/comment` (`server/routes/issue.ts:280-325`)
- **权限**：MANAGE_ISSUES 或 (CREATE_ISSUES 且是 Issue 创建者)
- **流程**：

```
验证 Issue 存在
   → 权限校验（创建者可评 / 管理员可评）
   → new IssueComment { message, user: req.user }
   → issue.comments = [...old, comment]  + 更新 updatedAt
   → save(issue) 级联保存
   → IssueCommentSubscriber.afterInsert 触发通知
```

#### 方式 C：独立 Comment 路由操作

**路由文件**：`server/routes/issueComment.ts`

| 操作 | 方法 | 路径 | 权限与限制 |
|------|------|------|------------|
| 查看单条 | GET | `/comment/:commentId` | 管理员 / 自己的评论 |
| 编辑 | PUT | `/comment/:commentId` | **只能编辑自己的**（即使管理员也不能改别人的） |
| 删除 | DELETE | `/comment/:commentId` | 管理员 / 自己的评论 |

### 2. Comment 的关键约束

1. **编辑约束最强**：即使有 MANAGE_ISSUES 权限，也只能编辑自己发布的评论（`issueComment.ts:71-76`）
2. **删除约束较松**：管理员可以删任何人的评论
3. **Issue 删除的特殊限制**：普通用户删除 Issue 时，若该 Issue 已有 **超过 1 条 Comment**（即除了自己创建时的第一条外还有别人评论过），则不允许删除（`issue.ts:402-404`）

---

## 五、状态流转与自动关闭机制

### 1. 状态枚举

```typescript
enum IssueStatus {
  OPEN = 1,      // 未解决
  RESOLVED = 2,  // 已解决
}
```

### 2. 状态变更入口

**路由**：`POST /issue/:issueId/:status` (`server/routes/issue.ts:327-385`)

`status` 路径参数只能是 `'resolved'` 或 `'open'`，对应状态值切换。

**权限**：
- MANAGE_ISSUES 权限者可任意改
- Issue 创建者只能改自己的 Issue

### 3. 完整生命周期

```
         POST /issue 创建
               │
               ▼
     ┌──────────────────┐
     │   status: OPEN   │
     │  modifiedBy: null│
     └────────┬─────────┘
              │
              │ POST /issue/:id/resolved
              │ modifiedBy = 当前用户
              ▼
     ┌──────────────────┐
     │ status: RESOLVED │
     └────────┬─────────┘
              │
              │ POST /issue/:id/open   (重新打开)
              │ modifiedBy = 当前用户
              ▼
     ┌──────────────────┐
     │   status: OPEN   │
     └──────────────────┘
```

### 4. ⚠️ 不存在"媒体可用后自动关闭 Issue"机制

经过全代码库搜索（`availabilitySync`、`MediaSubscriber`、各 scanner、设置项），**确认 Jellyseerr 当前没有这个功能**。证据如下：

| 搜索维度 | 结果 |
|----------|------|
| 关键词 `autoCloseIssue`/`autoResolve` | 0 命中 |
| `availabilitySync.ts` 中 issue 关键词 | 0 命中（仅处理 MediaRequest 状态） |
| `MediaSubscriber.ts` 中 issue 关键词 | 0 命中（仅处理 MediaRequest 状态） |
| 设置项 `settings/index.ts` 中 issue | 0 命中 |
| 前端页面 issue 设置 | 无自动关闭相关 UI |

**对比：MediaRequest 是有自动处理的**。`MediaSubscriber.afterUpdate()` 会在媒体状态变为 AVAILABLE 时，把关联的 MediaRequest 标记为 COMPLETED。但同样的逻辑**没有延伸到 Issue**。

**结论**：Issue 的状态变更只能通过用户手动调用 `POST /issue/:id/resolved` 或 `POST /issue/:id/open` 触发，没有任何自动关闭/自动解决机制。这是一个设计上的空白，而不是代码藏得深。

---

## 六、通知系统：Issue、备注、状态的交汇点

### 1. 通知类型枚举（位标志）

**定义**：`server/lib/notifications/index.ts:6-20`

```typescript
enum Notification {
  ISSUE_CREATED   = 256,   // 0b0001_0000_0000
  ISSUE_COMMENT   = 512,   // 0b0010_0000_0000
  ISSUE_RESOLVED  = 1024,  // 0b0100_0000_0000
  ISSUE_REOPENED  = 2048,  // 0b1000_0000_0000
}
```

> 使用位运算（`&`）判断用户是否开启了某类通知，见 `hasNotificationType()` 函数。

### 2. 通知触发点（TypeORM Subscribers）

系统通过两个 EventSubscriber 监听数据库写入事件，**解耦了业务代码与通知逻辑**。

#### Subscriber 1：IssueSubscriber（监听 Issue 表变化）
**文件**：`server/subscriber/IssueSubscriber.ts`

| 钩子 | 触发时机 | 发送的通知类型 |
|------|----------|----------------|
| `afterInsert` | Issue 新建完成 | `ISSUE_CREATED` |
| `beforeUpdate` | Issue 更新时，且 **status 字段发生变更** | OPEN→RESOLVED 发 `ISSUE_RESOLVED`；反之发 `ISSUE_REOPENED` |

⚠️ `beforeUpdate` 的关键判断：
```typescript
// 必须是"状态变化"才发通知，普通修改（如添加 comment 时的 updatedAt 变化）不发
if (newStatus === RESOLVED && databaseEntity.status !== RESOLVED) { ... }
if (newStatus === OPEN     && databaseEntity.status !== OPEN)     { ... }
```

#### Subscriber 2：IssueCommentSubscriber（监听 IssueComment 表变化）
**文件**：`server/subscriber/IssueCommentSubscriber.ts`

| 钩子 | 触发时机 | 发送的通知类型 |
|------|----------|----------------|
| `afterInsert` | Comment 新建完成 | `ISSUE_COMMENT`，但有排除条件 |

⚠️ **关键过滤逻辑**：如果当前插入的 Comment 是该 Issue 的 **第一条 Comment**（即 id === sortBy(issue.comments, 'id')[0].id），则**跳过通知**。

因为第一条 Comment 是创建 Issue 时附带的，此时 IssueSubscriber 已经发送了 `ISSUE_CREATED`，不需要再发重复的 `ISSUE_COMMENT`。

### 3. 通知接收者判定逻辑

每次调用 `notificationManager.sendNotification(type, payload)` 时，payload 中三个字段决定谁会收到通知：

| Payload 字段 | 含义 |
|--------------|------|
| `notifySystem: true` | 发送到系统通知中心（站内信） |
| `notifyAdmin: true` | 发送给所有拥有 MANAGE_ISSUES 权限的管理员 |
| `notifyUser?: User` | 可选：单独发送给某个普通用户 |

#### 管理员通知的进一步过滤

`shouldSendAdminNotification()` 函数 (`notifications/index.ts:67-90`) 会**排除事件的触发者本人**，避免"自己操作自己收到通知"的尴尬：

| 通知类型 | 排除的人 |
|----------|----------|
| ISSUE_CREATED | Issue 创建者本人 |
| ISSUE_COMMENT | 发表评论的用户 |
| ISSUE_RESOLVED / ISSUE_REOPENED | 修改状态的人（modifiedBy） |

#### 普通用户（notifyUser）触发条件

仅在以下场景会单独通知 Issue 创建者：

**场景 A：状态变更通知**（IssueSubscriber.ts:88-94）
```
notifyUser = createdBy，当且仅当：
  1. createdBy 不是管理员（!hasPermission(MANAGE_ISSUES)）
  2. 修改者不是创建者自己（modifiedBy.id !== createdBy.id）
  3. 类型是 ISSUE_RESOLVED 或 ISSUE_REOPENED
```
→ 即：**普通用户创建的 Issue 被管理员解决/重开时，通知创建者**

**场景 B：新评论通知**（IssueCommentSubscriber.ts:76-80）
```
notifyUser = createdBy，当且仅当：
  1. createdBy 不是管理员
  2. 发表评论的人不是 createdBy 自己
```
→ 即：**别人在普通用户创建的 Issue 下评论时，通知创建者**

### 4. 通知流程图

```
用户操作                     DB操作                 Subscriber                  通知接收者
─────────                   ────────               ──────────                 ──────────
POST /issue             →  INSERT Issue +
                          INSERT Comment(第1条)  →  IssueSubscriber
                                                       .afterInsert()
                                                    └→ ISSUE_CREATED ──────── 管理员们（排除创建者）

POST /issue/:id/comment →  INSERT Comment(N)     →  IssueCommentSubscriber
                                                       .afterInsert()
                                                       │ 跳过第1条
                                                    └→ ISSUE_COMMENT ──────── 管理员们（排除评论者）
                                                                              + Issue创建者（若普通用户且非评论者）

POST /issue/:id/resolved → UPDATE Issue.status  →  IssueSubscriber
                          (OPEN→RESOLVED)           .beforeUpdate()
                                                       │ status确实变化
                                                    └→ ISSUE_RESOLVED ─────── 管理员们（排除修改者）
                                                                              + Issue创建者（若普通用户且非修改者）

POST /issue/:id/open     → UPDATE Issue.status  →  IssueSubscriber
                          (RESOLVED→OPEN)           .beforeUpdate()
                                                    └→ ISSUE_REOPENED ─────── 同上
```

### 5. 通知发送的完整链路

#### 总览：三层分发架构

```
Subscriber（触发源）
     │
     ▼
NotificationManager（调度中心）
     │
     ├─→ EmailAgent
     ├─→ DiscordAgent
     ├─→ WebPushAgent
     ├─→ TelegramAgent
     ├─→ SlackAgent
     ├─→ PushoverAgent
     ├─→ PushbulletAgent
     ├─→ GotifyAgent
     ├─→ NtfyAgent
     └─→ WebhookAgent
```

#### 第一层：Subscriber 构造 Payload

以 `IssueSubscriber.sendIssueNotification()` 为例，它构造 `NotificationPayload` 并调用：

```typescript
notificationManager.sendNotification(type, {
  event: "New Video Issue Reported",
  subject: "电影名称",
  message: "第一条评论内容",
  issue: entity,
  media: entity.media,
  image: "海报URL",
  extra: [{ name: "Affected Season", value: "2" }],
  notifyAdmin: true,      // 通知管理员组
  notifySystem: true,     // 系统级通知开关
  notifyUser: createdBy,  // 可选：单独通知某用户
});
```

#### 第二层：NotificationManager 分发给所有 Agent

**位置**：`server/lib/notifications/index.ts:92-115`

```typescript
class NotificationManager {
  private activeAgents: NotificationAgent[] = [];

  sendNotification(type: Notification, payload: NotificationPayload): void {
    this.activeAgents.forEach((agent) => {
      if (agent.shouldSend()) {
        agent.send(type, payload);  // 每个 agent 自行决定怎么发、发给谁
      }
    });
  }
}
```

Manager 不关心具体发给谁，只做两件事：
1. 遍历所有已注册的 agent
2. 对每个 agent 问一句 `shouldSend()`，返回 true 就调用 `send()`

#### 第三层：各 Agent 自行处理接收者

**每个 agent 的 `send()` 方法都包含完整的接收者筛选逻辑**，这也是抄送/订阅机制的核心。以 EmailAgent 为例：

```typescript
async send(type, payload) {
  // 通路 A：单独通知指定用户（notifyUser）
  if (payload.notifyUser) {
    if (user.settings?.hasNotificationType(EMAIL, type)) {
      // 给这个用户发邮件
    }
  }

  // 通路 B：通知所有管理员（notifyAdmin）
  if (payload.notifyAdmin) {
    const allUsers = await userRepository.find();
    allUsers
      .filter(user => 
        user.hasPermission(MANAGE_ISSUES) &&          // 是管理员
        shouldSendAdminNotification(type, user, payload) &&  // 排除触发者
        user.settings?.hasNotificationType(EMAIL, type)       // 订阅了该类型
      )
      .forEach(user => { /* 发邮件 */ });
  }
}
```

### 6. 用户通知偏好（订阅机制）

#### 存储结构

**位置**：`server/entity/UserSettings.ts`

每个用户有一个 `UserSettings` 实体，其中 `notificationTypes` 字段存储各通知渠道的订阅偏好：

```typescript
// 各 agent 的通知类型用位标志存储
notificationTypes: {
  email: 4095,     // 全量默认开启（所有类型）
  discord: 0,      // 默认全关
  webpush: 4095,   // 默认全开
  slack: 0,
  telegram: 0,
  pushover: 0,
  pushbullet: 0,
  webhook: 0,
  gotify: 0,
  ntfy: 0,
}
```

> `4095` 是 `0b1111_1111_1111`，即所有 12 种通知类型全部开启。

#### 判读方法

```typescript
// UserSettings 上的方法
hasNotificationType(key: NotificationAgentKey, type: Notification): boolean {
  return hasNotificationType(type, this.notificationTypes[key] ?? 0);
}

// 底层位运算
export const hasNotificationType = (types, value): boolean => {
  const total = Array.isArray(types) ? types.reduce((a, v) => a + v, 0) : types;
  return !!(value & total);  // 按位与判断是否包含
};
```

### 7. 抄送机制详解

系统中的"抄送"不是显式的 cc 列表，而是通过 **notifyAdmin + notifyUser 两条并行通路** 实现的。

#### 通路 A：管理员广播（notifyAdmin = true）

- **接收群体**：所有拥有 `MANAGE_ISSUES` 权限的用户
- **过滤条件**：
  1. `shouldSendAdminNotification()` — 排除事件触发者本人
  2. `user.settings.hasNotificationType()` — 用户必须在对应 agent 上订阅了该通知类型
- **遍历方式**：每次通知都 `userRepository.find()` 查出所有用户，然后逐个过滤

#### 通路 B：单点通知（notifyUser = 某用户）

- **接收个体**：单个指定用户（通常是 Issue 创建者）
- **过滤条件**：用户在对应 agent 上订阅了该通知类型
- **使用场景**：
  - 普通用户的 Issue 被解决/重开 → 通知创建者
  - 普通用户的 Issue 下有新评论 → 通知创建者

#### 为什么是两套独立通路而不是合并？

因为接收逻辑不同：
- 管理员通路需要**先查权限**（MANAGE_ISSUES），再排除触发者
- 单点通路不需要查权限，**Issue 创建者直接收通知**

两个通路可能会命中同一个人（比如创建者同时也是管理员），但各自独立判断，不会互相去重。

### 8. 各 Agent 的差异

| Agent 类型 | 系统通知依赖 | 管理员通知 | 单点通知 | 备注 |
|-----------|-------------|-----------|----------|------|
| Email | 不检查 notifySystem | ✅ 遍历所有用户过滤 | ✅ notifyUser | 支持 PGP 加密 |
| Discord | 需 `notifySystem && hasNotificationType()` | ✅ 遍历 + @提及 | ✅ notifyUser + @提及 | 支持 roles/users mention |
| WebPush | 不检查 notifySystem | ✅ 遍历订阅设备 | ✅ notifyUser 的设备 | 浏览器推送 |
| Telegram/Slack/Pushover 等 | 各有不同 | ✅  | ✅  | 逻辑结构类似 |

> 注意：`notifySystem` 字段在不同 agent 中解释不同。Discord/Webhook 等 agent 把它当作"全局开关"，而 Email/WebPush 等 agent 不检查此字段，直接按用户偏好发送。

---

## 七、评论 Markdown 渲染管道与 XSS 防护

### 1. 渲染架构概览

```
用户输入（textarea）
      │
      ▼  服务端原样存储（IssueComment.message = 原始文本）
      │
      ▼  前端渲染时通过 ReactMarkdown 组件转 HTML
      │
      ├─ allowedElements 白名单过滤
      ├─ skipHtml 禁止原始 HTML
      └─ React 虚拟 DOM 自动转义（二次保障）
```

### 2. 前端渲染组件

**IssueDescription**（`src/components/IssueDetails/IssueDescription/index.tsx:148-155`）

渲染 Issue 的第一条 Comment（即问题描述）：

```tsx
<div className="prose mt-4">
  <ReactMarkdown
    allowedElements={['p', 'em', 'strong', 'ul', 'ol', 'li']}
    skipHtml
  >
    {description}
  </ReactMarkdown>
</div>
```

**IssueComment**（`src/components/IssueDetails/IssueComment/index.tsx:225-232`）

渲染后续评论：

```tsx
<div className="prose w-full max-w-full">
  <ReactMarkdown
    skipHtml
    allowedElements={['p', 'em', 'strong', 'ul', 'ol', 'li']}
  >
    {comment.message}
  </ReactMarkdown>
</div>
```

### 3. XSS 防护的三层机制

| 层级 | 机制 | 说明 |
|------|------|------|
| **第一层：`skipHtml`** | react-markdown 配置 | 完全禁止 HTML 标签。`<script>alert(1)</script>` 不会被执行，而是被当作纯文本渲染 |
| **第二层：`allowedElements`** | react-markdown 配置 | 白名单只允许 6 种元素：`p`, `em`, `strong`, `ul`, `ol`, `li`。标题（`h1-h6`）、链接（`a`）、图片（`img`）、代码块（`code`/`pre`）等全部被过滤 |
| **第三层：React 自动转义** | React JSX 机制 | 即使前两层被绕过，React 默认对 `{}` 内插值做 HTML 转义，不会执行恶意脚本 |

### 4. 被过滤掉的 Markdown 特性

| 用户输入 | 渲染结果 | 原因 |
|----------|----------|------|
| `# 标题` | 纯文本 `# 标题` | `h1` 不在白名单 |
| `[链接](https://...)` | 纯文本 `链接` | `a` 不在白名单 |
| `![图片](url)` | 纯文本 `![图片](url)` | `img` 不在白名单 |
| `` `代码` `` | 纯文本 `代码` | `code` 不在白名单 |
| `> 引用` | 纯文本 `> 引用` | `blockquote` 不在白名单 |
| `<script>alert(1)</script>` | 纯文本 `alert(1)` | `skipHtml` + 白名单双重过滤 |
| `**粗体**` | **粗体** | `strong` 在白名单 ✅ |
| `*斜体*` | *斜体* | `em` 在白名单 ✅ |
| `- 列表项` | · 列表项 | `ul`/`li` 在白名单 ✅ |

### 5. 列表页的纯文本截断渲染

**IssueItem**（`src/components/IssueList/IssueItem/index.tsx:112-117`）

Issue 列表页中，评论描述**不走 Markdown 渲染**，直接截断纯文本：

```tsx
const description = issue.comments?.[0]?.message || '';
const maxDescriptionLength = 120;
const truncatedDescription = shouldTruncate
  ? description.substring(0, maxDescriptionLength) + '...'
  : description;
```

截断后的文本放在 `<span>` 中，Tooltip 展示全文时也用 `whitespace-pre-wrap` 纯文本样式，**不经过 ReactMarkdown**。

### 6. 服务端无额外过滤

后端 `IssueComment.message` 字段为 `text` 类型，直接存储用户原始输入，不做任何 sanitize 或 HTML 转义。XSS 防护完全依赖前端渲染时 react-markdown 的白名单 + React 的自动转义。

---

## 八、Issue 与 Admin 投诉处理（Blocklist）的协同关系

### 1. 结论：Issue 与 Blocklist 是两套独立系统

经过全代码库搜索，Issue 和 Blocklist 之间**没有任何直接的代码关联**。搜索 `issue.*blocklist` / `blocklist.*issue` 返回 0 命中。

两者共享同一媒体资源（Media 实体），但各自独立运作：

```
                    ┌──────────────┐
                    │  Media 实体   │
                    └──────┬───────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────────┐
    │  Issues  │   │ Requests │   │  Blocklist   │
    │ (投诉报告)│   │ (请求)   │   │ (黑名单屏蔽) │
    └──────────┘   └──────────┘   └──────────────┘
    独立 CRUD      独立 CRUD       独立 CRUD
    独立通知        独立通知         无通知
```

### 2. 权限体系对比

| 权限 | 值 | 说明 |
|------|-----|------|
| `MANAGE_ISSUES` | 1048576 | 管理 Issue（解决/重开/删除任意 Issue） |
| `VIEW_ISSUES` | 2097152 | 查看 Issue 列表和详情 |
| `CREATE_ISSUES` | 4194304 | 创建 Issue 和评论 |
| `MANAGE_BLOCKLIST` | 268435456 | 管理 Blocklist（添加/删除黑名单） |
| `VIEW_BLOCKLIST` | 1073741824 | 查看 Blocklist |

**关键区别**：
- Issue 权限面向**所有用户**开放（CREATE_ISSUES 是独立权限，普通用户可拥有）
- Blocklist 权限面向**管理员**（MANAGE_BLOCKLIST/VIEW_BLOCKLIST 只授予管理员）
- Issue 的 `MANAGE_ISSUES` 和 Blocklist 的 `MANAGE_BLOCKLIST` 是**完全独立的位标志**，一个用户可以只有其一

### 3. 管理员的实际协同工作流

虽然代码层面无关联，但管理员在实际操作中存在**隐性的手动协同**：

```
1. 用户报告 Issue（如"这个视频有严重的版权问题"）
2. 管理员查看 Issue 详情，判断严重程度
3. 管理员可能：
   a. 解决 Issue → POST /issue/:id/resolved
   b. 同时将媒体加入 Blocklist → POST /blocklist（另开页面操作）
4. Blocklist 生效后：
   - Media.status → BLOCKLISTED
   - 该媒体不会再出现在发现页/搜索中
   - 但已存在的 Issue 不会自动解决
```

### 4. Blocklist 的自动化机制

**BlocklistedTagsProcessor**（`server/job/blocklistedTagsProcessor.ts`）

这是一个定时任务（`server/job/schedule.ts:246-260`），自动将包含特定 TMDB 关键词的媒体加入黑名单：

```
定时运行（默认每天）
  → 清除之前按标签加入的 Blocklist 条目
  → 遍历 blocklistedTags 设置中的每个关键词
  → 通过 TMDB Discover API 搜索带该关键词的影视
  → 将结果逐个 addToBlocklist()
  → Media.status = BLOCKLISTED
```

> 注意：此任务**只处理 Blocklist**，不触碰 Issue。即使媒体因标签被加入黑名单，其上的 Open Issue 也不会自动解决。

### 5. Blocklist 与 Media 的级联删除

当从 Blocklist 中移除条目时（`server/routes/blocklist.ts:264-312`）：

```
删除 Blocklist 条目
  → 同时删除对应的 Media 条目（mediaRepository.remove）
  → Media 的 onDelete: 'CASCADE' → 级联删除所有关联的 Issues 和 Comments
```

这意味着**解除 Blocklist 会连带删除该媒体的所有 Issue**。这是一个容易忽略的副作用。

---

## 九、Issue 统计接口与数据归档

### 1. 统计接口：GET /issue/count

**路由位置**：`server/routes/issue.ts:167-227`

**权限**：无（任何已登录用户均可访问，甚至未检查 isAuthenticated）

**返回结构**：

```json
{
  "total": 42,
  "video": 15,
  "audio": 8,
  "subtitles": 12,
  "others": 7,
  "open": 30,
  "closed": 12
}
```

**实现方式**：执行 **7 次独立 COUNT 查询**：

```typescript
const totalCount    = query.getCount();                                    // 全量
const videoCount    = query.where('issue.issueType = :issueType', { issueType: 1 }).getCount();
const audioCount    = query.where('issue.issueType = :issueType', { issueType: 2 }).getCount();
const subtitlesCount = query.where('issue.issueType = :issueType', { issueType: 3 }).getCount();
const othersCount   = query.where('issue.issueType = :issueType', { issueType: 4 }).getCount();
const openCount     = query.where('issue.status = :issueStatus', { issueStatus: 1 }).getCount();
const closedCount   = query.where('issue.status = :issueStatus', { issueStatus: 2 }).getCount();
```

> ⚠️ 性能隐患：7 次独立 SQL 而非 GROUP BY 聚合。当 Issue 数量大时，可优化为 `SELECT issueType, status, COUNT(*) FROM issue GROUP BY issueType, status` 一次查完。

### 2. 统计数据的消费方

| 消费位置 | 使用的字段 | 说明 |
|----------|-----------|------|
| `src/components/Layout/index.tsx:33-34` | `issueResponse?.open` | 全局 SWR 请求 `/api/v1/issue/count` |
| `src/components/Layout/Sidebar/index.tsx:306-316` | `openIssuesCount` | 侧边栏 "Issues" 菜单项旁的红色 Badge |
| `src/components/Layout/MobileMenu/index.tsx` | `openIssuesCount` | 移动端菜单的红色 Badge |
| `src/components/IssueDetails/index.tsx:151` | `mutate('/api/v1/issue/count')` | 状态变更后手动刷新计数 |
| `src/components/IssueModal/CreateIssueModal/index.tsx:135` | `mutate('/api/v1/issue/count')` | 创建 Issue 后手动刷新计数 |

**数据流**：

```
GET /issue/count
    │
    ▼ Layout 组件全局 SWR 缓存
    │
    ├─→ Sidebar.openIssuesCount = response.open
    └─→ MobileMenu.openIssuesCount = response.open
    
    Issue 状态变更 / 创建 → mutate('/api/v1/issue/count') → SWR 重新请求 → Badge 更新
```

### 3. Issue 列表接口：GET /issue

**路由位置**：`server/routes/issue.ts:18-100`

**权限**：`MANAGE_ISSUES` 或 `VIEW_ISSUES` 或 `CREATE_ISSUES`（任一即可）

**查询参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| take | number | 10 | 每页条数 |
| skip | number | 0 | 跳过条数 |
| filter | string | - | `all`/`open`/`resolved` |
| sort | string | - | `added`/`modified` |
| userId | number | - | 按创建者过滤 |

**排序逻辑**：

```typescript
switch (sort) {
  case 'modified':
    query.orderBy('issue.updatedAt', 'DESC');  // 最后修改时间
    break;
  default:
    query.orderBy('issue.createdAt', 'DESC');  // 默认：创建时间倒序
}
```

**过滤逻辑**：

```typescript
switch (filter) {
  case 'open':
    query.andWhere('issue.status = :status', { status: IssueStatus.OPEN });
    break;
  case 'resolved':
    query.andWhere('issue.status = :status', { status: IssueStatus.RESOLVED });
    break;
  // 'all' 不加过滤
}
```

**userId 过滤**：额外条件 `issue.createdBy.id = :userId`，仅管理员可用（普通用户自动追加自己的 id）。

### 4. 数据归档与清理机制

**结论：Issue 没有任何归档或自动清理机制。**

经过全代码库搜索：

| 搜索维度 | 结果 |
|----------|------|
| 关键词 `archive`/`purge`/`retention` 在 server 目录 | 0 命中（与 Issue 相关） |
| 定时任务 `server/job/schedule.ts` | 无 Issue 相关定时任务 |
| `availabilitySync` | 不处理 Issue |
| `MediaSubscriber` | 不处理 Issue |
| 媒体删除时的级联行为 | `onDelete: 'CASCADE'` → Issue 和 Comments 随 Media 一起删除 |

**Issue 数据的"归档"实际上只有两种方式**：

1. **手动解决**：`POST /issue/:id/resolved`，标记为 RESOLVED，数据仍在数据库
2. **手动删除**：`DELETE /issue/:issueId`，硬删除 Issue + Comments
3. **被动级联删除**：Media 被删除时（包括 Blocklist 解除时），Issue 随之删除

**RESOLVED 状态的 Issue 会永远留在数据库中**，没有 TTL、没有归档表、没有软删除标记。唯一清理方式是管理员手动删除或关联 Media 被删。

---

## 十、Issue 自动分类与标签系统

### 1. 结论：Issue 只有固定分类，没有标签系统，也没有自动分类

经过全代码库搜索，Issue 模块的"分类/标签"体系非常简单：

| 功能 | 是否存在 | 实现方式 |
|------|---------|----------|
| Issue 分类（Type） | ✅ 有 | `issueType` 枚举，4 种固定类型 |
| Issue 标签（Tags） | ❌ 无 | Issue 实体无 tags 字段 |
| 自动分类 | ❌ 无 | 没有任何基于内容/关键词的自动归类逻辑 |
| 自定义分类 | ❌ 无 | 用户不能创建自己的 Issue 类别 |

### 2. 现有的分类机制：IssueType 枚举

**定义位置**：`server/constants/issue.ts`

```typescript
enum IssueType {
  VIDEO = 1,     // 视频问题（画面、播放等）
  AUDIO = 2,     // 音频问题
  SUBTITLES = 3, // 字幕问题
  OTHER = 4,     // 其他问题
}
```

**特点**：
- 固定 4 种，不可扩展
- 创建 Issue 时由用户手动选择
- 后端仅存储枚举值，不做任何自动判断
- 列表页可按 `issueType` 过滤，但 API 层面的 count 接口有按类型统计

### 3. 与 MediaRequest 标签系统的对比

作为参照，`MediaRequest`（媒体请求）有完整的标签系统，但 Issue 没有：

| 特性 | MediaRequest | Issue |
|------|-------------|-------|
| tags 字段 | ✅ `string[]` 数组 | ❌ 无 |
| 自定义标签 | ✅ 用户可添加任意标签 | ❌ 无 |
| 按标签过滤 | ✅ 支持 | ❌ 不支持 |
| 自动打标 | ✅ 从 Radarr/Sonarr 同步 tag | ❌ 无 |

**为什么 Issue 没有标签？** 从代码来看，Issue 设计上是一个**轻量级的问题反馈系统**，定位是简单的"报 bug/提问题"，而不是工单系统（ticket system）。没有优先级、没有标签、没有指派、没有自动分类。

### 4. 前端分类选择 UI

**CreateIssueModal**（`src/components/IssueModal/CreateIssueModal/index.tsx`）

创建 Issue 时通过 `Selector` 组件选择 issueType，4 个选项对应 4 个枚举值，纯手动选择，无智能推荐。

---

## 十一、Issue 反垃圾与防刷机制

### 1. 结论：Issue 模块没有专门的反垃圾过滤

经过全代码库搜索，Issue 相关接口**没有任何反垃圾/防刷保护**：

| 反垃圾手段 | 是否存在 | 说明 |
|-----------|---------|------|
| 速率限制（Rate Limit） | ❌ 无 | Issue 路由没有 rate-limit 中间件 |
| 验证码（Captcha） | ❌ 无 | 全站无 Captcha |
| 内容过滤（关键词/敏感词） | ❌ 无 | 后端不校验评论内容 |
| Akismet 等反垃圾服务 | ❌ 无 | 无集成 |
| 用户信任度/等级 | ❌ 无 | Plex/Jellyfin 用户一律平等 |
| Flood 检测（短时间大量发帖） | ❌ 无 | 无时间窗口内的计数限制 |
| 媒体级别频率限制 | ❌ 无 | 同一媒体可以被无限次报 Issue |

### 2. 全站仅有的速率限制

全项目唯一一处 `express-rate-limit` 在 Plex 用户导入接口：

**位置**：`server/routes/settings/index.ts:540`

```typescript
settingsRoutes.get(
  '/plex/users',
  isAuthenticated(Permission.ADMIN),
  rateLimit({ windowMs: 60 * 1000, max: 50 }), // 每分钟 50 次
  // ...
);
```

这个限制只针对**管理员调用 Plex 用户列表**的接口，和 Issue 完全无关。

### 3. 间接的"门槛"保护

Issue 虽然没有显式反垃圾，但有一些隐式门槛：

| 门槛 | 说明 |
|------|------|
| 必须登录 | `router.use('/issue', isAuthenticated(), issueRoutes)`，所有 Issue 操作都需要登录 |
| CREATE_ISSUES 权限 | 不是所有用户都能创建 Issue，需要管理员授予权限 |
| 媒体必须存在 | 创建 Issue 时验证 mediaId，不能凭空发垃圾 |
| Plex/Jellyfin 账号体系 | 用户都来自真实的媒体服务器账号，不是随便注册的 |

### 4. 与反垃圾最接近的代码：Blocklist

`Blocklist`（黑名单）系统是最接近"反垃圾"的机制，但它针对的是**媒体内容**（按 TMDB ID 或关键词屏蔽整部影视），不是针对 Issue 评论内容或用户行为。

**路径**：用户报 Issue → 管理员判断严重 → 手动将媒体加入 Blocklist

这是**事后人工处理**，不是事前自动过滤。

---

## 十二、Issue 数据备份与导出 API

### 1. 结论：没有 Issue 专属的导出/备份接口

经过全代码库搜索，Issue 模块**没有任何导出、备份、批量下载功能**：

| 导出方式 | 是否存在 | 说明 |
|----------|---------|------|
| Issue CSV 导出 | ❌ 无 | 无 CSV 导出接口 |
| Issue JSON 导出 | ❌ 无 | 无 JSON 批量导出 |
| Issue 数据备份 | ❌ 无 | 没有备份 API |
| Issue 数据导入 | ❌ 无 | 没有导入 API |
| 数据库整体备份 | ❌ 无 | 应用内无备份按钮 |

### 2. 全局数据迁移：Overseerr Merge

唯一和"数据迁移"沾边的是 `overseerrMerge.ts`，但它**不是用户可主动调用的导出功能**：

**位置**：`server/lib/overseerrMerge.ts`

**作用**：首次启动时，检测到用户是从 Overseerr 迁移过来的（数据库已存在但 Jellyseerr 配置为空），自动合并数据库结构并修复数据。

**触发条件**：
```typescript
const checkOverseerrMerge = async (): Promise<boolean> => {
  const settings = await new Settings().load(undefined, true);
  if (settings.main.mediaServerType) {
    return false; // 已配置过就不运行
  }
  // ... 迁移逻辑
};
```

**和 Issue 的关系**：迁移会保留 Issue 表结构（通过 TypeORM migration 兼容处理），但没有 Issue 专属的导入逻辑。Issue 数据跟着数据库文件走，SQLite 就是整个库一起迁移。

### 3. About 接口的统计数据（不是导出）

`GET /settings/about` 接口返回一些统计数字，但**不含 Issue 数量**：

**位置**：`server/routes/settings/index.ts:822-836`

```typescript
settingsRoutes.get('/about', async (req, res) => {
  const totalMediaItems = await mediaRepository.count();
  const totalRequests = await mediaRequestRepository.count();

  return res.status(200).json({
    version: getAppVersion(),
    totalMediaItems,  // 媒体总数
    totalRequests,    // 请求总数
    tz: process.env.TZ,
    appDataPath: appDataPath(),
  });
});
```

注意：只有 `totalMediaItems` 和 `totalRequests`，**没有 `totalIssues`**。Issue 统计走的是独立的 `/issue/count` 接口（已在第九节分析）。

### 4. 实际备份方式：文件级备份

由于 Jellyseerr 使用 SQLite（默认）或 PostgreSQL，数据备份的实际方式是**文件级或数据库级备份**，不是应用内导出：

| 部署方式 | 备份方法 |
|---------|---------|
| Docker SQLite | 备份 `config/db.sqlite` 文件 |
| 原生 SQLite | 备份整个 `.sqlite` 文件 |
| PostgreSQL | `pg_dump` 导出整库 |

应用代码中**没有任何触发备份的逻辑**，完全靠运维层面处理。

### 5. 相关的数据持久化机制

| 机制 | 文件 | 说明 |
|------|------|------|
| 设置存储 | `settings.json` | 存在 `appDataPath` 目录，JSON 格式 |
| 数据库 | `db.sqlite` | SQLite 数据库文件 |
| 缓存 | `cache/` 目录 | 图片缓存等 |

Issue 数据全部存在数据库表 `issue` 和 `issue_comment` 中，没有单独的导出文件。

---

## 十八、核心代码文件索引

| 文件 | 职责 |
|------|------|
| `server/constants/issue.ts` | IssueType、IssueStatus 枚举定义 |
| `server/entity/Issue.ts` | Issue 实体 + OneToMany 关系 + AfterLoad 排序 |
| `server/entity/IssueComment.ts` | IssueComment 实体 |
| `server/entity/Media.ts` | Media 实体 + getMedia/getRelatedMedia 静态方法 + issues 反查 |
| `server/entity/Blocklist.ts` | Blocklist 实体 + addToBlocklist 静态方法 |
| `server/entity/UserSettings.ts` | 用户通知偏好存储 + hasNotificationType 方法 |
| `server/interfaces/api/issueInterfaces.ts` | 前后端 API 接口类型 |
| `server/routes/issue.ts` | Issue CRUD + 评论 + 状态变更 + count 统计 |
| `server/routes/issueComment.ts` | Comment 独立 CRUD |
| `server/routes/blocklist.ts` | Blocklist CRUD + 集合黑名单 |
| `server/routes/movie.ts` | 电影详情页，调用 Media.getMedia() 反查 issues |
| `server/routes/tv.ts` | 剧集详情页，调用 Media.getMedia() 反查 issues |
| `server/subscriber/IssueSubscriber.ts` | Issue 创建/状态变更 → 触发通知 |
| `server/subscriber/IssueCommentSubscriber.ts` | Comment 创建 → 触发通知（排除第1条） |
| `server/subscriber/MediaSubscriber.ts` | 媒体状态变更 → 只更新 MediaRequest，**不处理 Issue** |
| `server/lib/permissions.ts` | 权限枚举（含 MANAGE_ISSUES/VIEW_ISSUES/CREATE_ISSUES/MANAGE_BLOCKLIST） |
| `server/lib/notifications/index.ts` | 通知类型枚举 + 权限过滤 + Manager |
| `server/lib/notifications/agents/agent.ts` | 通知代理接口 + Payload 结构 |
| `server/lib/notifications/agents/email.ts` | Email 通知代理，含完整管理员/用户双通路 |
| `server/lib/notifications/agents/discord.ts` | Discord 通知代理，含 @提及逻辑 |
| `server/lib/notifications/agents/webpush.ts` | WebPush 浏览器推送代理 |
| `server/lib/availabilitySync.ts` | 媒体可用性同步，**不处理 Issue 自动关闭** |
| `server/job/schedule.ts` | 定时任务调度器，**无 Issue 相关任务** |
| `server/job/blocklistedTagsProcessor.ts` | 关键词黑名单自动扫描，**只处理 Blocklist** |
| `src/components/IssueDetails/IssueDescription/index.tsx` | Issue 描述的 Markdown 渲染（allowedElements + skipHtml） |
| `src/components/IssueDetails/IssueComment/index.tsx` | Issue 评论的 Markdown 渲染（allowedElements + skipHtml） |
| `src/components/IssueDetails/index.tsx` | Issue 详情页主组件（SWR 数据流 + 操作入口） |
| `src/components/IssueList/index.tsx` | Issue 列表页（过滤/排序/分页） |
| `src/components/IssueList/IssueItem/index.tsx` | Issue 列表项（纯文本截断描述，无 Markdown） |
| `src/components/IssueModal/CreateIssueModal/index.tsx` | 创建 Issue 弹窗 |
| `src/components/Layout/index.tsx` | 全局 SWR 请求 /issue/count |
| `src/components/Layout/Sidebar/index.tsx` | 侧边栏 Issue 计数 Badge |
| `server/lib/overseerrMerge.ts` | Overseerr 数据库合并迁移（首次启动时运行，非用户导出工具） |
| `server/interfaces/api/settingsInterfaces.ts` | SettingsAboutResponse 定义（不含 totalIssues） |
| `server/routes/settings/index.ts` | 全站唯一的 rate-limit 应用处（Plex 用户导入） |
| `server/routes/search.ts` | TMDB 搜索路由（不搜索 Issue 数据） |
| `server/middleware/auth.ts` | 认证中间件（checkUser + isAuthenticated） |
| `server/lib/notifications/agents/webhook.ts` | Webhook Agent，含 Issue 字段 KeyMap 模板变量映射 |

---

## 十九、容易混淆的点总结

1. **Issue 没有 message 字段**：描述全在 comments[0]，创建 Issue 时的 message 直接变成第一条 Comment
2. **第一条 Comment 不触发 ISSUE_COMMENT**：IssueCommentSubscriber 中有判断跳过，避免和 ISSUE_CREATED 重复
3. **状态变更才发通知**：IssueSubscriber.beforeUpdate 严格对比 `event.entity.status` vs `event.databaseEntity.status`，普通字段更新不触发
4. **通知三重去重**：① 排除触发者本人（shouldSendAdminNotification）② 管理员和创建者重复时 ③ 第1条 Comment 跳过 ISSUE_COMMENT
5. **Comment 编辑不可越权**：管理员也不能改别人的评论内容，只能删
6. **用户删除 Issue 的评论数限制**：comments.length > 1（即有别人参与过）时普通用户不能删
7. **Issue ↔ Media 双向加载不对称**：Issue 侧 eager 自动带 Media，但 Media 侧默认不带 issues，需要显式 `relations: { issues: true }`
8. **没有自动关闭 Issue 的机制**：Media 状态变 AVAILABLE 时只更新 MediaRequest，Issue 必须手动解决
9. **抄送是两条独立通路**：notifyAdmin（管理员组）和 notifyUser（单点用户）各自独立判断，可能重复命中同一人
10. **notifySystem 含义不一致**：Discord/Webhook 当全局开关用，Email/WebPush 直接忽略此字段按用户偏好发
11. **用户通知偏好是位标志**：每个 agent 一个数字，通过 `&` 运算判断是否订阅某类通知
12. **Manager 不决定接收者**：NotificationManager 只分发给 agent，每个 agent 自己查用户表、自己过滤接收人
13. **Markdown 白名单极严格**：只允许 p/em/strong/ul/ol/li 六种元素，链接、图片、代码块、标题全部被过滤
14. **XSS 防护全在前端**：后端原样存储，不 sanitize；前端靠 react-markdown 的 skipHtml + allowedElements + React 自动转义三层防护
15. **列表页不走 Markdown**：IssueItem 组件直接 substring 截断纯文本，Tooltip 也用 whitespace-pre-wrap 而非 ReactMarkdown
16. **Issue 与 Blocklist 完全独立**：代码中零关联，管理员需手动在两个页面分别操作
17. **解除 Blocklist 会级联删除 Issue**：删除 Blocklist 条目 → 删除 Media → onDelete:CASCADE → Issues 全没了
18. **count 接口无权限检查**：`GET /issue/count` 未调用 isAuthenticated，且执行 7 次独立 COUNT 查询（可优化为 GROUP BY）
19. **Issue 没有归档/清理机制**：RESOLVED 的 Issue 永远留在数据库，无 TTL、无归档表、无定时清理任务
20. **BlocklistedTagsProcessor 不影响 Issue**：关键词黑名单自动扫描只处理 Blocklist + Media 状态，不碰 Issue
21. **Issue 没有标签系统**：只有 issueType 枚举（4 种固定类型），没有 tags 字段，也不能自定义分类
22. **MediaRequest 有标签但 Issue 没有**：两套系统设计定位不同，Issue 是轻量反馈，MediaRequest 是工单式管理
23. **Issue 没有专门的反垃圾保护**：没有 rate limit、没有 captcha、没有内容过滤，全靠登录门槛和权限控制
24. **全站唯一的 rate limit 在 Plex 用户导入**：和 Issue 完全无关，是 settings 里 `/plex/users` 接口每分钟 50 次
25. **Issue 没有导出/备份接口**：没有 CSV/JSON 导出，没有应用内备份按钮，备份靠文件级（SQLite 文件或 pg_dump）
26. **About 接口不含 Issue 统计**：`/settings/about` 只有 totalMediaItems 和 totalRequests，Issue 统计走独立的 `/issue/count`
27. **Overseerr Merge 不是导出工具**：只在首次启动时自动运行一次，用于兼容 Overseerr 数据库，不是用户可调用的导入/导出功能

---

## 十五、Issue 跨语言搜索

### 1. 结论：Issue 不参与搜索，也没有跨语言搜索能力

经过全代码库搜索，Issue 模块**不存在搜索功能**，更不存在跨语言搜索。

### 2. 全局搜索路由与 Issue 的关系

**搜索路由**：`server/routes/search.ts`

```
GET /search?query=xxx&language=zh
GET /search/keyword?query=xxx
GET /search/company?query=xxx
```

这些搜索全部调用 **TMDB API**（The Movie Database），搜索对象是影视、关键词、公司，**不搜索本地 Issue 数据**。

```
用户输入搜索词
    │
    ▼
searchProvider = findSearchProvider(queryString)  // 匹配 IMDB ID / TMDB ID 等特殊格式
    │
    ├─ 匹配到特殊 provider → 调用 provider.search()
    └─ 未匹配 → tmdb.searchMulti({ query, language })  // TMDB 多语言搜索
    │
    ▼
返回影视结果 + 关联的本地 Media 数据
```

`language` 参数传入 TMDB API，用于返回对应语言的影视标题/简介。这和 Issue 没有任何关系。

### 3. Issue 列表的"过滤"不是"搜索"

Issue 列表页（`GET /issue`）的查询参数只有过滤/排序，没有全文搜索：

| 参数 | 作用 | 是否搜索 |
|------|------|---------|
| filter | open / resolved / all | 过滤，不是搜索 |
| sort | added / modified | 排序 |
| userId | 按创建者 | 过滤 |
| take / skip | 分页 | 分页 |

后端查询是纯 SQL `WHERE` 条件，没有 `LIKE`、没有 `FULLTEXT`、没有 `ILIKE`。

### 4. 前端 Issue 页面的搜索

`src/pages/issues/index.tsx` 中没有搜索框，只有过滤器（状态、类型、排序）。用户无法按评论内容或标题搜索 Issue。

### 5. 为什么没有 Issue 搜索？

Issue 的数据量设计上很小（每个媒体一般只有 0-2 个 Issue），SQL `WHERE status = OPEN` 就够了。没有引入 Elasticsearch / Meilisearch 等搜索引擎的必要。如果需要跨语言搜索，需要在 Issue 数据上建全文索引，目前代码中完全缺失这个方向。

---

## 十六、Issue 与监控告警系统集成

### 1. 结论：没有独立监控告警系统，Webhook 是唯一的集成出口

Jellyseerr **没有内置 Prometheus / Grafana / Zabbix 等监控集成**。但通知系统中的 **Webhook Agent** 可以作为对外集成的桥梁，Issue 事件通过它推送到外部系统。

### 2. Webhook Agent 的 Issue 数据映射

**位置**：`server/lib/notifications/agents/webhook.ts:17-64`

Webhook Agent 定义了 `KeyMap`，将 Issue 相关字段映射为模板变量：

| 模板变量 | 映射值 | 说明 |
|---------|--------|------|
| `{{issue_id}}` | `payload.issue.id` | Issue ID |
| `{{issue_type}}` | `IssueType[issue.issueType]` | 问题类型名（VIDEO/AUDIO/SUBTITLES/OTHER） |
| `{{issue_status}}` | `IssueStatus[issue.status]` | 状态名（OPEN/RESOLVED） |
| `{{reportedBy_username}}` | `issue.createdBy.displayName` | 报告人用户名 |
| `{{reportedBy_email}}` | `issue.createdBy.email` | 报告人邮箱 |
| `{{reportedBy_avatar}}` | `issue.createdBy.avatar` | 报告人头像 |
| `{{reportedBy_settings_discordIds}}` | `issue.createdBy.settings.discordIds` | 报告人 Discord ID |
| `{{reportedBy_settings_telegramChatId}}` | `issue.createdBy.settings.telegramChatId` | 报告人 Telegram Chat ID |
| `{{comment_message}}` | `comment.message` | 评论内容 |
| `{{commentedBy_username}}` | `comment.user.displayName` | 评论人用户名 |
| `{{commentedBy_email}}` | `comment.user.email` | 评论人邮箱 |
| `{{commentedBy_avatar}}` | `comment.user.avatar` | 评论人头像 |
| `{{notification_type}}` | `Notification[type]` | 通知类型名（ISSUE_CREATED/ISSUE_COMMENT 等） |
| `{{event}}` | `payload.event` | 事件描述 |
| `{{subject}}` | `payload.subject` | 主题（媒体名） |
| `{{message}}` | `payload.message` | 消息内容 |

### 3. Webhook 集成流程

```
Issue 事件触发
    │
    ▼
Subscriber 构造 NotificationPayload
    │
    ▼
NotificationManager 分发给 WebhookAgent
    │
    ▼
WebhookAgent.send():
    1. 检查 notifySystem && hasNotificationType
    2. 从 settings 读取 webhookUrl + jsonPayload（Base64 编码的 JSON 模板）
    3. 解码 jsonPayload → JSON.parse
    4. parseKeys() 递归替换模板变量 {{xxx}} → 实际值
    5. 支持 URL 变量替换（webhookUrl 中的 {{xxx}} 也会被替换）
    6. axios.post(webhookUrl, parsedPayload, { headers })
```

### 4. 外部监控系统集成方案

虽然 Jellyseerr 没有原生监控集成，但通过 Webhook 可以桥接：

```
Jellyseerr Issue Event
    │
    ▼ Webhook POST
    │
    ├─→ n8n / Zapier（自动化工作流）→ 邮件/Slack/JIRA
    ├─→ 自建中间件 → Prometheus Pushgateway → Grafana
    ├─→ Alertmanager Webhook → 告警升级
    └─→ 任意 HTTP 端点
```

### 5. 其他 Agent 的 Issue 告警能力

| Agent | Issue 事件是否支持 | 说明 |
|-------|-------------------|------|
| Webhook | ✅ 完整支持 | 可自定义 JSON 模板，灵活度最高 |
| Discord | ✅ 内嵌 Embed | 含 Issue 类型、状态、报告人、链接 |
| Email | ✅ 模板邮件 | 含 Issue 详情和链接 |
| Slack | ✅ Block Kit | 含 Issue 字段 |
| Telegram | ✅ 消息 | 含 Issue 文本 |
| Pushover | ✅ 推送 | 简洁文本 |
| WebPush | ✅ 浏览器通知 | 简洁文本 |
| Gotify/Ntfy/Pushbullet | ✅ 推送 | 简洁文本 |

所有 10 个通知 Agent 都能接收 Issue 事件，但只有 Webhook 支持自定义 JSON 结构，适合对接监控系统。其余 Agent 是面向人阅读的通知。

### 6. `/status` 端点（非 Issue 专用）

**位置**：`server/routes/index.ts:50-96`

```
GET /api/v1/status → { version, commitTag, updateAvailable, commitsBehind, restartRequired }
```

这个端点返回的是应用版本和更新状态，**不包含 Issue 统计**。没有健康检查端点、没有 `/metrics`、没有 Prometheus 格式输出。

---

## 十七、用户匿名提交 Issue 的处理路径

### 1. 结论：Jellyseerr 不支持匿名提交 Issue

经过全代码库搜索，Issue 创建**强制要求已认证用户**，不存在匿名提交路径。

### 2. 认证中间件链

**路由注册**：`server/routes/index.ts:172`

```typescript
router.use('/issue', isAuthenticated(), issueRoutes);
```

`isAuthenticated()` 中间件（`server/middleware/auth.ts:43-58`）的逻辑：

```typescript
if (!req.user || !req.user.hasPermission(permissions ?? 0)) {
  res.status(403).json({
    status: 403,
    error: 'You do not have permission to access this endpoint',
  });
}
```

**没有 user → 直接 403**，没有 fallback，没有匿名角色。

### 3. 用户身份的来源

`checkUser` 中间件（`server/middleware/auth.ts:9-41`）在 `isAuthenticated` 之前执行，确定用户身份：

```
请求进入
    │
    ▼ checkUser()
    │
    ├─ X-API-Key header 匹配 → 使用 API Key 认证
    │   ├─ 无 X-API-User header → 默认 userId = 1（管理员）
    │   └─ 有 X-API-User header → 使用指定用户
    │
    ├─ session.userId 存在 → 使用 Session 认证（浏览器登录）
    │
    └─ 都没有 → req.user = undefined → 后续 isAuthenticated() 返回 403
```

### 4. Issue 创建时的用户关联

**创建路由**：`server/routes/issue.ts:102-165`

```typescript
// 必须有用户
if (!req.user) {
  return next({ status: 500, message: 'User missing from request.' });
}

// createdBy 的确定逻辑
let createdBy = req.user; // 默认：当前登录用户
if (req.body.userId && req.body.userId !== req.user.id) {
  // 管理员代为创建：需要 MANAGE_ISSUES 权限
  if (!req.user.hasPermission(Permission.MANAGE_ISSUES)) {
    return next({ status: 403, message: '...' });
  }
  createdBy = await userRepository.findOneOrFail({ where: { id: req.body.userId } });
}
```

**所有 Issue 都必须有 createdBy（User 实体）**，没有 user 就无法创建 Issue。

### 5. API Key 认证路径的"伪匿名"

唯一接近"匿名"的场景是 API Key 认证：

```
X-API-Key: <server-api-key>
X-API-User: <target-user-id>  (可选)
```

- 有 `X-API-Key` 但没有 `X-API-User` → 以 userId=1（管理员）身份创建 Issue
- 有 `X-API-Key` + `X-API-User` → 以指定用户身份创建 Issue

这不是匿名，而是**服务端认证代持**。Issue 仍然绑定到一个真实用户。

### 6. 前端也没有匿名入口

所有 Issue 相关前端组件（CreateIssueModal、IssueList 等）都在受保护路由下。未登录用户看不到 Issue 入口，也无法访问 `/api/v1/issue` 接口。

### 7. 对比：哪些端点不需要认证？

```
无需认证的端点                    需要认证的端点
─────────────────               ─────────────────
GET /status                      GET/POST /issue（所有）
GET /status/appdata              GET/POST /issueComment（所有）
GET /settings/public             GET /search
POST /auth/plex                  GET /request（所有）
POST /auth/jellyfin              GET /media（所有）
POST /auth/local                 ... 其他所有接口
GET / (API info)
```

Issue 相关的所有接口都在认证墙后面。

28. **Issue 没有搜索功能**：全局搜索只查 TMDB（影视/关键词/公司），不搜索本地 Issue 数据；Issue 列表只有过滤（状态/排序/用户），没有全文搜索
29. **Issue 没有跨语言搜索**：连搜索本身都不存在，更谈不上跨语言；`language` 参数只传给 TMDB API，和 Issue 无关
30. **监控集成的唯一出口是 Webhook**：没有 Prometheus/Grafana/Zabbix 原生集成，但 Webhook Agent 支持自定义 JSON 模板 + URL 变量替换，可桥接到任意外部系统
31. **Webhook 的 Issue 字段映射很完整**：`KeyMap` 有 16 个 Issue/Comment 相关模板变量，包括报告人信息、评论人信息、Issue 类型/状态等
32. **`/status` 端点不包含 Issue 数据**：只有版本号和更新状态，没有健康检查、没有 `/metrics`、没有 Prometheus 格式输出
33. **不支持匿名提交 Issue**：`isAuthenticated()` 中间件强制拦截无用户请求返回 403；API Key 认证是"服务端认证代持"，Issue 仍绑定真实用户
34. **认证链路：checkUser → isAuthenticated**：checkUser 先从 API Key 或 Session 提取 user，isAuthenticated 再验证权限；缺少任一环节都无法创建 Issue
