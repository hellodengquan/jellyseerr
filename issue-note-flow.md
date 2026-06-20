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

Media (1) ──── issues ────< (N) Issue
```

---

## 二、Issue 创建流程详解

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

## 三、备注（Comment）流转全路径

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

## 四、状态流转（Issue 生命周期）

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

---

## 五、通知系统：三者的交汇点

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

---

## 六、核心代码文件索引

| 文件 | 职责 |
|------|------|
| `server/constants/issue.ts` | IssueType、IssueStatus 枚举定义 |
| `server/entity/Issue.ts` | Issue 实体 + OneToMany 关系 + AfterLoad 排序 |
| `server/entity/IssueComment.ts` | IssueComment 实体 |
| `server/interfaces/api/issueInterfaces.ts` | 前后端 API 接口类型 |
| `server/routes/issue.ts` | Issue CRUD + 评论 + 状态变更 |
| `server/routes/issueComment.ts` | Comment 独立 CRUD |
| `server/subscriber/IssueSubscriber.ts` | Issue 创建/状态变更 → 触发通知 |
| `server/subscriber/IssueCommentSubscriber.ts` | Comment 创建 → 触发通知（排除第1条） |
| `server/lib/notifications/index.ts` | 通知类型枚举 + 权限过滤 + Manager |
| `server/lib/notifications/agents/agent.ts` | 通知代理接口 + Payload 结构 |

---

## 七、容易混淆的点总结

1. **Issue 没有 message 字段**：描述全在 comments[0]，创建 Issue 时的 message 直接变成第一条 Comment
2. **第一条 Comment 不触发 ISSUE_COMMENT**：IssueCommentSubscriber 中有判断跳过，避免和 ISSUE_CREATED 重复
3. **状态变更才发通知**：IssueSubscriber.beforeUpdate 严格对比 `event.entity.status` vs `event.databaseEntity.status`，普通字段更新不触发
4. **通知三重去重**：① 排除触发者本人（shouldSendAdminNotification）② 管理员和创建者重复时 ③ 第1条 Comment 跳过 ISSUE_COMMENT
5. **Comment 编辑不可越权**：管理员也不能改别人的评论内容，只能删
6. **用户删除 Issue 的评论数限制**：comments.length > 1（即有别人参与过）时普通用户不能删
