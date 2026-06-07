# 致远CTP流程事件开发完整指南

基于致远官方文档和真实客开项目实战总结。

## 事件体系概览

致远CTP提供三套事件机制，适用不同场景：

| 机制 | 作用域 | 绑定方式 | 适用场景 |
|------|--------|---------|---------|
| **AbstractWorkflowEvent** | 单个流程模板 | 流程设计器中选择 | 特定表单/模板的审批后处理 |
| **@ListenEvent** | 全局所有流程 | 注解 + Spring Bean | 通用监听、待办推送等 |
| **前端事件拦截** | 前端交互 | `$.ctp.bind()` | 发送前校验、UI拦截 |

---

## 一、AbstractWorkflowEvent（流程级事件）

### 1.1 概述
绑定到具体流程模板/表单，在流程设计器的"触发设置"中选择。每个实现类对应一个（或一类）流程模板。**非全局**，只对绑定的模板生效。

### 1.2 基类方法

```java
public abstract class AbstractWorkflowEvent {

    /** 唯一标识，一旦生成不可变更 */
    public abstract String getId();

    /** 事件显示名称（在流程设计器中显示） */
    public abstract String getLabel();

    /** 返回指定的模板编号（空字符串表示不限定） */
    public String getTemplateCode() { return ""; }

    /** 应用类别 */
    public ApplicationCategoryEnum getAppName() {
        return ApplicationCategoryEnum.form;
    }

    /** 事件类型，客开固定返回 Ext */
    public WorkflowEventConstants.WorkflowEventType getType() {
        return WorkflowEventConstants.WorkflowEventType.Ext;
    }

    // ===== 发起事件 =====
    /** 发起前事件（可阻止发起） */
    public WorkflowEventResult onBeforeStart(WorkflowEventData data) { return null; }
    /** 发起后事件 */
    public void onStart(WorkflowEventData data) {}

    // ===== 处理事件 =====
    /** 处理前事件（可阻止处理） */
    public WorkflowEventResult onBeforeFinishWorkitem(WorkflowEventData data) { return null; }
    /** 处理后事件（节点办结） */
    public void onFinishWorkitem(WorkflowEventData data) {}

    // ===== 终止事件 =====
    /** 终止前事件 */
    public WorkflowEventResult onBeforeStop(WorkflowEventData data) { return null; }
    /** 终止后事件 */
    public void onStop(WorkflowEventData data) {}

    // ===== 回退事件 =====
    /** 回退前事件（可阻止回退） */
    public WorkflowEventResult onBeforeStepBack(WorkflowEventData data) { return null; }
    /** 回退后事件 */
    public void onStepBack(WorkflowEventData data) {}

    // ===== 撤销事件 =====
    /** 撤销前事件 */
    public WorkflowEventResult onBeforeCancel(WorkflowEventData data) { return null; }
    /** 撤销后事件 */
    public void onCancel(WorkflowEventData data) {}

    // ===== 取回事件 =====
    /** 取回前事件 */
    public WorkflowEventResult onBeforeTakeBack(WorkflowEventData data) { return null; }
    /** 取回后事件 */
    public void onTakeBack(WorkflowEventData data) {}

    // ===== 流程结束事件 =====
    /** 流程结束后事件（只有后事件，无前事件） */
    public void onProcessFinished(WorkflowEventData data) {}
}
```

### 1.3 WorkflowEventData 数据结构

```java
public class WorkflowEventData {
    private long summaryId;      // 流程实例ID（onBeforeStart中为-1）
    private long affairId;       // 事务ID（onBeforeStart中获取不到）
    private String form;         // 节点绑定的表单ID
    private String formApp;      // 节点绑定的表单应用ID
    private String operationName;// 节点绑定的表单操作视图ID
    private Map businessData;    // 业务数据集合（含跨表数据）

    // 通过 businessData 获取表单数据：
    // data.getBusinessData().get("formDataBean") → FormDataMasterBean
}
```

### 1.4 WorkflowEventResult 返回值

```java
public class WorkflowEventResult {
    private String alertMessage = "";

    // 空构造 → 不阻塞流程
    public WorkflowEventResult() {}

    // 带消息构造 → 弹出提示框，阻塞当前行为
    public WorkflowEventResult(String alertMessage) {
        this.alertMessage = alertMessage;
    }
}
```

### 1.5 Spring XML 注册

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN"
    "http://www.springframework.org/dtd/spring-beans.dtd">
<beans default-autowire="byName">
    <bean id="myWorkflowEvent" class="com.seeyon.apps.xxx.event.MyWorkflowEvent" />
</beans>
```

### 1.6 实战代码模板

```java
public class ApprovalToExternalWorkFlowWriterBack extends AbstractWorkflowEvent {

    private static final Logger logger = Logger.getLogger(ApprovalToExternalWorkFlowWriterBack.class);

    @Override
    public String getId() {
        return "approvalToExternalWriterBack";
    }

    @Override
    public String getLabel() {
        return "H_审批结果回传外部系统";
    }

    @Override
    public ApplicationCategoryEnum getAppName() {
        return ApplicationCategoryEnum.form;
    }

    @Override
    public WorkflowEventConstants.WorkflowEventType getType() {
        return WorkflowEventConstants.WorkflowEventType.Ext;
    }

    // ===== 发起前校验 =====
    @Override
    public WorkflowEventResult onBeforeStart(WorkflowEventData data) {
        Map<String, Object> formData = data.getBusinessData();
        FormDataMasterBean masterBean = (FormDataMasterBean) formData.get("formDataBean");

        // 校验必填字段
        String externalId = FormCap4Kit.getFieldStrValue(masterBean, "外部系统单据ID");
        if (externalId == null || externalId.isEmpty()) {
            return new WorkflowEventResult("请先填写外部系统单据ID后再发起流程");
        }
        return null; // null = 不阻塞
    }

    // ===== 流程结束：推送数据到外部 =====
    @Override
    public void onProcessFinished(WorkflowEventData data) {
        try {
            Map<String, Object> formData = data.getBusinessData();
            FormDataMasterBean masterBean = (FormDataMasterBean) formData.get("formDataBean");

            // 1. 提取表单字段
            String externalId = FormCap4Kit.getFieldStrValue(masterBean, "外部系统单据ID");
            String auditResult = FormCap4Kit.getFieldStrValue(masterBean, "审批结果");

            // 2. 构造推送数据
            Map<String, Object> pushData = new HashMap<>();
            pushData.put("externalId", externalId);
            pushData.put("auditResult", auditResult);
            pushData.put("summaryId", String.valueOf(data.getSummaryId()));

            // 3. 调用外部API
            String resp = ExternalApiClient.post(apiUrl, pushData);

            // 4. 记录推送日志
            savePushLog(data.getSummaryId(), "FORM_SYNC", resp);

        } catch (Exception e) {
            logger.error("流程结束推送失败, summaryId=" + data.getSummaryId(), e);
        }
    }

    // ===== 流程终止 =====
    @Override
    public void onStop(WorkflowEventData data) {
        updateExternalStatus(data, "STOPPED");
    }

    // ===== 流程撤销 =====
    @Override
    public void onCancel(WorkflowEventData data) {
        updateExternalStatus(data, "CANCEL");
    }
}
```

### 1.7 关键注意事项

1. **onBeforeStart中 summaryId = -1**：首次发起时尚未生成summaryId，不要依赖
2. **事前事件中修改表单数据**：用 `addFieldValue()` 而非 `saveOrUpdateFormData()`，调用 `FormService.findDataById(masterId, formId)` 读取
3. **事前事件返回值**：null/空message → 继续；非空message → 弹窗阻塞
4. **流程结束只有后事件**：无 `onBeforeProcessFinished`

---

## 二、@ListenEvent（全局监听事件）

### 2.1 概述

全局作用域，对**所有流程模板**生效。注解驱动，监听类注册为Spring Bean即可。与AbstractWorkflowEvent不同，它是流程无关的通用监听。

### 2.2 三种执行模式

```java
// 模式1：同步执行（默认）- 异常会导致事务回滚
@ListenEvent(event = CollaborationStartEvent.class)
public void onStart1(CollaborationStartEvent event) {
    // 发送短信：如果短信网关出错，协同发起也会失败！
}

// 模式2：异步执行 - 异常不影响主流程，但主流程回滚时此代码仍会执行
@ListenEvent(event = CollaborationStartEvent.class, async = true)
public void onStart2(CollaborationStartEvent event) {
    // 发送短信：协同发起出错，但短信已经发出，无法追回
}

// 模式3：事务提交后执行（推荐） - 只有主流程成功才执行
@ListenEvent(event = CollaborationStartEvent.class, mode = EventTriggerMode.afterCommit)
public void onStart3(CollaborationStartEvent event) {
    // 发送短信：只有协同发起成功才发出
}
```

### 2.3 完整事件列表

#### 协同流程事件

| 事件类 | 触发时机 | 可用属性 |
|--------|---------|---------|
| `CollaborationStartEvent` | 流程发起（含协同/转发/移动/接口发起） | `getSummaryId()` |
| `CollaborationProcessEvent` | 流程处理（V61+，可获取Comment） | `getSummaryId()`, `getComment()` |
| `CollaborationFinishEvent` | 流程结束 | `getSummaryId()` |
| `CollaborationCancelEvent` | 流程撤销 | `getSummaryId()` |
| `CollaborationStepBackEvent` | 流程回退 | `getSummaryId()` |
| `CollaborationStopEvent` | 流程终止 | `getSummaryId()` |
| `CollaborationAppointStepBackEvent` | 指定回退（V61+） | `getSummaryId()`, `getSelectTargetNodeId()` |
| `CollaborationTakeBackEvent` | 流程取回（V61+） | `getSummaryId()` |
| `CollaborationAddCommentEvent` | 流程处理回复（V61+） | `getSummaryId()` |
| `CollaborationAutoSkipEvent` | 流程自动跳过（V61+） | `getSummaryId()` |
| `CollaborationDelEvent` | 流程删除（V61+） | `getSummaryId()` |

#### 其他事件

| 事件类 | 触发时机 |
|--------|---------|
| `EdocSignEvent` | 公文签收 |
| `BulletinAddEvent` | 公告发起（V61+） |
| `NewsAddEvent` | 新闻发起（V61+） |
| `MeetingAffairsAssignedEvent` | 会议分配 |
| `FormDataAfterSubmitEvent` | CAP4表单提交后 |
| `FileDownloadEvent` | 文件下载 |
| `AddMemberEvent` / `UpdateMemberEvent` / `DeleteMemberEvent` | 组织架构变更 |

### 2.4 实战代码模板

```java
public class MyCollaborationEventListener {

    /**
     * 指定回退到发起节点时，通知外部系统取消审批
     */
    @ListenEvent(event = CollaborationAppointStepBackEvent.class, async = true)
    public void onAppointStepBack(CollaborationAppointStepBackEvent event) {
        try {
            // 判断是否回退到发起节点
            if ("start".equals(event.getSelectTargetNodeId())) {
                ColSummary summary = collaborationApi.getColSummary(event.getSummaryId());
                // 只处理特定模板
                if ("目标模板编码".equals(summary.getFormApp())) {
                    String externalId = getExternalIdFromForm(summary);
                    externalApiClient.cancelAudit(externalId);
                }
            }
        } catch (Exception e) {
            logger.error("指定回退处理失败", e);
        }
    }
}
```

### 2.5 Spring XML 注册

```xml
<beans default-autowire="byName">
    <bean id="myCollaborationEventListener"
          class="com.seeyon.apps.xxx.event.MyCollaborationEventListener" />
</beans>
```

---

## 三、前端事件拦截

### 3.1 概述

在浏览器端拦截协同操作，用于发送前校验或自定义逻辑。通过 `$.ctp.bind()` 绑定。

### 3.2 事件列表

| 事件名称 | 说明 | V61+参数 |
|---------|------|---------|
| `beforSendColl` | 协同发送前 | - |
| `beforeSaveDraftColl` | 协同保存待发前 | - |
| `beforeDealSubmit` | 协同处理提交 | `summaryID`, `affairID` |
| `beforeDealSaveWait` | 协同暂存待办 | `summaryID`, `affairID` |
| `beforeDealCancel` | 协同撤销 | `summaryID`, `affairID` |
| `beforeDealstepback` | 协同回退 | `summaryID`, `affairID` |
| `beforeDealstepstop` | 协同终止 | `summaryID`, `affairID` |
| `beforeDealaddnode` | 协同加签 | `summaryID`, `affairID` |
| `beforeDealdeletenode` | 协同减签 | `summaryID`, `affairID` |
| `beforeDealspecifiesReturn` | 协同指定回退 | `summaryID`, `affairID` |
| `beforeDoneTakeBack` | 已办取回 | `summaryID`, `affairID` |
| `beforeWaitSendDelete` | 待发删除 | `summaryID`, `affairID` |
| `beforeSentCancel` | 已发撤销 | `summaryID`, `affairID` |
| `dealRepeatChange` | 表单重复行变更 | `formId`, `dataId` |
| `fieldValueChange` | 表单字段值变更 | `formId`, `dataId` |
| `businessFormInit` | 无流程表单修改 | `formId`, `dataId` |
| `businessFormDel` | 无流程表单删除 | `formId`, `dataId` |
| `businessFormSave` | 无流程表单保存 | - |

### 3.3 使用示例

```javascript
// 协同发送前校验
$.ctp.bind('beforSendColl', function() {
    // 自定义校验逻辑
    if (!validateFormData()) {
        alert("请填写必要信息后再发送");
        return false;  // 阻止发送
    }
    return true;  // 允许发送
});

// 协同处理提交前校验（V61+可获取summaryID）
$.ctp.bind('beforeDealSubmit', function() {
    var summaryId = arguments[0].summaryID;
    var affairId = arguments[0].affairID;
    // 处理逻辑
    return true;
});
```

**规则**：同一事件可绑定多个监听，任一返回 `false` 即中断。

---

## 四、两种后端事件对比

| 维度 | AbstractWorkflowEvent | @ListenEvent |
|------|----------------------|-------------|
| **作用域** | 绑定到具体模板 | 全局所有模板 |
| **配置方式** | 流程设计器中绑定 | Spring Bean注册 |
| **筛选模板** | 天然隔离 | 需代码中判断 `summary.getFormApp()` |
| **阻止流程** | 事前事件返回 `WorkflowEventResult` | 不可阻止（事后响应） |
| **获取表单数据** | `businessData.get("formDataBean")` | 需通过API查询 |
| **事务关联** | 同步（跟随流程事务） | 可选 async/afterCommit |
| **适用场景** | 特定模板的后处理 | 通用监听（待办推送、日志等） |
| **推荐优先级** | 优先（模板级隔离） | 补充（全局需求） |

---

## 五、实战模式：流程事件驱动外部集成

基于真实项目的完整集成模式（以审批后推送CRM为例）：

### 5.1 架构分层

```
AbstractWorkflowEvent / @ListenEvent
      │
      ▼
  ┌──────────────────┐
  │  事件处理层       │  提取表单数据、判断触发条件
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │  服务调用层       │  组装请求、调用外部API
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │  日志记录层       │  写入推送日志表，支持补偿
  └──────────────────┘
```

### 5.2 完整事件覆盖策略

| 事件 | 做什么 |
|------|--------|
| `onBeforeStart` | 校验外部系统单据ID必填 |
| `onFinishWorkitem` | 节点处理完成，通知外部更新节点状态 |
| `onProcessFinished` | 审批通过，推送完整表单数据 |
| `onStop` | 审批终止，通知外部状态变更 |
| `onCancel` | 审批撤销，通知外部取消 |
| `onStepBack` | 流程回退，通知外部 |
| `onBeforeStepBack` | 回退前拦截（可选） |

### 5.3 推送日志表设计

```sql
CREATE TABLE crm_push_log (
    id BIGINT PRIMARY KEY,
    summary_id BIGINT,
    push_type VARCHAR(50),     -- FORM_SYNC / AUDIT_STATUS / ATTACHMENT / NOTIFICATION
    trigger_source VARCHAR(50),-- WORKFLOW / COMPENSATION / MANUAL
    request_body TEXT,
    response_body TEXT,
    push_status VARCHAR(20),   -- SUCCESS / FAILED
    create_time DATETIME,
    error_msg TEXT
);
```

### 5.4 防重复机制

```java
// 通过日志表判断是否已推送
public boolean isAlreadyPushed(Long summaryId, String pushType) {
    String sql = "SELECT COUNT(1) FROM crm_push_log WHERE summary_id=? AND push_type=? AND push_status='SUCCESS'";
    return DBKit.count(sql, new Object[]{summaryId, pushType}) > 0;
}
```

---

## 六、注意事项

1. **事前事件中不要调用外部API阻塞流程**：外部接口超时会导致整个流程卡死
2. **@ListenEvent 推荐用 afterCommit 模式**：避免主流程失败但消息已发出
3. **异常必须 catch**：事件方法中的未捕获异常会影响流程流转
4. **onBeforeStart 中 summaryId 不可用**：此时流程尚未创建，summaryId = -1
5. **formDataBean 获取时机**：仅在 `onBeforeStart`、`onBeforeFinishWorkitem` 中可用，`onProcessFinished` 中需通过API查询
