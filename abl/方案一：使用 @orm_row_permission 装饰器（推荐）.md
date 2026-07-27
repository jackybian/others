在针对 SQLAlchemy ORM 的场景下，应用数据权限的方式略有不同，但我们可以保持一致的“装饰器”或“工具函数”风格。

我已经更新了 abl-python-common SDK，增加了对 ORM 的原生支持。如果 pg_sys_user_repository.py 需要应用权限，可以采用以下两种方式：

### 方案一：使用 @orm_row_permission 装饰器（推荐）
为了保持您偏好的装饰器风格，我们可以将 Select 语句的构建逻辑提取出来，并挂上装饰器。

```
from abl_common.core.db import orm_row_permission

# 1. 定义一个返回 Select 对象的内部函数，并挂上装饰器
@orm_row_permission(table_name="sys_user")
def _get_user_stmt(user_id: int):
    return select(SysUser.email).where(SysUser.id == user_id, SysUser.deleted == 0)

# 2. 业务函数调用该语句
def get_user_email(user_id: int) -> str | None:
    with session_scope() as session:
        # 装饰器会自动在 stmt 中注入 WHERE (dept_id IN ...) 等权限 SQL
        stmt = _get_user_stmt(user_id)
        return session.execute(stmt).scalar_one_or_none()
```
### 方案二：使用 apply_orm_permission 工具函数（更灵活）
如果您不想拆分函数，可以直接在代码流中调用我新写的工具函数：

```
from abl_common.core.db import apply_orm_permission

def get_user_email(user_id: int) -> str | None:
    with session_scope() as session:
        stmt = select(SysUser.email).where(SysUser.id == user_id, SysUser.deleted 
        == 0)
        
        # 手动应用权限注入
        stmt = apply_orm_permission(stmt, table_name="sys_user")
        
        return session.execute(stmt).scalar_one_or_none()
```
### 底层原理：SQLAlchemy text() 注入
我在 SDK 的 db.py 中利用了 SQLAlchemy 的 text() 方法。它会将权限中心返回的原始 SQL（例如 dept_id IN (1,2,3) ）无缝注入到 ORM 的 where 子句中，如下所示：

```
# SDK 内部实现片段
from sqlalchemy import text
stmt = stmt.where(text(perm.filterSql)) 
```
### 总结
虽然 ORM 和 Doris 的原生 SQL 语法不同，但通过我封装的 apply_orm_permission ，我们可以实现：

- 逻辑统一 ：无论是 Doris 还是 PostgreSQL，都共享同一套权限规则。
- 自动化 ：依然支持 ${emails} 等动态占位符的自动替换。
- 安全 ：利用 ORM 的构建机制，确保权限 SQL 被正确地包裹在 () 中，避免逻辑冲突。