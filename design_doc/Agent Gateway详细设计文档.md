# Agent Gateway详细设计文档

| 项目 | 内容 |
| --- | --- |
| 文档版本 | v1.0 |
| 创建日期 | 2026-06-04 |
| 作者 | CodeAgent |
| 状态 | 设计完成 |
| 服务名称 | Agent Gateway Service |
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

Agent Gateway是Agent人机委托机制的核心服务，负责Agent注册管理、Copilot到Agent的统一入口代理路由、A2A路由、Token生成、Session管理和Cookie换取。作为独立的微服务，Agent Gateway对Web Copilot暴露注册Agent的统一访问地址，对后端维护Agent真实地址和协议适配配置，运行时完成Cookie到任务级Token的转换、委托授权检查、Token注入和请求转发。

### 1.2 核心职责

**主要职责**：
- Token生成与管理：生成任务级Token，管理Token生命周期
- Session管理：管理用户-Agent会话，存储Cookie和Session信息
- Cookie换取：提供Cookie换取接口，支持MCP Gateway还原Cookie
- Agent注册管理：管理Agent注册信息，维护Agent元数据
- Agent入口代理路由：接收Copilot访问注册Agent的请求，完成授权、Token注入和真实Agent转发
- A2A路由：支持Agent调用Agent的路由和委托链传递
- 实时确认处理：处理实时确认请求，转发确认消息给用户
- 审计日志记录：记录Token生成、Session管理、A2A调用等审计日志

**边界职责**：
- 不负责委托策略配置（策略中心职责）
- 不负责工具调用鉴权（MCP Gateway职责）
- 不负责委托权限验证（策略中心职责）
- 不负责Cookie还原调用传统API（MCP Gateway职责）
- 不要求业务Agent主动调用Token生成、Cookie换取、委托授权等网关内部API

### 1.3 服务依赖

**上游服务**：
- IDaaS认证服务：获取用户身份信息和Cookie
- Web Copilot：接收用户交互请求，提供实时确认界面
- 策略中心：查询委托策略，验证委托权限

**下游服务**：
- MCP Gateway：提供Token验证和Cookie换取接口
- 策略中心：提供Agent注册信息查询接口
- Elasticsearch：审计日志存储
- MySQL：Session、Token、Agent注册信息持久化存储
- Redis：Session缓存、Token缓存

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
- JWT（io.jsonwebtoken:jjwt）

**可选技术**：
- WebSocket（实时确认消息推送，P1特性）
- Kafka（审计日志异步写入，P2特性）
- Netty（高性能网络通信，P3特性）

---

## 2. 特性分级

### 2.1 分级标准

同策略中心分级标准。

### 2.2 特性清单

#### P0核心功能

| 特性ID | 特性名称 | 特性描述 | 优先级 |
|--------|---------|---------|--------|
| P0-001 | Token生成 | 根据用户Cookie生成任务级Token，绑定Task和Session | HIGH |
| P0-002 | Token验证 | 验证Token有效性，提取Token中的环境信息 | HIGH |
| P0-003 | Session创建 | 创建用户-Agent会话，存储Cookie和Session信息 | HIGH |
| P0-004 | Session管理 | 管理Session生命周期，更新Session状态 | HIGH |
| P0-005 | Cookie换取 | 提供Cookie换取接口，根据Token还原Cookie | HIGH |
| P0-006 | Agent注册 | 支持Agent注册，维护Agent元数据 | HIGH |
| P0-007 | Agent查询 | 支持查询Agent信息，供策略中心和MCP Gateway调用 | HIGH |
| P0-008 | Token缓存管理 | 支持Token缓存，提升验证性能 | HIGH |
| P0-009 | Agent入口代理路由 | 代理Copilot到注册Agent的调用，完成Cookie转Token、委托授权和Token注入 | HIGH |

#### P1重要功能

| 特性ID | 特性名称 | 特性描述 | 优先级 |
|--------|---------|---------|--------|
| P1-001 | A2A路由 | 支持Agent调用Agent的路由和委托链传递 | MEDIUM |
| P1-002 | 实时确认处理 | 处理实时确认请求，转发确认消息给用户 | MEDIUM |
| P1-003 | 实时确认响应 | 接收用户确认响应，转发给MCP Gateway | MEDIUM |
| P1-004 | Session历史查询 | 支持查询Session历史，追踪会话变更 | MEDIUM |
| P1-005 | Token历史查询 | 支持查询Token历史，追踪Token使用 | MEDIUM |
| P1-006 | Agent分类管理 | 支持Agent分类，按类型管理Agent | MEDIUM |
| P1-007 | Token滥用检测 | 检测Token滥用行为，自动锁定异常Token | MEDIUM |

#### P2优化功能

| 特性ID | 特性名称 | 特性描述 | 优先级 |
|--------|---------|---------|--------|
| P2-001 | Token预生成 | 预生成Token池，提升Token生成性能 | LOW |
| P2-002 | Token生命周期监控 | 监控Token生命周期状态转换 | LOW |
| P2-003 | Session预热 | 系统启动时预热热点Session | LOW |
| P2-004 | Agent推荐策略 | 基于任务类型推荐Agent | LOW |
| P2-005 | Token批量生成 | 支持批量生成Token，提升效率 | LOW |
| P2-006 | WebSocket实时推送 | 使用WebSocket推送实时确认消息 | LOW |

#### P3扩展功能

| 特性ID | 特性名称 | 特性描述 | 优先级 |
|--------|---------|---------|--------|
| P3-001 | Token加密存储 | Token加密存储，增强安全性 | OPTIONAL |
| P3-002 | Session持久化策略 | 支持Session持久化策略配置 | OPTIONAL |
| P3-003 | Agent性能监控 | 监控Agent性能指标 | OPTIONAL |
| P3-004 | A2A调用链可视化 | 可视化A2A调用链 | OPTIONAL |
| P3-005 | Token自动续期 | 支持Token自动续期机制 | OPTIONAL |
| P3-006 | Agent负载均衡 | Agent负载均衡和故障转移 | OPTIONAL |

---

## 3. 详细设计

### 3.1 P0核心功能详细设计

#### 3.1.1 P0-001：Token生成

**功能描述**：
根据用户Cookie生成任务级Token，绑定Task和Session，包含环境信息、委托链等信息。

**业务规则**：
1. Token必须与Task绑定（taskId）
2. Token必须与Session绑定（sessionId）
3. Token包含用户信息（userId）
4. Token包含Agent信息（agentId）
5. Token包含环境信息（appName、tenantId）
6. Token包含委托链信息（delegationChain）
7. Token有效期与Task生命周期一致
8. Token使用JWT格式，签名防篡改

**数据结构**：
```java
@Data
public class TaskToken {
    private String tokenId;
    private String taskId;
    private String sessionId;
    private String userId;
    private String agentId;
    private EnvironmentContext environmentContext;
    private DelegationChain delegationChain;
    private Date createTime;
    private Date expiryTime;
    private TokenStatus status;
}

@Data
public class EnvironmentContext {
    private String appName;
    private String tenantId;
}

@Data
public class DelegationChain {
    private String chainId;
    private List<DelegationLink> links;
}

@Data
public class DelegationLink {
    private String from;
    private String to;
    private String delegationId;
}
```

**JWT Token结构**：
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "tokenId": "token-uuid-001",
    "taskId": "task-uuid-001",
    "sessionId": "session-uuid-001",
    "userId": "l00867517",
    "agentId": "agent-001",
    "environmentContext": {
      "appName": "C00001-O0023",
      "tenantId": "1111111111111111"
    },
    "delegationChain": [
      {
        "from": "user:l00867517",
        "to": "agent:agent-001",
        "delegationId": "del-uuid-001"
      }
    ],
    "iat": 1704067200,
    "exp": 1704070800
  },
  "signature": "..."
}
```

**接口设计**：
```
POST /api/v1/tokens/generate
Content-Type: application/json

Request Body:
{
  "userId": "l00867517",
  "agentId": "agent-001",
  "taskId": "task-uuid-001",
  "sessionId": "session-uuid-001",
  "cookie": "IDaaS_SSO_Cookie_Value",
  "environmentContext": {
    "appName": "C00001-O0023",
    "tenantId": "1111111111111111"
  },
  "delegationChain": [
    {
      "from": "user:l00867517",
      "to": "agent:agent-001",
      "delegationId": "del-uuid-001"
    }
  ]
}

Response:
{
  "tokenId": "token-uuid-001",
  "tokenValue": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "createTime": "2026-06-04T10:00:00Z",
  "expiryTime": "2026-06-04T11:00:00Z",
  "status": "ACTIVE"
}
```

**流程设计**：
```
流程1：Token生成流程
1. 接收Token生成请求
2. 验证请求参数完整性
3. 验证用户身份（从Cookie提取userId）
4. 验证Agent存在性（调用Agent查询接口）
5. 验证Task存在性（调用策略中心查询接口）
6. 验证Session存在性（查询Session表）
7. 验证委托链（调用策略中心委托链解析接口）
8. 提取环境信息（appName、tenantId）
9. 生成Token ID（UUID）
10. 构建JWT Payload
11. 计算Token有效期（与Task生命周期一致）
12. 使用私钥签名JWT
13. 持久化Token到MySQL（task_token表）
14. 缓存Token到Redis（Key: token:{tokenId})
15. 缓存Cookie到Redis（Key: cookie:{tokenId})
16. 记录审计日志到Elasticsearch
17. 返回Token ID和Token Value

性能要求：
- Token生成延迟：< 20ms
- Token生成吞吐量：> 1000 TPS
- JWT签名延迟：< 10ms
```

**异常处理**：
```
异常1：参数缺失
- 异常代码：AG4001
- 异常信息：Required parameter missing: {parameterName}
- 处理方式：返回400错误

异常2：Agent不存在
- 异常代码：AG4002
- 异常信息：Agent not found: {agentId}
- 处理方式：返回404错误

异常3：Session不存在
- 异常代码：AG4003
- 异常信息：Session not found: {sessionId}
- 处理方式：返回404错误

异常4：委托链无效
- 异常代码：AG4004
- 异常信息：Delegation chain invalid
- 处理方式：返回400错误

异常5：环境信息缺失
- 异常代码：AG4005
- 异常信息：Environment context missing: appName or tenantId
- 处理方式：返回400错误
```

**测试用例**：
```
测试用例1：正常生成Token
- 输入：完整的Token生成参数
- 预期：Token生成成功，返回Token ID和Token Value

测试用例2：生成Token包含委托链
- 输入：包含委托链的Token生成参数
- 预期：Token生成成功，委托链信息正确包含

测试用例3：参数缺失
- 输入：缺少必填参数
- 预期：返回400错误，提示参数缺失

测试用例4：Agent不存在
- 输入：不存在的agentId
- 预期：返回404错误，提示Agent不存在

测试用例5：委托链无效
- 输入：无效的委托链
- 预期：返回400错误，提示委托链无效
```

---

#### 3.1.2 P0-002：Token验证

**功能描述**：
验证Token有效性，提取Token中的环境信息和委托链信息。

**业务规则**：
1. 验证Token签名有效性
2. 验证Token未过期
3. 验证Token状态为ACTIVE
4. 提取Token中的环境信息（appName、tenantId）
5. 提取Token中的委托链信息
6. 提取Token中的用户和Agent信息

**接口设计**：
```
POST /api/v1/tokens/verify
Content-Type: application/json

Request Body:
{
  "tokenValue": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
}

Response:
{
  "valid": true,
  "tokenId": "token-uuid-001",
  "userId": "l00867517",
  "agentId": "agent-001",
  "taskId": "task-uuid-001",
  "sessionId": "session-uuid-001",
  "environmentContext": {
    "appName": "C00001-O0023",
    "tenantId": "1111111111111111"
  },
  "delegationChain": [
    {
      "from": "user:l00867517",
      "to": "agent:agent-001",
      "delegationId": "del-uuid-001"
    }
  ],
  "status": "ACTIVE"
}
```

**流程设计**：
```
流程1：Token验证流程
1. 接收Token验证请求
2. 解析JWT Token
3. 验证JWT签名（使用公钥验证）
4. 验证JWT未过期（exp > 当前时间）
5. 提取Token ID
6. 查询Token状态（从Redis或MySQL查询）
7. 验证Token状态为ACTIVE
8. 提取Token中的所有信息
9. 返回验证结果

性能要求：
- Token验证延迟：< 5ms
- Token验证吞吐量：> 5000 TPS
```

**测试用例**：
```
测试用例1：验证有效Token
- 输入：有效的Token
- 预期：验证成功，返回Token信息

测试用例2：验证过期Token
- 输入：过期的Token
- 预期：验证失败，返回valid=false

测试用例3：验证签名错误Token
- 输入：签名错误的Token
- 预期：验证失败，返回valid=false

测试用例4：验证已撤销Token
- 输入：已撤销的Token
- 预期：验证失败，返回valid=false
```

---

#### 3.1.3 P0-003：Session创建

**功能描述**：
创建用户-Agent会话，存储Cookie和Session信息。

**业务规则**：
1. Session必须绑定用户（userId）
2. Session必须绑定Agent（agentId）
3. Session存储用户Cookie（加密存储）
4. Session包含会话状态（ACTIVE、EXPIRED、CLOSED）
5. Session包含创建时间和过期时间
6. Session过期时间可配置（默认1小时）

**数据结构**：
```java
@Data
public class UserAgentSession {
    private String sessionId;
    private String userId;
    private String agentId;
    private String encryptedCookie;
    private SessionStatus status;
    private Date createTime;
    private Date expiryTime;
    private Date updateTime;
    private String createdBy;
}

public enum SessionStatus {
    ACTIVE, EXPIRED, CLOSED
}
```

**接口设计**：
```
POST /api/v1/sessions/create
Content-Type: application/json

Request Body:
{
  "userId": "l00867517",
  "agentId": "agent-001",
  "cookie": "IDaaS_SSO_Cookie_Value",
  "expiryDuration": 3600
}

Response:
{
  "sessionId": "session-uuid-001",
  "status": "ACTIVE",
  "createTime": "2026-06-04T10:00:00Z",
  "expiryTime": "2026-06-04T11:00:00Z"
}
```

**流程设计**：
```
流程1：Session创建流程
1. 接收Session创建请求
2. 验证请求参数完整性
3. 验证用户身份（从Cookie提取userId）
4. 验证Agent存在性
5. 生成Session ID（UUID）
6. 加密Cookie（AES-256加密）
7. 设置Session状态为ACTIVE
8. 设置Session过期时间（createTime + expiryDuration）
9. 持久化Session到MySQL（user_agent_session表）
10. 缓存Session到Redis（Key: session:{sessionId})
11. 记录审计日志到Elasticsearch
12. 返回Session ID和状态

性能要求：
- Session创建延迟：< 30ms
- Session创建吞吐量：> 500 TPS
```

**测试用例**：
```
测试用例1：正常创建Session
- 输入：完整的Session创建参数
- 预期：Session创建成功，返回Session ID

测试用例2：创建Session并加密Cookie
- 输入：包含Cookie的Session创建参数
- 预期：Session创建成功，Cookie加密存储

测试用例3：参数缺失
- 输入：缺少必填参数
- 预期：返回400错误

测试用例4：Agent不存在
- 输入：不存在的agentId
- 预期：返回404错误
```

---

#### 3.1.4 P0-004：Session管理

**功能描述**：
管理Session生命周期，更新Session状态，延长Session有效期。

**业务规则**：
1. 支持更新Session状态（ACTIVE、EXPIRED、CLOSED）
2. 支持延长Session有效期
3. 支持关闭Session
4. Session过期自动更新状态为EXPIRED
5. Session关闭后无法恢复

**接口设计**：
```
接口1：更新Session状态
PUT /api/v1/sessions/{sessionId}/status
Content-Type: application/json

Request Body:
{
  "status": "CLOSED"
}

Response:
{
  "sessionId": "session-uuid-001",
  "status": "CLOSED",
  "updateTime": "2026-06-04T12:00:00Z"
}

接口2：延长Session有效期
PUT /api/v1/sessions/{sessionId}/extend
Content-Type: application/json

Request Body:
{
  "extendDuration": 1800
}

Response:
{
  "sessionId": "session-uuid-001",
  "expiryTime": "2026-06-04T12:30:00Z"
}

接口3：查询Session详情
GET /api/v1/sessions/{sessionId}

Response:
{
  "sessionId": "session-uuid-001",
  "userId": "l00867517",
  "agentId": "agent-001",
  "status": "ACTIVE",
  "createTime": "2026-06-04T10:00:00Z",
  "expiryTime": "2026-06-04T11:00:00Z"
}
```

**流程设计**：
```
流程1：Session状态更新流程
1. 接收Session状态更新请求
2. 验证Session ID存在性
3. 验证新状态合法性
4. 更新Session状态
5. 设置更新时间
6. 持久化更新到MySQL
7. 刷新Redis缓存
8. 记录审计日志
9. 返回更新结果

流程2：Session有效期延长流程
1. 接收Session有效期延长请求
2. 验证Session ID存在性
3. 验证Session状态为ACTIVE
4. 计算新的过期时间（当前过期时间 + extendDuration）
5. 更新Session过期时间
6. 持久化更新到MySQL
7. 刷新Redis缓存
8. 返回新的过期时间

流程3：Session过期自动处理
1. 定时任务扫描过期Session（每分钟）
2. 查询expiryTime < 当前时间的Session
3. 更新Session状态为EXPIRED
4. 删除Redis缓存
5. 记录审计日志
```

**测试用例**：
```
测试用例1：更新Session状态
- 输入：Session ID和新状态
- 预期：状态更新成功

测试用例2：延长Session有效期
- 输入：Session ID和延长时长
- 预期：有效期延长成功

测试用例3：查询Session详情
- 输入：Session ID
- 预期：返回Session详情

测试用例4：Session不存在
- 输入：不存在的Session ID
- 预期：返回404错误
```

---

#### 3.1.5 P0-005：Cookie换取

**功能描述**：
提供Cookie换取接口，根据Token还原Cookie，供MCP Gateway调用。

**业务规则**：
1. Cookie换取必须验证Token有效性
2. Cookie换取必须验证Token状态为ACTIVE
3. Cookie换取必须验证Session状态为ACTIVE
4. Cookie换取返回解密后的Cookie
5. Cookie换取记录审计日志

**接口设计**：
```
POST /api/v1/cookies/exchange
Content-Type: application/json

Request Body:
{
  "tokenId": "token-uuid-001"
}

Response:
{
  "cookie": "IDaaS_SSO_Cookie_Value",
  "userId": "l00867517",
  "sessionId": "session-uuid-001"
}
```

**流程设计**：
```
流程1：Cookie换取流程
1. 接收Cookie换取请求
2. 验证Token ID存在性
3. 查询Token详情（从Redis或MySQL）
4. 验证Token状态为ACTIVE
5. 提取Session ID
6. 查询Session详情（从Redis或MySQL）
7. 验证Session状态为ACTIVE
8. 解密Cookie（AES-256解密）
9. 记录审计日志到Elasticsearch
10. 返回解密后的Cookie

性能要求：
- Cookie换取延迟：< 20ms
- Cookie换取吞吐量：> 1000 TPS
```

**测试用例**：
```
测试用例1：正常换取Cookie
- 输入：有效的Token ID
- 预期：换取成功，返回Cookie

测试用例2：Token不存在
- 输入：不存在的Token ID
- 预期：返回404错误

测试用例3：Token已过期
- 输入：已过期的Token ID
- 预期：返回400错误，提示Token已过期

测试用例4：Session已过期
- 输入：Session已过期的Token ID
- 预期：返回400错误，提示Session已过期
```

---

#### 3.1.6 P0-006：Agent注册

**功能描述**：
支持Agent注册，维护Agent元数据。

**业务规则**：
1. Agent注册必须提供Agent ID、Agent名称、Agent类型
2. Agent注册必须提供Agent描述和推荐场景
3. Agent注册后状态为ACTIVE
4. Agent注册信息持久化到MySQL
5. Agent注册信息缓存到Redis

**数据结构**：
```java
@Data
public class AgentRegistration {
    private String agentId;
    private String agentName;
    private AgentType agentType;
    private String agentDescription;
    private String agentEndpoint;
    private String gatewayEndpoint;
    private ProtocolAdapterConfig protocolAdapter;
    private List<String> recommendedScenarios;
    private AgentStatus status;
    private Date createTime;
    private Date updateTime;
    private String createdBy;
}

public enum AgentType {
    DATA_INTEGRATION, MONITORING, OPERATIONS, ANALYSIS, GENERAL
}

public enum AgentStatus {
    ACTIVE, INACTIVE, DELETED
}

@Data
public class ProtocolAdapterConfig {
    private RequestMode requestMode;
    private TokenInjectionMode tokenInjection;
    private String contextHeaderName;
    private Map<String, String> headerMappings;
}
```

**接口设计**：
```
POST /api/v1/agents/register
Content-Type: application/json

Request Body:
{
  "agentId": "agent-001",
  "agentName": "数据集成Agent",
  "agentType": "DATA_INTEGRATION",
  "agentDescription": "负责数据集成任务的Agent",
  "agentEndpoint": "http://agent-001.internal:8080/chat",
  "protocolAdapter": {
    "requestMode": "HTTP_JSON",
    "tokenInjection": "AUTHORIZATION_BEARER",
    "contextHeaderName": "X-Agent-Task-Context"
  },
  "recommendedScenarios": ["数据源管理", "集成任务配置"]
}

Response:
{
  "agentId": "agent-001",
  "gatewayEndpoint": "https://agent-gateway.example.com/agents/agent-001/invoke",
  "status": "ACTIVE",
  "createTime": "2026-06-04T10:00:00Z"
}
```

**流程设计**：
```
流程1：Agent注册流程
1. 接收Agent注册请求
2. 验证请求参数完整性
3. 验证Agent ID唯一性（查询agent_registration表）
4. 设置Agent状态为ACTIVE
5. 设置创建时间和创建人
6. 持久化Agent注册信息到MySQL
7. 缓存Agent注册信息到Redis
8. 同步Agent信息到策略中心（调用策略中心接口）
9. 记录审计日志到Elasticsearch
10. 返回Agent ID和状态

性能要求：
- Agent注册延迟：< 50ms
- Agent注册吞吐量：> 100 TPS
```

**测试用例**：
```
测试用例1：正常注册Agent
- 输入：完整的Agent注册参数
- 预期：Agent注册成功，返回Agent ID

测试用例2：Agent ID冲突
- 输入：已存在的Agent ID
- 预期：返回409错误，提示Agent ID已存在

测试用例3：参数缺失
- 输入：缺少必填参数
- 预期：返回400错误
```

---

#### 3.1.7 P0-007：Agent查询

**功能描述**：
支持查询Agent信息，供策略中心和MCP Gateway调用。

**业务规则**：
1. 支持按Agent ID查询Agent详情
2. 支持按Agent类型查询Agent列表
3. 支持查询所有Agent列表
4. 查询结果包含Agent完整信息

**接口设计**：
```
接口1：按Agent ID查询
GET /api/v1/agents/{agentId}

Response:
{
  "agentId": "agent-001",
  "agentName": "数据集成Agent",
  "agentType": "DATA_INTEGRATION",
  "agentDescription": "负责数据集成任务的Agent",
  "gatewayEndpoint": "https://agent-gateway.example.com/agents/agent-001/invoke",
  "recommendedScenarios": ["数据源管理", "集成任务配置"],
  "status": "ACTIVE",
  "createTime": "2026-06-04T10:00:00Z"
}

接口2：按Agent类型查询
GET /api/v1/agents?type={agentType}

Response:
[
  {
    "agentId": "agent-001",
    "agentName": "数据集成Agent",
    ...
  }
]

接口3：查询所有Agent
GET /api/v1/agents

Response:
[
  {
    "agentId": "agent-001",
    ...
  }
]
```

**流程设计**：
```
流程1：Agent查询流程
1. 接收Agent查询请求
2. 查询Redis缓存（Key: agent:{agentId})
3. 缓存命中则返回缓存数据
4. 缓存未命中则查询MySQL
5. 查询成功则缓存到Redis
6. 返回Agent信息

性能要求：
- Agent查询延迟：< 10ms（缓存命中）
- Agent查询吞吐量：> 1000 TPS
```

**测试用例**：
```
测试用例1：按Agent ID查询
- 输入：存在的Agent ID
- 预期：返回Agent详情

测试用例2：按Agent类型查询
- 输入：Agent类型
- 预期：返回该类型的Agent列表

测试用例3：Agent不存在
- 输入：不存在的Agent ID
- 预期：返回404错误
```

---

#### 3.1.8 P0-008：Token缓存管理

**功能描述**：
支持Token缓存，提升验证性能。

**业务规则**：
1. Token缓存到Redis，Key为token:{tokenId}
2. Token缓存包含Token完整信息
3. Token缓存过期时间与Token有效期一致
4. Token状态变更时刷新缓存
5. Token撤销时删除缓存

**缓存设计**：
```
缓存Key设计：
1. Token缓存：
   - Key: token:{tokenId}
   - Value: TaskToken JSON
   - TTL: 与Token有效期一致

2. Cookie缓存：
   - Key: cookie:{tokenId}
   - Value: encryptedCookie
   - TTL: 与Token有效期一致

3. Session缓存：
   - Key: session:{sessionId}
   - Value: UserAgentSession JSON
   - TTL: 与Session有效期一致

4. Agent缓存：
   - Key: agent:{agentId}
   - Value: AgentRegistration JSON
   - TTL: 1小时
```

**流程设计**：
```
流程1：Token缓存写入流程
1. Token生成成功
2. 序列化Token为JSON
3. 写入Redis缓存（Key: token:{tokenId})
4. 设置TTL为Token有效期

流程2：Token缓存查询流程
1. Token验证请求
2. 先查询Redis缓存
3. 缓存命中则返回缓存数据
4. 缓存未命中则查询MySQL
5. 查询成功则写入缓存

流程3：Token缓存删除流程
1. Token撤销成功
2. 删除Redis缓存（Key: token:{tokenId})
3. 删除Redis缓存（Key: cookie:{tokenId})
```

**性能要求**：
- 缓存命中率：> 95%
- 缓存查询延迟：< 5ms

---

#### 3.1.9 P0-009：Agent入口代理路由

**功能描述**：
对Web Copilot暴露注册Agent的统一入口，代理用户请求到Agent真实服务。运行时由Agent Gateway完成Cookie到任务级Token的转换、委托授权检查、Token注入、协议适配和路由转发，避免Copilot直接感知Agent真实地址，也避免Agent主动调用Token生成等网关内部API。

**业务规则**：
1. Copilot配置的Agent调用地址必须是Agent Gateway返回的gatewayEndpoint
2. Agent真实agentEndpoint只保存在Agent注册信息中，不暴露给浏览器端
3. 网关接收请求后必须校验用户Session和Cookie有效性
4. 网关必须检查目标Agent注册状态为ACTIVE
5. 网关必须检查用户对目标Agent的委托授权；未授权时通过Copilot触发授权流程
6. 授权完成后，网关生成与Task绑定的Token，并注入到转发给Agent的请求中
7. Agent不直接调用Token生成、Cookie换取、委托授权等网关内部API
8. 对OpenClaw等开源Agent，优先通过协议适配器、请求头映射或Sidecar方式接入，保持Agent主体独立演进

**接口设计**：
```
POST /agents/{agentId}/invoke
Content-Type: application/json
Cookie: IDaaS_SSO_Cookie_Value

Request Body:
{
  "taskId": "task-uuid-001",
  "sessionId": "session-uuid-001",
  "message": "查询当前组织的数据源",
  "context": {
    "source": "web-copilot"
  }
}

Forward To Agent:
POST {agentEndpoint}
Authorization: Bearer {taskToken}
X-Agent-Task-Context: {"taskId":"task-uuid-001","agentId":"agent-001","sessionId":"session-uuid-001"}

Response:
{
  "taskId": "task-uuid-001",
  "agentId": "agent-001",
  "result": {
    "type": "agent_response",
    "content": "..."
  }
}
```

**流程设计**：
```
流程1：Copilot到Agent代理调用流程
1. Copilot根据注册Agent列表获取gatewayEndpoint
2. 用户在Copilot提交任务请求
3. Copilot调用/agents/{agentId}/invoke，请求携带用户Cookie和任务上下文
4. Agent Gateway解析Cookie并校验Session
5. Agent Gateway查询Agent注册信息，获取agentEndpoint和protocolAdapter
6. Agent Gateway查询策略中心，检查用户到目标Agent的委托授权
7. 如需授权，Agent Gateway通过Copilot触发授权/确认，授权完成后继续
8. Agent Gateway生成任务级Token，绑定taskId、sessionId、userId、agentId和委托链
9. Agent Gateway按protocolAdapter注入Token和任务上下文
10. Agent Gateway转发原始用户请求到agentEndpoint
11. Agent执行业务逻辑，调用MCP工具时携带网关注入的Token
12. Agent返回结果给Agent Gateway
13. Agent Gateway返回结果给Copilot
14. 任务结束后Token失效，记录审计日志

性能要求：
- 入口代理额外延迟：< 30ms（不含目标Agent处理时间）
- 授权已存在场景成功率：> 99.9%
- Agent真实地址不出现在浏览器URL、前端配置或响应体中
```

**测试用例**：
```
测试用例1：Copilot通过网关调用已授权Agent
- 输入：有效Cookie、ACTIVE Agent、已有委托授权
- 预期：网关生成Token并转发到真实Agent，Agent返回结果

测试用例2：Copilot调用未授权Agent
- 输入：有效Cookie、ACTIVE Agent、无委托授权
- 预期：网关触发授权流程，授权完成后继续转发

测试用例3：Agent真实地址变更
- 输入：Agent更新agentEndpoint，Copilot仍调用gatewayEndpoint
- 预期：Copilot无感知，网关按新地址转发成功

测试用例4：开源Agent轻量接入
- 输入：Agent只支持标准Authorization Header
- 预期：网关通过protocolAdapter注入Bearer Token，Agent无需调用网关Token API
```

---

### 3.2 P1重要功能详细设计

#### 3.2.1 P1-001：A2A路由

**功能描述**：
支持Agent通过Agent Gateway调用其他注册Agent。Agent Gateway验证调用方Token和A2A委托权限，生成被调Agent的任务级Token，并按被调Agent注册信息代理转发请求，保证调用方不直接依赖目标Agent真实地址。

**业务规则**：
1. A2A调用必须验证委托链
2. A2A调用必须生成新的Token（包含完整委托链）
3. A2A调用必须记录审计日志
4. A2A调用支持多级委托链传递
5. A2A调用方只能访问Agent Gateway统一入口，不能直接访问被调Agent真实agentEndpoint

**接口设计**：
```
POST /api/v1/a2a/route
Content-Type: application/json

Request Body:
{
  "callerAgentId": "agent-001",
  "calleeAgentId": "agent-002",
  "callerTokenId": "token-uuid-001",
  "delegationId": "del-uuid-002",
  "payload": {
    "taskDescription": "将数据源列表同步到IDATA",
    "inputData": {}
  }
}

Response:
{
  "routeId": "route-uuid-001",
  "calleeAgentId": "agent-002",
  "result": {
    "type": "agent_response",
    "content": "..."
  },
  "delegationChain": [
    {
      "from": "user:l00867517",
      "to": "agent:agent-001",
      "delegationId": "del-uuid-001"
    },
    {
      "from": "agent:agent-001",
      "to": "agent:agent-002",
      "delegationId": "del-uuid-002"
    }
  ]
}
```

**流程设计**：
```
流程1：A2A路由流程
1. 接收A2A路由请求
2. 验证callerAgentId存在性
3. 验证calleeAgentId存在性
4. 验证callerTokenId有效性
5. 提取callerToken中的委托链
6. 验证delegationId有效性（callerAgentId委托给calleeAgentId）
7. 构建新的委托链（追加新的委托关系）
8. 生成calleeToken（包含完整委托链）
9. 查询calleeAgent注册信息，获取agentEndpoint和protocolAdapter
10. 按protocolAdapter注入calleeToken和任务上下文
11. 代理转发payload到calleeAgent真实服务
12. 接收calleeAgent响应
13. 记录审计日志到Elasticsearch
14. 返回calleeAgent执行结果给callerAgent

性能要求：
- A2A路由额外延迟：< 30ms（不含被调Agent处理时间）
- A2A路由吞吐量：> 500 TPS
```

**测试用例**：
```
测试用例1：正常A2A路由
- 输入：有效的A2A路由参数
- 预期：路由成功，网关生成calleeToken并代理调用calleeAgent，返回执行结果

测试用例2：callerAgent不存在
- 输入：不存在的callerAgentId
- 预期：返回404错误

测试用例3：calleeAgent不存在
- 输入：不存在的calleeAgentId
- 预期：返回404错误

测试用例4：callerToken无效
- 输入：无效的callerTokenId
- 预期：返回400错误

测试用例5：delegationId无效
- 输入：无效的delegationId
- 预期：返回400错误
```

---

#### 3.2.2 P1-002：实时确认处理

**功能描述**：
处理实时确认请求，转发确认消息给用户。

**业务规则**：
1. 实时确认请求来自MCP Gateway
2. 实时确认消息包含Agent、工具、操作信息
3. 实时确认消息推送给用户（通过Web Copilot）
4. 实时确认超时时间可配置（默认30秒）

**接口设计**：
```
POST /api/v1/confirmations/request
Content-Type: application/json

Request Body:
{
  "tokenId": "token-uuid-001",
  "toolName": "datasource_delete",
  "operation": "删除数据源: test-datasource",
  "confirmationTimeout": 30
}

Response:
{
  "confirmationId": "conf-uuid-001",
  "status": "PENDING",
  "createTime": "2026-06-04T10:00:00Z"
}
```

**流程设计**：
```
流程1：实时确认请求流程
1. 接收实时确认请求
2. 验证Token有效性
3. 提取userId和agentId
4. 生成确认ID（UUID）
5. 构建确认消息
6. 推送确认消息给用户（通过WebSocket或HTTP）
7. 设置确认超时定时器
8. 记录审计日志
9. 返回确认ID和状态

性能要求：
- 实时确认请求延迟：< 10ms
- 实时确认推送延迟：< 100ms
```

---

#### 3.2.3 P1-003：实时确认响应

**功能描述**：
接收用户确认响应，转发给MCP Gateway。

**业务规则**：
1. 用户确认响应包含确认ID和决策（允许、拒绝）
2. 确认响应转发给MCP Gateway
3. 确认响应记录审计日志
4. 确认超时自动拒绝

**接口设计**：
```
POST /api/v1/confirmations/respond
Content-Type: application/json

Request Body:
{
  "confirmationId": "conf-uuid-001",
  "decision": "ALLOWED",
  "userId": "l00867517"
}

Response:
{
  "confirmationId": "conf-uuid-001",
  "status": "COMPLETED",
  "decision": "ALLOWED"
}
```

**流程设计**：
```
流程1：实时确认响应流程
1. 接收用户确认响应
2. 验证确认ID存在性
3. 验证确认状态为PENDING
4. 更新确认状态为COMPLETED
5. 记录确认决策
6. 转发确认响应给MCP Gateway
7. 记录审计日志
8. 返回确认结果

性能要求：
- 实时确认响应延迟：< 10ms
- 确认响应转发延迟：< 20ms
```

---

### 3.3 P2-P3功能详细设计

由于篇幅限制，P2和P3功能的详细设计省略，参考策略中心的P2-P3设计模式。

---

## 4. 数据结构设计

### 4.1 数据库表设计

#### 4.1.1 Task Token表（task_token）

```sql
CREATE TABLE task_token (
    token_id VARCHAR(64) PRIMARY KEY COMMENT 'Token ID',
    task_id VARCHAR(64) NOT NULL COMMENT 'Task ID',
    session_id VARCHAR(64) NOT NULL COMMENT 'Session ID',
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    agent_id VARCHAR(64) NOT NULL COMMENT 'Agent ID',
    environment_context JSON NOT NULL COMMENT '环境信息',
    delegation_chain JSON COMMENT '委托链',
    token_value TEXT NOT NULL COMMENT 'Token值（JWT）',
    create_time DATETIME NOT NULL COMMENT '创建时间',
    expiry_time DATETIME NOT NULL COMMENT '过期时间',
    status VARCHAR(32) NOT NULL COMMENT '状态：ACTIVE/EXPIRED/REVOKED/ABUSED',
    update_time DATETIME COMMENT '更新时间',
    INDEX idx_task_id (task_id),
    INDEX idx_session_id (session_id),
    INDEX idx_user_agent (user_id, agent_id),
    INDEX idx_status (status),
    INDEX idx_expiry_time (expiry_time)
) COMMENT 'Task Token表';
```

#### 4.1.2 User Agent Session表（user_agent_session）

```sql
CREATE TABLE user_agent_session (
    session_id VARCHAR(64) PRIMARY KEY COMMENT 'Session ID',
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    agent_id VARCHAR(64) NOT NULL COMMENT 'Agent ID',
    encrypted_cookie TEXT NOT NULL COMMENT '加密Cookie',
    status VARCHAR(32) NOT NULL COMMENT '状态：ACTIVE/EXPIRED/CLOSED',
    create_time DATETIME NOT NULL COMMENT '创建时间',
    expiry_time DATETIME NOT NULL COMMENT '过期时间',
    update_time DATETIME COMMENT '更新时间',
    created_by VARCHAR(64) COMMENT '创建人',
    INDEX idx_user_agent (user_id, agent_id),
    INDEX idx_status (status),
    INDEX idx_expiry_time (expiry_time)
) COMMENT 'User Agent Session表';
```

#### 4.1.3 Agent Registration表（agent_registration）

```sql
CREATE TABLE agent_registration (
    agent_id VARCHAR(64) PRIMARY KEY COMMENT 'Agent ID',
    agent_name VARCHAR(128) NOT NULL COMMENT 'Agent名称',
    agent_type VARCHAR(64) NOT NULL COMMENT 'Agent类型',
    agent_description VARCHAR(512) COMMENT 'Agent描述',
    agent_endpoint VARCHAR(512) NOT NULL COMMENT 'Agent真实服务地址',
    gateway_endpoint VARCHAR(512) NOT NULL COMMENT 'Agent Gateway暴露入口',
    protocol_adapter JSON COMMENT '协议适配与Token注入配置',
    recommended_scenarios JSON COMMENT '推荐场景',
    status VARCHAR(32) NOT NULL COMMENT '状态：ACTIVE/INACTIVE/DELETED',
    create_time DATETIME NOT NULL COMMENT '创建时间',
    update_time DATETIME NOT NULL COMMENT '更新时间',
    created_by VARCHAR(64) COMMENT '创建人',
    INDEX idx_agent_type (agent_type),
    INDEX idx_status (status)
) COMMENT 'Agent Registration表';
```

#### 4.1.4 Confirmation Record表（confirmation_record）

```sql
CREATE TABLE confirmation_record (
    confirmation_id VARCHAR(64) PRIMARY KEY COMMENT '确认ID',
    token_id VARCHAR(64) NOT NULL COMMENT 'Token ID',
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    agent_id VARCHAR(64) NOT NULL COMMENT 'Agent ID',
    tool_name VARCHAR(128) NOT NULL COMMENT '工具名称',
    operation TEXT COMMENT '操作描述',
    status VARCHAR(32) NOT NULL COMMENT '状态：PENDING/COMPLETED/TIMEOUT',
    decision VARCHAR(32) COMMENT '决策：ALLOWED/REJECTED',
    create_time DATETIME NOT NULL COMMENT '创建时间',
    complete_time DATETIME COMMENT '完成时间',
    INDEX idx_token_id (token_id),
    INDEX idx_user_agent (user_id, agent_id),
    INDEX idx_status (status)
) COMMENT 'Confirmation Record表';
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
| API-001 | /api/v1/tokens/generate | POST | 生成Token | P0-001 |
| API-002 | /api/v1/tokens/verify | POST | 验证Token | P0-002 |
| API-003 | /api/v1/sessions/create | POST | 创建Session | P0-003 |
| API-004 | /api/v1/sessions/{sessionId} | GET | 查询Session详情 | P0-004 |
| API-005 | /api/v1/sessions/{sessionId}/status | PUT | 更新Session状态 | P0-004 |
| API-006 | /api/v1/sessions/{sessionId}/extend | PUT | 延长Session有效期 | P0-004 |
| API-007 | /api/v1/cookies/exchange | POST | Cookie换取 | P0-005 |
| API-008 | /api/v1/agents/register | POST | Agent注册 | P0-006 |
| API-009 | /api/v1/agents/{agentId} | GET | 查询Agent详情 | P0-007 |
| API-010 | /api/v1/agents | GET | 查询Agent列表 | P0-007 |
| API-011 | /agents/{agentId}/invoke | POST | Copilot到Agent代理调用 | P0-009 |
| API-012 | /api/v1/a2a/route | POST | A2A代理路由 | P1-001 |
| API-013 | /api/v1/confirmations/request | POST | 实时确认请求 | P1-002 |
| API-014 | /api/v1/confirmations/respond | POST | 实时确认响应 | P1-003 |

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
