# TASK006: 作者管理系统

## 任务信息

**任务编号**: TASK006  
**任务名称**: 作者管理系统  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**创建时间**: 2026-01-03 20:48:19 CST

## 任务描述

实现作者管理功能，包括作者注册、登录、作者信息管理、作者状态管理、作者认证等。实现作者与用户的关联关系，支持作者权限管理。

这是核心模块的完整功能任务，为后续内容管理等任务提供作者支撑。所有功能必须遵循项目开发规范，包括API设计规范、Golang编码规范、数据库设计规范、安全开发规范等。

**依赖任务**: 本项目依赖TASK001（基础框架）、TASK002（数据库设计和初始化，包括authors表）、TASK004（权限管理系统，部分功能依赖）、TASK005（用户管理系统，复用部分服务）。

## 技术栈

- **开发语言**: Golang 1.24+
- **开发框架**: Gin
- **数据库**: PostgreSQL
- **缓存**: Redis
- **API 规范**: OpenAPI 3.1.1
- **响应格式**: Envelope 格式
- **认证机制**: JWT (JSON Web Token)
- **密码加密**: bcrypt

## 任务拆分说明

> **重要提示**: 本任务已按照"一个任务仅操作一个功能"的标准进行细化拆分。每个任务专注于一个独立的操作步骤，遵循以下拆分原则：
> - 创建模型结构 = 一个任务
> - 实现接口和实现 = 一个任务
> - 编写单元测试 = 一个任务
> - 实现API端点 = 一个任务
> - 配置中间件和路由 = 一个任务

## 子任务列表

### 数据模型层（3个任务）

#### TASK006-01: 创建Author模型

**任务编号**: TASK006-01  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 创建作者管理的GORM模型，定义作者表结构。包括：1) 创建Author模型，定义字段（基于TASK002-13的authors表设计，包含author_name、email、phone、password_hash、email_verified、phone_verified、is_active、is_locked、author_type_ids数组、author_level_id、user_id等）；2) 添加模型标签和索引定义；3) 添加表注释和字段注释。所有代码必须遵循Golang编码规范和PostgreSQL数据库设计规范。参考TASK002-13数据库设计任务和TASK005-01用户模型实现。

**详细文档**: [TASK006-01.md](TASK006-01.md)

---

#### TASK006-02: 创建AuthorSession模型

**任务编号**: TASK006-02  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 创建作者会话管理的GORM模型，用于JWT token存储和黑名单管理。包括：1) 创建AuthorSession模型，定义字段（id、author_id、token、token_type、expires_at、created_at等）；2) 添加模型标签和索引定义；3) 添加表注释。所有代码必须遵循Golang编码规范。参考TASK005-02用户会话模型实现。

**详细文档**: [TASK006-02.md](TASK006-02.md)

---

#### TASK006-03: 创建AuthorLoginLog模型

**任务编号**: TASK006-03  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 创建作者登录日志的GORM模型，用于记录作者登录历史。包括：1) 创建AuthorLoginLog模型，定义字段（id、author_id、ip_address、user_agent、login_at、status等）；2) 添加模型标签和索引定义；3) 添加表注释。所有代码必须遵循Golang编码规范。参考TASK005-03用户登录日志模型实现。

**详细文档**: [TASK006-03.md](TASK006-03.md)

---

### 仓储层（3个任务）

#### TASK006-04: 实现AuthorRepository接口和实现

**任务编号**: TASK006-04  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-01

**任务描述**: 实现AuthorRepository接口，提供作者数据的CRUD操作。包括：1) 定义AuthorRepository接口（Create、GetByID、GetByAuthorName、GetByEmail、GetAll、Update、Delete、UpdateStatus、UpdateLastLoginAt等方法）；2) 实现AuthorRepository接口（使用GORM进行数据库操作）；3) 实现条件查询方法（支持分页、筛选、排序，包括按author_type_ids数组查询）。所有代码必须遵循Golang编码规范，错误消息使用中文。参考TASK005-04用户仓储层实现。

**详细文档**: [TASK006-04.md](TASK006-04.md)

---

#### TASK006-05: 实现AuthorSessionRepository接口和实现

**任务编号**: TASK006-05  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-02

**任务描述**: 实现AuthorSessionRepository接口，提供作者会话数据的存储和查询操作。包括：1) 定义AuthorSessionRepository接口（Create、GetByToken、BlacklistToken、IsBlacklisted、DeleteExpired等方法）；2) 实现AuthorSessionRepository接口（可以使用数据库或Redis）；3) 实现token黑名单管理。所有代码必须遵循Golang编码规范。参考TASK005-05会话仓储层实现。

**详细文档**: [TASK006-05.md](TASK006-05.md)

---

#### TASK006-06: 编写AuthorRepository单元测试

**任务编号**: TASK006-06  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-04

**任务描述**: 编写作者仓储层单元测试。包括：1) 测试AuthorRepository的所有方法；2) 测试正常情况、边界情况和异常情况；3) 测试数组查询（author_type_ids）；4) 使用测试数据库或Mock进行测试；5) 确保测试覆盖率达标（80%以上）。所有测试用例描述和错误消息必须使用中文。参考TASK005-06用户仓储层单元测试。

**详细文档**: [TASK006-06.md](TASK006-06.md)

---

### 核心服务层（8个任务）

#### TASK006-07: 复用PasswordService（无需修改）

**任务编号**: TASK006-07  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 复用TASK005-07实现的PasswordService，无需修改。作者管理系统使用相同的密码加密和验证逻辑。在AuthorService中注入PasswordService依赖。

**详细文档**: [TASK006-07.md](TASK006-07.md)

---

#### TASK006-08: 扩展JWTService支持作者token

**任务编号**: TASK006-08  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-05

**任务描述**: 扩展TASK005-09实现的JWTService，支持作者类型的token。包括：1) 定义AuthorClaims结构体（包含author_id、author_name、author_type_ids等）；2) 扩展JWTService接口（GenerateAuthorToken、ValidateAuthorToken等方法）；3) 实现作者token生成和验证。所有代码必须遵循Golang编码规范和安全规范。参考TASK005-09 JWT服务实现。

**详细文档**: [TASK006-08.md](TASK006-08.md)

---

#### TASK006-09: 编写JWTService扩展单元测试

**任务编号**: TASK006-09  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-08

**任务描述**: 编写JWT服务扩展的单元测试。包括：1) 测试作者token生成和验证功能；2) 测试token黑名单功能；3) 测试过期token处理；4) 确保测试覆盖率达标。所有测试用例描述必须使用中文。参考TASK005-10 JWT服务单元测试。

**详细文档**: [TASK006-09.md](TASK006-09.md)

---

#### TASK006-10: 创建AuthService扩展 - 作者登录和登出

**任务编号**: TASK006-10  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-04、TASK006-07、TASK006-08

**任务描述**: 扩展TASK005-11实现的AuthService，提供作者登录和登出功能。包括：1) 扩展AuthService接口（AuthorLogin、AuthorLogout方法）；2) 实现作者登录逻辑（验证作者名/邮箱和密码、生成JWT token、记录登录日志、更新最后登录时间）；3) 实现作者登出逻辑（将token加入黑名单）。所有代码必须遵循Golang编码规范，错误消息使用中文。参考TASK005-11认证服务实现。

**详细文档**: [TASK006-10.md](TASK006-10.md)

---

#### TASK006-11: 创建AuthService扩展 - 作者注册

**任务编号**: TASK006-11  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-10、TASK006-07

**任务描述**: 扩展TASK005-12实现的AuthService，提供作者注册功能。包括：1) 扩展AuthService接口（AuthorRegister方法）；2) 实现注册逻辑（验证注册配置、验证作者名和邮箱唯一性、验证密码强度、创建作者、生成JWT token、分配注册积分等）。所有代码必须遵循Golang编码规范，错误消息使用中文。参考TASK005-12用户注册实现。

**详细文档**: [TASK006-11.md](TASK006-11.md)

---

#### TASK006-12: 编写AuthService扩展单元测试

**任务编号**: TASK006-12  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-11

**任务描述**: 编写认证服务扩展的单元测试。包括：1) 测试作者登录功能（正常登录、错误密码、作者锁定等）；2) 测试作者注册功能（正常注册、重复作者名、弱密码等）；3) 测试作者登出功能；4) 使用Mock Repository和Service进行测试；5) 确保测试覆盖率达标。所有测试用例描述必须使用中文。参考TASK005-13认证服务单元测试。

**详细文档**: [TASK006-12.md](TASK006-12.md)

---

### 作者服务层（4个任务）

#### TASK006-13: 创建AuthorService接口和实现 - 基础CRUD

**任务编号**: TASK006-13  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-04、TASK006-07

**任务描述**: 创建作者服务，提供作者管理的基础CRUD功能。包括：1) 定义AuthorService接口（GetAuthor、GetAuthors、CreateAuthor、UpdateAuthor、DeleteAuthor方法）；2) 实现作者查询（单个作者、作者列表，支持分页和筛选）；3) 实现作者创建（管理员创建作者）；4) 实现作者更新；5) 实现作者删除。所有代码必须遵循Golang编码规范，错误消息使用中文。参考TASK005-14用户服务实现。

**详细文档**: [TASK006-13.md](TASK006-13.md)

---

#### TASK006-14: 创建AuthorService接口和实现 - 状态和封禁管理

**任务编号**: TASK006-14  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-13

**任务描述**: 扩展作者服务，提供作者状态管理和封禁管理功能。包括：1) 扩展AuthorService接口（UpdateStatus、BanAuthor、UnbanAuthor方法）；2) 实现状态更新（激活/禁用作者）；3) 实现作者封禁（设置is_locked为true，可以添加封禁原因和封禁期限）；4) 实现作者解封。所有代码必须遵循Golang编码规范，错误消息使用中文。参考TASK005-15用户状态和封禁管理实现。

**详细文档**: [TASK006-14.md](TASK006-14.md)

---

#### TASK006-15: 创建AuthorService接口和实现 - 审核、奖励和惩罚

**任务编号**: TASK006-15  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-14

**任务描述**: 扩展作者服务，提供作者审核、奖励和惩罚功能。包括：1) 扩展AuthorService接口（ApproveAuthor、RejectAuthor、RewardAuthor、PunishAuthor方法）；2) 实现作者审核（审核通过/拒绝）；3) 实现作者奖励（管理员奖励作者，可能涉及积分或货币操作）；4) 实现作者惩罚（管理员惩罚作者）。所有代码必须遵循Golang编码规范，错误消息使用中文。

**详细文档**: [TASK006-15.md](TASK006-15.md)

---

#### TASK006-16: 编写AuthorService单元测试

**任务编号**: TASK006-16  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-15

**任务描述**: 编写作者服务单元测试。包括：1) 测试所有CRUD方法；2) 测试状态管理和封禁管理功能；3) 测试审核、奖励、惩罚功能；4) 使用Mock Repository和Service进行测试；5) 确保测试覆盖率达标。所有测试用例描述必须使用中文。参考TASK005-17用户服务单元测试。

**详细文档**: [TASK006-16.md](TASK006-16.md)

---

### API控制器层（9个任务）

#### TASK006-17: 创建AuthorHandler接口和实现 - 作者注册API

**任务编号**: TASK006-17  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-11

**任务描述**: 创建作者控制器，提供作者注册的HTTP API处理。包括：1) 定义AuthorHandler接口；2) 实现Register方法（处理POST /api/v1/authors/register请求）；3) 实现请求参数绑定和验证；4) 调用AuthService.AuthorRegister；5) 格式化响应（Envelope格式）。所有代码必须遵循Golang编码规范和API设计规范，错误消息使用中文。参考TASK005-18用户注册API实现。

**详细文档**: [TASK006-17.md](TASK006-17.md)

---

#### TASK006-18: 创建AuthorHandler接口和实现 - 作者登录API

**任务编号**: TASK006-18  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-17、TASK006-10

**任务描述**: 扩展作者控制器，提供作者登录的HTTP API处理。包括：1) 扩展AuthorHandler接口（Login方法）；2) 实现Login方法（处理POST /api/v1/authors/login请求）；3) 实现请求参数绑定和验证；4) 调用AuthService.AuthorLogin；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。参考TASK005-19用户登录API实现。

**详细文档**: [TASK006-18.md](TASK006-18.md)

---

#### TASK006-19: 创建AuthorHandler接口和实现 - 作者登出API

**任务编号**: TASK006-19  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-18、TASK006-10

**任务描述**: 扩展作者控制器，提供作者登出的HTTP API处理。包括：1) 扩展AuthorHandler接口（Logout方法）；2) 实现Logout方法（处理POST /api/v1/authors/logout请求）；3) 从请求头获取JWT token；4) 调用AuthService.AuthorLogout；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。参考TASK005-20用户登出API实现。

**详细文档**: [TASK006-19.md](TASK006-19.md)

---

#### TASK006-20: 创建AuthorHandler接口和实现 - 作者列表和详情API

**任务编号**: TASK006-20  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-19、TASK006-13

**任务描述**: 扩展作者控制器，提供作者列表查询和详情查询的HTTP API处理。包括：1) 扩展AuthorHandler接口（GetAuthors、GetAuthor方法）；2) 实现GetAuthors方法（处理GET /api/v1/authors请求，支持分页、筛选、排序）；3) 实现GetAuthor方法（处理GET /api/v1/authors/{id}请求）；4) 调用AuthorService；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。参考TASK005-21用户列表和详情API实现。

**详细文档**: [TASK006-20.md](TASK006-20.md)

---

#### TASK006-21: 创建AuthorHandler接口和实现 - 作者创建和更新API

**任务编号**: TASK006-21  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-20、TASK006-13

**任务描述**: 扩展作者控制器，提供作者创建和更新的HTTP API处理。包括：1) 扩展AuthorHandler接口（CreateAuthor、UpdateAuthor方法）；2) 实现CreateAuthor方法（处理POST /api/v1/authors请求）；3) 实现UpdateAuthor方法（处理PUT /api/v1/authors/{id}请求）；4) 调用AuthorService；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。参考TASK005-22用户创建和更新API实现。

**详细文档**: [TASK006-21.md](TASK006-21.md)

---

#### TASK006-22: 创建AuthorHandler接口和实现 - 作者状态管理API

**任务编号**: TASK006-22  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-21、TASK006-14

**任务描述**: 扩展作者控制器，提供作者状态管理的HTTP API处理。包括：1) 扩展AuthorHandler接口（UpdateAuthorStatus方法）；2) 实现UpdateAuthorStatus方法（处理PUT /api/v1/authors/{id}/status请求）；3) 调用AuthorService.UpdateStatus；4) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。参考TASK005-23用户状态管理API实现。

**详细文档**: [TASK006-22.md](TASK006-22.md)

---

#### TASK006-23: 创建AuthorHandler接口和实现 - 作者封禁管理API

**任务编号**: TASK006-23  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-22、TASK006-14

**任务描述**: 扩展作者控制器，提供作者封禁管理的HTTP API处理。包括：1) 扩展AuthorHandler接口（BanAuthor、UnbanAuthor方法）；2) 实现BanAuthor方法（处理PUT /api/v1/authors/{id}/ban请求）；3) 实现UnbanAuthor方法（处理PUT /api/v1/authors/{id}/unban请求，或使用同一个端点通过参数区分）；4) 调用AuthorService；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。参考TASK005-24用户封禁管理API实现。

**详细文档**: [TASK006-23.md](TASK006-23.md)

---

#### TASK006-24: 创建AuthorHandler接口和实现 - 作者审核、奖励和惩罚API

**任务编号**: TASK006-24  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-23、TASK006-15

**任务描述**: 扩展作者控制器，提供作者审核、奖励和惩罚的HTTP API处理。包括：1) 扩展AuthorHandler接口（ApproveAuthor、RejectAuthor、RewardAuthor、PunishAuthor方法）；2) 实现ApproveAuthor方法（处理PUT /api/v1/authors/{id}/approve请求）；3) 实现RejectAuthor方法（处理PUT /api/v1/authors/{id}/reject请求，可选）；4) 实现RewardAuthor方法（处理PUT /api/v1/authors/{id}/reward请求）；5) 实现PunishAuthor方法（处理PUT /api/v1/authors/{id}/punish请求）；6) 调用AuthorService；7) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。参考TASK005-25用户审核和密码管理API实现。

**详细文档**: [TASK006-24.md](TASK006-24.md)

---

#### TASK006-25: 创建AuthorHandler接口和实现 - 作者删除API

**任务编号**: TASK006-25  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-24、TASK006-13

**任务描述**: 扩展作者控制器，提供作者删除的HTTP API处理。包括：1) 扩展AuthorHandler接口（DeleteAuthor方法）；2) 实现DeleteAuthor方法（处理DELETE /api/v1/authors/{id}请求）；3) 调用AuthorService.DeleteAuthor；4) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。参考TASK005-26用户删除API实现。

**详细文档**: [TASK006-25.md](TASK006-25.md)

---

### 中间件和路由（2个任务）

#### TASK006-26: 扩展JWT认证中间件支持作者token

**任务编号**: TASK006-26  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-08

**任务描述**: 扩展TASK005-27实现的JWT认证中间件，支持作者token验证。包括：1) 扩展JWTAuthMiddleware函数，支持作者token验证；2) 从请求头提取JWT token；3) 验证token类型（用户token或作者token）；4) 将作者信息存储到上下文（context）中；5) 如果token无效，返回401错误。所有代码必须遵循Golang编码规范和API设计规范。参考TASK005-27 JWT认证中间件实现。

**详细文档**: [TASK006-26.md](TASK006-26.md)

---

#### TASK006-27: 配置作者管理API路由

**任务编号**: TASK006-27  
**版本信息**: v1.2.0  
**当前状态**: 计划中  
**依赖任务**: TASK006-25、TASK006-26

**任务描述**: 配置作者管理的API路由。包括：1) 在路由配置文件中定义作者管理路由；2) 配置公开路由（注册、登录）；3) 配置需要认证的路由（登出、作者信息查询等）；4) 配置需要管理员权限的路由（作者管理操作）；5) 配置JWT认证中间件和权限中间件（如果TASK004完成）。所有路由必须遵循OpenAPI 3.1.1规范，路径使用/api/v1/authors前缀。参考TASK005-28用户管理API路由配置。

**详细文档**: [TASK006-27.md](TASK006-27.md)

---

## 参考文档

- [项目整体任务文档](../TASKS.md)
- [核心模块需求文档](../../Requirements/Core.md)
- [核心模块功能概述](../../Features/Core.md)
- [作者表数据库设计文档](../TASK002/TASK002-13.md)
- [用户管理系统任务文档](../TASK005/TASK005.md)

---

**文档版本**: 1.0  
**最后更新**: 2026-01-03 20:48:19 CST

