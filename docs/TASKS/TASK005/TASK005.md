# TASK005: 用户管理系统

## 任务信息

**任务编号**: TASK005  
**任务名称**: 用户管理系统  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**创建时间**: 2026-01-03 20:06:59 CST

## 任务描述

实现用户管理功能，包括用户注册、登录、用户信息管理、用户状态管理、用户认证等。实现JWT认证机制，支持用户会话管理。

这是核心模块的基础功能任务，为后续作者管理、内容管理等任务提供用户支撑。所有功能必须遵循项目开发规范，包括API设计规范、Golang编码规范、数据库设计规范、安全开发规范等。

**依赖任务**: 本项目依赖TASK001（基础框架）、TASK002（数据库设计和初始化，包括users表）、TASK004（权限管理系统，部分功能依赖）。

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

#### TASK005-01: 创建User模型

**任务编号**: TASK005-01  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 创建用户管理的GORM模型，定义用户表结构。包括：1) 创建User模型，定义字段（基于TASK002-10的users表设计）；2) 添加模型标签和索引定义；3) 添加表注释和字段注释。所有代码必须遵循Golang编码规范和PostgreSQL数据库设计规范。参考TASK002-10数据库设计任务。

**详细文档**: [TASK005-01.md](TASK005-01.md)

---

#### TASK005-02: 创建UserSession模型

**任务编号**: TASK005-02  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 创建用户会话管理的GORM模型，用于JWT token存储和黑名单管理。包括：1) 创建UserSession模型，定义字段（id、user_id、token、token_type、expires_at、created_at等）；2) 添加模型标签和索引定义；3) 添加表注释。所有代码必须遵循Golang编码规范。

**详细文档**: [TASK005-02.md](TASK005-02.md)

---

#### TASK005-03: 创建UserLoginLog模型

**任务编号**: TASK005-03  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 创建用户登录日志的GORM模型，用于记录用户登录历史。包括：1) 创建UserLoginLog模型，定义字段（id、user_id、ip_address、user_agent、login_at、status等）；2) 添加模型标签和索引定义；3) 添加表注释。所有代码必须遵循Golang编码规范。

**详细文档**: [TASK005-03.md](TASK005-03.md)

---

### 仓储层（3个任务）

#### TASK005-04: 实现UserRepository接口和实现

**任务编号**: TASK005-04  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-01

**任务描述**: 实现UserRepository接口，提供用户数据的CRUD操作。包括：1) 定义UserRepository接口（Create、GetByID、GetByUsername、GetByEmail、GetAll、Update、Delete、UpdateStatus、UpdateLastLoginAt等方法）；2) 实现UserRepository接口（使用GORM进行数据库操作）；3) 实现条件查询方法（支持分页、筛选、排序）。所有代码必须遵循Golang编码规范，错误消息使用中文。

**详细文档**: [TASK005-04.md](TASK005-04.md)

---

#### TASK005-05: 实现SessionRepository接口和实现

**任务编号**: TASK005-05  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-02

**任务描述**: 实现SessionRepository接口，提供会话数据的存储和查询操作。包括：1) 定义SessionRepository接口（Create、GetByToken、BlacklistToken、IsBlacklisted、DeleteExpired等方法）；2) 实现SessionRepository接口（可以使用数据库或Redis）；3) 实现token黑名单管理。所有代码必须遵循Golang编码规范。

**详细文档**: [TASK005-05.md](TASK005-05.md)

---

#### TASK005-06: 编写UserRepository单元测试

**任务编号**: TASK005-06  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-04

**任务描述**: 编写用户仓储层单元测试。包括：1) 测试UserRepository的所有方法；2) 测试正常情况、边界情况和异常情况；3) 使用测试数据库或Mock进行测试；4) 确保测试覆盖率达标（80%以上）。所有测试用例描述和错误消息必须使用中文。

**详细文档**: [TASK005-06.md](TASK005-06.md)

---

### 核心服务层（8个任务）

#### TASK005-07: 创建PasswordService接口和实现

**任务编号**: TASK005-07  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: 无

**任务描述**: 创建密码服务，提供密码加密和验证功能。包括：1) 定义PasswordService接口（HashPassword、VerifyPassword、ValidatePasswordStrength等方法）；2) 实现密码哈希（使用bcrypt，成本因子12）；3) 实现密码验证；4) 实现密码强度验证（根据配置验证密码规则）。所有代码必须遵循Golang编码规范和 security.mdc 安全规范。

**详细文档**: [TASK005-07.md](TASK005-07.md)

---

#### TASK005-08: 编写PasswordService单元测试

**任务编号**: TASK005-08  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-07

**任务描述**: 编写密码服务单元测试。包括：1) 测试密码哈希和验证功能；2) 测试密码强度验证功能；3) 测试各种密码规则（长度、字符类型等）；4) 确保测试覆盖率达标。所有测试用例描述必须使用中文。

**详细文档**: [TASK005-08.md](TASK005-08.md)

---

#### TASK005-09: 创建JWTService接口和实现

**任务编号**: TASK005-09  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-05

**任务描述**: 创建JWT服务，提供token生成、验证和刷新功能。包括：1) 定义JWTService接口（GenerateToken、ValidateToken、RefreshToken、BlacklistToken等方法）；2) 实现token生成（使用github.com/golang-jwt/jwt/v5，HS256算法）；3) 实现token验证；4) 实现token刷新；5) 集成SessionRepository进行token黑名单管理。所有代码必须遵循Golang编码规范和安全规范。

**详细文档**: [TASK005-09.md](TASK005-09.md)

---

#### TASK005-10: 编写JWTService单元测试

**任务编号**: TASK005-10  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-09

**任务描述**: 编写JWT服务单元测试。包括：1) 测试token生成和验证功能；2) 测试token刷新功能；3) 测试token黑名单功能；4) 测试过期token处理；5) 确保测试覆盖率达标。所有测试用例描述必须使用中文。

**详细文档**: [TASK005-10.md](TASK005-10.md)

---

#### TASK005-11: 创建AuthService接口和实现 - 登录和登出

**任务编号**: TASK005-11  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-04、TASK005-07、TASK005-09

**任务描述**: 创建认证服务，提供用户登录和登出功能。包括：1) 定义AuthService接口（Login、Logout方法）；2) 实现登录逻辑（验证用户名/邮箱和密码、生成JWT token、记录登录日志、更新最后登录时间）；3) 实现登出逻辑（将token加入黑名单）。所有代码必须遵循Golang编码规范，错误消息使用中文。

**详细文档**: [TASK005-11.md](TASK005-11.md)

---

#### TASK005-12: 创建AuthService接口和实现 - 用户注册

**任务编号**: TASK005-12  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-11、TASK005-07

**任务描述**: 扩展认证服务，提供用户注册功能。包括：1) 扩展AuthService接口（Register方法）；2) 实现注册逻辑（验证注册配置、验证用户名和邮箱唯一性、验证密码强度、创建用户、生成JWT token、分配注册积分等）。所有代码必须遵循Golang编码规范，错误消息使用中文。

**详细文档**: [TASK005-12.md](TASK005-12.md)

---

#### TASK005-13: 编写AuthService单元测试

**任务编号**: TASK005-13  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-12

**任务描述**: 编写认证服务单元测试。包括：1) 测试登录功能（正常登录、错误密码、用户锁定等）；2) 测试注册功能（正常注册、重复用户名、弱密码等）；3) 测试登出功能；4) 使用Mock Repository和Service进行测试；5) 确保测试覆盖率达标。所有测试用例描述必须使用中文。

**详细文档**: [TASK005-13.md](TASK005-13.md)

---

### 用户服务层（4个任务）

#### TASK005-14: 创建UserService接口和实现 - 基础CRUD

**任务编号**: TASK005-14  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-04、TASK005-07

**任务描述**: 创建用户服务，提供用户管理的基础CRUD功能。包括：1) 定义UserService接口（GetUser、GetUsers、CreateUser、UpdateUser、DeleteUser方法）；2) 实现用户查询（单个用户、用户列表，支持分页和筛选）；3) 实现用户创建（管理员创建用户）；4) 实现用户更新；5) 实现用户删除。所有代码必须遵循Golang编码规范，错误消息使用中文。

**详细文档**: [TASK005-14.md](TASK005-14.md)

---

#### TASK005-15: 创建UserService接口和实现 - 状态和封禁管理

**任务编号**: TASK005-15  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-14

**任务描述**: 扩展用户服务，提供用户状态管理和封禁管理功能。包括：1) 扩展UserService接口（UpdateStatus、BanUser、UnbanUser方法）；2) 实现状态更新（激活/禁用用户）；3) 实现用户封禁（设置is_locked为true，可以添加封禁原因和封禁期限）；4) 实现用户解封。所有代码必须遵循Golang编码规范，错误消息使用中文。

**详细文档**: [TASK005-15.md](TASK005-15.md)

---

#### TASK005-16: 创建UserService接口和实现 - 审核和密码管理

**任务编号**: TASK005-16  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-14、TASK005-07

**任务描述**: 扩展用户服务，提供用户审核和密码管理功能。包括：1) 扩展UserService接口（ApproveUser、RejectUser、ResetPassword方法）；2) 实现用户审核（如果注册需要审核，设置用户状态为已审核）；3) 实现密码重置（管理员重置用户密码）。所有代码必须遵循Golang编码规范，错误消息使用中文。

**详细文档**: [TASK005-16.md](TASK005-16.md)

---

#### TASK005-17: 编写UserService单元测试

**任务编号**: TASK005-17  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-16

**任务描述**: 编写用户服务单元测试。包括：1) 测试所有CRUD方法；2) 测试状态管理和封禁管理功能；3) 测试审核和密码管理功能；4) 使用Mock Repository和Service进行测试；5) 确保测试覆盖率达标。所有测试用例描述必须使用中文。

**详细文档**: [TASK005-17.md](TASK005-17.md)

---

### API控制器层（9个任务）

#### TASK005-18: 创建UserHandler接口和实现 - 用户注册API

**任务编号**: TASK005-18  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-12

**任务描述**: 创建用户控制器，提供用户注册的HTTP API处理。包括：1) 定义UserHandler接口；2) 实现Register方法（处理POST /api/v1/users/register请求）；3) 实现请求参数绑定和验证；4) 调用AuthService.Register；5) 格式化响应（Envelope格式）。所有代码必须遵循Golang编码规范和API设计规范，错误消息使用中文。

**详细文档**: [TASK005-18.md](TASK005-18.md)

---

#### TASK005-19: 创建UserHandler接口和实现 - 用户登录API

**任务编号**: TASK005-19  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-18、TASK005-11

**任务描述**: 扩展用户控制器，提供用户登录的HTTP API处理。包括：1) 扩展UserHandler接口（Login方法）；2) 实现Login方法（处理POST /api/v1/users/login请求）；3) 实现请求参数绑定和验证；4) 调用AuthService.Login；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。

**详细文档**: [TASK005-19.md](TASK005-19.md)

---

#### TASK005-20: 创建UserHandler接口和实现 - 用户登出API

**任务编号**: TASK005-20  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-19、TASK005-11

**任务描述**: 扩展用户控制器，提供用户登出的HTTP API处理。包括：1) 扩展UserHandler接口（Logout方法）；2) 实现Logout方法（处理POST /api/v1/users/logout请求）；3) 从请求头获取JWT token；4) 调用AuthService.Logout；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。

**详细文档**: [TASK005-20.md](TASK005-20.md)

---

#### TASK005-21: 创建UserHandler接口和实现 - 用户列表和详情API

**任务编号**: TASK005-21  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-20、TASK005-14

**任务描述**: 扩展用户控制器，提供用户列表查询和详情查询的HTTP API处理。包括：1) 扩展UserHandler接口（GetUsers、GetUser方法）；2) 实现GetUsers方法（处理GET /api/v1/users请求，支持分页、筛选、排序）；3) 实现GetUser方法（处理GET /api/v1/users/{id}请求）；4) 调用UserService；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。

**详细文档**: [TASK005-21.md](TASK005-21.md)

---

#### TASK005-22: 创建UserHandler接口和实现 - 用户创建和更新API

**任务编号**: TASK005-22  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-21、TASK005-14

**任务描述**: 扩展用户控制器，提供用户创建和更新的HTTP API处理。包括：1) 扩展UserHandler接口（CreateUser、UpdateUser方法）；2) 实现CreateUser方法（处理POST /api/v1/users请求）；3) 实现UpdateUser方法（处理PUT /api/v1/users/{id}请求）；4) 调用UserService；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。

**详细文档**: [TASK005-22.md](TASK005-22.md)

---

#### TASK005-23: 创建UserHandler接口和实现 - 用户状态管理API

**任务编号**: TASK005-23  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-22、TASK005-15

**任务描述**: 扩展用户控制器，提供用户状态管理的HTTP API处理。包括：1) 扩展UserHandler接口（UpdateUserStatus方法）；2) 实现UpdateUserStatus方法（处理PUT /api/v1/users/{id}/status请求）；3) 调用UserService.UpdateStatus；4) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。

**详细文档**: [TASK005-23.md](TASK005-23.md)

---

#### TASK005-24: 创建UserHandler接口和实现 - 用户封禁管理API

**任务编号**: TASK005-24  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-23、TASK005-15

**任务描述**: 扩展用户控制器，提供用户封禁管理的HTTP API处理。包括：1) 扩展UserHandler接口（BanUser、UnbanUser方法）；2) 实现BanUser方法（处理PUT /api/v1/users/{id}/ban请求）；3) 实现UnbanUser方法（处理PUT /api/v1/users/{id}/unban请求，或使用同一个端点通过参数区分）；4) 调用UserService；5) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。

**详细文档**: [TASK005-24.md](TASK005-24.md)

---

#### TASK005-25: 创建UserHandler接口和实现 - 用户审核和密码管理API

**任务编号**: TASK005-25  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-24、TASK005-16

**任务描述**: 扩展用户控制器，提供用户审核和密码管理的HTTP API处理。包括：1) 扩展UserHandler接口（ApproveUser、RejectUser、ResetPassword方法）；2) 实现ApproveUser方法（处理PUT /api/v1/users/{id}/approve请求）；3) 实现RejectUser方法（处理PUT /api/v1/users/{id}/reject请求，可选）；4) 实现ResetPassword方法（处理PUT /api/v1/users/{id}/password请求）；5) 调用UserService；6) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。

**详细文档**: [TASK005-25.md](TASK005-25.md)

---

#### TASK005-26: 创建UserHandler接口和实现 - 用户删除API

**任务编号**: TASK005-26  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-25、TASK005-14

**任务描述**: 扩展用户控制器，提供用户删除的HTTP API处理。包括：1) 扩展UserHandler接口（DeleteUser方法）；2) 实现DeleteUser方法（处理DELETE /api/v1/users/{id}请求）；3) 调用UserService.DeleteUser；4) 格式化响应（Envelope格式）。所有代码必须遵循API设计规范。

**详细文档**: [TASK005-26.md](TASK005-26.md)

---

### 中间件和路由（2个任务）

#### TASK005-27: 创建JWT认证中间件

**任务编号**: TASK005-27  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-09

**任务描述**: 创建JWT认证中间件，用于验证HTTP请求中的JWT token。包括：1) 创建JWTAuthMiddleware函数；2) 从请求头提取JWT token（Authorization: Bearer {token}）；3) 验证token（使用JWTService）；4) 将用户信息存储到上下文（context）中；5) 如果token无效，返回401错误。所有代码必须遵循Golang编码规范和API设计规范。

**详细文档**: [TASK005-27.md](TASK005-27.md)

---

#### TASK005-28: 配置用户管理API路由

**任务编号**: TASK005-28  
**版本信息**: v1.1.0  
**当前状态**: 计划中  
**依赖任务**: TASK005-26、TASK005-27

**任务描述**: 配置用户管理的API路由。包括：1) 在路由配置文件中定义用户管理路由；2) 配置公开路由（注册、登录）；3) 配置需要认证的路由（登出、用户信息查询等）；4) 配置需要管理员权限的路由（用户管理操作）；5) 配置JWT认证中间件和权限中间件（如果TASK004完成）。所有路由必须遵循OpenAPI 3.1.1规范，路径使用/api/v1/users前缀。

**详细文档**: [TASK005-28.md](TASK005-28.md)

---

## 参考文档

- [项目整体任务文档](../TASKS.md)
- [核心模块需求文档](../../Requirements/Core.md)
- [核心模块功能概述](../../Features/Core.md)
- [用户表数据库设计文档](../TASK002/TASK002-10.md)

---

**文档版本**: 1.0  
**最后更新**: 2026-01-03 20:06:59 CST

