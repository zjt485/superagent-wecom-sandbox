# 健康检查接口说明（Health Check API）

本文档描述本仓库/服务对外提供的健康检查接口设计，用于负载均衡探活、容器编排（Kubernetes）存活与就绪探针、以及监控系统的可用性巡检。

## 1. 设计目标

- **轻量**：不依赖业务逻辑，不触发下游重调用，响应时间应在毫秒级。
- **分层**：区分「进程存活」与「依赖就绪」，避免依赖抖动导致实例被误杀。
- **可观测**：返回结构化 JSON，便于监控系统解析与告警。
- **安全**：不泄露敏感信息（密钥、连接串、内部地址等）。

## 2. 接口列表

| 路径 | 方法 | 用途 | 是否检查依赖 |
| --- | --- | --- | --- |
| `/healthz` | GET | 存活探针（liveness）：进程是否存活 | 否 |
| `/readyz` | GET | 就绪探针（readiness）：是否可接收流量 | 是 |
| `/health` | GET | 综合健康检查：返回各依赖明细 | 是 |

> 约定：`/healthz` 与 `/readyz` 遵循 Kubernetes 探针惯例，成功返回 `200`，失败返回 `503`。

## 3. 请求约定

- 方法：`GET`
- 无需鉴权（探针路径应在网关/鉴权中间件中放行）
- 无需请求体
- 可选查询参数：
  - `verbose=true`：返回各依赖项的详细检查结果（仅 `/health` 支持）
- 建议超时：客户端 2s，服务端内部依赖检查单项超时 1s

## 4. 响应格式

### 4.1 成功响应（200）

```json
{
  "status": "ok",
  "version": "1.4.2",
  "uptimeSeconds": 3612,
  "timestamp": "2025-01-01T08:00:00Z",
  "checks": {
    "database": { "status": "ok", "latencyMs": 3 },
    "redis": { "status": "ok", "latencyMs": 1 },
    "wecomApi": { "status": "ok", "latencyMs": 42 }
  }
}
```

### 4.2 失败响应（503）

```json
{
  "status": "degraded",
  "version": "1.4.2",
  "uptimeSeconds": 3612,
  "timestamp": "2025-01-01T08:00:00Z",
  "checks": {
    "database": { "status": "ok", "latencyMs": 3 },
    "redis": { "status": "error", "message": "connection timeout" }
  }
}
```

### 4.3 字段说明

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `status` | string | `ok` / `degraded` / `error` |
| `version` | string | 当前服务版本号（来自构建信息） |
| `uptimeSeconds` | number | 进程已运行秒数 |
| `timestamp` | string | 服务端 UTC 时间（RFC3339） |
| `checks` | object | 各依赖项检查结果，键为依赖名 |
| `checks.*.status` | string | `ok` / `error` / `skipped` |
| `checks.*.latencyMs` | number | 检查耗时（毫秒） |
| `checks.*.message` | string | 失败原因（仅失败时返回，不含敏感信息） |

## 5. 各探针语义

### 5.1 `/healthz`（liveness）

仅检查进程自身是否可响应 HTTP 请求，不访问任何外部依赖。

- 正常：`200`，`{"status":"ok"}`
- 异常：进程无响应或事件循环阻塞，由编排系统重启实例

### 5.2 `/readyz`（readiness）

检查服务是否具备接收业务流量的条件，包括：

1. 数据库连接可用（执行轻量 `SELECT 1`）
2. 缓存（Redis）连接可用（`PING`）
3. 企微 API 凭证有效（可选，建议缓存结果，避免高频调用）

任一关键依赖失败即返回 `503`，实例从负载均衡中摘除，但不重启。

### 5.3 `/health`（综合）

返回全部依赖明细，供监控与人工排查使用。支持 `verbose=true` 输出每项检查的耗时与错误信息。

## 6. 缓存与限流

- 依赖检查结果缓存 5 秒，避免探针高频调用打爆下游。
- 单 IP 限流：60 次/分钟，超出返回 `429`。
- 探针路径不写入访问日志正文，仅记录状态码与耗时。

## 7. 状态码汇总

| 状态码 | 含义 |
| --- | --- |
| `200` | 健康 / 就绪 |
| `429` | 请求过于频繁 |
| `503` | 依赖不可用，服务降级 |

## 8. 监控与告警建议

- 采集 `/health` 的 `status` 与各 `checks.*.status`，连续 3 次 `degraded` 触发告警。
- 采集 `checks.*.latencyMs`，P99 超过阈值（如数据库 200ms、企微 API 1s）时告警。
- 存活探针间隔 10s、失败阈值 3；就绪探针间隔 5s、失败阈值 2。

## 9. 安全注意事项

- 响应中不得包含数据库连接串、密钥、内网 IP 等敏感信息。
- 生产环境建议仅在内网或通过网关白名单暴露探针路径。
- `verbose=true` 建议仅对运维角色开放。
