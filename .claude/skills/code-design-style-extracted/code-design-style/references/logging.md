# 日志规范指南

## 核心原则

**日志应该记录有意义的上下文信息，选择合适的日志级别，并采用结构化格式。**

好的日志能帮助快速定位问题、追踪业务流程、监控系统健康状态。

## 日志级别选择

### ERROR - 系统错误、异常情况

**使用场景**：需要立即关注的错误，影响系统功能

#### Java 示例

```java
// 数据库连接失败
try {
    Connection conn = dataSource.getConnection();
} catch (SQLException e) {
    log.error("Failed to connect to database: url={}, error={}", dbUrl, e.getMessage(), e);
}

// 关键业务异常
if (balance < amount) {
    log.error("Insufficient balance: userId={}, balance={}, amount={}", userId, balance, amount);
    throw new InsufficientBalanceException();
}
```

#### Python 示例

```python
# 数据库连接失败
try:
    conn = get_db_connection()
except Exception as e:
    logger.error("Failed to connect to database: url=%s, error=%s", db_url, e, exc_info=True)

# 关键业务异常
if balance < amount:
    logger.error("Insufficient balance: userId=%s, balance=%s, amount=%s", user_id, balance, amount)
    raise InsufficientBalanceException()
```

---

### WARN - 警告但不影响运行

**使用场景**：潜在问题、降级操作、边界情况

#### Java 示例

```java
// 重试操作
if (retryCount > 3) {
    log.warn("Retry limit reached: operation={}, attempts={}", operationName, retryCount);
}

// 使用默认值
int timeout = config.getTimeout();
if (timeout == 0) {
    log.warn("Timeout not configured, using default: operation={}, defaultTimeout={}",
             operationName, DEFAULT_TIMEOUT);
    timeout = DEFAULT_TIMEOUT;
}

// 缓存未命中
String cacheKey = "user:" + userId;
String cachedValue = cache.get(cacheKey);
if (cachedValue == null) {
    log.warn("Cache miss: key={}", cacheKey);
}
```

#### Python 示例

```python
# 重试操作
if retry_count > 3:
    logger.warning("Retry limit reached: operation=%s, attempts=%s", operation_name, retry_count)

# 使用默认值
timeout = config.get('timeout')
if timeout == 0:
    logger.warning("Timeout not configured, using default: operation=%s, defaultTimeout=%s",
                   operation_name, DEFAULT_TIMEOUT)
    timeout = DEFAULT_TIMEOUT

# 缓存未命中
cache_key = f"user:{user_id}"
cached_value = cache.get(cache_key)
if cached_value is None:
    logger.warning("Cache miss: key=%s", cache_key)
```

---

### INFO - 关键业务节点、状态变更

**使用场景**：重要的业务操作、状态变化、启动/停止

#### Java 示例

```java
// 用户登录
log.info("User login: userId={}, ip={}, device={}", userId, ipAddress, deviceType);

// 订单创建
log.info("Order created: orderId={}, userId={}, amount={}, status={}",
         orderId, userId, amount, "PENDING");

// 状态变更
log.info("Order status changed: orderId={}, from={}, to={}",
         orderId, oldStatus, newStatus);

// 系统启动
log.info("Application started: version={}, environment={}, port={}",
         version, environment, port);
```

#### Python 示例

```python
# 用户登录
logger.info("User login: userId=%s, ip=%s, device=%s", user_id, ip_address, device_type)

# 订单创建
logger.info("Order created: orderId=%s, userId=%s, amount=%s, status=%s",
            order_id, user_id, amount, "PENDING")

# 状态变更
logger.info("Order status changed: orderId=%s, from=%s, to=%s",
            order_id, old_status, new_status)

# 系统启动
logger.info("Application started: version=%s, environment=%s, port=%s",
            version, environment, port)
```

---

### DEBUG - 调试信息（生产环境通常关闭）

**使用场景**：详细的调试信息、中间状态、参数详情

#### Java 示例

```java
// 详细参数
log.debug("Processing request: params={}", requestParams);

// 中间状态
log.debug("Step 1: parsed userId={}", userId);
log.debug("Step 2: fetched user={}", user);
log.debug("Step 3: calculated discount={}", discount);

// 数据库查询详情
log.debug("Executing query: sql={}, params={}", sql, queryParams);
```

#### Python 示例

```python
# 详细参数
logger.debug("Processing request: params=%s", request_params)

# 中间状态
logger.debug("Step 1: parsed userId=%s", user_id)
logger.debug("Step 2: fetched user=%s", user)
logger.debug("Step 3: calculated discount=%s", discount)

# 数据库查询详情
logger.debug("Executing query: sql=%s, params=%s", sql, query_params)
```

---

## 日志格式规范

### 结构化日志格式

使用结构化格式（JSON 或 key=value）便于日志分析和查询。

#### Java - JSON 格式

```java
// 推荐使用 logstash-logback-encoder 或类似库
log.info("User action: userId={}, action={}, resource={}, timestamp={}",
         userId, actionType, resourceId, Instant.now());

// JSON 输出示例：
// {"userId":"123","action":"view","resource":"item-456","timestamp":"2025-01-17T10:30:00Z","level":"INFO"}
```

#### Python - 结构化日志

```python
# 使用 python-json-logger 或 structlog
import structlog

logger = structlog.get_logger()

logger.info(
    "user_action",
    user_id=user_id,
    action=action_type,
    resource=resource_id,
    timestamp=datetime.now().isoformat()
)

# JSON 输出示例：
# {"event":"user_action","user_id":"123","action":"view","resource":"item-456","timestamp":"2025-01-17T10:30:00Z","level":"info"}
```

---

### 关键上下文信息

日志必须包含关键上下文，便于追踪和排查问题。

#### Java 示例

```java
// ✅ 包含关键上下文
log.info("Payment processed: orderId={}, userId={}, amount={}, method={}, status={}",
         orderId, userId, amount, paymentMethod, "SUCCESS");

// ❌ 缺少上下文
log.info("Payment processed");

// ✅ 使用 RequestId 追踪请求链路
log.info("API call: requestId={}, endpoint={}, method={}, userId={}",
         requestId, endpoint, httpMethod, userId);
```

#### Python 示例

```python
# ✅ 包含关键上下文
logger.info(
    "Payment processed",
    order_id=order_id,
    user_id=user_id,
    amount=amount,
    method=payment_method,
    status="SUCCESS"
)

# ❌ 缺少上下文
logger.info("Payment processed")

# ✅ 使用 RequestId 追踪请求链路
logger.info(
    "API call",
    request_id=request_id,
    endpoint=endpoint,
    method=http_method,
    user_id=user_id
)
```

---

## 日志记录的最佳实践

### 1. 避免在循环中频繁记录日志

#### Java 示例

❌ **不推荐**
```java
for (Order order : orders) {
    log.info("Processing order: orderId={}", order.getId());
    processOrder(order);
}
```

✅ **推荐**
```java
log.info("Starting batch order processing: count={}", orders.size());
for (Order order : orders) {
    processOrder(order);
}
log.info("Batch order processing completed: count={}", orders.size());
```

#### Python 示例

❌ **不推荐**
```python
for order in orders:
    logger.info("Processing order: orderId=%s", order.id)
    process_order(order)
```

✅ **推荐**
```python
logger.info("Starting batch order processing: count=%s", len(orders))
for order in orders:
    process_order(order)
logger.info("Batch order processing completed: count=%s", len(orders))
```

---

### 2. 敏感信息脱敏

#### Java 示例

```java
// ❌ 不推荐 - 直接记录敏感信息
log.info("User login: userId={}, password={}, card={}",
         userId, password, creditCard);

// ✅ 推荐 - 脱敏处理
log.info("User login: userId={}, password=***, card=****-{}",
         userId, maskPassword(password), maskCard(creditCard));
```

#### Python 示例

```python
# ❌ 不推荐 - 直接记录敏感信息
logger.info("User login: userId=%s, password=%s, card=%s",
            user_id, password, credit_card)

# ✅ 推荐 - 脱敏处理
logger.info("User login: userId=%s, password=***, card=****-%s",
            user_id, mask_password(password), mask_card(credit_card))
```

---

### 3. 异常日志应包含堆栈

#### Java 示例

```java
// ✅ 推荐 - 包含异常对象
try {
    riskyOperation();
} catch (Exception e) {
    log.error("Operation failed: userId={}, operation={}", userId, operationName, e);
    // 最后的 e 参数会自动包含堆栈信息
}
```

#### Python 示例

```python
# ✅ 推荐 - 使用 exc_info=True
try:
    risky_operation()
except Exception as e:
    logger.error(
        "Operation failed: userId=%s, operation=%s",
        user_id, operation_name,
        exc_info=True  # 包含堆栈信息
    )
```

---

## 检查清单

记录日志时，问自己以下问题：

- [ ] 日志级别是否合适？（ERROR/WARN/INFO/DEBUG）
- [ ] 是否包含关键上下文信息？（userId、requestId、资源ID 等）
- [ ] 是否使用了结构化格式？
- [ ] 敏感信息是否已脱敏？
- [ ] 异常日志是否包含堆栈信息？
- [ ] 是否避免了循环中的频繁日志？
- [ ] 日志消息是否清晰易懂？

---

## 反面模式

| 模式 | 问题 | 替代方案 |
|------|------|---------|
| `log.info("Done")` | 缺少上下文 | `log.info("Order created: orderId={}", id)` |
| `log.error(e)` | 没有上下文 | `log.error("Payment failed: orderId={}", id, e)` |
| `for (item: list) { log.info(...); }` | 循环内频繁日志 | 批量记录开始和结束 |
| `log.info("password=" + password)` | 敏感信息泄露 | 脱敏处理 `password=***` |
| `System.out.println(...)` | 使用标准输出 | 使用日志框架 |
