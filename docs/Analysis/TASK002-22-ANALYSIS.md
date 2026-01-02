# TASK002-22 设计合理性分析报告

## 分析背景

用户要求分析小说章节附件表（novel_chapter_attachments）设计的合理性，重点关注存储模块可选性、文件唯一性原则、文件编号和文件URL的使用等问题。

## 当前设计概述

当前附件表设计：
- `file_id BIGINT NOT NULL` - 文件ID，外键关联 `files(id)`
- `files` 表位于存储模块中
- 外键约束：`fk_novel_chapter_attachments_file_id` 关联 `files(id)`，级联删除

## 关键问题分析

### 1. 存储模块可选性问题

**问题描述**：
- 存储模块是可选的，未部署时文件存储在核心模块本地或云存储
- 当前设计假设 `files` 表在存储模块中
- 如果存储模块未部署，`files` 表不存在，外键关联会失败

**影响**：
- 系统无法在未使用存储模块的情况下正常工作
- 违反了模块可选性的设计原则

### 2. 跨数据库外键关联问题

**问题描述**：
- 存储模块支持独立数据库，也支持与核心模块共用数据库
- 如果存储模块使用独立数据库，跨数据库的外键关联不可行（PostgreSQL不支持跨数据库外键）
- 即使在同一数据库，跨模块的表依赖也会增加系统耦合度

**影响**：
- 数据库架构灵活性受限
- 模块间耦合度增加
- 数据一致性难以保证

### 3. 文件唯一性原则问题

**问题描述**：
- 文件资源存在唯一性原则，会进行去重操作
- 当前设计依赖存储模块的 `files` 表实现唯一性
- 如果未使用存储模块，核心模块如何处理文件唯一性？

**影响**：
- 文件去重机制不统一
- 可能导致重复文件存储

### 4. 文件编号和文件URL的使用

**问题描述**：
- 用户提到"附件的存储、调用，应该是基于文件编号来使用，或者文件URL来使用"
- 当前设计只使用 `file_id`（文件编号）
- 没有考虑直接使用文件URL的情况

**影响**：
- 无法支持外部文件URL的直接使用
- 灵活性不足

## 设计合理性评估

### 当前设计的优点

1. **数据一致性**：通过外键约束保证文件存在性
2. **级联删除**：删除文件时自动删除关联的附件
3. **索引优化**：为 `file_id` 创建索引，查询效率高

### 当前设计的缺点

1. **模块耦合**：核心模块附件表直接依赖存储模块的表
2. **可选性缺失**：无法在未使用存储模块时正常工作
3. **灵活性不足**：无法支持外部文件URL
4. **架构问题**：跨数据库外键关联不可行

### 合理性结论

**当前设计不合理**，主要原因：
1. 违反了存储模块可选性的设计原则
2. 跨数据库外键关联不可行
3. 无法支持未使用存储模块的场景
4. 模块间耦合度过高

## 改进建议

### 方案一：核心模块统一文件管理（推荐）

**设计思路**：
- 核心模块应该有 `files` 表（无论是否使用存储模块）
- 附件表关联核心模块的 `files` 表
- 如果使用存储模块，通过 gRPC 同步文件信息到核心模块 `files` 表

**表结构设计**：
```sql
-- 核心模块files表（需要单独设计）
CREATE TABLE files (
    id BIGSERIAL PRIMARY KEY,
    file_name VARCHAR(255) NOT NULL,
    file_size BIGINT NOT NULL,
    file_type VARCHAR(50) NOT NULL,
    file_hash VARCHAR(64),
    storage_type VARCHAR(50) NOT NULL,
    storage_path VARCHAR(500),
    storage_url VARCHAR(500),
    -- ... 其他字段
);

-- 附件表关联核心模块files表
CREATE TABLE novel_chapter_attachments (
    id BIGSERIAL PRIMARY KEY,
    chapter_id BIGINT NOT NULL,
    file_id BIGINT NOT NULL,  -- 关联核心模块files表
    attachment_type VARCHAR(50) NOT NULL,
    attachment_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_novel_chapter_attachments_chapter_id 
        FOREIGN KEY (chapter_id) REFERENCES novel_chapters(id) ON DELETE CASCADE,
    CONSTRAINT fk_novel_chapter_attachments_file_id 
        FOREIGN KEY (file_id) REFERENCES files(id) ON DELETE CASCADE
);
```

**优点**：
- 支持存储模块可选性
- 统一文件管理机制
- 数据一致性好
- 文件唯一性统一处理
- 文件URL通过 `files.storage_url` 获取

**缺点**：
- 需要同步机制（如果使用存储模块）
- 核心模块需要维护文件信息

### 方案二：支持文件ID和文件URL（备选）

**设计思路**：
- 附件表同时支持 `file_id`（关联files表）和 `file_url`（直接存储URL）
- 使用存储模块时使用 `file_id`
- 未使用存储模块或外部文件时使用 `file_url`
- 约束：`file_id` 和 `file_url` 至少有一个不为NULL

**表结构设计**：
```sql
CREATE TABLE novel_chapter_attachments (
    id BIGSERIAL PRIMARY KEY,
    chapter_id BIGINT NOT NULL,
    file_id BIGINT,  -- 可为NULL，关联核心模块files表
    file_url VARCHAR(500),  -- 可为NULL，直接存储文件URL
    attachment_type VARCHAR(50) NOT NULL,
    attachment_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_novel_chapter_attachments_chapter_id 
        FOREIGN KEY (chapter_id) REFERENCES novel_chapters(id) ON DELETE CASCADE,
    CONSTRAINT fk_novel_chapter_attachments_file_id 
        FOREIGN KEY (file_id) REFERENCES files(id) ON DELETE SET NULL,
    CONSTRAINT ck_novel_chapter_attachments_file 
        CHECK (file_id IS NOT NULL OR file_url IS NOT NULL)
);
```

**优点**：
- 灵活性高，支持多种场景
- 可以支持外部文件URL

**缺点**：
- 数据一致性较差
- 文件唯一性难以保证（URL方式）
- 查询逻辑复杂（需要判断使用file_id还是file_url）

## 推荐方案

**推荐使用方案一：核心模块统一文件管理**

**理由**：
1. **符合架构设计**：核心模块作为基础模块，应该统一管理文件信息
2. **保证数据一致性**：所有文件信息统一管理，便于维护
3. **支持模块可选性**：无论是否使用存储模块，都能正常工作
4. **文件唯一性统一**：通过核心模块files表统一实现文件去重
5. **文件URL获取**：通过files表的storage_url字段获取，不需要在附件表中重复存储

**实施建议**：
1. 在核心模块设计中添加files表（或确认是否已存在）
2. 附件表关联核心模块的files表
3. 如果使用存储模块，设计文件信息同步机制（通过gRPC）
4. 文件URL通过files表的storage_url字段获取，不在附件表中存储

## 总结

当前小说章节附件表设计存在以下问题：
1. 违反存储模块可选性原则
2. 跨数据库外键关联不可行
3. 无法支持未使用存储模块的场景
4. 模块间耦合度过高

建议采用**核心模块统一文件管理**的方案，确保系统架构的灵活性和数据一致性。

