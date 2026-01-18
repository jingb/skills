# 函数声明设计指南

## 核心原则

**函数参数和返回值应该直接反映业务语义，而非技术实现细节。**

函数声明是代码接口的核心，好的声明能让调用者一目了然地知道"需要什么"、"得到什么"。

## 设计原则

### 1. 参数直接体现业务含义

**问题：将 HTTP 请求等容器对象作为参数，隐藏了真实的数据需求**

#### Java 示例

❌ **不推荐 - 耦合了 HTTP 协议**
```java
// 调用者无法一眼看出这个函数需要什么数据
String getPhoneLocation(HttpServletRequest request) {
    String json = request.getParameter("phone");
    long phone = parsePhone(json);
    return queryLocation(phone);
}
```

✅ **推荐 - 语义清晰**
```java
// 一眼就能看出需要什么参数，返回什么结果
String getPhoneLocation(long phoneNumber) {
    return queryLocation(phoneNumber);
}
```

#### Python 示例

❌ **不推荐 - 耦合了 HTTP 协议**
```python
# 无法看出函数需要什么参数
def get_phone_location(request):
    phone = parse_phone(request.json.get('phone'))
    return query_location(phone)
```

✅ **推荐 - 语义清晰**
```python
# 清晰表达输入和输出
def get_phone_location(phone_number: int) -> str:
    return query_location(phone_number)
```

---

### 2. 参数类型应该明确

#### Java 示例

❌ **不推荐 - 泛型容器，类型不明确**
```java
// Map 里面有什么？key 是什么？value 是什么？
User createUser(Map<String, Object> params) {
    String name = (String) params.get("name");
    int age = (Integer) params.get("age");
    // ...
}
```

✅ **推荐 - 类型明确**
```java
// 清晰表达每个参数的含义
User createUser(String name, int age, String email) {
    // ...
}
```

#### Python 示例

❌ **不推荐 - 使用 Dict，类型不明确**
```python
def create_user(params: dict) -> User:
    name = params.get('name')
    age = params.get('age')
    # ...
```

✅ **推荐 - 类型明确（使用 dataclass 或具名参数）**
```python
def create_user(name: str, age: int, email: str) -> User:
    # ...
```

---

### 3. 参数数量应该合理

**原则：函数参数不宜过多，通常不超过 4-5 个**

#### Java 示例

❌ **不推荐 - 参数过多**
```java
User createUser(String name, int age, String email, String phone,
                String address, String city, String country, String zip) {
    // ...
}
```

✅ **推荐 - 使用参数对象**
```java
class CreateUserParams {
    String name;
    int age;
    String email;
    String phone;
    String address;
    String city;
    String country;
    String zip;
}

User createUser(CreateUserParams params) {
    // ...
}
```

#### Python 示例

❌ **不推荐 - 参数过多**
```python
def create_user(name: str, age: int, email: str, phone: str,
                address: str, city: str, country: str, zip_code: str) -> User:
    # ...
```

✅ **推荐 - 使用 dataclass**
```python
from dataclasses import dataclass

@dataclass
class CreateUserParams:
    name: str
    age: int
    email: str
    phone: str
    address: str
    city: str
    country: str
    zip_code: str

def create_user(params: CreateUserParams) -> User:
    # ...
```

---

### 4. 返回值应该有意义

#### Java 示例

❌ **不推荐 - 返回值不明确**
```java
// 返回 true/false，不知道成功还是失败
boolean createUser(String name, int age) {
    // 成功返回 true？还是其他含义？
}
```

✅ **推荐 - 返回明确的结果对象或抛出异常**
```java
// 选项 1: 返回 User 对象，失败抛异常
User createUser(String name, int age) throws UserAlreadyExistsException {
    // ...
}

// 选项 2: 返回 Result 对象
Result<User> createUser(String name, int age) {
    // ...
}
```

#### Python 示例

❌ **不推荐 - 返回值不明确**
```python
def create_user(name: str, age: int):
    # 返回 True？还是 User？不清楚
    pass
```

✅ **推荐 - 明确返回类型**
```python
def create_user(name: str, age: int) -> User:
    """创建用户，失败时抛出 UserAlreadyExistsException"""
    # ...
```

---

### 5. 不要耦合无关的技术层

**原则：函数应该只依赖它真正需要的东西，而不是整个技术栈**

#### Java 示例

❌ **不推荐 - 耦合了 Servlet**
```java
// 这个函数只是查询数据，为什么需要 HttpServletRequest？
List<User> searchUsers(HttpServletRequest request) {
    String keyword = request.getParameter("keyword");
    return userDao.search(keyword);
}
```

✅ **推荐 - 解耦**
```java
// 只需要真正的业务参数
List<User> searchUsers(String keyword) {
    return userDao.search(keyword);
}
```

#### Python 示例

❌ **不推荐 - 耦合了 Flask**
```python
# 函数只是计算，为什么需要 Flask Request 对象？
def calculate_discount(request):
    price = request.args.get('price')
    return float(price) * 0.9
```

✅ **推荐 - 解耦**
```python
def calculate_discount(price: float) -> float:
    return price * 0.9
```

---

## 检查清单

在设计函数声明时，问自己以下问题：

- [ ] 函数的参数名称是否清晰表达其用途？
- [ ] 参数类型是否明确，不是泛型容器（Map, Dict, Object）？
- [ ] 参数数量是否合理（≤ 5 个）？
- [ ] 参数是否直接反映业务含义，而非技术实现细节？
- [ ] 返回值是否明确表达成功/失败的结果？
- [ ] 函数是否耦合了无关的技术层（HTTP、数据库等）？

---

## 反面模式

| 模式 | 问题 | 替代方案 |
|------|------|---------|
| `void do(Map params)` | 语义不明 | 使用具名参数或参数对象 |
| `Result foo(Request req)` | 耦合 HTTP 层 | 直接传业务参数 |
| `boolean bar()` | 返回值模糊 | 返回具体对象或抛异常 |
| `List<Thing> baz(String a, int b, String c, boolean d, float e, ...)` | 参数过多 | 使用参数对象 |
