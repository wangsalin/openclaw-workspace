# AI 客服系统 - 数据库设计

**版本：** V1.0
**日期：** 2026-03-01

---

## 1. 数据库概览

| 数据库 | 用途 | 字符集 |
|--------|------|--------|
| ai_customer_service | 业务数据 | utf8mb4 |
| vector_db | 向量数据 | - |

---

## 2. 表结构

### 2.1 用户表 (users)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT | 主键 |
| username | VARCHAR(50) | 用户名 |
| password_hash | VARCHAR(255) | 密码 |
| role | ENUM | admin/agent/user |
| status | TINYINT | 0禁用 1启用 |
| created_at | DATETIME | 创建时间 |
| updated_at | DATETIME | 更新时间 |

```sql
CREATE TABLE `users` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `username` VARCHAR(50) NOT NULL UNIQUE,
  `password_hash` VARCHAR(255) NOT NULL,
  `role` ENUM('admin', 'agent', 'user') DEFAULT 'user',
  `status` TINYINT DEFAULT 1,
  `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
  `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

### 2.2 知识库表 (knowledge)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT | 主键 |
| title | VARCHAR(200) | 标题 |
| content | TEXT | 内容 |
| category | VARCHAR(50) | 分类 |
| status | TINYINT | 0待审核 1已发布 |
| vector_status | TINYINT | 0未向量 1已向量 |
| created_by | BIGINT | 创建人 |
| created_at | DATETIME | 创建时间 |
| updated_at | DATETIME | 更新时间 |

```sql
CREATE TABLE `knowledge` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `title` VARCHAR(200) NOT NULL,
  `content` TEXT NOT NULL,
  `category` VARCHAR(50) DEFAULT 'general',
  `status` TINYINT DEFAULT 0,
  `vector_status` TINYINT DEFAULT 0,
  `created_by` BIGINT,
  `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
  `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX `idx_category` (`category`),
  INDEX `idx_status` (`status`)
);
```

---

### 2.3 对话表 (conversations)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT | 主键 |
| user_id | BIGINT | 用户ID |
| channel | VARCHAR(20) | weixin/feishu/web/app |
| channel_user_id | VARCHAR(100) | 渠道用户ID |
| status | ENUM | pending/answered/transferred/closed |
| rating | TINYINT | 满意度 1-5 |
| started_at | DATETIME | 开始时间 |
| closed_at | DATETIME | 结束时间 |

```sql
CREATE TABLE `conversations` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `user_id` BIGINT NOT NULL,
  `channel` VARCHAR(20) NOT NULL,
  `channel_user_id` VARCHAR(100),
  `status` ENUM('pending', 'answered', 'transferred', 'closed') DEFAULT 'pending',
  `rating` TINYINT,
  `started_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
  `closed_at` DATETIME,
  INDEX `idx_user_id` (`user_id`),
  INDEX `idx_channel` (`channel`),
  INDEX `idx_status` (`status`)
);
```

---

### 2.4 消息表 (messages)

| 字段 | 类型 |说明 |
|------|------|------|
| id | BIGINT | 主键 |
| conversation_id | BIGINT | 对话ID |
| role | ENUM | user/assistant/agent |
| content | TEXT | 消息内容 |
| source | VARCHAR(20) | ai/human/robot |
| created_at | DATETIME | 创建时间 |

```sql
CREATE TABLE `messages` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `conversation_id` BIGINT NOT NULL,
  `role` ENUM('user', 'assistant', 'agent') NOT NULL,
  `content` TEXT NOT NULL,
  `source` VARCHAR(20) DEFAULT 'ai',
  `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX `idx_conversation_id` (`conversation_id`)
);
```

---

### 2.5 渠道配置表 (channels)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT | 主键 |
| name | VARCHAR(50) | 渠道名称 |
| type | VARCHAR(20) | weixin/feishu/web |
| config | JSON | 配置信息 |
| status | TINYINT | 0禁用 1启用 |
| created_at | DATETIME | 创建时间 |

```sql
CREATE TABLE `channels` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `name` VARCHAR(50) NOT NULL,
  `type` VARCHAR(20) NOT NULL,
  `config` JSON,
  `status` TINYINT DEFAULT 1,
  `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

### 2.6 统计表 (statistics)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT | 主键 |
| date | DATE | 日期 |
| channel | VARCHAR(20) | 渠道 |
| total_conversations | INT | 对话数 |
| ai_answered | INT | AI 回答数 |
| human_answered | INT | 人工回答数 |
| transferred | INT | 转人工数 |
| avg_rating | DECIMAL | 平均评分 |

```sql
CREATE TABLE `statistics` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `date` DATE NOT NULL,
  `channel` VARCHAR(20),
  `total_conversations` INT DEFAULT 0,
  `ai_answered` INT DEFAULT 0,
  `human_answered` INT DEFAULT 0,
  `transferred` INT DEFAULT 0,
  `avg_rating` DECIMAL(3,2),
  UNIQUE INDEX `idx_date_channel` (`date`, `channel`)
);
```

---

## 3. ER 关系图

```
users ─────┬───────── conversations ──────┬────── messages
           │                              │
           │                              │
           └────────── channels ──────────┘
           
knowledge ──────► vector_db (独立)

statistics (独立统计)
```

---

## 4. 索引优化

| 表 | 索引 | 用途 |
|----|------|------|
| users | username | 登录查询 |
| knowledge | category, status | 知识库筛选 |
| conversations | user_id, channel, status | 对话查询 |
| messages | conversation_id | 消息历史 |
| statistics | date, channel | 统计报表 |

---

**文档状态：** 初稿
