---
name: "v5-quartz-job"
description: "Develop V5/A8 scheduled task plugins based on QuartzJob. Use when user needs to generate a plugin that supports fixed-time execution, interval execution, QuartzHolder registration, a QuartzJob.execute business entry, and a controller for manual execution."
---

# V5 定时任务插件开发 Skill（QuartzJob + QuartzHolder）

本 skill 用于生成致远 V5/A8 客开定时任务插件。参考 `kkMeetingSync` 插件模式：

- 通过 `SystemInitializer` 在系统启动时注册定时任务
- 通过 `QuartzHolder` 支持两种执行方式：
  1. **指定时间执行定时任务**
  2. **每隔多长时间执行一次任务**
- 实现 `QuartzJob` 的类是定时任务核心类
- `execute(Map<String, String> map)` 是实际业务执行入口
- 生成插件时，`execute()` 方法先保留空实现，不写具体业务逻辑，等待后续指令补充
- Controller 层用于手动执行定时任务，方便联调或临时触发
- 类命名规则：插件名驼峰式 + 对应功能，例如：`{插件名驼峰}QuartzJob`
- `jobBeanId` 必须等于 Spring XML 中声明的 QuartzJob bean 的 `id`

---

## Part 1: 插件结构

以插件名 `{pluginName}` 为例，推荐结构如下：

```text
apps-customize/
└── src/
    └── main/
        ├── java/
        │   └── com/seeyon/apps/{pluginName}/
        │       ├── {PluginNameCamel}Initializer.java
        │       ├── controller/
        │       │   └── {PluginNameCamel}Controller.java
        │       └── quartzJob/
        │           └── {PluginNameCamel}QuartzJob.java
        └── webapp/
            └── WEB-INF/
                └── cfgHome/
                    └── plugin/
                        └── {pluginName}/
                            ├── pluginCfg.xml
                            ├── pluginProperties.xml
                            └── spring/
                                └── spring-{pluginName}-plugin.xml
```

如需手动执行 Controller 免登录访问，可额外生成：

```text
apps-customize/src/main/resources/
├── needless_check_login_{pluginName}.xml
└── needless_check_login_recheck_{pluginName}.xml
```

---

## Part 2: 命名规则

### 2.1 类命名

插件下类的命名规则是：**插件名驼峰式 + 对应功能**。

例如插件名为 `kkMeetingSync`：

| 功能 | 类名 |
|---|---|
| 初始化类 | `KkMeetingSyncInitializer` 或沿用项目风格 `MeetingSyncInitializer` |
| 定时任务核心类 | `KkMeetingSyncQuartzJob` / `MeetingSyncQuartzJob` |
| 手动执行 Controller | `KkMeetingSyncController` / `MeetingSyncCompletionController` |

新生成插件时优先使用标准规则：

```text
{PluginNameCamel}Initializer
{PluginNameCamel}QuartzJob
{PluginNameCamel}Controller
```

示例：

```text
pluginName = kkArchiveSync
PluginNameCamel = KkArchiveSync

KkArchiveSyncInitializer
KkArchiveSyncQuartzJob
KkArchiveSyncController
```

### 2.2 Bean ID 命名

QuartzJob 的 bean id 推荐：

```text
{pluginName}QuartzJob
```

例如：

```xml
<bean id="kkArchiveSyncQuartzJob" class="com.seeyon.apps.kkArchiveSync.quartzJob.KkArchiveSyncQuartzJob"/>
```

Initializer 中必须保持一致：

```java
public static final String jobBeanId = "kkArchiveSyncQuartzJob";
```

> **重要规则：`jobBeanId` 取值必须等于 Spring XML 中 QuartzJob 类 bean 的 `id`。**

---

## Part 3: Initializer — 系统启动注册定时任务

Initializer 实现 `SystemInitializer`，系统启动时注册 Quartz 任务。

```java
package com.seeyon.apps.{pluginName};

import com.seeyon.ctp.common.AppContext;
import com.seeyon.ctp.common.SystemInitializer;
import com.seeyon.ctp.common.quartz.QuartzHolder;
import com.seeyon.ctp.util.Strings;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.util.Calendar;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

public class {PluginNameCamel}Initializer implements SystemInitializer {

    private static final Log log = LogFactory.getLog({PluginNameCamel}Initializer.class);

    public static final String JOB_NAME = "{PluginNameCamel}Job";

    /**
     * jobBeanId 必须与 spring-{pluginName}-plugin.xml 中 QuartzJob bean 的 id 保持一致。
     */
    public static final String jobBeanId = "{pluginName}QuartzJob";

    /**
     * 任务模式从 pluginProperties.xml 读取：specifiedTime = 指定时间执行；interval = 间隔执行。
     * 生成插件时，根据用户输入的模式，只在 pluginProperties.xml 中生成该模式需要的配置项。
     */
    private static final String RUN_MODE = AppContext.getSystemProperty("{pluginName}.runMode");

    @Override
    public void initialize() {
        log.info("【客开Log】{pluginName}定时任务初始化开始");
        try {
            if (QuartzHolder.hasQuartzJob(JOB_NAME)) {
                QuartzHolder.deleteQuartzJob(JOB_NAME);
            }

            String runMode = getRunMode();
            if ("interval".equals(runMode)) {
                int intervalHour = getIntProperty("{pluginName}.intervalHour", 0);
                int intervalMin = getIntProperty("{pluginName}.intervalMin", 5);
                setInterval(intervalHour, intervalMin);
            } else {
                int hour = getIntProperty("{pluginName}.hour", 3);
                int minute = getIntProperty("{pluginName}.minute", 0);
                setSpecifiedTime(hour, minute);
            }
        } catch (Exception e) {
            log.error("【客开Log】{pluginName}定时任务初始化异常：" + e.getMessage(), e);
            throw new RuntimeException(e);
        }
    }

    private String getRunMode() {
        if (Strings.isBlank(RUN_MODE)) {
            return "specifiedTime";
        }
        return RUN_MODE;
    }

    private int getIntProperty(String key, int defaultValue) {
        String value = AppContext.getSystemProperty(key);
        if (Strings.isBlank(value)) {
            return defaultValue;
        }
        try {
            return Integer.parseInt(value);
        } catch (Exception e) {
            log.error("【客开Log】定时任务配置项不是有效数字，key=" + key + "，value=" + value, e);
            return defaultValue;
        }
    }

    /**
     * 方式一：每隔指定时间执行一次。
     */
    private void setInterval(int intervalHour, int intervalMin) {
        try {
            int delta = (intervalHour * 60 + intervalMin) * 60 * 1000;
            Map<String, String> params = new HashMap<String, String>();
            QuartzHolder.newQuartzJob(JOB_NAME, new Date(), delta, jobBeanId, params);
            log.info("【客开Log】{pluginName}间隔执行定时任务已启动，间隔：" + intervalHour + "小时" + intervalMin + "分钟");
        } catch (Exception e) {
            log.error("【客开Log】{pluginName}间隔执行定时任务初始化失败", e);
            throw new RuntimeException(e);
        }
    }

    /**
     * 方式二：每天指定时间执行一次。
     */
    private void setSpecifiedTime(int hour, int minute) {
        try {
            Calendar calendar = Calendar.getInstance();
            calendar.set(Calendar.HOUR_OF_DAY, hour);
            calendar.set(Calendar.MINUTE, minute);
            calendar.set(Calendar.SECOND, 0);
            Date date = calendar.getTime();

            Map<String, String> params = new HashMap<String, String>();
            QuartzHolder.newQuartzJobPerDay(null, JOB_NAME, date, null, jobBeanId, params);
            log.info("【客开Log】{pluginName}指定时间定时任务已启动，每天 " + hour + ":" + minute + " 执行");
        } catch (Exception e) {
            log.error("【客开Log】{pluginName}指定时间定时任务初始化失败", e);
            throw new RuntimeException(e);
        }
    }
}
```

### 3.1 间隔执行

使用：

```java
QuartzHolder.newQuartzJob(JOB_NAME, new Date(), delta, jobBeanId, params);
```

含义：

| 参数 | 说明 |
|---|---|
| `JOB_NAME` | Quartz 任务名称，需唯一 |
| `new Date()` | 第一次执行时间 |
| `delta` | 间隔毫秒数 |
| `jobBeanId` | Spring 中 QuartzJob bean 的 id |
| `params` | 传给 `execute(Map<String,String>)` 的参数 |

### 3.2 指定时间执行

使用：

```java
QuartzHolder.newQuartzJobPerDay(null, JOB_NAME, date, null, jobBeanId, params);
```

常用场景：每天凌晨、每天固定时分执行。

---

## Part 4: QuartzJob — 定时任务核心类

实现 `com.seeyon.ctp.common.quartz.QuartzJob`。

> 生成 skill 或插件模板时，`execute()` 方法中**不要添加具体业务内容**，只保留日志和 TODO，后续根据用户指令补充业务逻辑。

```java
package com.seeyon.apps.{pluginName}.quartzJob;

import com.seeyon.ctp.common.quartz.QuartzJob;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.util.Map;

public class {PluginNameCamel}QuartzJob implements QuartzJob {

    private static final Log log = LogFactory.getLog({PluginNameCamel}QuartzJob.class);

    @Override
    public void execute(Map<String, String> map) {
        log.info("【客开Log】开始执行{pluginName}定时任务");

        // TODO 后续根据具体业务指令补充定时任务逻辑。
        // 注意：生成插件模板时，此处不要预置业务代码。

        log.info("【客开Log】{pluginName}定时任务执行结束");
    }
}
```

---

## Part 5: Controller — 手动执行定时任务

Controller 用于手动执行定时任务，便于测试、联调、补偿执行。

```java
package com.seeyon.apps.{pluginName}.controller;

import com.seeyon.apps.{pluginName}.quartzJob.{PluginNameCamel}QuartzJob;
import com.seeyon.ctp.common.controller.BaseController;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.web.servlet.ModelAndView;

import javax.inject.Inject;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

public class {PluginNameCamel}Controller extends BaseController {

    private static final Log log = LogFactory.getLog({PluginNameCamel}Controller.class);

    @Inject
    private {PluginNameCamel}QuartzJob {pluginName}QuartzJob;

    public ModelAndView execute(HttpServletRequest request, HttpServletResponse response) throws Exception {
        log.info("【客开Log】进入{pluginName}手动执行Controller");

        Map<String, String> params = new HashMap<String, String>();
        String type = request.getParameter("type");
        if (type != null) {
            params.put("type", type);
        }

        {pluginName}QuartzJob.execute(params);
        render(response, "执行完成");
        return null;
    }

    private void render(HttpServletResponse response, String text) {
        response.setContentType("application/json;charset=UTF-8");
        try {
            response.setContentLength(text.getBytes("UTF-8").length);
            response.getWriter().write(text);
        } catch (IOException e) {
            log.error("【客开Log】响应输出失败", e);
        }
    }
}
```

手动执行 URL 推荐：

```text
/{pluginName}.do?method=execute
```

或按项目 Spring MVC 风格：

```text
/{pluginName}.do
```

以 XML 中注册的 controller bean name 为准。

---

## Part 6: Spring XML

`spring-{pluginName}-plugin.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans default-autowire="byName">

    <bean id="{pluginName}QuartzJob" class="com.seeyon.apps.{pluginName}.quartzJob.{PluginNameCamel}QuartzJob"/>
    <bean id="{pluginName}Initializer" class="com.seeyon.apps.{pluginName}.{PluginNameCamel}Initializer"/>
    <bean name="/{pluginName}.do" class="com.seeyon.apps.{pluginName}.controller.{PluginNameCamel}Controller"/>

</beans>
```

关键点：

```xml
<bean id="{pluginName}QuartzJob" class="...{PluginNameCamel}QuartzJob"/>
```

必须和 Initializer 中保持一致：

```java
public static final String jobBeanId = "{pluginName}QuartzJob";
```

---

## Part 7: pluginCfg.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plugin>
    <id>{pluginName}</id>
    <name>{插件中文名}</name>
    <category>{yyyyMMddNN}</category>
</plugin>
```

示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plugin>
    <id>kkMeetingSync</id>
    <name>会议同步</name>
    <category>2025111506</category>
</plugin>
```

---

## Part 8: pluginProperties.xml

`RUN_MODE` 和对应运行时间参数必须配置在 `pluginProperties.xml` 中。

生成插件时，先根据用户输入判断运行模式，再生成对应配置项：

| 用户输入 | runMode | 生成的时间配置项 |
|---|---|---|
| 指定时间执行，例如“每天 3 点执行” | `specifiedTime` | `hour`、`minute` |
| 每隔多长时间执行，例如“每隔 5 分钟执行” | `interval` | `intervalHour`、`intervalMin` |

> 不要在同一个模板中同时强制生成两套时间参数。用户选择指定时间模式，就生成 `hour/minute`；用户选择间隔模式，就生成 `intervalHour/intervalMin`。

### 8.1 指定时间模式配置

当用户要求“指定时间执行定时任务”时，生成：

```xml
<?xml version="1.0" encoding="utf-8"?>
<ctpConfig>
    <{pluginName}>
        <runMode mark="{VE}" desc="定时任务模式：specifiedTime指定时间，interval间隔执行">specifiedTime</runMode>
        <hour mark="{VE}" desc="指定时间执行：小时，0-23">3</hour>
        <minute mark="{VE}" desc="指定时间执行：分钟，0-59">0</minute>
    </{pluginName}>
</ctpConfig>
```

对应 Initializer 读取：

```java
String runMode = AppContext.getSystemProperty("{pluginName}.runMode");
String hour = AppContext.getSystemProperty("{pluginName}.hour");
String minute = AppContext.getSystemProperty("{pluginName}.minute");
```

### 8.2 间隔执行模式配置

当用户要求“每隔多长时间执行任务”时，生成：

```xml
<?xml version="1.0" encoding="utf-8"?>
<ctpConfig>
    <{pluginName}>
        <runMode mark="{VE}" desc="定时任务模式：specifiedTime指定时间，interval间隔执行">interval</runMode>
        <intervalHour mark="{VE}" desc="间隔执行：间隔小时数">0</intervalHour>
        <intervalMin mark="{VE}" desc="间隔执行：间隔分钟数">5</intervalMin>
    </{pluginName}>
</ctpConfig>
```

对应 Initializer 读取：

```java
String runMode = AppContext.getSystemProperty("{pluginName}.runMode");
String intervalHour = AppContext.getSystemProperty("{pluginName}.intervalHour");
String intervalMin = AppContext.getSystemProperty("{pluginName}.intervalMin");
```

### 8.3 生成规则

- 如果用户说“每天几点”“指定时间”“凌晨/上午/下午某个时间”，使用 `specifiedTime`。
- 如果用户说“每隔 N 分钟/小时”“间隔执行”“周期执行”，使用 `interval`。
- 如果用户没有说明运行模式，先询问用户，不要默认同时生成两套配置。
- `RUN_MODE` 必须从 `pluginProperties.xml` 读取，不要在 Java 中写死。
- 对应模式不需要的时间参数不要生成，避免配置含义混乱。

---

## Part 9: 免登录白名单（可选）

如果 Controller 需要未登录访问，则生成白名单。

`needless_check_login_{pluginName}.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
    <bean>
        <id>/{pluginName}.do</id>
        <methods>
            <method>execute</method>
        </methods>
    </bean>
</beans>
```

`needless_check_login_recheck_{pluginName}.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
    <bean>
        <id>/{pluginName}.do</id>
        <name>com.seeyon.apps.{pluginName}.controller.{PluginNameCamel}Controller</name>
        <methods>
            <method>execute</method>
        </methods>
    </bean>
</beans>
```

---

## Part 10: 生成插件时的必做清单

1. 创建插件 Java 包：`com.seeyon.apps.{pluginName}`
2. 创建 `{PluginNameCamel}Initializer`，实现 `SystemInitializer`
3. 创建 `{PluginNameCamel}QuartzJob`，实现 `QuartzJob`
4. `execute(Map<String,String> map)` 只保留空业务模板，不写具体业务逻辑
5. 创建 `{PluginNameCamel}Controller`，用于手动执行定时任务
6. 创建 `pluginCfg.xml`
7. 创建 `pluginProperties.xml`
8. 创建 `spring-{pluginName}-plugin.xml`
9. 确认 XML 中 QuartzJob bean id 与 Initializer 中 `jobBeanId` 完全一致
10. 根据用户输入的运行模式生成 `pluginProperties.xml`：指定时间模式只生成 `runMode/hour/minute`；间隔模式只生成 `runMode/intervalHour/intervalMin`
11. 如 Controller 需免登录访问，生成两份 `needless_check_login` XML
12. 如已有同名任务，Initializer 中先 `QuartzHolder.deleteQuartzJob(JOB_NAME)` 再重新注册

---

## Part 11: 关键规则总结

| # | 规则 |
|---|---|
| 1 | 定时任务核心类必须实现 `QuartzJob` |
| 2 | `execute(Map<String,String> map)` 是业务逻辑入口 |
| 3 | 生成模板时 `execute()` 不写具体业务，等待后续指令补充 |
| 4 | Controller 用于手动执行定时任务 |
| 5 | 插件类命名：插件名驼峰式 + 功能，例如 `{PluginNameCamel}QuartzJob` |
| 6 | `jobBeanId` 必须等于 Spring XML 中 QuartzJob bean 的 `id` |
| 7 | 指定时间执行用 `QuartzHolder.newQuartzJobPerDay(...)` |
| 8 | 间隔执行用 `QuartzHolder.newQuartzJob(...)` |
| 9 | 初始化时先判断并删除已有同名任务，避免重复注册 |
| 10 | 日志使用 `LogFactory` 或项目已有日志风格 |

---

## Part 12: 参考实现

参考插件：`kkMeetingSync`

关键文件：

- `apps-customize/src/main/java/com/seeyon/apps/kkMeetingSync/MeetingSyncInitializer.java`
- `apps-customize/src/main/java/com/seeyon/apps/kkMeetingSync/quartzJob/MeetingSyncQuartzJob.java`
- `apps-customize/src/main/java/com/seeyon/apps/kkMeetingSync/controller/MeetingSyncCompletionController.java`
- `apps-customize/src/main/webapp/WEB-INF/cfgHome/plugin/kkMeetingSync/spring/spring-kkMeetingSync-plugin.xml`

其中：

```java
public static final String jobBeanId = "meetingSyncQuartzJob";
```

对应 XML：

```xml
<bean id="meetingSyncQuartzJob" class="com.seeyon.apps.kkMeetingSync.quartzJob.MeetingSyncQuartzJob"/>
```

这两个值必须保持一致。
