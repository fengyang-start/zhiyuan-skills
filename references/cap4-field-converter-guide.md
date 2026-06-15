# CAP4表单字段取值与转换器指南

基于致远CTP CAP4表单实战代码整理，重点覆盖：根据中文数据域/字段显示名取值、字段编码解析、子表遍历、动态字段映射、枚举/组织/附件/日期等转换器。

## 一、根据中文字段显示名取值

### 1.1 核心取值链路

```java
public static Object getFieldValue(FormDataBean bean, String displayName) {
    if (bean == null) {
        return null;
    }
    FormTableBean table = bean.getFormTable();
    if (table == null) {
        return null;
    }
    // displayName 是OA表单中的中文数据域名称/字段显示名
    FormFieldBean field = table.getFieldBeanByDisplay(displayName);
    if (field == null) {
        return null;
    }
    // field.getName() 通常是 field0001 / field0012 这种真实字段编码
    return bean.getFieldValue(field.getName());
}
```

### 1.2 常用类型封装

```java
public static String getString(FormDataBean bean, String displayName) {
    Object val = getFieldValue(bean, displayName);
    return val == null ? "" : val.toString().trim();
}

public static Long getLong(FormDataBean bean, String displayName) {
    Object val = getFieldValue(bean, displayName);
    if (val == null) return null;
    if (val instanceof Number) return ((Number) val).longValue();
    try { return Long.parseLong(val.toString().trim()); }
    catch (NumberFormatException e) { return null; }
}

public static Double getDouble(FormDataBean bean, String displayName) {
    Object val = getFieldValue(bean, displayName);
    if (val == null) return null;
    if (val instanceof Number) return ((Number) val).doubleValue();
    try { return Double.parseDouble(val.toString().trim()); }
    catch (NumberFormatException e) { return null; }
}
```

### 1.3 获取字段编码/字段对象

```java
// 根据中文显示名获取真实字段编码 field0012
public static String getFieldName(FormDataBean bean, String displayName) {
    FormFieldBean field = getFieldBean(bean, displayName);
    return field == null ? null : field.getName();
}

// 根据中文显示名获取 FormFieldBean
public static FormFieldBean getFieldBean(FormDataBean bean, String displayName) {
    if (bean == null || bean.getFormTable() == null) return null;
    return bean.getFormTable().getFieldBeanByDisplay(displayName);
}
```

## 二、表单赋值

### 2.1 按中文显示名赋值

```java
public static void setCellValue(FormDataBean bean, String displayName, Object value) {
    FormFieldBean cell = getFieldBean(bean, displayName);
    if (cell != null) {
        bean.addFieldValue(cell.getName(), value);
    }
}
```

> 在流程事前事件中修改字段时，一般用 `addFieldValue()`，不要额外调用 `saveOrUpdateFormData()`，避免影响流程引擎自身保存逻辑。

## 三、子表读取

### 3.1 单子表读取

```java
public static List<FormDataSubBean> getSubBeans(FormDataMasterBean master) throws Exception {
    Map<String, List<FormDataSubBean>> subs = master.getSubTables();
    if (subs != null && subs.size() > 0) {
        for (String key : subs.keySet()) {
            return subs.get(key); // 只适用于只有一个子表
        }
    }
    return null;
}
```

### 3.2 多子表读取

```java
public static List<List<FormDataSubBean>> getAllSubBeans(FormDataMasterBean master) throws Exception {
    Map<String, List<FormDataSubBean>> subs = master.getSubTables();
    List<List<FormDataSubBean>> allSubBeans = new ArrayList<>();
    if (subs != null && subs.size() > 0) {
        for (Map.Entry<String, List<FormDataSubBean>> entry : subs.entrySet()) {
            allSubBeans.add(entry.getValue());
        }
    }
    return allSubBeans;
}
```

### 3.3 根据子表显示名匹配映射

```java
private String getSubTableDisplay(FormDataSubBean subBean) {
    if (subBean != null && subBean.getFormTable() != null) {
        return subBean.getFormTable().getDisplay();
    }
    return "";
}
```

## 四、动态字段映射设计

### 4.1 字段映射配置对象

```java
public class FieldMapping extends BasePO {
    private String typeCode;         // 业务类型编码
    private String fieldScope;       // MAIN / SUB
    private String sourceType;       // FORM / FIXED
    private String subTableDisplay;  // 子表中文显示名
    private String targetParentKey;  // 子表输出父级数组key
    private String oaDisplayName;    // OA中文字段显示名
    private String targetField;      // 第三方接口字段名，支持 a.b.c 嵌套路径
    private String fieldType;        // STRING/NUMBER/INTEGER/BOOLEAN/DATE/ARRAY_STRING/OBJECT_ARRAY
    private String converterCode;    // 转换器编码
    private String fixedValue;       // 固定值
    private String defaultValue;     // 默认值
    private String datePattern;      // 日期格式
    private String skipIfBlank;      // 空值是否跳过
    private String requiredFlag;     // 是否必填
    private String enabled;          // 是否启用
}
```

### 4.2 MAIN/SUB字段构建规则

```java
// MAIN字段：输出到JSON根节点
{
  "fieldA": "valueA",
  "fieldB": 123
}

// SUB字段：按 subTableDisplay 找OA子表，再按 targetParentKey 输出数组
{
  "detail_lines": [
    {"name": "A", "amount": 100},
    {"name": "B", "amount": 200}
  ]
}
```

### 4.3 固定值和表单值

```java
private Object resolveValue(FormDataBean bean, FieldMapping mapping, BuildContext context) {
    Object rawValue;
    if ("FIXED".equals(mapping.getSourceType())) {
        rawValue = mapping.getFixedValue() == null ? "" : mapping.getFixedValue();
    } else {
        rawValue = FieldMapper.getFieldValue(bean, mapping.getOaDisplayName());
    }
    return valueConverterRegistry.convert(rawValue, mapping, context);
}
```

### 4.4 嵌套目标字段

支持 `targetField = "parent.child.name"`：

```java
// targetField: storage.note.text
{
  "storage": {
    "note": {
      "text": "xxx"
    }
  }
}
```

### 4.5 多个附件控件合并到同一数组字段

```java
// 多个 mapping 的 targetField 都是 attachment_ids，且转换结果是 List
// builder 中执行 addAll 合并，避免后一个附件覆盖前一个
if (value instanceof List) {
    Object existing = target.get(targetField);
    if (existing instanceof List) {
        ((List) existing).addAll((List) value);
        return;
    }
}
```

## 五、转换器注册表

### 5.1 转换器编码总览

| converterCode | 作用 | 输入 | 输出 |
|---------------|------|------|------|
| `DEFAULT` | 按字段类型基础转换 | 任意 | String/Number/Boolean/List等 |
| `DATE_YYYY_MM_DD` | 日期格式化 | Date/时间戳/String | `yyyy-MM-dd` 或自定义格式 |
| `ENUM_CODE` | OA枚举ID转枚举编码 | 枚举ID | 编码(可截前缀) |
| `MULTI_ENUM_CODE_ARRAY` | 多选枚举ID转编码数组 | `1,2,3` | `List<String>` |
| `ENUM_SHOW_VALUE` | OA枚举ID转显示值 | 枚举ID | 显示值 |
| `MULTI_ENUM_SHOW_VALUE_ARRAY` | 多选枚举ID转显示值数组 | `1,2,3` | `List<String>` |
| `ORG_UNIT_CODE` | 选单位ID转单位编码 | 单位ID | 单位code |
| `ORG_MEMBER_CODE` | 选人ID转人员编码 | 人员ID | 人员code |
| `ATTACHMENT_URL` | 简单附件ID拼下载URL | 附件ID | URL |
| `ATTACHMENT_BOUND_URL` | 上传控件取第一个附件URL | subReference | 完整下载URL |
| `ATTACHMENT_BOUND_FILENAME` | 上传控件取第一个文件名 | subReference | filename |
| `ATTACHMENT_LIST` | 上传控件取全部附件 | subReference | `List<Map>` |
| `TAXES_OBJECT` | 枚举转税项对象数组 | 枚举ID | `[{type_tax_use,name}]` |
| `RELATED_CONTRACT_ID` | 关联合同ID | 任意 | 按字段类型输出 |
| `ENUM_MAP` | 特殊值映射预留 | 任意 | 当前默认透传 |

### 5.2 字段类型转换

```java
private Object applyFieldType(Object rawValue, FieldMapping mapping) {
    String fieldType = nvl(mapping.getFieldType(), "STRING").toUpperCase();
    if (rawValue == null) return null;

    if ("BOOLEAN".equals(fieldType)) return toBoolean(rawValue);
    if ("INTEGER".equals(fieldType)) return toInteger(rawValue);
    if ("NUMBER".equals(fieldType)) return toDouble(rawValue);
    if ("DATE".equals(fieldType)) return formatDate(rawValue, mapping);
    if ("ARRAY_STRING".equals(fieldType)) return toStringList(rawValue);
    if ("ARRAY_INT".equals(fieldType)) return toIntegerList(rawValue);
    if ("ARRAY_NUMBER".equals(fieldType)) return toDoubleList(rawValue);
    if ("OBJECT_ARRAY".equals(fieldType)) return rawValue;
    return rawValue;
}
```

## 六、枚举转换

### 6.1 枚举ID转编码

```java
private String enumCode(Object rawValue, FieldMapping mapping, BuildContext context) throws Exception {
    Long enumId = toLong(rawValue);
    CtpEnumItem item = context.getEnumManager().getCtpEnumItem(enumId);
    if (item == null) return "";

    String code = item.getEnumItemCode();
    if (Strings.isBlank(code)) return "";

    // 部分项目中编码前4位作为分类前缀，如 abcd1 -> 1
    return removeEnumCodePrefix(code.trim());
}
```

### 6.2 枚举编码截前缀

```java
private String removeEnumCodePrefix(String enumItemCode) {
    if (Strings.isBlank(enumItemCode)) return "";
    String code = enumItemCode.trim();
    if (code.length() <= 4) return code;
    return code.substring(4);
}
```

> 是否截前缀是项目约定，不是平台通用规则。整理新项目时必须先确认枚举编码维护规则。

### 6.3 枚举ID转显示值

```java
private String enumShowValue(Object rawValue, FieldMapping mapping, BuildContext context) throws Exception {
    Long enumId = toLong(rawValue);
    CtpEnumItem item = context.getEnumManager().getCtpEnumItem(enumId);
    if (item == null) return "";
    String showValue = item.getShowvalue();
    return Strings.isBlank(showValue) ? "" : showValue.trim();
}
```

### 6.4 多选枚举

```java
private List<String> multiEnumCodeArray(Object rawValue, FieldMapping mapping, BuildContext context) {
    List<String> result = new ArrayList<>();
    if (rawValue == null) return result;
    String[] parts = rawValue.toString().split(",");
    for (String part : parts) {
        if (Strings.isBlank(part)) continue;
        String code = enumCode(part.trim(), mapping, context);
        if (Strings.isNotBlank(code)) result.add(code);
    }
    return result;
}
```

## 七、组织字段转换

### 7.1 选单位ID转单位编码

```java
private String orgUnitCode(Object rawValue, FieldMapping mapping, BuildContext context) throws Exception {
    Long unitId = toLong(rawValue);
    V3xOrgUnit unit = context.getOrgManager().getUnitById(unitId);
    String code = unit == null ? "" : unit.getCode();
    return Strings.isBlank(code) ? "" : code.trim();
}
```

### 7.2 选人ID转人员编码

```java
private String orgMemberCode(Object rawValue, FieldMapping mapping, BuildContext context) throws Exception {
    Long memberId = toLong(rawValue);
    V3xOrgMember member = context.getOrgManager().getMemberById(memberId);
    String code = member == null ? "" : member.getCode();
    return Strings.isBlank(code) ? "" : code.trim();
}
```

## 八、附件转换

### 8.1 上传控件字段值说明

CAP4上传控件字段原始值通常是 `subReference`，不是最终附件ID。处理步骤：

```java
// 1. 字段原始值 rawValue -> subReference
Long subRef = toLong(rawValue);

// 2. AttachmentManager 根据 subReference 获取 fileURL 列表
List<Long> fileUrls = attachmentManager.getBySubReference(subRef);

// 3. 根据 fileURL 获取 Attachment 对象
Attachment att = attachmentManager.getAttachmentByFileURL(fileUrl);

// 4. 用 att.getId() 拼接自定义下载接口
String url = oaBaseUrl + "/seeyon/ext/xxxAttachment.do?method=download&attachmentId=" + att.getId();
```

### 8.2 取第一个附件URL

```java
private String attachmentBoundUrl(Object rawValue, FieldMapping mapping, BuildContext context) {
    Attachment att = firstAttachment(rawValue, mapping, context);
    return att == null ? "" : buildAttachmentDownloadUrl(context, att);
}
```

### 8.3 取第一个附件文件名

```java
private String attachmentBoundFilename(Object rawValue, FieldMapping mapping, BuildContext context) {
    Attachment att = firstAttachment(rawValue, mapping, context);
    if (att == null || Strings.isBlank(att.getFilename())) return "";
    return att.getFilename().trim();
}
```

### 8.4 取全部附件列表

```java
private List<Map<String, Object>> attachmentList(Object rawValue, FieldMapping mapping, BuildContext context) {
    List<Map<String, Object>> result = new ArrayList<>();
    List<Long> fileUrls = attachmentFileUrlsBySubReference(rawValue, mapping, context);
    for (Long fileUrl : fileUrls) {
        Attachment att = context.getAttachmentManager().getAttachmentByFileURL(fileUrl);
        if (att == null || att.getFileUrl() == null) continue;
        Map<String, Object> item = new LinkedHashMap<>();
        item.put("name", att.getFilename());
        item.put("filename", att.getFilename());
        item.put("url", buildAttachmentDownloadUrl(context, att));
        result.add(item);
    }
    return result;
}
```

输出示例：

```json
[
  {"name":"附件A.pdf", "filename":"附件A.pdf", "url":"https://oa.example.com/seeyon/ext/xxxAttachment.do?method=download&attachmentId=123"}
]
```

## 九、构建上下文 BuildContext

转换器不要反复从 `AppContext` 取Bean，应在Service层一次性装配上下文。

```java
BuildContext context = new BuildContext();
context.setTypeConfig(typeConfig);
context.setMasterBean(masterBean);
context.setEnumManager((EnumManager) AppContext.getBean("enumManager"));
context.setOrgManager((OrgManager) AppContext.getBean("orgManager"));
context.setAttachmentManager((AttachmentManager) AppContext.getBean("attachmentManager"));
context.setOaBaseUrl(AppContext.getSystemProperty("plugin.oa.baseurl"));
```

## 十、空值/默认值/必填策略

```java
private void putIfNeeded(Map<String, Object> target, String targetField, Object value, FieldMapping mapping) {
    boolean blank = isBlankValue(value);

    // 必填为空：记录error，但是否中断流程由项目决定
    if (blank && "1".equals(mapping.getRequiredFlag())) {
        LOGGER.error("必填字段为空: " + mapping.getOaDisplayName());
    }

    // 默认值
    if (blank && Strings.isNotBlank(mapping.getDefaultValue())) {
        value = mapping.getDefaultValue();
        blank = false;
    }

    // 空值跳过
    if (blank && "1".equals(mapping.getSkipIfBlank())) {
        return;
    }

    target.put(targetField, value);
}
```

## 十一、检查清单

- [ ] 中文显示名是否与OA表单数据域完全一致
- [ ] 子表映射的 `subTableDisplay` 是否与子表显示名一致
- [ ] 枚举项是否维护了编码（`enumItemCode`）
- [ ] 是否需要截取枚举编码前缀（项目约定）
- [ ] 选人/选单位字段是否使用 `ORG_MEMBER_CODE` / `ORG_UNIT_CODE`
- [ ] 附件字段是否为上传控件，是否使用 `ATTACHMENT_BOUND_*`
- [ ] 多附件是否需要合并到同一个数组字段
- [ ] 日期格式是否符合第三方要求
- [ ] 空值是否跳过、默认值、必填策略是否配置
- [ ] 日志中不要输出敏感配置和真实接口地址
