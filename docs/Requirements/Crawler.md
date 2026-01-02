# 采集模块需求文档

## 文档说明

本文档为 LNCms 项目采集模块的详细需求文档，基于功能描述文档编写，遵循项目开发规范。

**文档版本**: 1.0  
**最后更新**: 2026-01-02 20:29:13 CST  
**编写依据**: [采集模块功能概述](../Features/Crawler.md)

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

采集模块（Crawler）为程序采集模块，负责提供远端信息采集、入库、远程文件下载等功能。采集模块支持规则管理、任务管理、进度管理、采集统计和健康检查等功能，为自动化内容采集提供支持。

### 技术栈

- **开发语言**: Golang 1.24+
- **开发框架**: Gin
- **数据库**: PostgreSQL
- **缓存**: Redis
- **API 规范**: OpenAPI 3.1.1

### 模块通信

#### gRPC 通信

- 对外提供 gRPC 接口服务
- 核心模块、存储模块、日志模块通过 gRPC 接口与采集模块进行通信
- 模块间使用 gRPC 均支持双向通信
- gRPC 接口用于模块间规则管理、任务管理、进度查询等操作

## 功能需求

### 1. 规则管理

规则管理功能提供采集规则的创建、编辑、删除等功能，支持规则定制、规则验证和规则确认等流程。

#### 1.1 规则创建

**功能描述**: 核心模块为采集模块创建采集规则，支持分步骤的规则定制流程。

**业务流程**:

1. **接收请求**: 采集模块接收核心模块提交的规则创建请求，请求包含规则唯一名称、网站名称、网站地址
2. **创建初始化**: 采集模块接收到创建请求，将请求信息写入缓存，并返回创建成功信息
3. **规则定制**: 核心模块接收到创建反馈，开始制定规则，每次制定规则，均会发起一次规则验证请求
4. **规则验证**: 采集模块接收到规则验证请求，根据规则信息发起远程访问请求、数据处理等，并将最终处理结果反馈给核心模块以供确认
5. **规则确认**: 核心模块接收到规则验证数据，确认规则无误，发送规则确认请求，采集模块将此条规则写入缓存
6. **规则完成**: 核心模块完成所有规则创建后，发送规则完成请求，采集模块接收到请求，将缓存中的采集规则信息写入数据库，并返回完成信息

**特殊说明**:

1. **流程记录**: 采集规则以核心模块发起的规则定制顺序为标准，后续采集模块进行数据采集时，以此顺序完成数据采集
2. **规则定制**: 规则定制可为三种类型（发起请求、数据处理、文件下载），发起请求为对制定 URL 发起访问请求，获取远端 HTML 数据，数据处理则是对上一次的请求结果进行数据处理，文件下载则是对远程文件进行下载
3. **标签处理**: 在规则定制中，允许使用标签处理，通过使用 `{$xxx}` 的格式，可对获取到的结果进行数据替换、过滤、使用等，标签需要先制定标签名称及所代表的数据（可为采集到的数据），后续在使用标签时，自动将标签替换为对应数据进行使用
4. **数据匹配**: 采集模块使用特殊的占位符形式对远程数据进行匹配、获取，占位符有（`{{*}}`、`{{**}}`、`{{!}}`、`{{!!}}`、`{{~}}`、`{{~~}}`、`{{^}}`、`{{^^}}`、`{{$}}`、`{{$$}}`）
   - `{{*}}` 表示匹配任意字符串
   - `{{**}}` 表示获取任意字符串
   - `{{!}}` 表示匹配除了<和>以外的任意字符串
   - `{{!!}}` 表示获取除了<和>以外的任意字符串
   - `{{~}}` 表示匹配除了<>'"以外的任意字符串
   - `{{~~}}` 表示获取除了<>'"以外的任意字符串
   - `{{^}}` 表示匹配除了数字和<>之外字符串
   - `{{^^}}` 表示获取除了数字和<>之外字符串
   - `{{$}}` 表示匹配数字字符串
   - `{{$$}}` 表示获取数字字符串

**接口需求**:

- `rpc CreateRule(CreateRuleRequest) returns (CreateRuleResponse)` - 创建规则初始化（gRPC）
- `rpc ValidateRule(ValidateRuleRequest) returns (ValidateRuleResponse)` - 验证规则（gRPC）
- `rpc ConfirmRule(ConfirmRuleRequest) returns (ConfirmRuleResponse)` - 确认规则（gRPC）
- `rpc CompleteRule(CompleteRuleRequest) returns (CompleteRuleResponse)` - 完成规则创建（gRPC）

**数据验证**:

- 规则名称：字符串类型，必填，长度限制 1-100 字符，唯一
- 网站名称：字符串类型，必填，长度限制 1-200 字符
- 网站地址：URL 类型，必填，格式验证
- 规则步骤：数组类型，必填，至少包含一个步骤
- 规则步骤类型：枚举类型，必填，可选值：request（发起请求）、process（数据处理）、download（文件下载）
- 规则步骤顺序：整数类型，必填，从 1 开始递增

**业务规则**:

- 规则创建采用分步骤流程，支持多次验证和确认
- 规则步骤按顺序执行，后续步骤可以使用前面步骤的结果
- 标签处理支持数据替换和过滤，提高规则灵活性
- 占位符支持多种匹配模式，适应不同的数据格式
- 规则验证失败时，返回详细错误信息，便于规则调整

#### 1.2 规则编辑

**功能描述**: 核心模块为采集模块修改采集规则。

**业务流程**: 采集规则修改流程与规则创建流程一致。

**接口需求**:

- `rpc UpdateRule(UpdateRuleRequest) returns (UpdateRuleResponse)` - 更新规则（gRPC）
- `rpc ValidateRule(ValidateRuleRequest) returns (ValidateRuleResponse)` - 验证规则（gRPC）
- `rpc ConfirmRule(ConfirmRuleRequest) returns (ConfirmRuleResponse)` - 确认规则（gRPC）
- `rpc CompleteRule(CompleteRuleRequest) returns (CompleteRuleResponse)` - 完成规则更新（gRPC）

**数据验证**: 与规则创建相同。

**业务规则**:

- 规则编辑时，需要先验证规则的有效性
- 规则更新后，需要重新验证所有步骤
- 正在使用的规则更新时，需要通知相关任务

#### 1.3 规则删除

**功能描述**: 核心模块为采集模块删除采集规则。

**接口需求**:

- `rpc DeleteRule(DeleteRuleRequest) returns (DeleteRuleResponse)` - 删除规则（gRPC）

**数据验证**:

- 规则ID：整数类型，必填，有效的规则ID

**业务规则**:

- 删除规则前，需要检查是否有任务正在使用该规则
- 如果有任务使用该规则，需要先停止或删除相关任务
- 删除规则时，同时删除规则相关的缓存数据

### 2. 任务管理

任务管理功能提供采集任务的创建、编辑、删除等功能，支持多种任务类型和采集目标。

#### 2.1 任务创建

**功能描述**: 核心模块为采集模块创建采集任务。

**任务配置项**:

- **任务名称** (task_name): 字符串类型，必填，长度限制 1-200 字符
- **采集规则** (rule_id): 整数类型，必填，引用已创建的采集规则
- **任务类型** (task_type): 枚举类型，必填，可选值：once（一次性任务）、loop（循环任务）、schedule（定时任务）
- **采集目标** (target_type): 枚举类型，必填，可选值：single_id（单一ID）、batch_id（批量ID）、range_id（范围ID）、page_list（页面列表）、pagination_list（分页列表）
- **采集目标配置** (target_config): JSON 对象类型，必填，根据采集目标类型不同而不同
- **数据处理** (data_process): 枚举类型，可选，可选值：skip（跳过）、clear_and_recollect（清空本地重新采集），默认 skip
- **文件处理** (file_process): 布尔类型，可选，是否将远端文件下载至本地，默认 false

**接口需求**:

- `rpc CreateTask(CreateTaskRequest) returns (CreateTaskResponse)` - 创建任务（gRPC）

**数据验证**:

- 任务名称：字符串类型，必填，长度限制 1-200 字符
- 采集规则：整数类型，必填，必须是有效的规则ID
- 任务类型：枚举类型，必填，必须是预定义的值之一
- 采集目标：枚举类型，必填，必须是预定义的值之一
- 采集目标配置：JSON 对象类型，必填，必须符合对应目标类型的要求

**业务规则**:

- 任务创建时，需要验证采集规则是否存在且有效
- 任务类型为循环任务时，需要配置循环间隔
- 任务类型为定时任务时，需要配置定时表达式（如 Cron 表达式）
- 采集目标配置根据目标类型不同而不同：
  - 单一ID：配置单个ID值
  - 批量ID：配置ID列表
  - 范围ID：配置起始ID和结束ID
  - 页面列表：配置列表URL和列表规则
  - 分页列表：配置分页URL模板和分页规则
- 文件处理开启时，下载的文件会根据核心模块中的附件配置，提交到对应的存储目的地

#### 2.2 任务编辑

**功能描述**: 核心模块为采集模块编辑采集任务。

**接口需求**:

- `rpc UpdateTask(UpdateTaskRequest) returns (UpdateTaskResponse)` - 更新任务（gRPC）

**数据验证**: 与任务创建相同。

**业务规则**:

- 任务编辑时，需要检查任务状态
- 正在执行的任务，部分字段不允许修改（如采集规则）
- 任务更新后，需要重新验证任务配置的有效性

#### 2.3 任务删除

**功能描述**: 核心模块为采集模块删除采集任务。

**接口需求**:

- `rpc DeleteTask(DeleteTaskRequest) returns (DeleteTaskResponse)` - 删除任务（gRPC）

**数据验证**:

- 任务ID：整数类型，必填，有效的任务ID

**业务规则**:

- 删除任务前，需要先停止正在执行的任务
- 删除任务时，可以选择是否删除已采集的数据
- 删除任务时，同时删除任务相关的缓存数据

### 3. 进度管理

进度管理功能提供核心模块对任务执行情况进行管理的功能，包括进度查询、进度执行、进度暂停、进度终止等操作。

#### 3.1 进度查询

**功能描述**: 查询任务的执行进度和状态信息。

**接口需求**:

- `rpc GetTaskProgress(GetTaskProgressRequest) returns (GetTaskProgressResponse)` - 获取任务进度（gRPC）

**查询信息**:

- **任务状态** (task_status): 枚举类型，可选值：pending（待执行）、running（执行中）、paused（已暂停）、completed（已完成）、failed（失败）、cancelled（已取消）
- **执行进度** (progress): 整数类型，百分比 0-100
- **已处理数量** (processed_count): 整数类型，已处理的记录数
- **总数量** (total_count): 整数类型，总记录数（如果可确定）
- **开始时间** (started_at): 时间戳类型，任务开始执行时间
- **预计完成时间** (estimated_completion_time): 时间戳类型，预计完成时间（可选）
- **错误信息** (error_message): 字符串类型，错误消息（任务失败时）

**数据验证**:

- 任务ID：整数类型，必填，有效的任务ID

**业务规则**:

- 进度信息实时更新，反映当前任务执行情况
- 进度查询支持缓存，减少数据库查询压力
- 任务失败时，返回详细错误信息

#### 3.2 进度执行

**功能描述**: 启动或恢复任务的执行。

**接口需求**:

- `rpc StartTask(StartTaskRequest) returns (StartTaskResponse)` - 启动任务（gRPC）
- `rpc ResumeTask(ResumeTaskRequest) returns (ResumeTaskResponse)` - 恢复任务（gRPC）

**数据验证**:

- 任务ID：整数类型，必填，有效的任务ID

**业务规则**:

- 启动任务时，需要验证任务配置的有效性
- 恢复任务时，从上次暂停的位置继续执行
- 任务执行采用异步方式，不阻塞调用方

#### 3.3 进度暂停

**功能描述**: 暂停正在执行的任务。

**接口需求**:

- `rpc PauseTask(PauseTaskRequest) returns (PauseTaskResponse)` - 暂停任务（gRPC）

**数据验证**:

- 任务ID：整数类型，必填，有效的任务ID

**业务规则**:

- 暂停任务时，保存当前执行状态
- 暂停的任务可以从暂停位置恢复执行
- 暂停操作应在安全位置执行，避免数据不一致

#### 3.4 进度终止

**功能描述**: 终止正在执行的任务。

**接口需求**:

- `rpc CancelTask(CancelTaskRequest) returns (CancelTaskResponse)` - 终止任务（gRPC）

**数据验证**:

- 任务ID：整数类型，必填，有效的任务ID

**业务规则**:

- 终止任务时，停止所有相关操作
- 终止的任务不能恢复，需要重新创建任务
- 终止操作应确保资源正确释放

### 4. 采集统计

采集统计功能提供核心模块进行采集统计的功能，包括采集统计和采集分析等功能。

#### 4.1 采集统计

**功能描述**: 统计采集任务的数量、成功率、数据量等信息。

**统计维度**:

- **总体统计**: 任务总数、执行中任务数、已完成任务数、失败任务数
- **规则统计**: 各规则的任务数量、成功率
- **时间统计**: 按时间维度统计任务数量和成功率
- **数据统计**: 采集的数据总量、文件下载总量等

**接口需求**:

- `rpc GetCollectionStatistics(GetCollectionStatisticsRequest) returns (GetCollectionStatisticsResponse)` - 获取采集统计（gRPC）

**统计参数**:

- **统计维度** (dimension): 枚举类型，必填，可选值：overall（总体）、rule（规则）、time（时间）、data（数据）
- **时间维度** (time_dimension): 枚举类型，可选，可选值：hour（小时）、day（天）、week（周）、month（月），仅在统计维度为 time 时有效
- **开始时间** (start_time): 时间戳类型，可选，统计开始时间
- **结束时间** (end_time): 时间戳类型，可选，统计结束时间
- **规则ID** (rule_id): 整数类型，可选，筛选指定规则的统计

**数据验证**:

- 统计维度：枚举类型，必填，必须是预定义的值之一
- 时间维度：枚举类型，可选，必须是预定义的值之一
- 开始时间：时间戳类型，可选，必须是有效的 ISO 8601 格式
- 结束时间：时间戳类型，可选，必须是有效的 ISO 8601 格式，且必须大于等于开始时间
- 规则ID：整数类型，可选，必须是有效的规则ID

**业务规则**:

- 统计结果包含数量、占比、趋势等信息
- 时间统计支持图表数据格式（时间序列数据）
- 统计查询应优化性能，使用聚合查询和缓存
- 大量数据统计时，应使用采样或预聚合数据

#### 4.2 采集分析

**功能描述**: 分析采集任务的执行情况、成功率、性能等。

**分析类型**:

- **执行分析**: 分析任务执行时间、成功率、失败原因等
- **性能分析**: 分析采集速度、资源使用情况等
- **数据质量分析**: 分析采集数据的完整性、准确性等
- **异常检测**: 检测异常任务、异常数据等

**接口需求**:

- `rpc AnalyzeCollection(AnalyzeCollectionRequest) returns (AnalyzeCollectionResponse)` - 分析采集数据（gRPC）

**分析参数**:

- **分析类型** (analysis_type): 枚举类型，必填，可选值：execution（执行分析）、performance（性能分析）、quality（数据质量分析）、anomaly（异常检测）
- **开始时间** (start_time): 时间戳类型，可选，分析开始时间
- **结束时间** (end_time): 时间戳类型，可选，分析结束时间
- **分析深度** (depth): 枚举类型，可选，可选值：basic（基础）、detailed（详细）、deep（深度），默认 basic
- **规则ID** (rule_id): 整数类型，可选，筛选指定规则的分析

**数据验证**:

- 分析类型：枚举类型，必填，必须是预定义的值之一
- 开始时间：时间戳类型，可选，必须是有效的 ISO 8601 格式
- 结束时间：时间戳类型，可选，必须是有效的 ISO 8601 格式，且必须大于等于开始时间
- 分析深度：枚举类型，可选，必须是预定义的值之一
- 规则ID：整数类型，可选，必须是有效的规则ID

**业务规则**:

- 分析操作可能耗时较长，应异步执行
- 分析结果包含分析报告、图表数据、建议等
- 深度分析应使用机器学习或统计分析算法
- 分析结果支持导出和分享
- 异常检测应支持告警功能

### 5. 健康检查

健康检查功能提供核心模块健康检查接口，支持运行状态和连接数值查询等功能。

#### 5.1 运行状态

**功能描述**: 提供采集模块的运行状态信息，包括服务状态、版本信息、启动时间等。

**状态信息**:

- **服务状态** (status): 枚举类型，可选值：running（运行中）、stopped（已停止）、error（错误）
- **版本信息** (version): 字符串类型，服务版本号
- **启动时间** (start_time): 时间戳类型，服务启动时间
- **运行时长** (uptime): 整数类型，服务运行时长（单位：秒）
- **最后心跳时间** (last_heartbeat): 时间戳类型，最后心跳时间

**接口需求**:

- `rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse)` - 健康检查（gRPC）

**业务规则**:

- 服务状态实时更新，反映当前服务运行情况
- 心跳机制定期更新最后心跳时间
- 服务异常时，状态应更新为 error

#### 5.2 连接数值

**功能描述**: 提供采集模块的连接数值信息，包括数据库连接数、Redis 连接数、gRPC 连接数等。

**连接信息**:

- **数据库连接数** (db_connections): 对象类型，包含：
  - 当前连接数 (current): 整数类型
  - 最大连接数 (max): 整数类型
  - 空闲连接数 (idle): 整数类型
- **Redis 连接数** (redis_connections): 对象类型，包含：
  - 当前连接数 (current): 整数类型
  - 最大连接数 (max): 整数类型
  - 空闲连接数 (idle): 整数类型
- **gRPC 连接数** (grpc_connections): 对象类型，包含：
  - 当前连接数 (current): 整数类型
  - 活跃连接数 (active): 整数类型

**业务规则**:

- 连接数值实时更新，反映当前连接使用情况
- 连接数超过阈值时，应记录告警信息
- 连接信息用于监控和性能优化

## 非功能需求

### 性能需求

- **规则验证性能**: 规则验证响应时间不超过 2s（正常情况）
- **任务执行性能**: 支持并发执行多个采集任务，单个任务执行速度应满足业务需求
- **进度查询性能**: 进度查询响应时间不超过 500ms（正常情况）
- **统计查询性能**: 统计查询响应时间不超过 2s（正常情况）
- **并发处理**: 支持至少 100 个并发采集任务

### 可用性需求

- **服务可用性**: 服务可用性达到 99.9% 以上
- **数据可靠性**: 采集数据不丢失，支持数据备份和恢复
- **故障恢复**: 服务故障后能在 5 分钟内自动恢复
- **任务容错**: 任务执行失败时，支持自动重试和错误恢复

### 可扩展性需求

- **水平扩展**: 支持多实例部署，通过负载均衡分发请求
- **规则扩展**: 支持插件化规则扩展，便于添加新的采集规则类型
- **功能扩展**: 支持插件化扩展，便于添加新的采集功能

### 可维护性需求

- **日志记录**: 采集模块自身操作应记录详细日志
- **监控告警**: 支持监控指标采集和告警功能
- **配置管理**: 支持配置热更新，无需重启服务
- **规则调试**: 提供规则调试工具，便于规则开发和调试

## 接口需求

### gRPC 接口

采集模块通过 gRPC 接口对外提供服务，所有接口遵循 gRPC 规范。

#### 规则管理接口

- **服务名**: `crawler.RuleService`
- **方法列表**:
  - `rpc CreateRule(CreateRuleRequest) returns (CreateRuleResponse)` - 创建规则初始化
  - `rpc ValidateRule(ValidateRuleRequest) returns (ValidateRuleResponse)` - 验证规则
  - `rpc ConfirmRule(ConfirmRuleRequest) returns (ConfirmRuleResponse)` - 确认规则
  - `rpc CompleteRule(CompleteRuleRequest) returns (CompleteRuleResponse)` - 完成规则创建
  - `rpc UpdateRule(UpdateRuleRequest) returns (UpdateRuleResponse)` - 更新规则
  - `rpc DeleteRule(DeleteRuleRequest) returns (DeleteRuleResponse)` - 删除规则
  - `rpc GetRule(GetRuleRequest) returns (GetRuleResponse)` - 获取规则详情
  - `rpc ListRules(ListRulesRequest) returns (ListRulesResponse)` - 获取规则列表

#### 任务管理接口

- **服务名**: `crawler.TaskService`
- **方法列表**:
  - `rpc CreateTask(CreateTaskRequest) returns (CreateTaskResponse)` - 创建任务
  - `rpc UpdateTask(UpdateTaskRequest) returns (UpdateTaskResponse)` - 更新任务
  - `rpc DeleteTask(DeleteTaskRequest) returns (DeleteTaskResponse)` - 删除任务
  - `rpc GetTask(GetTaskRequest) returns (GetTaskResponse)` - 获取任务详情
  - `rpc ListTasks(ListTasksRequest) returns (ListTasksResponse)` - 获取任务列表

#### 进度管理接口

- **服务名**: `crawler.ProgressService`
- **方法列表**:
  - `rpc GetTaskProgress(GetTaskProgressRequest) returns (GetTaskProgressResponse)` - 获取任务进度
  - `rpc StartTask(StartTaskRequest) returns (StartTaskResponse)` - 启动任务
  - `rpc ResumeTask(ResumeTaskRequest) returns (ResumeTaskResponse)` - 恢复任务
  - `rpc PauseTask(PauseTaskRequest) returns (PauseTaskResponse)` - 暂停任务
  - `rpc CancelTask(CancelTaskRequest) returns (CancelTaskResponse)` - 终止任务

#### 统计接口

- **服务名**: `crawler.StatisticsService`
- **方法列表**:
  - `rpc GetCollectionStatistics(GetCollectionStatisticsRequest) returns (GetCollectionStatisticsResponse)` - 获取采集统计
  - `rpc AnalyzeCollection(AnalyzeCollectionRequest) returns (AnalyzeCollectionResponse)` - 分析采集数据

#### 健康检查接口

- **服务名**: `crawler.HealthService`
- **方法列表**:
  - `rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse)` - 健康检查

### 接口规范

- **服务定义**: 使用 Protocol Buffers 定义 gRPC 服务
- **双向通信**: 支持客户端和服务器双向通信
- **错误处理**: 使用 gRPC 标准错误码和错误消息
- **流式传输**: 支持流式传输，用于大文件下载和进度推送

## 数据需求

### 数据库设计

#### 采集规则表 (crawler_rules)

**表名**: `crawler_rules`

**字段定义**:

- `id` (BIGSERIAL): 主键，自增
- `rule_name` (VARCHAR(100)): 规则名称，唯一索引
- `site_name` (VARCHAR(200)): 网站名称
- `site_url` (VARCHAR(500)): 网站地址
- `rule_steps` (JSONB): 规则步骤，JSON 数组格式
- `rule_order` (INTEGER): 规则执行顺序
- `is_active` (BOOLEAN): 是否启用，默认 true
- `created_at` (TIMESTAMPTZ): 创建时间，索引
- `updated_at` (TIMESTAMPTZ): 更新时间

**索引设计**:

- 主键索引: `pk_crawler_rules` (id)
- 唯一索引: `uk_crawler_rules_rule_name` (rule_name)
- 普通索引: `idx_crawler_rules_is_active` (is_active)
- 普通索引: `idx_crawler_rules_created_at` (created_at)
- GIN 索引: `idx_crawler_rules_rule_steps` (rule_steps) - 用于 JSONB 字段查询

**表注释**:

```sql
COMMENT ON TABLE crawler_rules IS '采集规则表，存储采集规则配置信息';
COMMENT ON COLUMN crawler_rules.id IS '规则ID，自增主键';
COMMENT ON COLUMN crawler_rules.rule_name IS '规则名称，唯一标识';
COMMENT ON COLUMN crawler_rules.site_name IS '网站名称';
COMMENT ON COLUMN crawler_rules.site_url IS '网站地址，URL格式';
COMMENT ON COLUMN crawler_rules.rule_steps IS '规则步骤，JSON数组格式，包含规则执行步骤';
COMMENT ON COLUMN crawler_rules.rule_order IS '规则执行顺序，数字越小越先执行';
COMMENT ON COLUMN crawler_rules.is_active IS '是否启用，true-启用，false-禁用';
COMMENT ON COLUMN crawler_rules.created_at IS '创建时间';
COMMENT ON COLUMN crawler_rules.updated_at IS '更新时间';
```

#### 采集任务表 (crawler_tasks)

**表名**: `crawler_tasks`

**字段定义**:

- `id` (BIGSERIAL): 主键，自增
- `task_name` (VARCHAR(200)): 任务名称
- `rule_id` (BIGINT): 规则ID，外键关联 crawler_rules 表
- `task_type` (VARCHAR(20)): 任务类型，索引
- `target_type` (VARCHAR(20)): 采集目标类型
- `target_config` (JSONB): 采集目标配置，JSON 对象格式
- `data_process` (VARCHAR(20)): 数据处理方式
- `file_process` (BOOLEAN): 是否处理文件，默认 false
- `task_status` (VARCHAR(20)): 任务状态，索引
- `progress` (INTEGER): 执行进度，百分比 0-100
- `processed_count` (BIGINT): 已处理数量
- `total_count` (BIGINT): 总数量（如果可确定）
- `started_at` (TIMESTAMPTZ): 开始时间
- `completed_at` (TIMESTAMPTZ): 完成时间
- `error_message` (TEXT): 错误消息
- `created_at` (TIMESTAMPTZ): 创建时间，索引
- `updated_at` (TIMESTAMPTZ): 更新时间

**索引设计**:

- 主键索引: `pk_crawler_tasks` (id)
- 外键索引: `idx_crawler_tasks_rule_id` (rule_id)
- 普通索引: `idx_crawler_tasks_task_type` (task_type)
- 普通索引: `idx_crawler_tasks_task_status` (task_status)
- 普通索引: `idx_crawler_tasks_created_at` (created_at)
- 复合索引: `idx_crawler_tasks_rule_status` (rule_id, task_status)

**外键约束**:

- `fk_crawler_tasks_rule_id` FOREIGN KEY (rule_id) REFERENCES crawler_rules(id) ON DELETE RESTRICT

**表注释**:

```sql
COMMENT ON TABLE crawler_tasks IS '采集任务表，存储采集任务信息';
COMMENT ON COLUMN crawler_tasks.id IS '任务ID，自增主键';
COMMENT ON COLUMN crawler_tasks.task_name IS '任务名称';
COMMENT ON COLUMN crawler_tasks.rule_id IS '规则ID，外键关联crawler_rules表';
COMMENT ON COLUMN crawler_tasks.task_type IS '任务类型：once-一次性任务、loop-循环任务、schedule-定时任务';
COMMENT ON COLUMN crawler_tasks.target_type IS '采集目标类型：single_id-单一ID、batch_id-批量ID、range_id-范围ID、page_list-页面列表、pagination_list-分页列表';
COMMENT ON COLUMN crawler_tasks.target_config IS '采集目标配置，JSON对象格式，根据目标类型不同而不同';
COMMENT ON COLUMN crawler_tasks.data_process IS '数据处理方式：skip-跳过、clear_and_recollect-清空本地重新采集';
COMMENT ON COLUMN crawler_tasks.file_process IS '是否处理文件，true-下载文件，false-不下载文件';
COMMENT ON COLUMN crawler_tasks.task_status IS '任务状态：pending-待执行、running-执行中、paused-已暂停、completed-已完成、failed-失败、cancelled-已取消';
COMMENT ON COLUMN crawler_tasks.progress IS '执行进度，百分比0-100';
COMMENT ON COLUMN crawler_tasks.processed_count IS '已处理数量';
COMMENT ON COLUMN crawler_tasks.total_count IS '总数量，如果可确定则设置';
COMMENT ON COLUMN crawler_tasks.started_at IS '开始时间';
COMMENT ON COLUMN crawler_tasks.completed_at IS '完成时间';
COMMENT ON COLUMN crawler_tasks.error_message IS '错误消息，任务失败时使用';
COMMENT ON COLUMN crawler_tasks.created_at IS '创建时间';
COMMENT ON COLUMN crawler_tasks.updated_at IS '更新时间';
```

#### 采集数据表 (crawler_data)

**表名**: `crawler_data`

**字段定义**:

- `id` (BIGSERIAL): 主键，自增
- `task_id` (BIGINT): 任务ID，外键关联 crawler_tasks 表
- `rule_id` (BIGINT): 规则ID，外键关联 crawler_rules 表
- `source_id` (VARCHAR(200)): 源数据ID，唯一索引
- `data_content` (JSONB): 采集的数据内容，JSON 对象格式
- `file_urls` (JSONB): 文件URL列表，JSON 数组格式（如果下载了文件）
- `collected_at` (TIMESTAMPTZ): 采集时间，索引
- `created_at` (TIMESTAMPTZ): 创建时间
- `updated_at` (TIMESTAMPTZ): 更新时间

**索引设计**:

- 主键索引: `pk_crawler_data` (id)
- 唯一索引: `uk_crawler_data_source_id` (source_id)
- 外键索引: `idx_crawler_data_task_id` (task_id)
- 外键索引: `idx_crawler_data_rule_id` (rule_id)
- 普通索引: `idx_crawler_data_collected_at` (collected_at)
- 复合索引: `idx_crawler_data_task_collected` (task_id, collected_at)
- GIN 索引: `idx_crawler_data_data_content` (data_content) - 用于 JSONB 字段查询

**外键约束**:

- `fk_crawler_data_task_id` FOREIGN KEY (task_id) REFERENCES crawler_tasks(id) ON DELETE CASCADE
- `fk_crawler_data_rule_id` FOREIGN KEY (rule_id) REFERENCES crawler_rules(id) ON DELETE RESTRICT

**表注释**:

```sql
COMMENT ON TABLE crawler_data IS '采集数据表，存储采集到的数据';
COMMENT ON COLUMN crawler_data.id IS '数据ID，自增主键';
COMMENT ON COLUMN crawler_data.task_id IS '任务ID，外键关联crawler_tasks表';
COMMENT ON COLUMN crawler_data.rule_id IS '规则ID，外键关联crawler_rules表';
COMMENT ON COLUMN crawler_data.source_id IS '源数据ID，唯一标识采集的数据来源';
COMMENT ON COLUMN crawler_data.data_content IS '采集的数据内容，JSON对象格式';
COMMENT ON COLUMN crawler_data.file_urls IS '文件URL列表，JSON数组格式，如果下载了文件则包含文件URL';
COMMENT ON COLUMN crawler_data.collected_at IS '采集时间';
COMMENT ON COLUMN crawler_data.created_at IS '创建时间';
COMMENT ON COLUMN crawler_data.updated_at IS '更新时间';
```

### 缓存设计

#### Redis 数据结构

**规则缓存**:

- **键名格式**: `crawler:rule:{rule_id}`
- **数据类型**: Hash
- **字段**: 与规则表字段对应
- **过期时间**: 24 小时

**任务缓存**:

- **键名格式**: `crawler:task:{task_id}`
- **数据类型**: Hash
- **字段**: 与任务表字段对应，包含进度信息
- **过期时间**: 任务完成后 7 天

**任务进度缓存**:

- **键名格式**: `crawler:progress:{task_id}`
- **数据类型**: Hash
- **字段**: 任务状态、进度、已处理数量等
- **过期时间**: 任务完成后 7 天

**规则验证缓存**:

- **键名格式**: `crawler:validate:{rule_id}:{step_index}`
- **数据类型**: String（JSON 格式）
- **内容**: 规则验证结果
- **过期时间**: 1 小时

## 安全需求

### 身份认证和授权

- **gRPC 认证**: gRPC 接口使用 TLS 加密传输，支持 mTLS 双向认证
- **请求验证**: 验证请求来源，防止未授权访问
- **权限控制**: 规则管理、任务管理等敏感操作需要管理员权限

### 数据安全

- **规则安全**: 采集规则配置应加密存储，防止规则泄露
- **数据验证**: 所有采集数据应进行验证，防止恶意数据注入
- **URL 验证**: 采集的 URL 应进行验证，防止 SSRF 攻击

### 传输安全

- **TLS 加密**: gRPC 接口使用 TLS 加密，确保模块间通信安全
- **数据加密**: 敏感数据支持加密传输
- **防重放攻击**: 支持请求签名和防重放机制

### 访问控制

- **IP 白名单**: 支持配置 IP 白名单，限制访问来源
- **访问频率限制**: 支持配置访问频率限制，防止恶意访问
- **任务限制**: 支持配置任务数量限制，防止资源滥用

### 日志和审计

- **操作日志**: 记录所有规则管理、任务管理等关键操作日志
- **执行日志**: 记录任务执行日志，便于问题排查
- **错误日志**: 记录所有错误和异常信息
- **审计日志**: 记录管理员操作日志，支持审计追溯

## 验收标准

### 功能验收

#### 规则管理功能

- ✅ 支持规则创建、编辑、删除
- ✅ 支持规则验证和确认流程
- ✅ 支持标签处理和占位符匹配
- ✅ 支持规则步骤顺序管理

#### 任务管理功能

- ✅ 支持任务创建、编辑、删除
- ✅ 支持多种任务类型（一次性、循环、定时）
- ✅ 支持多种采集目标类型
- ✅ 支持文件下载配置

#### 进度管理功能

- ✅ 支持进度查询
- ✅ 支持任务启动、暂停、恢复、终止
- ✅ 支持进度实时更新

#### 采集统计功能

- ✅ 支持采集统计查询
- ✅ 支持采集数据分析
- ✅ 支持多维度统计

#### 健康检查功能

- ✅ 支持运行状态查询
- ✅ 支持连接数值查询

### 性能验收

- ✅ 规则验证响应时间不超过 2s
- ✅ 进度查询响应时间不超过 500ms
- ✅ 统计查询响应时间不超过 2s
- ✅ 支持至少 100 个并发采集任务

### 安全验收

- ✅ gRPC 接口使用 TLS 加密
- ✅ 规则配置加密存储
- ✅ 访问控制功能正常
- ✅ 操作日志完整记录

### 可用性验收

- ✅ 服务可用性达到 99.9% 以上
- ✅ 故障恢复时间不超过 5 分钟
- ✅ 任务容错功能正常
- ✅ 数据备份和恢复功能正常

### 文档验收

- ✅ gRPC 接口文档完整
- ✅ 接口文档包含请求示例和响应示例
- ✅ 错误码文档完整
- ✅ 使用文档完整

---

**文档版本**: 1.0  
**最后更新**: 2026-01-02 20:29:13 CST  
**编写依据**: [采集模块功能概述](../Features/Crawler.md)

