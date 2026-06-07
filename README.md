# 致远CTP二开技能库

基于多个致远CTP客开项目的实战模式提炼，覆盖V8.2SP1/V9.0SP1/V10.0SP1版本。

## 目录结构

```
zhiyuanskill-yw/
├── README.md                       # 本文件
├── CHANGELOG.md                    # 变更记录
├── SKILL.md                        # 汇总Skill（入口）
├── .gitignore
├── agents/
│   └── openai.yaml                 # Agent配置
└── references/                     # 参考文档
    ├── zhiyuan-sources.md          # 官方文档索引 + API速查
    ├── cap4-guide.md               # CAP4表单操作指南
    ├── api-integration.md          # 第三方API集成模式
    ├── jsp-guide.md                # JSP维护页面模板
    └── bpm-event-guide.md          # 流程事件开发完整指南
```

## 规划中的目录（后续补充）

```
├── skills/                         # 独立子技能（精细化拆分）
│   ├── ctp-new-plugin/             # 新建插件规范
│   ├── ctp-cap4-form/              # CAP4表单操作
│   ├── ctp-event-listener/         # 事件监听
│   ├── ctp-super-node/             # 超级节点
│   ├── ctp-rest-api/               # REST资源
│   ├── ctp-http-client/            # 第三方API集成
│   ├── ctp-jsp-page/               # JSP维护页面
│   ├── ctp-sso/                    # SSO单点登录
│   └── ctp-db-access/              # 数据访问
└── templates/                      # 代码模板（可直接复制）
```

## 使用方式

在Claude Code中加载Skill后，AI会自动：
- 遵循致远CTP插件开发规范生成代码
- 使用正确的事件监听模式
- 优先低侵入实现方案

## 覆盖模式

| 模式分类 | 覆盖内容 |
|---------|---------|
| 插件开发 | 目录结构、配置规范、分层架构、初始化器 |
| Controller | BaseController/BaseResource、Spring XML映射、免登录配置 |
| 事件监听 | 协同、公文、CAP4表单、文件下载、组织变更14+事件 |
| CAP4表单 | 字段取值、子表遍历、枚举转换、附件处理、自定义控件 |
| 超级节点 | 标准节点、轻量级抽象基类多类型模式 |
| 数据访问 | JDBCAgent/DBKit、DBAgent、CAP4FormDataDAO |
| API集成 | OkHttp/Apache/HttpURLConnection、Token缓存、SM4/AES签名 |
| 补偿机制 | 待办推送补偿、定时重试、退避策略 |
| SSO | 握手模式、Controller跳转模式、票据加解密 |
| JSP页面 | CRUD维护页模板、分页、弹窗、AJAX交互 |

## 已覆盖的版本

- CTP V8.2SP1 — 插件化、Hibernate映射、门户定制
- CTP V9.0SP1 — CAP4控件、水印、组织同步、批量打印
- CTP V10.0SP1 — OkHttp、动态字段映射、轻量超级节点

## 贡献方式

1. 从新项目中发现模式 → 提Issue
2. 确认后更新对应Skill文件
3. 更新CHANGELOG.md
4. Push到仓库
