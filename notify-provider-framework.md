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

## 十三、关键代码引用速查

### 13.1 核心框架

| 功能 | 文件 | 行号 |
|------|------|------|
| NotificationManager | `server/lib/notifications/index.ts` | 92-115 |
| 通知类型枚举 | `server/lib/notifications/index.ts` | 6-20 |
| hasNotificationType | `server/lib/notifications/index.ts` | 22-46 |
| shouldSendAdminNotification | `server/lib/notifications/index.ts` | 67-90 |
| Agent 基类与接口 | `server/lib/notifications/agents/agent.ts` | 26-38 |
| NotificationPayload | `server/lib/notifications/agents/agent.ts` | 9-24 |
| Agent 注册 | `server/index.ts` | 132-143 |

### 13.2 配置与存储

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

### 13.3 各 Agent 实现

| 功能 | 文件 | 行号 |
|------|------|------|
| Discord Agent | `server/lib/notifications/agents/discord.ts` | 78-352 |
| Email Agent | `server/lib/notifications/agents/email.ts` | 62-410 |
| WebPush Agent | `server/lib/notifications/agents/webpush.ts` | 57-401 |
| Webhook Agent | `server/lib/notifications/agents/webhook.ts` | 66-250 |
| Gotify Agent | `server/lib/notifications/agents/gotify.ts` | 19-161 |
| Pushover Agent | `server/lib/notifications/agents/pushover.ts` | 36-353 |
| Telegram Agent | `server/lib/notifications/agents/telegram.ts` | 37-325 |

### 13.4 业务触发点

| 功能 | 文件 | 行号 |
|------|------|------|
| 媒体请求通知触发 | `server/subscriber/MediaRequestSubscriber.ts` | 78-94 |

### 13.5 重试、降级与去重相关

| 功能 | 文件 | 行号 |
|------|------|------|
| WebPush 永久失败清理 | `server/lib/notifications/agents/webpush.ts` | 249-273 |
| Pushover 图片加载降级 | `server/lib/notifications/agents/pushover.ts` | 64-88 |
| Pushover 同渠道去重 | `server/lib/notifications/agents/pushover.ts` | 244-256 |
| Telegram 同渠道去重 | `server/lib/notifications/agents/telegram.ts` | 220-228 |
| 管理员通知操作者去重 | `server/lib/notifications/index.ts` | 67-90 |
| Manager 并发 fire-and-forget | `server/lib/notifications/index.ts` | 109-113 |
