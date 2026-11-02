# JWT：从登录令牌到身份校验

这是一篇结合当前“苍穹外卖”项目的 JWT 入门教程。目标不是背 API，而是理解一个请求从登录、拿到 Token、携带 Token，到后端校验身份的完整过程。

## 1. 先理解为什么需要身份凭证

HTTP 请求彼此独立。用户第一次请求登录接口时提交了账号密码，但第二次查询员工信息时，服务器不会天然知道这两个请求来自同一个人。

因此，登录成功后需要给客户端一个身份凭证：

```text
第一次请求：账号 + 密码 → 登录成功 → 服务器签发凭证
后续请求：携带凭证       → 服务器验证 → 确认用户身份
```

这个“凭证”统称为 Token。JWT 是 Token 的一种实现格式。

> Token 是一个大类，JWT 是其中一种。苍穹外卖中的 `token` 变量保存的正是 JWT 字符串。

## 2. JWT 是什么

JWT 全称 JSON Web Token。它把声明数据和防篡改签名组合成一个字符串，通常长这样：

```text
xxxxx.yyyyy.zzzzz
```

中间两个点把它分为三段：

```text
Header.Payload.Signature
```

### 2.1 Header：说明如何签名

Header 通常是：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

- `alg` 表示签名算法。
- `typ` 表示令牌类型。

本项目没有手写 Header，因为 JJWT 会根据下面的代码自动生成：

```java
SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;

builder.signWith(
        signatureAlgorithm,
        secretKey.getBytes(StandardCharsets.UTF_8)
);
```

### 2.2 Payload：存放声明

Payload 是 JWT 携带的数据，也叫 Claims（声明）。本项目放入了员工 ID：

```java
Map<String, Object> claims = new HashMap<>();
claims.put(JwtClaimsConstant.EMP_ID, employee.getId());
```

生成后的 Payload 概念上类似：

```json
{
  "empId": 1,
  "exp": 1788074728
}
```

- `empId` 是项目自定义声明。
- `exp` 是标准声明，表示过期时间。

Payload 只是 Base64URL 编码，不是加密。不要往里面放密码、身份证号、银行卡号或服务端秘钥。

### 2.3 Signature：证明令牌没被篡改

使用 HS256 时，可以把签名过程简化理解为：

```text
Signature = HMAC-SHA256(
    Base64Url(Header) + "." + Base64Url(Payload),
    secretKey
)
```

客户端如果把 `empId: 1` 改成 `empId: 999`，Payload 会变化，但客户端不知道 `secretKey`，无法生成正确的新签名。后端校验签名时就会拒绝这个 Token。

重要区别：

- Payload 可以被看到。
- Payload 被修改后会被发现。
- `secretKey` 只保存在服务器，绝不能放进 JWT 或前端代码。

## 3. 本项目的完整登录流程

### 第一步：前端提交账号密码

浏览器向 Nginx 发送：

```http
POST /api/employee/login
Content-Type: application/json

{"username":"admin","password":"123456"}
```

Nginx 将它转发给后端：

```text
/api/employee/login
        ↓
/admin/employee/login
```

### 第二步：Controller 调用业务层验证账号密码

代码位置：

```text
sky-server/src/main/java/com/sky/controller/admin/EmployeeController.java
```

核心代码：

```java
Employee employee = employeeService.login(employeeLoginDTO);
```

业务层查询数据库，并检查账号是否存在、密码是否正确、账号是否启用。

### 第三步：登录成功后生成 JWT

```java
Map<String, Object> claims = new HashMap<>();
claims.put(JwtClaimsConstant.EMP_ID, employee.getId());

String token = JwtUtil.createJWT(
        jwtProperties.getAdminSecretKey(),
        jwtProperties.getAdminTtl(),
        claims
);
```

三个参数分别表示：

| 参数 | 含义 |
|---|---|
| `adminSecretKey` | 服务端签名秘钥 |
| `adminTtl` | Token 有效时长，单位毫秒 |
| `claims` | 需要携带的业务数据，这里是员工 ID |

### 第四步：JWT 在响应体中返回

```java
EmployeeLoginVO employeeLoginVO = EmployeeLoginVO.builder()
        .id(employee.getId())
        .userName(employee.getUsername())
        .name(employee.getName())
        .token(token)
        .build();

return Result.success(employeeLoginVO);
```

响应大致为：

```json
{
  "code": 1,
  "data": {
    "id": 1,
    "userName": "admin",
    "name": "管理员",
    "token": "eyJhbGciOiJIUzI1NiJ9..."
  }
}
```

注意：登录时 JWT 位于 HTTP 响应体，不在响应头。

### 第五步：前端把 JWT 保存到 Cookie

课程前端使用 `js-cookie`：

```typescript
const tokenKey = 'token'

export const getToken = () => Cookies.get(tokenKey)
export const setToken = (token: string) => Cookies.set(tokenKey, token)
```

登录成功后：

```typescript
setToken(data.data.token)
```

可以在浏览器开发者工具查看：

```text
Application → Cookies → http://localhost:8081 → token
```

### 第六步：后续请求把 Token 放进请求头

前端请求拦截器执行：

```typescript
config.headers['token'] = UserModule.token
```

于是后续请求类似：

```http
GET /api/employee/page?page=1&pageSize=10
token: eyJhbGciOiJIUzI1NiJ9...
```

这里要区分：

- 登录响应：Token 在响应体。
- 后续请求：Token 在请求头。
- Cookie 是前端持久化 Token 的地方。

## 4. `JwtUtil.createJWT()` 是怎样工作的

代码位置：

```text
sky-common/src/main/java/com/sky/utils/JwtUtil.java
```

简化后的代码：

```java
public static String createJWT(
        String secretKey,
        long ttlMillis,
        Map<String, Object> claims) {

    SignatureAlgorithm algorithm = SignatureAlgorithm.HS256;

    long expMillis = System.currentTimeMillis() + ttlMillis;
    Date exp = new Date(expMillis);

    JwtBuilder builder = Jwts.builder()
            .setClaims(claims)
            .signWith(algorithm, secretKey.getBytes(StandardCharsets.UTF_8))
            .setExpiration(exp);

    return builder.compact();
}
```

逐句理解：

```java
SignatureAlgorithm.HS256
```

选择 HS256 算法，JJWT 会据此生成 Header 中的 `alg`。

```java
System.currentTimeMillis() + ttlMillis
```

计算绝对过期时间。例如当前时间是 10:00，有效期是 2 小时，则过期时间是 12:00。

```java
.setClaims(claims)
```

将 `empId` 等自定义数据写入 Payload。

```java
.setExpiration(exp)
```

将标准声明 `exp` 写入 Payload。

```java
.signWith(algorithm, secretKeyBytes)
```

使用算法和秘钥计算 Signature。

```java
.compact()
```

完成编码、签名和拼接，最终返回 `Header.Payload.Signature`。

## 5. 后端怎样校验 JWT

代码位置：

```text
sky-server/src/main/java/com/sky/interceptor/JwtTokenAdminInterceptor.java
```

### 5.1 从请求头读取 Token

```java
String token = request.getHeader(
        jwtProperties.getAdminTokenName()
);
```

配置中的请求头名称是：

```yaml
sky:
  jwt:
    admin-token-name: token
```

因此实际读取的是名为 `token` 的请求头。

### 5.2 解析并验证

```java
Claims claims = JwtUtil.parseJWT(
        jwtProperties.getAdminSecretKey(),
        token
);
```

工具类内部执行：

```java
Claims claims = Jwts.parser()
        .setSigningKey(secretKey.getBytes(StandardCharsets.UTF_8))
        .parseClaimsJws(token)
        .getBody();
```

`parseClaimsJws()` 不只是“把字符串拆开”，还会验证：

- JWT 格式是否正确。
- 签名是否能用同一个秘钥验证。
- Token 是否已经过期。

任何一步失败都会抛出异常。

### 5.3 取出员工 ID

```java
Long empId = Long.valueOf(
        claims.get(JwtClaimsConstant.EMP_ID).toString()
);
```

这就是登录时放进去的员工 ID。

### 5.4 放行或拒绝

```java
try {
    // 解析成功
    return true;
} catch (Exception ex) {
    response.setStatus(401);
    return false;
}
```

- 返回 `true`：允许进入 Controller。
- 返回 `false`：结束请求并返回 HTTP 401。

当前学习阶段的拦截器只取出了 `empId` 并打印日志，还没有将它保存到 `BaseContext`。后续学习公共字段自动填充时，会继续使用“当前员工 ID”这个信息。

## 6. 为什么登录接口不需要 Token

如果登录接口也要求 Token，就会出现死循环：

```text
想登录 → 必须有 Token
想有 Token → 必须先登录
```

所以在拦截器配置中排除登录接口：

```java
registry.addInterceptor(jwtTokenAdminInterceptor)
        .addPathPatterns("/admin/**")
        .excludePathPatterns("/admin/employee/login");
```

含义是：

- `/admin/**` 默认都需要校验 JWT。
- `/admin/employee/login` 例外，直接放行。

## 7. JWT 配置从哪里来

配置文件：

```yaml
sky:
  jwt:
    admin-secret-key: itcast
    admin-ttl: 7200000
    admin-token-name: token
```

配置类：

```java
@Component
@ConfigurationProperties(prefix = "sky.jwt")
@Data
public class JwtProperties {
    private String adminSecretKey;
    private long adminTtl;
    private String adminTokenName;
}
```

Spring Boot 会按“短横线命名 → 驼峰属性”自动绑定：

```text
admin-secret-key → adminSecretKey
admin-ttl        → adminTtl
admin-token-name → adminTokenName
```

`7200000` 毫秒等于：

```text
7,200,000 ÷ 1000 ÷ 60 ÷ 60 = 2 小时
```

## 8. 常见误区

### 误区一：JWT 是加密后的用户数据

不是。JWT 的 Payload 默认可以被解码查看，签名的主要作用是防篡改。

### 误区二：有 JWT 就一定安全

不是。JWT 被别人复制后，在过期前可能被冒用，因此生产环境必须使用 HTTPS，并避免 Token 泄露。

### 误区三：退出登录会让 JWT 立即失效

当前项目的退出接口只是前端删除 Token。已经签发的 JWT 在服务器端仍然可以通过签名校验，直到过期。要做到立即失效，需要黑名单、Token 版本号或更短有效期配合刷新令牌。

### 误区四：所有 Token 都是 JWT

不是。随机字符串、Session ID 和 OAuth Access Token 都可以称为 Token。

### 误区五：秘钥可以放在前端

绝对不可以。前端只保存签发后的 Token，签名秘钥必须仅由后端持有。

## 9. 在 IDEA 中跟读代码

推荐按下面的顺序打断点：

1. `EmployeeController.login()`：观察账号验证结果和 `claims`。
2. `JwtUtil.createJWT()`：观察过期时间、算法和最终 Token。
3. 前端登录后，在浏览器 Cookie 中查看 Token。
4. `JwtTokenAdminInterceptor.preHandle()`：观察请求头 Token。
5. `JwtUtil.parseJWT()`：观察解析后的 `Claims`。

常用快捷键：

| 操作 | macOS IDEA |
|---|---|
| 跳到声明或方法 | `Command + B` |
| 跳到接口实现 | `Option + Command + B` |
| 返回上一位置 | `Command + [` |
| 查找类 | `Command + O` |
| 查找使用位置 | `Option + F7` |

## 10. 动手验证

### 实验一：正常登录

```bash
curl -i \
  -X POST 'http://localhost:8081/api/employee/login' \
  -H 'Content-Type: application/json' \
  --data-raw '{"username":"admin","password":"123456"}'
```

观察响应体中的 `data.token`。

### 实验二：不带 Token 访问受保护接口

选择一个 `/admin/**` 且不是登录的接口，不携带 `token` 请求头，预期返回 HTTP 401。

### 实验三：携带正确 Token

```bash
curl -i 'http://localhost:8081/api/employee/page?page=1&pageSize=10' \
  -H 'token: 替换成登录返回的JWT'
```

预期通过拦截器并进入 Controller。

### 实验四：篡改 Token

修改 JWT 中任意一个字符后再次请求，签名校验应失败并返回 401。

### 实验五：缩短有效期

仅在学习环境中把 `admin-ttl` 临时改为 `10000`（10 秒），登录后等待 10 秒再请求，观察过期后的 401。实验结束后恢复配置。

## 11. 学完后应该能回答

1. Token 和 JWT 是什么关系？
2. JWT 的三部分分别是什么？
3. 为什么 Payload 能看到但不能随意修改？
4. 本项目 Token 在登录响应的哪里？
5. 前端把 Token 保存在哪里？
6. 后续请求把 Token 放在哪里？
7. 后端哪个拦截器负责校验？
8. 为什么登录接口必须排除在拦截器外？
9. `secretKey` 和 Token 有什么区别？
10. 当前项目退出登录为什么不能让已签发 JWT 立即失效？

如果这些问题都能用自己的话说明，就已经掌握了本项目当前阶段所需的 JWT 知识。

