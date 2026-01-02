# 存储模块需求文档

## 文档说明

本文档为 LNCms 项目存储模块的详细需求文档，基于功能描述文档编写，遵循项目开发规范。

**文档版本**: 1.0  
**最后更新**: 2026-01-02 20:04:31 CST  
**编写依据**: [存储模块功能概述](../Features/Storage.md)

## 目录

- [概述](#概述)
- [功能需求](#功能需求)
- [非功能需求](#非功能需求)
- [接口需求](#接口需求)
- [数据需求](#数据需求)
- [安全需求](#安全需求)
- [验收标准](#验收标准)

## 概述

### 模块概述

存储模块（Storage）为程序存储模块，负责提供项目文件远端存储、转码、预览等功能。存储模块支持文件上传、远程存储、文件迁移、文件预览、文件管理和统计分析等功能。

### 技术栈

- **开发语言**: Golang 1.24+
- **开发框架**: Gin
- **数据库**: PostgreSQL
- **缓存**: Redis
- **API 规范**: OpenAPI 3.1.1

### 模块通信

#### API 接口

- 对外提供 RESTful HTTP API 接口服务
- 所有 API 遵循 OpenAPI 3.1.1 规范
- 所有响应统一遵循 Envelope 响应格式（包含 meta、data、errors 三个部分）
- 前端模块通过 RESTful HTTP API 接口与存储模块进行通信
- API 路径使用版本控制：`/api/v1/...`

#### gRPC 通信

- 对外提供 gRPC 接口服务
- 核心模块、日志模块、采集模块通过 gRPC 接口与存储模块进行通信
- 模块间使用 gRPC 均支持双向通信

## 功能需求

### 1. 文件存储

文件存储功能提供文件上传、远程存储和文件迁移等核心功能，支持文件去重、分片上传、转码处理等特性。

#### 1.1 文件上传

**功能描述**: 前端用户通过核心模块将文件上传至存储模块中。所有上传均遵循唯一性原则，同文件仅存储一份，数据库仅存储一条记录。

**业务流程**:

1. **上传初始化**: 前端用户提交文件名称、文件大小、文件哈希值到后端初始化上传
2. **唯一性检测**: 后端通过所提交的哈希值进行查询，若存在相同文件，则实现秒传，返回已有文件信息，流程结束
3. **缓存数据存储**: 唯一性检测未通过，则自动生成上传ID信息，并将ID、文件名、文件大小、哈希值写入 Redis 缓存记录
4. **返回初始化信息**: 后端返回上传ID、上传类型（小文件上传、分片上传）
5. **开始上传**: 前端获取到初始化信息，使用初始化信息进行文件上传，同时依据返回的上传类型使用不同的上传接口
6. **上传过程**: 针对小文件上传与分片上传，拥有不同处理流程，小文件上传直接一次性上传即可；分片上传则会在前端将文件以每个文件2至10MB左右进行分片，每个分片上传时会同步提交分片哈希值
7. **上传完成**: 小文件上传完成后等待后端处理；分片上传完成后，前端发起分片合并请求，提交上传ID，后端通过上传ID查询缓存记录进行分片合并
8. **文件校验**: 上传或合并完成后，后端通过缓存中的哈希值进行文件校验，确保文件正确
9. **文件处理**: 根据核心模块中附件转码设置，后端检测文件是否需要进行转码，需要转码则添加至 Redis 队列进行排队处理
10. **处理结束**: 当文件转码结束，根据核心模块中的附件存储路径配置，自动将转码后的文件存储到对应路径
11. **写入数据**: 获取转码后的文件信息，并将数据写入数据库中，哈希值写入原始文件哈希值
12. **返回数据**: 将写入的数据信息返回前端，完成文件上传
13. **进度查询**: 在转码过程中，前端可通过上传ID进行转码进度查询，转码结束后会写入转码记录中

**接口需求**:

- `POST /api/v1/storage/upload/init` - 初始化文件上传
- `POST /api/v1/storage/upload/small` - 小文件上传
- `POST /api/v1/storage/upload/chunk` - 分片上传
- `POST /api/v1/storage/upload/merge` - 分片合并
- `GET /api/v1/storage/upload/{upload_id}/progress` - 查询上传/转码进度

**数据验证**:

- 文件名称：字符串类型，必填，长度限制 1-255 字符
- 文件大小：整数类型，必填，非负整数，单位字节
- 文件哈希值：字符串类型，必填，SHA256 格式，64 字符
- 分片大小：整数类型，范围 2MB-10MB（2097152-10485760 字节）
- 分片哈希值：字符串类型，必填，SHA256 格式，64 字符

**业务规则**:

- 文件大小小于 10MB 时，使用小文件上传方式
- 文件大小大于等于 10MB 时，使用分片上传方式
- 分片大小由前端根据文件大小动态计算，建议范围 2MB-10MB
- 相同哈希值的文件仅存储一份，实现文件去重
- 转码任务通过 Redis 队列异步处理，支持进度查询

#### 1.2 远程存储

**功能描述**: 采集模块通过远程提交文件到存储模块。

**业务流程**:

1. **接收请求**: 采集模块通过远程下载方式，将采集到的文件临时存储到采集模块本地，并提交存储请求，提交文件名、文件大小、文件哈希值
2. **唯一性检测**: 存储模块通过所提交的哈希值进行查询，若存在相同文件，则实现秒传，返回已有文件信息，流程结束
3. **缓存数据存储**: 唯一性检测未通过，则自动生成上传ID信息，并将ID、文件名、文件大小、哈希值写入 Redis 缓存记录
4. **请求回馈**: 存储模块返回上传ID至采集模块
5. **开始上传**: 采集模块提交上传ID与上传文件进行上传
6. **文件校验**: 上传完成后，存储模块通过缓存中的哈希值进行文件校验，确保文件正确
7. **文件处理**: 根据核心模块中附件转码设置，后端检测文件是否需要进行转码，需要转码则添加至 Redis 队列进行排队处理
8. **处理结束**: 当文件转码结束，根据核心模块中的附件存储路径配置，自动将转码后的文件存储到对应路径
9. **写入数据**: 获取转码后的文件信息，并将数据写入数据库中，哈希值写入原始文件哈希值
10. **完成通知**: 将写入的数据信息通知采集模块，完成远程存储

**接口需求**:

- `POST /api/v1/storage/remote/init` - 初始化远程存储（gRPC）
- `POST /api/v1/storage/remote/upload` - 远程文件上传（gRPC）
- `POST /api/v1/storage/remote/complete` - 完成远程存储（gRPC）

**数据验证**:

- 文件名称：字符串类型，必填，长度限制 1-255 字符
- 文件大小：整数类型，必填，非负整数，单位字节
- 文件哈希值：字符串类型，必填，SHA256 格式，64 字符
- 采集模块标识：字符串类型，必填，用于标识来源模块

**业务规则**:

- 远程存储仅通过 gRPC 接口提供服务
- 支持文件去重，相同哈希值文件仅存储一份
- 转码任务异步处理，完成后通知采集模块

#### 1.3 文件迁移

**功能描述**: 针对现有的核心模块本地存储、云存储、其他存储模块数据，进行存储迁移。

**业务流程**:

1. **接收请求**: 存储模块接收核心模块发送的迁移请求，请求包括迁移类型（远端到本地、本地到远端）、远端连接信息、远端数据库信息、迁移文件列表、是否转码（仅远端到本地有效）、转码方式（同步转码、迁移后转码）等
2. **连接远端**: 通过核心模块发送的远端连接信息、远端数据库信息进行远程连接，验证连接可用性
3. **初始化迁移**: 生成迁移ID，并将迁移ID、迁移文件列表写入缓存
4. **请求回馈**: 返回迁移ID至核心模块
5. **数据迁移**: 存储模块先进行数据库迁移，并在迁移的同时修正为当前存储模块信息
6. **文件迁移**: 根据请求信息，存储模块进行文件迁移，文件迁移完成，会自动修改缓存中迁移文件列表对应文件状态
7. **文件转码**: 根据请求信息，若当前为远端到本地，且配置了转码请求，则进行文件转码，若为同步转码，则在文件迁移完成的同时，根据核心模块附件转码设置中的配置，同步提交此文件至队列排队，若为迁移后转码，则等待所有文件迁移完成后，再将需要转码的文件一次性提交至队列排队
8. **转码完成**: 文件转码完成后，自动对数据库中的文件信息进行修改，修改为新的文件信息
9. **迁移完成**: 等待所有文件转码结束，迁移正式完成
10. **完成通知**: 发送迁移成功信息至核心模块，结束本次文件迁移
11. **进度查询**: 核心模块可通过迁移ID对本次迁移进度进行查询

**接口需求**:

- `POST /api/v1/storage/migration/init` - 初始化文件迁移（gRPC）
- `GET /api/v1/storage/migration/{migration_id}/progress` - 查询迁移进度（gRPC）
- `POST /api/v1/storage/migration/{migration_id}/cancel` - 取消迁移任务（gRPC）

**数据验证**:

- 迁移类型：枚举类型，必填，可选值：remote_to_local（远端到本地）、local_to_remote（本地到远端）
- 远端连接信息：JSON 对象类型，必填，包含连接地址、端口、协议等
- 远端数据库信息：JSON 对象类型，必填，包含数据库类型、连接字符串、认证信息等
- 迁移文件列表：JSON 数组类型，必填，包含文件ID列表
- 是否转码：布尔类型，可选，仅远端到本地有效
- 转码方式：枚举类型，可选，可选值：sync（同步转码）、async（迁移后转码）

**业务规则**:

- 文件迁移仅通过 gRPC 接口提供服务
- 支持批量文件迁移，迁移过程中可查询进度
- 支持迁移任务取消
- 转码方式支持同步和异步两种模式

### 2. 文件预览

**功能描述**: 通过 API 调用存储模块中的文件进行在线预览。

**业务流程**:

1. **接收请求**: 核心模块通过 gRPC 获取存储模块文件预览链接
2. **预览初始化**: 存储模块根据配置的 API 地址，自动生成预览链接、时效 Token，完成链接组合，并将时效 Token 写入缓存
3. **返回信息**: 存储模块将预览链接、文件名称、文件类型返回核心模块，核心模块返回前端提供给前端使用
4. **预览访问**: 前端通过预览链接访问文件，存储模块接收到预览请求，获取时效 Token，并进行缓存查询，若 Token 时效则返回错误，否则返回文件信息
5. **防文件下载**: 前端访问预览链接时，存储模块进行安全验证，避免文件被直接下载

**接口需求**:

- `POST /api/v1/storage/preview/generate` - 生成预览链接（gRPC）
- `GET /api/v1/storage/preview/{token}` - 访问预览链接（HTTP）

**数据验证**:

- 文件ID：整数类型，必填，有效的文件ID
- Token 有效期：整数类型，可选，单位秒，默认 3600 秒（1小时），范围 60-86400 秒

**业务规则**:

- 预览链接具有时效性，默认有效期 1 小时
- 预览链接通过 Token 进行安全验证
- 预览访问时进行安全验证，防止直接下载文件
- 支持多种文件格式预览（图片、视频、音频、文档等）

### 3. 文件管理

**功能描述**: 通过核心模块对存储模块中的文件进行管理。

#### 3.1 文件分页列表

**功能描述**: 获取文件分页列表，支持分页、排序、筛选等功能。

**接口需求**:

- `GET /api/v1/storage/files` - 获取文件分页列表

**查询参数**:

- `page`: 整数类型，可选，页码，默认 1，最小值 1
- `limit`: 整数类型，可选，每页数量，默认 20，范围 1-100
- `keyword`: 字符串类型，可选，搜索关键字，支持文件名搜索
- `file_type`: 字符串类型，可选，文件类型筛选（image、video、audio、document 等）
- `storage_type`: 字符串类型，可选，存储类型筛选
- `start_date`: 日期类型，可选，开始日期，格式 YYYY-MM-DD
- `end_date`: 日期类型，可选，结束日期，格式 YYYY-MM-DD
- `sort_by`: 字符串类型，可选，排序字段，默认 created_at，可选值：created_at、file_size、file_name
- `sort_order`: 字符串类型，可选，排序顺序，默认 desc，可选值：asc、desc

**响应格式**:

- 使用 Envelope 响应格式
- 包含分页信息（meta.pagination）
- 包含链接信息（links）

#### 3.2 文件查询

**功能描述**: 根据文件ID或哈希值查询文件详细信息。

**接口需求**:

- `GET /api/v1/storage/files/{file_id}` - 根据文件ID查询文件详情
- `GET /api/v1/storage/files/hash/{hash}` - 根据哈希值查询文件详情

**响应格式**:

- 使用 Envelope 响应格式
- 返回文件详细信息，包括文件ID、文件名、文件大小、文件类型、存储路径、哈希值、创建时间等

#### 3.3 文件筛选

**功能描述**: 支持多种条件组合筛选文件。

**筛选条件**:

- 文件类型：图片、视频、音频、文档等
- 存储类型：本地存储、存储模块存储、云存储等
- 文件大小范围：最小大小、最大大小
- 创建时间范围：开始日期、结束日期
- 文件状态：正常、转码中、转码失败等

**接口需求**:

- 文件筛选功能集成在文件分页列表接口中，通过查询参数实现

#### 3.4 文件删除

**功能描述**: 删除存储模块中的文件，包括数据库记录和物理文件。

**接口需求**:

- `DELETE /api/v1/storage/files/{file_id}` - 删除文件（需要管理员权限）

**业务规则**:

- 删除文件前需要检查文件是否被其他资源引用
- 如果文件被引用，需要先解除引用关系才能删除
- 删除文件时同时删除数据库记录和物理文件
- 支持批量删除（通过文件ID列表）

**数据验证**:

- 文件ID：整数类型，必填，有效的文件ID
- 批量删除：JSON 数组类型，包含文件ID列表

### 4. 统计数据

**功能描述**: 通过核心模块对存储模块中的文件进行统计分析。

#### 4.1 文件统计

**功能描述**: 统计文件总数、文件类型分布、存储空间使用情况等。

**接口需求**:

- `GET /api/v1/storage/statistics/files` - 获取文件统计信息

**统计指标**:

- 文件总数：所有文件的总数量
- 文件类型分布：按文件类型统计文件数量和占用空间
- 存储空间使用：总存储空间、已使用空间、可用空间
- 文件大小分布：按文件大小区间统计文件数量

**响应格式**:

- 使用 Envelope 响应格式
- 返回统计数据的 JSON 对象

#### 4.2 文件分析

**功能描述**: 分析文件使用情况、文件增长趋势等。

**接口需求**:

- `GET /api/v1/storage/statistics/analysis` - 获取文件分析数据

**分析维度**:

- 文件增长趋势：按时间维度统计文件增长情况
- 文件使用频率：统计文件访问次数
- 热门文件：按访问次数排序的热门文件列表
- 文件类型趋势：按时间维度统计各类型文件增长趋势

#### 4.3 预览分析

**功能描述**: 统计文件预览次数、预览类型分布等。

**接口需求**:

- `GET /api/v1/storage/statistics/preview` - 获取预览统计信息

**统计指标**:

- 预览总次数：所有文件的预览总次数
- 预览类型分布：按文件类型统计预览次数
- 预览时间分布：按时间维度统计预览次数
- 热门预览文件：按预览次数排序的热门文件列表

#### 4.4 空间分析

**功能描述**: 分析存储空间使用情况、空间增长趋势等。

**接口需求**:

- `GET /api/v1/storage/statistics/space` - 获取空间分析数据

**分析维度**:

- 存储空间使用率：已使用空间占总空间的比例
- 空间增长趋势：按时间维度统计空间增长情况
- 存储类型分布：按存储类型统计空间使用情况
- 空间预警：当空间使用率超过阈值时进行预警

### 5. 健康检查

**功能描述**: 提供核心模块健康检查接口。

#### 5.1 运行状态

**功能描述**: 检查存储模块的运行状态。

**接口需求**:

- `GET /api/v1/storage/health/status` - 获取运行状态

**检查项**:

- 服务状态：运行中、停止、异常
- 数据库连接状态：正常、异常
- Redis 连接状态：正常、异常
- 存储空间状态：正常、不足、异常

**响应格式**:

- 使用 Envelope 响应格式
- 返回各检查项的状态信息

#### 5.2 存储状况

**功能描述**: 检查存储模块的存储状况。

**接口需求**:

- `GET /api/v1/storage/health/storage` - 获取存储状况

**检查项**:

- 存储空间使用率：已使用空间占总空间的比例
- 存储路径可写性：检查存储路径是否可写
- 存储类型可用性：检查各存储类型是否可用
- 存储性能：检查存储读写性能

#### 5.3 连接数值

**功能描述**: 统计存储模块的连接数值。

**接口需求**:

- `GET /api/v1/storage/health/connections` - 获取连接数值

**统计项**:

- 当前连接数：当前活跃的连接数量
- 最大连接数：系统支持的最大连接数
- 连接数趋势：按时间维度统计连接数变化
- 连接类型分布：按连接类型统计连接数

## 非功能需求

### 性能需求

- **响应时间**: API 接口响应时间应控制在 500ms 以内（正常情况）
- **并发处理**: 支持至少 1000 并发请求
- **文件上传**: 支持单文件最大 10GB 上传
- **分片上传**: 分片大小支持 2MB-10MB 动态调整
- **转码性能**: 转码任务处理速度应满足业务需求，支持队列并发处理

### 可靠性需求

- **可用性**: 系统可用性应达到 99.9% 以上
- **数据一致性**: 确保文件存储和数据库记录的一致性
- **错误恢复**: 支持上传失败重试、转码失败重试等错误恢复机制
- **数据备份**: 支持定期数据备份和恢复

### 可扩展性需求

- **存储扩展**: 支持多种存储类型（本地存储、云存储、存储模块存储）
- **转码扩展**: 支持多种文件格式转码，易于扩展新的转码格式
- **队列扩展**: 支持 Redis 队列扩展，支持多队列并发处理

### 可维护性需求

- **日志记录**: 记录所有关键操作日志，便于问题排查
- **监控告警**: 支持系统监控和告警机制
- **文档完善**: 提供完善的 API 文档和使用文档

## 接口需求

### RESTful API 接口

所有 RESTful API 接口遵循以下规范：

- **API 版本**: 使用 URL 路径进行版本控制，格式为 `/api/v1/...`
- **响应格式**: 所有响应统一使用 Envelope 格式，包含 meta、data、errors 三个部分
- **错误处理**: 所有错误信息使用中文，包含错误代码和错误消息
- **分页支持**: 列表接口支持分页，分页信息包含在 meta.pagination 中
- **认证授权**: 需要认证的接口使用 Bearer Token 认证

### gRPC 接口

所有 gRPC 接口遵循以下规范：

- **服务定义**: 使用 Protocol Buffers 定义服务接口
- **双向通信**: 支持客户端和服务器双向通信
- **错误处理**: 使用 gRPC 标准错误码和错误消息
- **流式传输**: 支持流式传输，用于大文件上传和下载

### 接口列表

#### RESTful API 接口

**文件上传接口**:

- `POST /api/v1/storage/upload/init` - 初始化文件上传
- `POST /api/v1/storage/upload/small` - 小文件上传
- `POST /api/v1/storage/upload/chunk` - 分片上传
- `POST /api/v1/storage/upload/merge` - 分片合并
- `GET /api/v1/storage/upload/{upload_id}/progress` - 查询上传/转码进度

**文件管理接口**:

- `GET /api/v1/storage/files` - 获取文件分页列表
- `GET /api/v1/storage/files/{file_id}` - 根据文件ID查询文件详情
- `GET /api/v1/storage/files/hash/{hash}` - 根据哈希值查询文件详情
- `DELETE /api/v1/storage/files/{file_id}` - 删除文件

**文件预览接口**:

- `GET /api/v1/storage/preview/{token}` - 访问预览链接

**统计接口**:

- `GET /api/v1/storage/statistics/files` - 获取文件统计信息
- `GET /api/v1/storage/statistics/analysis` - 获取文件分析数据
- `GET /api/v1/storage/statistics/preview` - 获取预览统计信息
- `GET /api/v1/storage/statistics/space` - 获取空间分析数据

**健康检查接口**:

- `GET /api/v1/storage/health/status` - 获取运行状态
- `GET /api/v1/storage/health/storage` - 获取存储状况
- `GET /api/v1/storage/health/connections` - 获取连接数值

#### gRPC 接口

**远程存储接口**:

- `rpc InitRemoteStorage(InitRemoteStorageRequest) returns (InitRemoteStorageResponse)` - 初始化远程存储
- `rpc UploadRemoteFile(UploadRemoteFileRequest) returns (UploadRemoteFileResponse)` - 远程文件上传
- `rpc CompleteRemoteStorage(CompleteRemoteStorageRequest) returns (CompleteRemoteStorageResponse)` - 完成远程存储

**文件迁移接口**:

- `rpc InitMigration(InitMigrationRequest) returns (InitMigrationResponse)` - 初始化文件迁移
- `rpc GetMigrationProgress(GetMigrationProgressRequest) returns (GetMigrationProgressResponse)` - 查询迁移进度
- `rpc CancelMigration(CancelMigrationRequest) returns (CancelMigrationResponse)` - 取消迁移任务

**文件预览接口**:

- `rpc GeneratePreviewLink(GeneratePreviewLinkRequest) returns (GeneratePreviewLinkResponse)` - 生成预览链接

## 数据需求

### 数据库设计

#### 文件表 (files)

**表名**: `files`

**字段定义**:

- `id`: BIGSERIAL PRIMARY KEY - 文件ID，自增主键
- `file_name`: VARCHAR(255) NOT NULL - 文件名，长度限制 255 字符
- `file_size`: BIGINT NOT NULL - 文件大小，单位字节
- `file_type`: VARCHAR(50) NOT NULL - 文件类型（image、video、audio、document 等）
- `mime_type`: VARCHAR(100) NOT NULL - MIME 类型
- `file_hash`: VARCHAR(64) NOT NULL UNIQUE - 文件哈希值，SHA256 格式，64 字符，唯一索引
- `storage_type`: VARCHAR(50) NOT NULL - 存储类型（local、storage_module、cloud）
- `storage_path`: TEXT NOT NULL - 存储路径
- `storage_url`: TEXT - 存储URL（云存储时使用）
- `thumbnail_path`: TEXT - 缩略图路径（图片、视频时使用）
- `thumbnail_url`: TEXT - 缩略图URL（云存储时使用）
- `width`: INTEGER - 宽度（图片、视频时使用）
- `height`: INTEGER - 高度（图片、视频时使用）
- `duration`: INTEGER - 时长（视频、音频时使用），单位秒
- `transcode_status`: VARCHAR(20) DEFAULT 'pending' - 转码状态（pending、processing、completed、failed）
- `transcode_info`: JSONB - 转码信息（JSON 格式，包含转码参数、转码结果等）
- `created_at`: TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP - 创建时间
- `updated_at`: TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP - 更新时间

**索引**:

- 主键索引：`pk_files` (id)
- 唯一索引：`uk_files_hash` (file_hash)
- 普通索引：`idx_files_file_type` (file_type)
- 普通索引：`idx_files_storage_type` (storage_type)
- 普通索引：`idx_files_created_at` (created_at)
- 普通索引：`idx_files_transcode_status` (transcode_status)
- 复合索引：`idx_files_type_created` (file_type, created_at)

**表注释**:

```sql
COMMENT ON TABLE files IS '文件表，存储文件的基本信息和元数据';
COMMENT ON COLUMN files.id IS '文件ID，自增主键';
COMMENT ON COLUMN files.file_name IS '文件名，长度限制255字符';
COMMENT ON COLUMN files.file_size IS '文件大小，单位字节';
COMMENT ON COLUMN files.file_type IS '文件类型：image-图片、video-视频、audio-音频、document-文档等';
COMMENT ON COLUMN files.mime_type IS 'MIME类型，如image/jpeg、video/mp4等';
COMMENT ON COLUMN files.file_hash IS '文件哈希值，SHA256格式，64字符，用于文件去重';
COMMENT ON COLUMN files.storage_type IS '存储类型：local-本地存储、storage_module-存储模块存储、cloud-云存储';
COMMENT ON COLUMN files.storage_path IS '存储路径，本地存储时使用';
COMMENT ON COLUMN files.storage_url IS '存储URL，云存储时使用';
COMMENT ON COLUMN files.thumbnail_path IS '缩略图路径，图片、视频时使用';
COMMENT ON COLUMN files.thumbnail_url IS '缩略图URL，云存储时使用';
COMMENT ON COLUMN files.width IS '宽度，图片、视频时使用，单位像素';
COMMENT ON COLUMN files.height IS '高度，图片、视频时使用，单位像素';
COMMENT ON COLUMN files.duration IS '时长，视频、音频时使用，单位秒';
COMMENT ON COLUMN files.transcode_status IS '转码状态：pending-待处理、processing-处理中、completed-已完成、failed-失败';
COMMENT ON COLUMN files.transcode_info IS '转码信息，JSON格式，包含转码参数、转码结果等';
COMMENT ON COLUMN files.created_at IS '创建时间，自动设置为当前时间';
COMMENT ON COLUMN files.updated_at IS '更新时间，每次更新时自动更新';
```

#### 上传记录表 (upload_records)

**表名**: `upload_records`

**字段定义**:

- `id`: BIGSERIAL PRIMARY KEY - 上传记录ID，自增主键
- `upload_id`: VARCHAR(64) NOT NULL UNIQUE - 上传ID，唯一标识一次上传任务
- `file_name`: VARCHAR(255) NOT NULL - 文件名
- `file_size`: BIGINT NOT NULL - 文件大小，单位字节
- `file_hash`: VARCHAR(64) NOT NULL - 文件哈希值，SHA256 格式
- `upload_type`: VARCHAR(20) NOT NULL - 上传类型（small、chunk、remote）
- `upload_status`: VARCHAR(20) DEFAULT 'pending' - 上传状态（pending、uploading、completed、failed）
- `chunk_count`: INTEGER DEFAULT 0 - 分片数量（分片上传时使用）
- `chunk_uploaded`: INTEGER DEFAULT 0 - 已上传分片数量（分片上传时使用）
- `progress`: INTEGER DEFAULT 0 - 上传进度，百分比 0-100
- `file_id`: BIGINT - 文件ID，上传完成后关联到 files 表
- `error_message`: TEXT - 错误消息（上传失败时使用）
- `created_at`: TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP - 创建时间
- `updated_at`: TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP - 更新时间
- `expires_at`: TIMESTAMPTZ - 过期时间，上传记录过期后自动清理

**索引**:

- 主键索引：`pk_upload_records` (id)
- 唯一索引：`uk_upload_records_upload_id` (upload_id)
- 普通索引：`idx_upload_records_file_hash` (file_hash)
- 普通索引：`idx_upload_records_upload_status` (upload_status)
- 普通索引：`idx_upload_records_created_at` (created_at)
- 普通索引：`idx_upload_records_expires_at` (expires_at)
- 外键索引：`idx_upload_records_file_id` (file_id)

**表注释**:

```sql
COMMENT ON TABLE upload_records IS '上传记录表，记录文件上传任务信息';
COMMENT ON COLUMN upload_records.id IS '上传记录ID，自增主键';
COMMENT ON COLUMN upload_records.upload_id IS '上传ID，唯一标识一次上传任务';
COMMENT ON COLUMN upload_records.file_name IS '文件名';
COMMENT ON COLUMN upload_records.file_size IS '文件大小，单位字节';
COMMENT ON COLUMN upload_records.file_hash IS '文件哈希值，SHA256格式，用于文件去重';
COMMENT ON COLUMN upload_records.upload_type IS '上传类型：small-小文件上传、chunk-分片上传、remote-远程上传';
COMMENT ON COLUMN upload_records.upload_status IS '上传状态：pending-待上传、uploading-上传中、completed-已完成、failed-失败';
COMMENT ON COLUMN upload_records.chunk_count IS '分片数量，分片上传时使用';
COMMENT ON COLUMN upload_records.chunk_uploaded IS '已上传分片数量，分片上传时使用';
COMMENT ON COLUMN upload_records.progress IS '上传进度，百分比0-100';
COMMENT ON COLUMN upload_records.file_id IS '文件ID，上传完成后关联到files表';
COMMENT ON COLUMN upload_records.error_message IS '错误消息，上传失败时使用';
COMMENT ON COLUMN upload_records.created_at IS '创建时间，自动设置为当前时间';
COMMENT ON COLUMN upload_records.updated_at IS '更新时间，每次更新时自动更新';
COMMENT ON COLUMN upload_records.expires_at IS '过期时间，上传记录过期后自动清理';
```

#### 转码记录表 (transcode_records)

**表名**: `transcode_records`

**字段定义**:

- `id`: BIGSERIAL PRIMARY KEY - 转码记录ID，自增主键
- `file_id`: BIGINT NOT NULL - 文件ID，外键关联 files 表
- `transcode_type`: VARCHAR(50) NOT NULL - 转码类型（image、video、audio 等）
- `transcode_status`: VARCHAR(20) DEFAULT 'pending' - 转码状态（pending、processing、completed、failed）
- `progress`: INTEGER DEFAULT 0 - 转码进度，百分比 0-100
- `source_file_path`: TEXT NOT NULL - 源文件路径
- `target_file_path`: TEXT - 目标文件路径（转码完成后）
- `transcode_params`: JSONB - 转码参数（JSON 格式，包含转码配置）
- `transcode_result`: JSONB - 转码结果（JSON 格式，包含转码后的文件信息）
- `error_message`: TEXT - 错误消息（转码失败时使用）
- `started_at`: TIMESTAMPTZ - 开始时间
- `completed_at`: TIMESTAMPTZ - 完成时间
- `created_at`: TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP - 创建时间
- `updated_at`: TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP - 更新时间

**索引**:

- 主键索引：`pk_transcode_records` (id)
- 外键索引：`idx_transcode_records_file_id` (file_id)
- 普通索引：`idx_transcode_records_transcode_status` (transcode_status)
- 普通索引：`idx_transcode_records_created_at` (created_at)
- 复合索引：`idx_transcode_records_file_status` (file_id, transcode_status)

**外键约束**:

- `fk_transcode_records_file_id` FOREIGN KEY (file_id) REFERENCES files(id) ON DELETE CASCADE

**表注释**:

```sql
COMMENT ON TABLE transcode_records IS '转码记录表，记录文件转码任务信息';
COMMENT ON COLUMN transcode_records.id IS '转码记录ID，自增主键';
COMMENT ON COLUMN transcode_records.file_id IS '文件ID，外键关联files表';
COMMENT ON COLUMN transcode_records.transcode_type IS '转码类型：image-图片转码、video-视频转码、audio-音频转码等';
COMMENT ON COLUMN transcode_records.transcode_status IS '转码状态：pending-待处理、processing-处理中、completed-已完成、failed-失败';
COMMENT ON COLUMN transcode_records.progress IS '转码进度，百分比0-100';
COMMENT ON COLUMN transcode_records.source_file_path IS '源文件路径';
COMMENT ON COLUMN transcode_records.target_file_path IS '目标文件路径，转码完成后使用';
COMMENT ON COLUMN transcode_records.transcode_params IS '转码参数，JSON格式，包含转码配置';
COMMENT ON COLUMN transcode_records.transcode_result IS '转码结果，JSON格式，包含转码后的文件信息';
COMMENT ON COLUMN transcode_records.error_message IS '错误消息，转码失败时使用';
COMMENT ON COLUMN transcode_records.started_at IS '开始时间';
COMMENT ON COLUMN transcode_records.completed_at IS '完成时间';
COMMENT ON COLUMN transcode_records.created_at IS '创建时间，自动设置为当前时间';
COMMENT ON COLUMN transcode_records.updated_at IS '更新时间，每次更新时自动更新';
```

#### 迁移记录表 (migration_records)

**表名**: `migration_records`

**字段定义**:

- `id`: BIGSERIAL PRIMARY KEY - 迁移记录ID，自增主键
- `migration_id`: VARCHAR(64) NOT NULL UNIQUE - 迁移ID，唯一标识一次迁移任务
- `migration_type`: VARCHAR(20) NOT NULL - 迁移类型（remote_to_local、local_to_remote）
- `remote_config`: JSONB NOT NULL - 远端配置（JSON 格式，包含连接信息、数据库信息等）
- `file_list`: JSONB NOT NULL - 迁移文件列表（JSON 数组格式，包含文件ID列表）
- `total_files`: INTEGER NOT NULL DEFAULT 0 - 总文件数
- `migrated_files`: INTEGER NOT NULL DEFAULT 0 - 已迁移文件数
- `failed_files`: INTEGER NOT NULL DEFAULT 0 - 失败文件数
- `migration_status`: VARCHAR(20) DEFAULT 'pending' - 迁移状态（pending、migrating、completed、failed、cancelled）
- `progress`: INTEGER DEFAULT 0 - 迁移进度，百分比 0-100
- `transcode_enabled`: BOOLEAN DEFAULT FALSE - 是否启用转码（仅远端到本地有效）
- `transcode_mode`: VARCHAR(20) - 转码方式（sync、async）
- `error_message`: TEXT - 错误消息（迁移失败时使用）
- `started_at`: TIMESTAMPTZ - 开始时间
- `completed_at`: TIMESTAMPTZ - 完成时间
- `created_at`: TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP - 创建时间
- `updated_at`: TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP - 更新时间

**索引**:

- 主键索引：`pk_migration_records` (id)
- 唯一索引：`uk_migration_records_migration_id` (migration_id)
- 普通索引：`idx_migration_records_migration_status` (migration_status)
- 普通索引：`idx_migration_records_created_at` (created_at)

**表注释**:

```sql
COMMENT ON TABLE migration_records IS '迁移记录表，记录文件迁移任务信息';
COMMENT ON COLUMN migration_records.id IS '迁移记录ID，自增主键';
COMMENT ON COLUMN migration_records.migration_id IS '迁移ID，唯一标识一次迁移任务';
COMMENT ON COLUMN migration_records.migration_type IS '迁移类型：remote_to_local-远端到本地、local_to_remote-本地到远端';
COMMENT ON COLUMN migration_records.remote_config IS '远端配置，JSON格式，包含连接信息、数据库信息等';
COMMENT ON COLUMN migration_records.file_list IS '迁移文件列表，JSON数组格式，包含文件ID列表';
COMMENT ON COLUMN migration_records.total_files IS '总文件数';
COMMENT ON COLUMN migration_records.migrated_files IS '已迁移文件数';
COMMENT ON COLUMN migration_records.failed_files IS '失败文件数';
COMMENT ON COLUMN migration_records.migration_status IS '迁移状态：pending-待处理、migrating-迁移中、completed-已完成、failed-失败、cancelled-已取消';
COMMENT ON COLUMN migration_records.progress IS '迁移进度，百分比0-100';
COMMENT ON COLUMN migration_records.transcode_enabled IS '是否启用转码，仅远端到本地有效';
COMMENT ON COLUMN migration_records.transcode_mode IS '转码方式：sync-同步转码、async-迁移后转码';
COMMENT ON COLUMN migration_records.error_message IS '错误消息，迁移失败时使用';
COMMENT ON COLUMN migration_records.started_at IS '开始时间';
COMMENT ON COLUMN migration_records.completed_at IS '完成时间';
COMMENT ON COLUMN migration_records.created_at IS '创建时间，自动设置为当前时间';
COMMENT ON COLUMN migration_records.updated_at IS '更新时间，每次更新时自动更新';
```

#### 预览记录表 (preview_records)

**表名**: `preview_records`

**字段定义**:

- `id`: BIGSERIAL PRIMARY KEY - 预览记录ID，自增主键
- `file_id`: BIGINT NOT NULL - 文件ID，外键关联 files 表
- `preview_token`: VARCHAR(64) NOT NULL UNIQUE - 预览Token，唯一标识一次预览请求
- `preview_url`: TEXT NOT NULL - 预览链接
- `expires_at`: TIMESTAMPTZ NOT NULL - 过期时间
- `access_count`: INTEGER DEFAULT 0 - 访问次数
- `last_accessed_at`: TIMESTAMPTZ - 最后访问时间
- `created_at`: TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP - 创建时间

**索引**:

- 主键索引：`pk_preview_records` (id)
- 唯一索引：`uk_preview_records_token` (preview_token)
- 外键索引：`idx_preview_records_file_id` (file_id)
- 普通索引：`idx_preview_records_expires_at` (expires_at)
- 普通索引：`idx_preview_records_created_at` (created_at)

**外键约束**:

- `fk_preview_records_file_id` FOREIGN KEY (file_id) REFERENCES files(id) ON DELETE CASCADE

**表注释**:

```sql
COMMENT ON TABLE preview_records IS '预览记录表，记录文件预览链接信息';
COMMENT ON COLUMN preview_records.id IS '预览记录ID，自增主键';
COMMENT ON COLUMN preview_records.file_id IS '文件ID，外键关联files表';
COMMENT ON COLUMN preview_records.preview_token IS '预览Token，唯一标识一次预览请求';
COMMENT ON COLUMN preview_records.preview_url IS '预览链接';
COMMENT ON COLUMN preview_records.expires_at IS '过期时间，Token过期后无法访问';
COMMENT ON COLUMN preview_records.access_count IS '访问次数';
COMMENT ON COLUMN preview_records.last_accessed_at IS '最后访问时间';
COMMENT ON COLUMN preview_records.created_at IS '创建时间，自动设置为当前时间';
```

### Redis 缓存设计

#### 上传缓存

**键名格式**: `upload:{upload_id}`

**数据结构**: Hash

**字段定义**:

- `upload_id`: 字符串类型，上传ID
- `file_name`: 字符串类型，文件名
- `file_size`: 整数类型，文件大小
- `file_hash`: 字符串类型，文件哈希值
- `upload_type`: 字符串类型，上传类型（small、chunk）
- `upload_status`: 字符串类型，上传状态
- `chunk_count`: 整数类型，分片数量
- `chunk_uploaded`: 整数类型，已上传分片数量
- `progress`: 整数类型，上传进度
- `created_at`: 字符串类型，创建时间
- `expires_at`: 字符串类型，过期时间

**过期时间**: 24 小时

#### 转码队列

**队列名称**: `transcode:queue`

**数据结构**: List

**队列元素**: JSON 格式，包含转码任务信息

**字段定义**:

- `file_id`: 整数类型，文件ID
- `transcode_type`: 字符串类型，转码类型
- `source_file_path`: 字符串类型，源文件路径
- `transcode_params`: JSON 对象类型，转码参数
- `priority`: 整数类型，优先级，数字越大优先级越高

#### 转码进度缓存

**键名格式**: `transcode:progress:{file_id}`

**数据结构**: Hash

**字段定义**:

- `file_id`: 整数类型，文件ID
- `transcode_status`: 字符串类型，转码状态
- `progress`: 整数类型，转码进度（0-100）
- `started_at`: 字符串类型，开始时间
- `updated_at`: 字符串类型，更新时间

**过期时间**: 7 天

#### 迁移缓存

**键名格式**: `migration:{migration_id}`

**数据结构**: Hash

**字段定义**:

- `migration_id`: 字符串类型，迁移ID
- `migration_type`: 字符串类型，迁移类型
- `total_files`: 整数类型，总文件数
- `migrated_files`: 整数类型，已迁移文件数
- `failed_files`: 整数类型，失败文件数
- `migration_status`: 字符串类型，迁移状态
- `progress`: 整数类型，迁移进度（0-100）
- `file_list`: JSON 数组类型，迁移文件列表（包含文件状态）
- `created_at`: 字符串类型，创建时间
- `updated_at`: 字符串类型，更新时间

**过期时间**: 30 天

#### 预览Token缓存

**键名格式**: `preview:token:{token}`

**数据结构**: Hash

**字段定义**:

- `token`: 字符串类型，预览Token
- `file_id`: 整数类型，文件ID
- `preview_url`: 字符串类型，预览链接
- `expires_at`: 字符串类型，过期时间
- `access_count`: 整数类型，访问次数
- `created_at`: 字符串类型，创建时间

**过期时间**: 根据Token有效期动态设置，默认 1 小时

## 安全需求

### 身份认证和授权

- **API 认证**: RESTful API 接口使用 Bearer Token 认证，Token 通过核心模块获取
- **gRPC 认证**: gRPC 接口使用 TLS 双向认证，确保模块间通信安全
- **权限控制**: 文件删除、文件管理等敏感操作需要管理员权限
- **Token 验证**: 预览Token 具有时效性，过期后自动失效

### 数据安全

- **文件哈希验证**: 所有上传文件必须进行哈希验证，确保文件完整性
- **文件去重**: 通过文件哈希值实现文件去重，相同文件仅存储一份
- **存储路径安全**: 存储路径不能包含路径遍历字符（../），防止路径遍历攻击
- **文件类型验证**: 验证文件类型和MIME类型，防止恶意文件上传

### 传输安全

- **HTTPS 传输**: 所有 API 接口必须通过 HTTPS 传输
- **TLS 加密**: gRPC 接口使用 TLS 加密，确保模块间通信安全
- **文件加密**: 敏感文件支持加密存储，加密密钥由核心模块管理

### 访问控制

- **预览链接安全**: 预览链接通过Token验证，防止未授权访问
- **文件下载控制**: 预览接口禁止直接下载文件，仅支持在线预览
- **IP 白名单**: 支持配置IP白名单，限制访问来源
- **访问频率限制**: 支持配置访问频率限制，防止恶意访问

### 日志和审计

- **操作日志**: 记录所有文件上传、删除、迁移等关键操作日志
- **访问日志**: 记录文件预览、下载等访问日志
- **错误日志**: 记录所有错误和异常信息，便于问题排查
- **审计日志**: 记录管理员操作日志，支持审计追溯

## 验收标准

### 功能验收

#### 文件上传功能

- ✅ 支持小文件上传（小于 10MB）
- ✅ 支持分片上传（大于等于 10MB）
- ✅ 支持文件去重，相同哈希值文件仅存储一份
- ✅ 支持上传进度查询
- ✅ 支持上传失败重试
- ✅ 支持文件哈希验证

#### 远程存储功能

- ✅ 支持采集模块远程文件存储
- ✅ 支持文件去重
- ✅ 支持转码处理
- ✅ 支持完成通知

#### 文件迁移功能

- ✅ 支持远端到本地迁移
- ✅ 支持本地到远端迁移
- ✅ 支持批量文件迁移
- ✅ 支持迁移进度查询
- ✅ 支持迁移任务取消
- ✅ 支持同步转码和异步转码

#### 文件预览功能

- ✅ 支持生成预览链接
- ✅ 支持Token验证
- ✅ 支持预览链接过期控制
- ✅ 支持多种文件格式预览
- ✅ 支持防文件下载

#### 文件管理功能

- ✅ 支持文件分页列表
- ✅ 支持文件查询（ID、哈希值）
- ✅ 支持文件筛选（类型、存储类型、时间范围等）
- ✅ 支持文件删除
- ✅ 支持批量删除

#### 统计功能

- ✅ 支持文件统计
- ✅ 支持文件分析
- ✅ 支持预览分析
- ✅ 支持空间分析

#### 健康检查功能

- ✅ 支持运行状态检查
- ✅ 支持存储状况检查
- ✅ 支持连接数值统计

### 性能验收

- ✅ API 接口响应时间控制在 500ms 以内
- ✅ 支持至少 1000 并发请求
- ✅ 支持单文件最大 10GB 上传
- ✅ 转码任务处理速度满足业务需求

### 安全验收

- ✅ 所有 API 接口通过 HTTPS 传输
- ✅ gRPC 接口使用 TLS 加密
- ✅ 文件上传进行哈希验证
- ✅ 预览链接通过Token验证
- ✅ 敏感操作需要权限验证
- ✅ 操作日志完整记录

### 可靠性验收

- ✅ 系统可用性达到 99.9% 以上
- ✅ 支持上传失败重试
- ✅ 支持转码失败重试
- ✅ 支持数据备份和恢复

### 文档验收

- ✅ API 文档完整，符合 OpenAPI 3.1.1 规范
- ✅ 接口文档包含请求示例和响应示例
- ✅ 错误码文档完整
- ✅ 使用文档完整

---

**文档版本**: 1.0  
**最后更新**: 2026-01-02 20:04:31 CST  
**编写依据**: [存储模块功能概述](../Features/Storage.md)

