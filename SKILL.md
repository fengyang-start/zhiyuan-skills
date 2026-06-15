---
name: zhiyuanskill-yw
description: 致远CTP V5/V8二次开发完整指南 — 基于客开项目实战模式总结，覆盖V8.2SP1/V9.0SP1/V10.0SP1
---

# 致远CTP二次开发Skill（完整版）

基于对致远CTP客开项目的深度分析，覆盖V8.2SP1/V9.0SP1/V10.0SP1多个版本。

## 工作流程

1. **识别平台版本** — 通过pom.xml父版本号确定（`V8.2SP1`/`V9.0SP1`/`V10.0SP1`），不要依赖目录名
2. **优先低侵入路径** — CIP配置 > 插件化 > REST/OpenAPI > Event监听 > APPS API > 源码修改
3. **保持插件隔离** — 每个插件在`cfgHome/plugin/<id>/`下独立配置，遵循项目已有命名约定
4. **复用项目组件** — 使用项目已有的DBKit/CAP4FormKit/HttpKit等工具类，不重复造轮子
5. **验证编码** — V5老项目中文可能乱码，编辑前检测文件编码

## V5插件标准结构

### 目录布局
```
apps-customize/
├── pom.xml                          # 父POM: apps-root
├── src/main/java/com/seeyon/apps/
│   └── <pluginId>/
│       ├── controller/              # BaseController子类
│       ├── manager/                 # Manager接口+Impl
│       ├── dao/                     # DAO接口+Impl
│       ├── po/                      # PO/VO实体
│       ├── node/                    # 超级节点(BaseSuperNodeAction)
│       ├── event/                   # 事件监听(@ListenEvent)
│       ├── kit/ 或 util/            # 工具类(DBKit/FormCap4Kit/HttpKit)
│       ├── task/                    # 定时任务(QuartzJob)
│       ├── service/                 # Service接口+Impl
│       ├── enums/                   # 枚举常量
│       └── initializer/             # 插件初始化器
└── src/main/webapp/WEB-INF/
    └── cfgHome/
        ├── plugin/<pluginId>/
        │   ├── pluginCfg.xml        # 插件定义
        │   ├── pluginProperties.xml # 业务参数配置
        │   ├── spring/              # Spring XML（*-controller/manager/dao/node.xml）
        │   └── sql/                 # 建表脚本(MySQL/达梦/人大金仓)
        └── component/<componentId>/ # CAP4自定义控件配置
            ├── pluginCfg.xml
            ├── spring/
            └── i18n/
                ├── export_to_js.xml
                └── *_zh_CN.properties
```

### pluginCfg.xml 格式
```xml
<plugin>
    <id>pluginId</id>
    <name>插件名称</name>
    <category>20250101</category>
</plugin>
```

### pluginProperties.xml 格式
```xml
<properties>
    <prop key="external.api.url" value="http://..." />
    <prop key="external.api.token" value="xxx" />
</properties>
```

运行时读取：`AppContext.getSystemProperty("external.api.url")`

## Controller模式

### Spring XML配置（DTD格式）
```xml
<beans default-autowire="byName">
    <bean name="/customPlugin.do"
          class="com.seeyon.apps.customplugin.controller.CustomPluginController"/>
</beans>
```

### Controller类
```java
public class CustomPluginController extends BaseController {

    @NeedlessCheckLogin  // 免登录注解(V8+特性)
    public ModelAndView ssoMobile(HttpServletRequest req, HttpServletResponse res) {
        // 业务逻辑
        return redirectModelAndView("/login/sso?from=xxx&ticket=" + ticket);
    }

    @Override
    public ModelAndView index(HttpServletRequest req, HttpServletResponse res) {
        return new ModelAndView("apps/plugin/xxx/list");
    }
}
```

### AJAX暴露配置（ajax-plugin-xxx.xml）
```xml
<services>
    <service name="managerBean" method="methodName" />
</services>
```

## 事件监听 (@ListenEvent + AbstractWorkflowEvent)

> 完整指南见 `references/bpm-event-guide.md`（基于官方文档 + 项目实战，467行）

致远CTP提供**两套后端事件机制**：

| 机制 | 作用域 | 绑定方式 | 适用场景 |
|------|--------|---------|---------|
| **AbstractWorkflowEvent** | 单个模板 | 流程设计器中绑定 | 特定表单审批后处理、发起前校验 |
| **@ListenEvent** | 全局 | 注解 + Spring Bean | 通用监听、待办推送、日志记录 |

### AbstractWorkflowEvent（流程级 — 首选）

```java
public class MyWorkflowEvent extends AbstractWorkflowEvent {
    @Override public String getId() { return "uniqueId"; }
    @Override public String getLabel() { return "事件显示名"; }
    @Override public WorkflowEventConstants.WorkflowEventType getType() {
        return WorkflowEventConstants.WorkflowEventType.Ext;
    }

    // 发起前校验（可阻止发起）
    @Override
    public WorkflowEventResult onBeforeStart(WorkflowEventData data) {
        if (缺少必要字段) return new WorkflowEventResult("提示信息"); // 弹窗阻塞
        return null; // 不阻塞
    }

    // 流程结束（只有后事件）
    @Override
    public void onProcessFinished(WorkflowEventData data) {
        FormDataMasterBean bean = data.getBusinessData().get("formDataBean");
        // 推送数据到外部系统
    }

    // 其他可覆写方法：
    // onStart, onFinishWorkitem, onStop, onStepBack, onCancel, onTakeBack
    // onBeforeFinishWorkitem, onBeforeStop, onBeforeStepBack, onBeforeCancel, onBeforeTakeBack
}
```

### @ListenEvent（全局监听）

三种执行模式：

```java
// ❌ 同步（默认）— 异常导致事务回滚
@ListenEvent(event = CollaborationStartEvent.class)
public void onStart(CollaborationStartEvent event) { }

// ⚠️ 异步 — 异常不影响主流程，但主流程回滚时此代码仍执行
@ListenEvent(event = CollaborationStartEvent.class, async = true)
public void onStart(CollaborationStartEvent event) { }

// ✅ 事务提交后（推荐）— 只有主流程成功才执行
@ListenEvent(event = CollaborationStartEvent.class, mode = EventTriggerMode.afterCommit)
public void onStart(CollaborationStartEvent event) { }
```

### 常用事件类型
| 事件类 | 触发时机 | 用途 |
|--------|---------|------|
| `CollaborationStartEvent` | 协同发起 | 推送待办 |
| `CollaborationProcessEvent` | 协同处理 | 推送已办+新待办 |
| `CollaborationFinishEvent` | 协同结束 | 推送已办/触发后处理 |
| `CollaborationStopEvent` | 协同终止 | 取消待办 |
| `CollaborationCancelEvent` | 协同撤销 | 取消待办 |
| `CollaborationStepBackEvent` | 协同回退 | 推送已办+新待办 |
| `CollaborationTakeBackEvent` | 协同取回 | 更新待办 |
| `EdocStartEvent/ProcessEvent/FinishEvent` | 公文事件 | 公文待办同步 |
| `FormDataAfterSubmitEvent` | CAP4表单提交后 | 数据触发同步 |
| `FileDownloadEvent` | 文件下载 | 水印/日志 |
| `AddMemberEvent/UpdateMemberEvent/DeleteMemberEvent` | 组织变更 | 组织同步 |

### 防重复触发模式
```java
// 使用延迟队列聚合短时间内多次事件
private final Map<String, ScheduledFuture<?>> pendingTasks = new ConcurrentHashMap<>();

scheduler.schedule(() -> {
    // 实际执行逻辑
    pendingTasks.remove(key);
}, 2000, TimeUnit.MILLISECONDS);
```

## CAP4表单操作

### 表单数据获取标准链路
```
超级节点params → params.get("CTP_FORM_DATA") → Map
  → data.get("formDataBean") → FormDataMasterBean
    → bean.getFormTable() → FormTableBean
      → table.getFieldBeanByDisplay("显示名") → FormFieldBean
        → bean.getFieldValue("field0012") → Object
```

### FormCap4Kit/CAP4FormKit 工具方法集
```java
// 获取表单Bean
FormBean getFormBean(CAP4FormManager manager, String templateCode);

// 按显示名获取字段值
Object getFieldValue(FormDataBean bean, String displayName);
String getFieldStrValue(FormDataBean bean, String displayName);
Long getFieldLongValue(FormDataBean bean, String displayName);

// 枚举转换
String getEnumvalue(Long enumId);       // 枚举Value
String getEnumName(Long enumId);        // 枚举显示名
String getEnumCode(Long enumId);        // 枚举编码

// 人员/部门/岗位处理
String getDeptCode(FormDataBean bean, String fieldName);
String getMemberCode(FormDataBean bean, String fieldName);
String getMemberLoginName(FormDataBean bean, String fieldName);

// 子表遍历
List<FormDataSubBean> getSubBeans(FormDataMasterBean master);
List<List<FormDataSubBean>> getAllSubBeans(FormDataMasterBean master);

// 字段赋值
void setCellValue(FormDataBean bean, String displayName, Object value);

// 附件操作
File downloadFile(FormFieldBean fieldBean);
```

## 数据访问模式

### DBKit（JDBCAgent封装 — 最常用）
```java
// 标准用法
JDBCAgent jdbc = new JDBCAgent(true);  // true=开启事务
jdbc.execute(sql, params);
List<Map<String, Object>> list = jdbc.resultSetToList();
jdbc.close();

// 分页查询
jdbc.findNameByPaging(sql, parameterMap, flipInfo);
int total = jdbc.count(countSql, parameterMap);

// 单条查询
Map<String, Object> row = jdbc.resultSetToMap();
```

### DBAgent（HQL/ORM操作）
```java
// 查询
List<Entity> list = DBAgent.find(hql, params, flipInfo);
Entity e = DBAgent.get(Entity.class, id);

// 保存/更新
DBAgent.save(po);
DBAgent.update(po);
DBAgent.updateAll(list);

// 原生SQL
List<Object[]> rows = cap4FormDataDAO.selectDataBySql(sql);
```

## 超级节点（BaseSuperNodeAction）

### 基类方法
```java
public class BankNode extends BaseSuperNodeAction {
    @Override
    public String getNodeId() { return "bankNode"; }
    @Override
    public String getNodeName() { return "银行付款节点"; }
    @Override
    public int getOrder() { return 0; }

    @Override
    public SuperNodeResponse executeAction(Map<String, Object> params) {
        // 1. 获取CAP4表单数据
        FormDataMasterBean bean = (FormDataMasterBean)
            ((Map)params.get("CTP_FORM_DATA")).get("formDataBean");
        // 2. 调用业务逻辑
        // 3. 返回结果
        if (success) {
            return new SuperNodeResponse(1);  // WAIT（继续流转）
        } else {
            return new SuperNodeResponse(2);  // BACK（回退节点）
        }
    }
}
```

### 轻量级超级节点模式（适用于多类型场景）
```java
public abstract class AbstractLightweightContractNode extends BaseSuperNodeAction {
    // 公共逻辑抽取到基类
    @Override
    public SuperNodeResponse executeAction(Map<String, Object> params) {
        return contractPushService.push(getTypeCode(), token, activityId, params);
    }
    protected abstract String getTypeCode();
}

// 子类只需声明类型
public class DomesticSaleContractNode extends AbstractLightweightContractNode {
    @Override protected String getTypeCode() { return "DOMESTIC_SALE"; }
}
```

## REST资源（BaseResource）

### JAX-RS风格
```java
@Path("hsbj")
public class HsBjResource extends BaseResource {
    @POST
    @Path("getBjCategory")
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    @RestInterfaceAnnotation  // V5客开REST注解
    public Map<String, Object> getBjCategory(Map<String, Object> params) {
        // 调用Manager
        return result;
    }
}
```

### 免登录配置（needless_check_login_*.xml）
```xml
<config>
    <path>/sasso/sasso.do</path>
    <method>entry,mobileSSO</method>
</config>
```

## 定时任务与初始化器

### QuartzJob定时任务
```java
public class UpdateOrderDataTask implements QuartzJob {
    @Override
    public void execute(JobExecutionContext context) {
        nodeBankManager.updateAllOrderPayStatus();
    }
    // 注册方法
    public static void registerSyncTask(boolean enabled, int hour, int minute) {
        // 通过致远任务框架注册
    }
}
```

### AbstractSystemInitializer初始化器
```java
public class CustomPluginInitializer extends AbstractSystemInitializer {
    @Override
    public void initialize() {
        System.out.println("◆启动客开插件!");
        // 注册定时任务
        UpdateOrderDataTask.registerSyncTask(true, 23, 50);
        // 执行初始化SQL
        ExcuteSqlFileUtil.initSqlFile();
    }
}
```

## HTTP客户端集成模式

### OkHttp模式（推荐，V10.0SP1+）
```java
public class KingdeeClient {
    private static final OkHttpClient client = new OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .build();

    // Token缓存（双重检查锁）
    private volatile String token;
    private final ReentrantLock lock = new ReentrantLock();

    public String getToken() {
        if (token == null) {
            lock.lock();
            try {
                if (token == null) {
                    token = fetchToken();
                }
            } finally { lock.unlock(); }
        }
        return token;
    }
}
```

### Apache HttpClient模式（兼容V8.x）
```java
public class HttpKit {
    private static final PoolingHttpClientConnectionManager cm;

    public static String post(String url, Map<String, String> headers, String body) { ... }
    public static String get(String url, Map<String, String> headers) { ... }
}
```

## 补偿与重试机制

### 待办推送补偿模式（推荐参考模式）
```
事件触发/初始扫描
  → 创建PendingSyncTask（状态=PENDING）
  → PendingDispatchService.dispatch()
    → 1. markSending() 原子更新（防并发）
    → 2. HTTP推送
    → 3. 写入调用日志
    → 4. 更新状态：SUCCESS/FAILED_RETRYABLE/FAILED_FINAL
  → PendingRetryJob守护线程（60秒扫描，退避策略）
  → 管理员人工干预（重试/标记完成/忽略/修改重推）
```

## CAP4自定义控件（FormFieldCustomCtrl）

### 后端控件定义
```java
public class DataReportingButton extends CommonBtn {
    // CommonBtn是V5 CAP4自定义按钮标准基类
}
```

### 前端runtime.js（原生JS）
```javascript
// 按钮点击调用后端Spring Bean
callBackendMethod("dataReportingManager", "dataReporting", {
    formData: getFormData(),
    templateCode: getTemplateCode()
}, function(response) {
    if (response.success) {
        alert("推送成功!");
    }
});
```

### 控件目录结构
```
apps_res/cap/customCtrlResources/<controlId>/
├── html/design.html    # 设计器配置界面
└── js/runtime.js       # 运行时JS逻辑
```

## 安全实践

1. **SQL注入防护** — 表名/字段名使用白名单校验
   ```java
   if (!TableWhitelistChecker.isValid(metaTableName)) {
       throw new SecurityException("非法表名: " + metaTableName);
   }
   ```
2. **参数化查询** — 使用JDBCAgent的params参数绑定，不拼接SQL
3. **Token自动刷新** — 使用Hutool TimedCache或自定义缓存+双重检查锁
4. **密码重置限制** — 测试环境专用功能加警告注释和条件判断

## 常见项目规模对比

| 特征 | 小型项目 | 中型项目 | 大型项目 |
|------|---------|---------|---------|
| Maven模块数 | 1-3 | 5-10 | 15-40+ |
| 插件数 | 1-3 | 5-10 | 15-20+ |
| CAP4控件数 | 0-1 | 3-5 | 8-10 |
| Controller数 | 1-2 | 3-5 | 10+ |
| 超级节点数 | 0-1 | 1-5 | 20+ |
| 外部集成系统 | 1 | 2-3 | 4+ |

## 参考文档索引

| 文档 | 内容 |
|------|------|
| `references/zhiyuan-sources.md` | 官方文档链接、关键API索引、事件类速查 |
| `references/cap4-guide.md` | CAP4表单取值、子表遍历、枚举转换、自定义控件 |
| `references/api-integration.md` | HTTP客户端封装、Token管理、签名加密、补偿机制 |
| `references/jsp-guide.md` | JSP CRUD页面模板、分页、弹窗、AJAX交互 |
| `references/bpm-event-guide.md` | **流程事件开发完整指南** — AbstractWorkflowEvent + @ListenEvent + 前端拦截 |
| `references/seeyonapi-event-index.md` | **SeeyonAPI 8.2 事件完整索引** — 20模块、180+事件速查 |
| `references/org-architecture-guide.md` | **组织架构开发完整指南** — 39事件 + REST同步 + 平台API |
| `references/super-node-guide.md` | **超级节点开发指南** — BaseSuperNodeAction + 轻量节点抽象 + 返回值规范 |
| `references/cap4-field-converter-guide.md` | **CAP4字段取值与转换器指南** — 中文字段取值 + 枚举/组织/附件转换 |

## 注意事项

- **编码**：老V5项目中文可能使用GBK编码，编辑前用`file`命令检测
- **平台版本**：目录名可能误导（如某些项目目录含"V5"但实际是V8.2SP1），以pom.xml为准
- **代码复用**：不同插件间DBKit/StrKit常被复制粘贴，建议提取公共模块
- **硬编码**：API地址、密钥建议通过pluginProperties.xml配置，避免硬编码
- **废弃代码**：检查是否有类似`*bAK`（备份）或完全注释掉的文件，及时清理
