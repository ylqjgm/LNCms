# TASK003: 系统配置和模块管理

## 任务信息

**任务编号**: TASK003  
**任务名称**: 系统配置和模块管理  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**创建时间**: 2026-01-02 21:30:48 CST

## 任务描述

实现系统配置管理功能，包括网站设置、注册设置、站群设置、积分设置、附件设置、邮箱设置、屏蔽设置等配置项的管理。实现模块管理功能，包括模块部署、模块启用/禁用、模块配置等。

这是核心模块的基础功能任务，为后续权限管理、用户管理等任务提供配置支撑。所有功能必须遵循项目开发规范，包括API设计规范、Golang编码规范、数据库设计规范等。

## 技术栈

- **开发语言**: Golang 1.24+
- **开发框架**: Gin
- **数据库**: PostgreSQL
- **缓存**: Redis
- **API 规范**: OpenAPI 3.1.1
- **响应格式**: Envelope 格式

## 依赖任务

- **TASK001**: 项目初始化和基础框架搭建（HTTP服务器、gRPC服务器、Redis连接）
- **TASK002**: 数据库设计和初始化（system_configs表、modules表）

## 子任务列表

### 子任务 1: 系统配置数据库模型和仓储层实现

**任务编号**: TASK003-01  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 创建系统配置的GORM模型和仓储层实现。包括：1) 创建SystemConfig模型，定义配置表结构（id、config_type、config_data、created_at、updated_at）；2) 实现ConfigRepository接口，提供配置的CRUD操作（Create、GetByType、Update、Delete）；3) 实现配置类型枚举和常量定义；4) 编写单元测试验证仓储层功能。所有代码必须遵循Golang编码规范和PostgreSQL数据库设计规范。

**详细文档**: [TASK003-01.md](TASK003-01.md)

---

### 子任务 2: 系统配置管理核心服务实现

**任务编号**: TASK003-02  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK003-01

**任务描述**: 实现系统配置管理的核心业务逻辑。包括：1) 创建ConfigService接口和实现，提供配置的读取、更新、验证功能；2) 实现Redis缓存管理，配置读取优先从缓存获取，更新时清除缓存；3) 实现配置验证器，为每种配置类型提供验证规则；4) 实现配置热更新机制；5) 编写单元测试验证服务层功能。所有代码必须遵循Golang编码规范，错误消息使用中文。

**详细文档**: [TASK003-02.md](TASK003-02.md)

---

### 子任务 3: 系统配置管理API - 基础配置

**任务编号**: TASK003-03  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK003-02

**任务描述**: 实现系统配置管理的基础配置API接口。包括：1) 网站设置API（GET/PUT /api/v1/system/config/site）；2) 注册设置API（GET/PUT /api/v1/system/config/register）；3) 站群设置API（GET/PUT /api/v1/system/config/cluster，包括节点管理接口）；4) 积分设置API（GET/PUT /api/v1/system/config/points）。所有API必须遵循OpenAPI 3.1.1规范和Envelope响应格式，需要管理员权限的接口必须进行权限验证。编写API测试验证接口功能。

**详细文档**: [TASK003-03.md](TASK003-03.md)

---

### 子任务 4: 系统配置管理API - 高级配置

**任务编号**: TASK003-04  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK003-03

**任务描述**: 实现系统配置管理的高级配置API接口。包括：1) 附件设置API（GET/PUT /api/v1/system/config/attachment/image、audio、video，存储渠道管理接口）；2) 邮箱设置API（GET/PUT /api/v1/system/config/email，包括测试邮箱配置接口）；3) 屏蔽设置API（GET/PUT /api/v1/system/config/shield）。所有API必须遵循OpenAPI 3.1.1规范和Envelope响应格式，需要管理员权限的接口必须进行权限验证。编写API测试验证接口功能。

**详细文档**: [TASK003-04.md](TASK003-04.md)

---

### 子任务 5: 模块管理数据库模型和仓储层实现

**任务编号**: TASK003-05  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 创建模块管理的GORM模型和仓储层实现。包括：1) 创建Module模型，定义模块表结构（id、module_id、module_name、module_type、module_url、module_status、health_check_url、config_data、last_health_check、created_at、updated_at）；2) 实现ModuleRepository接口，提供模块的CRUD操作（Create、GetByID、GetByModuleID、List、Update、Delete）；3) 实现模块类型和状态枚举定义；4) 编写单元测试验证仓储层功能。所有代码必须遵循Golang编码规范和PostgreSQL数据库设计规范。

**详细文档**: [TASK003-05.md](TASK003-05.md)

---

### 子任务 6: 模块管理核心服务实现

**任务编号**: TASK003-06  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK003-05

**任务描述**: 实现模块管理的核心业务逻辑。包括：1) 创建ModuleService接口和实现，提供模块注册、启用、禁用、健康检查功能；2) 实现模块状态管理，支持状态转换和验证；3) 实现gRPC健康检查机制，使用grpc_health_v1.HealthClient检查模块健康状态；4) 实现定时健康检查任务（使用goroutine）；5) 实现健康检查重试和超时机制；6) 编写单元测试验证服务层功能。所有代码必须遵循Golang编码规范，错误消息使用中文。

**详细文档**: [TASK003-06.md](TASK003-06.md)

---

### 子任务 7: 模块管理API实现

**任务编号**: TASK003-07  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK003-06

**任务描述**: 实现模块管理的API接口。包括：1) 获取模块列表API（GET /api/v1/modules，支持分页）；2) 获取模块详情API（GET /api/v1/modules/{module_id}）；3) 注册模块API（POST /api/v1/modules，需要管理员权限）；4) 启用模块API（PUT /api/v1/modules/{module_id}/enable，需要管理员权限）；5) 禁用模块API（PUT /api/v1/modules/{module_id}/disable，需要管理员权限）；6) 检查模块健康状态API（GET /api/v1/modules/{module_id}/health）。所有API必须遵循OpenAPI 3.1.1规范和Envelope响应格式。编写API测试验证接口功能。

**详细文档**: [TASK003-07.md](TASK003-07.md)

---

### 子任务 8: 集成测试和文档完善

**任务编号**: TASK003-08  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK003-04、TASK003-07

**任务描述**: 完成TASK003的集成测试和文档编写。包括：1) 编写集成测试，测试系统配置管理和模块管理的完整流程；2) 验证配置缓存机制和热更新功能；3) 验证模块健康检查机制；4) 编写API文档，使用OpenAPI 3.1.1格式；5) 检查代码覆盖率，确保达到80%以上；6) 进行代码审查，确保符合编码规范。所有测试必须通过，文档必须完整。

**详细文档**: [TASK003-08.md](TASK003-08.md)

---

## Git 提交信息任务

任务完成后，需要生成 Git 提交信息。请根据实际修改内容生成合适的提交信息。

---

## 参考文档

- [项目整体任务文档](../TASKS.md)
- [项目整体需求文档](../../Requirements/Overview.md)
- [核心模块需求文档](../../Requirements/Core.md)
- [API设计规范](../../../.cursor/rules/api.mdc)
- [Golang编码规范](../../../.cursor/rules/golang.mdc)
- [数据库设计规范](../../../.cursor/rules/database.mdc)

---

**文档版本**: 1.0  
**最后更新**: 2026-01-02 21:30:48 CST

