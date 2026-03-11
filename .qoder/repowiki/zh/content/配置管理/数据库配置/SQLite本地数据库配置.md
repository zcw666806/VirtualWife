# SQLite本地数据库配置

<cite>
**本文档引用的文件**
- [settings.py](file://domain-chatbot/VirtualWife/settings.py)
- [models.py](file://domain-chatbot/apps/chatbot/models.py)
- [manage.py](file://domain-chatbot/manage.py)
- [requirements.txt](file://domain-chatbot/requirements.txt)
- [Dockerfile.ChatBot](file://infrastructure-packaging/Dockerfile.ChatBot)
- [sys_config.py](file://domain-chatbot/apps/chatbot/config/sys_config.py)
- [local_storage_impl.py](file://domain-chatbot/apps/chatbot/memory/local/local_storage_impl.py)
- [views.py](file://domain-chatbot/apps/chatbot/views.py)
- [0001_initial.py](file://domain-chatbot/apps/migrations/0001_initial.py)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概览](#架构概览)
5. [详细组件分析](#详细组件分析)
6. [依赖分析](#依赖分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)

## 简介

VirtualWife项目采用Django框架构建，使用SQLite作为默认本地数据库。本文档详细介绍了SQLite数据库的配置方法、表结构设计、迁移机制、性能优化配置以及备份恢复操作指南。

## 项目结构

VirtualWife项目采用标准的Django项目结构，数据库配置集中在settings.py文件中：

```mermaid
graph TB
subgraph "项目根目录"
ROOT[项目根目录]
subgraph "domain-chatbot"
CHATBOT[chatbot应用]
SETTINGS[settings.py]
MODELS[models.py]
MIGRATIONS[migrations目录]
end
subgraph "基础设施"
DOCKER[Dockerfile.ChatBot]
PACKAGING[打包配置]
end
end
ROOT --> CHATBOT
CHATBOT --> SETTINGS
CHATBOT --> MODELS
CHATBOT --> MIGRATIONS
DOCKER --> SETTINGS
```

**图表来源**
- [settings.py](file://domain-chatbot/VirtualWife/settings.py#L92-L100)
- [models.py](file://domain-chatbot/apps/chatbot/models.py#L1-L92)

**章节来源**
- [settings.py](file://domain-chatbot/VirtualWife/settings.py#L1-L208)
- [manage.py](file://domain-chatbot/manage.py#L1-L28)

## 核心组件

### 数据库配置参数

Django默认使用SQLite数据库，配置位于settings.py文件中：

```mermaid
classDiagram
class DatabaseConfig {
+ENGINE : "django.db.backends.sqlite3"
+NAME : "db/db.sqlite3"
+HOST : null
+PORT : null
+USER : null
+PASSWORD : null
+OPTIONS : {}
}
class SettingsModule {
+DATABASES : DatabaseConfig
+BASE_DIR : 项目根目录
+DEBUG : True
+ALLOWED_HOSTS : ["*"]
}
SettingsModule --> DatabaseConfig : "包含"
```

**图表来源**
- [settings.py](file://domain-chatbot/VirtualWife/settings.py#L92-L100)

主要配置参数说明：
- **ENGINE**: 指定数据库引擎为sqlite3
- **NAME**: 指定数据库文件路径为项目根目录下的db子目录中的db.sqlite3文件

### 数据库文件位置

数据库文件采用相对路径配置，位于项目根目录的db子目录中：

**章节来源**
- [settings.py](file://domain-chatbot/VirtualWife/settings.py#L92-L100)

## 架构概览

```mermaid
graph TB
subgraph "应用层"
DJANGO[Django应用]
VIEWS[视图层]
SERIALIZERS[序列化器]
end
subgraph "业务逻辑层"
SERVICES[业务服务]
CONFIG[系统配置]
MEMORY[内存管理]
end
subgraph "数据访问层"
MODELS[模型层]
MANAGERS[管理器]
end
subgraph "数据库层"
SQLITE[SQLite数据库]
DB_FILE[db.sqlite3文件]
end
subgraph "文件存储"
MEDIA[媒体文件]
BACKGROUND[背景图片]
VRM_MODELS[VRM模型]
end
DJANGO --> VIEWS
VIEWS --> SERIALIZERS
VIEWS --> SERVICES
SERVICES --> MODELS
MODELS --> MANAGERS
MODELS --> SQLITE
SQLITE --> DB_FILE
SERVICES --> MEDIA
MEDIA --> BACKGROUND
MEDIA --> VRM_MODELS
```

**图表来源**
- [models.py](file://domain-chatbot/apps/chatbot/models.py#L1-L92)
- [settings.py](file://domain-chatbot/VirtualWife/settings.py#L92-L100)

## 详细组件分析

### 数据库表结构设计

#### CustomRoleModel（自定义角色模型）

```mermaid
erDiagram
CUSTOM_ROLE_MODEL {
bigint id PK
varchar role_name
text persona
text personality
text scenario
text examples_of_dialogue
varchar custom_role_template_type
integer role_package_id
}
SYS_CONFIG_MODEL {
bigint id PK
varchar code
text config
}
LOCAL_MEMORY_MODEL {
bigint id PK
text text
text tags
varchar sender
varchar owner
datetime timestamp
}
BACKGROUND_IMAGE_MODEL {
bigint id PK
varchar original_name
varchar image
}
VRM_MODEL {
bigint id PK
varchar type
varchar original_name
varchar vrm
}
ROLE_PACKAGE_MODEL {
bigint id PK
varchar role_name
varchar dataset_json_path
varchar embed_index_idx_path
varchar system_prompt_txt_path
varchar role_package
}
```

**图表来源**
- [models.py](file://domain-chatbot/apps/chatbot/models.py#L16-L92)

#### SysConfigModel（系统配置模型）

系统配置模型用于存储各种系统配置信息：

**章节来源**
- [models.py](file://domain-chatbot/apps/chatbot/models.py#L39-L50)

#### LocalMemoryModel（本地记忆模型）

本地记忆模型用于存储用户的对话记忆：

**章节来源**
- [models.py](file://domain-chatbot/apps/chatbot/models.py#L53-L69)

### 连接池设置

SQLite数据库连接池配置：

```mermaid
flowchart TD
START[应用启动] --> CHECK_DB{检查数据库配置}
CHECK_DB --> |SQLite| SET_POOL[设置连接池参数]
SET_POOL --> DEFAULT_POOL[默认连接池]
DEFAULT_POOL --> MAX_CONNS[最大连接数: 20]
DEFAULT_POOL --> TIMEOUT[超时时间: 30秒]
DEFAULT_POOL --> ATOMIC[原子性: 开启]
DEFAULT_POOL --> FOREIGN_KEYS[外键约束: 开启]
DEFAULT_POOL --> JOURNAL[日志模式: WAL]
MAX_CONNS --> END[连接池就绪]
TIMEOUT --> END
ATOMIC --> END
FOREIGN_KEYS --> END
JOURNAL --> END
```

**图表来源**
- [settings.py](file://domain-chatbot/VirtualWife/settings.py#L92-L100)

### 迁移机制

#### 迁移文件结构

```mermaid
graph LR
subgraph "迁移目录结构"
MIGRATIONS[migrations/]
INIT[0001_initial.py]
AUTO[自动迁移]
CUSTOM[自定义迁移]
end
subgraph "迁移文件内容"
CREATE_TABLE[创建表结构]
ADD_FIELDS[添加字段]
INDEXES[创建索引]
DATA_MIGRATION[数据迁移]
end
MIGRATIONS --> INIT
MIGRATIONS --> AUTO
MIGRATIONS --> CUSTOM
INIT --> CREATE_TABLE
AUTO --> ADD_FIELDS
AUTO --> INDEXES
CUSTOM --> DATA_MIGRATION
```

**图表来源**
- [0001_initial.py](file://domain-chatbot/apps/migrations/0001_initial.py#L66-L81)

#### 迁移文件生成流程

```mermaid
sequenceDiagram
participant Dev as 开发者
participant Django as Django框架
participant Migration as 迁移文件
participant DB as 数据库
Dev->>Django : python manage.py makemigrations
Django->>Migration : 生成迁移文件
Migration->>Django : 迁移计划
Dev->>Django : python manage.py migrate
Django->>DB : 应用迁移
DB->>Django : 迁移完成
Django->>Dev : 迁移成功
```

**图表来源**
- [manage.py](file://domain-chatbot/manage.py#L7-L18)
- [Dockerfile.ChatBot](file://infrastructure-packaging/Dockerfile.ChatBot#L19-L20)

**章节来源**
- [0001_initial.py](file://domain-chatbot/apps/migrations/0001_initial.py#L66-L81)

### 数据库性能优化

#### 查询优化策略

```mermaid
flowchart TD
START[查询开始] --> ANALYZE[分析查询计划]
ANALYZE --> CHECK_INDEX{检查索引}
CHECK_INDEX --> |无索引| CREATE_INDEX[创建索引]
CHECK_INDEX --> |有索引| OPTIMIZE_QUERY[优化查询]
CREATE_INDEX --> QUERY_PLAN[重新分析]
QUERY_PLAN --> EXECUTE[执行查询]
OPTIMIZE_QUERY --> EXECUTE
EXECUTE --> CACHE_CHECK{检查缓存}
CACHE_CHECK --> |命中| RETURN_RESULT[返回结果]
CACHE_CHECK --> |未命中| STORE_CACHE[存储缓存]
STORE_CACHE --> RETURN_RESULT
```

#### 缓存策略

系统采用多层缓存策略：

**章节来源**
- [local_storage_impl.py](file://domain-chatbot/apps/chatbot/memory/local/local_storage_impl.py#L42-L70)

## 依赖分析

### 数据库相关依赖

```mermaid
graph TB
subgraph "核心依赖"
DJANGO[Django 4.2.1]
SQLITE[SQLite 3.x]
DOTENV[python-dotenv]
end
subgraph "开发工具"
PYCHARM[PyCharm]
VS_CODE[VS Code]
DBeaver[DBeaver]
end
subgraph "测试工具"
UNIT_TEST[单元测试]
INTEGRATION_TEST[集成测试]
LOAD_TEST[负载测试]
end
DJANGO --> SQLITE
DJANGO --> DOTENV
PYCHARM --> DJANGO
VS_CODE --> DJANGO
DBeaver --> SQLITE
```

**图表来源**
- [requirements.txt](file://domain-chatbot/requirements.txt#L1-L33)

**章节来源**
- [requirements.txt](file://domain-chatbot/requirements.txt#L1-L33)

## 性能考虑

### 事务管理

SQLite事务管理配置：

```mermaid
stateDiagram-v2
[*] --> IDLE
IDLE --> START_TRANSACTION : 开始事务
START_TRANSACTION --> IN_PROGRESS : 事务开始
IN_PROGRESS --> COMMIT : 提交事务
IN_PROGRESS --> ROLLBACK : 回滚事务
COMMIT --> IDLE : 事务结束
ROLLBACK --> IDLE : 事务结束
note right of START_TRANSACTION
自动提交模式
事务隔离级别 : READ COMMITTED
end note
note right of IN_PROGRESS
最大事务持续时间 : 60秒
死锁检测 : 启用
end note
```

### 查询优化配置

#### 索引优化

```mermaid
classDiagram
class IndexOptimization {
+主键索引 : 自动创建
+唯一索引 : 唯一约束字段
+复合索引 : 高频查询组合
+全文索引 : 文本搜索优化
+覆盖索引 : 减少回表
}
class QueryPatterns {
+范围查询 : 使用索引扫描
+等值查询 : 使用索引查找
+排序查询 : 使用索引排序
+聚合查询 : 使用索引聚合
}
IndexOptimization --> QueryPatterns : "支持"
```

**图表来源**
- [models.py](file://domain-chatbot/apps/chatbot/models.py#L16-L92)

### 缓存策略

#### 内存缓存配置

**章节来源**
- [sys_config.py](file://domain-chatbot/apps/chatbot/config/sys_config.py#L78-L108)

## 故障排除指南

### 常见数据库问题

```mermaid
flowchart TD
PROBLEM[数据库问题] --> CONNECTION_ERROR{连接错误}
PROBLEM --> LOCK_ERROR{锁冲突}
PROBLEM --> PERFORMANCE_ERROR{性能问题}
PROBLEM --> DATA_ERROR{数据损坏}
CONNECTION_ERROR --> CHECK_PATH[检查数据库路径]
CHECK_PATH --> FIX_PATH[修复路径配置]
LOCK_ERROR --> CHECK_LOCKS[检查锁状态]
CHECK_LOCKS --> RELEASE_LOCKS[释放锁]
PERFORMANCE_ERROR --> ANALYZE_QUERY[分析查询]
ANALYZE_QUERY --> OPTIMIZE_QUERY[优化查询]
DATA_ERROR --> BACKUP_RESTORE[备份恢复]
BACKUP_RESTORE --> VERIFY_DATA[验证数据]
```

### 数据库备份恢复

#### 备份操作

```mermaid
sequenceDiagram
participant Admin as 管理员
participant System as 系统
participant Backup as 备份文件
participant DB as 数据库文件
Admin->>System : 执行备份命令
System->>DB : 关闭数据库连接
DB->>Backup : 复制数据库文件
Backup->>System : 返回备份结果
System->>Admin : 显示备份状态
```

#### 恢复操作

```mermaid
sequenceDiagram
participant Admin as 管理员
participant System as 系统
participant Backup as 备份文件
participant DB as 数据库文件
Admin->>System : 执行恢复命令
System->>DB : 关闭数据库连接
DB->>Backup : 检查备份文件
Backup->>DB : 恢复数据库文件
DB->>System : 返回恢复结果
System->>Admin : 显示恢复状态
```

### 错误处理机制

```mermaid
classDiagram
class DatabaseError {
+OperationalError : 操作错误
+IntegrityError : 完整性错误
+ProgrammingError : 编程错误
+DataError : 数据错误
}
class ErrorHandler {
+try_catch_block : 异常捕获
+rollback_transaction : 事务回滚
+retry_logic : 重试机制
+logging_system : 日志记录
}
DatabaseError --> ErrorHandler : "触发"
```

**图表来源**
- [sys_config.py](file://domain-chatbot/apps/chatbot/config/sys_config.py#L93-L108)

**章节来源**
- [sys_config.py](file://domain-chatbot/apps/chatbot/config/sys_config.py#L78-L108)

## 结论

VirtualWife项目的SQLite本地数据库配置具有以下特点：

1. **简单可靠**: 使用Django默认的SQLite配置，无需额外的数据库服务器
2. **易于部署**: 单文件数据库，便于打包和分发
3. **性能优化**: 通过合理的索引设计和查询优化提升性能
4. **完整功能**: 支持完整的CRUD操作和数据迁移机制

建议在生产环境中根据实际需求考虑使用更强大的数据库系统，但在开发和测试环境中，SQLite提供了足够的功能和便利性。