# 通知发送框架实现分析

## 一、整体架构

通知框架采用 **Manager + Agent** 的设计模式，通过统一的入口管理多种推送渠道，每种渠道以独立的 Agent 实现，支持按用户偏好选择 provider。

```
NotificationManager (单例)
       │
       ├── 注册: registerAgents()
       └── 分发: sendNotification()
              │
              ├── DiscordAgent
              ├── EmailAgent
              ├── GotifyAgent
              ├── NtfyAgent
              ├── PushbulletAgent
              ├── PushoverAgent
              ├── SlackAgent
              ├── TelegramAgent
              ├── WebhookAgent
              └── WebPushAgent
```

**核心文件位置：**
- 管理器: `server/lib/notifications/index.ts`
- 代理基类: `server/lib/notifications/agents/agent.ts`
- 各渠道实现: `server/lib/notifications/agents/*.ts`
- 系统设置: `server/lib/settings/index.ts`
- 用户设置: `server/entity/UserSettings.ts`

---

## 二、通知类型（Bitmask 模式）

通知类型使用 **位掩码（Bitmask）** 设计，每个类型对应一个二进制位，支持灵活组合。

### 2.1 通知类型枚举

定义在 `server/lib/notifications/index.ts:6-20`：

```typescript
export enum Notification {
  NONE = 0,
  MEDIA_PENDING = 2,        // 媒体待审核
  MEDIA_APPROVED = 4,       // 媒体已批准
  MEDIA_AVAILABLE = 8,      // 媒体已可用
  MEDIA_FAILED = 16,        // 媒体处理失败
  TEST_NOTIFICATION = 32,   // 测试通知
  MEDIA_DECLINED = 64,      // 媒体被拒绝
  MEDIA_AUTO_APPROVED = 128,// 媒体自动批准
  ISSUE_CREATED = 256,      // 问题创建
  ISSUE_COMMENT = 512,      // 问题评论
  ISSUE_RESOLVED = 1024,    // 问题解决
  ISSUE_REOPENED = 2048,    // 问题重开
  MEDIA_AUTO_REQUESTED = 4096, // 媒体自动请求
}
```

### 2.2 类型检查函数

`hasNotificationType()` 函数用于检查某个通知类型是否被启用：

```typescript
export const hasNotificationType = (
  types: Notification | Notification[],
  value: number
): boolean => {
  let total: number;
  if (Array.isArray(types)) {
    total = types.reduce((a, v) => a + v, 0);
  } else {
    total = types;
  }
  // 测试通知始终允许
  if (!(value & Notification.TEST_NOTIFICATION)) {
    value += Notification.TEST_NOTIFICATION;
  }
  return !!(value & total);
};
```

**设计要点：**
- 使用位运算 `&` 进行高效判断
- `TEST_NOTIFICATION` 类型特殊处理，无需显式启用
- 支持传入数组，自动合并多个类型

---

## 三、Agent 框架设计

### 3.1 核心接口

定义在 `server/lib/notifications/agents/agent.ts`：

```typescript
export interface NotificationPayload {
  event?: string;
  subject: string;
  notifySystem: boolean;
  notifyAdmin: boolean;
  notifyUser?: User;
  media?: Media;
  image?: string;
  message?: string;
  extra?: { name: string; value: string }[];
  request?: MediaRequest;
  issue?: Issue;
  comment?: IssueComment;
  pendingRequestsCount?: number;
  isAdmin?: boolean;
}

export interface NotificationAgent {
  shouldSend(): boolean;
  send(type: Notification, payload: NotificationPayload): Promise<boolean>;
}

export abstract class BaseAgent<T extends NotificationAgentConfig> {
  protected settings?: T;
  public constructor(settings?: T) {
    this.settings = settings;
  }
  protected abstract getSettings(): T;
}
```

### 3.2 Agent 生命周期

每个 Agent 遵循统一的生命周期：

1. **实例化**：服务启动时创建 Agent 实例
2. **注册**：通过 `notificationManager.registerAgents()` 注册
3. **判断是否发送**：调用 `shouldSend()` 检查是否启用
4. **发送通知**：调用 `send()` 执行实际发送逻辑

---

## 四、NotificationManager 管理器

### 4.1 单例模式

`NotificationManager` 是单例，全局唯一入口：

```typescript
class NotificationManager {
  private activeAgents: NotificationAgent[] = [];

  public registerAgents = (agents: NotificationAgent[]): void => {
    this.activeAgents = [...this.activeAgents, ...agents];
    logger.info('Registered notification agents', { label: 'Notifications' });
  };

  public sendNotification(
    type: Notification,
    payload: NotificationPayload
  ): void {
    logger.info(`Sending notification(s) for ${Notification[type]}`, {
      label: 'Notifications',
      subject: payload.subject,
    });

    this.activeAgents.forEach((agent) => {
      if (agent.shouldSend()) {
        agent.send(type, payload);
      }
    });
  }
}
```

### 4.2 Agent 注册时机

在 `server/index.ts:131-143` 服务启动时注册所有 Agent：

```typescript
notificationManager.registerAgents([
  new DiscordAgent(),
  new EmailAgent(),
  new GotifyAgent(),
  new NtfyAgent(),
  new PushbulletAgent(),
  new PushoverAgent(),
  new SlackAgent(),
  new TelegramAgent(),
  new WebhookAgent(),
  new WebPushAgent(),
]);
```

---

## 五、系统级配置

### 5.1 配置结构

系统级通知配置存储在 `settings.json` 中，结构定义在 `server/lib/settings/index.ts`：

```typescript
export enum NotificationAgentKey {
  DISCORD = 'discord',
  EMAIL = 'email',
  GOTIFY = 'gotify',
  NTFY = 'ntfy',
  PUSHBULLET = 'pushbullet',
  PUSHOVER = 'pushover',
  SLACK = 'slack',
  TELEGRAM = 'telegram',
  WEBHOOK = 'webhook',
  WEBPUSH = 'webpush',
}

interface NotificationAgents {
  discord: NotificationAgentDiscord;
  email: NotificationAgentEmail;
  gotify: NotificationAgentGotify;
  // ... 其他 agent
}

interface NotificationSettings {
  agents: NotificationAgents;
}
```

### 5.2 通用配置接口

每个 Agent 都继承 `NotificationAgentConfig` 基础配置：

```typescript
export interface NotificationAgentConfig {
  enabled: boolean;      // 是否启用
  embedPoster: boolean;  // 是否嵌入海报图
  types?: number;        // 启用的通知类型（bitmask）
  options: Record<string, unknown>; // 渠道特定选项
}
```

---

## 六、用户偏好系统

### 6.1 用户设置实体

用户级通知偏好存储在 `UserSettings` 实体中，定义在 `server/entity/UserSettings.ts`：

```typescript
@Entity()
export class UserSettings {
  @Column({
    type: 'text',
    nullable: true,
    transformer: { /* JSON 序列化/反序列化 */ },
  })
  public notificationTypes: Partial<NotificationAgentTypes>;

  public hasNotificationType(
    key: NotificationAgentKey,
    type: Notification
  ): boolean {
    return hasNotificationType(type, this.notificationTypes[key] ?? 0);
  }
}
```

### 6.2 默认值策略

不同 Agent 的默认启用状态不同：

```typescript
const defaultTypes = {
  email: ALL_NOTIFICATIONS,      // 邮件：默认全部启用
  discord: 0,                     // Discord：默认关闭
  pushbullet: 0,
  pushover: 0,
  slack: 0,
  telegram: 0,
  webhook: 0,
  webpush: ALL_NOTIFICATIONS,    // Web推送：默认全部启用
};
```

### 6.3 各渠道用户特定配置

除了通知类型开关，用户还可以配置各渠道的专属参数：

| 渠道 | 用户配置字段 | 说明 |
|------|-------------|------|
| Discord | `discordIds` | 用户的 Discord ID 列表，用于 @提及 |
| Email | `pgpKey` | PGP 公钥，用于加密邮件 |
| Pushbullet | `pushbulletAccessToken` | Pushbullet 访问令牌 |
| Pushover | `pushoverApplicationToken` / `pushoverUserKey` / `pushoverSound` | Pushover 配置 |
| Telegram | `telegramChatId` / `telegramMessageThreadId` / `telegramSendSilently` | Telegram 配置 |

---

## 七、消息组装与分发流程

### 7.1 完整流程图

```
业务逻辑（Subscriber）
       │
       ▼
notificationManager.sendNotification(type, payload)
       │
       ├─→ 遍历所有注册的 Agent
       │
       ├─→ Agent.shouldSend()
       │     └─ 检查系统级 enabled + 必要配置
       │
       └─→ Agent.send(type, payload)
             ├─ 检查系统级 types 配置（该通知类型是否启用）
             ├─ 确定接收人（notifyUser / notifyAdmin）
             ├─ 检查用户级 notificationTypes 偏好
             ├─ 组装消息格式（各渠道差异化）
             └─ 调用渠道 API 发送
```

### 7.2 触发入口：业务侧调用

以媒体请求为例，在 `server/subscriber/MediaRequestSubscriber.ts` 中触发通知：

```typescript
notificationManager.sendNotification(Notification.MEDIA_AVAILABLE, {
  event: 'Movie Request Now Available',
  notifyAdmin: false,
  notifySystem: true,
  notifyUser: entity.requestedBy,
  subject: movie.title,
  message: movie.overview,
  media: latestMedia,
  image: posterUrl,
  request: entity,
});
```

**Payload 关键字段说明：**

| 字段 | 作用 |
|------|------|
| `notifySystem` | 是否通过系统级渠道发送（如 Discord/Slack 群通知） |
| `notifyAdmin` | 是否需要通知管理员 |
| `notifyUser` | 需要通知的具体用户 |
| `subject` | 通知标题 |
| `message` | 通知正文 |
| `media` / `request` / `issue` | 关联的业务实体 |

### 7.3 分发逻辑：系统级检查

每个 Agent 的 `send()` 方法首先进行系统级检查：

```typescript
public async send(type, payload): Promise<boolean> {
  const settings = this.getSettings();

  // 系统级检查：是否通知系统渠道 + 该通知类型是否启用
  if (
    !payload.notifySystem ||
    !hasNotificationType(type, settings.types ?? 0)
  ) {
    return true; // 不发送，返回成功
  }
  // ... 发送逻辑
}
```

### 7.4 分发逻辑：用户级检查

针对用户通知（如 Email、WebPush），需要检查用户偏好：

```typescript
// 检查该用户是否启用了该渠道的该通知类型
if (payload.notifyUser.settings?.hasNotificationType(
  NotificationAgentKey.EMAIL,
  type
) ?? true) {
  // 发送给用户
}
```

### 7.5 分发逻辑：管理员通知

当 `notifyAdmin: true` 时，需要遍历所有管理员并发送：

```typescript
if (payload.notifyAdmin) {
  const users = await userRepository.find();
  await Promise.all(
    users
      .filter((user) =>
        // 检查用户偏好 + 管理员权限 + 排除操作者本人
        (user.settings?.hasNotificationType(agentKey, type) ?? true) &&
        shouldSendAdminNotification(type, user, payload)
      )
      .map(async (user) => {
        // 发送给每个管理员
      })
  );
}
```

`shouldSendAdminNotification()` 函数确保：
- 操作者本人不会收到通知
- 用户具有相应的管理权限
- 根据通知类型做特定过滤

---

## 八、各渠道 Agent 实现分析

### 8.1 渠道分类

根据发送方式，10 种渠道可分为三类：

#### 第一类：系统广播渠道（Channel-based）

发送到固定的频道/群组，不区分用户。

| 渠道 | 特点 | 配置位置 |
|------|------|----------|
| Discord | Webhook，支持 Rich Embed | 系统设置 |
| Slack | Webhook | 系统设置 |
| Telegram | Bot API，发送到固定 chatId | 系统设置 |
| Gotify | 自托管推送服务 | 系统设置 |
| Ntfy | 基于 HTTP 的推送服务 | 系统设置 |
| Webhook | 通用 HTTP 回调，支持自定义模板 | 系统设置 |

#### 第二类：用户定向渠道（User-based）

每个用户有独立的接收地址/设备。

| 渠道 | 特点 | 配置位置 |
|------|------|----------|
| Email | SMTP 邮件 | 系统设置 + 用户设置 |
| WebPush | 浏览器 Web Push API | 系统设置 + 用户订阅 |
| Pushbullet | 跨平台推送 | 系统设置 + 用户设置 |
| Pushover | 移动推送服务 | 系统设置 + 用户设置 |

### 8.2 Discord Agent 详解

文件：`server/lib/notifications/agents/discord.ts`

**特点：**
- 使用 Webhook 发送到 Discord 频道
- 支持 Rich Embed 富文本消息
- 支持 @用户 和 @角色
- 支持用户本地化语言

**消息组装：**

```typescript
public buildEmbed(type, payload, locale?): DiscordRichEmbed {
  const intl = getIntl(locale);
  const fields: Field[] = [];

  // 根据通知类型设置颜色和状态
  switch (type) {
    case Notification.MEDIA_PENDING:
      color = EmbedColors.ORANGE;
      status = pendingApproval;
      break;
    case Notification.MEDIA_AVAILABLE:
      color = EmbedColors.GREEN;
      status = available;
      break;
    // ... 其他类型
  }

  // 组装 fields
  if (payload.request) {
    fields.push({ name: requestedBy, value: userName, inline: true });
    // ...
  }

  return {
    title: payload.subject,
    description: payload.message,
    color,
    timestamp: new Date().toISOString(),
    fields,
    thumbnail: { url: payload.image },
  };
}
```

**用户提及机制：**
- 如果启用了 `enableMentions`，会在消息中 @具体用户
- 通过用户设置中的 `discordIds` 获取用户 Discord ID
- 支持管理员通知场景下批量 @所有相关管理员

### 8.3 Email Agent 详解

文件：`server/lib/notifications/agents/email.ts`

**特点：**
- 使用 SMTP 发送邮件
- 基于模板引擎（email-templates）
- 支持 PGP 加密
- 支持 HTML + 纯文本格式

**消息组装：**

```typescript
private buildMessage(type, payload, recipientEmail, recipientName, locale?) {
  const intl = getIntl(locale);
  
  // 根据通知类型选择邮件模板和正文
  if (payload.request) {
    switch (type) {
      case Notification.MEDIA_PENDING:
        body = intl.formatMessage(messages.pendingRequest, { mediaType });
        break;
      // ...
    }

    return {
      template: 'media-request',  // 模板名称
      message: { to: recipientEmail },
      locals: {
        body,
        mediaName: payload.subject,
        imageUrl: payload.image,
        actionUrl: applicationUrl,
        // ... 其他模板变量
      },
    };
  }
}
```

### 8.4 WebPush Agent 详解

文件：`server/lib/notifications/agents/webpush.ts`

**特点：**
- 基于 Web Push API + VAPID 认证
- 用户设备级订阅，存储在 `UserPushSubscription` 表
- 支持徽章数更新（pending requests count）
- 自动清理失效订阅

**错误处理：**

```typescript
// RFC 8030: 410/404 是永久性失败，其他是临时性的
const isPermanentFailure = statusCode === 410 || statusCode === 404;

if (isPermanentFailure) {
  await userPushSubRepository.remove(pushSub); // 移除无效订阅
}
```

### 8.5 Webhook Agent 详解

文件：`server/lib/notifications/agents/webhook.ts`

**特点：**
- 高度可定制的 HTTP 回调
- 支持变量模板（`{{variable}}` 语法）
- 支持自定义请求头和认证
- 支持 URL 路径变量替换

**变量映射：**

```typescript
const KeyMap: Record<string, string | KeyMapFunction> = {
  notification_type: (_payload, type) => Notification[type],
  subject: 'subject',
  message: 'message',
  media_type: 'media.mediaType',
  request_id: 'request.id',
  issue_status: (payload) => 
    payload.issue ? IssueStatus[payload.issue.status] : '',
  // ... 30+ 个变量
};
```

**模板解析：**
递归遍历 JSON 结构，替换所有 `{{key}}` 变量
支持特殊块：`{{media}}`、`{{request}}`、`{{issue}}`、`{{comment}}`、`{{extra}}`

---

## 九、设计模式与架构特点

### 9.1 策略模式（Strategy Pattern）

每种通知渠道都是一个策略实现，`NotificationManager` 根据配置选择执行哪些策略。

**优点：**
- 新增渠道只需添加新的 Agent 类
- 各渠道实现完全独立，互不影响
- 运行时可动态启用/禁用渠道

### 9.2 模板方法模式

`BaseAgent` 提供基础框架，子类实现具体的 `getSettings()`、消息构建和发送逻辑。

### 9.3 位掩码配置

使用位掩码存储通知类型偏好：

**优点：**
- 存储高效（一个数字代表所有类型开关）
- 判断高效（位运算 O(1)）
- 组合灵活（可随意组合多种类型）

**示例：**
```typescript
// 用户希望接收 "待审核" 和 "已可用" 通知
const userTypes = Notification.MEDIA_PENDING | Notification.MEDIA_AVAILABLE;
// 值为 2 | 8 = 10

// 检查是否启用了某类型
hasNotificationType(Notification.MEDIA_PENDING, userTypes); // true
hasNotificationType(Notification.MEDIA_FAILED, userTypes);  // false
```

### 9.4 两层过滤机制

**系统级过滤** + **用户级过滤** 双层设计：

1. **系统级**：管理员配置全局启用哪些渠道和哪些通知类型
2. **用户级**：每个用户自定义自己接收哪些类型的通知

---

## 十、关键代码引用速查

| 功能 | 文件 | 行号 |
|------|------|------|
| NotificationManager | `server/lib/notifications/index.ts` | 92-115 |
| 通知类型枚举 | `server/lib/notifications/index.ts` | 6-20 |
| hasNotificationType | `server/lib/notifications/index.ts` | 22-46 |
| shouldSendAdminNotification | `server/lib/notifications/index.ts` | 67-90 |
| Agent 基类与接口 | `server/lib/notifications/agents/agent.ts` | 26-38 |
| NotificationPayload | `server/lib/notifications/agents/agent.ts` | 9-24 |
| Agent 注册 | `server/index.ts` | 132-143 |
| 系统配置结构 | `server/lib/settings/index.ts` | 220-351 |
| NotificationAgentKey 枚举 | `server/lib/settings/index.ts` | 323-334 |
| 用户设置实体 | `server/entity/UserSettings.ts` | 31-154 |
| 用户 hasNotificationType | `server/entity/UserSettings.ts` | 148-153 |
| 媒体请求通知触发 | `server/subscriber/MediaRequestSubscriber.ts` | 78-94 |
| Discord Agent | `server/lib/notifications/agents/discord.ts` | 78-352 |
| Email Agent | `server/lib/notifications/agents/email.ts` | 62-410 |
| WebPush Agent | `server/lib/notifications/agents/webpush.ts` | 57-401 |
| Webhook Agent | `server/lib/notifications/agents/webhook.ts` | 66-250 |
| Gotify Agent | `server/lib/notifications/agents/gotify.ts` | 19-161 |
