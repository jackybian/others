# ABL（艾博）平台 Sprint01 系统管理详细设计

> 版本：v1.0  
> 日期：2026-06-02  
> 状态：待评审

---

## 目录

1. [概述](#1-概述)
2. [用户管理](#2-用户管理)
3. [租户管理](#4-租户管理)
4. [应用管理](#5-应用管理)
5. [权限管理](#6-权限管理)
6. [认证管理](#7-认证管理)
7. [前端设计](#8-前端设计)
8. [附录](#9-附录)

---
> 注意数据库示例脚本可能为mysql格式，实际需要转换为PostgreSQL格式。
## 1. 概述

### 1.1 设计目标

系统管理模块是 ABL 平台的**基础设施核心**，负责管理平台的运行基础：用户、租户、应用、权限、认证。其设计遵循以下原则：

- **多租户隔离**：租户间数据完全隔离，资源独立配额
- **应用自治**：每个应用拥有独立的用户体系和权限空间
- **权限互通**：平台级权限与应用级权限无缝衔接，避免重复配置
- **统一认证**：支持多种认证方式，单点登录体验一致

### 1.2 核心概念

| 概念 | 定义 | 关系 |
|------|------|------|
| **平台 (Platform)** | ABL 系统本身 | 包含多个租户 |
| **租户 (Tenant)** | 企业/组织隔离单元 | 包含多个应用、多个用户 |
| **系统用户 (SysUser)** | 平台级用户账号 | 可跨租户（管理员）或单租户（普通用户） |
| **应用 (App)** | 业务系统单元 | 属于某个租户，包含应用内用户 |
| **应用用户 (AppUser)** | 应用级用户账号 | 属于某个应用，与系统用户可绑定 |
| **角色 (Role)** | 权限集合 | 系统角色 / 应用角色 |
| **权限 (Permission)** | 操作许可 | 菜单、按钮、API、数据范围 |
| **部门 (Dept)** | 组织架构节点 | 树形结构，支持多租户独立 |

### 1.3 权限体系全景

```
┌─────────────────────────────────────────────────────────────┐
│                     ABL 权限体系全景                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  平台级权限  │    │  应用级权限  │    │  数据级权限  │     │
│  │  (Platform) │◄──►│   (App)     │◄──►│   (Data)    │     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘     │
│         │                   │                   │           │
│         ▼                   ▼                   ▼           │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ 租户管理     │    │ 应用内功能   │    │ 行级过滤     │     │
│  │ 系统用户管理 │    │ 应用用户管理 │    │ 字段脱敏     │     │
│  │ 系统角色     │    │ 应用角色     │    │ 数据范围     │     │
│  │ 系统菜单     │    │ 应用菜单     │    │             │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              权限互通层 (Permission Bridge)            │   │
│  │   系统用户 ──绑定──► 应用用户                          │   │
│  │   系统角色 ──映射──► 应用角色                          │   │
│  │   平台权限 ──继承──► 应用权限                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 用户管理

### 2.1 用户模型设计

#### 2.1.1 系统用户表 (sys_user)

```sql
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY,
    tenant_id BIGINT NOT NULL COMMENT '所属租户ID（0表示平台级用户）',
    username VARCHAR(50) NOT NULL COMMENT '登录账号',
    password_hash VARCHAR(100) COMMENT '密码哈希（bcrypt）',
    real_name VARCHAR(50) COMMENT '真实姓名',
    avatar_url VARCHAR(200) COMMENT '头像URL',
    email VARCHAR(100) COMMENT '邮箱',
    phone VARCHAR(20) COMMENT '手机号',
    status TINYINT DEFAULT 1 COMMENT '状态：0-禁用 1-启用 2-锁定',
    user_type TINYINT DEFAULT 1 COMMENT '类型：1-普通用户 2-租户管理员 9-平台管理员',
    source VARCHAR(20) DEFAULT 'local' COMMENT '来源：local/ldap/oauth/wechat',
    source_id VARCHAR(100) COMMENT '第三方来源用户ID',
    last_login_time TIMESTAMP COMMENT '最后登录时间',
    last_login_ip VARCHAR(50) COMMENT '最后登录IP',
    pwd_error_count INT DEFAULT 0 COMMENT '密码错误次数',
    pwd_error_time TIMESTAMP COMMENT '密码错误时间',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted TINYINT DEFAULT 0 COMMENT '逻辑删除标记',
    UNIQUE KEY uk_tenant_username (tenant_id, username),
    INDEX idx_tenant (tenant_id),
    INDEX idx_status (status)
) ENGINE=InnoDB COMMENT='系统用户表';
```

#### 2.1.2 应用用户表 (app_user)

```sql
CREATE TABLE app_user (
    id BIGINT PRIMARY KEY,
    app_id BIGINT NOT NULL COMMENT '所属应用ID',
    tenant_id BIGINT NOT NULL COMMENT '所属租户ID（冗余，便于查询）',
    sys_user_id BIGINT COMMENT '绑定的系统用户ID（可为空，表示仅应用内用户）',
    username VARCHAR(50) NOT NULL COMMENT '应用内登录账号（可与系统用户不同）',
    password_hash VARCHAR(100) COMMENT '密码哈希（独立密码，可选）',
    display_name VARCHAR(50) COMMENT '显示名称',
    email VARCHAR(100) COMMENT '邮箱',
    phone VARCHAR(20) COMMENT '手机号',
    status TINYINT DEFAULT 1 COMMENT '状态：0-禁用 1-启用',
    user_type TINYINT DEFAULT 1 COMMENT '类型：1-普通用户 2-应用管理员',
    dept_id BIGINT COMMENT '所属部门ID',
    position VARCHAR(50) COMMENT '职位',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted TINYINT DEFAULT 0,
    UNIQUE KEY uk_app_username (app_id, username),
    INDEX idx_app (app_id),
    INDEX idx_sys_user (sys_user_id),
    INDEX idx_dept (dept_id)
) ENGINE=InnoDB COMMENT='应用用户表';
```

#### 2.1.3 用户绑定关系表 (user_binding)

```sql
CREATE TABLE user_binding (
    id BIGINT PRIMARY KEY,
    sys_user_id BIGINT NOT NULL COMMENT '系统用户ID',
    app_user_id BIGINT NOT NULL COMMENT '应用用户ID',
    app_id BIGINT NOT NULL COMMENT '应用ID',
    binding_type TINYINT DEFAULT 1 COMMENT '绑定类型：1-自动绑定 2-手动绑定 3-强制映射',
    status TINYINT DEFAULT 1 COMMENT '状态：0-解绑 1-绑定',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_sys_app (sys_user_id, app_id),
    UNIQUE KEY uk_app_user (app_user_id),
    INDEX idx_sys_user (sys_user_id),
    INDEX idx_app (app_id)
) ENGINE=InnoDB COMMENT='用户绑定关系表';
```

### 2.2 用户生命周期

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  创建   │───▶│  启用   │───▶│  锁定   │───▶│  禁用   │
│ (Create)│    │ (Active)│    │ (Locked)│    │(Disabled)│
└─────────┘    └────┬────┘    └────┬────┘    └────┬────┘
     │              │              │              │
     │              │              │              │
     ▼              ▼              ▼              ▼
  平台管理员      用户登录       密码错误        管理员操作
  或租户管理员    或管理员启用    超5次锁定       或自动策略
  创建账号                        30分钟自动解锁
```

### 2.3 用户来源与同步

| 来源 | 创建方式 | 密码管理 | 同步机制 |
|------|----------|----------|----------|
| **本地 (local)** | 管理员手动创建 | 平台管理 | 无 |
| **LDAP** | LDAP 首次登录自动创建 | LDAP 校验 | 定时同步组织架构 |
| **OAuth2** | 授权登录自动创建 | 无（OAuth2 托管） | 实时同步基础信息 |
| **企微** | 扫码授权自动创建 | 无（企微托管） | 实时同步部门信息 |

### 2.4 用户查询 API

```java
// 系统用户查询（平台管理员/租户管理员）
GET /api/v1/sys/users?tenantId=1&status=1&keyword=zhang&page=1&size=20

// 应用用户查询（应用管理员）
GET /api/v1/app/{appId}/users?deptId=5&status=1&keyword=li&page=1&size=20

// 用户绑定查询
GET /api/v1/users/{sysUserId}/bindings

// 用户详情（含权限）
GET /api/v1/users/{userId}/profile?include=roles,permissions,apps
```

---

## 3. 租户管理

### 3.1 租户模型设计

#### 3.1.1 租户表 (sys_tenant)

```sql
CREATE TABLE sys_tenant (
    id BIGINT PRIMARY KEY,
    tenant_code VARCHAR(50) NOT NULL COMMENT '租户编码（唯一，如：altos）',
    tenant_name VARCHAR(100) NOT NULL COMMENT '租户名称',
    tenant_type TINYINT DEFAULT 1 COMMENT '类型：1-企业 2-个人 3-试用',
    status TINYINT DEFAULT 1 COMMENT '状态：0-停用 1-启用 2-过期',
    logo_url VARCHAR(200) COMMENT 'Logo URL',
    contact_name VARCHAR(50) COMMENT '联系人姓名',
    contact_phone VARCHAR(20) COMMENT '联系人电话',
    contact_email VARCHAR(100) COMMENT '联系人邮箱',
    max_users INT DEFAULT 100 COMMENT '最大用户数',
    max_apps INT DEFAULT 10 COMMENT '最大应用数',
    storage_quota BIGINT DEFAULT 10737418240 COMMENT '存储配额（字节，默认10GB）',
    db_schema VARCHAR(50) COMMENT '独立数据库Schema（多Schema模式）',
    expire_time TIMESTAMP COMMENT '过期时间',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_code (tenant_code),
    INDEX idx_status (status)
) ENGINE=InnoDB COMMENT='租户表';
```

#### 3.1.2 租户配置表 (sys_tenant_config)

```sql
CREATE TABLE sys_tenant_config (
    id BIGINT PRIMARY KEY,
    tenant_id BIGINT NOT NULL COMMENT '租户ID',
    config_key VARCHAR(100) NOT NULL COMMENT '配置键',
    config_value TEXT COMMENT '配置值',
    config_type VARCHAR(20) DEFAULT 'string' COMMENT '类型：string/int/boolean/json',
    description VARCHAR(200) COMMENT '说明',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_tenant_key (tenant_id, config_key)
) ENGINE=InnoDB COMMENT='租户配置表';
```

### 3.2 租户隔离策略

| 隔离级别 | 实现方式 | 适用场景 |
|----------|----------|----------|
| **逻辑隔离** | 所有租户共享数据库，通过 tenant_id 字段区分 | 默认方案，成本最低 |
| **Schema 隔离** | 每个租户独立 Schema，共享数据库实例 | 数据安全要求较高 |
| **数据库隔离** | 每个租户独立数据库实例 | 大型企业客户 |

**本期采用**：逻辑隔离（共享数据库 + tenant_id）为主，Schema 隔离为可选扩展。

### 3.3 租户管理员机制

```sql
-- 租户管理员自动权限（不存储在角色表，由系统硬编码）
-- 租户管理员拥有租户内所有资源的 MANAGE 权限
-- 但不能管理平台级资源（如其他租户）
```

| 权限 | 平台管理员 | 租户管理员 | 普通用户 |
|------|------------|------------|----------|
| 创建租户 | ✅ | ❌ | ❌ |
| 管理租户配置 | ✅ | ✅（本租户） | ❌ |
| 创建应用 | ✅ | ✅ | ❌ |
| 管理应用 | ✅ | ✅（本租户） | ❌ |
| 管理用户 | ✅ | ✅（本租户） | ❌ |
| 查看数据 | ✅ | ✅（本租户） | 按权限 |

---

## 4. 应用管理

### 4.1 应用模型设计

#### 4.1.1 应用表 (sys_app)

```sql
CREATE TABLE sys_app (
    id BIGINT PRIMARY KEY,
    tenant_id BIGINT NOT NULL COMMENT '所属租户ID',
    app_code VARCHAR(50) NOT NULL COMMENT '应用编码',
    app_name VARCHAR(100) NOT NULL COMMENT '应用名称',
    app_type TINYINT NOT NULL COMMENT '类型：1-低代码 2-内置 3-外部',
    app_key VARCHAR(64) COMMENT '应用密钥（外部应用使用）',
    app_secret VARCHAR(128) COMMENT '应用密钥（外部应用使用）',
    icon_url VARCHAR(200) COMMENT '应用图标',
    description VARCHAR(500) COMMENT '应用描述',
    status TINYINT DEFAULT 0 COMMENT '状态：0-开发中 1-已发布 2-已下线',
    version VARCHAR(20) COMMENT '当前版本',
    home_page VARCHAR(200) COMMENT '首页路径',
    api_prefix VARCHAR(50) COMMENT 'API前缀（如：/crm）',
    metadata_id BIGINT COMMENT '关联的低代码元数据ID',
    external_url VARCHAR(200) COMMENT '外部应用URL（外部应用使用）',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted TINYINT DEFAULT 0,
    UNIQUE KEY uk_tenant_code (tenant_id, app_code),
    INDEX idx_tenant (tenant_id),
    INDEX idx_type (app_type),
    INDEX idx_status (status)
) ENGINE=InnoDB COMMENT='应用表';
```

#### 4.1.2 应用菜单表 (sys_app_menu)

```sql
CREATE TABLE sys_app_menu (
    id BIGINT PRIMARY KEY,
    app_id BIGINT NOT NULL COMMENT '所属应用ID',
    parent_id BIGINT DEFAULT 0 COMMENT '父菜单ID（0表示根节点）',
    menu_name VARCHAR(100) NOT NULL COMMENT '菜单名称',
    menu_code VARCHAR(100) NOT NULL COMMENT '菜单编码（权限标识）',
    menu_type TINYINT DEFAULT 1 COMMENT '类型：1-目录 2-菜单 3-按钮',
    icon VARCHAR(50) COMMENT '图标',
    path VARCHAR(200) COMMENT '路由路径',
    component VARCHAR(200) COMMENT '组件路径',
    permission VARCHAR(100) COMMENT '权限标识（如：crm:order:list）',
    sort_order INT DEFAULT 0 COMMENT '排序',
    status TINYINT DEFAULT 1 COMMENT '状态：0-禁用 1-启用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_app_code (app_id, menu_code),
    INDEX idx_app (app_id),
    INDEX idx_parent (parent_id)
) ENGINE=InnoDB COMMENT='应用菜单表';
```

### 4.2 应用生命周期

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  创建   │───▶│  开发   │───▶│  发布   │───▶│  下线   │
│(Create) │    │(Develop)│    │(Publish)│    │(Offline)│
└─────────┘    └────┬────┘    └────┬────┘    └────┬────┘
                    │              │              │
                    ▼              ▼              ▼
              低代码设计/编码    沙盒测试通过    管理员操作
              菜单配置            版本归档
```

### 4.3 应用与系统关系

```
┌─────────────────────────────────────────┐
│              租户 (Tenant)              │
│  ┌─────────────────────────────────┐   │
│  │           应用 (App)             │   │
│  │  ┌─────────┐ ┌─────────┐       │   │
│  │  │ 应用用户 │ │ 应用角色 │       │   │
│  │  │ (AppUser)│ │ (AppRole)│       │   │
│  │  └────┬────┘ └────┬────┘       │   │
│  │       │            │            │   │
│  │       └─────┬──────┘            │   │
│  │             │                   │   │
│  │       ┌─────┴─────┐             │   │
│  │       │ 应用权限   │             │   │
│  │       │ (AppPerm)  │             │   │
│  │       └───────────┘             │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │      系统用户 (SysUser)          │   │
│  │  ┌─────────┐ ┌─────────┐       │   │
│  │  │ 系统角色 │ │ 平台权限 │       │   │
│  │  │(SysRole)│ │(SysPerm)│       │   │
│  │  └─────────┘ └─────────┘       │   │
│  └─────────────────────────────────┘   │
│              │                           │
│              ▼                           │
│  ┌─────────────────────────────────┐   │
│  │      用户绑定 (UserBinding)      │   │
│  │   系统用户 ◄──────► 应用用户    │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

## 5. 权限管理

### 5.1 权限模型设计

#### 5.1.1 系统角色表 (sys_role)

```sql
CREATE TABLE sys_role (
    id BIGINT PRIMARY KEY,
    tenant_id BIGINT DEFAULT 0 COMMENT '租户ID（0表示平台级角色）',
    role_code VARCHAR(50) NOT NULL COMMENT '角色编码',
    role_name VARCHAR(100) NOT NULL COMMENT '角色名称',
    role_type TINYINT DEFAULT 1 COMMENT '类型：1-自定义 2-系统预设',
    data_scope TINYINT DEFAULT 1 COMMENT '数据范围：1-全部 2-本部门 3-本部门及下级 4-仅本人 5-自定义',
    custom_data_scope TEXT COMMENT '自定义数据范围（JSON）',
    description VARCHAR(200) COMMENT '说明',
    status TINYINT DEFAULT 1 COMMENT '状态：0-禁用 1-启用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted TINYINT DEFAULT 0,
    UNIQUE KEY uk_tenant_code (tenant_id, role_code),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB COMMENT='系统角色表';
```

#### 5.1.2 应用角色表 (app_role)

```sql
CREATE TABLE app_role (
    id BIGINT PRIMARY KEY,
    app_id BIGINT NOT NULL COMMENT '所属应用ID',
    tenant_id BIGINT NOT NULL COMMENT '租户ID（冗余）',
    role_code VARCHAR(50) NOT NULL COMMENT '角色编码',
    role_name VARCHAR(100) NOT NULL COMMENT '角色名称',
    role_type TINYINT DEFAULT 1 COMMENT '类型：1-自定义 2-系统预设',
    data_scope TINYINT DEFAULT 1 COMMENT '数据范围',
    custom_data_scope TEXT COMMENT '自定义数据范围',
    mapped_sys_role_id BIGINT COMMENT '映射的系统角色ID',
    description VARCHAR(200) COMMENT '说明',
    status TINYINT DEFAULT 1 COMMENT '状态',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted TINYINT DEFAULT 0,
    UNIQUE KEY uk_app_code (app_id, role_code),
    INDEX idx_app (app_id),
    INDEX idx_mapped_role (mapped_sys_role_id)
) ENGINE=InnoDB COMMENT='应用角色表';
```

#### 5.1.3 权限定义表 (sys_permission)

```sql
CREATE TABLE sys_permission (
    id BIGINT PRIMARY KEY,
    perm_code VARCHAR(100) NOT NULL COMMENT '权限编码（如：system:user:create）',
    perm_name VARCHAR(100) NOT NULL COMMENT '权限名称',
    perm_type TINYINT NOT NULL COMMENT '类型：1-菜单 2-按钮 3-API 4-数据',
    resource_type VARCHAR(20) COMMENT '资源类型（menu/api/dataset/dashboard）',
    resource_id VARCHAR(100) COMMENT '资源ID',
    http_method VARCHAR(10) COMMENT 'HTTP方法（API权限）',
    http_path VARCHAR(200) COMMENT 'API路径（支持AntPath）',
    parent_id BIGINT DEFAULT 0 COMMENT '父权限ID',
    sort_order INT DEFAULT 0,
    status TINYINT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_code (perm_code),
    INDEX idx_type (perm_type),
    INDEX idx_resource (resource_type, resource_id)
) ENGINE=InnoDB COMMENT='权限定义表';
```

#### 5.1.4 角色-权限关联表 (sys_role_permission)

```sql
CREATE TABLE sys_role_permission (
    id BIGINT PRIMARY KEY,
    role_id BIGINT NOT NULL COMMENT '角色ID',
    permission_id BIGINT NOT NULL COMMENT '权限ID',
    role_type TINYINT DEFAULT 1 COMMENT '角色类型：1-系统角色 2-应用角色',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_role_perm (role_id, permission_id, role_type),
    INDEX idx_role (role_id),
    INDEX idx_perm (permission_id)
) ENGINE=InnoDB COMMENT='角色权限关联表';
```

#### 5.1.5 用户-角色关联表 (sys_user_role)

```sql
CREATE TABLE sys_user_role (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL COMMENT '用户ID',
    role_id BIGINT NOT NULL COMMENT '角色ID',
    user_type TINYINT DEFAULT 1 COMMENT '用户类型：1-系统用户 2-应用用户',
    app_id BIGINT COMMENT '应用ID（应用用户时必填）',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_role (user_id, role_id, user_type, app_id),
    INDEX idx_user (user_id),
    INDEX idx_role (role_id)
) ENGINE=InnoDB COMMENT='用户角色关联表';
```

### 5.2 数据权限设计

#### 5.2.1 数据权限规则表 (sys_data_permission)

```sql
CREATE TABLE sys_data_permission (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL COMMENT '规则名称',
    rule_type TINYINT NOT NULL COMMENT '规则类型：1-行级 2-字段级',
    target_type VARCHAR(20) NOT NULL COMMENT '作用对象类型：role/user/dept',
    target_id BIGINT NOT NULL COMMENT '作用对象ID',
    entity_name VARCHAR(100) COMMENT '实体名称（如：Order, Customer）',
    entity_table VARCHAR(100) COMMENT '实体表名',

    -- 行级权限条件
    filter_condition TEXT COMMENT '过滤条件（SpEL表达式或SQL片段）',
    filter_sql TEXT COMMENT '预编译SQL条件',

    -- 字段级权限
    field_name VARCHAR(100) COMMENT '字段名',
    field_policy VARCHAR(20) COMMENT '策略：show/hide/mask/encrypt',
    mask_pattern VARCHAR(50) COMMENT '脱敏模式',

    priority INT DEFAULT 0 COMMENT '优先级',
    enabled TINYINT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_target (target_type, target_id),
    INDEX idx_entity (entity_name),
    INDEX idx_rule_type (rule_type)
) ENGINE=InnoDB COMMENT='数据权限规则表';
```

#### 5.2.2 数据范围枚举

| 数据范围 | 编码 | 说明 | 适用场景 |
|----------|------|------|----------|
| 全部数据 | ALL | 无限制 | 管理员 |
| 本部门数据 | DEPT | 用户所在部门 | 部门经理 |
| 本部门及下级 | DEPT_AND_CHILD | 部门树 | 区域总监 |
| 仅本人数据 | SELF | 创建者=当前用户 | 普通员工 |
| 自定义 | CUSTOM | 自定义SQL条件 | 特殊场景 |

### 5.3 权限互通机制

#### 5.3.1 互通架构

```
┌─────────────────────────────────────────────────────────────┐
│                     权限互通层 (Permission Bridge)            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐      ┌─────────────────┐              │
│  │   系统用户层      │      │   应用用户层      │              │
│  │                 │      │                 │              │
│  │  SysUser        │◄────►│  AppUser        │              │
│  │  ├─ 系统角色     │      │  ├─ 应用角色     │              │
│  │  ├─ 系统权限     │      │  ├─ 应用权限     │              │
│  │  └─ 数据权限     │      │  └─ 数据权限     │              │
│  │                 │      │                 │              │
│  └────────┬────────┘      └────────┬────────┘              │
│           │                        │                       │
│           │    ┌──────────────┐    │                       │
│           └───►│  权限映射引擎  │◄───┘                       │
│                │              │                            │
│                │ 1. 角色映射   │                            │
│                │ 2. 权限继承   │                            │
│                │ 3. 数据透传   │                            │
│                └──────────────┘                            │
│                         │                                 │
│                         ▼                                 │
│                ┌──────────────┐                            │
│                │  统一权限缓存  │                            │
│                │  (Redis)     │                            │
│                └──────────────┘                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 5.3.2 互通规则

**规则1：系统用户 → 应用用户自动绑定**

```java
// 当系统用户首次访问应用时，自动创建/绑定应用用户
public AppUser autoBindAppUser(Long sysUserId, Long appId) {
    // 1. 检查是否已绑定
    UserBinding binding = userBindingDao.findBySysUserAndApp(sysUserId, appId);
    if (binding != null) {
        return appUserDao.findById(binding.getAppUserId());
    }

    // 2. 获取系统用户信息
    SysUser sysUser = sysUserDao.findById(sysUserId);

    // 3. 创建应用用户（复制基础信息）
    AppUser appUser = new AppUser();
    appUser.setAppId(appId);
    appUser.setTenantId(sysUser.getTenantId());
    appUser.setSysUserId(sysUserId);
    appUser.setUsername(sysUser.getUsername());
    appUser.setDisplayName(sysUser.getRealName());
    appUser.setEmail(sysUser.getEmail());
    appUser.setPhone(sysUser.getPhone());
    appUser.setStatus(1);
    appUserDao.save(appUser);

    // 4. 创建绑定关系
    binding = new UserBinding();
    binding.setSysUserId(sysUserId);
    binding.setAppUserId(appUser.getId());
    binding.setAppId(appId);
    binding.setBindingType(1); // 自动绑定
    binding.setStatus(1);
    userBindingDao.save(binding);

    // 5. 继承系统角色到应用角色
    inheritRoles(sysUserId, appUser.getId(), appId);

    return appUser;
}
```

**规则2：系统角色 → 应用角色映射**

```sql
-- 角色映射配置表
CREATE TABLE role_mapping (
    id BIGINT PRIMARY KEY,
    sys_role_id BIGINT NOT NULL COMMENT '系统角色ID',
    app_role_id BIGINT NOT NULL COMMENT '应用角色ID',
    app_id BIGINT NOT NULL COMMENT '应用ID',
    mapping_type TINYINT DEFAULT 1 COMMENT '映射类型：1-自动继承 2-手动配置',
    status TINYINT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_sys_app_role (sys_role_id, app_id),
    INDEX idx_app_role (app_role_id)
) ENGINE=InnoDB COMMENT='角色映射表';
```

```java
// 角色继承逻辑
public void inheritRoles(Long sysUserId, Long appUserId, Long appId) {
    // 1. 获取系统用户的所有角色
    List<SysRole> sysRoles = sysUserRoleDao.findRolesByUser(sysUserId);

    for (SysRole sysRole : sysRoles) {
        // 2. 查找映射的应用角色
        RoleMapping mapping = roleMappingDao.findBySysRoleAndApp(sysRole.getId(), appId);

        if (mapping != null) {
            // 3. 赋予应用用户对应的应用角色
            AppRole appRole = appRoleDao.findById(mapping.getAppRoleId());
            appUserRoleDao.save(appUserId, appRole.getId(), appId);
        } else {
            // 4. 无映射时，创建默认应用角色（可选）
            AppRole defaultRole = createDefaultAppRole(sysRole, appId);
            appUserRoleDao.save(appUserId, defaultRole.getId(), appId);
        }
    }
}
```

**规则3：权限统一缓存**

```java
// 用户权限缓存结构（Redis Hash）
// Key: user:permission:{userId}:{userType}:{appId?}
// Field: permissions / roles / data_scope / row_filters

public UserPermissionContext getUserPermission(Long userId, Integer userType, Long appId) {
    String cacheKey = String.format("user:permission:%d:%d:%s", userId, userType, 
                                    appId != null ? appId : "0");

    // 1. 尝试从缓存获取
    UserPermissionContext context = redisTemplate.opsForValue().get(cacheKey);
    if (context != null) return context;

    // 2. 缓存未命中，实时计算
    context = new UserPermissionContext();

    if (userType == 1) { // 系统用户
        context.setRoles(sysUserRoleDao.findRolesByUser(userId));
        context.setPermissions(sysPermissionDao.findByUser(userId));
        context.setDataScopes(sysDataPermissionDao.findByUser(userId));
    } else { // 应用用户
        context.setRoles(appUserRoleDao.findRolesByUser(userId));
        context.setPermissions(sysPermissionDao.findByAppUser(userId, appId));
        context.setDataScopes(sysDataPermissionDao.findByAppUser(userId, appId));
    }

    // 3. 写入缓存（TTL 30分钟）
    redisTemplate.opsForValue().set(cacheKey, context, 30, TimeUnit.MINUTES);

    return context;
}
```

### 5.4 权限校验流程

```
用户请求 ──▶ API Gateway ──▶ JWT 解析 ──▶ 获取用户上下文
                              │
                              ▼
                    ┌─────────────────┐
                    │  1. 身份认证      │
                    │  校验Token有效性  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  2. 权限校验      │
                    │  检查用户是否拥有  │
                    │  该API的访问权限   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  3. 数据权限注入   │
                    │  根据数据范围规则  │
                    │  修改查询条件     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  4. 执行请求      │
                    │  返回过滤后的数据  │
                    └─────────────────┘
```

---

## 6. 认证管理

### 6.1 认证方式配置

#### 6.1.1 认证配置表 (sys_auth_config)

```sql
CREATE TABLE sys_auth_config (
    id BIGINT PRIMARY KEY,
    tenant_id BIGINT DEFAULT 0 COMMENT '租户ID（0表示平台级）',
    auth_type VARCHAR(20) NOT NULL COMMENT '认证类型：ldap/oauth2/wechat/local',
    auth_name VARCHAR(50) NOT NULL COMMENT '认证名称',
    config_json TEXT NOT NULL COMMENT '配置内容（JSON）',
    enabled TINYINT DEFAULT 1,
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_tenant_type (tenant_id, auth_type)
) ENGINE=InnoDB COMMENT='认证配置表';
```

### 6.2 LDAP 认证

```json
// LDAP 配置示例
{
  "ldap": {
    "server_url": "ldap://ldap.company.com:389",
    "base_dn": "dc=company,dc=com",
    "user_dn_pattern": "uid={0},ou=users,dc=company,dc=com",
    "group_search_base": "ou=groups,dc=company,dc=com",
    "group_filter": "(member={0})",
    "group_role_mapping": {
      "cn=admin,ou=groups,dc=company,dc=com": "platform_admin",
      "cn=users,ou=groups,dc=company,dc=com": "default_user"
    },
    "sync_interval": 3600,
    "sync_fields": ["cn", "mail", "telephoneNumber", "department"]
  }
}
```

### 6.3 OAuth2 / OIDC 认证

```json
// OAuth2 配置示例
{
  "oauth2": {
    "provider": "google",
    "client_id": "xxx.apps.googleusercontent.com",
    "client_secret": "xxx",
    "authorization_uri": "https://accounts.google.com/o/oauth2/v2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "user_info_uri": "https://openidconnect.googleapis.com/v1/userinfo",
    "scope": "openid profile email",
    "redirect_uri": "https://abl.company.com/auth/callback",
    "auto_create_user": true,
    "role_mapping": {
      "admin": "tenant_admin",
      "user": "default_user"
    }
  }
}
```

### 6.4 企微认证

```json
// 企微配置示例
{
  "wechat_work": {
    "corp_id": "wwxxxxxxxxxxxxxxxx",
    "agent_id": "1000002",
    "secret": "xxx",
    "redirect_uri": "https://abl.company.com/auth/wechat/callback",
    "sync_dept": true,
    "sync_user": true,
    "auto_create_user": true
  }
}
```

### 6.5 统一认证流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   用户访问   │────▶│  认证网关    │────▶│  认证策略器  │
│  登录页面   │     │ (Gateway)   │     │ (Strategy)  │
└─────────────┘     └──────┬──────┘     └──────┬──────┘
                           │                   │
                           │    ┌──────────────┘
                           │    │
                           ▼    ▼
                    ┌─────────────────┐
                    │  认证方式选择     │
                    │  1. 本地账号密码   │
                    │  2. LDAP         │
                    │  3. OAuth2       │
                    │  4. 企微扫码      │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  身份提供者      │
                    │  (IdP)          │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  用户创建/绑定   │
                    │  生成 JWT Token  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  权限加载        │
                    │  写入缓存        │
                    └─────────────────┘
```

---

## 7. 前端设计

### 7.1 系统管理前端架构

```
┌─────────────────────────────────────────────────────────┐
│              abl-ui-admin (系统管理前端)                   │
│  ┌─────────────────────────────────────────────────────┐│
│  │  布局框架 (Layout)                                    ││
│  │  ├─ 顶部导航：Logo + 租户切换 + 消息通知 + 用户头像   ││
│  │  ├─ 左侧菜单：动态菜单树（根据权限渲染）              ││
│  │  └─ 内容区：面包屑 + 页面内容 + 页签（可选）         ││
│  └─────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────┐│
│  │  页面模块                                             ││
│  │  ├─ 租户管理                                          ││
│  │  ├─ 用户管理                                          ││
│  │  ├─ 应用管理                                          ││
│  │  ├─ 角色权限                                          ││
│  │  ├─ 数据权限                                          ││
│  │  ├─ 菜单管理                                          ││
│  │  ├─ 部门管理                                          ││
│  │  ├─ 字典管理                                          ││
│  │  └─ 认证配置                                          ││
│  └─────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────┐│
│  │  共享组件                                             ││
│  │  ├─ DataTable：通用表格（分页/排序/筛选/导出）        ││
│  │  ├─ FormBuilder：动态表单（支持校验/联动）            ││
│  │  ├─ TreeSelector：树形选择器（部门/菜单）            ││
│  │  ├─ PermissionPicker：权限选择器                     ││
│  │  ├─ RoleTransfer：角色穿梭框                          ││
│  │  └─ DataScopePicker：数据范围选择器                   ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### 7.2 核心页面设计

#### 7.2.1 租户管理页面

```
┌─────────────────────────────────────────────────────────┐
│  租户管理                                    [+ 新建租户]│
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐ │
│  │ 筛选：状态 [全部 ▼] 类型 [全部 ▼] 搜索 [______]    │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ 租户名称    │ 编码   │ 类型   │ 状态   │ 用户数   │ │
│  │─────────────│────────│────────│────────│──────────│ │
│  │ 艾博科技    │ altos  │ 企业   │ ● 启用 │ 128/500 │ │
│  │ 测试租户    │ test   │ 试用   │ ● 启用 │ 5/10    │ │
│  │ ...         │ ...    │ ...    │ ...    │ ...     │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ 分页：◀ 1 2 3 ... 10 ▶    每页 20 条               │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘

// 租户详情抽屉
┌─────────────────────────────────────────┐
│ 租户详情                    [×]         │
├─────────────────────────────────────────┤
│ 基本信息                                │
│  租户名称：艾博科技                      │
│  租户编码：altos                         │
│  联系人：张三 / 138****8888              │
│  资源配额：                              │
│    ├─ 用户数：128/500                    │
│    ├─ 应用数：3/10                       │
│    └─ 存储：2.3GB/10GB                   │
│                                         │
│ 应用列表                                │
│  ├─ CRM系统 (内置)                       │
│  ├─ BI分析 (内置)                        │
│  └─ 自定义审批 (低代码)                   │
│                                         │
│ 操作：[编辑] [配置] [停用] [删除]        │
└─────────────────────────────────────────┘
```

#### 7.2.2 用户管理页面

```
┌─────────────────────────────────────────────────────────┐
│  用户管理                                    [+ 新建用户]│
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐ │
│  │ 筛选：租户 [艾博科技 ▼] 部门 [全部 ▼] 状态 [全部 ▼]  │ │
│  │ 搜索：[________________] [查询] [重置]              │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ [批量操作：启用 禁用 删除 分配角色]                 │ │
│  │                                                     │ │
│  │ □ │ 用户名 │ 姓名   │ 部门   │ 角色   │ 状态   │ 操作 │ │
│  │───│────────│────────│────────│────────│────────│──────│ │
│  │ □ │ zhang3 │ 张三   │ 销售部 │ 经理   │ ● 启用 │ [编辑]│ │
│  │ □ │ li4    │ 李四   │ 技术部 │ 开发   │ ● 启用 │ [编辑]│ │
│  │ □ │ wang5  │ 王五   │ 销售部 │ -      │ ○ 禁用 │ [编辑]│ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘

// 用户编辑弹窗
┌─────────────────────────────────────────┐
│ 编辑用户                                │
├─────────────────────────────────────────┤
│ 基本信息                                │
│  用户名：zhang3          [不可修改]      │
│  姓名：  [张三          ]                │
│  邮箱：  [zhang3@altos.com]              │
│  手机：  [138****8888  ]                │
│  部门：  [销售部      ▼]                │
│  状态：  ○ 禁用  ● 启用                 │
│                                         │
│ 角色分配                                │
│  ┌─────────────┐    ┌─────────────┐     │
│  │ 可选角色     │───▶│ 已选角色     │     │
│  │ ├─ 销售经理  │    │ ├─ 销售经理 ★ │     │
│  │ ├─ 普通销售  │    │ └─ 数据查看   │     │
│  │ ├─ 数据查看  │    │               │     │
│  │ └─ 系统管理  │    │               │     │
│  └─────────────┘    └─────────────┘     │
│                                         │
│ 数据范围：[本部门及下级 ▼]              │
│                                         │
│  [取消]              [保存]             │
└─────────────────────────────────────────┘
```

#### 7.2.3 角色权限管理页面

```
┌─────────────────────────────────────────────────────────┐
│  角色权限管理                              [+ 新建角色]  │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────────────────────┐│
│  │ 角色列表         │  │ 权限配置                         ││
│  │                 │  │                                 ││
│  │ ● 平台管理员     │  │ 角色名称：平台管理员              ││
│  │ ○ 租户管理员     │  │ 角色编码：platform_admin          ││
│  │ ○ 销售经理       │  │ 数据范围：全部数据               ││
│  │ ○ 普通销售       │  │                                 ││
│  │ ○ 数据查看       │  │ [菜单权限] [API权限] [数据权限]   ││
│  │ ○ ...           │  │                                 ││
│  │                 │  │ ┌─────────────────────────────┐ ││
│  │                 │  │ │ 系统管理                     │ ││
│  │                 │  │ │  □ 租户管理                  │ ││
│  │                 │  │ │    □ 查看  □ 创建  □ 编辑  □ 删除│ ││
│  │                 │  │ │  □ 用户管理                  │ ││
│  │                 │  │ │    □ 查看  □ 创建  □ 编辑  □ 删除│ ││
│  │                 │  │ │  □ 应用管理                  │ ││
│  │                 │  │ │    □ 查看  □ 创建  □ 编辑  □ 删除│ ││
│  │                 │  │ │                             │ ││
│  │                 │  │ │ CRM系统                      │ ││
│  │                 │  │ │  □ 订单管理                  │ ││
│  │                 │  │ │    □ 查看  □ 创建  □ 编辑  □ 删除│ ││
│  │                 │  │ │    □ 导入  □ 导出  □ 提交SAP  │ ││
│  │                 │  │ │                             │ ││
│  │                 │  │ │ BI分析                       │ ││
│  │                 │  │ │  □ 订单汇总看板              │ ││
│  │                 │  │ │    □ 查看  □ 编辑  □ 分享    │ ││
│  │                 │  │ └─────────────────────────────┘ ││
│  │                 │  │                                 ││
│  │                 │  │ [取消]              [保存]     ││
│  └─────────────────┘  └─────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

#### 7.2.4 数据权限配置页面

```
┌─────────────────────────────────────────────────────────┐
│  数据权限规则                              [+ 新建规则]  │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐ │
│  │ 规则名称 │ 作用对象  │ 实体    │ 类型   │ 状态   │  │
│  │──────────│───────────│─────────│────────│────────│  │
│  │ 销售隔离  │ 角色:销售 │ Order   │ 行级   │ ● 启用│  │
│  │ 成本隐藏  │ 角色:销售 │ Product │ 字段级 │ ● 启用│  │
│  │ ...      │ ...       │ ...     │ ...    │ ...   │  │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘

// 行级权限规则编辑
┌─────────────────────────────────────────┐
│ 编辑行级权限规则                        │
├─────────────────────────────────────────┤
│ 规则名称：[销售数据隔离                ]│
│                                         │
│ 作用对象：                              │
│  类型：● 角色  ○ 用户  ○ 部门          │
│  选择：[销售代表          ▼]            │
│                                         │
│ 作用实体：[Order (订单表)              ▼]│
│                                         │
│ 过滤条件：                              │
│  ┌─────────────────────────────────┐   │
│  │ 条件类型：[自定义SQL              ▼]│   │
│  │                                   │   │
│  │ SQL条件：                         │   │
│  │ [sales_rep_id = ${user.id}      ]│   │
│  │                                   │   │
│  │ 变量说明：                        │   │
│  │  ${user.id} - 当前用户ID         │   │
│  │  ${user.dept_id} - 当前部门ID    │   │
│  │  ${user.tenant_id} - 当前租户ID  │   │
│  └─────────────────────────────────┘   │
│                                         │
│ 优先级：[100                          ]│
│ 状态：  ● 启用  ○ 禁用                  │
│                                         │
│ [取消]                    [保存]       │
└─────────────────────────────────────────┘

// 字段级权限规则编辑
┌─────────────────────────────────────────┐
│ 编辑字段级权限规则                      │
├─────────────────────────────────────────┤
│ 规则名称：[成本价字段隐藏              ]│
│                                         │
│ 作用对象：角色 [外部用户               ▼]│
│ 作用实体：[Product (产品表)            ▼]│
│                                         │
│ 字段配置：                              │
│  ┌─────────────────────────────────┐   │
│  │ 字段名      │ 策略    │ 脱敏规则  │   │
│  │─────────────│─────────│──────────│   │
│  │ cost_price  │ [隐藏 ▼]│ -        │   │
│  │ supplier    │ [隐藏 ▼]│ -        │   │
│  │ phone       │ [脱敏 ▼]│ [手机 ▼] │   │
│  │ email       │ [脱敏 ▼]│ [邮箱 ▼] │   │
│  └─────────────────────────────────┘   │
│                                         │
│ 脱敏预览：                              │
│  原始：13812345678 → 脱敏：138****5678  │
│  原始：zhang@altos.com → 脱敏：z****@altos.com│
│                                         │
│ [取消]                    [保存]       │
└─────────────────────────────────────────┘
```

#### 7.2.5 应用管理页面

```
┌─────────────────────────────────────────────────────────┐
│  应用管理                                    [+ 新建应用]│
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐ │
│  │ 筛选：租户 [艾博科技 ▼] 类型 [全部 ▼] 状态 [全部 ▼]  │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ 应用名称    │ 类型   │ 版本   │ 状态   │ 用户数   │  │
│  │─────────────│────────│────────│────────│──────────│  │
│  │ CRM系统     │ 内置   │ v1.0   │ ● 发布 │ 45       │  │
│  │ BI分析      │ 内置   │ v1.0   │ ● 发布 │ 128      │  │
│  │ 自定义审批  │ 低代码 │ v0.9   │ ○ 开发 │ 0        │  │
│  │ 供应商门户  │ 外部   │ v2.1   │ ● 发布 │ 12       │  │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘

// 应用详情页面
┌─────────────────────────────────────────────────────────┐
│  CRM系统                                    [编辑] [配置]│
├─────────────────────────────────────────────────────────┤
│ 基本信息                                                │
│  应用ID：1001  类型：内置  状态：已发布                   │
│  版本：v1.0  创建时间：2026-05-01                        │
│                                                         │
│ 应用用户 (45人)                              [+ 添加用户]│
│  ┌─────────────────────────────────────────────────────┐│
│  │ 用户名 │ 姓名   │ 部门   │ 角色   │ 绑定状态 │ 操作  ││
│  │────────│────────│────────│────────│──────────│──────││
│  │ zhang3 │ 张三   │ 销售部 │ 经理   │ ● 已绑定 │ [解绑]││
│  │ li4    │ 李四   │ 技术部 │ 开发   │ ● 已绑定 │ [解绑]││
│  │ temp01 │ 临时工 │ -      │ 测试   │ ○ 未绑定 │ [绑定]││
│  └─────────────────────────────────────────────────────┘│
│                                                         │
│ 应用角色                                                │
│  ├─ 销售经理 (12人)                                      │
│  ├─ 普通销售 (28人)                                      │
│  ├─ 技术顾问 (3人)                                       │
│  └─ 访客 (2人)                                           │
│                                                         │
│ 应用菜单                                                │
│  ├─ 订单管理                                             │
│  │  ├─ 订单列表                                          │
│  │  ├─ 导入订单                                          │
│  │  └─ 提交SAP                                           │
│  ├─ 客户查询                                             │
│  └─ 报表中心                                             │
│                                                         │
│ 权限映射                                                │
│  ┌─────────────────────────────────────────────────────┐│
│  │ 系统角色      │ 映射应用角色    │ 映射方式           ││
│  │───────────────│─────────────────│────────────────────││
│  │ 平台管理员    │ 应用管理员      │ 自动继承           ││
│  │ 租户管理员    │ 销售经理        │ 手动配置           ││
│  │ 普通用户      │ 普通销售        │ 自动继承           ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### 7.3 权限选择器组件设计

```typescript
// PermissionPicker.tsx - 权限选择器组件
interface PermissionPickerProps {
  value: string[];           // 已选权限编码
  onChange: (perms: string[]) => void;
  resourceType?: 'menu' | 'api' | 'dataset' | 'dashboard';
  appId?: number;            // 应用ID（应用级权限时必填）
  showDataScope?: boolean;   // 是否显示数据范围配置
}

// 使用示例
<PermissionPicker
  value={selectedPermissions}
  onChange={setSelectedPermissions}
  resourceType="menu"
  appId={1001}
  showDataScope={true}
/>
```

```
// 权限选择器 UI
┌─────────────────────────────────────────┐
│ 权限选择                                │
├─────────────────────────────────────────┤
│ ┌─────────────────────────────────────┐ │
│ │ 搜索：[____________________] [搜索] │ │
│ │                                     │ │
│ │ □ 全部展开/收起                     │ │
│ │                                     │ │
│ │ □ 系统管理                          │ │
│ │   □ 租户管理                        │ │
│ │     □ 查看 ☑ 创建 ☑ 编辑 □ 删除    │ │
│ │   □ 用户管理                        │ │
│ │     ☑ 查看 ☑ 创建 □ 编辑 □ 删除    │ │
│ │                                     │ │
│ │ □ CRM系统                           │ │
│ │   □ 订单管理                        │ │
│ │     ☑ 查看 ☑ 创建 ☑ 编辑 ☑ 删除   │ │
│ │     ☑ 导入 □ 导出 ☑ 提交SAP        │ │
│ │   □ 客户查询                        │ │
│ │     ☑ 查看 □ 导出                   │ │
│ │                                     │ │
│ │ □ BI分析                            │ │
│ │   □ 订单汇总看板                    │ │
│ │     ☑ 查看 □ 编辑 □ 分享           │ │
│ │   □ 库存查询看板                    │ │
│ │     ☑ 查看 □ 编辑                   │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ 已选权限 (12)：查看 创建 编辑 删除 导入  │
│                 提交SAP 查看 查看 查看   │
│                                         │
│ [取消]                      [确定]      │
└─────────────────────────────────────────┘
```

### 7.4 数据范围选择器组件

```typescript
// DataScopePicker.tsx
interface DataScopePickerProps {
  value: DataScopeType;
  customScope?: string;
  onChange: (scope: DataScopeType, custom?: string) => void;
  deptTree: DeptNode[];      // 部门树数据
}

type DataScopeType = 'ALL' | 'DEPT' | 'DEPT_AND_CHILD' | 'SELF' | 'CUSTOM';
```

```
// 数据范围选择器 UI
┌─────────────────────────────────────────┐
│ 数据范围设置                            │
├─────────────────────────────────────────┤
│                                         │
│  ○ 全部数据（无限制）                    │
│                                         │
│  ● 本部门数据                          │
│    当前部门：销售部                      │
│                                         │
│  ○ 本部门及下级数据                    │
│    ┌─────────────────────────────────┐  │
│    │ 销售部                          │  │
│    │  ├─ 华东区销售                   │  │
│    │  │  ├─ 上海销售                   │  │
│    │  │  └─ 杭州销售                   │  │
│    │  └─ 华北区销售                   │  │
│    │     └─ 北京销售                  │  │
│    └─────────────────────────────────┘  │
│                                         │
│  ○ 仅本人数据                          │
│                                         │
│  ○ 自定义规则                          │
│    [部门 IN ('销售部', '市场部')       ]│
│    AND 创建日期 > 2026-01-01            │
│                                         │
│ [取消]                      [确定]      │
└─────────────────────────────────────────┘
```

---

## 8. 附录

### 8.1 权限编码规范

```
{模块}:{资源}:{操作}

示例：
  system:tenant:create    - 系统管理-租户-创建
  system:tenant:edit      - 系统管理-租户-编辑
  system:user:delete       - 系统管理-用户-删除
  crm:order:view         - CRM-订单-查看
  crm:order:submit_sap   - CRM-订单-提交SAP
  bi:dashboard:share     - BI-看板-分享
  bi:dataset:query       - BI-数据集-查询
```

### 8.2 数据库索引清单

| 表名 | 索引名 | 字段 | 类型 | 说明 |
|------|--------|------|------|------|
| sys_user | uk_tenant_username | tenant_id, username | UNIQUE | 租户内用户名唯一 |
| sys_user | idx_tenant | tenant_id | INDEX | 租户查询 |
| app_user | uk_app_username | app_id, username | UNIQUE | 应用内用户名唯一 |
| app_user | idx_sys_user | sys_user_id | INDEX | 绑定查询 |
| sys_role | uk_tenant_code | tenant_id, role_code | UNIQUE | 租户内角色编码唯一 |
| app_role | uk_app_code | app_id, role_code | UNIQUE | 应用内角色编码唯一 |
| user_binding | uk_sys_app | sys_user_id, app_id | UNIQUE | 用户应用绑定唯一 |
| sys_permission | uk_code | perm_code | UNIQUE | 权限编码全局唯一 |
| sys_role_permission | uk_role_perm | role_id, permission_id, role_type | UNIQUE | 角色权限唯一 |
| sys_user_role | uk_user_role | user_id, role_id, user_type, app_id | UNIQUE | 用户角色唯一 |

### 8.3 API 清单

#### 用户管理 API

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/v1/sys/users | 查询系统用户 | system:user:view |
| POST | /api/v1/sys/users | 创建系统用户 | system:user:create |
| PUT | /api/v1/sys/users/{id} | 更新系统用户 | system:user:edit |
| DELETE | /api/v1/sys/users/{id} | 删除系统用户 | system:user:delete |
| GET | /api/v1/app/{appId}/users | 查询应用用户 | app:user:view |
| POST | /api/v1/app/{appId}/users | 创建应用用户 | app:user:create |
| POST | /api/v1/users/{id}/bind | 绑定应用用户 | app:user:bind |
| POST | /api/v1/users/{id}/unbind | 解绑应用用户 | app:user:unbind |

#### 角色权限 API

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/v1/sys/roles | 查询系统角色 | system:role:view |
| POST | /api/v1/sys/roles | 创建系统角色 | system:role:create |
| PUT | /api/v1/sys/roles/{id}/permissions | 配置角色权限 | system:role:edit |
| GET | /api/v1/app/{appId}/roles | 查询应用角色 | app:role:view |
| POST | /api/v1/app/{appId}/roles | 创建应用角色 | app:role:create |
| POST | /api/v1/roles/mapping | 配置角色映射 | system:role:edit |
| GET | /api/v1/users/{id}/permissions | 查询用户权限 | -（自身或管理员） |

#### 数据权限 API

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/v1/data-permissions | 查询数据权限规则 | system:data:view |
| POST | /api/v1/data-permissions | 创建数据权限规则 | system:data:create |
| PUT | /api/v1/data-permissions/{id} | 更新数据权限规则 | system:data:edit |
| DELETE | /api/v1/data-permissions/{id} | 删除数据权限规则 | system:data:delete |
| POST | /api/v1/data-permissions/validate | 验证规则SQL | system:data:edit |

### 8.4 时序图

#### 用户登录权限加载

```
用户        Gateway      AuthService    UserService    PermissionService    Redis
 │            │              │              │                  │                │
 │──登录────▶│              │              │                  │                │
 │            │────────────▶│              │                  │                │
 │            │              │──验证密码───▶│                  │                │
 │            │              │◀──成功──────│                  │                │
 │            │              │              │                  │                │
 │            │              │──加载用户───▶│                  │                │
 │            │              │◀──用户信息──│                  │                │
 │            │              │              │                  │                │
 │            │              │──────────────│─────────────────▶│                │
 │            │              │              │                  │──加载权限──────▶│
 │            │              │              │                  │◀──缓存命中─────│
 │            │              │◀────────────│──────────────────│                │
 │            │◀─────────────│              │                  │                │
 │◀──Token───│              │              │                  │                │
 │            │              │              │                  │                │
 │──访问API──▶│              │              │                  │                │
 │            │──解析Token──│              │                  │                │
 │            │              │              │                  │                │
 │            │──权限校验────│              │                  │                │
 │            │              │              │                  │                │
 │            │◀──通过──────│              │                  │                │
 │◀──数据────│              │              │                  │                │
 │            │              │              │                  │                │
```

---

*文档结束*
