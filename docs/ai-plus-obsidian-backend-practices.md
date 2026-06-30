# AI + Obsidian 管理后端多依赖服务的工程实践

## 核心思路

```
Obsidian 笔记（知识层）
    ↓ 每次任务开始前读取
AGENTS.md（约束层）→ 告诉 AI 去哪里读、用什么模式
    ↓
AI 生成代码（执行层）
    ↓ 任务完成后
更新 Obsidian 笔记（反馈层）
```

---

## 一、每依赖库建一份「API 解剖笔记」

在 `wiki/concepts/` 下，每个第三方库一个 `.md` 文件：

```md
# Redis (ioredis v5.4)

## 安装
- 版本: `ioredis@5.4.1`
- 原因: 原生 Cluster / Sentinel 支持，Pipeline 性能最优

## 初始化模式
```typescript
// ✅ 正确模式（连接池复用）
const redis = new Redis({ host, port, retryStrategy: (t) => Math.min(t * 50, 2000) })

// ❌ 不要每次请求 new Redis()
```

## 核心 API 图谱
| 方法 | 参数 | 返回值 | 注意 |
|------|------|--------|------|
| get(key) | string | string \| null | 不存在返回 null，不是 undefined |
| set(key, val, 'EX', ttl) | string, string \| Buffer, ... | 'OK' | TTL 单位秒 |

## 集成模式
- 与 BullMQ 配合: 直接用 Redis 实例，不要 new 第二个
- 与 Express 配合: 挂到 req.app.locals.redis

## 已知陷阱
- ❌ Pipeline 中 exec 后不要再用每条结果的 callback（双重回调）
- ❌ Cluster 模式下 KEYS 命令不可用，用 SCAN 替代

## 测试策略
- 用 `ioredis-mock` 做单元测试
- 集成测试用 Docker compose 起真实 Redis
```

每个笔记必须包含三要素：✅ 正确示例、❌ 错误示例、🪤 已知陷阱。

---

## 二、AGENTS.md 做硬约束

```md
## Pre-flight
Before writing any code involving a third-party library:
1. Read the library's anatomy note from Obsidian
2. State which APIs you plan to use and why
3. Confirm no known traps apply

## How to read Obsidian
Use:
curl -skH "Authorization: Bearer ${OBSIDIAN_TOKEN}" \
  "https://127.0.0.1:27124/vault/wiki/concepts/库名.md"

## Strict Boundaries (NEVER DO)
- Never guess API signatures — always reference the note
- Never use deprecated methods listed in notes
```

---

## 三、Prompt 模板

```markdown
我要实现 [XXX 功能]，涉及 [库A、库B、库C]。
在写代码之前，请：

1. 读取 Obsidian 中这三个库的解剖笔记：
   curl -skH "..." "https://.../库A.md"
   curl -skH "..." "https://.../库B.md"

2. 提炼出关键 API 和陷阱，列个清单给我确认

3. 确认后再开始实现
```

---

## 四、按任务类型加载知识矩阵

| 任务类型 | 需要加载的笔记 |
|----------|---------------|
| 数据库操作 | PostgreSQL / Prisma / Drizzle |
| 缓存策略 | Redis / Memcached / Cache 模式 |
| 消息队列 | BullMQ / RabbitMQ / Kafka |
| HTTP 服务 | Express / Fastify / Hono / 中间件 |
| 认证鉴权 | Passport / JWT / OAuth2 / Session |
| 部署运维 | Docker / PM2 / Nginx / SSL |

批量拉取：

```bash
for lib in Redis PostgreSQL Nginx; do
  curl -skH "..." "https://127.0.0.1:27124/vault/wiki/concepts/$lib.md"
done
```

---

## 五、反馈闭环

任务完成后，如果发现了笔记中没覆盖的陷阱或新模式，立即更新笔记：

```
我发现这个库有一个坑 / 新用法 → 写入 Obsidian 笔记 → 下次 AI 自动避坑
```

知识在持续积累，比任何提示词工程都有效。

---

## 六、建议工具化

创建一个 opencode skill 做三件事：

1. `/load-libs [数据库 缓存 消息队列]` — 自动从 Obsidian 加载对应笔记
2. `/capture-trap [库名]` — 交互式引导将新陷阱写入笔记
3. `/scaffold-module [库名]` — 根据笔记生成样板代码骨架

---

## 一句话总结

每一个第三方库都有一份解剖笔记 + 每次任务前 AGENTS.md 强制 AI 先读笔记 + 每次任务后更新笔记。三个环节形成正向循环，精度越来越高。