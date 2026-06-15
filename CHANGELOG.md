# 变更记录

## [0.5.0] - 2026-06-08

### 新增
- `references/super-node-guide.md` — 超级节点开发指南
  - BaseSuperNodeAction完整模板
  - 轻量级超级节点抽象模式
  - SuperNodeResponse返回值规范
  - 从超级节点参数获取CAP4表单数据
  - Spring XML注册、服务层推送、日志与容错规范
- `references/cap4-field-converter-guide.md` — CAP4字段取值与转换器指南
  - 根据中文数据域/字段显示名取值
  - 获取field0012等真实字段编码
  - 主表/子表/多子表处理
  - 动态字段映射设计
  - 枚举编码/显示值、多选枚举、人员/单位编码、附件URL/文件名/列表等转换器

## [0.4.0] - 2026-06-08

### 新增
- `references/org-architecture-guide.md` — 组织架构开发完整指南
  - 基于平台源码(OrgManagerDirectImpl) + SeeyonAPI + 客开实战
  - 39个组织事件完整清单(单位/部门/人员/岗位/职级/组/兼职)
  - REST API驱动组织同步完整架构
  - 人员/部门/兼职同步核心代码流程
  - 组织主数据扩展(CAP4公式引擎集成)
  - 常用组织API速查表

## [0.3.0] - 2026-06-07

### 新增
- `references/seeyonapi-event-index.md` — SeeyonAPI 8.2 事件完整索引
  - 覆盖20个模块、180+事件
  - 每个事件列出完整类路径和关键属性
  - 附场景→事件映射速查表
  - 数据来源：https://open.seeyoncloud.com/seeyonapi/8.2/event/

## [0.2.0] - 2026-06-07

### 新增
- `references/bpm-event-guide.md` — 流程事件开发完整指南
  - 基于致远官方文档 + 项目实战代码
  - 覆盖 AbstractWorkflowEvent 和 @ListenEvent 两套机制
  - 包含前端事件拦截 `$.ctp.bind()`
  - 完整的外部集成实战模式（含日志、防重复、补偿）

## [0.1.1] - 2026-06-07

### 安全修复
- 移除所有客户名称、系统编码、URL路径等敏感信息
- 项目名称全部替换为版本号+模式描述

## [0.1.0] - 2026-06-07

### 初始版本
- 基于多个致远CTP客开项目分析生成
- 覆盖V8.2SP1/V9.0SP1/V10.0SP1三个版本
- 创建汇总Skill：插件规范、CAP4表单、事件监听、超级节点、API集成、JSP页面
