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

## 七、核心代码文件索引

| 文件 | 职责 |
|------|------|
| `server/constants/issue.ts` | IssueType、IssueStatus 枚举定义 |
| `server/entity/Issue.ts` | Issue 实体 + OneToMany 关系 + AfterLoad 排序 |
| `server/entity/IssueComment.ts` | IssueComment 实体 |
| `server/entity/Media.ts` | Media 实体 + getMedia/getRelatedMedia 静态方法 + issues 反查 |
| `server/entity/UserSettings.ts` | 用户通知偏好存储 + hasNotificationType 方法 |
| `server/interfaces/api/issueInterfaces.ts` | 前后端 API 接口类型 |
| `server/routes/issue.ts` | Issue CRUD + 评论 + 状态变更 |
| `server/routes/issueComment.ts` | Comment 独立 CRUD |
| `server/routes/movie.ts` | 电影详情页，调用 Media.getMedia() 反查 issues |
| `server/routes/tv.ts` | 剧集详情页，调用 Media.getMedia() 反查 issues |
| `server/subscriber/IssueSubscriber.ts` | Issue 创建/状态变更 → 触发通知 |
| `server/subscriber/IssueCommentSubscriber.ts` | Comment 创建 → 触发通知（排除第1条） |
| `server/subscriber/MediaSubscriber.ts` | 媒体状态变更 → 只更新 MediaRequest，**不处理 Issue** |
| `server/lib/notifications/index.ts` | 通知类型枚举 + 权限过滤 + Manager |
| `server/lib/notifications/agents/agent.ts` | 通知代理接口 + Payload 结构 |
| `server/lib/notifications/agents/email.ts` | Email 通知代理，含完整管理员/用户双通路 |
| `server/lib/notifications/agents/discord.ts` | Discord 通知代理，含 @提及逻辑 |
| `server/lib/notifications/agents/webpush.ts` | WebPush 浏览器推送代理 |
| `server/lib/availabilitySync.ts` | 媒体可用性同步，**不处理 Issue 自动关闭** |

---

## 八、容易混淆的点总结

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
