# ABL（艾博）平台 Sprint Xia开发文档

> 版本：v1.0 
> 日期：2026-06-02  
> 状态：待评审

---

## 目录

1. [系统概述](#1-系统概述)
2. [系统架构设计](#2-系统架构设计)
3. [核心功能模块](#3-核心功能模块)
4. [功能详细分析](#4-功能详细分析)
5. [数据仓库设计](#5-数据仓库设计)
6. [前端架构设计](#6-前端架构设计)
7. [监控与运维](#7-监控与运维)
8. [附录](#8-附录)

---

## 1. 系统概述

### 1.1 平台定位

ABL（艾博）平台融合经典微服务架构、低代码开发、AI Agentic 等技术路线，打造适合中小规模企业的新型数智化中台。平台基于**租户（Tenant）**和**应用（App）**进行分离式管理，内置低代码服务，支持原生业务应用开发，同时支持外部应用托管。

### 1.2 应用分类

| 类型 | 说明 | 示例 |
|------|------|------|
| 低代码应用 (Lowcode App) | 通过可视化设计器构建 | 自定义审批流、数据收集表单 |
| 内置应用 (Built-in App) | 平台预置的标准应用 | BI 数据分析、CRM |
| 外部应用 (External App) | 第三方系统接入托管 | 企业已有 ERP 的扩展模块 |

### 1.3 核心能力矩阵

```
┌─────────────────────────────────────────────────────────┐
│                    ABL 平台能力层                        │
├─────────────┬─────────────┬─────────────┬─────────────┤
│  低代码引擎  │  AI 服务层   │  数据仓库   │  统一网关   │
│  ├─表单设计  │  ├─Agent    │  ├─实时同步  │  ├─鉴权    │
│  ├─流程引擎  │  ├─LLM网关  │  ├─BI分析   │  ├─路由    │
│  ├─页面搭建  │  ├─隐私网关  │  ├─数据服务 │  ├─限流    │
│  └─数据模型  │  └─轻量模型  │  └─物化视图 │  └─负载均衡 │
└─────────────┴─────────────┴─────────────┴─────────────┘
```

### 1.4 系统管理范围

- **系统用户管理**：平台级超级管理员、运维人员
- **租户管理**：多租户隔离、资源配额、计费
- **应用管理**：
  - 应用生命周期（创建/配置/发布/下线）
  - 应用内用户管理（与系统用户隔离）
  - 应用级权限管理（RBAC）
- **认证管理**：
  - LDAP 认证
  - OAuth2 Authorization Server
  - OpenID Connect 1.0
  - 企微授权认证

### 1.5 通用服务

| 服务 | 功能 | 关键特性 |
|------|------|----------|
| AI Service | Agent 前台与后台服务 | Tools 注册、Memory、ReAct Loop |
| LLM Gateway | 统一模型调用 | 应用级免 Key、数据脱敏、流量控制 |
| Agent 集群管理 | 多 Agent 编排 | 负载均衡、会话保持 |
| Privacy Gateway | 安全护栏 | 规则引擎、语义保持、分级还原 |
| 轻量模型 | NER、OCR 服务 | 本地部署、低延迟 |
| 权限管理 | RBAC + 映射 | OAuth/LDAP Group 与角色映射 |

---

## 2. 系统架构设计

### 2.1 总体架构

ABL 采用**异构微服务架构**，前后端分离，多技术栈协同：

```
┌──────────────────────────────────────────────────────────────┐
│                        接入层                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  abl-ui-public │  │ abl-ui-admin  │  │ abl-lowcode-frontend │  │
│  │   (用户端)    │  │   (管理端)    │  │    (低代码设计器)      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                        网关层                                 │
│              ┌─────────────────────────┐                    │
│              │      abl-gateway        │                    │
│              │  ├─统一鉴权 (JWT/OAuth2)  │                    │
│              │  ├─路由转发               │                    │
│              │  ├─负载均衡               │                    │
│              │  ├─限流熔断 (Sentinel)    │                    │
│              │  └─请求日志               │                    │
│              └─────────────────────────┘                    │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                        服务层                                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐│
│  │  abl-infra   │ │ abl-lowcode  │ │      abl-modules        ││
│  │  ├─system    │ │  ├─metadata  │ │  ├─crm (本期)           ││
│  │  ├─security  │ │  └─runtime   │ │  └─xxx (后续)           ││
│  │  └─monitor   │ │              │ │                         ││
│  └─────────────┘ └─────────────┘ └─────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    abl-python 服务群                     ││
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       ││
│  │  │ai-common│ │ai-agent │ │dw-job   │ │dw-sync   │       ││
│  │  │dw-api   │ │         │ │         │ │         │       ││
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       ││
│  └─────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                        数据层                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐│
│  │  PostgreSQL  │  │ Apache Doris │  │      Nacos           ││
│  │  (业务数据)   │  │  (数据仓库)   │  │   (注册/配置中心)      ││
│  └─────────────┘  └─────────────┘  └─────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

### 2.2 技术栈

| 层级 | 技术选型 | 版本/说明 |
|------|----------|-----------|
| 后端框架 | Java Spring Cloud Alibaba | 2023.x (兼容 Spring Boot 3.x) |
| 服务治理 | Nacos | 2.3+ (注册中心 + 配置中心) |
| 网关 | Spring Cloud Gateway | 4.x |
| 熔断限流 | Sentinel | 1.8+ |
| 分布式事务 | Seata | 1.7+ (按需启用) |
| 数据仓库 | Apache Doris | 2.1+ |
| 业务数据库 | PostgreSQL | 15+ |
| 缓存 | Redis | 7.x |
| 消息队列 | RocketMQ | 5.x (按需启用) |
| AI 服务 | Python FastAPI | 0.110+ |
| 前端框架 | React 18 + TypeScript | 函数组件 + Hooks |
| 前端状态 | Zustand + React Query | 轻量状态 + 服务端状态 |
| 前端组件 | Ant Design 5.x | 企业级 UI |
| BI 组件 | Apache Superset | 3.x (嵌入式) |
| 监控 | Micrometer + Prometheus + Grafana | 指标采集与可视化 |

### 2.3 服务划分与职责

#### 2.3.1 abl-gateway（统一网关）

**核心职责**：
- 所有后端服务的统一入口
- 服务注册发现（Nacos 集成）
- 统一鉴权（JWT 验证、OAuth2 Token 校验）
- 路由转发与负载均衡
- 限流熔断（Sentinel 集成）
- 请求日志与审计

**非职责**：
- 不涉及业务逻辑处理
- 不直接访问数据库

#### 2.3.2 abl-infra（基础设施服务群）

| 服务 | 核心功能 | 数据存储 |
|------|----------|----------|
| abl-infra-system | 系统用户、租户、应用、角色权限、字典、菜单 | PostgreSQL |
| abl-infra-security | 认证、URL 拦截、单点登录、Token 管理 | PostgreSQL + Redis |
| abl-infra-monitor | 系统监控、业务异常追踪、健康检查 | Prometheus + 时序数据库 |

**职责边界说明**：
- `abl-infra-system` 管理**平台级**资源（租户、系统用户）
- `abl-infra-security` 处理**安全认证**全流程，不存储业务数据
- `abl-infra-monitor` 聚焦**框架级监控**（服务健康、调用链路、业务异常），非传统 PGA（Process Global Area）方式，而是基于 Micrometer 指标采集 + 日志聚合

#### 2.3.3 abl-lowcode（低代码服务群）

| 服务 | 核心功能 | 说明 |
|------|----------|------|
| abl-lowcode-metadata | 元数据管理 | Entity, EntityMeta, EntityRelation, FieldMeta, ModuleMeta, PageMeta, QueryMeta, EntityData, FieldRelease, ModuleRelease, PageRelease |
| abl-lowcode-runtime | 运行时引擎 | 动态表单渲染、数据 CRUD、规则执行 |

**设计原则**：
- 元数据与运行时分离，支持版本发布机制
- 设计时（Design Time）与运行时（Runtime）状态隔离

#### 2.3.4 abl-modules（业务模块服务群）

| 服务 | 本期范围 | 后续扩展 |
|------|----------|----------|
| abl-module-crm | 订单管理模块 | 客户管理、商机管理、合同管理 |
| abl-module-xxx | - | 按需新增 |

**模块规范**：
- 每个模块独立数据库 Schema（PostgreSQL）
- 模块间通过 Feign Client 或消息队列通信
- 模块内自包含业务逻辑、数据访问、对外 API

#### 2.3.5 abl-python（AI 与数据仓库服务群）

| 服务 | 核心功能 | 职责边界 |
|------|----------|----------|
| abl-ai-common | 公共 AI 功能：LLM 调用、数据脱敏、异常检测 | 提供 SDK 供其他服务调用 |
| abl-ai-agent | Agent 框架：Tools 注册、Memory、ReAct Loop、Pre/Post Process | 不直接对外暴露 HTTP，由网关路由 |
| abl-dw-job | 数据仓库**批量/自定义** ETL 任务 | 支持在线（低代码）创建任务，低频调度 |
| abl-dw-sync | 数据仓库**实时/准实时**同步服务 | 管理与适配多数据源（MySQL、SQL Server、PostgreSQL），基于 Change Tracking / CDC |
| abl-dw-api | 数据仓库数据访问与操作 API | 对外提供统一的数据查询接口，支持 SQL 透传和封装 API |

**职责边界明确化**：
- `abl-dw-sync` 负责**数据管道**（从源到 Doris ODS），关注实时性和一致性
- `abl-dw-job` 负责**数据加工**（ODS → DWS → ADS），关注复杂转换和调度
- `abl-dw-api` 负责**数据消费**（对外查询接口），关注查询性能和权限控制

### 2.4 非服务类库

| 类别 | 类库 | 核心功能 |
|------|------|----------|
| abl-infra | abl-infra-common | 公共定义、缓存管理、枚举、验证、工具类 |

---

## 3. 核心功能模块

### 3.1 平台框架

- 异构微服务系统搭建
- 基于 Java Spring Cloud Alibaba + Python FastAPI + Nacos + React
- 前后端分离的综合业务管理平台

### 3.2 系统管理

- 系统运维用户管理
- 租户管理（多租户隔离）
- 认证管理（LDAP、OAuth2、OIDC、企微）
- 日志查看与审计

### 3.3 应用管理

- 应用维护（创建、配置、发布、下线）
- 应用内用户管理
- 应用级权限管理（RBAC）

### 3.4 数仓管理

- 数据模型管理（ODS、DWS、ADS 分层）
- 数据同步任务管理（CT 同步配置、调度监控）
- 物化视图管理

### 3.5 BI 数据分析

- 订单汇总看板
- 库存查询（实时库存、库存水位、呆滞库存）
- 代理商明细
- 交付相关统计

### 3.6 CRM 应用

- 订单管理模块（本期唯一模块）
- 后续扩展：客户管理、商机管理、合同管理

---

## 4. 功能详细分析

### 4.1 BI 功能分析

#### 4.1.1 订单汇总看板

| 功能 | 分析 | 数据源 | 备注 |
|------|------|--------|------|
| 新接单统计 | 指定日期内新增订单数 | `U_ORDR`、`U_ORDR1` | 自定义订单表 |
| 已交付统计 | 指定日期内交货单数量 | `ODLN`、`DLN1` | 含寄售模式判断 |
| 待交订单 | 未清订单数量 | `U_ORDR` | 按状态筛选 |
| 超期应收 | 逾期应收账款金额 | 业财数据库 | 仅显示金额，明细跳转财务系统 |
| 代理商可用库存 | 各代理商可销售库存 | `T_OSFD`、`T_OSFD1`、`U_OSFD2` | 取 `inaction_stock` 字段 |
| 超期订单 | 超过下次交货日期的未清数量 | `T_OSFD`、`T_OSFD1`、`U_OSFD2` | 需计算交货日期差 |
| 交货计划审批中 | 待审批交货计划数量 | 外部接口 | 从审批系统读取 |
| 交货待办中 | 待处理交货任务数量 | 外部接口 | 从任务系统读取 |
| 已预约发货金额 | 已排程发货的订单金额 | 外部接口 | 从物流系统读取 |

**系统边界说明**：
- 寄售部分如需统计，考虑在汇聚时直接查询 SAP（不同步原始数据到 Doris）
- 超期应收通过业财数据库查询，仅展示汇总金额，明细需跳转外部系统
- 外部接口数据（审批、任务、物流）本期仅做展示，不做本地存储

#### 4.1.2 库存查询

| 功能 | 分析 | 数据源 | 实现方式 |
|------|------|--------|----------|
| 实时库存查询 | 当前各仓库各批次库存 | `T_OBTN`、`T_OBTQ`、`U_NHOD` | Doris 直接查询 |
| 库存水位查询 | 含预警线判断 | DWS 层预计算 | 可配置预警阈值 |
| 呆滞库存查询 | 长期未动销库存 | DWS 层预计算 | 按呆滞天数分级 |

**BI 页面技术方案**：
- 大部分页面使用 **Apache Superset** 嵌入式组件搭建
- 个别特殊形式（复杂交互、自定义算法）的统计分析需要定制 React 页面
- Superset 与 ABL 统一鉴权打通（通过 JWT 透传）

### 4.2 CRM 应用 — 订单管理

本期 CRM 应用仅包含**订单管理**模块，支持以下功能：

#### 4.2.1 功能列表

| 功能 | 说明 | 技术要点 |
|------|------|----------|
| 询价单查询 | 查询历史询价记录 | 对接 SAP B1 查询接口 |
| 验证客户料号 | 校验客户物料编码有效性 | 调用 SAP B1 主数据接口 |
| MOQ 验证 | 校验最小订单量 | 调用 SAP B1 物料主数据 |
| 订单提交 | 提交正式订单到 SAP B1 | 调用 SAP B1 DI API |

#### 4.2.2 AI 辅助订单录入

- 支持直接导入订单附件（PDF、Excel、图片）
- AI 自动解析附件内容，提取关键字段：
  - 客户信息
  - 物料编码 / 客户料号
  - 数量、价格、交货日期
- 自动生成订单草稿，进入人工确认流程

#### 4.2.3 订单状态机

```
[草稿] → [待审批] → [已提交] → [已同步]
  ↑         ↓           ↓
  └──── [已驳回] ←──── [同步失败]
```

**状态说明**：
- **草稿**：AI 解析或手动创建，可编辑
- **待审批**：提交审批流，等待审批人处理
- **已驳回**：审批不通过，退回草稿状态
- **已提交**：审批通过，已调用 SAP B1 接口
- **同步失败**：SAP B1 接口调用异常，需人工介入重试
- **已同步**：SAP B1 返回成功确认

#### 4.2.4 异常处理机制

- **AI 解析失败**：提示用户手动录入，保留附件备查
- **SAP B1 接口超时**：进入重试队列（最多 3 次），失败转人工
- **数据校验失败**：明确提示具体字段错误，阻止提交

#### 4.2.5 [Placeholder] 后续扩展功能

> 以下功能本期不实现，预留架构扩展点：

- [ ] 客户 360° 视图（客户主数据、交易历史、信用额度）
- [ ] 商机管理（销售漏斗、阶段推进、赢单预测）
- [ ] 合同管理（合同模板、电子签章、履约跟踪）
- [ ] 售后服务（工单管理、服务派工、客户满意度）

---

## 5. 数据仓库设计

### 5.1 架构分层

```
┌─────────────────────────────────────────┐
│              ADS 应用层                  │  ← BI 看板、报表直接查询
│         （物化视图 / 轻量汇总）           │
├─────────────────────────────────────────┤
│              DWS 汇总层                  │  ← 主题宽表、实时库存
│         （Unique Key，面向业务）          │
├─────────────────────────────────────────┤
│              ODS 贴源层                  │  ← 原始数据同步
│         （Unique Key，当前快照）          │
└─────────────────────────────────────────┘
```

**设计原则**：
- ODS 层保留**当前最新快照**，不保留历史变更轨迹（轨迹在源系统 SQL Server CT 中保留）
- DWS 层面向**业务主题**构建宽表，支持实时查询
- ADS 层通过**物化视图**自动维护，减少重复计算

### 5.2 数据源与同步策略

#### 5.2.1 SQL Server Change Tracking 配置

```sql
-- 数据库启用 CT
ALTER DATABASE [YourDB] SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON);

-- 交付信息跟踪
ALTER TABLE ODLN ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
ALTER TABLE DLN1 ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
ALTER TABLE T_OBTN ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
ALTER TABLE U_NHOD ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
ALTER TABLE T_OBTQ ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);

-- 订单信息跟踪
ALTER TABLE U_ORDR ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
ALTER TABLE U_ORDR1 ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);

-- 销售预测跟踪
ALTER TABLE T_OSFD ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
ALTER TABLE T_OSFD1 ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
ALTER TABLE U_OSFD2 ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
```

**Retention 策略**：
- 默认 2 天，根据 `syscommittab` 实际行数和空间动态调整
- 同步程序至少每天运行一次，确保不丢失变更

#### 5.2.2 监控 SQL

```sql
-- 1. 数据库级 CT 配置
SELECT 
    DB_NAME() AS database_name,
    is_auto_cleanup_on,
    retention_period,
    retention_period_units_desc
FROM sys.change_tracking_databases
WHERE database_id = DB_ID();

-- 2. 当前变更追踪版本
SELECT CHANGE_TRACKING_CURRENT_VERSION() AS current_version;

-- 3. 启用 CT 的用户表及版本信息
SELECT 
    OBJECT_NAME(ct.object_id) AS table_name,
    ct.object_id,
    ct.min_valid_version,
    CHANGE_TRACKING_CURRENT_VERSION() - ct.min_valid_version AS version_gap,
    ct.is_track_columns_updated_on
FROM sys.change_tracking_tables ct;

-- 4. CT 内部表空间占用
SELECT 
    o.name AS internal_table_name,
    p.rows AS row_count,
    SUM(a.total_pages) * 8 AS total_space_kb,
    SUM(a.used_pages) * 8 AS used_space_kb
FROM sys.objects o
INNER JOIN sys.partitions p ON o.object_id = p.object_id
INNER JOIN sys.allocation_units a ON p.partition_id = a.container_id
WHERE o.type = 'IT'
GROUP BY o.name, p.rows
HAVING o.name LIKE '%change_tracking%' 
    OR o.name LIKE 'syscommittab%'
ORDER BY total_space_kb DESC;

-- 5. syscommittab 单独查看
EXEC sp_spaceused 'sys.syscommittab';
```

### 5.3 ODS 层表设计

> **模型选择**：使用 `UNIQUE KEY`，保留当前最新快照，同一主键多次变更只保留最新状态。
> **生产环境注意**：`replication_num` 开发环境为 1，生产环境必须改为 3。

#### 5.3.1 ODLN 交货主表

```sql
CREATE TABLE IF NOT EXISTS ods_odln (
    doc_entry BIGINT COMMENT 'SAP 单据编号',
    card_code VARCHAR(50) COMMENT '客户代码',
    card_name VARCHAR(100) COMMENT '客户名称',
    doc_date DATE COMMENT '单据日期',
    doc_rate DECIMAL(18,6) COMMENT '汇率',
    tax_date DATE COMMENT '计税日期',
    canceled VARCHAR(1) COMMENT '取消标记 (Y/N)',
    doc_status VARCHAR(1) COMMENT '单据状态',
    u_order_num VARCHAR(50) COMMENT '订单号',
    u_end_dlv_date DATE COMMENT '最终交货日期',
    ct_version BIGINT COMMENT 'CT 版本号',
    ct_operation VARCHAR(10) COMMENT '操作类型 (I/U/D)',
    sync_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '同步时间'
)
UNIQUE KEY(doc_entry)
DISTRIBUTED BY HASH(doc_entry) BUCKETS 4
PROPERTIES (
    "replication_num" = "1",
    "enable_unique_key_merge_on_write" = "true"
);
```

#### 5.3.2 DLN1 交货明细表

```sql
CREATE TABLE IF NOT EXISTS ods_dln1 (
    doc_entry BIGINT COMMENT 'SAP 单据编号',
    line_num INT COMMENT '行号',
    u_sub_card_code VARCHAR(50) COMMENT '子客户代码',
    u_sub_card_name VARCHAR(100) COMMENT '子客户名称',
    item_code VARCHAR(50) COMMENT '物料编码',
    u_frgn_name VARCHAR(200) COMMENT '外文名称',
    u_opn VARCHAR(50) COMMENT 'OPN',
    u_cus_name VARCHAR(100) COMMENT '客户名称',
    ship_date DATE COMMENT '发货日期',
    quantity DECIMAL(18,6) COMMENT '数量',
    u_quantity DECIMAL(18,6) COMMENT '自定义数量',
    u_price_af_vat DECIMAL(18,6) COMMENT '含税单价',
    line_total DECIMAL(18,6) COMMENT '行总计',
    ct_version BIGINT COMMENT 'CT 版本号',
    ct_operation VARCHAR(10) COMMENT '操作类型',
    sync_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '同步时间'
)
UNIQUE KEY(doc_entry, line_num)
DISTRIBUTED BY HASH(doc_entry) BUCKETS 4
PROPERTIES (
    "replication_num" = "1",
    "enable_unique_key_merge_on_write" = "true"
);
```

#### 5.3.3 T_OBTN 批次序列表

```sql
CREATE TABLE IF NOT EXISTS ods_obtn (
    abs_entry BIGINT COMMENT '绝对条目号',
    item_code VARCHAR(50) COMMENT '物料编码',
    sys_number BIGINT COMMENT '系统编号',
    dist_number VARCHAR(50) COMMENT '批次号',
    u_frgn_name VARCHAR(200) COMMENT '外文名称',
    u_good_rate VARCHAR(50) COMMENT '良率',
    u_dc VARCHAR(50) COMMENT 'DC',
    u_box_num VARCHAR(50) COMMENT '箱号',
    u_order_num VARCHAR(50) COMMENT '订单号',
    u_maverick_flag VARCHAR(50) COMMENT 'Maverick标记',
    u_soft_bin VARCHAR(50) COMMENT 'Soft Bin',
    ct_version BIGINT COMMENT 'CT 版本号',
    ct_operation VARCHAR(10) COMMENT '操作类型',
    sync_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '同步时间'
)
UNIQUE KEY(abs_entry)
DISTRIBUTED BY HASH(abs_entry) BUCKETS 4
PROPERTIES (
    "replication_num" = "1",
    "enable_unique_key_merge_on_write" = "true"
);
```

#### 5.3.4 T_OBTQ 批次库存表

```sql
CREATE TABLE IF NOT EXISTS ods_obtq (
    abs_entry BIGINT COMMENT '绝对条目号（主键）',
    item_code VARCHAR(50) COMMENT '物料编码',
    sys_number BIGINT COMMENT '系统编号',
    whs_code VARCHAR(50) COMMENT '仓库代码',
    quantity DECIMAL(18,6) COMMENT '库存数量',
    ct_version BIGINT COMMENT 'CT 版本号',
    ct_operation VARCHAR(10) COMMENT '操作类型',
    sync_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '同步时间'
)
UNIQUE KEY(abs_entry)
DISTRIBUTED BY HASH(abs_entry) BUCKETS 4
PROPERTIES (
    "replication_num" = "1",
    "enable_unique_key_merge_on_write" = "true"
);
```

> **说明**：`abs_entry` 在 `T_OBTQ` 中为主键，同一批次在不同仓库的记录通过 `abs_entry` 区分（SAP B1 中 `abs_entry` 已包含仓库维度）。

#### 5.3.5 U_NHOD 批次占用表

```sql
CREATE TABLE IF NOT EXISTS ods_nhod (
    code VARCHAR(50) COMMENT '单据编号',
    line_num INT COMMENT '行号',
    base_type INT COMMENT '基础类型',
    base_entry BIGINT COMMENT '基础单据号',
    base_line INT COMMENT '基础行号',
    create_date DATE COMMENT '创建日期',
    dist_number VARCHAR(50) COMMENT '批次号',
    doc_entry BIGINT COMMENT '单据号',
    is_del VARCHAR(1) COMMENT '删除标记',
    item_code VARCHAR(50) COMMENT '物料编码',
    owner_code VARCHAR(50) COMMENT '所有者代码',
    quantity DECIMAL(18,6) COMMENT '占用数量',
    sys_number BIGINT COMMENT '系统编号',
    trans_type INT COMMENT '交易类型',
    user_sign INT COMMENT '用户签名',
    whs_code VARCHAR(50) COMMENT '仓库代码',
    ct_version BIGINT COMMENT 'CT 版本号',
    ct_operation VARCHAR(10) COMMENT '操作类型',
    sync_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '同步时间'
)
UNIQUE KEY(code)
DISTRIBUTED BY HASH(code) BUCKETS 4
PROPERTIES (
    "replication_num" = "1",
    "enable_unique_key_merge_on_write" = "true"
);
```

### 5.4 DWS 层表设计

#### 5.4.1 交货汇总表（已对冲取消交货）

```sql
CREATE TABLE IF NOT EXISTS dws_delivery_summary (
    doc_entry BIGINT COMMENT '单据编号',
    item_code VARCHAR(50) COMMENT '物料编码',
    u_sub_card_code VARCHAR(50) COMMENT '子客户代码',
    card_code VARCHAR(50) COMMENT '客户代码',
    card_name VARCHAR(100) COMMENT '客户名称',
    doc_date DATE COMMENT '单据日期',
    u_order_num VARCHAR(50) COMMENT '订单号',
    u_end_dlv_date DATE COMMENT '最终交货日期',
    u_sub_card_name VARCHAR(100) COMMENT '子客户名称',
    u_frgn_name VARCHAR(200) COMMENT '外文名称',
    quantity DECIMAL(18,6) COMMENT '数量',
    u_quantity DECIMAL(18,6) COMMENT '自定义数量',
    line_total DECIMAL(18,6) COMMENT '行金额',
    is_canceled BOOLEAN COMMENT '是否取消',
    doc_status VARCHAR(1) COMMENT '单据状态'
)
UNIQUE KEY(doc_entry, item_code, u_sub_card_code)
DISTRIBUTED BY HASH(doc_entry) BUCKETS 4
PROPERTIES (
    "replication_num" = "1",
    "enable_unique_key_merge_on_write" = "true"
);
```

#### 5.4.2 实时库存表（库存数量 - 占用数量 = 可用数量）

```sql
CREATE TABLE IF NOT EXISTS dws_inventory_realtime (
    item_code VARCHAR(50) COMMENT '物料编码',
    sys_number BIGINT COMMENT '系统编号',
    whs_code VARCHAR(50) COMMENT '仓库代码',
    dist_number VARCHAR(50) COMMENT '批次号',
    stock_qty DECIMAL(18,6) COMMENT '库存数量（来自 T_OBTQ）',
    occupied_qty DECIMAL(18,6) COMMENT '占用数量（来自 U_NHOD，IsDel=N）',
    available_qty DECIMAL(18,6) COMMENT '可用数量 = stock - occupied',
    u_good_rate VARCHAR(50) COMMENT '良率',
    u_dc VARCHAR(50) COMMENT 'DC',
    u_box_num VARCHAR(50) COMMENT '箱号',
    u_maverick_flag VARCHAR(50) COMMENT 'Maverick标记',
    update_time DATETIME COMMENT '更新时间'
)
UNIQUE KEY(item_code, sys_number, whs_code)
DISTRIBUTED BY HASH(sys_number) BUCKETS 4
PROPERTIES (
    "replication_num" = "1",
    "enable_unique_key_merge_on_write" = "true"
);
```

### 5.5 同步辅助表

```sql
CREATE TABLE IF NOT EXISTS sync_checkpoint (
    table_name VARCHAR(100) COMMENT '表名',
    last_version BIGINT COMMENT '最后同步版本',
    last_sync_time DATETIME COMMENT '最后同步时间'
)
UNIQUE KEY(table_name)
DISTRIBUTED BY HASH(table_name) BUCKETS 1
PROPERTIES (
    "replication_num" = "1",
    "enable_unique_key_merge_on_write" = "true"
);
```

### 5.6 物化视图

> **注意**：复制节点数需小于实际可用 BE 节点数。若默认 3 但节点不足，执行：
> ```sql
> ADMIN SET FRONTEND CONFIG ('default_replication_num' = '1');
> ```

#### 5.6.1 月度交货汇总物化视图

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS mv_delivery_monthly
BUILD DEFERRED
REFRESH AUTO ON SCHEDULE EVERY 1 DAY
PROPERTIES ("replication_num" = "1")
AS
SELECT 
    DATE_FORMAT(doc_date, '%Y-%m') AS month_period,
    card_code,
    item_code,
    COUNT(DISTINCT doc_entry) AS doc_count,
    SUM(quantity) AS total_qty,
    SUM(u_quantity) AS total_u_qty,
    SUM(line_total) AS total_amount,
    SUM(CASE WHEN is_canceled THEN -line_total ELSE line_total END) AS net_amount
FROM dws_delivery_summary
GROUP BY month_period, card_code, item_code;
```

### 5.7 预聚合策略决策

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **单表 + 查询时聚合** ✅ | 维护简单，数据一致，存储省 | 查询稍慢 | 当前数据规模（< 10 亿） |
| 多表预聚合 | 查询极快 | 存储翻倍，同步复杂 | 数据量 > 10 亿 |
| 物化视图 | 自动维护，查询加速 | 占用存储 | 固定维度汇总 |

**本期决策**：
- 采用 **DWS 宽表 + 物化视图** 方案
- 不额外构建预聚合表，简化同步链路
- 若后续数据量增长或查询延迟不满足，再引入预聚合表

---

## 6. 前端架构设计

### 6.1 总体架构

ABL 前端采用 **React 18 + TypeScript** 技术栈，借鉴 Twenty CRM 的交互设计理念，融合灵动交互和 AI 支持，为低代码自然嵌入铺路。

```
┌─────────────────────────────────────────────────────────┐
│                    abl-ui (统一前端工程)                   │
│  ┌─────────────────┐ ┌─────────────────┐               │
│  │  abl-ui-public  │ │  abl-ui-admin   │               │
│  │   (用户端)       │ │   (管理端)       │               │
│  │  ├─BI 看板      │ │  ├─系统管理      │               │
│  │  ├─CRM 模块     │ │  ├─应用管理      │               │
│  │  ├─数据查询      │ │  ├─数仓管理      │               │
│  │  └─个人中心      │ │  └─监控告警      │               │
│  └─────────────────┘ └─────────────────┘               │
│  ┌─────────────────────────────────────────┐           │
│  │      abl-lowcode-frontend (低代码设计器)  │           │
│  │  ├─可视化表单设计器                        │           │
│  │  ├─流程设计器                             │           │
│  │  ├─页面设计器                             │           │
│  │  ├─数据模型设计器                          │           │
│  │  └─沙盒预览与发布                          │           │
│  └─────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────┘
```

> **合并说明**：原 `abl-lowcode-frontend` 独立仓库合并至 `abl-ui` 工程，作为独立子包（Package）管理，共享基础组件和工具库，但保持构建产物独立。

### 6.2 技术栈详解

| 层级 | 技术选型 | 用途 |
|------|----------|------|
| 框架 | React 18 + TypeScript | UI 基础 |
| 构建 | Vite 5.x | 快速构建与 HMR |
| 路由 | React Router 6.x | 前端路由管理 |
| 状态管理 | Zustand | 客户端全局状态（轻量） |
| 服务端状态 | React Query (TanStack Query) | 服务端数据获取、缓存、同步 |
| UI 组件库 | Ant Design 5.x | 基础组件 |
| 图表 | Ant Design Charts / ECharts | 数据可视化 |
| BI 嵌入 | Apache Superset Embedded SDK | 嵌入式 BI 看板 |
| 低代码引擎 | @formily/react + @designable | 表单与页面设计器 |
| 代码规范 | ESLint + Prettier + Husky | 代码质量 |
| 测试 | Vitest + React Testing Library | 单元测试 |

### 6.3 模块架构设计

#### 6.3.1 目录结构

```
abl-ui/
├── packages/
│   ├── @abl/core/              # 核心基础设施
│   │   ├── api/                # API 客户端封装（axios 实例、拦截器）
│   │   ├── auth/               # 认证逻辑（JWT、OAuth、单点登录）
│   │   ├── router/             # 路由配置与守卫
│   │   ├── store/              # Zustand Store 定义
│   │   ├── query/              # React Query 配置
│   │   └── utils/              # 通用工具函数
│   ├── @abl/components/        # 共享业务组件库
│   │   ├── DataTable/          # 通用数据表格（含分页、排序、筛选）
│   │   ├── FormBuilder/        # 动态表单渲染器
│   │   ├── ChartCard/          # 图表卡片容器
│   │   ├── AIAssist/           # AI 辅助输入组件
│   │   └── Permission/         # 权限控制组件
│   ├── @abl/designer/         # 低代码设计器（原 abl-lowcode-frontend）
│   │   ├── form-designer/      # 表单设计器
│   │   ├── flow-designer/      # 流程设计器
│   │   ├── page-designer/      # 页面设计器
│   │   └── preview/            # 沙盒预览
│   ├── abl-ui-public/          # 用户端应用
│   └── abl-ui-admin/           # 管理端应用
├── apps/
│   └── storybook/              # 组件文档与调试
└── shared/
    ├── types/                  # 全局 TypeScript 类型定义
    └── constants/              # 全局常量
```

#### 6.3.2 核心设计模式

**1. 依赖倒置（Dependency Inversion）**

高层组件不依赖低层实现，通过抽象接口注入：

```typescript
// 抽象接口
interface DataSource {
  fetchData<T>(url: string): Promise<T>;
}

// 实现
class ApiDataSource implements DataSource {
  async fetchData<T>(url: string): Promise<T> {
    return apiClient.get(url);
  }
}

// 组件依赖接口
function DataList({ dataSource }: { dataSource: DataSource }) {
  const { data } = useQuery(['list'], () => dataSource.fetchData('/api/list'));
  return <Table dataSource={data} />;
}
```

**2. 容器/展示组件分离**

```typescript
// 容器组件：处理数据获取
function OrderListContainer() {
  const { data, isLoading } = useOrders();
  const { mutate } = useSubmitOrder();
  return <OrderList data={data} loading={isLoading} onSubmit={mutate} />;
}

// 展示组件：纯 UI 渲染
function OrderList({ data, loading, onSubmit }: OrderListProps) {
  return <Table loading={loading} dataSource={data} onRowClick={onSubmit} />;
}
```

**3. 组合组件模式（Compound Components）**

用于复杂 UI 元素（如自定义下拉、折叠面板）：

```typescript
// 低代码设计器中的组件库选择器
<ComponentSelector>
  <ComponentSelector.Search />
  <ComponentSelector.CategoryList>
    <ComponentSelector.Category id="basic" label="基础组件" />
    <ComponentSelector.Category id="advanced" label="高级组件" />
  </ComponentSelector.CategoryList>
  <ComponentSelector.Preview />
</ComponentSelector>
```

### 6.4 状态管理策略

| 状态类型 | 管理方案 | 说明 |
|----------|----------|------|
| 服务端状态 | React Query | 缓存、去重、后台刷新、乐观更新 |
| 全局 UI 状态 | Zustand | 主题、侧边栏展开、全局消息 |
| 局部组件状态 | useState / useReducer | 表单输入、弹窗显隐 |
| 低代码设计器状态 | Zustand + 不可变更新 | 组件树、属性面板、历史记录 |

### 6.5 低代码设计器架构

```
┌─────────────────────────────────────────┐
│           低代码设计器运行时               │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ 画布区   │ │ 组件库   │ │ 属性面板 │   │
│  │ (Canvas)│ │ (Library)│ │ (Props)  │   │
│  └─────────┘ └─────────┘ └─────────┘   │
│  ┌─────────┐ ┌─────────┐             │
│  │ 大纲树   │ │ 工具栏   │             │
│  │ (Tree)  │ │ (Toolbar)│             │
│  └─────────┘ └─────────┘             │
└─────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│           设计器核心引擎                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ 组件注册  │ │ 拖拽编排  │ │ 属性绑定 │   │
│  │ 器       │ │ 引擎     │ │ 引擎    │   │
│  └─────────┘ └─────────┘ └─────────┘   │
│  ┌─────────┐ ┌─────────┐             │
│  │ 校验引擎  │ │ 代码生成  │             │
│  │         │ │ 器       │             │
│  └─────────┘ └─────────┘             │
└─────────────────────────────────────────┘
```

**关键特性**：
- **组件注册**：通过 JSON Schema 定义组件元数据，支持动态扩展
- **拖拽编排**：基于 @designable/core 实现，支持嵌套、复制、撤销
- **属性绑定**：支持静态值、变量绑定、表达式、数据源绑定
- **沙盒预览**：独立 iframe 环境，隔离设计器与运行时状态
- **代码生成**：将设计结果生成 React 组件代码或 JSON 配置

### 6.6 与 AI 的融合设计

```
┌─────────────────────────────────────────┐
│           AI 辅助交互层                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ AI 表单  │ │ AI 页面  │ │ AI 数据  │   │
│  │ 填充    │ │ 生成    │ │ 分析    │   │
│  └─────────┘ └─────────┘ └─────────┘   │
│  ┌─────────┐                           │
│  │ 智能提示  │                           │
│  │ (Copilot)│                           │
│  └─────────┘                           │
└─────────────────────────────────────────┘
```

**交互场景**：
- **AI 表单填充**：上传订单附件，AI 解析后自动填充表单字段
- **AI 页面生成**：自然语言描述页面需求，生成低代码页面配置
- **AI 数据分析**：在 BI 看板中，自然语言提问生成 SQL 和图表
- **智能提示（Copilot）**：在表单、代码编辑器中提供上下文感知的建议

### 6.7 性能优化策略

| 策略 | 实现方式 | 适用场景 |
|------|----------|----------|
| 代码分割 | React.lazy + Suspense | 路由级、组件级懒加载 |
| 虚拟滚动 | react-window | 大数据表格、列表 |
| 数据预取 | React Query prefetch | 鼠标悬停时预加载详情 |
| 防抖节流 | lodash debounce/throttle | 搜索输入、窗口调整 |
| 缓存策略 | React Query staleTime | 合理设置缓存有效期 |
| 构建优化 | Vite splitChunks | 公共库抽取、路由分割 |

---

## 7. 监控与运维

### 7.1 监控体系架构

ABL 监控体系聚焦**框架级数据**和**业务异常**，非传统 PGA 方式：

```
┌─────────────────────────────────────────┐
│           监控数据采集层                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │Micrometer│ │Prometheus│ │ 日志聚合 │   │
│  │指标采集  │ │ 时序库   │ │ (Loki) │   │
│  └─────────┘ └─────────┘ └─────────┘   │
└─────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│           监控数据展示层                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ Grafana │ │ 告警中心 │ │ 链路追踪 │   │
│  │ 可视化  │ │ (Alert) │ │ (Tempo) │   │
│  └─────────┘ └─────────┘ └─────────┘   │
└─────────────────────────────────────────┘
```

### 7.2 监控范围

| 层级 | 监控对象 | 指标示例 |
|------|----------|----------|
| 基础设施 | JVM、CPU、内存、磁盘 | 堆内存使用、GC 频率、线程数 |
| 服务层 | HTTP 接口、Feign 调用 | QPS、延迟、错误率、饱和度 |
| 数据层 | SQL 执行、连接池 | 慢查询、连接数、事务耗时 |
| 业务层 | 订单同步、AI 调用 | 同步成功率、解析准确率、Token 消耗 |
| 异常追踪 | 业务异常、系统错误 | 异常类型、堆栈、影响范围 |

### 7.3 告警策略

| 告警级别 | 触发条件 | 通知方式 |
|----------|----------|----------|
| P0（紧急） | 服务宕机、数据同步中断 | 电话 + 短信 + 企微 |
| P1（高） | 接口错误率 > 5%、延迟 > 2s | 企微 + 邮件 |
| P2（中） | 资源使用率 > 80%、慢查询增多 | 企微 |
| P3（低） | 非核心功能异常 | 邮件日报 |

### 7.4 健康检查

```yaml
# Spring Boot Actuator 配置
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
```

---

## 8. 附录

### 8.1 术语表

| 术语 | 说明 |
|------|------|
| CT | Change Tracking，SQL Server 变更追踪 |
| ODS | Operational Data Store，贴源数据层 |
| DWS | Data Warehouse Summary，汇总数据层 |
| ADS | Application Data Store，应用数据层 |
| MV | Materialized View，物化视图 |
| RBAC | Role-Based Access Control，基于角色的访问控制 |
| NER | Named Entity Recognition，命名实体识别 |
| OCR | Optical Character Recognition，光学字符识别 |
| MOQ | Minimum Order Quantity，最小订单量 |
| OPN | Outgoing Product Number，出货产品编号 |

### 8.2 环境配置说明

| 环境 | replication_num | CT Retention | 备注 |
|------|-------------------|--------------|------|
| 开发 | 1 | 2 天 | 单机部署 |
| 测试 | 1 | 2 天 | 与开发一致 |
| 生产 | 3 | 3-7 天 | 根据数据量动态调整 |

### 8.3 相关文档索引

- [ABL API 设计规范]()
- [ABL 数据库设计规范]()
- [ABL 前端开发规范]()
- [ABL 部署运维手册]()
- [SAP B1 接口对接文档]()

---

*文档结束*
