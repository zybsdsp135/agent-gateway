# MCP Gateway详细设计文档

| 项目 | 内容 |
| --- | --- |
| 文档版本 | v1.0 |
| 创建日期 | 2026-06-04 |
| 作者 | CodeAgent |
| 状态 | 设计完成 |
| 服务名称 | MCP Gateway Service |
| 部署形态 | 独立微服务 |

---

## 目录

1. [服务概述](#1-服务概述)
2. [特性分级](#2-特性分级)
3. [详细设计](#3-详细设计)
4. [数据结构设计](#4-数据结构设计)
5. [接口设计](#5-接口设计)
6. [流程设计](#6-流程设计)
7. [异常处理](#7-异常处理)
8. [性能优化](#8-性能优化)
9. [安全设计](#9-安全设计)
10. [测试用例](#10-测试用例)
11. [实施计划](#11-实施计划)
12. [附录](#12-附录)

---

## 1. 服务概述

### 1.1 服务定位

MCP Gateway是Agent人机委托机制的核心服务，负责工具调用鉴权、Cookie还原、数据注入、工具元数据管理和MCP协议适配。作为独立的微服务，MCP Gateway提供RESTful API和MCP SSE接口供Agent调用。

### 1.2 核心职责

**主要职责**：
- 工具调用鉴权：验证Agent是否有权限调用指定工具
- Cookie还原：根据Token还原Cookie，调用传统API
- 数据注入：从Token提取环境信息，注入到工具参数
- 工具元数据管理：管理工具分类、标签、参数、返回值等元数据
- MCP协议适配：适配MCP SSE协议，提供工具列表和工具调用接口
- 实时确认处理：处理敏感工具的实时确认请求
- 工具调用审计：记录工具调用的审计日志

**边界职责**：
- 不负责Token生成（Agent Gateway职责）
- 不负责Session管理（Agent Gateway职责）
- 不负责委托策略配置（策略中心职责）
- 不负责委托权限验证（策略中心职责，MCP Gateway调用策略中心接口）

### 1.3 服务依赖

**上游服务**：
- Agent Gateway：Token验证、Cookie换取
- 策略中心：委托权限验证、工具标签查询
- B2B Admin Service：调用传统API（使用还原的Cookie）

**下游服务**：
- Agent：提供MCP SSE接口，供Agent调用工具
- Elasticsearch：审计日志存储
- MySQL：工具元数据持久化存储
- Redis：工具元数据缓存、Cookie缓存

### 1.4 技术栈

**核心技术**：
- Java 21
- Spring Boot 3.4.6
- Spring Framework 6.2.11
- Spring Security 6.4.10
- Jakarta EE（jakarta.* namespace）
- MySQL 8.3.0（持久化存储）
- Redis 7.2（缓存）
- Elasticsearch 8.18.1（审计日志）
- Lombok 1.18.32
- Fastjson 1.2.83
- MCP SDK（Model Context Protocol）

**可选技术**：
- SSE（Server-Sent Events，MCP协议）
- WebSocket（实时确认消息推送，P1特性）
- Netty（高性能网络通信，P2特性）

---

## 2. 特性分级

### 2.1 分级标准

同策略中心分级标准。

### 2.2 特性清单

#### P0核心功能

| 特性ID | 特性名称 | 特性描述 | 优先级 |
|--------|---------|---------|--------|
| P0-001 | 工具调用鉴权 | 验证Agent是否有权限调用指定工具 | HIGH |
| P0-002 | Cookie还原 | 根据Token还原Cookie，调用传统API | HIGH |
| P0-003 | 数据注入 | 从Token提取环境信息，注入到工具参数 | HIGH |
| P0-004 | 工具元数据管理 | 管理工具分类、标签、参数、返回值等元数据 | HIGH |
| P0-005 | MCP工具列表 | 提供MCP SSE工具列表接口 | HIGH |
| P0-006 | MCP工具调用 | 提供MCP SSE工具调用接口 | HIGH |
| P0-007 | 工具调用审计 | 记录工具调用的审计日志 | HIGH |
| P0-008 | 工具元数据缓存 | 支持工具元数据缓存，提升查询性能 | HIGH |

#### P1重要功能

| 特性ID | 特性名称 | 特性描述 | 优先级 |
|--------|---------|---------|--------|
| P1-001 | 实时确认请求 | 处理敏感工具的实时确认请求 | MEDIUM |
| P1-002 | 实时确认等待 | 等待用户实时确认响应 | MEDIUM |
| P1-003 | 工具调用历史查询 | 支持查询工具调用历史，追踪调用记录 | MEDIUM |
| P1-004 | 工具调用统计 | 支持工具调用统计，分析调用情况 | MEDIUM |
| P1-005 | 工具分类管理 | 支持工具分类管理，按类别管理工具 | MEDIUM |
| P1-006 | 工具风险等级管理 | 支持工具风险等级管理，定义敏感工具 | MEDIUM |
| P1-007 | Cookie缓存管理 | 支持Cookie缓存，提升还原性能 | MEDIUM |

#### P2优化功能

| 特性ID | 特性名称 | 特性描述 | 优先级 |
|--------|---------|---------|--------|
| P2-001 | 工具调用性能优化 | 优化工具调用性能，减少延迟 | LOW |
| P2-002 | Cookie批量还原 | 支持Cookie批量还原，提升效率 | LOW |
| P2-003 | 工具元数据预热 | 系统启动时预热工具元数据 | LOW |
| P2-004 | 工具推荐策略 | 基于任务类型推荐工具 | LOW |
| P2-005 | 工具调用限流 | 支持工具调用限流，防止滥用 | LOW |
| P2-006 | 工具调用重试 | 支持工具调用重试，提升成功率 | LOW |

#### P3扩展功能

| 特性ID | 特性名称 | 特性描述 | 优先级 |
|--------|---------|---------|--------|
| P3-001 | 工具调用链追踪 | 支持工具调用链追踪，可视化调用路径 | OPTIONAL |
| P3-002 | 工具性能监控 | 监控工具性能指标 | OPTIONAL |
| P3-003 | 工具故障自动恢复 | 支持工具故障自动恢复 | OPTIONAL |
| P3-004 | 工具负载均衡 | 工具负载均衡和故障转移 | OPTIONAL |
| P3-005 | 工具版本管理 | 支持工具版本管理，追踪工具变更 | OPTIONAL |
| P3-006 | 工具灰度发布 | 支持工具灰度发布，逐步上线 | OPTIONAL |

---

## 3. 详细设计

### 3.1 P0核心功能详细设计

#### 3.1.1 P0-001：工具调用鉴权

**功能描述**：
验证Agent是否有权限调用指定工具，调用策略中心接口验证委托权限。

**业务规则**：
1. 工具调用必须提供Token
2. 工具调用必须验证Token有效性（调用Agent Gateway接口）
3. 工具调用必须验证委托权限（调用策略中心接口）
4. 工具调用必须验证数据范围（调用策略中心接口）
5. 工具调用必须判断是否需要实时确认（调用策略中心接口）
6. 无权限则拒绝调用

**接口设计**：
```
POST /api/v1/tools/verify-call-permission
Content-Type: application/json

Request Body:
{
  "tokenId": "token-uuid-001",
  "toolName": "datasource_create",
  "toolParameters": {
    "name": "test-datasource",
    "type": "MYSQL"
  }
}

Response:
{
  "hasPermission": true,
  "requireConfirmation": false,
  "injectedParameters": {
    "name": "test-datasource",
    "type": "MYSQL",
    "appName": "C00001-O0023",
    "tenantId": "1111111111111111"
  }
}
```

**流程设计**：
```
流程1：工具调用鉴权流程
1. 接收工具调用鉴权请求
2. 验证Token有效性（调用Agent Gateway接口：POST /api/v1/tokens/verify）
3. 提取Token中的userId、agentId、environmentContext、delegationChain
4. 验证委托权限（调用策略中心接口：POST /api/v1/delegations/verify-permission）
5. 判断是否需要实时确认（从策略中心响应中提取requireConfirmation）
6. 数据注入（从Token提取appName、tenantId，注入到toolParameters）
7. 返回鉴权结果和注入后的参数

性能要求：
- 工具调用鉴权延迟：< 20ms
- 工具调用鉴权吞吐量：> 1000 TPS
```

**异常处理**：
```
异常1：Token无效
- 异常代码：MG4001
- 异常信息：Token invalid or expired
- 处理方式：返回hasPermission=false

异常2：无委托权限
- 异常代码：MG4002
- 异常信息：No delegation permission for tool: {toolName}
- 处理方式：返回hasPermission=false

异常3：数据范围不匹配
- 异常代码：MG4003
- 异常信息：Data scope mismatch
- 处理方式：返回hasPermission=false
```

**测试用例**：
```
测试用例1：正常鉴权成功
- 输入：有权限的工具调用
- 预期：鉴权成功，返回hasPermission=true

测试用例2：Token无效
- 输入：无效的Token
- 预期：鉴权失败，返回hasPermission=false

测试用例3：无委托权限
- 输入：无权限的工具
- 预期：鉴权失败，返回hasPermission=false

测试用例4：需要实时确认
- 输入：敏感工具
- 预期：鉴权成功，返回requireConfirmation=true

测试用例5：数据注入成功
- 输入：工具参数
- 预期：鉴权成功，返回注入后的参数（包含appName、tenantId）
```

---

#### 3.1.2 P0-002：Cookie还原

**功能描述**：
根据Token还原Cookie，调用传统API（B2B Admin Service）。

**业务规则**：
1. Cookie还原必须验证Token有效性
2. Cookie还原调用Agent Gateway接口（POST /api/v1/cookies/exchange）
3. Cookie还原后缓存到Redis（减少实时换取次数）
4. Cookie还原后调用传统API（使用还原的Cookie）
5. Cookie还原记录审计日志

**接口设计**：
```
POST /api/v1/tools/call-with-cookie
Content-Type: application/json

Request Body:
{
  "tokenId": "token-uuid-001",
  "apiPath": "/b2badmin/api/v1/datasources",
  "apiMethod": "POST",
  "apiBody": {
    "name": "test-datasource",
    "type": "MYSQL"
  }
}

Response:
{
  "apiResponse": {
    "datasourceId": "ds-001",
    "name": "test-datasource",
    "type": "MYSQL",
    "status": "ACTIVE"
  },
  "callTime": "2026-06-04T10:00:00Z"
}
```

**流程设计**：
```
流程1：Cookie还原调用传统API流程
1. 接收工具调用请求（包含Token、API路径、API方法、API参数）
2. 验证Token有效性（调用Agent Gateway接口）
3. 查询Cookie缓存（Key: cookie:{tokenId})
4. 缓存命中则使用缓存Cookie
5. 缓存未命中则换取Cookie（调用Agent Gateway接口：POST /api/v1/cookies/exchange）
6. 缓存Cookie到Redis（Key: cookie:{tokenId}, TTL: 与Token有效期一致）
7. 构建HTTP请求（包含Cookie、API路径、API方法、API参数）
8. 调用传统API（B2B Admin Service）
9. 记录审计日志到Elasticsearch
10. 返回API响应

性能要求：
- Cookie还原延迟：< 20ms（缓存命中），< 50ms（缓存未命中）
- 传统API调用延迟：< 200ms（取决于传统API性能）
```

**测试用例**：
```
测试用例1：正常调用传统API
- 输入：有效的Token和API参数
- 预期：调用成功，返回API响应

测试用例2：Cookie缓存命中
- 输入：已缓存的Token
- 预期：使用缓存Cookie，调用成功

测试用例3：Cookie缓存未命中
- 输入：未缓存的Token
- 预期：换取Cookie，缓存后调用成功

测试用例4：Token无效
- 输入：无效的Token
- 预期：调用失败，返回错误

测试用例5：传统API调用失败
- 输入：API参数错误
- 预期：调用失败，返回传统API错误
```

---

#### 3.1.3 P0-003：数据注入

**功能描述**：
从Token提取环境信息（appName、tenantId），注入到工具参数。

**业务规则**：
1. 数据注入必须验证Token有效性
2. 数据注入从Token提取appName、tenantId
3. 数据注入将appName、tenantId添加到工具参数
4. 数据注入不影响原有工具参数
5. 数据注入后的参数用于调用传统API

**数据注入规则**：
```
注入规则：
1. appName注入：
   - 工具参数中添加appName字段
   - appName值从Token的environmentContext提取

2. tenantId注入：
   - 工具参数中添加tenantId字段
   - tenantId值从Token的environmentContext提取

3. 其他环境信息注入：
   - 根据工具需求注入其他环境信息
   - 如userId、sessionId等

注入示例：
原始参数：
{
  "name": "test-datasource",
  "type": "MYSQL"
}

注入后参数：
{
  "name": "test-datasource",
  "type": "MYSQL",
  "appName": "C00001-O0023",
  "tenantId": "1111111111111111"
}
```

**流程设计**：
```
流程1：数据注入流程
1. 接收工具参数
2. 验证Token有效性
3. 提取Token中的environmentContext（appName、tenantId）
4. 复制原始工具参数
5. 添加appName到工具参数
6. 添加tenantId到工具参数
7. 返回注入后的参数

性能要求：
- 数据注入延迟：< 5ms
```

**测试用例**：
```
测试用例1：正常数据注入
- 输入：工具参数和Token
- 预期：注入成功，参数包含appName、tenantId

测试用例2：保留原有参数
- 输入：包含原有参数的工具参数
- 预期：注入后保留原有参数，新增appName、tenantId

测试用例3：Token无环境信息
- 输入：无environmentContext的Token
- 预期：注入失败，返回错误
```

---

#### 3.1.4 P0-004：工具元数据管理

**功能描述**：
管理工具分类、标签、参数、返回值等元数据，供策略中心和Agent使用。

**业务规则**：
1. 工具元数据包含工具名称、工具分类、工具标签、工具描述、风险等级
2. 工具元数据包含工具参数（参数名称、参数类型、是否必填、参数描述）
3. 工具元数据包含工具返回值（返回类型、返回描述）
4. 工具元数据包含工具示例
5. 工具元数据持久化到MySQL
6. 工具元数据缓存到Redis
7. 工具元数据同步到策略中心

**数据结构**：
```java
@Data
public class ToolMetadata {
    private String toolId;
    private String toolName;
    private String toolCategory;
    private List<String> toolTags;
    private String toolDescription;
    private RiskLevel riskLevel;
    private List<ToolParameter> parameters;
    private ToolReturn returnValue;
    private List<String> examples;
    private String apiPath;
    private String apiMethod;
    private Date createTime;
    private Date updateTime;
}

@Data
public class ToolParameter {
    private String paramName;
    private String paramType;
    private Boolean required;
    private String paramDescription;
    private String defaultValue;
    private Boolean injectFromToken;
}

@Data
public class ToolReturn {
    private String returnType;
    private String returnDescription;
}

public enum RiskLevel {
    LOW, MEDIUM, HIGH, CRITICAL
}
```

**接口设计**：
```
接口1：创建工具元数据
POST /api/v1/tools/metadata
Content-Type: application/json

Request Body:
{
  "toolName": "datasource_create",
  "toolCategory": "datasource",
  "toolTags": ["datasource", "create"],
  "toolDescription": "创建数据源",
  "riskLevel": "MEDIUM",
  "parameters": [
    {
      "paramName": "name",
      "paramType": "String",
      "required": true,
      "paramDescription": "数据源名称",
      "injectFromToken": false
    },
    {
      "paramName": "appName",
      "paramType": "String",
      "required": true,
      "paramDescription": "应用名称",
      "injectFromToken": true
    }
  ],
  "returnValue": {
    "returnType": "DatasourceVO",
    "returnDescription": "数据源详情"
  },
  "apiPath": "/b2badmin/api/v1/datasources",
  "apiMethod": "POST"
}

Response:
{
  "toolId": "tool-001",
  "status": "ACTIVE",
  "createTime": "2026-06-04T10:00:00Z"
}

接口2：查询工具元数据
GET /api/v1/tools/{toolName}/metadata

Response:
{
  "toolId": "tool-001",
  "toolName": "datasource_create",
  "toolCategory": "datasource",
  ...
}

接口3：同步工具元数据到策略中心
POST /api/v1/tools/sync-to-strategy-center

Response:
{
  "syncResult": {
    "successCount": 10,
    "failedCount": 0
  }
}
```

**流程设计**：
```
流程1：工具元数据创建流程
1. 接收工具元数据创建请求
2. 验证请求参数完整性
3. 验证工具名称唯一性
4. 生成工具ID（UUID）
5. 持久化工具元数据到MySQL（tool_metadata表）
6. 缓存工具元数据到Redis（Key: tool:{toolName}:metadata）
7. 同步工具元数据到策略中心（调用策略中心接口：POST /api/v1/tools/sync）
8. 记录审计日志到Elasticsearch
9. 返回工具ID和状态

流程2：工具元数据查询流程
1. 接收工具元数据查询请求
2. 查询Redis缓存（Key: tool:{toolName}:metadata）
3. 缓存命中则返回缓存数据
4. 缓存未命中则查询MySQL
5. 查询成功则缓存到Redis
6. 返回工具元数据

性能要求：
- 工具元数据创建延迟：< 50ms
- 工具元数据查询延迟：< 10ms（缓存命中）
```

**测试用例**：
```
测试用例1：正常创建工具元数据
- 输入：完整的工具元数据参数
- 预期：创建成功，返回工具ID

测试用例2：工具名称冲突
- 输入：已存在的工具名称
- 预期：返回409错误

测试用例3：查询工具元数据
- 输入：存在的工具名称
- 预期：返回工具元数据

测试用例4：工具不存在
- 输入：不存在的工具名称
- 预期：返回404错误
```

---

#### 3.1.5 P0-005：MCP工具列表

**功能描述**：
提供MCP SSE工具列表接口，供Agent查询可用工具。

**业务规则**：
1. MCP工具列表接口遵循MCP SSE协议
2. MCP工具列表返回所有可用工具的元数据
3. MCP工具列表包含工具名称、工具描述、工具参数
4. MCP工具列表支持按Token过滤（只返回有权限的工具）

**MCP协议适配**：
```
MCP SSE协议：
- 协议类型：Server-Sent Events（SSE）
- 事件类型：tools/list
- 数据格式：JSON

MCP工具列表请求：
GET /mcp/tools/list
Accept: text/event-stream

MCP工具列表响应：
event: tools/list
data: {"tools": [{"name": "datasource_create", "description": "创建数据源", ...}]}

event: tools/list
data: {"tools": [{"name": "datasource_query", "description": "查询数据源", ...}]}
```

**接口设计**：
```
GET /mcp/tools/list
Accept: text/event-stream
Authorization: Bearer {token}

Response (SSE):
event: tools/list
data: {
  "tools": [
    {
      "name": "datasource_create",
      "description": "创建数据源",
      "inputSchema": {
        "type": "object",
        "properties": {
          "name": {"type": "string", "description": "数据源名称"},
          "type": {"type": "string", "description": "数据源类型"}
        },
        "required": ["name", "type"]
      }
    }
  ]
}
```

**流程设计**：
```
流程1：MCP工具列表流程
1. 接收MCP工具列表请求（SSE）
2. 验证Token有效性
3. 提取Token中的userId、agentId
4. 查询所有工具元数据（从Redis或MySQL）
5. 过滤有权限的工具（调用策略中心接口验证权限）
6. 构建MCP工具列表响应（符合MCP协议格式）
7. 通过SSE返回工具列表

性能要求：
- MCP工具列表延迟：< 100ms
- MCP工具列表吞吐量：> 100 TPS
```

**测试用例**：
```
测试用例1：正常返回MCP工具列表
- 输入：有效的Token
- 预期：返回工具列表，格式符合MCP协议

测试用例2：过滤无权限工具
- 输入：部分工具无权限的Token
- 预期：只返回有权限的工具

测试用例3：Token无效
- 输入：无效的Token
- 预期：返回错误，无工具列表
```

---

#### 3.1.6 P0-006：MCP工具调用

**功能描述**：
提供MCP SSE工具调用接口，供Agent调用工具。

**业务规则**：
1. MCP工具调用接口遵循MCP SSE协议
2. MCP工具调用必须验证Token有效性
3. MCP工具调用必须验证委托权限
4. MCP工具调用必须注入数据（appName、tenantId）
5. MCP工具调用必须还原Cookie，调用传统API
6. MCP工具调用必须处理实时确认（如需要）
7. MCP工具调用必须记录审计日志

**MCP协议适配**：
```
MCP SSE协议：
- 协议类型：Server-Sent Events（SSE）
- 事件类型：tools/call
- 数据格式：JSON

MCP工具调用请求：
POST /mcp/tools/call
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "name": "datasource_create",
  "arguments": {
    "name": "test-datasource",
    "type": "MYSQL"
  }
}

MCP工具调用响应：
event: tools/call
data: {"result": {"datasourceId": "ds-001", "name": "test-datasource", ...}}

event: tools/call
data: {"status": "completed"}
```

**接口设计**：
```
POST /mcp/tools/call
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "name": "datasource_create",
  "arguments": {
    "name": "test-datasource",
    "type": "MYSQL"
  }
}

Response (SSE):
event: tools/call
data: {
  "result": {
    "datasourceId": "ds-001",
    "name": "test-datasource",
    "type": "MYSQL",
    "status": "ACTIVE"
  }
}

event: tools/call
data: {"status": "completed"}
```

**流程设计**：
```
流程1：MCP工具调用流程
1. 接收MCP工具调用请求（SSE）
2. 验证Token有效性（调用Agent Gateway接口）
3. 提取Token中的userId、agentId、environmentContext、delegationChain
4. 验证委托权限（调用策略中心接口）
5. 判断是否需要实时确认（从策略中心响应中提取requireConfirmation）
6. 如需要实时确认：
   - 调用Agent Gateway接口发送确认请求（POST /api/v1/confirmations/request）
   - 等待用户确认响应（通过WebSocket或HTTP）
   - 用户拒绝则返回错误
7. 数据注入（从Token提取appName、tenantId，注入到arguments）
8. 查询工具元数据（获取apiPath、apiMethod）
9. Cookie还原（调用Agent Gateway接口：POST /api/v1/cookies/exchange）
10. 调用传统API（使用还原的Cookie和注入后的参数）
11. 构建MCP工具调用响应（符合MCP协议格式）
12. 通过SSE返回工具调用结果
13. 记录审计日志到Elasticsearch

性能要求：
- MCP工具调用延迟：< 200ms（不含实时确认）
- MCP工具调用吞吐量：> 500 TPS
```

**测试用例**：
```
测试用例1：正常MCP工具调用
- 输入：有效的Token和工具参数
- 预期：调用成功，返回工具结果

测试用例2：MCP工具调用需要实时确认
- 输入：敏感工具
- 预期：发送确认请求，等待用户确认后调用

测试用例3：用户拒绝实时确认
- 输入：用户拒绝确认
- 预期：调用失败，返回错误

测试用例4：Token无效
- 输入：无效的Token
- 预期：调用失败，返回错误

测试用例5：无委托权限
- 输入：无权限的工具
- 预期：调用失败，返回错误
```

---

#### 3.1.7 P0-007：工具调用审计

**功能描述**：
记录工具调用的审计日志，包含调用时间、用户、Agent、工具、参数、结果等。

**业务规则**：
1. 工具调用审计日志存储到Elasticsearch
2. 工具调用审计日志包含调用时间、用户ID、Agent ID、工具名称、参数、结果、错误信息
3. 工具调用审计日志包含Token ID、Session ID
4. 工具调用审计日志包含API路径、API方法
5. 工具调用审计日志包含调用延迟、调用状态
6. 工具调用审计日志保留90天

**审计日志结构**：
```json
{
  "timestamp": "2026-06-04T10:00:00Z",
  "tokenId": "token-uuid-001",
  "sessionId": "session-uuid-001",
  "userId": "l00867517",
  "agentId": "agent-001",
  "toolName": "datasource_create",
  "toolParameters": {
    "name": "test-datasource",
    "type": "MYSQL",
    "appName": "C00001-O0023",
    "tenantId": "1111111111111111"
  },
  "apiPath": "/b2badmin/api/v1/datasources",
  "apiMethod": "POST",
  "callResult": "SUCCESS",
  "callResponse": {
    "datasourceId": "ds-001",
    "name": "test-datasource"
  },
  "callDuration": 150,
  "requireConfirmation": false,
  "confirmationDecision": null,
  "errorMessage": null
}
```

**流程设计**：
```
流程1：工具调用审计日志记录流程
1. 工具调用开始时记录调用开始日志
2. 工具调用结束时记录调用结束日志
3. 异步写入Elasticsearch（使用Disruptor队列）
4. 批量写入（1000条/次或100ms超时）

性能要求：
- 审计日志写入延迟：< 5ms（异步）
- 审计日志写入吞吐量：> 50000 TPS
```

**测试用例**：
```
测试用例1：正常记录审计日志
- 输入：工具调用
- 预期：审计日志记录成功

测试用例2：审计日志查询
- 输入：时间范围和用户ID
- 预期：返回审计日志列表

测试用例3：审计日志完整性
- 输入：工具调用
- 预期：审计日志包含所有必要字段
```

---

#### 3.1.8 P0-008：工具元数据缓存

**功能描述**：
支持工具元数据缓存，提升查询性能。

**业务规则**：
1. 工具元数据缓存到Redis，Key为tool:{toolName}:metadata
2. 工具元数据缓存包含工具完整信息
3. 工具元数据缓存过期时间1小时
4. 工具元数据更新时刷新缓存
5. 工具元数据删除时删除缓存

**缓存设计**：
```
缓存Key设计：
1. 工具元数据缓存：
   - Key: tool:{toolName}:metadata
   - Value: ToolMetadata JSON
   - TTL: 1小时

2. 工具分类缓存：
   - Key: tool:category:{category}
   - Value: List<String>（工具名称列表）
   - TTL: 1小时

3. 工具标签缓存：
   - Key: tool:tags
   - Value: Map<String, List<String>>（标签→工具列表）
   - TTL: 1小时

4. Cookie缓存：
   - Key: cookie:{tokenId}
   - Value: encryptedCookie
   - TTL: 与Token有效期一致
```

**流程设计**：
```
流程1：工具元数据缓存写入流程
1. 工具元数据创建/更新成功
2. 序列化工具元数据为JSON
3. 写入Redis缓存（Key: tool:{toolName}:metadata）
4. 设置TTL为1小时
5. 更新工具分类缓存和工具标签缓存

流程2：工具元数据缓存查询流程
1. 工具元数据查询请求
2. 先查询Redis缓存
3. 缓存命中则返回缓存数据
4. 缓存未命中则查询MySQL
5. 查询成功则写入缓存

流程3：工具元数据缓存删除流程
1. 工具元数据删除成功
2. 删除Redis缓存（Key: tool:{toolName}:metadata）
3. 更新工具分类缓存和工具标签缓存
```

**性能要求**：
- 缓存命中率：> 95%
- 缓存查询延迟：< 5ms

---

### 3.2 P1重要功能详细设计

#### 3.2.1 P1-001：实时确认请求

**功能描述**：
处理敏感工具的实时确认请求，调用Agent Gateway接口发送确认请求。

**业务规则**：
1. 实时确认请求来自工具调用鉴权（requireConfirmation=true）
2. 实时确认请求调用Agent Gateway接口（POST /api/v1/confirmations/request）
3. 实时确认请求包含Token ID、工具名称、操作描述
4. 实时确认请求设置超时时间（默认30秒）
5. 实时确认请求等待用户响应

**接口设计**：
已在P0-006 MCP工具调用流程中包含实时确认处理。

**流程设计**：
```
流程1：实时确认请求流程
1. 工具调用鉴权返回requireConfirmation=true
2. 构建确认请求（包含Token ID、工具名称、操作描述）
3. 调用Agent Gateway接口发送确认请求
4. 等待用户确认响应（通过WebSocket或HTTP轮询）
5. 用户确认响应后继续工具调用
6. 用户拒绝则返回错误

性能要求：
- 实时确认请求延迟：< 10ms
- 实时确认等待延迟：< 30秒（用户响应时间）
```

**测试用例**：
```
测试用例1：正常实时确认请求
- 输入：敏感工具调用
- 预期：发送确认请求，等待用户确认

测试用例2：用户确认后继续调用
- 输入：用户确认响应
- 预期：继续工具调用，返回结果

测试用例3：用户拒绝后返回错误
- 输入：用户拒绝响应
- 预期：返回错误，工具调用失败

测试用例4：实时确认超时
- 输入：用户未响应（超时）
- 预期：返回错误，工具调用失败
```

---

#### 3.2.2 P1-002：实时确认等待

**功能描述**：
等待用户实时确认响应，支持WebSocket或HTTP轮询。

**业务规则**：
1. 实时确认等待支持WebSocket实时推送
2. 实时确认等待支持HTTP轮询（备用方案）
3. 实时确认等待超时时间可配置（默认30秒）
4. 实时确认等待超时自动拒绝
5. 实时确认等待记录审计日志

**流程设计**：
```
流程1：实时确认等待流程（WebSocket）
1. 发送确认请求后，建立WebSocket连接
2. 监听WebSocket消息（用户确认响应）
3. 接收用户确认响应
4. 关闭WebSocket连接
5. 返回确认决策

流程2：实时确认等待流程（HTTP轮询）
1. 发送确认请求后，返回确认ID
2. 定时轮询确认状态（每1秒轮询一次）
3. 确认状态变为COMPLETED则返回决策
4. 超时则返回拒绝决策

性能要求：
- WebSocket延迟：< 100ms
- HTTP轮询延迟：< 1秒（轮询间隔）
```

**测试用例**：
```
测试用例1：WebSocket实时确认等待
- 输入：WebSocket连接
- 预期：实时接收用户确认响应

测试用例2：HTTP轮询实时确认等待
- 输入：HTTP轮询
- 预期：定时查询确认状态

测试用例3：实时确认超时
- 输入：超时时间
- 预期：超时后自动拒绝
```

---

### 3.3 P2-P3功能详细设计

由于篇幅限制，P2和P3功能的详细设计省略，参考策略中心的P2-P3设计模式。

---

## 4. 数据结构设计

### 4.1 数据库表设计

#### 4.1.1 Tool Metadata表（tool_metadata）

```sql
CREATE TABLE tool_metadata (
    tool_id VARCHAR(64) PRIMARY KEY COMMENT '工具ID',
    tool_name VARCHAR(128) NOT NULL COMMENT '工具名称',
    tool_category VARCHAR(64) NOT NULL COMMENT '工具分类',
    tool_tags JSON NOT NULL COMMENT '工具标签列表',
    tool_description VARCHAR(512) COMMENT '工具描述',
    risk_level VARCHAR(32) NOT NULL COMMENT '风险等级：LOW/MEDIUM/HIGH/CRITICAL',
    parameters JSON COMMENT '工具参数',
    return_value JSON COMMENT '返回值',
    examples JSON COMMENT '示例',
    api_path VARCHAR(256) COMMENT 'API路径',
    api_method VARCHAR(32) COMMENT 'API方法：GET/POST/PUT/DELETE',
    create_time DATETIME NOT NULL COMMENT '创建时间',
    update_time DATETIME NOT NULL COMMENT '更新时间',
    INDEX idx_tool_name (tool_name),
    INDEX idx_tool_category (tool_category),
    INDEX idx_risk_level (risk_level)
) COMMENT 'Tool Metadata表';
```

#### 4.1.2 Tool Call Audit表（tool_call_audit）

```sql
CREATE TABLE tool_call_audit (
    audit_id VARCHAR(64) PRIMARY KEY COMMENT '审计ID',
    token_id VARCHAR(64) NOT NULL COMMENT 'Token ID',
    session_id VARCHAR(64) COMMENT 'Session ID',
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    agent_id VARCHAR(64) NOT NULL COMMENT 'Agent ID',
    tool_name VARCHAR(128) NOT NULL COMMENT '工具名称',
    tool_parameters JSON COMMENT '工具参数',
    api_path VARCHAR(256) COMMENT 'API路径',
    api_method VARCHAR(32) COMMENT 'API方法',
    call_result VARCHAR(32) COMMENT '调用结果：SUCCESS/FAILED',
    call_response JSON COMMENT '调用响应',
    call_duration INT COMMENT '调用延迟（ms）',
    require_confirmation BOOLEAN COMMENT '是否需要实时确认',
    confirmation_decision VARCHAR(32) COMMENT '确认决策：ALLOWED/REJECTED',
    error_message TEXT COMMENT '错误信息',
    call_time DATETIME NOT NULL COMMENT '调用时间',
    INDEX idx_token_id (token_id),
    INDEX idx_user_agent (user_id, agent_id),
    INDEX idx_tool_name (tool_name),
    INDEX idx_call_time (call_time)
) COMMENT 'Tool Call Audit表';
```

### 4.2 Redis缓存设计

参考策略中心Redis缓存设计。

### 4.3 Elasticsearch索引设计

参考策略中心Elasticsearch索引设计。

---

## 5. 接口设计

### 5.1 核心接口清单

| 接口ID | 接口路径 | 接口方法 | 接口描述 | 特性ID |
|--------|---------|---------|---------|--------|
| API-001 | /api/v1/tools/verify-call-permission | POST | 工具调用鉴权 | P0-001 |
| API-002 | /api/v1/tools/call-with-cookie | POST | Cookie还原调用传统API | P0-002 |
| API-003 | /api/v1/tools/metadata | POST | 创建工具元数据 | P0-004 |
| API-004 | /api/v1/tools/{toolName}/metadata | GET | 查询工具元数据 | P0-004 |
| API-005 | /api/v1/tools/sync-to-strategy-center | POST | 同步工具元数据到策略中心 | P0-004 |
| API-006 | /mcp/tools/list | GET | MCP工具列表（SSE） | P0-005 |
| API-007 | /mcp/tools/call | POST | MCP工具调用（SSE） | P0-006 |
| API-008 | /api/v1/tools/audit | GET | 查询工具调用审计日志 | P0-007 |

---

## 6. 流程设计

### 6.1 核心流程

参考策略中心流程设计模式。

---

## 7. 异常处理

参考策略中心异常处理设计。

---

## 8. 性能优化

参考策略中心性能优化设计。

---

## 9. 安全设计

参考策略中心安全设计。

---

## 10. 测试用例

参考策略中心测试用例设计。

---

## 11. 实施计划

参考策略中心实施计划。

---

## 12. 附录

参考策略中心附录。

---

**文档结束**
