# 监控埋点指南

## 核心原则

**在关键操作和性能敏感路径埋点，监控成功率、延迟和业务指标。**

有效的监控能帮助及时发现系统问题、优化性能、了解业务健康状况。

## 监控指标类型

### 1. Counter - 计数器

**用途**：记录事件发生的次数，只增不减。

#### Java 示例

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Metrics;

// 请求计数
Counter requestCounter = Counter.builder("api.requests")
    .tag("endpoint", "/api/users")
    .tag("method", "GET")
    .tag("status", "200")
    .register(Metrics.globalRegistry);

requestCounter.increment();

// 错误计数
Counter errorCounter = Counter.builder("api.errors")
    .tag("type", "DatabaseConnectionException")
    .tag("service", "user-service")
    .register(Metrics.globalRegistry);

try {
    // 业务逻辑
} catch (Exception e) {
    errorCounter.increment();
    throw e;
}
```

#### Python 示例

```python
from prometheus_client import Counter

# 请求计数
request_counter = Counter(
    'api_requests',
    'API request count',
    ['endpoint', 'method', 'status']
)

request_counter.labels(
    endpoint='/api/users',
    method='GET',
    status='200'
).inc()

# 错误计数
error_counter = Counter(
    'api_errors',
    'API error count',
    ['type', 'service']
)

try:
    # 业务逻辑
except Exception as e:
    error_counter.labels(
        type='DatabaseConnectionException',
        service='user-service'
    ).inc()
    raise
```

---

### 2. Gauge - 快照值

**用途**：记录当前的瞬时值，可增可减。

#### Java 示例

```java
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.Metrics;

// 当前连接数
AtomicInteger connectionCount = new AtomicInteger(0);

Gauge.builder("db.connections.active", connectionCount, AtomicInteger::get)
    .tag("database", "postgres")
    .register(Metrics.globalRegistry);

// 在连接建立和关闭时更新
connectionCount.incrementAndGet();
// ... 使用连接
connectionCount.decrementAndGet();

// 队列长度
Gauge.builder("queue.size", queue, Queue::size)
    .tag("name", "task-queue")
    .register(Metrics.globalRegistry);
```

#### Python 示例

```python
from prometheus_client import Gauge

# 当前连接数
connection_count = Gauge(
    'db_connections_active',
    'Current database connections',
    ['database']
)

# 在连接建立和关闭时更新
connection_count.labels(database='postgres').inc()
# ... 使用连接
connection_count.labels(database='postgres').dec()

# 队列长度
queue_size = Gauge(
    'queue_size',
    'Current queue size',
    ['name']
)

queue_size.labels(name='task-queue').set(len(task_queue))
```

---

### 3. Histogram / Timer - 分布统计

**用途**：记录数值的分布情况，常用于延迟统计。

#### Java 示例

```java
import io.micrometer.core.instrument.Timer;

// HTTP 请求延迟
Timer requestTimer = Timer.builder("api.latency")
    .tag("endpoint", "/api/orders")
    .tag("method", "POST")
    .publishPercentiles(0.5, 0.95, 0.99)  // p50, p95, p99
    .register(Metrics.globalRegistry);

Timer.Sample sample = Timer.start();
try {
    // 处理请求
    createOrder(orderData);
} finally {
    sample.stop(requestTimer);
}

// 或使用 Timer 包装
Timer timer = Timer.builder("operation.duration")
    .tag("operation", "data-processing")
    .register(Metrics.globalRegistry);

timer.record(() -> {
    // 执行操作
    processData();
});
```

#### Python 示例

```python
from prometheus_client import Histogram
import time

# HTTP 请求延迟
request_latency = Histogram(
    'api_latency_seconds',
    'API request latency',
    ['endpoint', 'method']
)

with request_latency.labels(
    endpoint='/api/orders',
    method='POST'
).time():
    # 处理请求
    create_order(order_data)

# 或手动记录
start_time = time.time()
process_data()
duration = time.time() - start_time
request_latency.labels(
    endpoint='/api/orders',
    method='POST'
).observe(duration)
```

---

## 应该埋点的场景

### 1. 外部服务调用

监控第三方服务的可用性、延迟和错误率。

#### Java 示例

```java
public class PaymentService {
    private final Counter paymentSuccess = Counter.builder("payment.success")
        .tag("method", "alipay")
        .register(Metrics.globalRegistry);

    private final Counter paymentFailed = Counter.builder("payment.failed")
        .tag("method", "alipay")
        .register(Metrics.globalRegistry);

    private final Timer paymentLatency = Timer.builder("payment.latency")
        .tag("method", "alipay")
        .publishPercentiles(0.5, 0.95, 0.99)
        .register(Metrics.globalRegistry);

    public PaymentResult processPayment(PaymentRequest request) {
        Timer.Sample sample = Timer.start();
        try {
            PaymentResult result = alipayClient.pay(request);
            paymentSuccess.increment();
            return result;
        } catch (Exception e) {
            paymentFailed.increment();
            throw e;
        } finally {
            sample.stop(paymentLatency);
        }
    }
}
```

#### Python 示例

```python
from prometheus_client import Counter, Histogram

class PaymentService:
    def __init__(self):
        self.payment_success = Counter(
            'payment_success',
            'Payment success count',
            ['method']
        )
        self.payment_failed = Counter(
            'payment_failed',
            'Payment failed count',
            ['method']
        )
        self.payment_latency = Histogram(
            'payment_latency_seconds',
            'Payment latency',
            ['method']
        )

    def process_payment(self, request):
        with self.payment_latency.labels(method='alipay').time():
            try:
                result = self.alipay_client.pay(request)
                self.payment_success.labels(method='alipay').inc()
                return result
            except Exception as e:
                self.payment_failed.labels(method='alipay').inc()
                raise
```

---

### 2. 核心业务操作

记录订单创建、支付、用户注册等关键业务操作。

#### Java 示例

```java
// 订单创建
Counter orderCreated = Counter.builder("order.created")
    .tag("channel", request.getChannel())
    .register(Metrics.globalRegistry);

Timer orderProcessingTime = Timer.builder("order.processing.time")
    .register(Metrics.globalRegistry);

// 订单支付
Counter orderPaid = Counter.builder("order.paid")
    .tag("payment_method", request.getPaymentMethod())
    .register(Metrics.globalRegistry);

Gauge orderAmount = Gauge.builder("order.amount", () -> calculateTotalOrderAmount())
    .register(Metrics.globalRegistry);
```

#### Python 示例

```python
# 订单创建
order_created = Counter(
    'order_created',
    'Order created count',
    ['channel']
)

order_processing_time = Histogram(
    'order_processing_time_seconds',
    'Order processing time'
)

# 订单支付
order_paid = Counter(
    'order_paid',
    'Order paid count',
    ['payment_method']
)

order_amount = Gauge(
    'order_amount',
    'Total order amount'
)

order_amount.set(calculate_total_order_amount())
```

---

### 3. 慢查询和性能瓶颈

监控数据库查询、缓存访问等性能敏感操作。

#### Java 示例

```java
// 数据库查询延迟
Timer dbQueryLatency = Timer.builder("db.query.latency")
    .tag("table", "users")
    .tag("operation", "select")
    .publishPercentiles(0.95, 0.99)
    .register(Metrics.globalRegistry);

public User getUser(long userId) {
    return dbQueryLatency.record(() -> {
        return userRepository.findById(userId);
    });
}

// 慢查询计数
Counter slowQueryCounter = Counter.builder("db.slow.query")
    .tag("table", "orders")
    .tag("threshold_ms", "1000")
    .register(Metrics.globalRegistry);
```

#### Python 示例

```python
# 数据库查询延迟
db_query_latency = Histogram(
    'db_query_latency_seconds',
    'Database query latency',
    ['table', 'operation']
)

def get_user(user_id):
    with db_query_latency.labels(table='users', operation='select').time():
        return user_repository.find_by_id(user_id)

# 慢查询计数
slow_query_counter = Counter(
    'db_slow_query',
    'Slow query count',
    ['table', 'threshold_ms']
)
```

---

### 4. 错误和异常

记录系统错误、业务异常等。

#### Java 示例

```java
// 异常类型统计
Counter exceptionCounter = Counter.builder("exceptions")
    .tag("type", exception.getClass().getSimpleName())
    .tag("component", "order-service")
    .register(Metrics.globalRegistry);

try {
    // 业务逻辑
} catch (Exception e) {
    exceptionCounter.increment();
    throw e;
}
```

#### Python 示例

```python
# 异常类型统计
exception_counter = Counter(
    'exceptions',
    'Exception count',
    ['type', 'component']
)

try:
    # 业务逻辑
except Exception as e:
    exception_counter.labels(
        type=e.__class__.__name__,
        component='order-service'
    ).inc()
    raise
```

---

## 埋点命名规范

### 基本原则

- 使用下划线分隔：`api_request_count`
- 使用单位后缀：`api_latency_seconds`, `request_size_bytes`
- 保持一致性：`*_success`, `*_failed`, `*_latency`

### 示例

| 指标名称 | 说明 |
|---------|------|
| `api_requests_total` | API 请求总数 |
| `api_latency_seconds` | API 延迟（秒） |
| `api_errors_total` | API 错误总数 |
| `db_query_duration_seconds` | 数据库查询持续时间 |
| `cache_hit_ratio` | 缓存命中率 |
| `queue_size` | 队列长度 |
| `memory_usage_bytes` | 内存使用量 |

---

## 检查清单

添加埋点时，问自己以下问题：

- [ ] 是否选择了合适的指标类型？（Counter/Gauge/Histogram）
- [ ] 埋点是否在正确的位置？（开始/结束、成功/失败）
- [ ] 是否添加了必要的标签？（endpoint、method、status 等）
- [ ] 指标命名是否符合规范？
- [ ] 是否考虑了性能影响？（避免过度埋点）
- [ ] 是否有监控告警？（SLA、阈值等）

---

## 反面模式

| 模式 | 问题 | 替代方案 |
|------|------|---------|
| `Counter("request")` | 没有标签 | `Counter("requests", ["endpoint", "status"])` |
| 在每个循环内埋点 | 性能开销 | 批量埋点 |
| `Counter("foo", ["a", "b", "c"])` | 标签基数过大 | 减少标签或高基数标签单独处理 |
| 只有计数没有延迟 | 缺少性能指标 | 同时记录 Counter 和 Timer |
| 埋点但没有告警 | 发现问题不及时 | 配置告警规则 |
