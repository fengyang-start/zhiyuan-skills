# v5短信集成插件SKILL

该skill用于生成V5平台的短信集成插件代码，参考`sms028lk`插件的写法模式，遵循客开前置规范。

---

## 一、参考规范

生成代码时必须遵循`v5客开前置skill.md`中的通用写法规范，包括：
- JSON处理：使用`com.alibaba.fastjson.JSON`和`com.alibaba.fastjson.JSONObject`
- 日期格式化：使用`com.seeyon.ctp.util.DateUtil`
- Map处理：使用`com.seeyon.ctp.util.ParamUtil`

---

## 二、目录结构

```
apps-customize/
└── src/
    └── main/
        ├── java/
        │   └── com/seeyon/apps/{pluginName}/
        │       ├── HttpKit.java                                    # HTTP工具类（从sms028lk复制）
        │       ├── {PluginName}AdapterMobileMessageManagerImpl.java # 短信发送实现
        │       └── MobileTest{PluginName}.java                     # 测试类
        └── webapp/
            └── WEB-INF/
                └── cfgHome/
                    └── plugin/
                        └── {pluginName}/
                            ├── spring/
                            │   └── spring-smsmas-api.xml           # Spring Bean注册
                            ├── pluginCfg.xml                        # 插件元数据
                            └── pluginProperties.xml                 # 短信接口配置参数
```

---

## 三、短信接口说明

短信插件实现`AdapterMobileMessageManger`接口，V5平台通过该接口调用第三方短信网关发送短信。

| 方法 | 说明 | 返回值 |
|------|------|--------|
| `isAvailability()` | 短信服务是否可用 | `boolean` |
| `getName()` | 短信集成名称（显示在管理界面） | `String` |
| `sendMessage(Long messageId, String srcPhone, String destPhone, String content)` | 发送单条短信 | `boolean` |
| `sendMessage(Long messageId, String srcPhone, Collection<String> destPhones, String content)` | 群发短信 | `boolean` |
| `isSupportQueueSend()` | 是否支持队列发送 | `boolean` |
| `isSupportRecive()` | 是否支持接收短信 | `boolean` |
| `recive()` | 接收短信 | `List<MobileReciver>` |

---

## 四、命名规则

| 要素 | 规则 |
|------|------|
| 包名 | `com.seeyon.apps.{pluginName}` |
| 短信实现类 | `{PluginName}AdapterMobileMessageManagerImpl` |
| 测试类 | `MobileTest{PluginName}` |
| HTTP工具类 | `HttpKit`（从sms028lk直接复制） |

**`getAppName()`**: `ApplicationCategoryEnum.global`

### 命名示例

假设插件名为 `smsmydemo`：
- 短信实现类：`SmsmydemoAdapterMobileMessageManagerImpl`
- 测试类：`MobileTestSmsmydemo`
- `getName()` 返回：`"【我的短信集成】"`

---

## 五、代码模板

### 5.1 HttpKit.java — HTTP工具类

直接从 `com.seeyon.apps.sms028lk.HttpKit` 复制，无需修改。提供了 GET/POST/PUT 等方法，支持 JSON 参数和自定义 Header，内置SSL免证书验证。

### 5.2 {PluginName}AdapterMobileMessageManagerImpl.java — 短信实现类

```java
package com.seeyon.apps.{pluginName};

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.seeyon.ctp.common.AppContext;
import com.seeyon.ctp.common.log.CtpLogFactory;
import com.seeyon.ctp.organization.manager.OrgManager;
import com.seeyon.v3x.mobile.adapter.AdapterMobileMessageManger;
import com.seeyon.v3x.mobile.message.domain.MobileReciver;
import org.apache.commons.logging.Log;

import java.util.Collection;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class {PluginName}AdapterMobileMessageManagerImpl implements AdapterMobileMessageManger {

    private final Log log = CtpLogFactory.getLog({PluginName}AdapterMobileMessageManagerImpl.class);

    private final String url = AppContext.getSystemProperty("{pluginName}.sendUrl");
    private final String secretName = AppContext.getSystemProperty("{pluginName}.secretName");
    private final String secretKey = AppContext.getSystemProperty("{pluginName}.secretKey");
    private final String templateId = AppContext.getSystemProperty("{pluginName}.templateId");
    private final String signName = AppContext.getSystemProperty("{pluginName}.signName");
    private final String extCode = AppContext.getSystemProperty("{pluginName}.extCode");

    private OrgManager orgManager;

    public void setOrgManager(OrgManager orgManager) {
        this.orgManager = orgManager;
    }

    @Override
    public boolean isAvailability() {
        return true;
    }

    @Override
    public String getName() {
        return "{smsName}";
    }

    @Override
    public boolean sendMessage(Long messageId, String srcPhone, String destPhone, String content) {
        if (log.isInfoEnabled()) {
            log.info("{smsName}发送短消息, 源号码:" + srcPhone + ", 目标号码:" + destPhone + ", 内容:" + content);
        }
        try {
            Map<String, Object> map = new LinkedHashMap<>();
            map.put("SecretName", secretName);
            map.put("SecretKey", secretKey);
            map.put("TimeStamp", System.currentTimeMillis());
            map.put("Mobile", destPhone);
            map.put("Content", content);

            if (templateId != null && !templateId.isEmpty()) {
                map.put("TemplateId", templateId);
            }
            if (signName != null && !signName.isEmpty()) {
                map.put("SignName", signName);
            }
            if (extCode != null && !extCode.isEmpty()) {
                map.put("ExtCode", extCode);
            }

            String param = JSON.toJSONString(map);
            log.info("【客开Log】发送短消息请求体: " + param);
            String result = HttpKit.post(url, param);
            log.info("【客开Log】发送短消息结果: " + result);

            JSONObject jsonObject = JSONObject.parseObject(result);
            if (jsonObject.getInteger("code") != null && jsonObject.getInteger("code") == 0) {
                log.info("【客开Log】发送短消息成功, data: " + jsonObject.getString("data"));
            } else {
                log.info("【客开Log】发送短消息失败, code: " + jsonObject.getInteger("code")
                        + ", msg: " + jsonObject.getString("msg"));
            }
            return true;
        } catch (Exception e) {
            log.error("【客开Log】发送短消息失败", e);
        }
        return true;
    }

    @Override
    public boolean sendMessage(Long messageId, String srcPhone, Collection<String> destPhones, String content) {
        for (String destPhone : destPhones) {
            if (sendMessage(messageId, srcPhone, destPhone, content)) {
                return true;
            }
        }
        return false;
    }

    @Override
    public boolean isSupportQueueSend() {
        return true;
    }

    @Override
    public boolean isSupportRecive() {
        return false;
    }

    @Override
    public List<MobileReciver> recive() {
        return null;
    }
}
```

### 5.3 MobileTest{PluginName}.java — 测试类

```java
package com.seeyon.apps.{pluginName};

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;

import java.util.LinkedHashMap;
import java.util.Map;

public class MobileTest{PluginName} {

    private static final String url = "{testUrl}";
    private static final String secretName = "{testSecretName}";
    private static final String secretKey = "{testSecretKey}";
    private static final String templateId = "{testTemplateId}";
    private static final String signName = "{testSignName}";

    public static void main(String[] args) throws Exception {
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("SecretName", secretName);
        map.put("SecretKey", secretKey);
        map.put("TimeStamp", System.currentTimeMillis());
        map.put("Mobile", "13000000000");
        map.put("Content", "hello world");

        if (templateId != null && !templateId.isEmpty()) {
            map.put("TemplateId", templateId);
        }
        if (signName != null && !signName.isEmpty()) {
            map.put("SignName", signName);
        }

        String param = JSON.toJSONString(map);
        System.out.println("请求体: " + param);
        String result = HttpKit.post(url, param);
        System.out.println("发送短消息结果：" + result);

        JSONObject jsonObject = JSONObject.parseObject(result);
        if (jsonObject.getInteger("code") != null && jsonObject.getInteger("code") == 0) {
            System.out.println("发送短消息成功");
            System.out.println("data: " + jsonObject.getString("data"));
        } else {
            System.out.println("发送短消息失败, code: " + jsonObject.getInteger("code")
                    + ", msg: " + jsonObject.getString("msg"));
        }
    }
}
```

---

## 六、插件配置文件

### 6.1 pluginCfg.xml — 插件元数据

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plugin>
    <id>{pluginName}</id>
    <name>{pluginDescription}</name>
    <category>{date}01</category>
</plugin>
```

| 字段 | 说明 |
|------|------|
| `id` | 插件名（小写） |
| `name` | 插件显示名称，用户输入 |
| `category` | 生成日期 + `01`，如 `2026051301` |

### 6.2 pluginProperties.xml — 短信接口配置参数

所有调用参数统一在此文件中配置，Java代码通过 `AppContext.getSystemProperty("{pluginName}.{key}")` 读取。

```xml
<?xml version="1.0" encoding="utf-8"?>
<ctpConfig>
    <{pluginName}>
        <sendUrl mark="{VE}" desc="{sendUrlDesc}">{sendUrl}</sendUrl>
        <secretName mark="{VE}" desc="{secretNameDesc}">{secretName}</secretName>
        <secretKey mark="{VE}" desc="{secretKeyDesc}">{secretKey}</secretKey>
        <templateId mark="{VE}" desc="{templateIdDesc}">{templateId}</templateId>
        <signName mark="{VE}" desc="{signNameDesc}">{signName}</signName>
        <extCode mark="{VE}" desc="{extCodeDesc}">{extCode}</extCode>
    </{pluginName}>
</ctpConfig>
```

根据需要可增减配置项。`mark="{VE}"` 表示该配置可在平台界面中修改。

### 6.3 spring/spring-smsmas-api.xml — Spring Bean注册

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans default-autowire="byName">
    <bean id="{pluginName}AdapterMobileMessageManagerImpl" class="com.seeyon.apps.{pluginName}.{PluginName}AdapterMobileMessageManagerImpl"/>
</beans>
```

`bean id` 使用小写插件名 + `AdapterMobileMessageManagerImpl`。

---

## 七、生成规则

### 7.1 输入参数

用户需要提供以下参数：

| 参数 | 说明 | 示例 |
|------|------|------|
| `pluginName` | 插件名/短信供应商标识（小写） | `sms028lk` |
| `pluginDescription` | 插件显示名称 | `028LK短信集成` |
| `smsName` | `getName()` 返回值，显示在短信管理界面 | `【028LK短信集成】` |
| `sendUrl` | 短信接口URL | `https://api.028lk.com/Sms/Api/Send` |
| `secretName` | API密钥账号 | `API` |
| `secretKey` | API密钥 | `000000` |
| `templateId` | 模板ID | `000000` |
| `signName` | 短信签名 | `【签名】` |
| `extCode` | 扩展码 | 空 |

### 7.2 生成步骤

1. 在 `apps-customize/src/main/java/com/seeyon/apps/` 下创建 `{pluginName}` 目录
2. 从 `com.seeyon.apps.sms028lk` 复制 `HttpKit.java` 到 `{pluginName}` 目录（修改包名）
3. 生成 `{PluginName}AdapterMobileMessageManagerImpl.java`：
   - 实现 `AdapterMobileMessageManger` 接口
   - 配置参数通过 `AppContext.getSystemProperty("{pluginName}.{key}")` 读取
   - `getName()` 返回用户输入的 `{smsName}`
4. 生成 `MobileTest{PluginName}.java`（可选）
5. 在 `apps-customize/src/main/webapp/WEB-INF/cfgHome/plugin/` 下创建 `{pluginName}` 目录
6. 在 `{pluginName}` 下创建 `spring` 目录
7. 生成 `pluginCfg.xml`：`id`=`{pluginName}`，`name`=`{pluginDescription}`，`category`=当天日期+`01`
8. 生成 `pluginProperties.xml`：配置短信接口的 `sendUrl`、`secretName`、`secretKey`、`templateId`、`signName`、`extCode`
9. 生成 `spring/spring-smsmas-api.xml`：注册短信Bean

---

## 八、关键注意事项

1. **接口实现**：实现 `AdapterMobileMessageManger` 接口（位于 `com.seeyon.v3x.mobile.adapter` 包）
2. **配置读取**：所有调用参数通过 `AppContext.getSystemProperty("{pluginName}.{key}")` 从 `pluginProperties.xml` 读取
3. **HTTP工具**：`HttpKit.java` 从 sms028lk 直接复制，只需修改包名；支持 `post(url, json)` 发送JSON请求
4. **JSON处理**：短信集成使用 `com.alibaba.fastjson.JSON.toJSONString()` 和 `JSONObject.parseObject()`，不使用 `JSONUtil`
5. **注入方式**：`OrgManager` 使用 setter 方法注入
6. **sendMessage返回值**：无论成功失败都返回 `true`，避免平台重试导致重复发送
7. **日志规范**：使用 `CtpLogFactory.getLog()`，日志前缀用 `【客开Log】`
8. **名称格式**：`getName()` 建议使用 `【XXXX】` 格式以在管理界面中醒目显示