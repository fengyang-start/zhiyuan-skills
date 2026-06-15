# 致远CTP超级节点开发指南

基于致远CTP V5/V8客开项目中的流程超级节点实战模式整理，重点覆盖 `BaseSuperNodeAction`、`SuperNodeResponse`、CAP4表单参数获取、轻量级节点抽象和Spring注册规范。

## 一、超级节点适用场景

超级节点适合在流程流转到某个节点时自动执行业务动作，例如：

- 将当前表单数据推送到第三方系统
- 调用外部接口创建/更新业务单据
- 自动判断流程继续、回退或暂存
- 根据接口返回结果控制流程走向
- 记录完整推送日志用于排查和补偿

不建议在超级节点中处理耗时不可控、需要人工确认、或者与当前流程节点无强关联的全局事件；这类场景优先考虑 `@ListenEvent` 或定时补偿任务。

## 二、标准超级节点接口

### 2.1 基类

```java
public abstract class XxxNode extends BaseSuperNodeAction {
    @Override
    public String getNodeId() { return "uniqueNodeId"; }

    @Override
    public String getNodeName() { return "节点显示名称"; }

    @Override
    public int getOrder() { return 0; }

    @Override
    public SuperNodeResponse executeAction(String token, String activityId, Map<String, Object> params) {
        // 节点执行逻辑
    }

    @Override
    public SuperNodeResponse confirmAction(String token, String activityId, Map<String, Object> params) {
        return executeAction(token, activityId, params);
    }

    @Override
    public void cancelAction(String token, String activityId, Map<String, Object> params) {
        // 取消逻辑，通常只记录日志
    }
}
```

### 2.2 参数说明

| 参数 | 说明 |
|------|------|
| `token` | 超级节点当前令牌，可用于日志关联 |
| `activityId` | 当前流程节点ID |
| `params` | 工作流上下文参数，包含 `CTP_FORM_DATA` 等 |

### 2.3 返回值规范

```java
SuperNodeResponse response = new SuperNodeResponse();
response.setReturnCode(1); // 成功，流程继续/等待平台继续处理
response.setReturnMsg("处理成功");

response.setReturnCode(2); // 失败，流程回退或阻断
response.setReturnMsg("接口返回错误：xxx");
```

| returnCode | 常见含义 | 使用场景 |
|------------|----------|---------|
| `1` | 成功/继续 | 外部接口调用成功、数据推送成功 |
| `2` | 失败/回退 | 外部接口失败、必填配置缺失、异常 |
| `0` | 暂存/其他 | 少数历史项目使用，需结合项目平台行为确认 |

## 三、从超级节点获取CAP4表单数据

### 3.1 标准取值链路

```java
@SuppressWarnings("unchecked")
private FormDataMasterBean getFormDataBean(Map<String, Object> params) {
    if (params == null) {
        return null;
    }
    Map<String, Object> data = (Map<String, Object>) params.get("CTP_FORM_DATA");
    if (data == null) {
        return null;
    }
    return (FormDataMasterBean) data.get("formDataBean");
}
```

### 3.2 根据中文数据域/字段显示名取值

```java
FormDataMasterBean masterBean = getFormDataBean(params);
String title = ContractFieldMapper.getString(masterBean, "单据名称");
Double amount = ContractFieldMapper.getDouble(masterBean, "金额");
Long enumId = ContractFieldMapper.getLong(masterBean, "业务类型");
```

底层原理：

```java
FormTableBean table = bean.getFormTable();
FormFieldBean field = table.getFieldBeanByDisplay("单据名称");
Object value = bean.getFieldValue(field.getName()); // field.getName() 通常是 field0012
```

## 四、轻量级超级节点抽象模式

当一个项目中存在多个业务类型节点，但执行逻辑基本一致，仅类型编码、节点ID、节点名称不同，推荐使用抽象基类。

### 4.1 抽象基类模板

```java
public abstract class AbstractLightweightBusinessNode extends BaseSuperNodeAction {

    private static final Log LOGGER = CtpLogFactory.getLog(AbstractLightweightBusinessNode.class);

    private BusinessPushService businessPushService;

    /** 每个子类只声明自己的业务类型编码 */
    protected abstract String getTypeCode();

    protected String getLogPrefix() {
        return "[业务推送-" + getNodeName() + "]";
    }

    @Override
    public SuperNodeResponse executeAction(String token, String activityId, Map<String, Object> params) {
        LOGGER.info(getLogPrefix() + "开始, typeCode=" + getTypeCode()
                + ", token=" + token + ", activityId=" + activityId
                + ", paramsKeys=" + (params == null ? "" : params.keySet()));
        try {
            SuperNodeResponse response = getBusinessPushService().push(getTypeCode(), token, activityId, params);
            LOGGER.info(getLogPrefix() + "结束, returnCode=" + response.getReturnCode()
                    + ", returnMsg=" + response.getReturnMsg());
            return response;
        } catch (Exception e) {
            LOGGER.error(getLogPrefix() + "异常", e);
            SuperNodeResponse response = new SuperNodeResponse();
            response.setReturnCode(2);
            response.setReturnMsg(getNodeName() + "执行异常: " + e.getMessage());
            return response;
        }
    }

    @Override
    public SuperNodeResponse confirmAction(String token, String activityId, Map<String, Object> params) {
        return executeAction(token, activityId, params);
    }

    @Override
    public void cancelAction(String token, String activityId, Map<String, Object> params) {
        LOGGER.info(getLogPrefix() + "取消执行, typeCode=" + getTypeCode());
    }

    @Override
    public int getOrder() {
        return 0;
    }

    private BusinessPushService getBusinessPushService() {
        if (businessPushService == null) {
            businessPushService = (BusinessPushService) AppContext.getBean("businessPushService");
        }
        return businessPushService;
    }

    public void setBusinessPushService(BusinessPushService businessPushService) {
        this.businessPushService = businessPushService;
    }
}
```

### 4.2 子类模板

```java
public class TypeAContractNode extends AbstractLightweightBusinessNode {
    @Override
    protected String getTypeCode() {
        return "TYPE_A";
    }

    @Override
    public String getNodeId() {
        return "typeAContractNode";
    }

    @Override
    public String getNodeName() {
        return "类型A合同节点";
    }
}
```

### 4.3 设计优点

- **KISS**：子类只保留3个差异点：`typeCode`、`nodeId`、`nodeName`
- **DRY**：接口调用、异常处理、日志、返回值统一在基类
- **OCP**：新增业务类型只新增子类和Spring Bean，不修改既有类
- **兼容平台扫描**：每个节点仍然保留独立Java类，便于致远超级节点识别

## 五、Spring XML注册

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans default-autowire="byName">
    <!-- 旧版节点保持不动，避免影响已投产流程 -->
    <bean id="legacyTypeANode" class="com.seeyon.apps.xxx.node.LegacyTypeANode" />

    <!-- 新增轻量超级节点 -->
    <bean id="typeAContractNode" class="com.seeyon.apps.xxx.node.lightweight.TypeAContractNode" />
    <bean id="typeBContractNode" class="com.seeyon.apps.xxx.node.lightweight.TypeBContractNode" />
    <bean id="typeCContractNode" class="com.seeyon.apps.xxx.node.lightweight.TypeCContractNode" />
</beans>
```

## 六、服务层推送流程

超级节点不应直接拼接口参数、登录第三方、写日志。推荐将核心逻辑下沉到Service：

```java
public SuperNodeResponse push(String typeCode, String token, String activityId, Map<String, Object> params) {
    SuperNodeResponse response = new SuperNodeResponse();
    long startTime = System.currentTimeMillis();
    String requestJson = "";
    String responseJson = "";

    try {
        // 1. 读取类型配置
        TypeConfig typeConfig = typeConfigService.findByCode(typeCode);
        if (typeConfig == null || !"1".equals(typeConfig.getEnabled())) {
            return fail(response, "类型配置不存在或已停用: " + typeCode);
        }

        // 2. 获取CAP4表单数据
        FormDataMasterBean masterBean = getFormDataBean(params);
        if (masterBean == null) {
            return fail(response, "获取表单数据失败");
        }

        // 3. 读取关键字段用于日志
        String billName = ContractFieldMapper.getString(masterBean, "单据名称");
        String billCode = ContractFieldMapper.getString(masterBean, "单据编号");

        // 4. 根据字段映射动态构建请求参数
        List<FieldMapping> mappings = mappingService.findEnabledMappings(typeCode);
        ContractBuildContext context = buildContext(typeConfig, masterBean);
        Map<String, Object> requestData = dynamicParamBuilder.build(masterBean, mappings, context);
        requestJson = JsonKit.toJSONString(requestData);

        // 5. 登录/获取Token，调用第三方接口
        LoginResult loginResult = apiClient.login();
        responseJson = apiClient.post(typeConfig.getEndpoint(), requestJson, loginResult);

        // 6. 解析结果，返回SuperNodeResponse
        if (isSuccess(responseJson)) {
            response.setReturnCode(1);
            response.setReturnMsg("推送成功");
            saveLog(token, typeCode, billName, billCode, requestJson, responseJson, true, "", startTime);
            return response;
        }
        response.setReturnCode(2);
        response.setReturnMsg("接口返回错误");
        saveLog(token, typeCode, billName, billCode, requestJson, responseJson, false, "接口返回错误", startTime);
        return response;
    } catch (Exception e) {
        saveLog(token, typeCode, "", "", requestJson, responseJson, false, e.getMessage(), startTime);
        return fail(response, "推送异常: " + e.getMessage());
    }
}
```

## 七、日志与容错规范

1. **超级节点异常必须catch**：不要让异常直接冒泡到工作流引擎
2. **日志保存失败不影响流程主逻辑**：`saveLog()` 内部单独 `try-catch`
3. **记录上下文**：`typeCode`、`token`、`activityId`、关键字段、请求、响应、耗时
4. **敏感信息脱敏**：密码、token、Cookie、真实接口地址不要打完整日志
5. **配置缺失返回失败**：第三方地址、账号、密码缺失时 `returnCode=2`

## 八、新增超级节点检查清单

- [ ] 新增节点类继承抽象基类
- [ ] `getNodeId()` 唯一且与Spring Bean/配置一致
- [ ] `getNodeName()` 用户可读
- [ ] `getTypeCode()` 与类型配置表一致
- [ ] Spring XML注册Bean
- [ ] 类型配置启用，endpoint配置正确
- [ ] 字段映射配置完整
- [ ] 关键字段能按中文显示名取到值
- [ ] 转换器能处理枚举/人员/单位/附件
- [ ] 日志表记录成功/失败
- [ ] 接口异常时返回 `returnCode=2` 而非抛异常
