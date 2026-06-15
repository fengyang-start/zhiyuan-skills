# 第三方API集成模式

基于致远远程系统集成的实战总结。

## 通用HTTP客户端封装

### 模式1：OkHttp（推荐，V10.0SP1+）
```java
public class HttpClientUtil {
    private static final OkHttpClient client = new OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build();

    public static String postJson(String url, String json, Map<String, String> headers) {
        RequestBody body = RequestBody.create(json, MediaType.parse("application/json"));
        Request.Builder builder = new Request.Builder().url(url).post(body);
        if (headers != null) headers.forEach(builder::addHeader);
        try (Response response = client.newCall(builder.build()).execute()) {
            return response.body().string();
        }
    }
}
```

### 模式2：Apache HttpClient（兼容V8.x/V9.0SP1）
```java
public class HttpKit {
    private static final PoolingHttpClientConnectionManager cm =
        new PoolingHttpClientConnectionManager();
    static { cm.setMaxTotal(200); cm.setDefaultMaxPerRoute(20); }

    public static String post(String url, Map<String, String> headers, String body) {
        CloseableHttpClient client = HttpClients.custom()
            .setConnectionManager(cm).build();
        // ...execute and return
    }
}
```

### 模式3：HttpURLConnection（轻量兼容）
```java
public class HttpKit {
    private static final int CONNECT_TIMEOUT = 10000;
    private static final int READ_TIMEOUT = 10000;

    public static InputStream post(String urlStr, String postData) {
        URL url = new URL(urlStr);
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setConnectTimeout(CONNECT_TIMEOUT);
        conn.setReadTimeout(READ_TIMEOUT);
        // ...
    }
}
```

## Token管理模式

### 双重检查锁缓存（推荐）
```java
public class ApiClient {
    private volatile String token;
    private volatile long tokenExpireTime;
    private final ReentrantLock lock = new ReentrantLock();

    public String getToken() {
        if (token == null || System.currentTimeMillis() > tokenExpireTime) {
            lock.lock();
            try {
                if (token == null || System.currentTimeMillis() > tokenExpireTime) {
                    Map<String, Object> resp = fetchToken();
                    token = (String) resp.get("access_token");
                    tokenExpireTime = System.currentTimeMillis() +
                        ((Number) resp.get("expires_in")).longValue() * 1000 - 60000;
                }
            } finally { lock.unlock(); }
        }
        return token;
    }
}
```

### Hutool TimedCache
```java
private static final TimedCache<String, String> tokenCache =
    new TimedCache<>(30 * 60 * 1000);  // 30分钟过期

public static String getToken() {
    String token = tokenCache.get("api_token");
    if (token == null) {
        token = fetchNewToken();
        tokenCache.put("api_token", token);
    }
    return token;
}
```

## 签名与加密

### 国密SM4/SM3
```java
public class Sm4Utils {
    // SM4加密请求Body
    public static String encrypt(String plainText, String key) { ... }
    // SM3签名
    public static String sign(String data, String key) { ... }
}
```

### AES加密（SSO票据）
```java
public class TicketUtils {
    // 密钥从pluginProperties.xml读取
    // 票据格式: loginName|timestamp|specialKey
    public static String encrypt(String loginName, String key) { ... }
    public static String[] decrypt(String ticket, String key) { ... }
}
```

### MD5签名
```java
public class MD5SignUtil {
    public static String sign(Map<String, String> params, String secret) {
        // 1. 参数排序
        // 2. 拼接key=value&key=value
        // 3. MD5加密
    }
}
```

## 外部系统接口调用模式

### XML接口（如银企直连）
```java
public class XmlApiService {
    public Response callService(RequestPo po) {
        String xml = buildXml(po);
        String result = HttpClient.postXml(apiUrl, xml);
        return parseXmlResponse(result);
    }

    private String buildXml(RequestPo po) {
        return "<root><head><tr_code>BIZ001</tr_code>" +
               "<erp_syscode>" + erpSysCode + "</erp_syscode>" +
               "<cust_no>" + custNo + "</cust_no></head>" +
               "<body>" + buildBodyXml(po) + "</body></root>";
    }
}
```

### JSON REST API（通用第三方REST系统）
```java
public class JsonApiClient {
    // 1. 登录获取session/token
    public String login(String url, String user, String pwd) { ... }
    // 2. POST JSON
    public String postJson(String url, String json) { ... }
    // 3. GET JSON
    public String getJson(String url) { ... }
}
```

### 用友系API（BIP/友空间）
```java
public class BipSSOController extends BaseController {
    public ModelAndView sso(HttpServletRequest req, HttpServletResponse res) {
        String token = generateToken(user);
        String targetUrl = bipBaseUrl + "/service/sso?token=" + token;
        return redirectModelAndView(targetUrl);
    }
}
```

## SSO单点登录集成

### 握手模式（SSOLoginHandshakeAbstract）
```java
public class ThirdPartySSOLoginHandshake extends SSOLoginHandshakeAbstract {
    @Override
    public LoginUser doLogin(String ticket, SSOLoginInfo info) {
        // 1. 验证ticket（调用第三方验证接口）
        // 2. 返回LoginUser
    }
}
```

### Controller跳转模式
```java
public class SSOController extends BaseController {
    @NeedlessCheckLogin
    public ModelAndView sso(HttpServletRequest req, HttpServletResponse res) {
        String ticket = req.getParameter("ticket");
        String loginName = TicketValidator.validate(ticket);
        return redirectModelAndView("/login/sso?from=third&ticket=" + encrypt(loginName));
    }
}
```

## 异步推送与补偿

### 事件驱动推送 + 补偿任务
```
@ListenEvent → 创建同步任务(状态=PENDING) → 推送执行
  ├── 成功 → SUCCESS
  ├── 可重试失败 → FAILED_RETRYABLE → 定时补偿扫描
  └── 不可重试失败 → FAILED_FINAL → 人工干预
```

### 补偿定时任务
```java
public class PendingRetryJob extends Thread {
    private volatile boolean running = true;
    private static final long SCAN_INTERVAL = 60 * 1000;

    @Override
    public void run() {
        while (running) {
            try {
                List<Task> failedTasks = taskDao.findRetryable();
                for (Task task : failedTasks) {
                    dispatchService.retry(task);
                }
                Thread.sleep(SCAN_INTERVAL);
            } catch (InterruptedException e) { break; }
        }
    }
}
```

### 并发控制
```java
// 原子状态更新，防止重复推送
public boolean markSending(String taskId) {
    String sql = "UPDATE sync_task SET status='SENDING' " +
                 "WHERE id=? AND status='PENDING'";
    int updated = jdbc.executeUpdate(sql, new Object[]{taskId});
    return updated > 0;
}
```

## 返回值模式

### 超级节点返回值
```java
SuperNodeResponse(1)  // WAIT — 成功，等待继续流转
SuperNodeResponse(2)  // BACK — 失败，回退节点
SuperNodeResponse(0)  // 暂存
```

### Manager/Service通用响应
```java
public class Response {
    private int returnCode;  // 1=成功, 2=回退, 0=暂存
    private String message;
    private Object data;
}
```

### REST接口响应
```java
// 标准成功响应
{ "code": 0, "message": "success", "data": {...} }
// 标准失败响应
{ "code": -1, "message": "错误描述", "data": null }
```
