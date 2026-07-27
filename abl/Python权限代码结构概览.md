项目的 Python SDK 主要指 abl-python-common 这个独立的代码库。它是所有 Python 微服务（如 pcrm ）接入权限体系、服务发现和基础框架的“标准插件包”。

### SDK 代码结构概览
SDK 的核心代码位于 abl_common 目录下，按功能分为以下几个部分：
 1. 认证与权限中间件 (Middleware)
- auth.py ： SDK 的“大门” 。
  - 负责从 HTTP Header 中解析用户信息。
  - 调用 PermissionClient 从 Java 端抓取行级/列级权限规则。
  - 将解析后的权限存入 UserContext 供业务逻辑使用。 2. 核心能力 (Core)
- context.py ： 上下文管理器 。利用 contextvars 保证在异步请求（FastAPI）中，每个请求的权限数据是隔离且安全的。
- permission_client.py ： 通讯客户端 。封装了所有与 Java 权限中心（ abl-infra-system ）交互的 HTTP 接口。
- masking.py ： 数据脱敏引擎 。通过 Pydantic 的序列化拦截，自动将受限字段（如 line_num ）处理为 *** 。
- rule_engine.py ： 本地规则引擎 。当远程服务不可用时，提供基础的本地权限兜底。 3. 服务注册与发现 (Nacos)
- registry.py ： 服务寻址 。负责将环境变量中的 http://abl-infra-system 转换成 Nacos 中注册的真实服务器 IP。 4. 数据库权限增强 (DB)
- permission.py ： SQL 自动注入 。利用 SQLAlchemy 的事件监听机制，在执行 SQL 前自动追加权限过滤子句（如 WHERE u_ext_slp_email IN (...) ）。
### 为什么这样设计？
这种“SDK 模式”实现了 “一次开发，处处复用” 。当 Java 端的权限模型发生变化时，我们只需要更新这个 SDK 里的逻辑，所有 Python 业务模块（PC-RM, BI 等）只需升级 SDK 版本即可获得最新的权限保护功能，而不需要改动业务代码。



这里是 Python SDK (abl-python-common) 在整个权限体系中的全链路流程图，展示了从用户发起请求到数据脱敏返回的完整过程：

100 %

### 关键环节说明
1. 服务发现 (Nacos) ：
   - 通过 PERMISSION_SERVICE_URL=http://abl-infra-system 配置，SDK 不再死磕固定 IP，而是动态寻址，保证了高可用。
2. 上下文隔离 (ContextVars) ：
   - 在 context.py 中，每个请求的权限都是独立的，不会出现“主管 A 看到了主管 B 的下属数据”这种并发错乱。
3. 两级过滤 ：
   - 行级 (Row) ：在 SQL 层面解决“能看哪些行”，通过 doris_ship_summary_repository.py 注入 WHERE 条件。
   - 字段级 (Field) ：在返回对象层面解决“能看哪些列”，通过 masking.py 自动把敏感字段变星号。
   这个流程确保了 Java 端定义规则，Python 端严格执行 ，实现了跨语言的权限一致性。

为了方便你在文档中使用，我将上述流程整理为纯 Markdown 结构化描述和 Mermaid 源码两个版本：

### 1. 结构化 Markdown 流程说明
第一阶段：请求接入与身份识别

- 用户/前端 发起请求，携带 JWT Token。
- 网关 (Gateway) 校验 Token，并将解析出的 X-User-Id 注入 Header。
- Python 服务 接收请求， auth.py 中间件开始拦截。
第二阶段：动态寻址与权限抓取 (SDK 核心)

- 寻址 ：SDK 读取 PERMISSION_SERVICE_URL ，通过 registry.py 从 Nacos 动态获取 Java 权限服务的真实 IP。
- 抓取 ： permission_client.py 向 Java 端发起三次内部调用：
  1. 获取用户关联的所有 邮箱列表 （包含下属邮箱）。
  2. 获取 行权限规则 （SQL 过滤条件）。
  3. 获取 列权限规则 （字段脱敏策略）。
  第三阶段：上下文注入

- 存储 ：SDK 将抓取到的所有数据封装进 UserContext ，利用 Python 的 ContextVars 实现请求间的 逻辑隔离 。
第四阶段：数据库执行 (行过滤)

- 注入 ：在 doris_ship_summary_repository.py 执行查询时，SDK 自动将规则注入：
  - SELECT * FROM table WHERE ... AND (u_ext_slp_email IN ('email1', 'email2'))
  第五阶段：结果返回 (列脱敏)

- 遮掩 ：数据返回给前端前， masking.py 拦截 Pydantic 序列化过程。
- 输出 ：根据规则，将敏感字段（如 line_num ）替换为 *** ，最终输出给用户。
### 2. Mermaid 源码 (可直接粘贴到 Markdown 编辑器)
```mermaid
sequenceDiagram
    participant U as 用户 (Browser/API)
    participant G as Gateway (网关)
    participant SDK as Python SDK (Middleware)
    participant N as Nacos (注册中心)
    participant JAVA as abl-infra-system (Java 权限服务)
    participant DB as Doris/PG (数据库)

    U->>G: 发起请求 (携带 Token)
    G->>G: 校验 Token, 注入 X-User-Id
    G->>SDK: 转发请求
    
    Note over SDK: [GatewayAuthMiddleware] 拦截请求
    SDK->>N: 查询 "abl-infra-system" 实例 IP
    N-->>SDK: 返回 172.16.1.35:8081
    
    SDK->>JAVA: 获取用户邮箱 & 部门 ID
    SDK->>JAVA: 获取行权限 (Row Rules) & 字段规则 (Field Rules)
    JAVA-->>SDK: 返回规则 (例如: u_ext_slp_email IN (${emails}))
    
    Note over SDK: [UserContext] 存储当前请求的权限快照
    
    SDK->>DB: 业务查询 (Repository 层)
    Note over DB: [SQL 注入] 自动拼接 WHERE 子句<br/>注入下属邮箱列表
    DB-->>SDK: 返回原始数据行
    
    Note over SDK: [Masking] Pydantic 序列化拦截
    Note over SDK: 检查字段规则 (如: line_num -> MASK)
    
    SDK-->>G: 返回已脱敏的 JSON
    G-->>U: 最终结果