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

## 十、发送失败的重试与降级策略

### 10.1 整体错误处理模式

通知框架采用 **Fire-and-Forget（发后即忘）** + **静默失败** 的设计哲学。所有 Agent 的发送失败都不会中断整体流程，也不会向上层抛出异常。

**统一错误处理模式**（以 Discord Agent 为例）：

```typescript
public async send(type, payload): Promise<boolean> {
  try {
    // 发送逻辑...
    await axios.post(webhookUrl, payload);
    return true;
  } catch (e) {
    logger.error('Error sending Discord notification', {
      label: 'Notifications',
      type: Notification[type],
      subject: payload.subject,
      errorMessage: e.message,
      response: e?.response?.data,
    });
    return false;
  }
}
```

**设计要点：**
- 每个 Agent 内部独立 try-catch，错误不冒泡
- 详细记录错误日志（类型、标题、错误信息、响应数据）
- 返回 `boolean` 表示成功与否，但调用方不依赖此返回值
- 单个 Agent 失败不影响其他 Agent 的发送

### 10.2 重试机制现状

**当前实现：无内置重试机制**

所有 10 个 Agent 均未实现自动重试逻辑。发送失败仅记录日志，不会进行二次尝试。

**原因分析：**
- 通知属于非核心路径，优先保证主流程不被阻塞
- 各渠道 API 的限流策略、重试成本差异较大
- 设计上倾向于"尽力而为"，而非"确保送达"

### 10.3 降级策略

虽然没有统一的重试机制，但部分 Agent 实现了特定场景下的降级处理：

#### WebPush：失效订阅自动清理

```typescript
// RFC 8030: 410/404 是永久性失败，其他是临时性的
const isPermanentFailure = statusCode === 410 || statusCode === 404;

logger.error(
  isPermanentFailure
    ? 'Error sending web push notification; removing invalid subscription'
    : 'Error sending web push notification (transient error, keeping subscription)',
  { /* ... */ }
);

if (isPermanentFailure) {
  await userPushSubRepository.remove(pushSub); // 永久失败：删除订阅
}
```

**降级策略：**
- **永久失败（410 Gone / 404 Not Found）**：主动删除数据库中的无效订阅，避免后续继续发送浪费资源
- **临时失败（其他状态码）**：保留订阅，下次通知时继续尝试

#### 图片加载降级（Pushover Agent）

```typescript
private async getImagePayload(imageUrl): Promise<Partial<PushoverImagePayload>> {
  try {
    const response = await axios.get(imageUrl, { responseType: 'arraybuffer' });
    return {
      attachment_base64: base64,
      attachment_type: contentType,
    };
  } catch (e) {
    logger.error('Error getting image payload', { /* ... */ });
    return {}; // 失败则返回空对象，不影响文字通知发送
  }
}
```

**降级策略：**
- 图片加载失败时，降级为纯文字通知
- 不因为附件问题导致整条通知发送失败

### 10.4 Manager 层面的容错

`NotificationManager.sendNotification()` 本身没有 try-catch，但由于：
1. 使用 `forEach` 而非 `for...of` + `await`
2. 每个 Agent 的 `send()` 都是异步且内部已捕获错误

因此单个 Agent 的异常不会影响其他 Agent，也不会导致 Manager 崩溃。

**潜在风险：**
- 发送结果完全异步，业务层无法知晓是否成功
- 没有失败统计和告警机制
- 长时间的静默失败可能导致问题被掩盖

---

## 十一、用户偏好配置：存储与读取热点分析

### 11.1 存储结构

用户通知偏好存储在 `UserSettings` 实体中，与 `User` 是一对一关系。

```typescript
@Entity()
export class User {
  @OneToOne(() => UserSettings, (settings) => settings.user, {
    cascade: true,
    eager: true,      // 关键：自动关联加载
    onDelete: 'CASCADE',
  })
  public settings?: UserSettings;
}
```

**核心字段：`notificationTypes`**

```typescript
@Column({
  type: 'text',
  nullable: true,
  transformer: {
    from: (value: string | null): Partial<NotificationAgentTypes> => {
      // JSON 反序列化 + 默认值填充
      const defaultTypes = {
        email: ALL_NOTIFICATIONS,
        webpush: ALL_NOTIFICATIONS,
        // ... 其他默认 0
      };
      // ...
    },
    to: (value): string | null => {
      // JSON 序列化 + 未知 key 过滤
      const allowedKeys = Object.values(NotificationAgentKey);
      (Object.keys(value)).forEach((key) => {
        if (!allowedKeys.includes(key)) {
          delete value[key];
        }
      });
      return JSON.stringify(value);
    },
  },
})
public notificationTypes: Partial<NotificationAgentTypes>;
```

**存储特点：**
- 数据库存储为 JSON 字符串（`text` 类型）
- 读取时自动反序列化为对象，并填充默认值
- 写入时自动序列化，并过滤未知 key（向前兼容）
- 每种渠道对应一个数字（bitmask），存储 10 个渠道偏好只需一条 JSON

### 11.2 读取路径分析

#### 路径一：请求认证时加载

在 `server/middleware/auth.ts:9-41` 的 `checkUser` 中间件中：

```typescript
if (req.session?.userId) {
  const userRepository = getRepository(User);
  user = await userRepository.findOne({
    where: { id: req.session.userId },
  });
}
// 由于 User.settings 配置了 eager: true，会自动 JOIN 查询
```

**特点：**
- 每个 HTTP 请求触发一次数据库查询
- 利用 `eager: true` 自动关联加载，无需显式 `relations`
- 请求处理期间用户对象挂在 `req.user` 上，可复用

#### 路径二：通知发送时查询

在管理员通知场景下，需要遍历所有用户：

```typescript
if (payload.notifyAdmin) {
  const userRepository = getRepository(User);
  const users = await userRepository.find(); // 全表扫描！

  await Promise.all(
    users
      .filter((user) => 
        user.settings?.hasNotificationType(agentKey, type) &&
        shouldSendAdminNotification(type, user, payload)
      )
      .map(async (user) => { /* 发送通知 */ })
  );
}
```

**热点问题：**
- 每次管理员通知都会执行一次全表查询 `userRepository.find()`
- 如果有 10 个启用的 Agent，每个都会独立查一次用户表
- 用户量大时（>1000），性能问题显著

### 11.3 系统级配置 vs 用户级配置

| 配置层级 | 存储位置 | 加载时机 | 缓存策略 |
|---------|---------|---------|---------|
| 系统级 | `settings.json` 文件 | 服务启动时加载一次 | 内存单例，全程复用 |
| 用户级 | 数据库 `user_settings` 表 | 请求时 / 发通知时查询 | 无缓存，每次查库 |

**系统级配置的单例模式：**

```typescript
let settings: Settings | undefined;

export const getSettings = (): Settings => {
  if (!settings) {
    settings = new Settings();
  }
  return settings;
};
```

系统设置启动时调用 `load()` 读取文件，之后全程内存访问，性能极高。

### 11.4 性能热点与优化空间

**当前热点：**

1. **管理员通知的 N+1 查询问题**
   - 每个 Agent 独立查询用户表
   - 10 个 Agent × 1 次全表查询 = 10 次重复查询

2. **JSON 反序列化开销**
   - 每次读取 `notificationTypes` 都要执行 `JSON.parse`
   - 还要进行默认值填充和类型转换

3. **无用户级缓存**
   - 同一次通知流程中，多个 Agent 可能重复查询同一用户
   - 短时间内多条通知触发时，用户数据反复加载

**可优化方向：**
- 在 `NotificationManager` 层统一查询一次用户列表，传给各 Agent
- 增加用户设置的内存缓存（带过期时间）
- 考虑将常用的通知类型字段冗余到 User 表，避免 JSON 解析

---

## 十二、多 Provider 并发发送：顺序与去重机制

### 12.1 并发发送模型

#### Manager 层：并发触发，顺序注册

```typescript
public sendNotification(type, payload): void {
  this.activeAgents.forEach((agent) => {
    if (agent.shouldSend()) {
      agent.send(type, payload); // 没有 await！
    }
  });
}
```

**并发特点：**
- 使用 `forEach` 遍历，按 Agent 注册顺序依次调用
- `agent.send()` 返回 Promise 但不被 await，属于 **fire-and-forget**
- 所有 Agent 几乎同时开始发送（并发执行）
- 完成顺序取决于各渠道 API 的响应速度，不确定

#### Agent 内部：用户级并发

对于需要发给多个用户的 Agent（如 Email、WebPush），内部使用 `Promise.all` 并发发送：

```typescript
if (payload.notifyAdmin) {
  const users = await userRepository.find();
  await Promise.all(
    users.filter(...).map(async (user) => {
      // 每个用户独立发送
      await sendToUser(user);
    })
  );
}
```

### 12.2 发送顺序

**触发顺序（确定）：** 按 `registerAgents()` 中的注册顺序

当前注册顺序（`server/index.ts:132-143`）：
1. DiscordAgent
2. EmailAgent
3. GotifyAgent
4. NtfyAgent
5. PushbulletAgent
6. PushoverAgent
7. SlackAgent
8. TelegramAgent
9. WebhookAgent
10. WebPushAgent

**完成顺序（不确定）：** 取决于各渠道 API 响应时间、网络延迟等因素。

### 12.3 去重机制

#### 层面一：同一渠道内的系统/用户去重

Pushover、Telegram 等同时支持系统级和用户级发送的 Agent，实现了**同一渠道内的去重判断**：

**Pushover Agent 示例**（`server/lib/notifications/agents/pushover.ts:244-256`）：

```typescript
if (payload.notifyUser) {
  if (
    payload.notifyUser.settings?.hasNotificationType(NotificationAgentKey.PUSHOVER, type) &&
    payload.notifyUser.settings.pushoverApplicationToken &&
    payload.notifyUser.settings.pushoverUserKey &&
    // 去重判断：用户 token 与系统 token 不同时才单独发送
    (payload.notifyUser.settings.pushoverApplicationToken !== settings.options.accessToken ||
     payload.notifyUser.settings.pushoverUserKey !== settings.options.userToken)
  ) {
    // 发送用户级通知
  }
}
```

**Telegram Agent 示例**（`server/lib/notifications/agents/telegram.ts:220-228`）：

```typescript
if (payload.notifyUser) {
  if (
    payload.notifyUser.settings?.hasNotificationType(NotificationAgent.TELEGRAM, type) &&
    payload.notifyUser.settings?.telegramChatId &&
    // 去重判断：用户 chatId 与系统 chatId 不同时才单独发送
    payload.notifyUser.settings.telegramChatId !== settings.options.chatId
  ) {
    // 发送用户级通知
  }
}
```

**逻辑：**
- 如果用户配置的接收地址（chatId / userKey）和系统级配置是同一个，就不重复发送
- 避免用户同时通过"系统频道"和"个人直达"收到两份相同通知

#### 层面二：管理员通知的操作者去重

`shouldSendAdminNotification()` 函数确保**操作触发者本人不会收到通知**：

```typescript
export const shouldSendAdminNotification = (type, user, payload): boolean => {
  return (
    user.id !== payload.notifyUser?.id &&
    user.hasPermission(getAdminPermission(type)) &&
    // 媒体自动批准：排除请求提交者
    (type !== Notification.MEDIA_AUTO_APPROVED ||
      user.id !== (payload.request?.modifiedBy ?? payload.request?.requestedBy)?.id) &&
    // 问题创建：排除创建者
    (type !== Notification.ISSUE_CREATED || user.id !== payload.issue?.createdBy.id) &&
    // 问题评论：排除评论者
    (type !== Notification.ISSUE_COMMENT || user.id !== payload.comment?.user.id) &&
    // 问题解决/重开：排除操作者
    ((type !== Notification.ISSUE_RESOLVED && type !== Notification.ISSUE_REOPENED) ||
      user.id !== payload.issue?.modifiedBy?.id)
  );
};
```

#### 层面三：跨渠道去重

**现状：无跨渠道去重机制**

如果用户同时启用了 Email、WebPush、Discord 等多个渠道，且这些渠道都配置了接收该类型通知，用户会在多个终端收到内容相同的通知。

**设计考量：**
- 各渠道定位不同（即时性、可达性、场景）
- 用户自主选择启用哪些渠道，即表示愿意接收多渠道通知
- 框架层面不做"智能选路"，保持简单透明

### 12.4 并发与去重总结

| 维度 | 机制 | 说明 |
|------|------|------|
| 触发方式 | 并发 fire-and-forget | 所有 Agent 同时触发，不等待结果 |
| 触发顺序 | 按注册顺序 | 顺序确定，但完成顺序不确定 |
| 同渠道去重 | 有（部分 Agent） | 比较系统配置和用户配置，相同则不重复发 |
| 操作者去重 | 有 | 触发事件的用户本人不会收到管理员通知 |
| 跨渠道去重 | 无 | 用户启用多个渠道会收到多份 |
| 多设备去重 | 无 | WebPush 多个订阅设备会各自收到 |

---

## 十三、通知模板的国际化与多语言支持

### 13.1 国际化框架选型

基于 `@formatjs/intl` 库实现，遵循 ICU Message Format 标准。核心初始化在 `server/i18n/index.ts`：

```typescript
const cache = createIntlCache();
const intls = new Map<string, IntlInstance>();

export function initI18n(): void {
  for (const locale of availableLocales) {
    const filePath = path.join(__dirname, `locale/${locale}.json`);
    if (!fs.existsSync(filePath)) continue;

    const messages = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
    intls.set(
      locale,
      createIntl(
        { locale, messages, defaultLocale: 'en' },
        cache
      )
    );
  }
}

export function getIntl(locale?: AvailableLocale): IntlInstance {
  return intls.get(locale ?? 'en') || intls.get('en')!;
}
```

**设计要点：**
- 服务启动时一次性加载所有语言包到内存
- 39 种语言支持（en、zh-Hans、zh-Hant、ja、ko、fr、de 等）
- 使用 `createIntlCache` 复用缓存，提升性能
- 默认回退到英语（en），保证不崩溃

### 13.2 消息定义与使用模式

#### 全局通用消息

定义在 `server/i18n/globalMessages.ts`，所有 Agent 共享：

```typescript
const globalMessages = defineMessages('notifications.common', {
  requestedBy: 'Requested By',
  requestStatus: 'Request Status',
  pendingApproval: 'Pending Approval',
  available: 'Available',
  declined: 'Declined',
  failed: 'Failed',
  commentFrom: 'Comment from {userName}',  // 支持变量插值
  reportedBy: 'Reported By',
  issueStatus: 'Issue Status',
  viewIssue: 'View Issue in {applicationTitle}',
  // ... 共 24 条通用消息
});
```

#### Agent 私有消息

部分 Agent 内部使用 `defineMessages` 定义私有消息（以 WebPush 为例）：

```typescript
const messages = defineMessages('notifications.webpush', {
  pendingRequest: 'A {mediaType} request requires your approval',
  requestApproved: 'Your {mediaType} request has been approved',
  requestAvailable: 'Your {mediaType} request is now available',
  // ...
});

// 使用方式
const intl = getIntl(locale);
const body = intl.formatMessage(messages.pendingRequest, { mediaType });
```

### 13.3 语言选择策略

#### 系统级通知（Channel-based）

使用系统配置的语言，每个 Agent 独立配置：

```typescript
// Slack Agent 示例
const intl = getIntl(settings.options.locale);
```

系统级 Agent 的 `locale` 配置在 `settings.json` 中：

```json
{
  "notifications": {
    "agents": {
      "discord": { "options": { "locale": "en" } },
      "slack":   { "options": { "locale": "zh-Hans" } },
      "email":   { "options": { "locale": "en" } }
    }
  }
}
```

#### 用户级通知（User-based）

使用接收用户的偏好语言，从 `user.settings.locale` 获取：

```typescript
// WebPush Agent 示例
const locale = payload.notifyUser?.settings?.locale as AvailableLocale;
const intl = getIntl(locale);
```

```typescript
// Email Agent 示例
this.buildMessage(type, payload, email, name, locale) {
  const intl = getIntl(locale);
  // ...
}
```

### 13.4 模板系统分层

#### 第一层：富文本模板（Discord/Slack 等）

通过程序化构建消息结构，不依赖模板文件：

```typescript
// Discord Agent buildEmbed 方法
public buildEmbed(type, payload, locale?): DiscordRichEmbed {
  const intl = getIntl(locale);
  
  switch (type) {
    case Notification.MEDIA_PENDING:
      color = EmbedColors.ORANGE;
      status = intl.formatMessage(globalMessages.pendingApproval);
      break;
    // ...
  }

  if (payload.request) {
    fields.push({
      name: intl.formatMessage(globalMessages.requestedBy),
      value: payload.request.requestedBy.displayName,
      inline: true,
    });
  }

  return { title, description, color, fields, thumbnail };
}
```

#### 第二层：邮件模板（Pug 模板引擎）

邮件使用 Pug 模板文件，位于 `server/templates/email/`，支持 HTML 和纯文本：

```
server/templates/email/
├── media-request/       # 媒体请求通知
│   ├── html.pug         # HTML 版本
│   └── subject.pug      # 邮件标题
├── media-issue/         # 问题通知
│   ├── html.pug
│   └── subject.pug
├── generatedpassword/   # 生成密码
├── resetpassword/       # 重置密码
└── test-email/          # 测试邮件
```

**模板使用方式（Email Agent）：**

```typescript
return {
  template: 'media-request',  // 模板目录名
  message: { to: recipientEmail },
  locals: {
    body,                    // 国际化后的正文
    mediaName: payload.subject,
    imageUrl: payload.image,
    actionUrl: applicationUrl,
    username: recipientName,
    applicationTitle,
    // ... 其他模板变量
  },
};
```

#### 第三层：简洁消息模板（Gotify/Ntfy/WebPush 等）

纯文本或简单 Markdown 格式，程序化拼接：

```typescript
// Ntfy Agent
const title = payload.event 
  ? `${payload.event} - ${payload.subject}` 
  : payload.subject;

let message = payload.message ?? '';
if (payload.request) {
  message += `\n**${intl.formatMessage(globalMessages.requestedBy)}:** ${userName}`;
  message += `\n**${intl.formatMessage(globalMessages.requestStatus)}:** ${status}`;
}
```

### 13.5 国际化覆盖范围

| 渠道 | 国际化支持 | 语言来源 | 模板方式 |
|------|-----------|---------|---------|
| Discord | ✅ 完整 | 系统配置 / 用户设置 | 程序化构建 Embed |
| Email | ✅ 完整 | 用户设置 | Pug 模板 + 国际化变量 |
| WebPush | ✅ 完整 | 用户设置 | 程序化拼接 |
| Slack | ✅ 完整 | 系统配置 | 程序化构建 Block |
| Telegram | ✅ 完整 | 系统配置 / 用户设置 | 程序化拼接 |
| Pushover | ✅ 完整 | 系统配置 / 用户设置 | 程序化拼接 |
| Pushbullet | ✅ 完整 | 系统配置 / 用户设置 | 程序化拼接 |
| Gotify | ✅ 完整 | 系统配置 | 程序化拼接 |
| Ntfy | ✅ 完整 | 系统配置 | 程序化拼接 |
| Webhook | ❌ 无 | - | 用户自定义模板，原始数据透传 |

**Webhook 的特殊处理：**
Webhook 不做国际化，直接传递原始数据（英文）给用户配置的回调地址，由接收方自行处理本地化。

### 13.6 语言配置存储

用户语言偏好存储在 `UserSettings.locale`：

```typescript
// server/entity/UserSettings.ts
@Column({ default: 'en' })
public locale: string;
```

数据库迁移记录：`server/migration/sqlite/1619239659754-AddUserSettingsLocale.ts`

---

## 十四、批量通知的合并与节流机制

### 14.1 当前机制：无内置合并与节流

通知框架目前**没有**实现任何批量合并（batching）或节流（throttling）机制。

**核心特征：**
- 每次业务事件触发都独立调用 `notificationManager.sendNotification()`
- 没有队列、没有缓冲、没有合并窗口
- 所有通知实时发出，不做延迟聚合

### 14.2 通知触发源分析

通知通过 TypeORM 生命周期钩子和 Subscriber 触发：

#### 触发源一：实体生命周期钩子

`MediaRequest` 实体中使用 `@AfterInsert` 和 `@AfterUpdate`：

```typescript
@Entity()
export class MediaRequest {
  @AfterInsert()
  public async notifyNewRequest(): Promise<void> {
    if (this.status === MediaRequestStatus.PENDING) {
      MediaRequest.sendNotification(this, media, Notification.MEDIA_PENDING);
      
      if (this.isAutoRequest) {
        MediaRequest.sendNotification(this, media, Notification.MEDIA_AUTO_REQUESTED);
      }
    }
  }

  @AfterUpdate()
  public async notifyApprovedOrDeclined(): Promise<void> {
    if (this.status === MediaRequestStatus.APPROVED) {
      MediaRequest.sendNotification(this, media, Notification.MEDIA_APPROVED);
    }
  }
}
```

**潜在问题：自动请求场景的重复通知**

当 `isAutoRequest: true` 时，`notifyNewRequest()` 会连续触发两次通知：
1. `MEDIA_PENDING` - 待审核通知
2. `MEDIA_AUTO_REQUESTED` - 自动请求通知

两条通知内容相似，同时推送给用户，造成干扰。

#### 触发源二：Subscriber 模式

`IssueSubscriber` 和 `MediaRequestSubscriber` 使用 TypeORM 的事件订阅：

```typescript
@EventSubscriber()
export class IssueSubscriber implements EntitySubscriberInterface<Issue> {
  public afterInsert(event: InsertEvent<Issue>): void {
    this.sendIssueNotification(event.entity, Notification.ISSUE_CREATED);
  }

  public beforeUpdate(event: UpdateEvent<Issue>): void {
    if (event.entity.status === IssueStatus.RESOLVED && 
        event.databaseEntity.status !== IssueStatus.RESOLVED) {
      this.sendIssueNotification(event.entity, Notification.ISSUE_RESOLVED);
    }
  }
}
```

### 14.3 批量场景下的行为

#### 场景一：批量导入媒体

当同步 Radarr/Sonarr 库时，可能短时间内大量媒体变为可用状态：

- 每个媒体实体独立触发 `AfterUpdate`
- 每个媒体独立调用 `sendNotification(Notification.MEDIA_AVAILABLE)`
- 100 个媒体变为可用 → 100 × 10 个 Agent = 1000 次 API 调用

#### 场景二：批量审核请求

管理员批量批准 50 个请求：

- 每个 `MediaRequest` 实体独立触发 `AfterUpdate`
- 每个请求独立发出 `MEDIA_APPROVED` 通知
- 50 个请求 × 10 个 Agent = 500 次 API 调用
- 每个用户可能瞬间收到 50 条通知

#### 场景三：剧集季级可用

一部剧集的 24 集同时下载完成：

- 每集独立触发状态更新
- 每集独立发出 `MEDIA_AVAILABLE` 通知
- 用户可能收到 24 条"XX 第 X 集已可用"的重复类型通知

### 14.4 唯一的隐式合并：状态跳转合并

在 `MediaRequest.notifyApprovedOrDeclined()` 中有一处**隐式合并**：

```typescript
@AfterUpdate()
public async notifyApprovedOrDeclined(autoApproved = false): Promise<void> {
  if (this.status === MediaRequestStatus.APPROVED) {
    // 如果媒体已经可用，跳过 APPROVED 通知，直接发 AVAILABLE 通知
    if (media[is4k ? 'status4k' : 'status'] === MediaStatus.AVAILABLE) {
      logger.info('Media is already available. Sending availability notification instead of approval.', {
        label: 'Media Request',
        requestId: this.id,
      });
      MediaRequest.sendNotification(this, media, Notification.MEDIA_AVAILABLE);
      return; // 跳过 APPROVED 通知
    }
    
    MediaRequest.sendNotification(this, media, Notification.MEDIA_APPROVED);
  }
}
```

**逻辑：**
- 当批准请求时，如果媒体已经是可用状态
- 不发送"已批准"通知，直接发送"已可用"通知
- 避免用户连续收到两条内容相似的通知

### 14.5 节流机制的缺失

目前完全没有节流保护：

| 维度 | 现状 | 风险 |
|------|------|------|
| 单用户通知频率 | 无限制 | 1 分钟内可能收到几十条通知 |
| 单渠道 API 调用频率 | 无限制 | 可能触发 Discord/Telegram 等平台的限流 |
| 全局通知 QPS | 无限制 | 批量操作时可能瞬间耗尽资源 |
| 重复内容检测 | 无 | 相同类型通知可能在短时间内反复发送 |

### 14.6 优化方向建议

1. **引入合并窗口（Batching Window）**
   - 同类型、同用户的通知在 5-10 秒窗口内合并
   - 合并为 "XX 等 5 部电影已可用" 的摘要通知

2. **按用户级别节流**
   - 单个用户每分钟最多接收 N 条通知
   - 超出部分延迟发送或合并

3. **按渠道级别限流**
   - 针对每个外部 API 设置 QPS 限制
   - 使用令牌桶或漏桶算法

4. **批量操作显式标记**
   - 批量批准/导入时，显式跳过单条通知
   - 操作完成后发送一条汇总通知

---

## 十五、通知失败时的告警和运维介入流程

### 15.1 日志系统基础

通知框架使用 `winston` 作为日志库，配置在 `server/logger.ts`。

#### 三个日志输出通道

```typescript
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL?.toLowerCase() || 'debug',
  format: winston.format.combine(
    winston.format.splat(),
    winston.format.timestamp(),
    hformat
  ),
  transports: [
    new winston.transports.Console({ /* 彩色控制台输出 */ }),
    seerrFileTransport,        // 人类可读文本日志
    machineLogFileTransport,   // 机器可读 JSON 日志
  ],
});
```

#### 文件日志配置

| 日志类型 | 文件 | 保留策略 | 格式 |
|---------|------|---------|------|
| 应用日志 | `seerr-%DATE%.log` | 7 天，20MB 滚动 | 人类可读文本 |
| 机器日志 | `.machinelogs-%DATE%.json` | 1 天，20MB 滚动 | JSON 格式 |

### 15.2 通知失败的日志记录

每个 Agent 的 `send()` 方法都有统一的错误日志模式：

```typescript
try {
  await axios.post(webhookUrl, payload);
  return true;
} catch (e) {
  logger.error('Error sending Discord notification', {
    label: 'Notifications',           // 固定标签，方便筛选
    type: Notification[type],         // 通知类型
    subject: payload.subject,         // 通知标题
    errorMessage: e.message,          // 错误消息
    response: e?.response?.data,      // API 响应（如果有）
  });
  return false;
}
```

**日志关键字段说明：**
- `label: 'Notifications'` - 可用于过滤所有通知相关日志
- `type` - 业务场景（MEDIA_PENDING / MEDIA_AVAILABLE 等）
- `errorMessage` - 错误原因（网络超时、认证失败、权限不足等）
- `response` - 外部 API 返回的错误详情，用于诊断

### 15.3 机器日志的结构化

`.machinelogs.json` 为结构化 JSON 格式，便于后续分析：

```json
{
  "timestamp": "2026-06-14T10:30:00.123Z",
  "level": "error",
  "label": "Notifications",
  "message": "Error sending Discord notification",
  "type": "MEDIA_AVAILABLE",
  "subject": "Inception (2010)",
  "errorMessage": "Request failed with status code 401",
  "response": { "message": "Invalid Webhook Token" }
}
```

### 15.4 告警机制：完全缺失

**当前状态：没有任何内置告警机制。**

| 告警维度 | 现状 | 说明 |
|---------|------|------|
| 失败率告警 | ❌ 无 | 即使 100% 的通知都失败，也不会主动告警 |
| 连续失败告警 | ❌ 无 | 某个渠道连续失败 N 次，不会触发告警 |
| 配置错误告警 | ❌ 无 | Webhook URL 无效、API Token 过期等配置问题，仅记录日志 |
| 外部服务不可用告警 | ❌ 无 | SMTP 服务器宕机、Discord API 不可用，仅记录日志 |
| 发送延迟告警 | ❌ 无 | 无超时监控，发送多久都不会告警 |

### 15.5 运维介入流程：完全被动

**当前运维流程：**

```
用户反馈 "没收到通知"
       ↓
管理员登录服务器查看 seerr.log
       ↓
搜索 label:Notifications 相关错误
       ↓
根据 errorMessage 和 response 诊断原因
       ↓
手动修复配置或联系外部服务
       ↓
通过测试通知功能验证修复
```

**典型故障排查路径：**

1. **Discord Webhook 失效**
   - 日志：`errorMessage: "Request failed with status code 404"`
   - 原因：Webhook URL 被删除或过期
   - 修复：重新配置 Webhook URL

2. **SMTP 认证失败**
   - 日志：`errorMessage: "Invalid login: 535 Authentication failed"`
   - 原因：邮箱密码过期或被封禁
   - 修复：更新 SMTP 密码

3. **WebPush 订阅过期**
   - 日志：`"removing invalid subscription"`
   - 原因：用户浏览器推送订阅已过期
   - 修复：系统自动处理，无需人工干预

### 15.6 失败通知的测试机制

系统提供测试通知功能，用于验证配置正确性：

```typescript
// 测试通知调用
notificationManager.sendNotification(Notification.TEST_NOTIFICATION, {
  event: 'Test Notification',
  subject: 'Test Notification',
  message: 'This is a test notification from Overseerr.',
  notifySystem: true,
  notifyUser: currentUser,
});
```

**注意：`TEST_NOTIFICATION` 类型特殊处理**
- 在 `hasNotificationType()` 中，测试通知会强制启用
- 不需要在用户/系统设置中显式启用该类型
- 方便测试配置，无需临时修改通知类型开关

### 15.7 优化方向建议

1. **失败计数器**
   - 为每个 Agent 维护连续失败计数器
   - 连续失败 N 次（如 10 次）触发告警

2. **失败率监控**
   - 统计滑动窗口内的失败率
   - 失败率 > 20% 触发告警

3. **健康检查端点**
   - 暴露 `/api/v1/health/notifications` 端点
   - 返回各渠道最近 1 小时的发送统计

4. **告警集成**
   - 支持将通知失败事件转发到运维告警系统
   - 复用现有 Webhook Agent 发送告警

5. **配置校验**
   - 保存配置时主动验证连通性
   - 配置错误立即提示用户，而非等到实际发送失败

---

## 十六、Webhook 自定义通知端点扩展

### 16.1 Webhook 的核心扩展能力

Webhook Agent 是所有渠道中扩展性最强的，提供了从端点 URL 到请求体、认证方式的全面自定义。

#### 配置结构定义

```typescript
export interface NotificationAgentWebhook extends NotificationAgentConfig {
  options: {
    webhookUrl: string;           // 端点 URL，支持变量替换
    jsonPayload: string;          // Base64 编码的自定义 JSON 模板
    authHeader?: string;          // 快捷 Authorization 头
    customHeaders?: {             // 任意自定义请求头
      key: string;
      value: string;
    }[];
    supportVariables?: boolean;   // URL 是否启用变量替换
  };
}
```

### 16.2 端点 URL 的动态路由

#### 静态端点模式（默认）

```json
{
  "webhookUrl": "https://api.example.com/notify"
}
```

所有通知都发送到同一个 URL，通过请求体中的字段区分业务类型。

#### 动态端点模式（`supportVariables: true`）

启用后，URL 路径中的 `{{变量}}` 会被替换为实际值，实现**按业务类型路由**：

```typescript
// 示例：按通知类型路由到不同端点
{
  "webhookUrl": "https://api.example.com/hook/{{notification_type}}?user={{notifyuser_username}}"
}
```

发送时的替换逻辑（`webhook.ts:186-202`）：

```typescript
if (settings.options.supportVariables) {
  Object.keys(KeyMap).forEach((keymapKey) => {
    const variableValue = 
      type === Notification.TEST_NOTIFICATION
        ? 'test'
        : typeof keymapValue === 'function'
          ? keymapValue(payload, type)
          : get(payload, keymapValue) || 'test';

    webhookUrl = webhookUrl.replace(
      new RegExp(`{{${keymapKey}}}`, 'g'),
      encodeURIComponent(variableValue)  // URL 编码，防止特殊字符
    );
  });
}
```

**动态路由应用场景：**
1. **按通知类型分发**：`/hook/{{notification_type}}` → `/hook/MEDIA_AVAILABLE`
2. **按用户分发**：`/hook/user/{{notifyuser_username}}`
3. **按媒体类型分发**：`/hook/media/{{media_type}}`
4. **查询参数透传**：`?media_id={{media_tmdbid}}&request_id={{request_id}}`

### 16.3 自定义请求体模板

#### 模板变量映射表

Webhook 内置 **63 个预定义变量**，通过 `KeyMap` 映射到 `NotificationPayload` 字段：

| 分类 | 变量名 | 说明 |
|------|--------|------|
| 通用 | `{{notification_type}}` | 通知类型枚举名 |
| 通用 | `{{event}}`, `{{subject}}`, `{{message}}`, `{{image}}` | 基础字段 |
| 通知用户 | `{{notifyuser_username}}`, `{{notifyuser_email}}`, `{{notifyuser_avatar}}` | 接收用户信息 |
| 通知用户 | `{{notifyuser_settings_discordIds}}`, `{{notifyuser_settings_telegramChatId}}` | 用户渠道配置 |
| 请求信息 | `{{request_id}}`, `{{requestedBy_username}}`, `{{requestedBy_email}}` | 请求相关 |
| 请求信息 | `{{requestedBy_settings_discordIds}}` 等 | 请求发起者信息 |
| 媒体信息 | `{{media_tmdbid}}`, `{{media_imdbid}}`, `{{media_tvdbid}}`, `{{media_type}}` | 媒体标识 |
| 媒体信息 | `{{media_status}}`, `{{media_status4k}}`, `{{media_jellyfinMediaId}}` | 媒体状态 |
| 问题信息 | `{{issue_id}}`, `{{issue_type}}`, `{{issue_status}}` | 问题相关 |
| 问题信息 | `{{reportedBy_username}}`, `{{commentedBy_username}}`, `{{comment_message}}` | 问题相关 |

**两种变量映射方式：**

```typescript
// 方式一：路径映射（直接从 payload 取值）
media_tmdbid: 'media.tmdbId',

// 方式二：函数映射（动态计算）
media_jellyfinMediaId: (payload) =>
  payload.media?.jellyfinMediaId ?? payload.media?.jellyfinMediaId4k ?? '',

notification_type: (_payload, type) => Notification[type],
```

#### 特殊块变量

除了标量变量，还支持 5 个**对象级块变量**，实现整块数据的透传：

```typescript
// parseKeys 中的特殊块处理
if (key === '{{media}}') {
  finalPayload.media = payload.media ? finalPayload[key] : null;
  delete finalPayload[key];
}
// 同理：{{request}}, {{issue}}, {{comment}}, {{extra}}
```

**使用示例：**

```json
{
  "event": "{{notification_type}}",
  "title": "{{subject}}",
  "{{media}}": {},
  "{{request}}": {},
  "custom": "data"
}
```

发送后 `{{media}}` 会被替换为完整的 `media` 对象，没有媒体时为 `null`。

#### 模板的编码存储

出于安全考虑，JSON 模板在保存时做 **Base64 编码**：

```typescript
// 保存时：JSON → 字符串化 → Base64
settings.notifications.agents.webhook = {
  options: {
    jsonPayload: Buffer.from(
      JSON.stringify(req.body.options.jsonPayload)
    ).toString('base64'),
  },
};

// 读取时：Base64 → 字符串化 → JSON 解析
private buildPayload(type, payload) {
  const payloadString = Buffer.from(
    this.getSettings().options.jsonPayload,
    'base64'
  ).toString('utf8');
  const parsedJSON = JSON.parse(JSON.parse(payloadString)); // 双 parse！
  return this.parseKeys(parsedJSON, payload, type);
}
```

> **注意双 JSON.parse**：保存时 `JSON.stringify(对象)` 再编码，读取时解码后得到 `'{"key":"value"}'` 字符串，再 `JSON.parse` 一次得到对象。

#### 模板递归解析

`parseKeys()` 使用 **深度优先递归**，确保嵌套对象中的所有变量都被替换：

```typescript
private parseKeys(finalPayload, payload, type) {
  Object.keys(finalPayload).forEach((key) => {
    // 先处理特殊块变量 ...
    
    if (typeof finalPayload[key] === 'string') {
      // 标量：遍历 KeyMap 做字符串替换
      Object.keys(KeyMap).forEach((keymapKey) => {
        finalPayload[key] = (finalPayload[key] as string).replace(
          `{{${keymapKey}}}`,
          value
        );
      });
    } else if (finalPayload[key] && typeof finalPayload[key] === 'object') {
      // 对象/数组：递归解析
      finalPayload[key] = this.parseKeys(
        finalPayload[key] as Record<string, unknown>,
        payload,
        type
      );
    }
  });
  return finalPayload;
}
```

### 16.4 认证与自定义请求头

#### 三层头配置

```typescript
const headers: Record<string, string> = {};

// 第一层：快捷 Authorization 头
if (settings.options.authHeader) {
  headers.Authorization = settings.options.authHeader;
}

// 第二层：自定义请求头数组
if (settings.options.customHeaders?.length > 0) {
  settings.options.customHeaders.forEach((header) => {
    const key = header.key?.trim();
    const value = header.value?.trim();

    if (key && value) {
      // 第三层：保护机制 - 自定义 Authorization 不覆盖快捷配置
      if (
        key.toLowerCase() !== 'authorization' ||
        !settings.options.authHeader
      ) {
        headers[key] = value;
      }
    }
  });
}
```

**认证配置示例：**

| 场景 | 配置方式 |
|------|---------|
| Bearer Token | `authHeader: "Bearer abc123"` |
| Basic Auth | `authHeader: "Basic dXNlcjpwYXNz"` |
| API Key（自定义头） | `customHeaders: [{key: "X-API-Key", value: "xyz"}]` |
| 签名回调 | `customHeaders: [{key: "X-Signature", value: "sha256=..."}]` |
| 多租户标识 | `customHeaders: [{key: "X-Tenant-Id", value: "tenant-42"}]` |

### 16.5 Webhook API 管理端点

通知设置路由位于 `server/routes/settings/notifications.ts`，提供完整的 CRUD + Test 接口：

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/settings/notifications/webhook` | 获取配置（自动解码 JSON 模板） |
| POST | `/settings/notifications/webhook` | 保存配置（自动编码 JSON 模板） |
| POST | `/settings/notifications/webhook/test` | 发送测试通知 |

所有 10 种渠道都有相同模式的三个接口（GET / POST / POST /test）。

---

## 十七、通知历史记录与审计查询

### 17.1 当前状态：完全缺失

**通知框架目前没有任何历史记录存储和审计查询功能。**

| 能力 | 现状 | 说明 |
|------|------|------|
| 发送记录持久化 | ❌ 无 | 发送结果仅写入日志文件，不入库 |
| 送达状态追踪 | ❌ 无 | 无法知道每条通知是否成功送达 |
| 发送历史查询 API | ❌ 无 | 无任何查询接口 |
| 管理员审计视图 | ❌ 无 | UI 层也没有通知历史页面 |
| 用户发送记录查询 | ❌ 无 | 用户无法查看自己收到过哪些通知 |
| 通知失败审计 | ❌ 无 | 失败记录仅在日志中，无法结构化查询 |

### 17.2 当前唯一的记录方式：日志

所有通知相关的信息仅通过 `winston` 日志系统记录：

#### 发送开始日志（Debug 级别）

```json
{
  "timestamp": "2026-06-14T10:30:00.100Z",
  "level": "debug",
  "label": "Notifications",
  "message": "Sending Slack notification",
  "type": "MEDIA_AVAILABLE",
  "subject": "Inception (2010)"
}
```

#### 发送成功日志（无显式记录）

目前成功发送时，只有 `return true`，**没有对应的成功日志**。这是一个设计缺陷。

#### 发送失败日志（Error 级别）

```json
{
  "timestamp": "2026-06-14T10:30:00.150Z",
  "level": "error",
  "label": "Notifications",
  "message": "Error sending Slack notification",
  "type": "MEDIA_AVAILABLE",
  "subject": "Inception (2010)",
  "errorMessage": "Request failed with status code 401",
  "response": { "message": "Invalid Webhook Token" }
}
```

#### 管理器总览日志（Info 级别）

```json
{
  "timestamp": "2026-06-14T10:30:00.090Z",
  "level": "info",
  "label": "Notifications",
  "message": "Sending notification(s) for MEDIA_AVAILABLE",
  "subject": "Inception (2010)"
}
```

### 17.3 日志分析的局限性

**为什么日志不能替代历史记录：**

1. **查询困难**：需要登录服务器、grep 文本日志，无法按条件筛选
2. **无法聚合**：无法统计某用户/某渠道/某类型的成功率
3. **保留时间短**：`seerr.log` 仅保留 7 天，`.machinelogs.json` 仅保留 1 天
4. **成功记录缺失**：成功发送没有对应日志条目，无法判断是否发送过
5. **无关联数据**：日志中缺少 requestId、userId 等关联键，无法追溯

### 17.4 建议的历史记录存储模型

```typescript
// 建议新增实体
@Entity()
export class NotificationLog {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  notificationType: Notification;  // MEDIA_PENDING 等

  @Column({ type: 'varchar', length: 50 })
  agentKey: NotificationAgentKey;  // discord/email 等

  @Column({ nullable: true })
  userId?: number;                 // 接收用户（如有）

  @Column({ nullable: true })
  requestId?: number;              // 关联请求（如有）

  @Column({ nullable: true })
  issueId?: number;                // 关联问题（如有）

  @Column({ length: 500 })
  subject: string;

  @Column({ type: 'boolean' })
  success: boolean;

  @Column({ type: 'text', nullable: true })
  errorMessage?: string;

  @Column({ type: 'json', nullable: true })
  errorResponse?: unknown;

  @CreateDateColumn()
  createdAt: Date;

  @Column({ type: 'integer' })
  durationMs: number;              // 发送耗时
}
```

### 17.5 建议的查询接口

| 接口 | 功能 | 权限 |
|------|------|------|
| `GET /api/v1/notifications/history` | 当前用户的发送记录 | 用户 |
| `GET /api/v1/admin/notifications/history` | 全局发送记录 | 管理员 |
| `GET /api/v1/admin/notifications/stats` | 发送统计仪表盘 | 管理员 |
| `GET /api/v1/admin/notifications/failures` | 近期失败记录列表 | 管理员 |

---

## 十八、通知重要级别与静默时段策略

### 18.1 通知重要级别（优先级）

#### 各渠道的优先级实现

通知框架没有**统一的优先级模型**，但部分渠道各自实现了类似概念：

| 渠道 | 优先级机制 | 配置位置 | 取值范围 |
|------|-----------|---------|---------|
| Ntfy | `priority` 字段 | 系统设置 | 1-5，默认 3 |
| Pushover | `sound` 字段（间接触发） | 系统设置 + 用户设置 | 各种铃声 |
| Telegram | `sendSilently` 开关 | 系统设置 + 用户设置 | true/false |
| Gotify | ❌ 无 | - | - |
| Email | ❌ 无 | - | - |
| Discord | ❌ 无 | - | - |
| Slack | ❌ 无 | - | - |
| WebPush | ❌ 无 | - | - |
| Pushbullet | ❌ 无 | - | - |
| Webhook | ❌ 无 | - | - |

#### Ntfy 优先级实现

```typescript
// ntfy.ts:38
const priority = settings.options.priority ?? 3;

const ntfyPayload = {
  topic,
  priority,    // 1=最低, 3=默认, 5=紧急
  title,
  message,
  markdown: true,
};
```

#### Telegram 静默发送实现

**系统级配置**（对所有通知生效）：

```typescript
// settings
options: {
  chatId: string;
  messageThreadId: string;
  sendSilently: boolean;  // 系统级静默开关
}
```

**用户级配置**（用户个人偏好）：

```typescript
// UserSettings
@Column({ nullable: true })
public telegramSendSilently?: boolean;  // 用户级静默开关
```

发送时的实际使用：

```typescript
// telegram.ts:199-206
await axios.post(endpoint, {
  ...notificationPayload,
  chat_id: settings.options.chatId,
  message_thread_id: settings.options.messageThreadId,
  disable_notification: !!settings.options.sendSilently,
});
```

#### Telegram 话题线程（Topic Thread）扩展

Telegram 支持将消息发送到超级群的特定话题线程中，实现**按业务分类到不同话题**：

```typescript
// 系统级话题线程
message_thread_id: settings.options.messageThreadId,

// 用户级话题线程（可进一步路由到用户专属话题）
// payload.notifyUser.settings?.telegramMessageThreadId
```

#### Pushover 声音配置

Pushover 通过不同铃声**间接区分重要性**，支持用户个性化：

- **系统级默认声音**：`settings.notifications.agents.pushover.options.sound`
- **用户级自定义声音**：`user.settings.pushoverSound`

### 18.2 通知类型与重要级别映射（缺失）

**当前问题：** 通知框架完全没有**按业务类型区分重要级别**的机制。

所有通知类型都是**平级**发送：

| 通知类型 | 实际重要程度 | 当前处理 |
|---------|-------------|---------|
| MEDIA_FAILED | 高（需处理） | 与低优先级通知无差异 |
| ISSUE_CREATED | 高（需关注） | 与低优先级通知无差异 |
| MEDIA_PENDING | 中（待审批） | 与低优先级通知无差异 |
| MEDIA_AVAILABLE | 低（信息） | 与低优先级通知无差异 |
| MEDIA_AUTO_REQUESTED | 很低 | 与低优先级通知无差异 |

### 18.3 静默时段策略：完全缺失

| 能力 | 现状 | 说明 |
|------|------|------|
| 用户级静默时段 | ❌ 无 | 用户无法设置夜间免打扰 |
| 系统级静默时段 | ❌ 无 | 无法设置全局维护窗口 |
| 按类型静默 | ❌ 无 | 无法只静默"可用通知"但接收"失败通知" |
| 静默期紧急通知放行 | ❌ 无 | 没有高优先级通知穿透静默的机制 |
| 用户时区支持 | ❌ 无 | 静默时段不按时区计算 |

#### 用户时区问题

用户分布在不同时区时，统一的"22:00-08:00 静默"没有意义。目前 `UserSettings` 中**不存在时区字段**。

### 18.4 Telegram sendSilently 的双层设计分析

Telegram 是唯一同时支持**系统级**和**用户级**静默配置的渠道，但目前实现有缺陷：

```typescript
// 当前：系统级 sendSilently 对所有发送生效
disable_notification: !!settings.options.sendSilently,

// 问题：用户级 telegramSendSilently 完全没有被使用！
// payload.notifyUser.settings?.telegramSendSilently 没有参与逻辑
```

这是一个**代码遗漏**，用户设置的静默偏好没有实际生效。

### 18.5 建议的优先级与静默模型

#### 统一优先级模型

```typescript
enum NotificationPriority {
  LOW = 1,      // 信息类：MEDIA_AVAILABLE, MEDIA_AUTO_REQUESTED
  NORMAL = 2,   // 常规类：MEDIA_APPROVED, MEDIA_PENDING
  HIGH = 3,     // 关注类：ISSUE_CREATED, ISSUE_COMMENT
  URGENT = 4,   // 紧急类：MEDIA_FAILED, ISSUE_RESOLVED
}
```

#### 静默时段配置建议

```typescript
// UserSettings 扩展
interface QuietHours {
  enabled: boolean;
  startHour: number;      // 0-23
  endHour: number;        // 0-23
  timezone: string;       // IANA 时区：'Asia/Shanghai'
  allowedPriorities: NotificationPriority[];  // 静默期允许哪些级别穿透
}
```

#### 静默判定逻辑

```typescript
function shouldSuppressNotification(
  priority: NotificationPriority,
  quietHours: QuietHours,
  userTimezone: string
): boolean {
  if (!quietHours.enabled) return false;
  const now = getCurrentTimeInTimezone(userTimezone);
  const inQuietHours = isHourInRange(now.hour, quietHours.startHour, quietHours.endHour);
  
  // 在静默期内，检查该优先级是否被允许穿透
  return inQuietHours && !quietHours.allowedPriorities.includes(priority);
}
```

---

## 十九、关键代码引用速查

### 19.1 核心框架

| 功能 | 文件 | 行号 |
|------|------|------|
| NotificationManager | `server/lib/notifications/index.ts` | 92-115 |
| 通知类型枚举 | `server/lib/notifications/index.ts` | 6-20 |
| hasNotificationType | `server/lib/notifications/index.ts` | 22-46 |
| shouldSendAdminNotification | `server/lib/notifications/index.ts` | 67-90 |
| Agent 基类与接口 | `server/lib/notifications/agents/agent.ts` | 26-38 |
| NotificationPayload | `server/lib/notifications/agents/agent.ts` | 9-24 |
| Agent 注册 | `server/index.ts` | 132-143 |

### 19.2 配置与存储

| 功能 | 文件 | 行号 |
|------|------|------|
| 系统配置结构 | `server/lib/settings/index.ts` | 220-351 |
| NotificationAgentKey 枚举 | `server/lib/settings/index.ts` | 323-334 |
| 系统设置单例 getSettings | `server/lib/settings/index.ts` | 888-896 |
| 用户设置实体 | `server/entity/UserSettings.ts` | 31-154 |
| 用户 hasNotificationType | `server/entity/UserSettings.ts` | 148-153 |
| notificationTypes transformer | `server/entity/UserSettings.ts` | 88-145 |
| User.settings 关联配置 | `server/entity/User.ts` | 137-142 |
| 用户认证中间件 | `server/middleware/auth.ts` | 9-41 |

### 19.3 各 Agent 实现

| 功能 | 文件 | 行号 |
|------|------|------|
| Discord Agent | `server/lib/notifications/agents/discord.ts` | 78-352 |
| Email Agent | `server/lib/notifications/agents/email.ts` | 62-410 |
| WebPush Agent | `server/lib/notifications/agents/webpush.ts` | 57-401 |
| Webhook Agent | `server/lib/notifications/agents/webhook.ts` | 66-250 |
| Gotify Agent | `server/lib/notifications/agents/gotify.ts` | 19-161 |
| Pushover Agent | `server/lib/notifications/agents/pushover.ts` | 36-353 |
| Telegram Agent | `server/lib/notifications/agents/telegram.ts` | 37-325 |

### 19.4 业务触发点

| 功能 | 文件 | 行号 |
|------|------|------|
| 媒体请求通知触发 | `server/subscriber/MediaRequestSubscriber.ts` | 78-94 |
| 实体生命周期钩子 @AfterInsert | `server/entity/MediaRequest.ts` | 629-655 |
| 实体生命周期钩子 @AfterUpdate | `server/entity/MediaRequest.ts` | 663-730 |
| IssueSubscriber 事件订阅 | `server/subscriber/IssueSubscriber.ts` | 16-136 |
| IssueCommentSubscriber | `server/subscriber/IssueCommentSubscriber.ts` | - |
| MediaSubscriber | `server/subscriber/MediaSubscriber.ts` | - |

### 19.5 重试、降级与去重相关

| 功能 | 文件 | 行号 |
|------|------|------|
| WebPush 永久失败清理 | `server/lib/notifications/agents/webpush.ts` | 249-273 |
| Pushover 图片加载降级 | `server/lib/notifications/agents/pushover.ts` | 64-88 |
| Pushover 同渠道去重 | `server/lib/notifications/agents/pushover.ts` | 244-256 |
| Telegram 同渠道去重 | `server/lib/notifications/agents/telegram.ts` | 220-228 |
| 管理员通知操作者去重 | `server/lib/notifications/index.ts` | 67-90 |
| Manager 并发 fire-and-forget | `server/lib/notifications/index.ts` | 109-113 |
| 状态跳转合并隐式去重 | `server/entity/MediaRequest.ts` | 682-696 |

### 19.6 国际化与多语言

| 功能 | 文件 | 行号 |
|------|------|------|
| i18n 初始化 initI18n | `server/i18n/index.ts` | 12-38 |
| getIntl 工具函数 | `server/i18n/index.ts` | 40-42 |
| defineMessages 工具函数 | `server/i18n/index.ts` | 48-62 |
| 全局通用消息 globalMessages | `server/i18n/globalMessages.ts` | 3-24 |
| Slack Agent 国际化使用 | `server/lib/notifications/agents/slack.ts` | 64-221 |
| Ntfy Agent 国际化使用 | `server/lib/notifications/agents/ntfy.ts` | 31-113 |
| 用户 locale 字段 | `server/entity/UserSettings.ts` | 155-157 |

### 19.7 日志与运维

| 功能 | 文件 | 行号 |
|------|------|------|
| winston logger 配置 | `server/logger.ts` | 1-75 |
| Agent 统一错误日志模式 | `server/lib/notifications/agents/*` | 各 Agent send() 方法 |
| 测试通知特殊处理 | `server/lib/notifications/index.ts` | 37-39 |
| 邮件模板目录 | `server/templates/email/` | - |

### 19.8 Webhook 自定义端点扩展

| 功能 | 文件 | 行号 |
|------|------|------|
| NotificationAgentWebhook 接口 | `server/lib/settings/index.ts` | 290-298 |
| KeyMap 变量映射表 | `server/lib/notifications/agents/webhook.ts` | 17-64 |
| 特殊块变量处理 | `server/lib/notifications/agents/webhook.ts` | 86-122 |
| 模板递归解析 parseKeys | `server/lib/notifications/agents/webhook.ts` | 80-144 |
| URL 动态变量替换 | `server/lib/notifications/agents/webhook.ts` | 186-202 |
| 三层请求头配置 | `server/lib/notifications/agents/webhook.ts` | 204-229 |
| JSON 模板 Base64 编解码 | `server/routes/settings/notifications.ts` | 277-325 |

### 19.9 通知设置路由 API

| 功能 | 文件 | 行号 |
|------|------|------|
| 通知路由总入口 | `server/routes/settings/notifications.ts` | 19-433 |
| sendTestNotification 测试函数 | `server/routes/settings/notifications.ts` | 26-36 |
| Webhook GET 配置（自动解码） | `server/routes/settings/notifications.ts` | 276-298 |
| Webhook POST 配置（自动编码） | `server/routes/settings/notifications.ts` | 300-325 |
| 各渠道 GET/POST/TEST 接口 | `server/routes/settings/notifications.ts` | 38-433 |

### 19.10 优先级与静默相关

| 功能 | 文件 | 行号 |
|------|------|------|
| Ntfy priority 配置 | `server/lib/settings/index.ts` | 318 / 563 |
| Telegram sendSilently 配置（系统级） | `server/lib/settings/index.ts` | 271 / 509 |
| Telegram messageThreadId 配置（系统级） | `server/lib/settings/index.ts` | 270 / 508 |
| Pushover sound 配置（系统级） | `server/lib/settings/index.ts` | 286 / 527 |
| Gotify priority 配置 | `server/lib/settings/index.ts` | 304 / 552 |
| 用户 pushoverSound 字段 | `server/entity/UserSettings.ts` | 71 |
| 用户 telegramSendSilently 字段 | `server/entity/UserSettings.ts` | 80 |
| 用户 telegramMessageThreadId 字段 | `server/entity/UserSettings.ts` | 77 |
| Telegram 发送时使用 sendSilently | `server/lib/notifications/agents/telegram.ts` | 205 |
| Telegram 发送时使用 messageThreadId | `server/lib/notifications/agents/telegram.ts` | 204 |
| Ntfy 发送时使用 priority | `server/lib/notifications/agents/ntfy.ts` | 38 / 100 |
