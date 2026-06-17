# Jellyseerr 点播请求全链路代码分析

## 一、整体流程概述

用户点播影视后的完整链路如下：

```
用户前端点播
    ↓
[API 层] POST /api/v1/request
    ↓
[鉴权层] 会话/API Key 认证 + 权限校验
    ↓
[业务层] MediaRequest.request() - 请求创建
    ├─ 权限检查 (REQUEST / REQUEST_4K 等)
    ├─ 配额检查 (movie/tv quota)
    ├─ 重复请求检查
    ├─ 黑名单检查
    ├─ 应用覆盖规则 (OverrideRule)
    └─ 保存 Media + MediaRequest 实体
    ↓
[事件订阅] MediaRequestSubscriber
    ├─ AfterInsert / AfterUpdate 触发
    ├─ sendToRadarr() / sendToSonarr() - 异步发送到媒体服务器
    └─ updateParentStatus() - 更新父级 Media 状态
    ↓
[通知层] NotificationManager
    ├─ AfterInsert 触发 notifyNewRequest()
    ├─ AfterUpdate 触发 notifyApprovedOrDeclined()
    └─ 遍历所有通知 Agent (Discord/Webhook/Email 等) 发送通知
    ↓
[定时任务层] 周期性扫描
    ├─ DownloadTracker - 每分钟拉取下载队列
    ├─ *arr Scanner - 每日扫描 Radarr/Sonarr 库
    ├─ Plex/Jellyfin Scanner - 扫描媒体服务器库
    └─ AvailabilitySync - 定期检查媒体可用性
    ↓
[状态回写] 扫描器更新本地 Media 状态
    └─ BaseScanner.processMovie() / processShow()
        ├─ AsyncLock 防止并发冲突
        ├─ 更新 Media.status / Media.status4k
        ├─ 更新 Season 状态
        └─ 触发 MEDIA_AVAILABLE 通知
```

---

## 二、鉴权机制实现

### 2.1 认证中间件

**文件**: `server/middleware/auth.ts:9-58`

支持两种认证方式：

1. **API Key 认证** (X-API-Key 头)
   ```typescript
   if (req.header('X-API-Key') === settings.main.apiKey) {
     // 可选 X-API-User 头指定操作用户
     const userId = req.header('X-API-User') ? Number(...) : 1;
     user = await userRepository.findOne({ where: { id: userId } });
   }
   ```

2. **会话认证** (Session Cookie)
   ```typescript
   if (req.session?.userId) {
     user = await userRepository.findOne({ where: { id: req.session.userId } });
   }
   ```

### 2.2 权限系统

**文件**: `server/lib/permissions.ts:1-74`

采用**位掩码权限**设计，每个权限是 2 的幂次方，支持组合权限检查：

```typescript
export enum Permission {
  NONE = 0,
  ADMIN = 2,
  MANAGE_REQUESTS = 16,
  REQUEST = 32,
  AUTO_APPROVE = 128,
  REQUEST_4K = 1024,
  REQUEST_4K_MOVIE = 2048,
  REQUEST_4K_TV = 4096,
  REQUEST_ADVANCED = 8192,
  // ... 其他权限
}
```

权限检查函数支持 `AND` / `OR` 模式：
```typescript
hasPermission(
  permissions: Permission | Permission[],
  value: number,
  options: { type: 'and' | 'or' } = { type: 'and' }
): boolean
```

**关键权限检查点**:
- `server/entity/MediaRequest.ts:80-112` - 请求前检查 `REQUEST` / `REQUEST_4K` 权限
- `server/entity/MediaRequest.ts:354-367` - 自动审批检查 `AUTO_APPROVE` 权限
- `server/routes/request.ts:668` - 审批操作检查 `MANAGE_REQUESTS` 权限

### 2.3 配额系统

**文件**: `server/entity/MediaRequest.ts:114-120`

在请求创建前检查用户配额：
```typescript
const quotas = await requestUser.getQuota();
if (requestBody.mediaType === MediaType.MOVIE && quotas.movie.restricted) {
  throw new QuotaRestrictedError('Movie Quota exceeded.');
}
```

---

## 三、节流与队列机制

### 3.1 扫描器节流

**文件**: `server/lib/scanners/baseScanner.ts:687-725`

采用**分批处理 + 间隔等待**实现节流：

```typescript
protected async loop(
  processFn: (item: T) => Promise<void>,
  {
    start = 0,
    end = this.bundleSize,  // 默认 20 条/批
    sessionId,
  } = {}
): Promise<void> {
  // 处理当前批次
  await this.processItems(processFn, slicedItems);
  
  // 等待后递归处理下一批 (默认间隔 4 秒)
  await new Promise<void>((resolve) =>
    setTimeout(() => {
      this.loop(processFn, {
        start: start + this.bundleSize,
        end: end + this.bundleSize,
        sessionId,
      }).then(() => resolve());
    }, this.updateRate)  // 默认 4000ms
  );
}
```

### 3.2 API 速率限制

**文件**: `server/api/externalapi.ts:45-50`

使用 `axios-rate-limit` 对外部 API 调用进行限流：
```typescript
if (options.rateLimit) {
  this.axios = rateLimit(this.axios, {
    maxRequests: options.rateLimit.maxRequests,
    maxRPS: options.rateLimit.maxRPS,
  });
}
```

#### 3.2.1 Rate-Limit 分布式部署分析

**当前实现的局限性**:
- `axios-rate-limit` 基于**内存计数器**实现，不支持分布式部署
- 多实例部署时，每个实例独立计数，无法共享限流状态
- 可能导致总请求量超过外部 API 的实际限制

**技术原理**:
`axios-rate-limit` 内部维护一个请求队列和时间窗口计数器：
```typescript
// axios-rate-limit 内部原理（示意）
const requestQueue = [];
let requestCount = 0;
let windowStart = Date.now();

// 新请求进入时检查窗口
if (Date.now() - windowStart > 1000) {
  windowStart = Date.now();
  requestCount = 0;
}

if (requestCount < maxRPS) {
  requestCount++;
  return axios(config);  // 立即执行
} else {
  return new Promise(resolve => {
    requestQueue.push({ config, resolve });  // 排队等待
  });
}
```

**分布式部署的潜在方案**:
1. **集中式限流**: 使用 Redis 实现分布式令牌桶或漏桶算法
2. **实例分流**: 通过负载均衡按 API Key 或用户 ID 做会话粘滞，确保同一外部服务的请求路由到同一实例
3. **配额分摊**: 将总配额除以实例数，每个实例独立限制
4. **降级策略**: 当检测到 429 Too Many Requests 时，触发全局熔断并在所有实例间同步

**当前代码中的规避措施**:
- `server/lib/cache.ts` 的多级缓存设计减少重复请求
- `getRolling()` 方法的后台刷新策略分散请求峰值
- 扫描器的 `updateRate` 间隔配置避免突发流量

#### 3.2.2 Rate-Limit 多节点 Redis 同步改造方案

**现状分析**:
- TypeORM 的 peerDependencies 中声明了 `ioredis ^5.0.4` 和 `redis ^3.1.1`，但 Jellyseerr 核心代码**未实际使用 Redis**
- `node-cache`（`server/lib/cache.ts`）和 `axios-rate-limit` 均为纯内存实现

**Redis 分布式令牌桶改造架构**:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Node A      │     │  Node B      │     │  Node N      │
│ (axios-     │     │ (axios-     │     │ (axios-     │
│  rate-      │     │  rate-      │     │  rate-      │
│  limit)     │     │  limit)     │     │  limit)     │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            ▼
                    ┌───────────────┐
                    │    Redis      │
                    │  ┌─────────┐  │
                    │  │令牌桶键 │  │
                    │  │ - radarr:│  │
                    │  │  tokens  │  │
                    │  │ - sonarr:│  │
                    │  │  tokens  │  │
                    │  │ - tmdb:  │  │
                    │  │  tokens  │  │
                    │  └─────────┘  │
                    └───────────────┘
```

**Redis Lua 脚本实现原子令牌桶**:
```typescript
// 建议的分布式限流脚本（需改造 externalapi.ts）
const acquireTokenScript = `
  local key = KEYS[1]
  local capacity = tonumber(ARGV[1])
  local rate = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])

  local state = redis.call('HMGET', key, 'tokens', 'timestamp')
  local tokens = tonumber(state[1]) or capacity
  local timestamp = tonumber(state[2]) or now

  local elapsed = now - timestamp
  local refilled = math.floor(elapsed * rate / 1000)
  tokens = math.min(capacity, tokens + refilled)
  timestamp = now

  if tokens > 0 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'timestamp', timestamp)
    redis.call('PEXPIRE', key, 60000)  -- 1 分钟自动过期
    return 1
  end
  return 0
`;

// 键命名空间：rate_limit:tmdb、rate_limit:radarr:server1
// 每个 API Key 独立限流，避免互相影响
```

**多节点同步策略**:
1. **读多写少场景**: 本地缓存 50ms，每 50ms 批量同步 Redis（减少网络开销）
2. **写入失败降级**: Redis 不可用时自动 fallback 到内存限流，日志标记 `fallback=memory`
3. **预热加载**: 节点启动时从 Redis 加载当前令牌数，避免冷启动过载

#### 3.2.3 Rate-Limit 故障切换：Sentinel 还是 Client 重试？

**源码现状分析**:
- **无 Redis Sentinel**：Jellyseerr 核心代码未集成任何 Redis 客户端，`TypeORM` 的 `peerDependencies` 声明 `ioredis` 但**从未实际调用**
- **无 `axios-retry` 插件**：全项目 grep 无 `axios-retry` 或 `retry-axios` 引用
- **故障切换完全由 Client 侧错误捕获驱动**

**代理层故障切换示例** (`server/utils/customProxyAgent.ts:104-121`):
```typescript
} catch (e) {
  logger.error('Failed to connect to the proxy: ' + e.message, {
    label: 'Proxy',
  });
  setGlobalDispatcher(defaultAgent);  // 🔴 故障切换：fallback 到默认 dispatcher
  return;
}

try {
  await axios.head('https://www.google.com');  // 连通性探测
  logger.debug('HTTP(S) proxy connected successfully', { label: 'Proxy' });
} catch (e) {
  logger.error('Failed to connect to the proxy: ' + e.message + ': ' + e.cause, {
    label: 'Proxy',
  });
  setGlobalDispatcher(defaultAgent);  // 🔴 故障切换：探测失败也 fallback
}
```

**外部 API 调用错误处理示例** (`server/api/externalapi.ts` + `server/subscriber/MediaRequestSubscriber.ts:789`):
```typescript
// sendToRadarr 外层 catch（P3 连接错误回滚）
} catch (e) {
  logger.error('Something went wrong while adding movie to Radarr.', {
    label: 'Media Request',
    errorMessage: e.message,
    fingerprint: ['rollback-p3-connect-fail', hostname, `error:${e.code}`],
  });
  // 纯内存降级：标记 FAILED 状态，不做 Redis 故障转移
  if (entity.status !== MediaRequestStatus.FAILED) {
    entity.status = MediaRequestStatus.FAILED;
    await requestRepository.save(entity);
  }
}
```

**故障切换决策树**：
```
请求失败 → 检查错误类型
    │
    ├─ 429 Too Many Requests → axios-rate-limit 队列化等待（内存内）
    │                          → 不切换，仅延时重试
    │
    ├─ ECONNREFUSED / ETIMEDOUT → 标记 FAILED + fallback 到内存限流
    │                          → 5 分钟后后台任务自动重试（P4）
    │                          → ❌ 不做实例级故障转移
    │
    ├─ 4xx 业务错误（无效参数）→ 标记 FAILED + 通知用户
    │                          → 不重试
    │
    └─ 5xx 服务端错误 → 指数退避重试 3 次（由调用方显式实现，非全局）
                    → 全部失败后标记 FAILED
```

**Redis Sentinel 改造建议**（如需多节点高可用）:
```typescript
// 建议的 Sentinel 配置
const redis = new Redis({
  sentinels: [
    { host: 'sentinel-1', port: 26379 },
    { host: 'sentinel-2', port: 26379 },
    { host: 'sentinel-3', port: 26379 },
  ],
  name: 'jellyseerr-master',
  enableReadyCheck: true,
  maxRetriesPerRequest: 3,
  retryDelayOnFailover: 100,  // Sentinel 故障转移期间延迟 100ms 重试
});

// 令牌桶 Redis key 失效策略
// 主节点宕机 → Sentinel 选举新主（~10s）→ 客户端自动重连
// 期间请求 fallback 到内存限流，日志标记 `redisRole: ${redis.status}`
```

**Client 重试 vs Sentinel 对比表**:
| 维度 | Client 重试（现状） | Redis Sentinel（改造） |
|------|-------------------|---------------------|
| 故障转移时间 | 取决于重试策略（~30s） | Sentinel 选举 + DNS 更新（~10s） |
| 数据一致性 | 内存计数，多实例不一致 | 强一致，全实例共享 |
| 实现复杂度 | 低（已有 catch 逻辑） | 中高（Sentinel 集群运维） |
| 单点故障风险 | 无（每个实例独立） | 有（Sentinel 集群需≥3节点） |
| 适用场景 | 单实例部署 | 多实例高可用部署 |

---

### 3.3 AsyncLock 死锁检测机制

**文件**: `server/utils/asyncLock.ts:1-54`

防止同一媒体的并发写入冲突，基于 EventEmitter 实现：

```typescript
public dispatch = async (
  key: string | number,    // 通常是 tmdbId
  callback: () => Promise<void>
) => {
  const skey = String(key);
  await this.acquire(skey);  // 获取锁，等待其他操作完成
  try {
    await callback();        // 执行业务逻辑
  } finally {
    this.release(skey);      // 释放锁
  }
};
```

#### 3.3.1 死锁风险分析

**死锁形成条件**:
1. **互斥持有**: `locked[key] = true` 标记持有锁
2. **循环等待**: 嵌套调用 `dispatch()` 且 key 顺序相反时可能发生
3. **不可抢占**: 锁只能由持有者释放
4. **持有并等待**: 持有锁 A 时等待锁 B

**死锁预防设计**:

1. **无界监听器预警**:
   ```typescript
   constructor() {
     this.ee.setMaxListeners(0);  // 移除默认 10 个监听器限制
   }
   ```
   > **风险**: 虽然移除了限制，但大量等待者（>1000）可能暗示潜在死锁。实际部署中应监控 `ee.listenerCount(key)`，超过阈值时告警。

2. **非递归锁设计**:
   - 同一 key 不可重入，会导致永久阻塞
   - 设计要求：`dispatch` 回调内严禁再次调用同一 key 的 `dispatch`

3. **`setImmediate` 释放策略**:
   ```typescript
   private release = (key: string): void => {
     delete this.locked[key];
     setImmediate(() => this.ee.emit(key));  // 异步释放，避免同步递归
   };
   ```
   - 将释放操作放到事件循环的下一个 tick
   - 防止 `dispatch` 嵌套调用导致的同步死锁

**死锁检测机制（需运维侧补充）**:
```typescript
// 建议增加的监控逻辑
public getDeadlockCandidates(timeoutMs = 30000): string[] {
  const now = Date.now();
  const candidates: string[] = [];
  for (const [key, locked] of Object.entries(this.locked)) {
    if (locked && (this.lockAcquiredAt[key] || 0) < now - timeoutMs) {
      candidates.push(key);
    }
  }
  return candidates;
}
```

**典型死锁场景示例**:
```typescript
// ❌ 错误：嵌套调用同一 key 会死锁
asyncLock.dispatch(123, async () => {
  await asyncLock.dispatch(123, async () => { /* ... */ });  // 永久等待
});

// ✅ 正确：避免嵌套，或使用不同的锁粒度
```

#### 3.3.2 AsyncLock 跨进程死锁识别

**现状分析**:
当前 `AsyncLock` 基于进程内 `EventEmitter`，多实例部署时：
- 实例 A 持有锁 key=123
- 实例 B 无法感知，也会尝试获取锁
- 导致两个实例同时操作同一 tmdbId，触发 SQLite `UNIQUE constraint failed` 错误

**Redis Redlock 分布式锁 + 死锁识别改造方案**:

```typescript
// 建议的跨进程锁识别（需改造 asyncLock.ts）
class DistributedAsyncLock {
  private processLocks: Map<string, { pid: string; acquiredAt: number; stack: string }> = new Map();
  private readonly REDIS_KEY_PREFIX = 'jellyseerr:lock:';
  private readonly DEADLOCK_THRESHOLD = 30000;  // 30 秒超时

  public dispatch = async (key: string | number, callback: () => Promise<void>) => {
    const skey = String(key);
    const lockValue = `${process.pid}:${this.instanceId}:${Date.now()}`;

    // 1. 先检查跨进程死锁：查询 Redis 是否有长期未释放的锁
    const existingLock = await this.redis.get(this.REDIS_KEY_PREFIX + skey);
    if (existingLock) {
      const [remotePid, remoteInstanceId, acquiredTimestamp] = existingLock.split(':');
      const heldMs = Date.now() - Number(acquiredTimestamp);

      if (heldMs > this.DEADLOCK_THRESHOLD) {
        // 死锁检测：超过 30 秒未释放，强制解锁
        const released = await this.redis.del(this.REDIS_KEY_PREFIX + skey);
        logger.warn('Deadlock detected and force-released', {
          label: 'AsyncLock',
          lockKey: skey,
          heldBy: { remotePid, remoteInstanceId, heldMs },
          released: released > 0,
          fingerprint: `deadlock:${skey}:${acquiredTimestamp}`,  // Sentry 幂等键
        });
      } else {
        // 正常等待：订阅 Redis Pub/Sub 释放通知
        await this.waitForRedisLock(skey);
      }
    }

    // 2. 获取分布式锁（SET NX PX 原子操作）
    const acquired = await this.redis.set(
      this.REDIS_KEY_PREFIX + skey,
      lockValue,
      'PX',
      this.DEADLOCK_THRESHOLD,  // 自动过期兜底
      'NX'
    );

    if (!acquired) {
      throw new Error(`Lock contention detected for key ${skey}`);
    }

    // 3. 记录进程内堆栈，用于本地死锁诊断
    this.processLocks.set(skey, {
      pid: process.pid.toString(),
      acquiredAt: Date.now(),
      stack: new Error().stack || '',  // 捕获获取锁时的调用栈
    });

    try {
      await callback();
    } finally {
      // 4. 释放锁（先比对锁持有者，避免误删）
      const currentHolder = await this.redis.get(this.REDIS_KEY_PREFIX + skey);
      if (currentHolder === lockValue) {
        await this.redis.del(this.REDIS_KEY_PREFIX + skey);
      }
      this.processLocks.delete(skey);

      // 5. Pub/Sub 通知等待者
      await this.redis.publish(`${this.REDIS_KEY_PREFIX}release:${skey}`, lockValue);
    }
  };

  // 跨进程死锁诊断 API
  public getDeadlockReport(): DeadlockReport[] {
    const now = Date.now();
    const reports: DeadlockReport[] = [];

    for (const [key, info] of this.processLocks.entries()) {
      const heldMs = now - info.acquiredAt;
      if (heldMs > this.DEADLOCK_THRESHOLD * 0.8) {  // 80% 阈值时预警
        reports.push({
          key,
          heldMs,
          warning: heldMs > this.DEADLOCK_THRESHOLD ? 'DEADLOCK_SUSPECTED' : 'APPROACHING_TIMEOUT',
          stackTrace: info.stack,
          pid: info.pid,
        });
      }
    }
    return reports;
  }
}
```

**死锁识别键 (Sentry Fingerprint) 映射表**:
| 场景 | Fingerprint 模板 | Sentry 级别 |
|------|-----------------|-------------|
| 本地持有超 80% 阈值 | `['asyncLock-warning', key, instanceId]` | warning |
| Redis 锁超 30s 强制释放 | `['asyncLock-deadlock-force-release', key, acquiredTimestamp]` | error |
| 获取锁竞争失败超过 5 次/分钟 | `['asyncLock-contention-high', key]` | warning |
| Redlock 多节点不一致 | `['asyncLock-redlock-inconsistency', key]` | fatal |

#### 3.3.3 AsyncLock 跨时区时钟漂移识别

**问题背景**：
多实例部署在不同时区或 NTP 同步不佳的服务器上时，`Date.now()` 的时钟差异会导致分布式锁 TTL 计算错误：

- 实例 A（时区 UTC+8）获取锁，设置 `acquiredAt = 1717200000000`（北京时间 08:00）
- 实例 B（时区 UTC-5）的系统时间可能是 `1717174800000`（纽约时间 19:00，比实际慢 7 小时）
- 实例 B 计算 `heldMs = 1717174800000 - 1717200000000 = -25200000`（负数！）
- 结果：**实例 B 误认为锁已过期 25200 秒，强制释放还在使用的锁**，导致数据冲突

**源码中的隐式保护** (`server/utils/asyncLock.ts` + `server/routes/discover.ts:359-361`):
```typescript
// 代码中仅使用 Date.now()，未做任何时区校正
const heldMs = Date.now() - Number(acquiredTimestamp);

// discover.ts 中有使用 getTimezoneOffset() 校正日期
const offset = now.getTimezoneOffset();
const date = new Date(now.getTime() - offset * 60 * 1000)
  .toISOString()
  .split('T')[0];  // 仅用于日期显示，不影响锁逻辑
```

**跨时区时钟漂移检测算法（建议补充）**:
```typescript
class ClockDriftDetector {
  private readonly DRIFT_THRESHOLD_MS = 5000;  // 可容忍 5 秒时钟漂移
  private readonly NTP_SERVERS = [
    'pool.ntp.org', 'time.nist.gov', 'time.google.com'
  ];

  // 启动时校准本地时钟偏差
  public async calibrateClockOffset(): Promise<number> {
    const offsets: number[] = [];

    for (const ntpServer of this.NTP_SERVERS) {
      try {
        const t0 = Date.now();
        const ntpTime = await this.queryNtpTime(ntpServer);
        const t1 = Date.now();
        const networkLatency = (t1 - t0) / 2;
        const offset = ntpTime - (t0 + networkLatency);
        offsets.push(offset);
      } catch (e) {
        // 跳过失败的 NTP 服务器
      }
    }

    if (offsets.length === 0) {
      logger.warn('NTP calibration failed, using local clock', {
        label: 'AsyncLock',
        fingerprint: ['clock-drift-ntp-failed'],
      });
      return 0;
    }

    // 取中位数作为最终偏移
    offsets.sort((a, b) => a - b);
    const medianOffset = offsets[Math.floor(offsets.length / 2)];

    if (Math.abs(medianOffset) > this.DRIFT_THRESHOLD_MS) {
      logger.warn('Clock drift detected', {
        label: 'AsyncLock',
        driftMs: medianOffset,
        action: Math.abs(medianOffset) > 30000 ? 'BLOCK_STARTUP' : 'WARNING_ONLY',
        fingerprint: ['clock-drift-detected', String(medianOffset)],
      });

      // 漂移超过 30 秒，阻塞启动（防止数据损坏）
      if (Math.abs(medianOffset) > 30000) {
        throw new Error(`Clock drift ${medianOffset}ms exceeds safety threshold 30000ms`);
      }
    }

    this.clockOffset = medianOffset;
    return medianOffset;
  }

  // 使用校正后的时间计算锁持有时间
  private getCorrectedNow(): number {
    return Date.now() + (this.clockOffset || 0);
  }

  // 时钟漂移检测：每次获取锁时检查
  private detectDrift(key: string, remoteTimestamp: number): DriftStatus {
    const correctedNow = this.getCorrectedNow();
    const drift = correctedNow - remoteTimestamp;

    if (drift < -this.DRIFT_THRESHOLD_MS) {
      // 本地时钟比 Redis 慢 → 可能误判锁已过期
      return { status: 'NEGATIVE_DRIFT', driftMs: drift, safe: false };
    }
    if (drift > this.DRIFT_THRESHOLD_MS * 2) {
      // 本地时钟比 Redis 快 → 可能导致锁提前释放
      return { status: 'POSITIVE_DRIFT', driftMs: drift, safe: false };
    }
    return { status: 'OK', driftMs: drift, safe: true };
  }

  // 漂移保护的锁获取逻辑
  public async acquireDistributedLock(key: string): Promise<boolean> {
    const lockValue = `${this.instanceId}:${this.getCorrectedNow()}`;
    const driftCheck = this.detectDrift(key, Number(lockValue.split(':')[1]));

    if (!driftCheck.safe) {
      logger.warn('Clock drift detected, refusing to acquire lock', {
        label: 'AsyncLock',
        lockKey: key,
        driftMs: driftCheck.driftMs,
        driftStatus: driftCheck.status,
        fingerprint: ['clock-drift-lock-refused', key, driftCheck.status],
      });
      return false;  // 时钟漂移时拒绝获取锁，避免数据损坏
    }

    // 正常获取锁...
    return true;
  }
}
```

**锁 TTL 自适应调整（基于检测到的漂移）**:
```typescript
// 根据时钟漂移动态调整锁超时阈值
private getAdaptiveDeadlockThreshold(baseThreshold: number): number {
  const driftAbs = Math.abs(this.clockOffset || 0);
  // 漂移越大，锁超时阈值越大（防止误释放）
  // 但最大不超过 baseThreshold 的 3 倍（避免死锁长时间不释放）
  return Math.min(baseThreshold + driftAbs * 2, baseThreshold * 3);
}
```

**NTP 时钟同步健康监控 API**:
```
GET /api/v1/settings/status/clock-drift
Response:
{
  "localClockOffsetMs": -1234,
  "driftStatus": "OK",
  "thresholdMs": 5000,
  "adaptiveLockThresholdMs": 32468,
  "lastCalibrationAt": "2026-06-17T08:00:00.000Z",
  "calibrationSource": "pool.ntp.org"
}
```

**跨时区部署配置建议**:
1. 所有实例强制配置 UTC 时区，避免夏令时切换
2. 配置本地 NTP 服务器（`ntpd -qg` 启动时强制校时）
3. 监控 `ntpq -p` offset 值，超过 5s 告警
4. Redis 服务器与应用实例使用同一 NTP 源

### 3.4 异步锁 (AsyncLock) 使用场景

**文件**: `server/utils/asyncLock.ts:1-54`

防止同一媒体的并发写入冲突，基于 EventEmitter 实现：

```typescript
public dispatch = async (
  key: string | number,    // 通常是 tmdbId
  callback: () => Promise<void>
) => {
  const skey = String(key);
  await this.acquire(skey);  // 获取锁，等待其他操作完成
  try {
    await callback();        // 执行业务逻辑
  } finally {
    this.release(skey);      // 释放锁
  }
};
```

**使用场景**:
- `BaseScanner.processMovie()` - 电影处理时按 tmdbId 加锁
- `BaseScanner.processShow()` - 剧集处理时按 tmdbId 加锁

### 3.5 缓存机制

**文件**: `server/lib/cache.ts:1-91`

使用 `node-cache` 为不同外部 API 配置独立缓存：

| 缓存 ID | TTL | 用途 |
|---------|-----|------|
| tmdb | 21600s (6h) | TMDB API 数据 |
| radarr | 300s (5min) | Radarr API 数据 |
| sonarr | 300s (5min) | Sonarr API 数据 |
| plexguid | 604800s (7d) | Plex GUID 映射 |

**滚动缓存策略** (`externalapi.ts:104-137`):
- 缓存过期前 10 秒触发后台刷新
- 立即返回旧数据，不阻塞用户请求

---

## 四、媒体服务器同步逻辑

### 4.1 请求发送到 Radarr/Sonarr

**文件**: `server/subscriber/MediaRequestSubscriber.ts:183-818`

触发时机：`AfterInsert` 和 `AfterUpdate` 事件

```typescript
public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
  await this.sendToRadarr(event.entity as MediaRequest);
  await this.sendToSonarr(event.entity as MediaRequest);
  await this.updateParentStatus(event.entity as MediaRequest);
}
```

**异步发送设计** (关键在 `sendToRadarr` 方法):

```typescript
// 异步执行，不阻塞 UI
radarr
  .addMovie(radarrMovieOptions)
  .then(async (radarrMovie) => {
    // 成功：回写外部服务 ID
    media.externalServiceId = radarrMovie.id;
    media.externalServiceSlug = radarrMovie.titleSlug;
    media.serviceId = radarrSettings?.id;
    await mediaRepository.save(media);
  })
  .catch(async () => {
    // 失败：标记为 FAILED
    entity.status = MediaRequestStatus.FAILED;
    await requestRepository.save(entity);
    // 发送失败通知
    MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
  })
  .finally(() => {
    // 清除相关缓存
    radarr.clearCache({ tmdbId: movie.id, externalId: ... });
  });
```

**服务选择逻辑**:
1. 优先使用请求指定的 `serverId`
2. 否则查找 `isDefault && is4k === entity.is4k` 的默认服务器
3. 支持 `rootFolder`、`profileId`、`tags` 的请求级覆盖

### 4.2 下载进度追踪

**文件**: `server/lib/downloadtracker.ts:1-227`

定时任务 (`download-sync`) 每分钟执行一次：

```typescript
public updateDownloads() {
  this.updateRadarrDownloads();
  this.updateSonarrDownloads();
}

private async updateRadarrDownloads() {
  // 调用 Radarr API 刷新监控的下载
  await radarr.refreshMonitoredDownloads();
  // 获取队列信息
  const queueItems = await radarr.getQueue();
  // 缓存到内存
  this.radarrServers[server.id] = queueItems.map(...);
}
```

### 4.3 媒体库扫描

**文件**: `server/lib/scanners/baseScanner.ts:56-755`

三种扫描模式：
1. **全量扫描** (`*-full-scan`) - 每日执行，扫描整个媒体库
2. **最近扫描** (`*-recently-added-scan`) - 每 5 分钟执行，只扫新增内容
3. **Radarr/Sonarr 扫描** (`radarr-scan` / `sonarr-scan`) - 每日执行

处理逻辑 (`processMovie` 示例):
```typescript
protected async processMovie(
  tmdbId: number,
  options: ProcessOptions
): Promise<void> {
  await this.asyncLock.dispatch(tmdbId, async () => {
    const existing = await this.getExisting(tmdbId, MediaType.MOVIE);
    
    if (existing) {
      // 更新现有媒体状态
      if (!processing && hasFile) {
        existing.status = MediaStatus.AVAILABLE;
      }
      // ... 更新其他字段
      await mediaRepository.save(existing);
    } else {
      // 创建新媒体记录
      const newMedia = new Media({ ... });
      await mediaRepository.save(newMedia);
    }
  });
}
```

---

## 五、外部 API 通知机制

### 5.1 通知管理器

**文件**: `server/lib/notifications/index.ts:92-115`

```typescript
class NotificationManager {
  private activeAgents: NotificationAgent[] = [];

  public sendNotification(type: Notification, payload: NotificationPayload): void {
    this.activeAgents.forEach((agent) => {
      if (agent.shouldSend()) {
        agent.send(type, payload);  // 异步发送，不等待结果
      }
    });
  }
}
```

### 5.2 通知类型

**文件**: `server/lib/notifications/index.ts:6-20`

```typescript
export enum Notification {
  NONE = 0,
  MEDIA_PENDING = 2,           // 新请求待审批
  MEDIA_APPROVED = 4,          // 请求已批准
  MEDIA_AVAILABLE = 8,         // 媒体已可用
  MEDIA_FAILED = 16,           // 请求失败
  MEDIA_DECLINED = 64,         // 请求被拒绝
  MEDIA_AUTO_APPROVED = 128,   // 自动批准
  MEDIA_AUTO_REQUESTED = 4096, // 自动请求
}
```

### 5.3 通知触发点

| 触发时机 | 位置 | 通知类型 |
|---------|------|---------|
| 请求创建后 | `MediaRequest.afterInsert()`:629-655 | `MEDIA_PENDING` |
| 自动批准 | `MediaRequest.autoapprovalNotification()`:722-727 | `MEDIA_AUTO_APPROVED` |
| 状态更新 | `MediaRequest.afterUpdate()`:663-720 | `MEDIA_APPROVED` / `MEDIA_DECLINED` |
| 扫描发现可用 | `AvailabilitySync` + `BaseScanner` | `MEDIA_AVAILABLE` |
| 发送到 *arr 失败 | `MediaRequestSubscriber.sendToRadarr()`:398-432 | `MEDIA_FAILED` |

### 5.4 通知 Agent 实现

以 Webhook Agent 为例 (`server/lib/notifications/agents/webhook.ts`):

```typescript
public async send(
  type: Notification,
  payload: NotificationPayload
): Promise<boolean> {
  // 检查是否启用该通知类型
  if (!hasNotificationType(type, settings.types ?? 0)) {
    return true;
  }

  // 构建 payload，支持模板变量替换
  const body = this.buildPayload(type, payload);
  
  // 发送请求
  await axios.post(webhookUrl, body, { headers });
  
  return true;
}
```

**支持的通知 Agent**:
- Discord, Email, Gotify, Ntfy, Pushbullet, Pushover, Slack, Telegram, Webhook, WebPush

### 5.5 10 通知 Agent 扩展机制

#### 5.5.1 架构设计

**文件**: `server/lib/notifications/agents/agent.ts`

采用**抽象基类 + 接口**的双重抽象设计：

```typescript
// 接口定义契约
export interface NotificationAgent {
  shouldSend(): boolean;
  send(type: Notification, payload: NotificationPayload): Promise<boolean>;
}

// 抽象基类提供通用能力
export abstract class BaseAgent<T extends NotificationAgentConfig> {
  protected settings?: T;
  public constructor(settings?: T) {
    this.settings = settings;
  }

  protected abstract getSettings(): T;
}
```

#### 5.5.2 注册机制

**文件**: `server/index.ts:131-143` + `server/lib/notifications/index.ts:95-97`

应用启动时批量注册所有 Agent：

```typescript
// server/index.ts - 启动时注册
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

// server/lib/notifications/index.ts - 注册实现
public registerAgents = (agents: NotificationAgent[]): void => {
  this.activeAgents = [...this.activeAgents, ...agents];
  logger.info('Registered notification agents', { label: 'Notifications' });
};
```

#### 5.5.3 配置类型系统

**文件**: `server/lib/settings/index.ts:220-347`

每个 Agent 有独立的配置接口，通过 `NotificationAgentKey` 枚举关联：

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

// 基类配置
export interface NotificationAgentConfig {
  enabled: boolean;
  embedPoster: boolean;
  types?: number;        // 位掩码，控制接收哪些通知类型
  options: Record<string, unknown>;
}

// 子类扩展配置
export interface NotificationAgentDiscord extends NotificationAgentConfig {
  options: {
    botUsername?: string;
    botAvatarUrl?: string;
    webhookUrl: string;
    locale: AvailableLocale;
  };
}

export interface NotificationAgentWebhook extends NotificationAgentConfig {
  options: {
    webhookUrl: string;
    jsonPayload: string;  // 支持自定义 JSON 模板
    authHeader?: string;
    method: 'POST' | 'PUT' | 'PATCH';
  };
}
```

#### 5.5.4 新增 Agent 扩展步骤

1. **创建 Agent 类**，继承 `BaseAgent` 并实现 `NotificationAgent` 接口：
   ```typescript
   class NewAgent extends BaseAgent<NotificationAgentNew> 
     implements NotificationAgent 
   {
     protected getSettings(): NotificationAgentNew { ... }
     shouldSend(): boolean { ... }
     async send(type, payload): Promise<boolean> { ... }
   }
   ```

2. **在 settings 中添加配置接口**，扩展 `NotificationAgentConfig`

3. **在 `NotificationAgentKey` 枚举中添加新 key**

4. **在 `NotificationSettings` 接口中添加配置字段**

5. **在 `server/index.ts` 中注册新 Agent**

6. **添加类型位掩码检查**：
   ```typescript
   if (!hasNotificationType(type, settings.types ?? 0)) {
     return true;  // 该类型通知未开启，静默跳过
   }
   ```

#### 5.5.5 位掩码通知类型过滤

**文件**: `server/lib/notifications/index.ts`

```typescript
export const hasNotificationType = (type: Notification, types: number): boolean => {
  return (types & type) === type;  // 位与运算检查
};
```

用户可精细控制每个 Agent 接收哪些通知类型：
- `types = 2 | 4 | 8` 只接收 PENDING、APPROVED、AVAILABLE
- `types = 0` 接收所有类型（默认）

#### 5.5.6 通知 Agent 插件热加载机制

**现状分析** (`server/index.ts:131-143` + `server/lib/notifications/index.ts:95-97`):
当前为**启动时静态注册**，修改 Agent 配置需重启服务：
```typescript
// ❌ 静态注册：启动后无法动态增删
notificationManager.registerAgents([
  new DiscordAgent(),    // 硬编码在 index.ts
  new EmailAgent(),
  // ...
]);
```

**热加载改造架构**:

```
┌──────────────────────────────────────────────────────┐
│                  热加载管理器                          │
│                                                      │
│  ┌────────────┐     fs.watch()    ┌───────────────┐  │
│  │ plugins/   │────────────────▶ │ 配置变更检测   │  │
│  │  discord.ts│                   └──────┬────────┘  │
│  │  webhook.ts│                          │           │
│  │  custom*.ts│                          ▼           │
│  └────────────┘              ┌──────────────────┐    │
│                              │ 模块 HMR 卸载/加载 │    │
│                              │ - clear require   │    │
│                              │ - 重新 import()  │    │
│                              └──────┬───────────┘    │
│                                     ▼                │
│                              ┌──────────────────┐    │
│                              │ Agent 注册表更新 │    │
│                              │ - 旧实例 dispose │    │
│                              │ - 新实例注册     │    │
│                              │ - 发送测试消息   │    │
│                              └──────────────────┘    │
└──────────────────────────────────────────────────────┘
```

**热加载核心实现（建议改造）**:
```typescript
// server/lib/notifications/pluginLoader.ts
import chokidar from 'chokidar';
import { NotificationAgent, NotificationAgentKey } from './agents/agent';

class NotificationPluginManager {
  private agentInstances = new Map<NotificationAgentKey, NotificationAgent & { dispose?: () => void }>();
  private watcher: chokidar.FSWatcher;

  // 启动热加载监听
  public async startHotReload() {
    const pluginDir = path.resolve(__dirname, '../../plugins/notifications');

    this.watcher = chokidar.watch(`${pluginDir}/**/*.ts`, {
      ignoreInitial: false,
      awaitWriteFinish: { stabilityThreshold: 500 },  // 等待文件写入完成
    });

    this.watcher
      .on('add', (filePath) => this.loadPlugin(filePath, 'add'))
      .on('change', (filePath) => this.loadPlugin(filePath, 'change'))
      .on('unlink', (filePath) => this.unloadPlugin(filePath));
  }

  // 动态加载插件
  private async loadPlugin(filePath: string, event: 'add' | 'change') {
    try {
      // 1. 清除 Node 模块缓存（关键）
      delete require.cache[require.resolve(filePath)];

      // 2. 动态导入（支持 ESM 和 CJS）
      const pluginModule = await import(filePath);
      const PluginClass = pluginModule.default || pluginModule.PluginClass;

      // 3. 验证接口契约
      if (!this.validateAgentInterface(PluginClass)) {
        logger.error('Plugin invalid: missing required methods', {
          label: 'Notifications',
          filePath,
          fingerprint: `plugin-validation:${path.basename(filePath)}`,  // Sentry 告警键
        });
        return;
      }

      // 4. 先销毁旧实例（如果存在）
      const agentKey = PluginClass.agentKey as NotificationAgentKey;
      if (event === 'change' && this.agentInstances.has(agentKey)) {
        const oldInstance = this.agentInstances.get(agentKey);
        if (oldInstance?.dispose) {
          await oldInstance.dispose();  // 清理资源（连接、定时器等）
        }
        notificationManager.unregisterAgent(agentKey);
      }

      // 5. 创建新实例并注册
      const settings = getSettings().notifications[agentKey];
      const newInstance = new PluginClass(settings);

      notificationManager.registerAgents([newInstance]);
      this.agentInstances.set(agentKey, newInstance);

      logger.info('Notification plugin loaded', {
        label: 'Notifications',
        agentKey,
        event,
        filePath,
      });
    } catch (e) {
      logger.error('Failed to load notification plugin', {
        label: 'Notifications',
        filePath,
        errorMessage: e.message,
        fingerprint: `plugin-load-fail:${path.basename(filePath)}:${e.code || 'unknown'}`,
      });
    }
  }

  // 插件接口校验（类型守卫）
  private validateAgentInterface(cls: unknown): cls is new (...args: any[]) => NotificationAgent {
    return typeof cls === 'function' &&
      typeof cls.prototype.shouldSend === 'function' &&
      typeof cls.prototype.send === 'function' &&
      NotificationAgentKey[cls.agentKey] !== undefined;
  }

  // 安全停止
  public async stop() {
    await this.watcher?.close();
    for (const instance of this.agentInstances.values()) {
      await instance.dispose?.();
    }
  }
}

// 插件需要导出的约定接口
export interface NotificationPlugin {
  agentKey: NotificationAgentKey;  // 静态属性，标识唯一键
  version: string;                 // 版本号，用于兼容性检查
  new (settings?: NotificationAgentConfig): NotificationAgent & {
    dispose?: () => Promise<void> | void;  // 可选的资源清理方法
  };
}
```

**插件目录结构约定**:
```
server/plugins/notifications/
├── custom-slack/
│   ├── index.ts          # 导出 CustomSlackAgent 类
│   ├── manifest.json     # name, version, author, dependencies
│   └── README.md
├── custom-wechat.ts      # 单文件插件也支持
└── _template.ts          # 插件开发模板
```

**热加载安全机制**:
1. **版本兼容性检查**: `manifest.json` 中声明 `compatibleApiVersion >= 1.0.0`
2. **沙箱隔离**: 使用 `vm.Module` 或 `isolated-vm` 隔离第三方插件（可选）
3. **回滚机制**: 新版本加载失败时，自动恢复上一个可用版本
4. **发送测试消息**: 加载成功后立即触发 `TEST_NOTIFICATION` 验证可用性

#### 5.5.7 通知 Agent 旧 ABI 升级兼容矩阵

**源码中的配置接口演变** (`server/lib/settings/index.ts:220-347`):

通过分析 10 个 Agent 的配置字段，可以推导出 ABI 版本历史和兼容性策略：

```typescript
// ABI v1.0 (初始版本)
export interface NotificationAgentConfig {
  enabled: boolean;
  embedPoster: boolean;
  types?: number;
  options: Record<string, unknown>;  // ❌ 弱类型，无结构约束
}

// ABI v1.1 (强类型化)
export interface NotificationAgentDiscord extends NotificationAgentConfig {
  options: {  // ✅ 强类型 options
    webhookUrl: string;
    botUsername?: string;
    botAvatarUrl?: string;
    locale: AvailableLocale;
    // 新增字段不破坏旧插件
  };
}

// ABI v1.2 (高级功能)
export interface NotificationAgentWebhook extends NotificationAgentConfig {
  options: {
    webhookUrl: string;
    jsonPayload: string;      // 新增：自定义 JSON 模板
    authHeader?: string;       // 新增：自定义认证头
    customHeaders?: { key: string; value: string }[];  // 新增：自定义 Headers
    method: 'POST' | 'PUT' | 'PATCH';  // 新增：HTTP 方法
  };
}

// ABI v1.3 (本地化)
export interface NotificationAgentNtfy extends NotificationAgentConfig {
  options: {
    url: string;
    topic: string;
    authMethodUsernamePassword?: boolean;  // 新增：多认证方式
    authMethodToken?: boolean;
    priority?: number;
    locale: AvailableLocale;  // ✅ 规范的 locale 字段
  };
}
```

**ABI 版本兼容矩阵**:

| 插件 ABI 版本 | 核心 API v1.0 | v1.1 | v1.2 | v1.3 | 说明 |
|--------------|--------------|------|------|------|------|
| v1.0 | ✅ 兼容 | ⚠️ 部分兼容 | ❌ 不兼容 | ❌ 不兼容 | 缺少强类型 options 和 locale |
| v1.1 | ✅ 兼容 | ✅ 兼容 | ⚠️ 部分兼容 | ⚠️ 部分兼容 | 缺少 webhook 高级字段和新认证方式 |
| v1.2 | ✅ 兼容 | ✅ 兼容 | ✅ 兼容 | ⚠️ 部分兼容 | 缺少 locale 规范字段 |
| v1.3 | ✅ 兼容 | ✅ 兼容 | ✅ 兼容 | ✅ 兼容 | 完整支持所有字段 |

**向后兼容策略（Adapter 模式）**:
```typescript
// server/lib/notifications/abiCompatibility.ts

// 配置适配器：旧 ABI → 新 ABI
class NotificationAgentAdapter {
  public static adaptToV13(
    agentKey: NotificationAgentKey,
    config: any,
    pluginVersion: string
  ): NotificationAgentConfig {
    const semver = require('semver');

    // v1.0 → v1.1：补充默认 locale
    if (semver.satisfies(pluginVersion, '>=1.0.0 <1.1.0')) {
      return {
        ...config,
        options: {
          ...config.options,
          locale: config.options.locale || 'en',  // 补充默认值
          useUserLocale: false,
        },
      };
    }

    // v1.1 → v1.2：补充 webhook 高级字段默认值
    if (semver.satisfies(pluginVersion, '>=1.1.0 <1.2.0')
        && agentKey === NotificationAgentKey.WEBHOOK) {
      return {
        ...config,
        options: {
          ...config.options,
          jsonPayload: config.options.jsonPayload || JSON.stringify({ text: '{{message}}' }),
          method: config.options.method || 'POST',
          customHeaders: config.options.customHeaders || [],
        },
      };
    }

    // v1.2 → v1.3：补充认证方式字段
    if (semver.satisfies(pluginVersion, '>=1.2.0 <1.3.0')) {
      return this.adaptAuthFields(agentKey, config);
    }

    return config;
  }

  private static adaptAuthFields(agentKey: string, config: any) {
    switch (agentKey) {
      case NotificationAgentKey.NTFY:
        return {
          ...config,
          options: {
            ...config.options,
            authMethodUsernamePassword: config.options.username ? true : false,
            authMethodToken: config.options.token ? true : false,
          },
        };
      default:
        return config;
    }
  }
}

// 插件加载时的兼容性检查
function validateAgentCompat(
  agentClass: any,
  manifestVersion: string
): { compatible: boolean; reason?: string } {
  const coreApiVersion = require('../../../package.json').apiVersion;
  const semver = require('semver');

  // 插件声明的兼容版本范围
  const pluginCompatibleRange = agentClass.compatibleApiVersion || '>=1.0.0';

  if (!semver.satisfies(coreApiVersion, pluginCompatibleRange)) {
    return {
      compatible: false,
      reason: `Plugin requires core API ${pluginCompatibleRange}, but running ${coreApiVersion}`,
    };
  }

  // 检查必需方法
  const requiredMethods = ['shouldSend', 'send', 'getSettings'];
  const missingMethods = requiredMethods.filter(
    m => typeof agentClass.prototype[m] !== 'function'
  );

  if (missingMethods.length > 0) {
    return {
      compatible: false,
      reason: `Missing required methods: ${missingMethods.join(', ')}`,
    };
  }

  return { compatible: true };
}
```

**插件 Manifest 规范（建议）**:
```json
{
  "name": "custom-discord",
  "version": "1.2.3",
  "author": "username",
  "compatibleApiVersion": ">=1.1.0 <1.4.0",
  "agentKey": "discord",
  "dependencies": {
    "discord.js": "^14.0.0"
  },
  "changelog": [
    {
      "version": "1.2.0",
      "changes": "适配核心 API v1.2，增加 webhookThreadId 支持"
    }
  ]
}
```

**不兼容升级的处理流程**:
```
检测到不兼容插件
    │
    ├─ 插件版本过旧 → 提示用户升级插件到兼容版本
    │                → 保留旧配置，不自动删除
    │
    ├─ 插件版本过新 → 提示用户升级核心 Jellyseerr
    │                → 插件被禁用，但配置保留
    │
    ├─ 缺少必需方法 → 插件加载失败，输出详细日志
    │                → fingerprint: ['plugin-incompatible', agentKey, 'missing-methods']
    │
    └─ 核心 ABI 破坏性变更 → 批量转换所有旧插件配置
                            → 执行 SQL 迁移脚本更新配置字段
```

**旧版本配置数据库迁移示例**（参考 `1610370640747-Add4kStatusFields.ts` 的思路）:
```typescript
// 迁移：将 v1.0 webhook 配置升级到 v1.2
await queryRunner.query(`
  UPDATE "settings" 
  SET "value" = json_set("value", '$.notifications.webhook.options.jsonPayload', 
    COALESCE(json_extract("value", '$.notifications.webhook.options.jsonPayload'), 
             '{"text":"{{message}}"}'),
    '$.notifications.webhook.options.method',
    COALESCE(json_extract("value", '$.notifications.webhook.options.method'), '"POST"'))
  WHERE json_extract("value", '$.notifications.webhook.enabled') = 1
`);
```

---

## 六、回执处理与状态回写

### 6.1 可用性同步 (AvailabilitySync)

**文件**: `server/lib/availabilitySync.ts:22-1180`

定时任务 (`availability-sync`) 定期执行，检查媒体是否仍然存在：

```typescript
async run() {
  for await (const media of this.loadAvailableMediaPaginated(50)) {
    if (media.mediaType === 'movie') {
      // 检查 Radarr + Plex/Jellyfin
      const existsInRadarr = await this.mediaExistsInRadarr(media, false);
      const { existsInPlex } = await this.mediaExistsInPlex(media, false);
      
      if (!existsInRadarr && !existsInPlex && media.status === AVAILABLE) {
        await this.mediaUpdater(media, false, mediaServerType);
      }
    }
    
    if (media.mediaType === 'tv') {
      // 类似逻辑，检查 Sonarr + Plex/Jellyfin
      // 额外检查每季的可用性
      await this.seasonUpdater(media, finalSeasons, false, ...);
    }
  }
}
```

### 6.2 媒体状态流转

**文件**: `server/constants/media.ts`

```typescript
export enum MediaStatus {
  UNKNOWN = 1,        // 初始状态
  PENDING = 2,        // 已请求待处理
  PROCESSING = 3,     // 已发送到 *arr，正在下载
  PARTIALLY_AVAILABLE = 4,  // 部分季可用 (剧集)
  AVAILABLE = 5,      // 完全可用
  BLOCKLISTED = 6,    // 被黑名单
  DELETED = 7,        // 已从媒体服务器删除
}

export enum MediaRequestStatus {
  PENDING = 1,        // 待审批
  APPROVED = 2,       // 已批准
  DECLINED = 3,       // 已拒绝
  FAILED = 4,         // 处理失败
  COMPLETED = 5,      // 已完成 (媒体已可用)
}
```

### 6.3 状态回写触发链

```
扫描器发现媒体已下载
    ↓
BaseScanner.processMovie() / processShow()
    ↓
更新 Media.status = AVAILABLE
    ↓
Media.lastSeasonChange 被更新 (仅剧集)
    ↓
(下次扫描或用户查看时)
    ↓
MediaRequestSubscriber 检查相关请求
    ↓
如果所有请求季都可用 → 标记请求 COMPLETED
    ↓
触发 MEDIA_AVAILABLE 通知给用户
```

### 6.4 4K 与非 4K 分离存储

Media 实体为 4K 和非 4K 维护独立的状态字段：
```typescript
// Media 实体字段
status: MediaStatus;          // 非 4K 状态
status4k: MediaStatus;        // 4K 状态
serviceId: number;            // 非 4K 服务 ID
serviceId4k: number;          // 4K 服务 ID
externalServiceId: number;    // 非 4K 外部 ID
externalServiceId4k: number;  // 4K 外部 ID
ratingKey: string;            // Plex 非 4K ratingKey
ratingKey4k: string;          // Plex 4K ratingKey
```

#### 6.4.1 4K 双轨字段数据库迁移

**文件**: `server/migration/sqlite/1610370640747-Add4kStatusFields.ts`

采用**临时表重建 + 数据迁移**的零停机迁移策略（SQLite 不支持 ADD COLUMN 带 DEFAULT 约束的某些场景）：

```typescript
public async up(queryRunner: QueryRunner): Promise<void> {
  // 1. 创建带 status4k 字段的临时表
  await queryRunner.query(
    `CREATE TABLE "temporary_media" (
      "id" integer PRIMARY KEY AUTOINCREMENT NOT NULL, 
      "mediaType" varchar NOT NULL, 
      "tmdbId" integer NOT NULL, 
      "tvdbId" integer, 
      "imdbId" varchar, 
      "status" integer NOT NULL DEFAULT (1), 
      "createdAt" datetime NOT NULL DEFAULT (datetime('now')), 
      "updatedAt" datetime NOT NULL DEFAULT (datetime('now')), 
      "lastSeasonChange" datetime NOT NULL DEFAULT (CURRENT_TIMESTAMP), 
      "status4k" integer NOT NULL DEFAULT (1)  -- 新增 4K 状态字段
    )`
  );

  // 2. 迁移历史数据（不包含 status4k，使用默认值 1 = UNKNOWN）
  await queryRunner.query(
    `INSERT INTO "temporary_media"("id", "mediaType", "tmdbId", "tvdbId", "imdbId", "status", "createdAt", "updatedAt", "lastSeasonChange") 
     SELECT "id", "mediaType", "tmdbId", "tvdbId", "imdbId", "status", "createdAt", "updatedAt", "lastSeasonChange" 
     FROM "media"`
  );

  // 3. 替换原表
  await queryRunner.query(`DROP TABLE "media"`);
  await queryRunner.query(`ALTER TABLE "temporary_media" RENAME TO "media"`);

  // 4. 重建索引
  await queryRunner.query(`CREATE INDEX "IDX_7157aad07c73f6a6ae3bbd5ef5" ON "media" ("tmdbId") `);
  await queryRunner.query(`CREATE INDEX "IDX_41a289eb1fa489c1bc6f38d9c3" ON "media" ("tvdbId") `);

  // 5. Season 表和 MediaRequest 表同步迁移
  await queryRunner.query(
    `CREATE TABLE "temporary_season" (..., "status4k" integer NOT NULL DEFAULT (1))`
  );
  await queryRunner.query(
    `CREATE TABLE "temporary_media_request" (..., "is4k" boolean NOT NULL DEFAULT (0))`
  );
}
```

**迁移关键设计决策**:

| 决策 | 原因 | 影响 |
|------|------|------|
| 临时表重建 | SQLite 不支持 `ALTER TABLE ADD COLUMN ... DEFAULT` 在某些场景 | 迁移期间只读，约 10-30 秒停机（取决于数据量） |
| `status4k DEFAULT 1` | 所有历史媒体默认为 `UNKNOWN` 状态，由后续扫描重新确认 | 首次升级后需要运行全量扫描重新检测 4K 状态 |
| `is4k DEFAULT 0` | 历史请求默认为非 4K | 不影响现有请求流 |
| 关闭外键约束 | SQLite 临时表替换时外键约束会阻塞 DROP TABLE | 迁移期间无外键检查，需确保数据一致性 |

**迁移回滚方案** (`down` 方法):
```typescript
public async down(queryRunner: QueryRunner): Promise<void> {
  // 反向操作：重建无 status4k 字段的原始表
  // 数据迁移时丢弃 status4k 列
}
```

#### 6.4.2 4K 降回 1080p 中间态字段为空场景

**场景分析**:
用户先请求 4K 版本，系统写入 `status4k=PROCESSING, serviceId4k=<值>, externalServiceId4k=<值>`，但下载失败或被删除后，用户再请求 1080p 版本。此时 `status4k` 相关字段可能残留或被清空，导致状态不一致。

**源码中的保护逻辑** (`server/subscriber/MediaRequestSubscriber.ts:838-856`):

```typescript
public async updateParentStatus(entity: MediaRequest): Promise<void> {
  const statusKey = entity.is4k ? 'status4k' : 'status';

  // 🔒 三重状态保护：只有在特定状态下才允许升级到 PROCESSING
  if (
    entity.status === MediaRequestStatus.APPROVED &&
    media[statusKey] !== MediaStatus.AVAILABLE &&           // 排除：已经可用
    media[statusKey] !== MediaStatus.PARTIALLY_AVAILABLE && // 排除：部分可用
    media[statusKey] !== MediaStatus.PROCESSING             // 排除：正在处理（可能是另一个请求的）
  ) {
    media[statusKey] = MediaStatus.PROCESSING;
    await mediaRepository.save(media);
  }

  // 🚫 拒绝时状态保护：电影拒绝只回退到 UNKNOWN
  if (
    media.mediaType === MediaType.MOVIE &&
    entity.status === MediaRequestStatus.DECLINED &&
    media[statusKey] !== MediaStatus.DELETED  // 不覆盖 DELETED（已被删除的媒体保持标记）
  ) {
    media[statusKey] = MediaStatus.UNKNOWN;
    await mediaRepository.save(media);
  }
```

**AvailabilitySync 中的中间态保护** (`server/lib/availabilitySync.ts:536-560`):

```typescript
private async mediaUpdater(media: Media, is4k: boolean, mediaServerType: MediaServerType) {
  // ... 先检测 isMediaProcessing ...

  // 🎯 核心：三元表达式保护 —— 处理中时保留原值，否则清空
  media[is4k ? 'status4k' : 'status'] = MediaStatus.DELETED;
  media[is4k ? 'serviceId4k' : 'serviceId'] = isMediaProcessing
    ? media[is4k ? 'serviceId4k' : 'serviceId']  // 处理中：保留原有服务 ID（可能是 4K 也可能是 1080p）
    : null;                                        // 未处理：清空，避免残留脏数据

  media[is4k ? 'externalServiceId4k' : 'externalServiceId'] =
    isMediaProcessing
      ? media[is4k ? 'externalServiceId4k' : 'externalServiceId']  // 保留
      : null;                                                       // 清空

  media[is4k ? 'ratingKey4k' : 'ratingKey'] = isMediaProcessing
    ? media[is4k ? 'ratingKey4k' : 'ratingKey']  // 保留 Plex ratingKey
    : null;
}
```

**Scanner 中状态降级保护** (`server/lib/scanners/baseScanner.ts:119-134`):

```typescript
// 多条件状态判定矩阵，避免异常覆盖
existing[statusField] =
  !processing && hasFile
    ? MediaStatus.AVAILABLE                              // ✅ 下载完成且有文件 → AVAILABLE
    : !processing && !hasFile && previousStatus === MediaStatus.PROCESSING
      ? MediaStatus.UNKNOWN                              // ⚠️ 4K 降回 1080p 场景：处理中但找不到文件 → 重置 UNKNOWN
      : processing
        ? previousStatus === MediaStatus.DELETED
          ? MediaStatus.DELETED                          // 🔒 已删除的不重新激活
          : MediaStatus.PROCESSING                       // ✅ 正常标记处理中
        : previousStatus;                                // 🛡️ 其他情况保持不变（关键兜底）
```

**典型中间态场景与行为矩阵**:

| 当前 `status4k` | 当前 `serviceId4k` | `is4k` 请求 | 操作 | 结果 `status4k` | 结果 `serviceId4k` |
|----------------|-------------------|------------|------|----------------|-------------------|
| PROCESSING | 非空（有值） | false（请求 1080p） | 审批通过 | PROCESSING（**不变**） | 非空（**保留**，避免 4K 正在下载时被误清） |
| PROCESSING | 非空 | false | AvailabilitySync 检测到媒体缺失 | DELETED | 非空（**保留**，isMediaProcessing=true） |
| UNKNOWN | null | true（请求 4K） | 审批通过 | PROCESSING | 填入新值 |
| DELETED | null | true | 审批通过 | PROCESSING（从 DELETED 恢复） | 填入新值 |
| PARTIALLY_AVAILABLE | 非空 | false | 审批通过 | PARTIALLY_AVAILABLE（**不变**，避免覆盖部分可用） | 非空（保留） |

**状态机转移图**:
```
UNKNOWN ──请求批准──▶ PROCESSING ──*arr 下载完成──▶ AVAILABLE
   ▲                         │                          │
   │                         │ 找不到文件              │ AvailabilitySync
   │                         ▼                          ▼
   └────拒绝/撤销──────── UNKNOWN                    DELETED
                               ▲                         │
                               │ 有新请求/下载中保留元数据 │
                               └─────────────────────────┘
```

#### 6.4.3 4K 中间态数据库回填策略

**问题背景**:
`1610370640747-Add4kStatusFields.ts` 迁移脚本使用 `DEFAULT (1)` 给新字段赋默认值，但存在以下问题：

1. 迁移后 `status4k=1 (UNKNOWN)`，`serviceId4k=null`，`externalServiceId4k=null`
2. 但实际上部分媒体**在迁移前就有 4K 版本存在**于 Radarr/Sonarr 中
3. 如果不回填，下次扫描时这些媒体会被错误地再次发送到 *arr，导致重复下载
4. 对于 `status=AVAILABLE` 且媒体服务器上有 4K 版本的媒体，`status4k` 应该直接回填为 `AVAILABLE`

**迁移脚本现状分析** (`server/migration/sqlite/1610370640747-Add4kStatusFields.ts:6-47`):
```typescript
// 迁移时仅迁移了非 4K 字段
await queryRunner.query(
  `INSERT INTO "temporary_media"("id", "mediaType", "tmdbId", ..., "status") 
   SELECT "id", "mediaType", "tmdbId", ..., "status" FROM "media"`
  // ❌ 没有回填 status4k、serviceId4k、externalServiceId4k
);
```

**4K 中间态回填算法（建议补充的迁移脚本或启动后作业）**:

```typescript
// server/jobs/backfill4kFields.ts

class Backfill4kFieldsJob {
  private readonly BATCH_SIZE = 100;
  private readonly DRY_RUN = false;  // 首次运行建议 dry-run

  public async run(): Promise<BackfillReport> {
    const report: BackfillReport = {
      totalMedia: 0,
      processedMedia: 0,
      backfilled4kAvailable: 0,
      backfilled4kProcessing: 0,
      skippedNoRadarr: 0,
      skippedNoSonarr: 0,
      errors: [],
    };

    // 获取所有已配置的 Radarr/Sonarr 实例
    const settings = getSettings();
    const radarrServers = settings.radarr.filter(s => s.is4k);  // 仅 4K 实例
    const sonarrServers = settings.sonarr.filter(s => s.is4k);

    if (radarrServers.length === 0 && sonarrServers.length === 0) {
      logger.info('No 4K *arr servers configured, skipping backfill', {
        label: 'Backfill4k',
      });
      return report;
    }

    // 分批处理媒体，避免内存溢出
    let offset = 0;
    let mediaBatch: Media[];

    do {
      mediaBatch = await mediaRepository
        .createQueryBuilder('media')
        .leftJoinAndSelect('media.requests', 'request')
        .where('media.status = :available', { available: MediaStatus.AVAILABLE })
        .andWhere('media.status4k = :unknown', { unknown: MediaStatus.UNKNOWN })
        .andWhere(qb => {
          // 只处理可能有 4K 版本的媒体
          const subQb = qb.subQuery()
            .select('1')
            .from('media_request', 'req')
            .where('req."mediaId" = media.id')
            .andWhere('req."is4k" = 1');
          return `EXISTS (${subQb.getQuery()}) OR media.mediaType = :movieType`;
        }, { movieType: MediaType.MOVIE })
        .skip(offset)
        .take(this.BATCH_SIZE)
        .getMany();

      report.totalMedia += mediaBatch.length;

      for (const media of mediaBatch) {
        try {
          await this.backfillMedia4kStatus(media, radarrServers, sonarrServers, report);
          report.processedMedia++;
        } catch (e) {
          report.errors.push({
            tmdbId: media.tmdbId,
            error: e.message,
          });
          logger.error('Failed to backfill 4K status', {
            label: 'Backfill4k',
            tmdbId: media.tmdbId,
            errorMessage: e.message,
            fingerprint: ['backfill-4k-error', String(media.tmdbId)],
          });
        }
      }

      offset += this.BATCH_SIZE;
      await this.delay(1000);  // 每批间隔 1 秒，避免压垮 *arr

    } while (mediaBatch.length > 0);

    logger.info('4K fields backfill completed', {
      label: 'Backfill4k',
      ...report,
      fingerprint: ['backfill-4k-completed', String(report.processedMedia)],
    });

    return report;
  }

  private async backfillMedia4kStatus(
    media: Media,
    radarrServers: RadarrSettings[],
    sonarrServers: SonarrSettings[],
    report: BackfillReport
  ): Promise<void> {
    // 电影：检查 Radarr 4K 实例
    if (media.mediaType === MediaType.MOVIE) {
      for (const radarr of radarrServers) {
        const radarrApi = new RadarrAPI(radarr);
        const movie = await radarrApi.getMovie(media.tmdbId);

        if (movie) {
          if (!this.DRY_RUN) {
            // ✅ 回填 4K 状态
            media.status4k = movie.hasFile
              ? MediaStatus.AVAILABLE
              : this.isMovieDownloading(movie)
                ? MediaStatus.PROCESSING
                : MediaStatus.UNKNOWN;

            media.serviceId4k = radarr.id;
            media.externalServiceId4k = movie.id;
            await mediaRepository.save(media);
          }

          movie.hasFile ? report.backfilled4kAvailable++ : report.backfilled4kProcessing++;
          return;  // 找到一个 4K 实例就足够
        }
      }
    }

    // 剧集：检查 Sonarr 4K 实例
    if (media.mediaType === MediaType.TV) {
      for (const sonarr of sonarrServers) {
        const sonarrApi = new SonarrAPI(sonarr);
        const series = media.tvdbId
          ? await sonarrApi.getSeriesByTvdbId(media.tvdbId)
          : await sonarrApi.getSeries({ tmdbId: media.tmdbId });

        if (series) {
          if (!this.DRY_RUN) {
            // ✅ 回填 4K 状态（按季粒度）
            const seasonStatuses = this.calculateSeasonStatuses(series);
            const overallStatus = this.getOverallStatus(seasonStatuses);

            media.status4k = overallStatus;
            media.serviceId4k = sonarr.id;
            media.externalServiceId4k = series.id;
            await mediaRepository.save(media);

            // 回填 Season 表的 status4k
            for (const season of media.seasons || []) {
              season.status4k = seasonStatuses.get(season.seasonNumber) || MediaStatus.UNKNOWN;
              await seasonRepository.save(season);
            }
          }

          (overallStatus === MediaStatus.AVAILABLE
            ? report.backfilled4kAvailable
            : report.backfilled4kProcessing)++;
          return;
        }
      }
    }
  }

  // 判断电影是否在下载中
  private isMovieDownloading(movie: RadarrMovie): boolean {
    return movie.queueState?.downloading ||
           movie.movieFile === null && movie.monitored;
  }

  // 计算剧集每一季的 4K 状态
  private calculateSeasonStatuses(series: SonarrSeries): Map<number, MediaStatus> {
    const statuses = new Map<number, MediaStatus>();

    for (const season of series.seasons) {
      const totalEpisodes = season.statistics?.episodeCount || 0;
      const availableEpisodes = season.statistics?.episodeFileCount || 0;

      if (totalEpisodes === 0) {
        statuses.set(season.seasonNumber, MediaStatus.UNKNOWN);
      } else if (availableEpisodes === totalEpisodes) {
        statuses.set(season.seasonNumber, MediaStatus.AVAILABLE);
      } else if (availableEpisodes > 0) {
        statuses.set(season.seasonNumber, MediaStatus.PARTIALLY_AVAILABLE);
      } else if (season.monitored) {
        statuses.set(season.seasonNumber, MediaStatus.PROCESSING);
      } else {
        statuses.set(season.seasonNumber, MediaStatus.UNKNOWN);
      }
    }

    return statuses;
  }
}
```

**回填优先级策略**:
```typescript
// 按媒体类型和请求状态确定回填优先级
const BACKFILL_PRIORITY = [
  // P0: 有 APPROVED 的 4K 请求（正在下载）
  {
    condition: 'request.is4k = 1 AND request.status = 2',  // APPROVED = 2
    priority: 0,
    action: '立即回填，避免重复发送到 *arr',
  },
  // P1: 有 PENDING 的 4K 请求
  {
    condition: 'request.is4k = 1 AND request.status = 1',  // PENDING = 1
    priority: 1,
    action: '高优先级回填，审批通过时需要正确状态',
  },
  // P2: 非 4K 状态为 AVAILABLE 的电影
  {
    condition: 'media.mediaType = 0 AND media.status = 5',  // MOVIE + AVAILABLE
    priority: 2,
    action: '可能有 4K 版本存在',
  },
  // P3: 非 4K 状态为 AVAILABLE 的剧集
  {
    condition: 'media.mediaType = 1 AND media.status = 5',  // TV + AVAILABLE
    priority: 3,
    action: '低优先级，按季检查成本高',
  },
];
```

**幂等性保证（防止重复回填）**:
```typescript
// 回填作业启动时检查标记
const lastBackfillTime = await systemSettingsRepository.findOne({ key: 'last4kBackfillAt' });
if (lastBackfillTime && Date.now() - Number(lastBackfillTime.value) < 24 * 60 * 60 * 1000) {
  logger.info('4K backfill already ran within 24h, skipping', {
    label: 'Backfill4k',
    lastRunAt: lastBackfillTime.value,
  });
  return;
}

// 回填完成后更新标记
await systemSettingsRepository.save({
  key: 'last4kBackfillAt',
  value: String(Date.now()),
});
```

---

### 6.5 异步非阻塞背压控制

#### 6.5.1 扫描器背压机制

**文件**: `server/lib/scanners/baseScanner.ts:687-725` + `:646-681`

三层背压控制防止系统过载：

**第一层 - 会话隔离 (Session ID)**:
```typescript
protected startRun(): string {
  const sessionId = randomUUID();  // 每次运行生成唯一会话
  this.sessionId = sessionId;
  this.running = true;
  return sessionId;
}

protected async loop(..., { sessionId } = {}): Promise<void> {
  if (this.sessionId !== sessionId) {
    throw new Error('New session was started. Old session aborted.');  // 新会话启动时终止旧会话
  }
  // ...
}
```

**第二层 - 分批节流**:
```typescript
// 默认配置
const BUNDLE_SIZE = 20;       // 每批 20 条
const UPDATE_RATE = 4 * 1000;  // 批间间隔 4 秒

await new Promise<void>((resolve) =>
  setTimeout(() => {
    this.loop(processFn, {
      start: start + this.bundleSize,
      sessionId,
    }).then(() => resolve());
  }, this.updateRate)  // 强制等待，防止 CPU/IO 过载
);
```

**第三层 - 可取消标志**:
```typescript
public cancel(): void {
  this.running = false;  // 由外部设置取消标志
}

if (!this.running) {
  throw new Error('Sync was aborted.');  // 每批次开始前检查
}
```

#### 6.5.2 AvailabilitySync 背压控制

**文件**: `server/lib/availabilitySync.ts:38-466`

```typescript
async run() {
  this.running = true;
  try {
    for await (const media of this.loadAvailableMediaPaginated(50)) {
      if (!this.running) {
        throw new Error('Job aborted');  // 每页检查取消标志
      }
      // 处理单条媒体...
    }
  } finally {
    this.running = false;
  }
}

// 异步生成器分页拉取，避免一次性加载所有数据
private async *loadAvailableMediaPaginated(pageSize: number) {
  let offset = 0;
  let mediaPage: Media[];
  do {
    yield* (mediaPage = await mediaRepository.find({
      where: whereOptions,
      skip: offset,
      take: pageSize,  // 每页 50 条
    }));
    offset += pageSize;
  } while (mediaPage.length > 0);
}
```

**背压效果**:
- 内存占用稳定（O(pageSize) 而非 O(total)）
- 数据库查询压力可控
- 可随时中断，重启时无需重头开始

#### 6.5.3 背压熔断水位线设计

**源码中的监控指标** (`server/lib/scanners/baseScanner.ts:15-19, 59-66, 646-681`):

```typescript
// 状态输出接口
export type StatusBase = {
  running: boolean;   // 熔断开关：false 时直接触发熔断
  progress: number;   // 已处理数量
  total: number;      // 总数量
};

// 扫描器状态
protected progress = 0;
protected items: T[] = [];
protected totalSize?: number = 0;
protected sessionId: string;    // 会话 ID：用于检测会话过期
protected running = false;      // 全局熔断标志
```

**四级熔断水位线（建议补充实现）**:

```
│               │  Level 4: FATAL          │  > 50% 超时 │  终止所有批次 + 报警 + 全局降级（跳过本次扫描）
│               │──────────────────────────│────────────│
│               │  Level 3: CRITICAL       │  > 30% 超时 │  updateRate × 3（降速到 12 秒）
│     慢批次   │──────────────────────────│────────────│
│     超时率   │  Level 2: WARNING        │  > 10% 超时 │  updateRate × 1.5（降速到 6 秒）
│               │──────────────────────────│────────────│
│               │  Level 1: INFO           │  < 10% 超时 │  正常速率 4 秒/批
│               │──────────────────────────│────────────│
│               │  Level 0: IDLE           │  running=false │  完全停止，等待 cancel() 释放
```

**熔断机制实现（建议改造 baseScanner.ts）**:
```typescript
abstract class BaseScanner<T> {
  // 水位线配置
  private readonly WATERMARK = {
    WARN_SLOW_BATCH_RATIO: 0.10,     // 10% 批次超过 2 倍平均耗时 → 警告
    CRITICAL_SLOW_BATCH_RATIO: 0.30, // 30% 批次超 2 倍 → 严重
    FATAL_SLOW_BATCH_RATIO: 0.50,    // 50% 批次超 3 倍 → 致命
    BATCH_TIMEOUT_BASELINE: 8000,    // 批次 8 秒基线超时
    DB_ERROR_THRESHOLD: 5,           // 单批次 DB 错误超过 5 次 → 熔断
    API_429_THRESHOLD: 3,            // 单批次 429 超过 3 次 → 降速
  } as const;

  // 运行时统计
  private batchStats = {
    totalBatches: 0,
    slowBatches: 0,
    dbErrors: 0,
    api429Errors: 0,
    avgBatchDurationMs: 0,
    sessionStartAt: 0,
  };

  protected async loop(processFn, { sessionId }) {
    // 会话级熔断：如果 sessionId 过期，直接终止
    if (this.sessionId !== sessionId) {
      throw new Error('New session was started. Old session aborted.');
    }

    // 全局熔断标志检查
    if (!this.running) {
      throw new Error('Sync was aborted.');
    }

    // 处理前计算当前水位
    const currentWatermark = this.calculateWatermark();

    // 动态调整 updateRate（自适应降速）
    const adaptiveDelay = this.updateRate * this.getBackoffMultiplier(currentWatermark);

    // Level 4 致命熔断
    if (currentWatermark === 'FATAL') {
      this.running = false;
      logger.error('Scan FATAL watermark exceeded, triggering global circuit breaker', {
        label: this.scannerName,
        sessionId,
        fingerprint: ['circuit-breaker-fatal', this.scannerName, sessionId],
        stats: { ...this.batchStats },
      });
      throw new Error('Circuit breaker: FATAL watermark exceeded');
    }

    // Level 2/3 降速（通过增大 setTimeout 延时实现）
    await new Promise<void>((resolve) =>
      setTimeout(() => this.loop(...).then(resolve), adaptiveDelay)
    );
  }

  // 计算当前水位
  private calculateWatermark(): 'IDLE' | 'NORMAL' | 'WARN' | 'CRITICAL' | 'FATAL' {
    if (!this.running) return 'IDLE';
    const { totalBatches, slowBatches, dbErrors, api429Errors } = this.batchStats;
    if (totalBatches === 0) return 'NORMAL';

    const slowRatio = slowBatches / totalBatches;

    // 任一硬阈值命中直接升级
    if (dbErrors >= this.WATERMARK.DB_ERROR_THRESHOLD ||
        api429Errors >= this.WATERMARK.API_429_THRESHOLD * 5) {
      return 'FATAL';
    }
    if (slowRatio >= this.WATERMARK.FATAL_SLOW_BATCH_RATIO) return 'FATAL';
    if (slowRatio >= this.WATERMARK.CRITICAL_SLOW_BATCH_RATIO) return 'CRITICAL';
    if (slowRatio >= this.WATERMARK.WARN_SLOW_BATCH_RATIO) return 'WARN';
    return 'NORMAL';
  }

  // 退避乘数
  private getBackoffMultiplier(level: string): number {
    switch (level) {
      case 'CRITICAL': return 3.0;
      case 'WARN': return 1.5;
      default: return 1.0;
    }
  }
}
```

**熔断状态外部暴露（健康检查 API）**:
```typescript
// GET /api/v1/settings/status/jobs
// Response:
{
  "plex-full-scan": {
    "running": true,
    "progress": 1540,
    "total": 8500,
    "watermark": "WARN",
    "backoffMultiplier": 1.5,
    "currentAdaptiveDelayMs": 6000,
    "stats": {
      "slowBatchRatio": 0.12,
      "avgBatchDurationMs": 2300,
      "dbErrors": 0,
      "api429Errors": 1
    },
    "estimatedRemainingTimeSec": 14800
  }
}
```

#### 6.5.4 背压基于消费速率的动态水位调整

**问题背景**:
静态水位线（10%/30%/50% 超时率）在不同负载下表现不佳：
- 系统空闲时，即使 20% 批次超时也可能是正常的（冷启动、网络抖动）
- 系统高负载时，5% 超时率就应该触发降速（CPU/IO 接近饱和）
- 不同扫描任务的消费速率差异很大（Plex 扫描 vs Radarr 同步）

**动态水位调整算法（基于 EWMA 平滑消费速率）**:

```typescript
// server/lib/scanners/baseScanner.ts - 扩展实现

abstract class BaseScanner<T> {
  // 指数加权移动平均（Exponential Weighted Moving Average）
  private readonly EWMA_ALPHA = 0.3;  // 平滑系数，0.3 表示最近批次占 30% 权重
  private readonly BASELINE_WINDOW = 10;  // 前 10 批作为基准线

  private batchDurations: number[] = [];
  private ewmaBatchDuration = 0;  // 平滑后的平均批次耗时
  private maxBatchDuration = 0;   // 历史最大批次耗时
  private minBatchDuration = Infinity;  // 历史最小批次耗时

  // 消费速率 = 处理数量 / 时间
  private consumptionRate = 0;    // items/sec
  private ewmaConsumptionRate = 0;

  // 动态水位线（根据消费速率调整）
  private dynamicWatermark = {
    WARN: 0.10,
    CRITICAL: 0.30,
    FATAL: 0.50,
  };

  // 批次处理后更新统计
  protected onBatchComplete(batchSize: number, batchDurationMs: number) {
    this.batchDurations.push(batchDurationMs);
    if (this.batchDurations.length > 100) {
      this.batchDurations.shift();  // 保留最近 100 批次
    }

    // 更新 EWMA 平均批次耗时
    if (this.ewmaBatchDuration === 0) {
      this.ewmaBatchDuration = batchDurationMs;  // 初始值
    } else {
      this.ewmaBatchDuration =
        this.EWMA_ALPHA * batchDurationMs +
        (1 - this.EWMA_ALPHA) * this.ewmaBatchDuration;
    }

    // 更新极值
    this.maxBatchDuration = Math.max(this.maxBatchDuration, batchDurationMs);
    this.minBatchDuration = Math.min(this.minBatchDuration, batchDurationMs);

    // 计算消费速率（items/sec）
    const currentRate = (batchSize / batchDurationMs) * 1000;
    if (this.ewmaConsumptionRate === 0) {
      this.ewmaConsumptionRate = currentRate;
    } else {
      this.ewmaConsumptionRate =
        this.EWMA_ALPHA * currentRate +
        (1 - this.EWMA_ALPHA) * this.ewmaConsumptionRate;
    }

    // 根据消费速率动态调整水位线
    this.adjustWatermarkDynamically();

    // 更新批次统计
    this.batchStats.totalBatches++;
    if (batchDurationMs > this.getSlowThreshold()) {
      this.batchStats.slowBatches++;
    }
  }

  // 动态水位线调整核心算法
  private adjustWatermarkDynamically() {
    // 前 10 批不调整，用于建立基准线
    if (this.batchStats.totalBatches < this.BASELINE_WINDOW) return;

    // 计算当前速率与基准速率的比值
    const baselineRate = this.getBaselineConsumptionRate();
    const rateRatio = this.ewmaConsumptionRate / baselineRate;

    // 速率越快（系统越空闲），水位线越高
    // 速率越慢（系统越繁忙），水位线越低
    const adjustmentFactor = Math.max(0.3, Math.min(2.0, rateRatio));

    // 调整各水位线
    this.dynamicWatermark = {
      WARN:     Math.min(0.25, 0.10 * adjustmentFactor),
      CRITICAL: Math.min(0.40, 0.30 * adjustmentFactor),
      FATAL:    Math.min(0.60, 0.50 * adjustmentFactor),
    };

    logger.debug('Dynamic watermark adjusted', {
      label: this.scannerName,
      rateRatio: rateRatio.toFixed(2),
      currentRate: this.ewmaConsumptionRate.toFixed(2),
      baselineRate: baselineRate.toFixed(2),
      watermark: { ...this.dynamicWatermark },
    });
  }

  // 计算基准消费速率（前 10 批平均）
  private getBaselineConsumptionRate(): number {
    if (this.batchDurations.length < this.BASELINE_WINDOW) return 0;

    const first10Batches = this.batchDurations.slice(0, this.BASELINE_WINDOW);
    const totalItems = first10Batches.length * this.bundleSize;
    const totalTime = first10Batches.reduce((a, b) => a + b, 0);
    return (totalItems / totalTime) * 1000;
  }

  // 慢批次阈值 = EWMA 平均的 2 倍
  private getSlowThreshold(): number {
    return this.ewmaBatchDuration * 2;
  }

  // 覆盖原 calculateWatermark 方法，使用动态水位
  protected calculateWatermark(): 'IDLE' | 'NORMAL' | 'WARN' | 'CRITICAL' | 'FATAL' {
    if (!this.running) return 'IDLE';
    const { totalBatches, slowBatches, dbErrors, api429Errors } = this.batchStats;
    if (totalBatches < this.BASELINE_WINDOW) return 'NORMAL';  // 基准线建立中

    const slowRatio = slowBatches / totalBatches;

    // 硬阈值检查（优先级高于动态水位）
    if (dbErrors >= this.WATERMARK.DB_ERROR_THRESHOLD ||
        api429Errors >= this.WATERMARK.API_429_THRESHOLD * 5) {
      return 'FATAL';
    }

    // 使用动态水位线
    if (slowRatio >= this.dynamicWatermark.FATAL) return 'FATAL';
    if (slowRatio >= this.dynamicWatermark.CRITICAL) return 'CRITICAL';
    if (slowRatio >= this.dynamicWatermark.WARN) return 'WARN';
    return 'NORMAL';
  }

  // 自适应退避乘数（基于动态水位和消费速率）
  protected getBackoffMultiplier(level: string): number {
    const baseMultiplier = super.getBackoffMultiplier(level);
    const rateRatio = this.ewmaConsumptionRate / this.getBaselineConsumptionRate();

    // 消费速率越低，退避乘数越大（更激进的降速）
    const rateAdjustment = Math.max(1.0, 2.0 - rateRatio);
    return baseMultiplier * rateAdjustment;
  }
}
```

**消费速率动态水位调整矩阵**:

| 消费速率比值 (当前/基准) | 说明 | WARN 水位 | CRITICAL 水位 | FATAL 水位 | 退避调整系数 |
|------------------------|------|-----------|--------------|-----------|------------|
| > 1.5 | 系统非常空闲，处理速度远超基准 | 0.15 | 0.40 | 0.60 | × 1.0 |
| 1.0 - 1.5 | 系统正常负载 | 0.10 | 0.30 | 0.50 | × 1.0 - 1.5 |
| 0.5 - 1.0 | 系统中等负载，处理变慢 | 0.075 | 0.225 | 0.375 | × 1.5 - 2.0 |
| 0.3 - 0.5 | 系统高负载，需要降速 | 0.05 | 0.15 | 0.25 | × 2.0 |
| < 0.3 | 系统过载，紧急降速 | 0.03 | 0.09 | 0.15 | × 2.0 |

**批次耗时统计样例**:
```
批次 1: 1200ms (基准)  EWMA = 1200ms
批次 2: 1300ms         EWMA = 0.3×1300 + 0.7×1200 = 1230ms
批次 3: 800ms          EWMA = 0.3×800  + 0.7×1230 = 1091ms
批次 4: 3500ms (慢)    EWMA = 0.3×3500 + 0.7×1091 = 1859ms
...
慢批次阈值 = 1859 × 2 = 3718ms
```

**动态水位调整效果**:
- 冷启动阶段（前 10 批）：不调整，使用默认水位
- 系统空闲时：水位上移，减少不必要的降速
- 系统繁忙时：水位下移，提前触发降速保护
- 长期稳定后：水位收敛到适合当前负载的最优值

---

### 6.6 AvailabilitySync 漂移补偿机制

**文件**: `server/lib/availabilitySync.ts:498-617`

#### 6.6.1 漂移检测

漂移场景：媒体在 Plex/Jellyfin 中被删除，但在 Radarr/Sonarr 中仍然存在（或反之）。

```typescript
// 双重校验：必须在 *arr 和媒体服务器中都不存在才标记为删除
const existsInRadarr = await this.mediaExistsInRadarr(media, false);
const { existsInPlex } = await this.mediaExistsInPlex(media, false);

// 逻辑与：只有两边都不存在才认为真正被删除
if (!existsInRadarr && !existsInPlex && media.status === MediaStatus.AVAILABLE) {
  await this.mediaUpdater(media, false, mediaServerType);
}
```

#### 6.6.2 处理中状态保护

当媒体正在下载时，即使暂时在 Plex 中找不到也不能标记为删除：

```typescript
private async mediaUpdater(media, is4k, mediaServerType) {
  // 检查是否有处理中的请求
  const request = await requestRepository
    .createQueryBuilder('request')
    .where('(media.id = :id)', { id: media.id })
    .andWhere('(request.is4k = :is4k AND request.status = :requestStatus)', {
      requestStatus: MediaRequestStatus.APPROVED,  // 处理中
      is4k: is4k,
    })
    .getOne();

  const isMediaProcessing = !!request;

  // 漂移补偿：处理中时保留外部服务元数据
  media[is4k ? 'serviceId4k' : 'serviceId'] = isMediaProcessing
    ? media[is4k ? 'serviceId4k' : 'serviceId']  // 保留原值
    : null;                                        // 否则清空
  
  media[is4k ? 'externalServiceId4k' : 'externalServiceId'] = isMediaProcessing
    ? media[is4k ? 'externalServiceId4k' : 'externalServiceId']
    : null;
  // ... 其他字段同理
  
  media[is4k ? 'status4k' : 'status'] = MediaStatus.DELETED;  // 仅状态标记为删除
}
```

#### 6.6.3 季级粒度补偿

剧集按季检查，避免单季删除影响整部剧：

```typescript
const filteredSeasonsMap: Map<number, boolean> = new Map();
media.seasons
  .filter(season => 
    season.status === AVAILABLE || season.status === PARTIALLY_AVAILABLE
  )
  .forEach(season => filteredSeasonsMap.set(season.seasonNumber, false));

// 交叉比对 Plex 和 Sonarr
const finalSeasons = new Map([
  ...filteredSeasonsMap,
  ...plexSeasonsMap,
  ...sonarrSeasonsMap,
]);

// 只有两边都不存在的季才标记为删除
if ([...finalSeasons.values()].includes(false)) {
  await this.seasonUpdater(media, finalSeasons, false, mediaServerType);
}
```

#### 6.6.4 TMDB 兜底校验

当媒体服务器数据可能不准确时，从 TMDB 获取真实的季信息：

```typescript
try {
  if (media.tmdbId) {
    tvShow = await this.tmdb.getTvShow({ tvId: Number(media.tmdbId) });
  }
} catch (e) {
  // TMDB 失败时跳过该季的检查，避免误删除
}

// 用 TMDB 数据补全可能遗漏的季
if (tvShow) {
  media.seasons.forEach(season => {
    if (season.seasonNumber === 0) return;  // 跳过特别篇
    if (!finalSeasons.has(season.seasonNumber) &&
        tvShow.seasons.find(s => s.season_number === season.seasonNumber)?.episode_count) {
      finalSeasons.set(season.seasonNumber, false);  // 标记需要检查
    }
  });
}
```

#### 6.6.5 漂移超 24h 兜底降级策略

**漂移问题背景**:
AvailabilitySync 定期（`availability-sync` 任务，默认每小时一次）检查媒体是否仍然存在。但以下场景可能导致**状态漂移**超过 24 小时未被纠正：
1. Radarr/Sonarr 实例离线超过 24 小时，恢复后数据库状态丢失
2. Plex/Jellyfin 媒体库迁移，文件被重新扫描但元数据变化
3. 数据库备份恢复后，Media 表与实际磁盘状态不一致
4. 扫描任务挂起（进程阻塞、Session ID 过期未清理）

**源码中的时间戳字段** (`server/lib/scanners/baseScanner.ts:644, availabilitySync.ts:644`):

```typescript
// Season 状态变更时更新 lastSeasonChange
media.lastSeasonChange = new Date();
await mediaRepository.save(media);

// Media 实体：updatedAt、createdAt、lastSeasonChange、mediaAddedAt
```

**超 24h 兜底降级策略（建议补充实现）**:

```typescript
// server/lib/availabilitySync.ts
class AvailabilitySync {
  private readonly DRIFT_THRESHOLD_MS = 24 * 60 * 60 * 1000;  // 24 小时
  private readonly DRIFT_FALLBACK_ENABLED = true;

  async run() {
    this.running = true;
    try {
      // 正常的可用性同步逻辑...

      // 兜底：扫描 PROCESSING/PENDING 状态超 24h 的媒体
      if (this.DRIFT_FALLBACK_ENABLED) {
        await this.runDriftFallback();
      }
    } finally {
      this.running = false;
    }
  }

  private async runDriftFallback() {
    const driftThreshold = new Date(Date.now() - this.DRIFT_THRESHOLD_MS);

    // 场景 1：电影 PROCESSING 超 24h → 降级为 UNKNOWN + 告警
    const stuckProcessingMovies = await mediaRepository
      .createQueryBuilder('media')
      .where('media.mediaType = :movieType', { movieType: MediaType.MOVIE })
      .andWhere('(media.status = :processing OR media.status4k = :processing)', {
        processing: MediaStatus.PROCESSING,
      })
      .andWhere('media.updatedAt < :threshold', { threshold: driftThreshold })
      .getMany();

    for (const media of stuckProcessingMovies) {
      await this.handleDriftedMedia(media, 'PROCESSING_TIMEOUT');
    }

    // 场景 2：剧集 PARTIALLY_AVAILABLE 超 24h → 检查每一季
    const stuckPartialShows = await mediaRepository
      .createQueryBuilder('media')
      .leftJoinAndSelect('media.seasons', 'season')
      .where('media.mediaType = :tvType', { tvType: MediaType.TV })
      .andWhere('(media.status = :partial OR media.status4k = :partial)', {
        partial: MediaStatus.PARTIALLY_AVAILABLE,
      })
      .andWhere('media.lastSeasonChange < :threshold', { threshold: driftThreshold })
      .getMany();

    for (const media of stuckPartialShows) {
      await this.handleDriftedShow(media);
    }

    // 场景 3：DELETED 超 24h 且有新请求 → 清理脏字段
    const deletedWithPending = await mediaRepository
      .createQueryBuilder('media')
      .innerJoinAndSelect('media.requests', 'request')
      .where('(media.status = :deleted OR media.status4k = :deleted)', {
        deleted: MediaStatus.DELETED,
      })
      .andWhere('request.status = :pending', { pending: MediaRequestStatus.PENDING })
      .andWhere('media.updatedAt < :threshold', { threshold: driftThreshold })
      .getMany();

    for (const media of deletedWithPending) {
      await this.cleanupDriftedDeletedMedia(media);
    }
  }

  private async handleDriftedMedia(media: Media, reason: string) {
    const mediaRepository = getRepository(Media);

    // 记录漂移告警
    logger.warn('Media status drift detected, applying fallback', {
      label: 'AvailabilitySync',
      tmdbId: media.tmdbId,
      mediaType: media.mediaType,
      status: media.status,
      status4k: media.status4k,
      updatedAgoMs: Date.now() - (media.updatedAt?.getTime() || 0),
      driftReason: reason,
      fingerprint: [
        'availability-drift',
        reason,
        `tmdb:${media.tmdbId}`,
        `type:${media.mediaType}`,
      ],
    });

    // 降级策略：
    // PROCESSING → UNKNOWN（等待下次扫描重新确认）
    // 但保留外部服务元数据，避免用户手动配置丢失
    if (media.status === MediaStatus.PROCESSING) {
      media.status = MediaStatus.UNKNOWN;
    }
    if (media.status4k === MediaStatus.PROCESSING) {
      media.status4k = MediaStatus.UNKNOWN;
    }

    await mediaRepository.save(media);
  }

  private async handleDriftedShow(media: Media) {
    const seasonRepository = getRepository(Season);
    const driftThreshold = new Date(Date.now() - this.DRIFT_THRESHOLD_MS);

    for (const season of media.seasons) {
      // 只有 24h 内未变更的季才触发兜底
      if (season.updatedAt && season.updatedAt > driftThreshold) continue;

      // 检查该季是否有处理中的请求
      const activeRequest = await getRepository(SeasonRequest)
        .createQueryBuilder('sr')
        .innerJoin('sr.request', 'request')
        .where('sr.seasonId = :seasonId', { seasonId: season.id })
        .andWhere('request.status IN (:...statuses)', {
          statuses: [MediaRequestStatus.APPROVED, MediaRequestStatus.PENDING],
        })
        .getExists();

      if (!activeRequest) {
        // 无活跃请求的漂移季：从 PARTIALLY_AVAILABLE 降级
        if (season.status === MediaStatus.PARTIALLY_AVAILABLE) {
          season.status = MediaStatus.UNKNOWN;
          await seasonRepository.save(season);
        }
        if (season.status4k === MediaStatus.PARTIALLY_AVAILABLE) {
          season.status4k = MediaStatus.UNKNOWN;
          await seasonRepository.save(season);
        }
      }
    }
  }

  private async cleanupDriftedDeletedMedia(media: Media) {
    const mediaRepository = getRepository(Media);

    // DELETED 状态但有待处理请求 → 可能是误标删除
    // 策略：降级为 UNKNOWN，清空外部 ID，触发下次扫描重新拉取
    if (media.status === MediaStatus.DELETED) {
      media.status = MediaStatus.UNKNOWN;
      media.serviceId = null;
      media.externalServiceId = null;
    }
    if (media.status4k === MediaStatus.DELETED) {
      media.status4k = MediaStatus.UNKNOWN;
      media.serviceId4k = null;
      media.externalServiceId4k = null;
    }

    logger.warn('Cleaned up drifted DELETED media with pending requests', {
      label: 'AvailabilitySync',
      tmdbId: media.tmdbId,
      fingerprint: ['availability-drift-deleted-with-pending', `tmdb:${media.tmdbId}`],
    });

    await mediaRepository.save(media);
  }
}
```

**漂移状态降级决策表**:

| 当前状态 | 持续时间 | 活跃请求 | 降级后状态 | 清理外部 ID | 告警级别 |
|---------|---------|---------|-----------|------------|---------|
| PROCESSING | > 24h | ❌ 无 | UNKNOWN | ❌ 保留 | warning |
| PROCESSING | > 24h | ✅ 有 | PROCESSING（不变） | ❌ 保留 | warning（仅告警） |
| PARTIALLY_AVAILABLE | > 24h | ❌ 无 | AVAILABLE / UNKNOWN（根据 *arr 确认） | ❌ 保留 | warning |
| PARTIALLY_AVAILABLE | > 24h | ✅ 有 | 不变 | ❌ 保留 | info |
| DELETED | > 24h | ✅ 有 | UNKNOWN | ✅ 清空 | warning |
| DELETED | > 24h | ❌ 无 | DELETED（不变） | ❌ 保留 | 不告警 |
| UNKNOWN | > 72h | ❌ 无 | UNKNOWN（不变） | ✅ 清理空字段 | info |

**扫描器任务挂起兜底 (Task Heartbeat)**:
```typescript
// server/job/schedule.ts - 任务心跳检查
const HEARTBEAT_TIMEOUT = 30 * 60 * 1000;  // 30 分钟无心跳

setInterval(() => {
  for (const [jobName, scanner] of runningScanners) {
    if (scanner.running && 
        (Date.now() - scanner.lastHeartbeatAt) > HEARTBEAT_TIMEOUT) {
      // 强制取消超时任务
      scanner.cancel();
      logger.error('Scanner stuck, force canceled', {
        label: 'Scheduler',
        jobName,
        stuckDurationMs: Date.now() - scanner.lastHeartbeatAt,
        fingerprint: ['scanner-heartbeat-timeout', jobName],
      });
      // 触发重新调度
      rescheduleJob(jobName);
    }
  }
}, 60000);  // 每分钟检查一次
```

#### 6.6.6 漂移降级每步回滚条件

**问题背景**:
漂移降级是一种**保守操作**，将 `PROCESSING → UNKNOWN`、`PARTIALLY_AVAILABLE → UNKNOWN`。但降级后如果外部系统恢复正常（*arr 重新上线、Plex 重新扫描完成），需要有明确的回滚条件，将状态恢复回正常值。

**降级操作与回滚条件对照表**:

| 降级前状态 | 降级后状态 | 降级触发条件 | 回滚条件（恢复原状） | 回滚检查频率 |
|-----------|-----------|-------------|---------------------|-------------|
| `PROCESSING` | `UNKNOWN` | 超 24h 未完成 + 无活跃请求 | ✅ Radarr/Sonarr 中该媒体仍在下载队列<br>✅ 或有新的 `APPROVED` 请求创建<br>✅ 或 Plex 扫描检测到文件存在 | 下次 AvailabilitySync 运行（每小时） |
| `PROCESSING` | `UNKNOWN` | 超 24h 未完成 + 有活跃请求 | ✅ 请求仍为 `APPROVED` 状态（保留降级，仅告警）<br>❌ 不自动回滚，需用户手动重试 | 下次 `availability-sync` 任务 |
| `PARTIALLY_AVAILABLE` | `UNKNOWN` | 超 24h 无变化 + 无活跃请求 | ✅ Sonarr 中该季有监控的缺失剧集<br>✅ 或 Plex 扫描检测到新增文件<br>✅ 或有新的部分请求创建 | 下次 `availability-sync` 任务 |
| `DELETED` | `UNKNOWN` | 误标删除 + 有待处理请求 | ✅ Radarr/Sonarr 中该媒体仍然存在<br>✅ 或 Plex 中检测到文件存在<br>✅ 且请求状态为 `APPROVED` | 下次 `availability-sync` 任务 |
| `DELETED` | 保持 `DELETED` | 超 24h + 无请求 | ⚠️ 无自动回滚，需用户手动删除请求 | 永不自动回滚 |

**回滚检测算法（AvailabilitySync 中补充）**:

```typescript
// server/lib/availabilitySync.ts

private async runDriftRollback() {
  const rollbackReport: RollbackReport = {
    totalRollback: 0,
    processingToProcessing: 0,
    unknownToAvailable: 0,
    errors: [],
  };

  // 场景 1: UNKNOWN 状态但 *arr 中有正在下载的记录 → 恢复 PROCESSING
  const unknownWithProcessing = await mediaRepository
    .createQueryBuilder('media')
    .leftJoinAndSelect('media.requests', 'request')
    .where('(media.status = :unknown OR media.status4k = :unknown)', {
      unknown: MediaStatus.UNKNOWN,
    })
    .andWhere(qb => {
      const subQb = qb.subQuery()
        .select('1')
        .from('media_request', 'req')
        .where('req."mediaId" = media.id')
        .andWhere('req.status = :approved');
      return `EXISTS (${subQb.getQuery()})`;
    }, { approved: MediaRequestStatus.APPROVED })
    .getMany();

  for (const media of unknownWithProcessing) {
    try {
      const rolledBack = await this.rollbackProcessingStatus(media);
      if (rolledBack) rollbackReport.processingToProcessing++;
    } catch (e) {
      rollbackReport.errors.push({ tmdbId: media.tmdbId, error: e.message });
    }
  }

  // 场景 2: UNKNOWN 状态但文件已存在 → 恢复 AVAILABLE
  const unknownWithFiles = await mediaRepository
    .createQueryBuilder('media')
    .where('(media.status = :unknown OR media.status4k = :unknown)', {
      unknown: MediaStatus.UNKNOWN,
    })
    .andWhere('media.updatedAt > :driftThreshold', {
      driftThreshold: new Date(Date.now() - 48 * 60 * 60 * 1000),  // 48h 内降级的
    })
    .getMany();

  for (const media of unknownWithFiles) {
    try {
      const rolledBack = await this.rollbackAvailableStatus(media);
      if (rolledBack) rollbackReport.unknownToAvailable++;
    } catch (e) {
      rollbackReport.errors.push({ tmdbId: media.tmdbId, error: e.message });
    }
  }

  rollbackReport.totalRollback =
    rollbackReport.processingToProcessing + rollbackReport.unknownToAvailable;

  if (rollbackReport.totalRollback > 0) {
    logger.info('Drift rollback completed', {
      label: 'AvailabilitySync',
      ...rollbackReport,
      fingerprint: ['drift-rollback', String(rollbackReport.totalRollback)],
    });
  }

  return rollbackReport;
}

// 回滚 PROCESSING 状态
private async rollbackProcessingStatus(media: Media): Promise<boolean> {
  let rolledBack = false;

  for (const is4k of [false, true]) {
    const statusField = is4k ? 'status4k' : 'status';
    const serviceIdField = is4k ? 'serviceId4k' : 'serviceId';
    const externalIdField = is4k ? 'externalServiceId4k' : 'externalServiceId';

    // 只有 UNKNOWN 状态的才考虑回滚
    if (media[statusField] !== MediaStatus.UNKNOWN) continue;

    // 检查是否有活跃请求
    const hasActiveRequest = media.requests?.some(
      r => r.is4k === is4k && r.status === MediaRequestStatus.APPROVED
    );
    if (!hasActiveRequest) continue;

    // 检查 *arr 中是否有正在下载的记录
    const existsInArr = await this.mediaExistsInArr(media, is4k);
    const isDownloading = await this.isMediaDownloadingInArr(media, is4k);

    if (existsInArr && isDownloading) {
      // ✅ 满足回滚条件：恢复 PROCESSING 状态
      media[statusField] = MediaStatus.PROCESSING;

      // 保留 serviceId 和 externalServiceId（如果存在）
      // 否则后续 BaseScanner 会重新填充
      logger.info('Rolled back drift: UNKNOWN → PROCESSING', {
        label: 'AvailabilitySync',
        tmdbId: media.tmdbId,
        is4k,
        fingerprint: ['drift-rollback-processing', String(media.tmdbId), String(is4k)],
      });

      rolledBack = true;
    }
  }

  if (rolledBack) {
    await mediaRepository.save(media);
  }

  return rolledBack;
}

// 回滚 AVAILABLE 状态
private async rollbackAvailableStatus(media: Media): Promise<boolean> {
  let rolledBack = false;

  for (const is4k of [false, true]) {
    const statusField = is4k ? 'status4k' : 'status';
    if (media[statusField] !== MediaStatus.UNKNOWN) continue;

    // 检查 Plex/Jellyfin 中是否有文件
    const { existsInPlex, hasMediaFiles } = await this.mediaExistsInPlex(media, is4k);

    // 检查 *arr 中是否有已下载完成的文件
    const existsInArr = await this.mediaExistsInArr(media, is4k);
    const hasArrFiles = await this.mediaHasFilesInArr(media, is4k);

    if ((existsInPlex && hasMediaFiles) || (existsInArr && hasArrFiles)) {
      // ✅ 文件存在，恢复 AVAILABLE 状态
      media[statusField] = MediaStatus.AVAILABLE;

      logger.info('Rolled back drift: UNKNOWN → AVAILABLE', {
        label: 'AvailabilitySync',
        tmdbId: media.tmdbId,
        is4k,
        source: existsInPlex ? 'plex' : 'arr',
        fingerprint: ['drift-rollback-available', String(media.tmdbId), String(is4k)],
      });

      rolledBack = true;
    }
  }

  if (rolledBack) {
    await mediaRepository.save(media);
  }

  return rolledBack;
}
```

**降级-回滚状态机**:
```
PROCESSING ──┐
             │  超 24h + 无活跃请求
             ▼
          UNKNOWN
             │
             ├─ ✅ *arr 仍在下载 + 有 APPROVED 请求
             │       └─ 回滚 → PROCESSING
             │
             ├─ ✅ Plex/*arr 检测到文件已存在
             │       └─ 回滚 → AVAILABLE
             │
             └─ ❌ 24h 内无变化 + 无请求
                     └─ 保持 UNKNOWN，等待下次扫描
```

**回滚幂等性保护**:
```typescript
// 每次回滚操作都会记录审计日志
private async recordRollbackAudit(
  mediaId: number,
  is4k: boolean,
  fromStatus: MediaStatus,
  toStatus: MediaStatus,
  reason: string
) {
  // 使用幂等键防止重复回滚
  const idempotencyKey = `rollback:${mediaId}:${is4k}:${fromStatus}:${toStatus}:${Date.now() / 86400000 | 0}`;

  const existing = await this.redis?.get(idempotencyKey);
  if (existing) {
    logger.debug('Rollback already executed today, skipping', {
      label: 'AvailabilitySync',
      idempotencyKey,
    });
    return false;
  }

  await this.redis?.set(idempotencyKey, '1', 'EX', 86400);  // 24h 幂等

  // 写入审计日志
  await driftRollbackLogRepository.save({
    mediaId,
    is4k,
    fromStatus,
    toStatus,
    reason,
    rolledBackAt: new Date(),
  });

  return true;
}
```

**回滚触发的通知**:
- 回滚到 `PROCESSING`: 静默处理，不通知（避免干扰用户）
- 回滚到 `AVAILABLE`: 触发 `MEDIA_AVAILABLE` 通知（用户期待的）
- 回滚失败: 发送 `MEDIA_FAILED` 告警给管理员

**回滚保护边界**:
1. **24 小时内只能回滚一次**（幂等键每日粒度）
2. **降级后 48 小时内才考虑回滚**（避免频繁状态抖动）
3. **DELETED 状态永不自动回滚**（需用户手动干预）
4. **有活跃请求时 PROCESSING 降级不回滚**（保持降级状态，仅告警）

---

## 七、错误回滚机制

### 7.0 5 种回滚场景优先级

回滚操作按**数据一致性影响程度**和**操作不可逆程度**确定优先级：

| 优先级 | 回滚场景 | 触发条件 | 影响范围 | 不可逆程度 |
|-------|---------|---------|---------|-----------|
| 🔴 P0 | **请求删除回滚** | 请求被用户/管理员删除 | 父级 Media 状态 | ⭐⭐⭐⭐⭐ |
| 🟠 P1 | **请求拒绝回滚** | 请求被管理员拒绝 | 父级 Media + Season 状态 | ⭐⭐⭐⭐ |
| 🟡 P2 | **发送到 *arr 失败回滚** | Radarr/Sonarr API 调用失败 | MediaRequest 状态 | ⭐⭐⭐ |
| 🟢 P3 | **连接/配置错误回滚** | 网络错误或配置无效 | MediaRequest 状态 | ⭐⭐ |
| 🔵 P4 | **重试机制** | 用户手动触发重试 | 重置状态重新发送 | ⭐ |

**优先级设计原则**:
1. **P0 > P1**: 删除比拒绝更严重，删除意味着用户请求完全消失
2. **P1 > P2**: 人为操作（拒绝）比系统错误优先级高
3. **P2 > P3**: API 业务失败比连接失败更需要明确标记
4. **P3 > P4**: 错误标记比重试更紧急

**高优先级回滚会覆盖低优先级状态**:
```typescript
// P0 删除回滚可以覆盖任何状态
if (needsStatusUpdate) {
  cleanMedia.status = hadCompleted ? DELETED : UNKNOWN;  // 无条件覆盖
}

// P1 拒绝回滚只在特定条件下执行
if (media.mediaType === MOVIE && entity.status === DECLINED) {
  // 只重置为 UNKNOWN，不设置 DELETED
  media.status = UNKNOWN;
}

// P2/P3 失败回滚有幂等检查
if (entity.status !== FAILED) {  // 防止重复标记
  entity.status = FAILED;
}
```

### 7.0.1 回滚优先级与 Sentry 告警键关联

**现状分析**:
当前项目未集成 Sentry（全项目 grep 无 `@sentry/node` 引用），但 `logger.error/warn` 的结构化日志参数天然适合映射为 Sentry `fingerprint` 和 `tags`。

**Sentry 告警键 (Fingerprint) 设计映射表**:

| 优先级 | 回滚场景 | Sentry Fingerprint 模板 | Sentry Level | 触发位置 |
|-------|---------|------------------------|--------------|---------|
| 🔴 P0 | **请求删除回滚** | `['rollback-p0-delete', 'media:'+mediaId, 'tmdb:'+tmdbId, 'request:'+requestId]` | error | `handleRemoveParentUpdate` (Subscriber.ts:946) |
| 🟠 P1 | **请求拒绝回滚** | `['rollback-p1-decline', 'media:'+mediaId, 'operator:'+modifiedById]` | warning | `updateParentStatus` (Subscriber.ts:849) |
| 🟡 P2 | **API 业务失败回滚** | `['rollback-p2-api-failed', 'radarr:'+serverName, 'error:'+errorCode]` | error | `sendToRadarr` .catch() (Subscriber.ts:398) |
| 🟢 P3 | **连接/配置错误回滚** | `['rollback-p3-connect-fail', hostname, 'error:'+e.code]` | warning | `sendToRadarr` 外层 catch (Subscriber.ts:789) |
| 🔵 P4 | **重试机制** | `['rollback-p4-retry', 'request:'+requestId, 'retry-count:'+count]` | info | `POST /request/:requestId/retry` (request.ts:633) |

**Sentry 集成示例（建议改造）**:
```typescript
// 在 logger 封装层增加 Sentry 桥接
const sentryError = (message: string, context: {
  label: string;
  fingerprint?: string[];
  tags?: Record<string, string>;
  user?: { id: number; email: string };
  extra?: Record<string, unknown>;
}) => {
  logger.error(message, context);

  // 自动根据回滚优先级映射 Sentry 级别
  const level = context.fingerprint?.[0]?.includes('p0') ? 'error'
              : context.fingerprint?.[0]?.includes('p1') ? 'warning'
              : context.fingerprint?.[0]?.includes('p2') ? 'error'
              : context.fingerprint?.[0]?.includes('p3') ? 'warning'
              : 'info';

  Sentry.withScope((scope) => {
    if (context.fingerprint) scope.setFingerprint(context.fingerprint);
    if (context.tags) Object.entries(context.tags).forEach(([k, v]) => scope.setTag(k, v));
    if (context.user) scope.setUser(context.user);
    if (context.extra) scope.setExtras(context.extra);
    scope.setLevel(level as any);
    scope.setTag('component', context.label);  // 用 label 作为组件分类
    Sentry.captureException(new Error(message));
  });
};

// 实际调用示例（P2 失败回滚）
sentryError('Failed to send to Radarr', {
  label: 'Media Request',
  fingerprint: ['rollback-p2-api-failed', `radarr:${radarrSettings.name}`, `error:${e.code}`],
  tags: {
    media_type: entity.type === MediaType.MOVIE ? 'movie' : 'tv',
    is_4k: String(entity.is4k),
    server_id: String(radarrSettings.id),
  },
  user: { id: entity.modifiedBy?.id, email: entity.modifiedBy?.email },
  extra: { requestId: entity.id, mediaId: entity.media.id },
});
```

**Sentry 告警分组策略（Rules）**:
```yaml
# Sentry Issue Alert Rule
conditions:
  - type: event.frequency
    value: 10  # 10 分钟内超过 10 次
    window: 600

# P0/P2 级 issue 直接 PagerDuty 呼叫 oncall
fingerprint_pattern:
  contains:
    - "rollback-p0"
    - "rollback-p2"
```

### 7.0.2 Sentry 告警去重窗口

**去重原理**:
Sentry 通过 `fingerprint` 对事件进行分组，相同 fingerprint 的事件会被合并到同一个 Issue 中。去重窗口（Deduplication Window）控制同一事件在多久内重复发生时不创建新告警。

**Sentry 内置去重机制**:
```
事件到达 Sentry → 检查 fingerprint
    │
    ├─ 已有同 fingerprint Issue 存在
    │   │
    │   ├─ 距上次事件 < 去重窗口（默认 24h）
    │   │   └─ 合并到已有 Issue，更新事件计数
    │   │
    │   └─ 距上次事件 >= 去重窗口
    │       └─ 创建新的 Issue 或者重新激活已 resolved 的 Issue
    │
    └─ 无同 fingerprint Issue → 创建新 Issue
```

**源码中的 fingerprint 设计与去重窗口映射**:

```typescript
// Sentry 集成示例（已在 §7.0.1 中定义）
const sentryError = (message: string, context: {
  fingerprint?: string[];
  dedupeWindow?: number;  // 自定义去重窗口（毫秒）
  // ...
}) => {
  Sentry.withScope((scope) => {
    if (context.fingerprint) scope.setFingerprint(context.fingerprint);

    // 自定义去重窗口：通过 Sentry SDK 的 beforeSend 钩子实现
    scope.addEventProcessor((event) => {
      const key = context.fingerprint?.join(':') || event.event_id;
      const lastSentAt = this.dedupeCache.get(key);
      const window = context.dedupeWindow || this.getDefaultDedupeWindow(context.fingerprint);

      if (lastSentAt && Date.now() - lastSentAt < window) {
        return null;  // 返回 null 表示丢弃该事件
      }

      this.dedupeCache.set(key, Date.now(), window);
      return event;
    });

    Sentry.captureException(new Error(message));
  });
};
```

**去重窗口按优先级分层**:

| 回滚优先级 | 默认去重窗口 | 去重键前缀 | 说明 |
|-----------|-------------|-----------|------|
| P0 删除 | 5 分钟 | `rollback-p0:` | 紧急事件，短窗口快速重复告警 |
| P1 拒绝 | 15 分钟 | `rollback-p1:` | 重要事件，中等窗口 |
| P2 API 失败 | 1 小时 | `rollback-p2:` | 业务失败，较长窗口 |
| P3 连接错误 | 4 小时 | `rollback-p3:` | 连接错误，长窗口避免告警轰炸 |
| P4 重试 | 24 小时 | `rollback-p4:` | 重试操作，默认 24h 窗口 |
| 死锁强制释放 | 30 分钟 | `asyncLock-deadlock:` | 死锁事件，中等窗口 |
| 时钟漂移检测 | 12 小时 | `clock-drift:` | 时钟问题，长窗口 |
| 4K 回填错误 | 1 小时 | `backfill-4k-error:` | 回填错误，中等窗口 |

**智能去重窗口（基于错误频率自适应）**:
```typescript
private getDefaultDedupeWindow(fingerprint?: string[]): number {
  if (!fingerprint) return 24 * 60 * 60 * 1000;  // 默认 24h

  const key = fingerprint.join(':');
  const eventCount = this.eventFrequency.get(key) || 0;

  // 根据事件频率动态调整窗口
  if (eventCount > 100) return 60 * 60 * 1000;      // 高频事件：1h 窗口
  if (eventCount > 50) return 30 * 60 * 1000;       // 中频事件：30min 窗口
  if (eventCount > 10) return 15 * 60 * 1000;       // 低频事件：15min 窗口
  return 5 * 60 * 1000;                              // 极少事件：5min 窗口
}
```

**多层去重架构**:
```
┌──────────────────────────────────────────────────────────┐
│  L1: 应用内去重（Javascript 内存 Map）                   │
│      window: 5min, 避免同一实例短时间重复上报             │
│      过期自动清理：Map + TTL                              │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│  L2: Sentry SDK beforeSend 去重                           │
│      window: 可配置，按优先级分层                          │
│      本地缓存 + 指纹匹配                                   │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│  L3: Sentry 服务端去重（Ingestion Pipeline）              │
│      window: 24h，基于 fingerprint 分组                    │
│      相同 Issue 内事件计数，不重复告警                      │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│  L4: Alert Rule 抑制（Rate Limit）                        │
│      10min 内超过 10 次才触发告警                           │
│      支持 PagerDuty 静默期配置                              │
└──────────────────────────────────────────────────────────┘
```

**Sentry 去重配置（服务端）**:
```yaml
# sentry.yml 配置
filters:
  - !Filter
    id: deduplicate
    config:
      window: 86400  # 24h 默认去重窗口

alert_rules:
  - name: P0 Critical Rollback
    conditions:
      - fingerprint:
          contains: ["rollback-p0"]
    actions:
      - pagerduty:
          account: oncall
          severity: critical
    rate_limit:
      count: 1
      window: 300  # 5 分钟内最多 1 次告警
```

---

### 7.1 发送到 *arr 失败回滚

**文件**: `server/subscriber/MediaRequestSubscriber.ts:398-432` (Radarr 示例)

```typescript
radarr
  .addMovie(radarrMovieOptions)
  .catch(async () => {
    try {
      const requestRepository = getRepository(MediaRequest);
      
      // 幂等检查：避免重复标记
      if (entity.status !== MediaRequestStatus.FAILED) {
        entity.status = MediaRequestStatus.FAILED;
        await requestRepository.save(entity);
      }
    } catch (saveError) {
      // 二次错误只记录日志
      logger.error('Failed to mark request as FAILED', { ... });
    }

    // 发送失败通知
    MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
  });
```

### 7.2 连接/配置错误回滚

```typescript
} catch (e) {
  // 连接错误或配置错误
  entity.status = MediaRequestStatus.FAILED;
  await requestRepository.save(entity);
  
  MediaRequest.sendNotification(entity, media, Notification.MEDIA_FAILED);
}
```

### 7.3 重试机制

**文件**: `server/routes/request.ts:633-661`

用户可通过 API 重试失败的请求：
```typescript
requestRoutes.post('/:requestId/retry', 
  isAuthenticated(Permission.MANAGE_REQUESTS),
  async (req, res, next) => {
    const request = await requestRepository.findOneOrFail(...);
    
    // 将状态重置为 APPROVED，触发 afterUpdate 重新发送
    request.status = MediaRequestStatus.APPROVED;
    request.modifiedBy = req.user;
    await requestRepository.save(request);
    
    return res.status(200).json(request);
  }
);
```

### 7.4 请求删除时的状态回滚

**文件**: `server/subscriber/MediaRequestSubscriber.ts:946-1004`

```typescript
public async handleRemoveParentUpdate(
  manager: EntityManager,
  entity: MediaRequest
): Promise<void> {
  // 检查是否还有其他活跃请求
  const hasActive = fullMedia.requests.some(
    (request) => !request.is4k && 
      request.status !== COMPLETED && 
      request.status !== DECLINED
  );

  // 如果没有活跃请求，重置媒体状态
  if (!hasActive && media.status !== AVAILABLE) {
    // 如果有过完成请求，标记为 DELETED，否则 UNKNOWN
    cleanMedia.status = hadCompleted ? MediaStatus.DELETED : MediaStatus.UNKNOWN;
    await manager.save(cleanMedia);
  }
}
```

### 7.5 请求拒绝时的回滚

**文件**: `server/subscriber/MediaRequestSubscriber.ts:864-932`

```typescript
// 电影：如果唯一请求被拒绝，重置媒体状态
if (media.mediaType === MOVIE && entity.status === DECLINED) {
  media.status = MediaStatus.UNKNOWN;
  await mediaRepository.save(media);
}

// 剧集：检查是否还有其他待处理请求
if (media.mediaType === TV && entity.status === DECLINED) {
  const pendingCount = await requestRepository.count({ ... });
  if (pendingCount === 0) {
    freshMedia.status = MediaStatus.UNKNOWN;
    await mediaRepository.save(freshMedia);
  }
  
  // 重置相关季的状态
  for (const seasonRequest of entity.seasons) {
    seasonRequest.status = DECLINED;
    await seasonRequestRepository.save(seasonRequest);
    
    // 如果该季没有其他活跃请求，重置状态
    if (otherActiveRequests === 0) {
      season.status = MediaStatus.UNKNOWN;
      await seasonRepository.save(season);
    }
  }
}
```

---

### 7.6 TypeORM EventSubscriber 事件回放机制

#### 7.6.1 事件订阅架构

**文件**: `server/subscriber/MediaRequestSubscriber.ts:1-1050` + `server/subscriber/MediaSubscriber.ts`

TypeORM 的 `EntitySubscriberInterface` 提供实体生命周期钩子：

```typescript
@EventSubscriber()
export class MediaRequestSubscriber 
  implements EntitySubscriberInterface<MediaRequest> 
{
  // 监听的实体类型
  listenTo() {
    return MediaRequest;
  }

  // 实体更新后触发
  public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
    if (!event.entity) return;
    
    // 注意：event.entity 只包含变更字段
    // event.databaseEntity 包含完整的数据库状态
    
    try {
      await this.sendToRadarr(event.entity as MediaRequest);
      await this.sendToSonarr(event.entity as MediaRequest);
    } catch (e) {
      logger.error('Error while sending to *arr', { ... });
    }
    
    try {
      await this.updateParentStatus(event.entity as MediaRequest);
    } catch (e) {
      logger.error('Error while updating parent status', { ... });
    }
  }

  // 实体删除前触发
  public async beforeRemove(event: RemoveEvent<MediaRequest>): Promise<void> {
    if (!event.entity) return;
    await this.handleRemoveParentUpdate(event.manager, event.entity);
  }
}
```

#### 7.6.2 事件回放与事务边界

**事件触发链**:
```
HTTP 请求 → Controller → repository.save() 
    ↓
TypeORM 事务 → UPDATE SQL → 提交事务
    ↓
@AfterUpdate 钩子触发 → MediaRequestSubscriber.afterUpdate()
    ↓
sendToRadarr() / sendToSonarr() → 异步调用外部 API
    ↓
外部 API 成功 → update Media.externalServiceId → repository.save()
    ↓
触发另一次 @AfterUpdate → 可能导致循环调用
```

**防循环调用设计**:
```typescript
// sendToRadarr 中先检查状态，避免重复处理
public async sendToRadarr(entity: MediaRequest): Promise<void> {
  if (
    entity.status === MediaRequestStatus.APPROVED &&  // 只有 APPROVED 状态才处理
    entity.type === MediaType.MOVIE
  ) {
    // ... 执行发送
  }
}

// 外部 API 成功后更新 Media，不会触发 MediaRequest 的 afterUpdate
media.externalServiceId = radarrMovie.id;
await mediaRepository.save(media);  // 只更新 Media，不更新 MediaRequest
```

**事务边界问题**:
```typescript
// ❌ 问题：afterUpdate 在事务提交后触发，但如果后续操作失败
public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
  await this.sendToRadarr(event.entity);  // 这个调用可能失败
  // 但数据库事务已经提交，无法回滚
}

// ✅ 解决方案：catch + 状态标记 + 通知
.catch(async () => {
  entity.status = MediaRequestStatus.FAILED;
  await requestRepository.save(entity);  // 二次写入标记失败
  MediaRequest.sendNotification(..., Notification.MEDIA_FAILED);
});
```

#### 7.6.3 事件实体状态对比

`UpdateEvent` 提供两个关键实体对象：

| 对象 | 包含内容 | 用途 |
|------|---------|------|
| `event.entity` | 仅包含被修改的字段 | 获取变更后的值 |
| `event.databaseEntity` | 数据库中的完整实体 | 获取变更前的状态 |

```typescript
// 状态变更检测示例
public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
  const { entity, databaseEntity } = event;
  
  if (entity && databaseEntity) {
    // 检测状态从 PENDING 变为 APPROVED
    if (databaseEntity.status === MediaRequestStatus.PENDING &&
        entity.status === MediaRequestStatus.APPROVED) {
      // 触发审批通知
      MediaRequest.sendNotification(..., Notification.MEDIA_APPROVED);
    }
    
    // 检测状态从 APPROVED 变为 COMPLETED
    if (databaseEntity.status === MediaRequestStatus.APPROVED &&
        entity.status === MediaRequestStatus.COMPLETED) {
      // 触发可用通知
      MediaRequest.sendNotification(..., Notification.MEDIA_AVAILABLE);
    }
  }
}
```

#### 7.6.4 删除事件的特殊处理

删除操作在 `beforeRemove` 中处理，因为 `afterRemove` 时实体已不存在：

```typescript
// server/subscriber/MediaRequestSubscriber.ts:946-1004
public async handleRemoveParentUpdate(
  manager: EntityManager,  // 使用事务内的 manager
  entity: MediaRequest
): Promise<void> {
  // 必须使用传入的 manager，而不是全局 getRepository()
  // 因为删除操作在事务中，需要保证原子性
  
  const fullMedia = await manager.findOneOrFail(Media, {
    where: { id: entity.media.id },
    relations: { requests: true },
  });
  
  // 计算新状态...
  
  // 重新 fetch 不带 relations 的实体，避免级联删除问题
  const cleanMedia = await manager.findOneOrFail(Media, {
    where: { id: entity.media.id },
  });
  
  cleanMedia.status = newStatus;
  await manager.save(cleanMedia);  // 同一事务内保存
}
```

#### 7.6.5 事件回放幂等键设计

**问题背景**:
TypeORM 的 `AfterInsert` / `AfterUpdate` 事件可能因重试机制、Promise 不等待、或数据库集群主从延迟导致**重复触发**。如果幂等保护不足，同一事件可能被处理多次，造成重复通知、重复发送到 *arr、重复状态更新等问题。

**源码中的隐式幂等保护**:

| 幂等场景 | 现有保护键 | 保护位置 |
|---------|-----------|---------|
| 请求审批重复发送到 Radarr | `entity.status === APPROVED` + `media.status !== PROCESSING` | `sendToRadarr():185` + `updateParentStatus():838` |
| 重复通知 | `media[statusKey] !== AVAILABLE && !== PARTIALLY_AVAILABLE` | `updateParentStatus():841` |
| 重复创建 Media 记录 | AsyncLock(tmdbId) + DB UNIQUE 约束 | `processMovie():113` |
| 重复保存失败状态 | `entity.status !== FAILED` | `sendToRadarr().catch():398` |

**显式幂等键设计（建议补充）**:

```typescript
// server/subscriber/MediaRequestSubscriber.ts
class MediaRequestSubscriber {
  // Redis 或内存去重表
  private idempotencyKeys = new Map<string, number>();  // key -> 过期时间戳
  private readonly IDEMPOTENCY_TTL = 60000;  // 60 秒内事件去重

  // 计算幂等键（基于事件唯一标识）
  private getIdempotencyKey(
    eventType: 'afterInsert' | 'afterUpdate' | 'beforeRemove',
    entity: MediaRequest,
    databaseEntity?: MediaRequest
  ): string {
    // 幂等键 = 事件类型 + 实体 ID + 更新时间戳 + 变更状态哈希
    const statusHash = databaseEntity
      ? `${databaseEntity.status}->${entity.status}`
      : 'INSERT';

    return `event:${eventType}:${entity.id}:${entity.updatedAt?.getTime() || 'now'}:${statusHash}`;
  }

  // 去重检查
  private isDuplicateEvent(
    eventType: 'afterInsert' | 'afterUpdate' | 'beforeRemove',
    entity: MediaRequest,
    databaseEntity?: MediaRequest
  ): boolean {
    const key = this.getIdempotencyKey(eventType, entity, databaseEntity);
    const now = Date.now();

    // 清理过期键（惰性删除）
    for (const [k, exp] of this.idempotencyKeys) {
      if (exp < now) this.idempotencyKeys.delete(k);
    }

    if (this.idempotencyKeys.has(key)) {
      logger.debug('Duplicate event skipped by idempotency key', {
        label: 'Media Request',
        eventType,
        requestId: entity.id,
        idempotencyKey: key,
        fingerprint: ['event-deduplicated', eventType, String(entity.id)],
      });
      return true;
    }

    this.idempotencyKeys.set(key, now + this.IDEMPOTENCY_TTL);
    return false;
  }

  // 应用示例：在 afterUpdate 中使用
  public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
    if (!event.entity) return;

    // 幂等检查：同一事件 60 秒内只处理一次
    if (this.isDuplicateEvent('afterUpdate', event.entity, event.databaseEntity)) {
      return;
    }

    // 原逻辑继续...
    await this.sendToRadarr(event.entity);
  }
}
```

**幂等键分层设计（多实例部署需 Redis 实现）**:

```
幂等键层级:
┌─────────────────────────────────────────────────────┐
│  Global Level (跨实例共享) - Redis SETNX            │
│  key: jseer:idem:global:<eventIdempotencyHash>      │
│  TTL: 300s（覆盖扫描周期的 5 倍）                    │
├─────────────────────────────────────────────────────┤
│  Instance Level (实例内) - 内存 Map                  │
│  key: <eventType>:<entityId>:<updatedAtHash>         │
│  TTL: 60s（覆盖同一会话内的重试）                     │
├─────────────────────────────────────────────────────┤
│  DB Level (持久化) - 数据库列约束                    │
│  UNIQUE(tmdbId, mediaType, is4k)                     │
│  + 状态变更前检查 entity.status !== targetStatus      │
└─────────────────────────────────────────────────────┘
```

**状态变更幂等矩阵**:
| databaseEntity.status | entity.status | 允许处理？ | 说明 |
|----------------------|---------------|-----------|------|
| PENDING | APPROVED | ✅ | 正常审批流程 |
| APPROVED | APPROVED | ❌ | 重复事件，跳过 |
| APPROVED | COMPLETED | ✅ | 媒体已下载完成 |
| PENDING | DECLINED | ✅ | 正常拒绝 |
| PENDING | PENDING | ❌ | 无状态变更，跳过 |
| APPROVED | FAILED | ✅ | 处理失败，标记失败状态 |

#### 7.6.6 跨进程回放幂等冲突

**问题背景**:
多实例部署时，两个或多个实例可能**几乎同时**接收到相同的事件（如 TypeORM 集群主从同步延迟导致重复触发 AfterUpdate）。此时：

```
Instance A → 检查 idempotencyKey → 不存在 → 标记为存在 → 处理事件
Instance B → 检查 idempotencyKey → 可能同时检测到不存在 → 也标记为存在 → 重复处理！
```

这就是经典的 **TOCTOU 竞态条件**（Time-of-check to time-of-use）。

**源码层面的竞态风险分析**:

```typescript
// ❌ 非原子的先检查后写入（当前实现有竞态风险）
private isDuplicateEvent(...): boolean {
  const key = this.getIdempotencyKey(...);
  const now = Date.now();

  // 这里存在竞态窗口！
  if (this.idempotencyKeys.has(key)) {  // Instance A 检查
    // Instance B 也检查到不存在
    return true;
  }

  // 在这个时间窗口，两个实例都可能通过检查
  this.idempotencyKeys.set(key, now + this.IDEMPOTENCY_TTL);  // Instance A 设置
  // Instance B 也设置，但实际上事件已被 A 处理了
  return false;
}
```

**Redis 原子幂等键实现（使用 SETNX）**:

```typescript
class DistributedIdempotencyManager {
  private readonly IDEMPOTENCY_TTL = 300000;  // 5 分钟，覆盖扫描周期
  private readonly LOCK_ACQUIRE_TIMEOUT = 100;  // SETNX 轮询间隔

  // ✅ 原子操作：使用 Redis SETNX + PX 保证只有一个实例能获取到幂等键
  public async tryAcquireIdempotencyKey(
    key: string,
    instanceId: string
  ): Promise<{ acquired: boolean; owner?: string; conflict: boolean }> {
    const lockValue = `${instanceId}:${Date.now()}`;

    // Redis SET NX PX 原子操作：设置值当且仅当键不存在
    const acquired = await this.redis.set(
      this.getKey(key),
      lockValue,
      'PX',
      this.IDEMPOTENCY_TTL,
      'NX'  // 仅当键不存在时设置
    );

    if (acquired === 'OK') {
      // 成功获取到幂等键
      return { acquired: true, owner: instanceId, conflict: false };
    }

    // 键已存在，读取当前持有者
    const currentValue = await this.redis.get(this.getKey(key));
    const [owner, timestamp] = (currentValue || '').split(':');

    return {
      acquired: false,
      owner: owner || 'unknown',
      conflict: true,
    };
  }

  // 幂等键释放（事件处理完成后主动删除，不等待 TTL）
  public async releaseIdempotencyKey(key: string, instanceId: string): Promise<boolean> {
    const currentValue = await this.redis.get(this.getKey(key));
    const [owner] = (currentValue || '').split(':');

    // 只有持有者才能释放，防止误删
    if (owner === instanceId) {
      await this.redis.del(this.getKey(key));
      return true;
    }

    logger.warn('Attempted to release idempotency key owned by another instance', {
      label: 'Idempotency',
      key,
      attemptedBy: instanceId,
      actualOwner: owner,
      fingerprint: ['idempotency-release-denied', key],
    });
    return false;
  }

  // 带重试的幂等获取（处理瞬时冲突）
  public async acquireWithRetry(
    key: string,
    instanceId: string,
    maxRetries = 3
  ): Promise<{ acquired: boolean; conflict: boolean }> {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
      const result = await this.tryAcquireIdempotencyKey(key, instanceId);

      if (result.acquired) {
        return result;
      }

      // 检查是否为瞬时冲突：检查剩余 TTL，如果很短可以等待并重试
      const ttl = await this.redis.pttl(this.getKey(key));
      if (ttl > 0 && ttl < 1000) {
        // 键将在 1s 内过期，等待后重试
        await this.delay(ttl + 50);
        continue;
      }

      // 否则为真冲突，直接返回
      return result;
    }

    return { acquired: false, conflict: true };
  }
}

// 在 MediaRequestSubscriber 中使用
class MediaRequestSubscriber {
  public async afterUpdate(event: UpdateEvent<MediaRequest>): Promise<void> {
    if (!event.entity) return;

    const key = this.getIdempotencyKey('afterUpdate', event.entity, event.databaseEntity);
    const { acquired, conflict, owner } = await this.idempotencyManager.acquireWithRetry(
      key,
      this.instanceId,
      3
    );

    if (!acquired) {
      if (conflict) {
        logger.debug('Cross-instance idempotency conflict, skipping', {
          label: 'Media Request',
          key,
          handledBy: owner,
          conflict: true,
          fingerprint: ['idempotency-conflict', key, this.instanceId],
        });
      } else {
        logger.debug('Duplicate event skipped by idempotency key', {
          label: 'Media Request',
          key,
          fingerprint: ['event-deduplicated', 'afterUpdate', String(event.entity.id)],
        });
      }
      return;
    }

    try {
      // 事件处理...
      await this.sendToRadarr(event.entity);
    } finally {
      await this.idempotencyManager.releaseIdempotencyKey(key, this.instanceId);
    }
  }
}
```

**竞态冲突类型与处理策略**:

| 冲突类型 | 触发场景 | 检测方式 | 处理策略 | Sentry 指纹 |
|---------|---------|---------|----------|------------|
| 真冲突（不同实例） | 两个实例同时接收到同一事件 | SETNX 返回失败 + 不同 owner | 直接跳过，由获取到键的实例处理 | `['idempotency-conflict', key, localInstanceId]` |
| 瞬时冲突（同实例重试） | 网络超时导致重试 | SETNX 失败但 TTL 很短 | 等待 TTL 过期后重试（最多 3 次） | `['idempotency-retry', key, 'attempt-'+attempt]` |
| 死键冲突（实例崩溃） | 获取到键的实例崩溃未释放 | SETNX 失败 + TTL 很长但实例已离线 | 等待 TTL 自动过期（最长 5min） | `['idempotency-dead-key', key, ownerInstanceId]` |
| 长时间冲突 | 某个实例长时间持有键不释放 | SETNX 持续失败 > 3 次 | 强制覆盖旧键（需配置 `forceOverride: true`） | `['idempotency-force-override', key, previousOwner]` |

**幂等键命名空间**:
```
// 完整键格式
jellyseerr:idem:<scope>:<entityType>:<entityId>:<eventHash>

// 示例
jellyseerr:idem:event:media_request:12345:afterUpdate:PENDING→APPROVED
jellyseerr:idem:scan:media:6789:processMovie:full
jellyseerr:idem:rollback:media:5555:sendToRadarr:ECONNREFUSED
```

**冲突监控指标**:
```typescript
// Prometheus 风格指标（建议暴露）
idempotency_acquired_total{instance="instance-a"} 12345
idempotency_conflict_total{type="cross-instance"} 67
idempotency_retry_total{attempts="1"} 42
idempotency_force_override_total 2
idempotency_dead_key_detected_total 1
```

---

## 八、关键设计模式总结

### 8.1 异步非阻塞设计
- 发送到 *arr 服务使用 Promise.then().catch()，不阻塞请求响应
- 通知发送 fire-and-forget，不等待结果
- 扫描任务分批处理，避免长时间阻塞
- **故障切换决策**: Rate-Limit 故障由 client 侧 catch 驱动，不依赖 Redis Sentinel
- **5 级故障切换决策树**: 429 队列等待 → ECONNREFUSED 标记 FAILED → 4xx 不重试 → 5xx 指数退避

### 8.2 事件驱动架构
- TypeORM 的 `@AfterInsert` / `@AfterUpdate` / `@AfterRemove` 钩子
- `EventSubscriber` 监听实体变化，触发后续流程
- **事件回放**: 利用 `event.databaseEntity` 与 `event.entity` 对比检测状态变更
- **事务边界**: 删除操作在 `beforeRemove` 中使用传入的 `manager` 保证原子性
- **回放幂等键**: 事件类型 + 实体 ID + updatedAt 时间戳 + 状态变更哈希，三层去重（Global Redis → Instance Map → DB Constraints）
- **跨进程幂等冲突**: Redis SETNX 原子操作解决 TOCTOU 竞态条件，4 类冲突处理策略 + 命名空间规范

### 8.3 幂等性设计
- 重复请求检查 (`DuplicateMediaRequestError`)
- 状态变更前检查当前状态，避免重复标记
- AsyncLock 防止同一媒体并发操作
- **防循环调用**: afterUpdate 中先校验状态再处理，避免级联更新触发死循环
- **事件回放幂等**: `getIdempotencyKey()` + `isDuplicateEvent()` 60 秒去重窗口
- **状态变更幂等矩阵**: 6 种典型状态转移的允许/拒绝决策表
- **SETNX 原子幂等**: 跨实例事件去重，`tryAcquireIdempotencyKey` + `acquireWithRetry` 3 次重试
- **4K 回填幂等**: `last4kBackfillAt` 标记 + 24h 窗口防止重复执行

### 8.4 容错机制
- 多级错误捕获 (API 调用层 + 实体保存层)
- 失败状态标记 + 通知 + 重试机制
- 缓存失效时返回旧数据 (`getRolling` 策略)
- **5 级回滚优先级**: P0 删除 → P1 拒绝 → P2 API 失败 → P3 连接错误 → P4 重试
- **Sentry 告警键关联**: 每级回滚绑定唯一 fingerprint 模板 + Level 自动映射 + PagerDuty 升级策略
- **4K 降回中间态保护**: 三元表达式条件赋值，处理中保留外部服务 ID 不被误清
- **Sentry 多层去重窗口**: 8 类告警优先级 × 4 层去重架构（L1 应用内 → L2 SDK → L3 服务端 → L4 Alert Rule）
- **智能去重窗口**: 基于事件频率自适应调整（高频 1h → 低频 5min）
- **漂移降级回滚**: 5 类降级场景 × 明确回滚条件 + 4 条保护边界

### 8.5 扩展性设计
- 通知 Agent 接口抽象，易于新增通知渠道
- 扫描器基类 `BaseScanner` 可扩展新的扫描源
- 媒体服务器类型通过枚举支持 (Plex/Jellyfin/Emby)
- **10 种通知 Agent 标准化扩展**: BaseAgent 抽象 + NotificationAgent 接口 + 位掩码类型过滤
- **Agent 插件热加载**: chokidar 文件监听 + require.cache 清除 + dispose 生命周期 + 接口校验 + 版本兼容性检查
- **ABI 版本兼容矩阵**: v1.0-v1.3 4 级版本 × Adapter 模式自动适配 + semver 范围检查
- **数据库配置迁移**: SQL `json_set` + `json_extract` 批量升级旧版本配置

### 8.6 背压与流控
- 扫描器三层背压: Session ID 隔离 → 分批节流 → 可取消标志
- AvailabilitySync 异步生成器分页，内存占用稳定
- axios-rate-limit 控制外部 API 调用频率
- **AsyncLock 死锁预防**: setImmediate 异步释放 + 无界监听器 + 非递归设计
- **四级熔断水位线**: IDLE → NORMAL → WARN(10% 超时, ×1.5) → CRITICAL(30% 超时, ×3) → FATAL(50% 超时, 全局终止)
- **硬阈值快速熔断**: DB 错误 ≥5 次、API 429 ≥15 次直接升级 FATAL
- **健康检查 API**: 暴露 watermark、backoffMultiplier、stats 等实时监控指标
- **动态水位调整**: EWMA 平滑消费速率 + 5 档速率比 × 动态阈值 + 自适应退避乘数
- **前 10 批基准线**: 冷启动阶段不调整，建立基线后动态优化

### 8.7 数据一致性
- **4K 双轨迁移**: 临时表重建 + 历史数据无缝迁移 + 回滚方案
- **AvailabilitySync 漂移补偿**: 双重校验 (*arr + 媒体服务器) + 处理中保护 + 季级粒度 + TMDB 兜底
- **状态保护**: 处理中请求保留外部服务元数据，不盲目清空
- **4K→1080p 中间态矩阵**: 5 种典型场景 × 6 种状态机转移路径，全覆盖无盲区
- **漂移超 24h 兜底降级**: 3 类漂移场景（PROCESSING 超时 / PARTIALLY_AVAILABLE 卡壳 / DELETED 误标）+ 7 级状态降级决策表
- **扫描器心跳检查**: 30 分钟无 heartbeat 自动 cancel + 重启调度
- **4K 回填策略**: 4 级回填优先级 + 分批处理 + DRY_RUN 模式 + 报告审计
- **跨时区时钟漂移**: NTP 3 服务器校准 + 5s 阈值 + 30s 启动阻塞 + TTL 自适应调整
- **漂移降级回滚**: 2 类回滚场景（PROCESSING → 恢复中 / AVAILABLE → 文件存在）+ 24h 幂等保护

### 8.8 分布式考量
- Rate-Limit 当前为内存实现，分布式部署需 Redis 集中限流
- AsyncLock 为进程内锁，多实例部署需分布式锁 (Redis Redlock)
- 通知发送无重复投递保证，需消费方幂等处理
- 扫描任务无分布式协调，多实例部署可能重复执行
- **Redis 令牌桶改造**: Lua 脚本原子令牌获取 + SET NX PX 自动过期 + 本地 50ms 批量同步
- **Redlock 死锁识别**: 30s 锁超时 + 强制释放日志 + Sentry fingerprint 聚合
- **全局幂等键同步**: Redis SETNX 300s TTL 覆盖多实例事件去重
- **跨实例锁诊断 API**: getDeadlockReport() 输出持锁堆栈、预警级别、PID 等信息
- **Sentinel vs Client 决策**: 单实例用 client 重试（低复杂度），多实例用 Redis Sentinel（高可用，10s 故障转移）
- **实例级时钟同步**: NTP 强制校时 + UTC 时区配置 + 偏移监控告警
- **跨进程冲突监控**: Prometheus 指标暴露（acquired/conflict/retry/dead_key）
