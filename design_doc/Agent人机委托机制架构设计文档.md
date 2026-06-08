# Agent人机委托机制架构设计文档

| 项目 | 内容 |
| --- | --- |
| 文档版本 | v1.0 |
| 创建日期 | 2026-05-28 |
| 作者 | CodeAgent |
| 状态 | 设计完成 |

---

## 目录

1. [背景与问题](#1-背景与问题)
2. [设计目标](#2-设计目标)
3. [整体架构](#3-整体架构)
4. [核心数据结构](#4-核心数据结构)
5. [核心流程设计](#5-核心流程设计)
6. [工具标签策略设计](#6-工具标签策略设计)
7. [A2A委托链传递](#7-a2a委托链传递)
8. [委托配置流程](#8-委托配置流程)
9. [Agent注册管理](#9-agent注册管理)
10. [审计日志与监控](#10-审计日志与监控)
11. [异常处理与安全防护](#11-异常处理与安全防护)
12. [系统部署与扩展性](#12-系统部署与扩展性)
13. [关键技术决策](#13-关键技术决策)
14. [总结与展望](#14-总结与展望)
15. [附录](#附录)
16. [场景测试用例设计](#15-场景测试用例设计)
17. [验收标准](#16-验收标准)
18. [总结](#17-总结)

---

## 1. 背景与问题

### 1.1 问题背景

当前系统中，用户通过Web Copilot与Agent进行人机交互，Agent调用MCP工具执行任务。现有机制存在以下问题：

**问题1：Cookie透传导致权限过大**
- 用户通过IDaaS SSO登录后，Cookie直接透传到Agent
- Agent拥有用户的全部权限，可能滥用权限执行未授权操作
- 用户无法限制Agent的权限范围

**问题2：委托机制缺失**
- 用户无法精细控制Agent可执行的操作
- 用户无法限制Agent可访问的数据范围
- Agent执行敏感操作时，无用户确认机制

**问题3：A2A调用权限传递问题**
- Agent调用Agent（A2A模式）时，权限传递机制不清晰
- 委托关系在A2A调用链中如何传递未定义
- 权限范围在A2A调用中如何限定未明确

**问题4：环境维度缺失**
- Agent调用工具时，环境维度（如appName、tenantId）由Agent传入
- Agent可能传入错误的环境维度，导致数据越权访问
- 缺乏强制限定机制，安全性不足

### 1.2 解决方案目标

设计一个安全的Agent人机委托机制，解决上述问题：

- **Cookie安全转换**：将Cookie转换为临时Token，避免Cookie透传
- **精细化委托控制**：用户可配置Agent的权限范围和数据范围
- **A2A权限传递**：支持Agent调用Agent的委托链传递
- **环境维度强制注入**：从Token中自动注入环境维度，防止数据越权
- **实时确认机制**：高危操作强制用户实时确认

---

## 2. 设计目标

### 2.1 核心设计目标

**安全性目标**：
- Cookie不透传到Agent，防止权限滥用
- Token任务级绑定，防止Token滥用
- 环境维度强制注入，防止数据越权
- 委托链完整传递，防止权限泄露
- 异常调用检测，实时安全防护

**灵活性目标**：
- 工具标签策略，灵活配置权限范围
- Agent分类推荐策略，简化用户配置
- A2A权限继承，支持复杂协作场景
- 实时确认机制，平衡安全与体验

**可扩展性目标**：
- 新Agent通过注册接入网关，Copilot只配置网关暴露的Agent入口
- Agent不直接调用Token生成、委托授权等网关内部API，运行时由网关代理转发
- 新工具注册自动同步
- 策略模板库可扩展
- 多级A2A支持

**可维护性目标**：
- 分层架构，职责清晰
- 审计日志完整，便于追溯
- 监控告警完善，快速定位问题
- 高可用部署，故障自动转移

---

## 3. 整体架构

### 3.1 分层委托架构

系统采用三层架构设计：

```
┌─────────────────────────────────────────────────────────────┐
│                      Agent Gateway层                          │
│                    (Token管理层)                               │
│                                                               │
│  核心职责：                                                    │
│  - Cookie→Token转换                                           │
│  - Token生命周期管理（生成、失效）                             │
│  - Session会话管理（保活、Cookie存储）                         │
│  - 委托链构建与传递                                            │
│  - Agent注册管理                                               │
│  - Copilot→Agent入口代理路由                                    │
│  - A2A调用路由                                                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      MCP Gateway层                            │
│                    (鉴权执行层)                                │
│                                                               │
│  核心职责：                                                    │
│  - 工具调用鉴权                                                │
│  - Token解析与验证                                             │
│  - 权限查询代理（调用策略中心）                                │
│  - Cookie还原调用传统API                                       │
│  - 数据范围注入                                                │
│  - 审计日志记录                                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      策略中心层                               │
│                    (权限管理层)                                │
│                                                               │
│  核心职责：                                                    │
│  - 委托配置管理服务                                            │
│  - 委托策略存储                                                │
│  - 委托链解析引擎                                              │
│  - 委托生命周期管理                                            │
│  - 实时确认服务                                                │
│  - 策略变更通知                                                │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 整体调用流程

```
用户 → IDaaS SSO登录 → Web Copilot → Agent Gateway（Agent统一入口）
                                        ↓
                                   Cookie转Token
                                        ↓
                            委托授权检查/必要时发起授权
                                        ↓
                          生成Task级Token并转发到目标Agent
                                        ↓
                                   Agent执行任务
                                        ↓
                                   调用MCP工具（携带Token）
                                        ↓
                                   MCP Gateway鉴权
                                        ↓
                                   查询策略中心获取权限
                                        ↓
                                   Cookie还原调用传统API
                                        ↓
                                   返回结果给Agent
                                        ↓
                                   Agent返回结果给用户
                                        ↓
                                   Task结束，Token失效
```

---

## 4. 核心数据结构

### 4.1 Session会话结构

Session与用户Copilot会话绑定，保存原始Cookie，保持长连接。

```json
{
  "sessionId": "session-uuid",
  "userId": "l00867517",
  "userAccount": "用户账号",
  "cookieEncrypted": "AES加密的原始Cookie",
  "createdAt": "2026-05-28T10:00:00Z",
  "lastActiveAt": "2026-05-28T10:30:00Z",
  "status": "ACTIVE|EXPIRED|TERMINATED"
}
```

**字段说明**：
- `sessionId`: 会话唯一标识
- `userId`: 用户ID
- `cookieEncrypted`: 加密存储的原始Cookie（用于还原调用传统API）
- `lastActiveAt`: 最后活跃时间（用于保活判断）
- `status`: 会话状态

### 4.2 Task任务结构

Task与用户单个请求绑定，包含多个工具调用。

```json
{
  "taskId": "task-uuid",
  "sessionId": "session-uuid",
  "userId": "l00867517",
  "agentId": "agent-001",
  "request": "查询当前组织的数据源",
  "status": "RUNNING|COMPLETED|FAILED|TERMINATED",
  "createdAt": "2026-05-28T10:00:00Z",
  "completedAt": "2026-05-28T10:05:00Z"
}
```

**字段说明**：
- `taskId`: 任务唯一标识
- `sessionId`: 关联Session
- `request`: 用户原始请求
- `status`: 任务状态

### 4.3 Token结构

Token与Task绑定，任务开始生成，任务结束失效。

```json
{
  "tokenId": "token-uuid",
  "taskId": "task-uuid",
  "sessionId": "session-uuid",

  "userId": "l00867517",
  "userAccount": "用户账号",

  "agentId": "agent-001",
  "agentType": "DATA_QUERY",

  "environmentContext": {
    "appName": "C00001-O0023",
    "tenantId": "1111111111111111",
    "region": "dgg",
    "orgCode": "C00001-O0023",
    "orgName": "PT Huawei Tech Investment",
    "country": "ID",
    "businessGroup": "03"
  },

  "delegationChain": [
    {
      "from": "user:l00867517",
      "to": "agent:agent-001",
      "delegationId": "del-123"
    }
  ],

  "createdAt": "2026-05-28T10:00:00Z",
  "status": "ACTIVE|EXPIRED"
}
```

**字段说明**：
- `tokenId`: Token唯一标识
- `taskId`: 关联Task（Token与Task绑定）
- `sessionId`: 关联Session（用于Cookie还原）
- `userId`: 用户ID
- `agentId`: Agent ID
- `agentType`: Agent类型（DATA_QUERY、DATA_SYNC等）
- `environmentContext`: 用户环境维度（appName、tenantId等，从用户身份提取）
- `delegationChain`: 委托链（支持A2A传递）
- `status`: Token状态

**关键设计点**：
- Token与Task绑定，任务结束立即失效
- Token包含环境维度，MCP Gateway自动注入到工具参数
- Token包含委托链，支持A2A调用链传递

### 4.4 委托策略结构

委托策略记录用户对Agent的权限委托关系。

```json
{
  "delegationId": "del-123",
  "userId": "l00867517",
  "agentId": "agent-001",
  "delegationType": "SESSION|TEMPORARY|LONG_TERM",

  "permissionScope": {
    "allowedTags": ["B2B", "READ_ONLY", "DATA_ACCESS"],
    "deniedTags": ["DELETE", "HIGH_RISK"],
    "dataRange": {
      "appName": "C00001-O0023",
      "datasourceIds": ["184", "305"],
      "objectIds": ["989", "990"]
    },
    "a2aPermission": {
      "enabled": true,
      "allowedTargetAgents": ["agent-002", "agent-003"],
      "inheritPermission": true,
      "restrictedTags": []
    }
  },

  "createdAt": "2026-05-28T10:00:00Z",
  "expiresAt": "2026-05-28T11:00:00Z",
  "status": "ACTIVE|EXPIRED|REVOKED"
}
```

**字段说明**：
- `delegationId`: 委托唯一标识
- `userId`: 用户ID
- `agentId`: Agent ID
- `delegationType`: 委托类型（长期、临时、会话级）
- `permissionScope`: 权限范围
  - `allowedTags`: 允许的工具标签组合
  - `deniedTags`: 禁止的工具标签
  - `dataRange`: 数据范围限定
  - `a2aPermission`: A2A权限配置
- `expiresAt`: 过期时间（TEMPORARY类型有效）
- `status`: 委托状态

### 4.5 Agent注册结构

Agent注册信息，包含Agent身份、能力声明、推荐策略。

```json
{
  "agentId": "agent-001",
  "agentName": "数据查询Agent",
  "agentType": "DATA_QUERY",
  "agentDescription": "负责查询B2B数据源、连接器、数据对象",
  "agentVersion": "1.0.0",
  "agentEndpoint": "http://agent-001.internal:8080",
  "gatewayEndpoint": "https://agent-gateway.example.com/agents/agent-001/invoke",
  "protocolAdapter": {
    "requestMode": "HTTP_JSON",
    "tokenInjection": "AUTHORIZATION_BEARER",
    "contextHeader": "X-Agent-Task-Context"
  },

  "capabilities": {
    "supportedTags": ["B2B", "READ_ONLY", "DATA_ACCESS"],
    "supportedTools": ["datasource_list", "connector_list", "object_list"],
    "a2aEnabled": true,
    "a2aTargetAgents": ["agent-002", "agent-003"]
  },

  "recommendedPolicy": {
    "policyTemplateId": "B2B_READ_ONLY",
    "templateName": "B2B只读类策略",
    "allowedTags": ["B2B", "READ_ONLY", "DATA_ACCESS"],
    "deniedTags": ["DELETE", "HIGH_RISK"]
  },

  "defaultPolicy": {
    "enabled": true,
    "policyTemplateId": "B2B_READ_ONLY"
  },

  "owner": "B2B平台团队",
  "contact": "b2b-team@huawei.com",
  "createdAt": "2026-05-01T00:00:00Z",
  "updatedAt": "2026-05-28T00:00:00Z",
  "status": "REGISTERED|DISABLED|DEPRECATED"
}
```

**字段说明**：
- `agentId`: Agent唯一标识
- `agentType`: Agent类型（DATA_QUERY、DATA_SYNC、TASK_MANAGE等）
- `agentEndpoint`: Agent真实服务地址，只由Agent Gateway调用，不暴露给Copilot前端
- `gatewayEndpoint`: Agent Gateway对Copilot暴露的统一调用地址
- `protocolAdapter`: 网关转发到Agent时的协议适配和Token注入方式
- `capabilities`: Agent能力声明
- `recommendedPolicy`: 推荐策略模板
- `defaultPolicy`: 默认策略（无委托时应用）
- `status`: Agent状态

---

## 5. 核心流程设计

### 5.1 流程1：Copilot通过Agent Gateway调用Agent

**触发场景**：用户在Copilot提交任务请求。Copilot配置的是Agent Gateway上注册Agent对应的统一入口，而不是Agent真实服务地址。

**详细步骤**：

1. **用户登录建立Session**：
   - 用户通过IDaaS SSO登录
   - Copilot与用户建立长会话（Session）
   - Session保存Cookie（加密存储）
   - Session保活机制：定期刷新Cookie

2. **用户提交任务请求**：
   - 用户在Copilot输入："查询当前组织的数据源"
   - Copilot根据已注册Agent列表选择目标agentId
   - Copilot向Agent Gateway统一入口发送请求，例如：`POST /agents/{agentId}/invoke`
   - Copilot创建Task：
     - taskId：UUID
     - sessionId：关联当前Session
     - request：用户原始请求
     - status：RUNNING

3. **Agent Gateway拦截并完成运行时准备**：
   - 解析Session，提取用户身份和Cookie
   - 检查Agent注册状态
   - 根据agentId查询真实agentEndpoint和协议适配配置
   - 查询策略中心，检查用户是否已对目标Agent完成委托授权
   - 如果尚未授权或权限不足，网关通过Copilot触发授权/确认流程，授权完成后继续执行
   - 生成Token：
     - tokenId：UUID
     - taskId：关联Task
     - sessionId：关联Session
     - userId、agentId：从Session提取
     - environmentContext：从用户身份提取（appName等）
     - delegationChain：包含用户到目标Agent的委托关系
     - status：ACTIVE

4. **Agent Gateway代理转发到目标Agent**：
   - Agent Gateway不把Token生成API暴露给Copilot或Agent编排使用
   - Agent Gateway按注册配置把用户原始请求转发到agentEndpoint
   - 转发时注入任务级Token和上下文，例如：
     - `Authorization: Bearer <task_token>`
     - `X-Agent-Task-Context: {"taskId":"task-001","agentId":"agent-001"}`
   - 对OpenClaw等开源Agent，可通过网关适配器把Token注入到其已有认证头或任务上下文字段，尽量保持Agent主体独立演进
   - Agent只需要按约定消费运行时凭证，后续MCP工具调用携带该Token，不需要主动调用Agent Gateway的Token生成API

5. **任务执行过程**：
   - Agent调用多个MCP工具（携带同一Token）
   - MCP Gateway每次验证Token是否属于当前Task
   - Token在任务执行期间一直有效

6. **任务结束Token失效**：
   - Agent完成任务，返回结果给用户
   - Task状态更新为COMPLETED
   - Token状态更新为EXPIRED（立即失效）
   - Token失效后无法再用于任何工具调用

**关键设计点**：
- Token与Task绑定，任务级生命周期
- Cookie只在Agent Gateway使用，不传递到Agent或MCP Gateway
- Token包含环境维度，自动注入到工具参数
- Copilot前端与Agent真实地址解耦，只访问Agent Gateway上的注册Agent入口
- Agent与网关内部API解耦，不直接编排Token生成、Cookie换取、委托授权等接口
- 对开源Agent优先采用网关协议适配、请求头注入、Sidecar/Adapter等轻量方式接入
- 任务结束立即失效，无需刷新机制

### 5.2 流程2：MCP工具调用鉴权

**触发场景**：Agent执行任务过程中调用MCP工具

**详细步骤**：

1. **Agent调用MCP工具**：
   - Agent构建请求：
     - Header：Token
     - Body：工具参数
   - 发送到MCP Gateway

2. **MCP Gateway接收请求**：
   - 解析Header中的Token
   - 验证Token有效性：
     - Token状态是否ACTIVE
     - Task状态是否RUNNING
     - Session状态是否ACTIVE
   - 如果Token无效：返回401错误

3. **工具分类检查**：
   - MCP Gateway查询工具元数据
   - 根据工具标签分类：
     - 普通工具：无需委托
     - 敏感工具：需要委托
     - 高危工具：需要实时确认

4. **普通工具鉴权**：
   - 直接使用用户权限（无需委托）
   - 调用Agent Gateway换取Cookie
   - 使用还原的Cookie调用传统API
   - 返回结果给Agent

5. **敏感工具鉴权**：
   - 查询策略中心：用户是否委托该Agent
   - 如果无委托：返回403错误，提示需要委托
   - 如果有委托：
     - 检查委托权限范围（工具标签匹配）
     - 检查数据范围（appName、datasourceIds）
     - 如果权限不足：返回403错误
     - 如果权限充足：调用API，注入数据范围参数

6. **高危工具鉴权**：
   - 强制要求用户实时确认
   - 返回403错误：USER_CONFIRMATION_REQUIRED
   - 用户确认后创建临时委托
   - 重新调用工具

7. **调用传统API**：
   - 调用Agent Gateway换取Cookie
   - 使用还原的Cookie调用传统API
   - API返回结果

8. **记录审计日志**：
   - 记录：taskId、tokenId、工具名称、调用时间、调用结果

**关键设计点**：
- 工具分类标签化，不同类别鉴权逻辑不同
- Cookie还原机制，调用传统API
- 数据范围注入，强制限定数据访问

### 5.3 流程3：Cookie还原调用传统API（通过Agent Gateway换取）

**详细步骤**：

1. **MCP Gateway需要调用传统API**：
   - Agent调用MCP工具
   - MCP Gateway需要调用b2b-admin-service的API

2. **MCP Gateway调用Agent Gateway换取Cookie**：
   - MCP Gateway调用Agent Gateway接口：
     ```
     POST /agent-gateway/cookie/exchange
     {
       "tokenId": "token-uuid",
       "sessionId": "session-uuid"
     }
     ```
   - Header包含Token（用于验证请求合法性）

3. **Agent Gateway验证请求**：
   - Agent Gateway解析Token
   - 验证Token有效性（status=ACTIVE，taskId关联）
   - 验证sessionId与Token中的sessionId一致
   - 如果Token无效：返回401错误

4. **Agent Gateway查询Session获取Cookie**：
   - 查询Session表：
     ```sql
     SELECT cookieEncrypted FROM sessions
     WHERE sessionId = 'session-uuid' AND status = 'ACTIVE'
     ```
   - 解密Cookie（AES加密存储）
   - 检查Session是否活跃（lastActiveAt是否在保活时间内）

5. **Agent Gateway返回Cookie给MCP Gateway**：
   - 返回解密后的Cookie：
     ```json
     {
       "cookie": "原始Cookie",
       "expiresAt": "Cookie过期时间",
       "status": "SUCCESS"
     }
     ```
   - 或返回错误：
     ```json
     {
       "error": "SESSION_EXPIRED",
       "message": "Session已失效",
       "status": "FAILURE"
     }
     ```

6. **MCP Gateway构建API请求**：
   - 使用Agent Gateway返回的Cookie
   - Header：
     - Cookie：还原的原始Cookie
     - appName：从Token的environmentContext提取
     - userAccount：从Token提取
   - Body：工具参数

7. **调用传统API**：
   - 发送请求到b2b-admin-service
   - API验证Cookie（IDaaS验证）
   - API执行业务逻辑
   - API返回结果

8. **返回结果给Agent**：
   - MCP Gateway接收API结果
   - 封装为MCP工具响应格式
   - 返回给Agent

**关键设计点**：
- **职责分离**：Agent Gateway负责Session管理和Cookie换取，MCP Gateway不直接访问Session表
- **安全性提升**：Cookie只在Agent Gateway解密，MCP Gateway通过Token换取Cookie
- Cookie加密存储，保证安全
- 从Token的environmentContext提取appName，自动注入
- Agent Gateway可控制Cookie换取频率，防止滥用

### 5.4 流程4：数据范围注入

**触发场景**：用户委托Agent时指定了数据范围

**详细步骤**：

1. **查询委托数据范围**：
   - MCP Gateway调用策略中心：
     ```
     GET /delegation/permissions?userId=l00867517&agentId=agent-001&tool=datasource_list
     ```
   - 策略中心返回：
     ```json
     {
       "tools": ["datasource_list", "connector_list"],
       "dataRange": {
         "appName": "C00001-O0023",
         "datasourceIds": ["184", "305"],
         "objectIds": ["989", "990"]
       }
     }
     ```

2. **注入数据范围参数**：
   - MCP Gateway修改工具参数：
     - 原始参数：`{"keyword": "ISALES"}`
     - 注入后：`{"keyword": "ISALES", "appName": "C00001-O0023", "datasourceIds": ["184", "305"]}`
   - 强制限定数据范围（用户无法绕过）

3. **调用API时传递数据范围**：
   - Header：appName=C00001-O0023（强制限定）
   - API按数据范围过滤结果
   - 返回限定范围内的数据

**关键设计点**：
- 数据范围双重限定：Token环境维度 + 委托数据范围
- 强制注入，Agent无法修改
- API执行时双重过滤

---

## 6. 工具标签策略设计

### 6.1 工具标签体系

工具分类标签化，支持灵活的权限配置。

**工具元数据结构**：
```json
{
  "toolName": "datasource_list",
  "toolDescription": "查询数据源列表",
  "toolTags": [
    "B2B",           // 业务域标签
    "READ_ONLY",     // 操作类型标签
    "DATA_ACCESS",   // 功能分类标签
    "COMMON"         // 风险等级标签
  ],
  "requiredParams": ["appName"],
  "riskLevel": "LOW|MEDIUM|HIGH",
  "category": "QUERY|CREATE|UPDATE|DELETE"
}
```

**标签分类**：

**业务域标签**：
- B2B：B2B业务域工具
- ISALES：ISALES业务域工具
- IDATA：IDATA业务域工具
- ACC：验收中心业务域工具

**操作类型标签**：
- READ_ONLY：只读操作
- WRITE：写入操作
- DELETE：删除操作

**功能分类标签**：
- DATA_ACCESS：数据访问类
- TASK_MANAGE：任务管理类
- CONFIGURATION：配置管理类

**风险等级标签**：
- COMMON：普通风险
- SENSITIVE：敏感操作
- HIGH_RISK：高危操作

### 6.2 委托策略标签化配置

**委托策略基于标签组合**：
```json
{
  "permissionScope": {
    "allowedTags": ["B2B", "READ_ONLY", "DATA_ACCESS"],
    "deniedTags": ["DELETE", "HIGH_RISK"],
    "dataRange": {
      "appName": "C00001-O0023",
      "datasourceIds": ["184", "305"]
    }
  }
}
```

**策略配置示例**：
- B2B只读类策略：允许标签 [B2B, READ_ONLY]，禁止标签 [DELETE, HIGH_RISK]
- 数据访问类策略：允许标签 [DATA_ACCESS, READ_ONLY]，禁止标签 [DELETE]
- 任务管理类策略：允许标签 [TASK_MANAGE]，禁止标签 [DELETE]
- 完全信任策略：允许所有标签（用户完全信任Agent）

### 6.3 Agent分类与推荐策略

**Agent类型分类**：
- DATA_QUERY（数据查询类）：推荐策略：B2B只读类、数据访问类
- DATA_SYNC（数据同步类）：推荐策略：数据访问+写入类
- TASK_MANAGE（任务管理类）：推荐策略：任务管理类
- CONFIG_MANAGE（配置管理类）：推荐策略：配置类

**Agent注册信息包含推荐策略**：
```json
{
  "agentType": "DATA_QUERY",
  "recommendedPolicy": {
    "policyTemplateId": "B2B_READ_ONLY",
    "allowedTags": ["B2B", "READ_ONLY", "DATA_ACCESS"]
  }
}
```

**策略推荐流程**：
- 用户首次委托Agent时，权限中心根据Agent类型推荐策略模板
- 用户可以选择：接受推荐策略、自定义策略、完全信任策略

### 6.4 默认策略机制

**无委托时的默认策略**：
- 如果用户未配置委托：
  - 检查Agent是否有默认策略模板
  - 应用默认策略（如DATA_QUERY类Agent默认B2B只读类）
  - 如果Agent无默认策略：返回错误，提示需要委托

**策略模板库**：
```json
{
  "policyTemplateId": "B2B_READ_ONLY",
  "templateName": "B2B只读类策略",
  "allowedTags": ["B2B", "READ_ONLY", "DATA_ACCESS"],
  "deniedTags": ["DELETE", "HIGH_RISK"],
  "applicableAgentTypes": ["DATA_QUERY"]
}
```

---

## 7. A2A委托链传递

### 7.1 A2A调用场景

**用户请求**："查询数据源列表，并将结果同步到IDATA"

**Agent A（数据查询Agent）**：
- 负责查询数据源列表
- 发现需要同步功能，决定调用Agent B

**Agent B（数据同步Agent）**：
- 负责数据同步操作
- 需要写入权限（敏感操作）

### 7.2 A2A调用流程

**详细步骤**：

1. **Agent A执行查询任务**：
   - Agent A调用datasource_list工具
   - MCP Gateway鉴权通过（Agent A有B2B只读类策略）
   - 返回数据源列表结果

2. **Agent A决定调用Agent B**：
   - Agent A分析任务：需要同步数据
   - Agent A根据任务上下文选择目标Agent ID（agent-002）
   - Agent A准备通过Agent Gateway调用Agent B，不直接访问Agent B真实地址

3. **Agent A向Agent Gateway发起A2A调用**：
   - Agent A调用的仍然是Agent Gateway上的目标Agent入口，或统一A2A代理接口
   - Agent A不直接访问Agent B真实地址
   - Agent A发送请求：
     ```json
     {
       "targetAgentId": "agent-002",
       "taskDescription": "将数据源列表同步到IDATA",
       "inputData": {"datasourceList": [...]},
       "callerToken": "token-001"
     }
     ```

4. **Agent Gateway处理A2A调用**：
   - 解析Agent A的Token
   - 检查Agent A是否有A2A权限
   - 构建委托链：
     ```json
     {
       "delegationChain": [
         {"from": "user:l00867517", "to": "agent:agent-001", "delegationId": "del-123"},
         {"from": "agent:agent-001", "to": "agent:agent-002", "delegationId": "del-a2a-456"}
       ]
     }
     ```

5. **生成Agent B的Token**：
   - Agent Gateway为Agent B生成新Token：
     - tokenId：token-002
     - taskId：task-001（同一任务）
     - userId：l00867517
     - agentId：agent-002
     - callerAgentId：agent-001
     - delegationChain：完整委托链
     - environmentContext：继承Agent A的环境维度

6. **Agent Gateway调用Agent B**：
   - Agent Gateway根据Agent B注册信息找到真实agentEndpoint
   - Agent Gateway转发请求给Agent B
   - Header包含Agent B的Token和任务上下文

7. **Agent B调用MCP工具**：
   - Agent B调用datasource_create工具（写入操作）
   - MCP Gateway鉴权：
     - 解析委托链：user→agent-001→agent-002
     - 查询策略中心：检查A2A委托策略
     - 检查权限：Agent B是否有写入权限
   - 如果权限充足：调用API执行同步操作

8. **Agent B返回结果给Agent A**：
   - Agent B完成同步任务
   - 返回结果给Agent Gateway
   - Agent Gateway转发给Agent A
   - Agent A不需要感知Agent B真实部署地址变化

9. **Agent A返回最终结果给用户**：
   - Agent A整合查询结果和同步结果
   - 返回给用户

### 7.3 A2A委托策略配置

**用户配置A2A权限**：
```json
{
  "a2aPermission": {
    "enabled": true,
    "allowedTargetAgents": ["agent-002", "agent-003"],
    "inheritPermission": true,
    "restrictedTags": []
  }
}
```

**A2A权限继承规则**：
- `inheritPermission=true`：Agent B继承Agent A的权限范围
- `inheritPermission=false`：Agent B使用自己的默认策略或独立委托
- `restrictedTags`：A2A调用时额外限制的标签

---

## 8. 委托配置流程

### 8.1 权限中心功能模块

**权限中心架构**：
```
权限中心
├── 委托配置服务
│   ├── 委托策略配置界面
│   ├── Agent注册管理
│   ├── 工具标签管理
│   ├── 策略模板管理
│   └── 委托关系查询
│
├── 委托策略存储
│   ├── 用户委托策略库
│   ├── A2A委托策略库
│   ├── 策略模板库
│   └── 工具元数据库
│
├── 委托链解析引擎
│   ├── 委托链构建
│   ├── 权限继承计算
│   ├── 数据范围计算
│   └── 策略冲突检测
│
├── 委托生命周期管理
│   ├── 委托创建
│   ├── 委托更新
│   ├── 委托撤销
│   ├── 委托过期清理
│   └── 委托审计日志
│
└── 实时确认服务
    ├── 实时确认请求
    ├── 用户确认界面
    ├── 临时委托创建
    └── 确认结果通知
```

### 8.2 委托配置详细步骤

**步骤1：用户登录权限中心**：
- 用户通过IDaaS SSO登录
- 显示用户信息和已有委托列表

**步骤2：查看Agent列表**：
- 查询Agent注册表
- 显示可用Agent列表

**步骤3：选择Agent配置委托**：
- 选择Agent
- 显示Agent详情和推荐策略

**步骤4：选择委托类型**：
- 长期委托：长期有效
- 临时委托：指定有效期
- 会话级委托：仅当前会话有效

**步骤5：配置权限范围**：
- 选项1：接受推荐策略（快速配置）
- 选项2：自定义策略（精细配置）
- 选项3：完全信任策略（无限制）

**步骤6：配置A2A权限**：
- 是否允许A2A调用
- 允许调用的目标Agent
- 权限继承规则

**步骤7：保存委托策略**：
- 创建委托记录
- 存储到委托策略库

**步骤8：返回配置成功**：
- 提示用户配置成功
- Agent可以开始执行任务

### 8.3 委托生命周期管理

**长期委托**：
- 创建时：无过期时间
- 有效期：一直有效，直到用户主动撤销
- 过期清理：无
- 撤销：用户在权限中心主动撤销
- 审计：定期审计提醒

**临时委托**：
- 创建时：指定有效期
- 有效期：expiresAt字段记录过期时间
- 过期清理：定时扫描过期委托，更新status为EXPIRED
- 延期：用户可以在过期前延期
- 审计：过期时记录审计日志

**会话级委托**：
- 创建时：绑定sessionId
- 有效期：随Session结束失效
- 过期清理：Session失效时，更新status为EXPIRED
- 审计：Session结束时记录审计日志

### 8.4 实时确认流程

**触发场景**：Agent调用高危工具

**详细步骤**：

1. **Agent调用高危工具**：
   - MCP Gateway返回403错误：USER_CONFIRMATION_REQUIRED

2. **Agent提示用户确认**：
   - Agent返回特殊响应：
     ```json
     {
       "type": "USER_CONFIRMATION_REQUIRED",
       "toolName": "datasource_delete",
       "operation": "删除数据源ID=184",
       "confirmationUrl": "https://b2b.huawei.com/permission-center/confirmation?taskId=task-001"
     }
     ```

3. **用户点击确认链接**：
   - Copilot弹出确认窗口
   - 显示操作详情和风险提示
   - 用户选择：确认执行或拒绝执行

4. **用户确认执行**：
   - 权限中心创建临时委托：
     ```json
     {
       "delegationId": "del-temp-789",
       "delegationType": "TEMPORARY",
       "permissionScope": {
         "allowedTools": ["datasource_delete"],
         "dataRange": {"datasourceIds": ["184"]},
         "oneTimeOnly": true
       },
       "expiresAt": "2026-05-28T10:05:00Z"
     }
     ```
   - 通知Agent Gateway：委托已创建

5. **Agent重新调用工具**：
   - Agent重新调用工具
   - MCP Gateway鉴权通过
   - 执行操作

6. **临时委托自动失效**：
   - 操作完成后立即失效
   - 或5分钟后自动过期失效

---

## 9. Agent注册管理

### 9.1 Agent注册流程

**步骤1：开发者准备注册信息**：
- 准备Agent注册配置文件
- 包含：agentId、agentName、agentType、capabilities、recommendedPolicy

**步骤2：开发者提交注册请求**：
- 调用Agent Gateway注册接口：
  ```
  POST /agent-gateway/agent/register
  ```

**步骤3：Agent Gateway验证注册信息**：
- 验证agentId唯一性
- 验证agentType有效性
- 验证supportedTags有效性
- 验证recommendedPolicy有效性

**步骤4：Agent Gateway保存注册信息**：
- 存储到Agent注册表
- 同步到权限中心

**步骤5：返回注册成功**：
- 返回注册结果
- Agent可以在Copilot中被用户调用

### 9.2 Agent更新与注销

**Agent更新流程**：
- 开发者提交更新请求
- Agent Gateway验证更新
- 保存更新
- 通知权限中心同步
- 通知受影响的用户

**Agent注销流程**：
- 开发者提交注销请求
- Agent Gateway检查依赖关系
- 处理依赖关系（通知用户撤销委托）
- 更新Agent状态为DEPRECATED
- 清理Agent注册表
- 通知权限中心同步注销

### 9.3 Agent健康检查

**定时健康检查**：
- Agent Gateway定时检查Agent健康状态（每5分钟）
- 调用Agent健康检查接口：
  ```
  GET http://agent-001.internal:8080/health
  ```

**状态管理**：
- HEALTHY：Agent正常运行，可被调用
- UNHEALTHY：Agent异常，暂停调用
- DISABLED：Agent被管理员禁用
- DEPRECATED：Agent已注销，不可调用

**异常处理**：
- Agent状态为UNHEALTHY时：
  - Agent Gateway拒绝新的Token生成
  - 通知权限中心：Agent异常
  - 通知用户："Agent当前异常，暂时无法使用"
  - 记录异常日志

---

## 10. 审计日志与监控

### 10.1 审计日志结构

```json
{
  "auditId": "audit-uuid",
  "timestamp": "2026-05-28T10:00:00Z",

  "userId": "l00867517",
  "agentId": "agent-001",
  "sessionId": "session-uuid",
  "taskId": "task-uuid",
  "tokenId": "token-uuid",

  "operationType": "TOOL_CALL|DELEGATION_CREATE|DELEGATION_UPDATE|AGENT_REGISTER|TOKEN_GENERATE",

  "operationDetail": {
    "toolName": "datasource_list",
    "toolParams": {"keyword": "ISALES"},
    "delegationId": "del-123"
  },

  "result": "SUCCESS|FAILURE",
  "errorCode": "PERMISSION_DENIED|TOKEN_EXPIRED",
  "errorMessage": "用户未委托该Agent",

  "environmentContext": {
    "appName": "C00001-O0023",
    "region": "dgg",
    "ipAddress": "10.10.10.10"
  },

  "delegationChain": [
    {"from": "user:l00867517", "to": "agent:agent-001"}
  ]
}
```

### 10.2 审计日志记录场景

**关键操作审计**：
- Token生成审计：记录Token生成时间、用户身份、Agent身份
- 工具调用审计：记录工具名称、工具参数、调用结果、委托链
- 委托操作审计：记录委托创建、更新、撤销、过期
- A2A调用审计：记录调用者Agent、目标Agent、委托链传递
- 异常操作审计：记录权限拒绝、Token失效、异常调用检测
- 管理操作审计：记录Agent注册、注销、策略变更

### 10.3 监控指标设计

**Agent Gateway监控**：
- Token生成速率（每分钟）
- Token活跃数量
- Session活跃数量
- Task执行数量
- A2A调用频率
- 响应时间（P50、P95、P99）

**MCP Gateway监控**：
- 工具调用速率（每分钟）
- 工具调用成功率
- 鉴权失败率
- 权限查询延迟
- Cookie还原成功率
- 响应时间（P50、P95、P99）

**策略中心监控**：
- 委托查询速率
- 委托创建/更新/撤销频率
- 策略变更通知延迟
- 数据库查询延迟

**安全监控指标**：
- 权限拒绝频率（每分钟）
- Token失效频率
- 异常调用检测频率
- 高危工具调用频率
- A2A调用异常频率

### 10.4 实时告警机制

**安全告警**：
- 权限拒绝频率 > 10次/分钟：告警级别HIGH
- 异常调用检测 > 5次/分钟：告警级别HIGH
- 高危工具调用 > 3次/分钟：告警级别MEDIUM
- Token失效频率 > 20次/分钟：告警级别MEDIUM

**性能告警**：
- Token生成延迟 > 500ms：告警级别MEDIUM
- 工具调用延迟 > 2s：告警级别MEDIUM
- 策略查询延迟 > 300ms：告警级别LOW

**可用性告警**：
- Agent Gateway响应失败率 > 5%：告警级别HIGH
- MCP Gateway响应失败率 > 5%：告警级别HIGH
- 策略中心响应失败率 > 5%：告警级别HIGH

---

## 11. 异常处理与安全防护

### 11.1 认证异常处理

**Cookie无效或过期**：
- Agent Gateway检测Cookie无效
- 返回401错误：COOKIE_EXPIRED
- Copilot提示用户重新登录
- Session清理

**Token无效或过期**：
- MCP Gateway检测Token无效
- 返回401错误：TOKEN_EXPIRED
- Agent处理错误：判断任务是否正常结束
- 记录审计日志

### 11.2 鉴权异常处理

**无委托配置**：
- MCP Gateway检测无委托
- 返回403错误：DELEGATION_REQUIRED
- Agent提示用户配置委托
- 用户配置委托后重新调用

**委托权限不足**：
- MCP Gateway检测权限不足
- 返回403错误：PERMISSION_DENIED
- Agent提示用户升级权限
- 用户选择：升级委托或实时确认

### 11.3 A2A异常处理

**A2A调用权限不足**：
- Agent Gateway检测A2A权限不足
- 返回403错误：A2A_PERMISSION_DENIED
- Agent A提示用户配置A2A权限
- 用户配置后重新发起A2A调用

**委托链断裂**：
- MCP Gateway检测委托链断裂
- 返回403错误：DELEGATION_CHAIN_BROKEN
- Agent A提示用户刷新A2A委托
- 用户重新授权，委托链重建

### 11.4 系统异常处理

**Agent服务不可用**：
- Agent Gateway检测Agent不可用
- 返回503错误：AGENT_UNAVAILABLE
- Copilot提示用户稍后重试
- Agent Gateway记录异常，更新Agent状态为UNHEALTHY

**策略中心服务异常**：
- MCP Gateway检测策略中心异常
- 启用降级策略：使用本地缓存策略
- 或返回保守策略：拒绝所有敏感工具调用
- 发送告警

### 11.5 安全异常处理

**异常调用检测**：
- MCP Gateway检测异常调用（频率异常、参数异常、模式异常）
- 立即失效Token
- 终止Task
- 返回403错误：SECURITY_VIOLATION
- 通知用户："检测到异常行为，任务已终止"
- 记录安全审计日志
- 发送安全告警

**Token滥用检测**：
- MCP Gateway检测Token滥用（Token跨Task使用、Token被多个Agent使用）
- 立即失效Token
- 终止所有关联Task
- 返回403错误：TOKEN_ABUSE_DETECTED
- 记录安全审计日志

---

## 12. 系统部署与扩展性

### 12.1 部署架构

```
┌─────────────────────────────────────────────────────────────┐
│                          用户层                               │
│  Web Copilot (浏览器) ←→ IDaaS SSO (认证中心)                │
└─────────────────────────────────────────────────────────────┘
                            ↓ HTTPS
┌─────────────────────────────────────────────────────────────┐
│                       接入层                                  │
│  Agent Gateway (Nginx/Kong)                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓ Internal Network
┌─────────────────────────────────────────────────────────────┐
│                       服务层                                  │
│  MCP Gateway │ 策略中心 │ Agent集群                           │
└─────────────────────────────────────────────────────────────┘
                            ↓ Internal Network
┌─────────────────────────────────────────────────────────────┐
│                       数据层                                  │
│  MySQL │ Redis │ Elasticsearch │ 对象存储                    │
└─────────────────────────────────────────────────────────────┘
                            ↓ Internal Network
┌─────────────────────────────────────────────────────────────┐
│                       业务系统层                              │
│  b2b-admin-service │ 其他业务系统                             │
└─────────────────────────────────────────────────────────────┘
```

### 12.2 高可用部署

**Agent Gateway高可用**：
- 多实例部署（至少3个实例）
- Nginx/Kong负载均衡
- Redis共享Session和Token

**MCP Gateway高可用**：
- 多实例部署（至少3个实例）
- Kubernetes Service负载均衡
- 本地缓存+Redis分布式缓存

**策略中心高可用**：
- 多实例部署（至少2个实例）
- MySQL主从复制或集群
- Redis集群

### 12.3 性能优化

**Token生成性能优化**：
- Cookie解析缓存到Redis
- Token生成异步写入数据库
- 批量生成优化

**策略查询性能优化**：
- MCP Gateway本地缓存委托策略（有效期5分钟）
- 策略预加载
- 批量查询优化

**Cookie还原性能优化**：
- Redis缓存Session信息
- AES-256-GCM高效加密
- Redis连接池

**审计日志性能优化**：
- 异步写入Elasticsearch
- 批量写入（每100条或每秒）
- 索引优化（按时间分区）

### 12.4 扩展性设计

**Agent扩展性**：
- 新Agent注册无需修改代码
- Agent类型扩展：新增Agent类型，更新策略模板库
- Agent能力扩展：更新能力声明，自动同步

**工具扩展性**：
- 新工具注册：注册到MCP Gateway
- 工具标签扩展：新增标签类型
- 工具分类扩展：新增业务域标签

**策略扩展性**：
- 新策略模板：无需修改代码
- 策略规则扩展：支持时间限制、IP限制
- 委托类型扩展：新增项目级委托

**A2A扩展性**：
- 多级A2A：支持多级调用链
- A2A策略扩展：更复杂的权限配置
- A2A监控扩展：调用链可视化

### 12.5 数据库设计

**Session表**：
```sql
CREATE TABLE sessions (
  session_id VARCHAR(64) PRIMARY KEY,
  user_id VARCHAR(32) NOT NULL,
  user_account VARCHAR(64),
  cookie_encrypted TEXT,
  created_at TIMESTAMP,
  last_active_at TIMESTAMP,
  status VARCHAR(16),
  INDEX idx_user_id (user_id),
  INDEX idx_status (status)
);
```

**Task表**：
```sql
CREATE TABLE tasks (
  task_id VARCHAR(64) PRIMARY KEY,
  session_id VARCHAR(64),
  user_id VARCHAR(32),
  agent_id VARCHAR(32),
  request TEXT,
  status VARCHAR(16),
  created_at TIMESTAMP,
  completed_at TIMESTAMP,
  INDEX idx_session_id (session_id),
  INDEX idx_user_agent (user_id, agent_id)
);
```

**Token表**：
```sql
CREATE TABLE tokens (
  token_id VARCHAR(64) PRIMARY KEY,
  task_id VARCHAR(64),
  session_id VARCHAR(64),
  user_id VARCHAR(32),
  agent_id VARCHAR(32),
  delegation_chain JSON,
  environment_context JSON,
  created_at TIMESTAMP,
  status VARCHAR(16),
  INDEX idx_task_id (task_id),
  INDEX idx_status (status)
);
```

**Delegation表**：
```sql
CREATE TABLE delegations (
  delegation_id VARCHAR(64) PRIMARY KEY,
  user_id VARCHAR(32) NOT NULL,
  agent_id VARCHAR(32) NOT NULL,
  delegation_type VARCHAR(16),
  permission_scope JSON,
  created_at TIMESTAMP,
  expires_at TIMESTAMP,
  status VARCHAR(16),
  INDEX idx_user_agent (user_id, agent_id),
  INDEX idx_status (status)
);
```

**Agent表**：
```sql
CREATE TABLE agents (
  agent_id VARCHAR(32) PRIMARY KEY,
  agent_name VARCHAR(64),
  agent_type VARCHAR(32),
  agent_endpoint VARCHAR(256),
  capabilities JSON,
  recommended_policy JSON,
  default_policy JSON,
  owner VARCHAR(64),
  status VARCHAR(16),
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  INDEX idx_agent_type (agent_type),
  INDEX idx_status (status)
);
```

**ToolMetadata表**：
```sql
CREATE TABLE tool_metadata (
  tool_name VARCHAR(64) PRIMARY KEY,
  tool_description VARCHAR(256),
  tool_tags JSON,
  risk_level VARCHAR(16),
  category VARCHAR(16),
  required_params JSON,
  INDEX idx_tags (tool_tags)
);
```

---

## 13. 关键技术决策

### 13.1 动态权限获取（方案一）

**决策**：Token不含权限，每次工具调用动态查询策略中心

**理由**：
- 权限变更实时生效，无需刷新Token
- 灵活性高，支持复杂权限策略
- Token结构简单，减少复杂度

**性能优化**：
- MCP Gateway本地缓存委托策略（有效期5分钟）
- 策略预加载：Agent Gateway生成Token时预加载策略
- 批量查询：连续调用多个工具时批量查询策略

### 13.2 权限中心统一配置

**决策**：委托配置在权限中心统一管理

**理由**：
- 集中管理，便于审计
- 用户体验一致
- 支持复杂的策略配置

**流程**：
- 用户首次委托时，权限中心推荐策略模板
- 用户可以选择：接受推荐、自定义、完全信任
- 配置完成后，策略中心通知Agent Gateway

### 13.3 Token自动刷新+MCP Gateway提示刷新

**决策**：双重刷新机制

**理由**：
- Agent Gateway自动刷新：减少Agent处理负担
- MCP Gateway提示刷新：补充机制，防止遗漏

**实现**：
- Agent Gateway监控Token有效期，自动刷新活跃Token
- MCP Gateway检测Token即将过期（剩余<60秒），提示刷新
- refreshToken用于刷新，不用于工具调用

### 13.4 完整委托链传递（A2A）

**决策**：Token包含完整委托链

**理由**：
- 委托链清晰，便于审计
- 支持多级A2A调用
- 权限继承明确

**实现**：
- Token包含委托链：user→agent-001→agent-002
- MCP Gateway解析委托链，逐级鉴权
- 支持权限继承、限制传递

### 13.5 系统自动管理委托生命周期

**决策**：系统自动管理委托生命周期

**理由**：
- 减少用户操作负担
- 自动清理过期委托
- 审计提醒机制

**实现**：
- 长期委托：定期审计提醒（每月）
- 临时委托：自动过期清理（定时扫描）
- 会话级委托：随Session失效自动清理

---

## 14. 总结与展望

### 14.1 设计总结

**核心设计要点**：

1. **任务级Token机制**：Token与Task绑定，任务开始生成，任务结束失效
2. **工具标签策略配置**：工具分类标签化，委托策略基于标签组合
3. **环境维度强制注入**：Token包含环境维度，MCP Gateway自动注入
4. **A2A委托链传递**：完整委托链传递，支持多级A2A调用
5. **Cookie还原机制**：Session保存Cookie，MCP Gateway还原调用传统API
6. **委托生命周期管理**：长期、临时、会话级委托，系统自动管理
7. **实时确认机制**：高危工具强制用户实时确认
8. **分层架构设计**：Agent Gateway、MCP Gateway、策略中心三层架构

### 14.2 系统优势

**安全性优势**：
- Cookie不透传，防止权限滥用
- Token任务级绑定，防止Token滥用
- 环境维度强制注入，防止数据越权
- 委托链完整传递，防止权限泄露
- 异常调用检测，实时安全防护

**灵活性优势**：
- 工具标签策略，灵活配置权限范围
- Agent分类推荐策略，简化用户配置
- A2A权限继承，支持复杂协作场景
- 实时确认机制，平衡安全与体验

**可扩展性优势**：
- 新Agent注册无需修改代码
- 新工具注册自动同步
- 策略模板库可扩展
- 多级A2A支持

**可维护性优势**：
- 分层架构，职责清晰
- 审计日志完整，便于追溯
- 监控告警完善，快速定位问题
- 高可用部署，故障自动转移

### 14.3 待完善事项

**性能优化**：
- Token生成性能基准测试
- 策略查询缓存策略优化
- Cookie还原并发性能测试
- 审计日志写入性能优化

**安全防护**：
- Token滥用检测算法优化
- 异常调用模式识别算法
- 委托链伪造检测机制
- Cookie篡改检测机制

**用户体验**：
- 委托配置界面交互设计
- 实时确认弹窗交互优化
- Agent推荐策略智能匹配
- 权限不足提示引导优化

**监控告警**：
- A2A调用链可视化监控
- 委托关系拓扑可视化
- Token生命周期监控
- 权限变更影响分析

### 14.4 未来展望

**智能委托推荐**：
- 根据用户历史行为，智能推荐委托策略
- 根据Agent调用模式，自动调整权限范围

**动态权限调整**：
- 实时监控Agent行为，动态调整权限
- 异常行为检测，自动限制权限

**多租户支持**：
- 支持多租户环境下的委托机制
- 租户间权限隔离

**区块链审计**：
- 关键操作上链，不可篡改审计
- 委托关系上链，可信存证

---

## 附录

### 附录A：术语表

| 术语 | 说明 |
|------|------|
| Agent | 智能代理，执行用户任务的AI助手 |
| MCP | Model Context Protocol，工具调用协议 |
| Token | 任务级临时凭证，包含身份和委托链 |
| Session | 用户与Copilot的长会话，保存Cookie |
| Task | 用户单个请求任务，包含多个工具调用 |
| Delegation | 用户对Agent的权限委托关系 |
| A2A | Agent调用Agent的协作模式 |
| Cookie | IDaaS认证凭证，用于调用传统API |

### 附录B：接口清单

**Agent Gateway接口**：
- POST /agent/register：Agent注册
- PUT /agent/update：Agent更新
- DELETE /agent/unregister：Agent注销
- POST /agents/{agentId}/invoke：Copilot到Agent代理调用
- POST /api/v1/a2a/route：A2A代理调用
- POST /token/generate：内部Token生成接口（不由Copilot或Agent直接编排）
- POST /cookie/exchange：Cookie换取（MCP Gateway调用）
- GET /agent/health：Agent健康检查

**MCP Gateway接口**：
- POST /tool/call：工具调用
- POST /delegation/query：查询委托策略

**策略中心接口**：
- POST /delegation/create：创建委托
- PUT /delegation/update：更新委托
- DELETE /delegation/revoke：撤销委托
- GET /delegation/query：查询委托
- POST /confirmation/request：实时确认请求
- POST /delegation/notify：策略变更通知

### 附录C：配置示例

**Agent注册配置示例**：
```yaml
agent:
  id: agent-001
  name: 数据查询Agent
  type: DATA_QUERY
  endpoint: http://agent-001.internal:8080
  capabilities:
    supportedTags:
      - B2B
      - READ_ONLY
      - DATA_ACCESS
    a2aEnabled: true
  recommendedPolicy:
    templateId: B2B_READ_ONLY
```

**委托策略配置示例**：
```yaml
delegation:
  userId: l00867517
  agentId: agent-001
  type: TEMPORARY
  permissionScope:
    allowedTags:
      - B2B
      - READ_ONLY
    deniedTags:
      - DELETE
    dataRange:
      appName: C00001-O0023
      datasourceIds:
        - 184
        - 305
  expiresAt: 2026-05-28T11:00:00Z
```

---

## 15. 场景测试用例设计

### 15.1 Cookie转Token场景测试用例

#### 测试用例1：正常Cookie转Token流程

**测试场景**：用户首次提交任务请求，Cookie转Token成功

**前置条件**：
- 用户已通过IDaaS SSO登录
- Session已建立并保活
- Agent已注册（agent-001）

**测试步骤**：
1. 用户在Copilot输入："查询当前组织的数据源"
2. Copilot创建Task（taskId=task-001）
3. Copilot发送请求到Agent Gateway上的Agent统一入口（包含Cookie、agentId）
4. Agent Gateway解析Cookie，提取用户身份
5. Agent Gateway检查Agent注册信息和委托授权状态
6. Agent Gateway生成Token（tokenId=token-001）
7. Agent Gateway将原始任务请求、Token和任务上下文转发给真实Agent地址

**预期结果**：
- Token生成成功
- Agent真实地址未暴露给Copilot
- Token包含正确的用户身份（userId=l00867517）
- Token包含正确的Agent身份（agentId=agent-001）
- Token包含正确的环境维度（appName=C00001-O0023）
- Token状态为ACTIVE
- Token与Task绑定（taskId=task-001）
- Agent不需要调用Agent Gateway的Token生成API

**验证方法**：
- 检查Token数据结构完整性
- 检查Token存储到数据库
- 检查Token与Session、Task关联关系

---

#### 测试用例2：Cookie无效场景

**测试场景**：用户Cookie已过期，Token生成失败

**前置条件**：
- 用户Cookie已过期
- Agent已注册

**测试步骤**：
1. Copilot发送请求到Agent Gateway（包含过期Cookie）
2. Agent Gateway解析Cookie
3. Agent Gateway验证Cookie（调用IDaaS）
4. IDaaS返回Cookie无效

**预期结果**：
- Token生成失败
- 返回401错误：COOKIE_EXPIRED
- 提示用户重新登录
- Session清理

**验证方法**：
- 检查错误响应格式
- 检查Session状态更新为EXPIRED
- 检查审计日志记录

---

#### 测试用例3：Agent未注册场景

**测试场景**：调用未注册的Agent，Token生成失败

**前置条件**：
- 用户Cookie有效
- Agent未注册（agent-999）

**测试步骤**：
1. Copilot发送请求到Agent Gateway（包含agentId=agent-999）
2. Agent Gateway查询Agent注册表
3. Agent未找到

**预期结果**：
- Token生成失败
- 返回404错误：AGENT_NOT_FOUND
- 提示Agent未注册

**验证方法**：
- 检查错误响应格式
- 检查审计日志记录

---

### 15.2 MCP工具调用鉴权场景测试用例

#### 测试用例4：普通工具调用成功

**测试场景**：Agent调用普通工具（datasource_list），鉴权通过

**前置条件**：
- Token已生成（token-001）
- Task状态为RUNNING
- 工具标签：[B2B, READ_ONLY, COMMON]

**测试步骤**：
1. Agent调用datasource_list工具（携带Token）
2. MCP Gateway验证Token有效性
3. MCP Gateway检查工具分类（普通工具）
4. MCP Gateway调用Agent Gateway换取Cookie
5. MCP Gateway使用Cookie调用b2b-admin-service API
6. API返回数据源列表

**预期结果**：
- 工具调用成功
- 返回数据源列表数据
- Cookie还原成功
- 审计日志记录

**验证方法**：
- 检查返回数据完整性
- 检查Cookie换取日志
- 检查API调用日志
- 检查审计日志

---

#### 测试用例5：敏感工具调用（有委托）

**测试场景**：Agent调用敏感工具（datasource_create），有委托权限

**前置条件**：
- Token已生成
- 用户已委托Agent（delegationId=del-123）
- 委托策略：allowedTags=[B2B, WRITE]
- 工具标签：[B2B, WRITE, SENSITIVE]

**测试步骤**：
1. Agent调用datasource_create工具（携带Token）
2. MCP Gateway验证Token有效性
3. MCP Gateway检查工具分类（敏感工具）
4. MCP Gateway查询策略中心（委托策略）
5. 策略中心返回委托权限范围
6. MCP Gateway检查权限匹配（allowedTags包含工具标签）
7. MCP Gateway调用Agent Gateway换取Cookie
8. MCP Gateway调用API创建数据源

**预期结果**：
- 工具调用成功
- 数据源创建成功
- 权限鉴权通过
- 审计日志记录

**验证方法**：
- 检查数据源创建结果
- 检查权限鉴权日志
- 检查委托策略查询日志
- 检查审计日志

---

#### 测试用例6：敏感工具调用（无委托）

**测试场景**：Agent调用敏感工具（datasource_create），无委托权限

**前置条件**：
- Token已生成
- 用户未委托Agent
- 工具标签：[B2B, WRITE, SENSITIVE]

**测试步骤**：
1. Agent调用datasource_create工具（携带Token）
2. MCP Gateway验证Token有效性
3. MCP Gateway检查工具分类（敏感工具）
4. MCP Gateway查询策略中心（委托策略）
5. 策略中心返回：无委托

**预期结果**：
- 工具调用失败
- 返回403错误：DELEGATION_REQUIRED
- 提示用户配置委托
- 提供权限中心链接

**验证方法**：
- 检查错误响应格式
- 检查权限中心链接
- 检查审计日志记录

---

#### 测试用例7：敏感工具调用（权限不足）

**测试场景**：Agent调用敏感工具，委托权限不足

**前置条件**：
- Token已生成
- 用户已委托Agent
- 委托策略：allowedTags=[B2B, READ_ONLY]
- 工具标签：[B2B, WRITE, SENSITIVE]

**测试步骤**：
1. Agent调用datasource_create工具（携带Token）
2. MCP Gateway验证Token有效性
3. MCP Gateway检查工具分类（敏感工具）
4. MCP Gateway查询策略中心（委托策略）
5. MCP Gateway检查权限匹配（READ_ONLY vs WRITE）
6. 权限不匹配

**预期结果**：
- 工具调用失败
- 返回403错误：PERMISSION_DENIED
- 提示用户升级权限或实时确认

**验证方法**：
- 检查错误响应格式
- 检查权限不足提示
- 检查审计日志记录

---

#### 测试用例8：高危工具调用（实时确认）

**测试场景**：Agent调用高危工具（datasource_delete），需要实时确认

**前置条件**：
- Token已生成
- 工具标签：[B2B, DELETE, HIGH_RISK]

**测试步骤**：
1. Agent调用datasource_delete工具（携带Token）
2. MCP Gateway验证Token有效性
3. MCP Gateway检查工具分类（高危工具）
4. MCP Gateway返回403错误：USER_CONFIRMATION_REQUIRED
5. Agent提示用户确认
6. 用户点击确认链接
7. 权限中心创建临时委托
8. Agent重新调用工具

**预期结果**：
- 第一次调用失败，返回实时确认提示
- 用户确认后，临时委托创建成功
- 第二次调用成功
- 数据源删除成功
- 临时委托自动失效

**验证方法**：
- 检查实时确认提示格式
- 检查临时委托创建记录
- 检查工具调用成功结果
- 检查临时委托失效时间

---

### 15.3 Cookie还原场景测试用例

#### 测试用例9：Cookie换取成功

**测试场景**：MCP Gateway调用Agent Gateway换取Cookie成功

**前置条件**：
- Token已生成（token-001）
- Session状态为ACTIVE
- Cookie已加密存储

**测试步骤**：
1. MCP Gateway调用Agent Gateway：POST /cookie/exchange
2. Agent Gateway验证Token有效性
3. Agent Gateway验证sessionId一致性
4. Agent Gateway查询Session表
5. Agent Gateway解密Cookie
6. Agent Gateway返回Cookie

**预期结果**：
- Cookie换取成功
- 返回原始Cookie
- Cookie未过期
- 审计日志记录

**验证方法**：
- 检查Cookie完整性
- 检查Cookie解密正确性
- 检查审计日志

---

#### 测试用例10：Session失效场景

**测试场景**：Session已失效，Cookie换取失败

**前置条件**：
- Token已生成
- Session状态为EXPIRED

**测试步骤**：
1. MCP Gateway调用Agent Gateway：POST /cookie/exchange
2. Agent Gateway验证Token有效性
3. Agent Gateway查询Session表
4. Session状态为EXPIRED

**预期结果**：
- Cookie换取失败
- 返回错误：SESSION_EXPIRED
- 提示Session已失效

**验证方法**：
- 检查错误响应格式
- 检查审计日志

---

#### 测试用例11：Token无效场景

**测试场景**：Token已失效，Cookie换取失败

**前置条件**：
- Token状态为EXPIRED
- Task状态为COMPLETED

**测试步骤**：
1. MCP Gateway调用Agent Gateway：POST /cookie/exchange
2. Agent Gateway验证Token有效性
3. Token状态为EXPIRED

**预期结果**：
- Cookie换取失败
- 返回401错误：TOKEN_EXPIRED
- 提示Token已失效

**验证方法**：
- 检查错误响应格式
- 检查审计日志

---

### 15.4 数据范围注入场景测试用例

#### 测试用例12：数据范围注入成功

**测试场景**：委托策略包含数据范围，自动注入到工具参数

**前置条件**：
- Token已生成
- 委托策略：dataRange={appName=C00001-O0023, datasourceIds=[184,305]}
- 工具参数：{keyword="ISALES"}

**测试步骤**：
1. Agent调用datasource_list工具（携带Token）
2. MCP Gateway鉴权通过
3. MCP Gateway查询委托策略
4. MCP Gateway获取数据范围
5. MCP Gateway注入数据范围到工具参数
6. MCP Gateway调用API（参数：{keyword="ISALES", appName="C00001-O0023", datasourceIds=[184,305]}）

**预期结果**：
- 数据范围注入成功
- API参数包含appName和datasourceIds
- API返回限定范围内的数据
- 数据范围强制限定

**验证方法**：
- 检查注入后的工具参数
- 检查API调用参数
- 检查返回数据范围正确性

---

#### 测试用例13：环境维度强制注入

**测试场景**：Token包含环境维度，强制注入到工具参数

**前置条件**：
- Token包含：environmentContext={appName=C00001-O0023}
- 工具参数：{keyword="ISALES"}
- Agent尝试传入appName=C00001-O9999（错误值）

**测试步骤**：
1. Agent调用工具（参数：{keyword="ISALES", appName="C00001-O9999"}）
2. MCP Gateway解析Token的environmentContext
3. MCP Gateway强制注入appName=C00001-O0023（覆盖Agent传入的错误值）
4. MCP Gateway调用API

**预期结果**：
- appName强制注入为C00001-O0023（Token中的正确值）
- Agent传入的错误appName被覆盖
- API使用正确的appName

**验证方法**：
- 检查注入后的appName值
- 检查API调用参数
- 验证Agent无法绕过环境维度限定

---

### 15.5 A2A调用场景测试用例

#### 测试用例14：A2A调用成功

**测试场景**：Agent A调用Agent B，委托链传递成功

**前置条件**：
- Agent A已执行查询任务
- 用户委托Agent A包含A2A权限
- A2A权限：enabled=true, allowedTargetAgents=[agent-002]
- Agent B已注册

**测试步骤**：
1. Agent A调用Agent Gateway：POST /api/v1/a2a/route（calleeAgentId=agent-002）
2. Agent Gateway解析Agent A的Token
3. Agent Gateway检查A2A权限
4. Agent Gateway构建委托链：user→agent-001→agent-002
5. Agent Gateway生成Agent B的Token
6. Agent Gateway根据Agent B注册信息代理调用Agent B
7. Agent B调用MCP工具（携带Agent B的Token）
8. MCP Gateway解析委托链
9. MCP Gateway鉴权通过

**预期结果**：
- A2A调用成功
- 委托链正确传递
- Agent B获得权限
- 工具调用成功

**验证方法**：
- 检查Agent B的Token包含完整委托链
- 检查委托链格式正确
- 检查Agent B工具调用成功

---

#### 测试用例15：A2A权限不足

**测试场景**：Agent A调用Agent B，但A2A权限不足

**前置条件**：
- 用户委托Agent A不含A2A权限
- A2A权限：enabled=false

**测试步骤**：
1. Agent A调用Agent Gateway：POST /api/v1/a2a/route
2. Agent Gateway解析Agent A的Token
3. Agent Gateway检查A2A权限
4. A2A权限不足

**预期结果**：
- A2A调用失败
- 返回403错误：A2A_PERMISSION_DENIED
- 提示配置A2A权限

**验证方法**：
- 检查错误响应格式
- 检查审计日志

---

#### 测试用例16：多级A2A调用

**测试场景**：Agent A→Agent B→Agent C，三级委托链传递

**前置条件**：
- Agent A有A2A权限调用Agent B
- Agent B有A2A权限调用Agent C
- 用户委托Agent A包含A2A权限

**测试步骤**：
1. Agent A调用Agent B
2. Agent Gateway生成Agent B的Token（委托链：user→agent-001→agent-002）
3. Agent B调用Agent C
4. Agent Gateway生成Agent C的Token（委托链：user→agent-001→agent-002→agent-003）
5. Agent C调用MCP工具
6. MCP Gateway解析三级委托链

**预期结果**：
- 三级A2A调用成功
- 委托链完整传递
- Agent C获得权限
- 工具调用成功

**验证方法**：
- 检查Agent C的Token包含三级委托链
- 检查委托链格式正确
- 检查权限继承正确

---

### 15.6 委托配置场景测试用例

#### 测试用例17：首次委托配置

**测试场景**：用户首次委托Agent，接受推荐策略

**前置条件**：
- 用户登录权限中心
- Agent已注册（agent-001）
- Agent类型：DATA_QUERY
- 推荐策略：B2B只读类

**测试步骤**：
1. 用户选择Agent（agent-001）
2. 权限中心显示推荐策略：B2B只读类
3. 用户点击"接受推荐策略"
4. 权限中心创建委托记录
5. 权限中心保存委托策略

**预期结果**：
- 委托创建成功
- 委托策略包含：allowedTags=[B2B, READ_ONLY, DATA_ACCESS]
- 委托状态为ACTIVE
- Agent可以开始执行任务

**验证方法**：
- 检查委托记录完整性
- 检查委托策略正确性
- 检查委托状态

---

#### 测试用例18：自定义委托配置

**测试场景**：用户自定义委托策略

**前置条件**：
- 用户登录权限中心
- Agent已注册

**测试步骤**：
1. 用户选择Agent
2. 用户点击"自定义策略"
3. 用户勾选允许的标签：[B2B, READ_ONLY]
4. 用户配置数据范围：datasourceIds=[184]
5. 用户配置A2A权限：enabled=true, allowedTargetAgents=[agent-002]
6. 用户保存委托

**预期结果**：
- 委托创建成功
- 委托策略包含用户自定义的配置
- 数据范围限定正确
- A2A权限配置正确

**验证方法**：
- 检查委托策略与用户配置一致
- 检查数据范围限定
- 检查A2A权限配置

---

#### 测试用例19：委托撤销

**测试场景**：用户撤销委托

**前置条件**：
- 用户已有委托（delegationId=del-123）
- 委托状态为ACTIVE

**测试步骤**：
1. 用户登录权限中心
2. 用户查看委托列表
3. 用户选择委托（del-123）
4. 用户点击"撤销委托"
5. 权限中心更新委托状态为REVOKED
6. 权限中心通知Agent Gateway

**预期结果**：
- 委托撤销成功
- 委托状态更新为REVOKED
- Agent Gateway收到通知
- 相关Token失效

**验证方法**：
- 检查委托状态为REVOKED
- 检查Agent Gateway收到通知
- 检查相关Token失效

---

### 15.7 实时确认场景测试用例

#### 测试用例20：实时确认流程

**测试场景**：高危工具调用，用户实时确认

**前置条件**：
- Agent调用高危工具（datasource_delete）
- 工具标签：HIGH_RISK

**测试步骤**：
1. Agent调用datasource_delete工具
2. MCP Gateway返回403错误：USER_CONFIRMATION_REQUIRED
3. Agent返回实时确认提示给Copilot
4. 用户点击确认链接
5. 权限中心弹出确认窗口
6. 用户点击"确认执行"
7. 权限中心创建临时委托
8. 权限中心通知Agent Gateway
9. Agent重新调用工具
10. 工具调用成功

**预期结果**：
- 临时委托创建成功
- 临时委托包含：allowedTools=[datasource_delete], oneTimeOnly=true
- 工具调用成功
- 临时委托5分钟后失效

**验证方法**：
- 检查临时委托创建记录
- 检查临时委托参数
- 检查工具调用成功
- 检查临时委托失效时间

---

### 15.8 异常处理场景测试用例

#### 测试用例21：Token失效异常

**测试场景**：Token已失效，工具调用失败

**前置条件**：
- Token状态为EXPIRED
- Task状态为COMPLETED

**测试步骤**：
1. Agent调用工具（携带失效Token）
2. MCP Gateway验证Token有效性
3. Token状态为EXPIRED
4. MCP Gateway返回401错误：TOKEN_EXPIRED

**预期结果**：
- 工具调用失败
- 返回401错误
- 提示Token已失效，任务已结束

**验证方法**：
- 检查错误响应格式
- 检查审计日志

---

#### 测试用例22：异常调用检测

**测试场景**：检测到异常调用频率，Token失效

**前置条件**：
- Agent短时间内大量工具调用（>100次/分钟）

**测试步骤**：
1. Agent连续调用工具（频率异常）
2. MCP Gateway监控调用频率
3. MCP Gateway检测频率异常（>100次/分钟）
4. MCP Gateway触发安全防护
5. MCP Gateway立即失效Token
6. MCP Gateway终止Task
7. MCP Gateway返回403错误：SECURITY_VIOLATION

**预期结果**：
- Token立即失效
- Task终止
- 返回安全异常错误
- 发送安全告警

**验证方法**：
- 检查Token状态为EXPIRED
- 检查Task状态为TERMINATED
- 检查安全告警发送
- 检查审计日志

---

#### 测试用例23：Agent服务不可用

**测试场景**：Agent服务异常，无法调用

**前置条件**：
- Agent服务状态为UNHEALTHY

**测试步骤**：
1. Agent Gateway定时健康检查
2. Agent返回错误或超时
3. Agent Gateway更新Agent状态为UNHEALTHY
4. 用户提交任务请求
5. Agent Gateway拒绝生成Token

**预期结果**：
- Agent状态更新为UNHEALTHY
- Token生成失败
- 返回503错误：AGENT_UNAVAILABLE
- 提示Agent服务暂时不可用

**验证方法**：
- 检查Agent状态为UNHEALTHY
- 检查错误响应格式
- 检查审计日志

---

#### 测试用例24：策略中心服务异常

**测试场景**：策略中心服务异常，启用降级策略

**前置条件**：
- 策略中心服务无响应

**测试步骤**：
1. MCP Gateway查询策略中心
2. 策略中心无响应或超时
3. MCP Gateway启用降级策略
4. MCP Gateway使用本地缓存策略
5. MCP Gateway鉴权通过或拒绝

**预期结果**：
- 使用缓存策略鉴权
- 或返回保守策略：拒绝所有敏感工具
- 记录警告：使用缓存策略
- 发送告警

**验证方法**：
- 检查使用缓存策略
- 检查警告日志
- 检查告警发送

---

### 15.9 Task生命周期场景测试用例

#### 测试用例25：Task正常结束

**测试场景**：Task正常完成，Token失效

**前置条件**：
- Task状态为RUNNING
- Token状态为ACTIVE

**测试步骤**：
1. Agent完成任务
2. Agent返回结果给用户
3. Task状态更新为COMPLETED
4. Token状态更新为EXPIRED

**预期结果**：
- Task状态为COMPLETED
- Token状态为EXPIRED
- Token立即失效
- 无法再用于工具调用

**验证方法**：
- 检查Task状态
- 检查Token状态
- 检查Token失效时间

---

#### 测试用例26：Task异常终止

**测试场景**：Task异常终止，Token失效

**前置条件**：
- Task状态为RUNNING
- 检测到异常调用

**测试步骤**：
1. MCP Gateway检测异常调用
2. MCP Gateway终止Task
3. Task状态更新为TERMINATED
4. Token状态更新为EXPIRED
5. Agent Gateway通知用户

**预期结果**：
- Task状态为TERMINATED
- Token状态为EXPIRED
- 用户收到异常通知
- 审计日志记录

**验证方法**：
- 检查Task状态
- 检查Token状态
- 检查用户通知
- 检查审计日志

---

### 15.10 委托生命周期场景测试用例

#### 测试用例27：临时委托过期

**测试场景**：临时委托自动过期清理

**前置条件**：
- 委托类型为TEMPORARY
- expiresAt=2026-05-28T11:00:00Z
- 当前时间超过expiresAt

**测试步骤**：
1. 策略中心定时扫描过期委托
2. 发现委托已过期
3. 策略中心更新委托状态为EXPIRED
4. 策略中心清理缓存
5. 策略中心记录审计日志

**预期结果**：
- 委托状态为EXPIRED
- 缓存清理
- 审计日志记录

**验证方法**：
- 检查委托状态
- 检查缓存清理
- 检查审计日志

---

#### 测试用例28：会话级委托失效

**测试场景**：Session失效，会话级委托自动失效

**前置条件**：
- 委托类型为SESSION
- sessionId=session-001
- Session状态为EXPIRED

**测试步骤**：
1. Session失效
2. 策略中心收到Session失效通知
3. 策略中心更新委托状态为EXPIRED
4. 策略中心清理委托关系

**预期结果**：
- 委托状态为EXPIRED
- 委托关系清理
- 审计日志记录

**验证方法**：
- 检查委托状态
- 检查委托关系清理
- 检查审计日志

---

## 16. 验收标准

### 16.1 功能验收标准

#### 标准1：Cookie转Token功能验收

**验收标准**：
- ✅ Cookie解析成功，提取用户身份
- ✅ Token生成成功，包含完整数据结构
- ✅ Token与Task绑定，任务级生命周期
- ✅ Token包含环境维度（appName等）
- ✅ Cookie无效时返回正确错误
- ✅ Agent未注册时返回正确错误
- ✅ Token存储到数据库
- ✅ 审计日志记录完整

**验收方法**：
- 执行测试用例1-3
- 检查Token数据结构完整性
- 检查数据库存储
- 检查审计日志

---

#### 标准2：MCP工具调用鉴权功能验收

**验收标准**：
- ✅ Token验证成功，检查Token状态
- ✅ 工具分类检查正确（普通、敏感、高危）
- ✅ 普通工具直接调用成功
- ✅ 敏感工具委托鉴权正确
- ✅ 无委托时返回正确错误
- ✅ 权限不足时返回正确错误
- ✅ 高危工具实时确认流程正确
- ✅ 临时委托创建成功
- ✅ 审计日志记录完整

**验收方法**：
- 执行测试用例4-8
- 检查鉴权逻辑正确性
- 检查错误响应格式
- 检查审计日志

---

#### 标准3：Cookie还原功能验收

**验收标准**：
- ✅ MCP Gateway调用Agent Gateway换取Cookie
- ✅ Agent Gateway验证Token有效性
- ✅ Agent Gateway查询Session获取Cookie
- ✅ Cookie解密成功
- ✅ Cookie返回给MCP Gateway
- ✅ Session失效时返回正确错误
- ✅ Token无效时返回正确错误
- ✅ 审计日志记录完整

**验收方法**：
- 执行测试用例9-11
- 检查Cookie换取流程
- 检查Cookie完整性
- 检查审计日志

---

#### 标准4：数据范围注入功能验收

**验收标准**：
- ✅ 委托数据范围正确注入
- ✅ 环境维度强制注入
- ✅ Agent无法绕过数据范围限定
- ✅ API参数包含注入的数据范围
- ✅ API返回限定范围内的数据

**验收方法**：
- 执行测试用例12-13
- 检查注入后的工具参数
- 检查API调用参数
- 检查返回数据范围

---

#### 标准5：A2A调用功能验收

**验收标准**：
- ✅ A2A调用成功
- ✅ 委托链正确传递
- ✅ 委托链包含完整信息
- ✅ Agent B获得权限
- ✅ A2A权限不足时返回正确错误
- ✅ 多级A2A调用成功
- ✅ 审计日志记录完整

**验收方法**：
- 执行测试用例14-16
- 检查委托链格式
- 检查权限继承
- 检查审计日志

---

#### 标准6：委托配置功能验收

**验收标准**：
- ✅ 权限中心委托配置界面可用
- ✅ 推荐策略显示正确
- ✅ 接受推荐策略成功
- ✅ 自定义策略配置成功
- ✅ 数据范围配置成功
- ✅ A2A权限配置成功
- ✅ 委托保存成功
- ✅ 委托撤销成功
- ✅ 委托状态更新正确

**验收方法**：
- 执行测试用例17-19
- 检查委托配置界面
- 检查委托策略正确性
- 检查委托状态

---

#### 标准7：实时确认功能验收

**验收标准**：
- ✅ 高危工具返回实时确认提示
- ✅ 用户确认界面可用
- ✅ 临时委托创建成功
- ✅ 临时委托包含正确参数
- ✅ 工具重新调用成功
- ✅ 临时委托自动失效

**验收方法**：
- 执行测试用例20
- 检查实时确认流程
- 检查临时委托参数
- 检查临时委托失效时间

---

#### 标准8：异常处理功能验收

**验收标准**：
- ✅ Token失效异常处理正确
- ✅ 异常调用检测成功
- ✅ 安全防护触发正确
- ✅ Agent服务不可用处理正确
- ✅ 策略中心异常降级策略正确
- ✅ 错误响应格式正确
- ✅ 审计日志记录完整

**验收方法**：
- 执行测试用例21-24
- 检查异常处理逻辑
- 检查错误响应格式
- 检查审计日志

---

#### 标准9：Task生命周期功能验收

**验收标准**：
- ✅ Task正常结束，Token失效
- ✅ Task异常终止，Token失效
- ✅ Task状态更新正确
- ✅ Token状态更新正确
- ✅ 审计日志记录完整

**验收方法**：
- 执行测试用例25-26
- 检查Task状态
- 检查Token状态
- 检查审计日志

---

#### 标准10：委托生命周期功能验收

**验收标准**：
- ✅ 临时委托自动过期清理
- ✅ 会话级委托随Session失效
- ✅ 委托状态更新正确
- ✅ 缓存清理正确
- ✅ 审计日志记录完整

**验收方法**：
- 执行测试用例27-28
- 检查委托状态
- 检查缓存清理
- 检查审计日志

---

### 16.2 性能验收标准

#### 标准11：Token生成性能验收

**验收标准**：
- ✅ Token生成延迟 < 200ms（P95）
- ✅ Token生成延迟 < 500ms（P99）
- ✅ Token生成成功率 > 99.9%
- ✅ Cookie解析延迟 < 100ms

**验收方法**：
- 性能基准测试
- 压力测试（1000次/分钟）
- 监控指标统计

---

#### 标准12：工具调用性能验收

**验收标准**：
- ✅ 工具调用延迟 < 1s（P50）
- ✅ 工具调用延迟 < 2s（P95）
- ✅ 工具调用延迟 < 5s（P99）
- ✅ 工具调用成功率 > 99%
- ✅ 鉴权延迟 < 300ms

**验收方法**：
- 性能基准测试
- 压力测试（500次/分钟）
- 监控指标统计

---

#### 标准13：Cookie换取性能验收

**验收标准**：
- ✅ Cookie换取延迟 < 100ms（P95）
- ✅ Cookie换取延迟 < 200ms（P99）
- ✅ Cookie换取成功率 > 99.9%

**验收方法**：
- 性能基准测试
- 压力测试（1000次/分钟）
- 监控指标统计

---

#### 标准14：策略查询性能验收

**验收标准**：
- ✅ 策略查询延迟 < 200ms（P95）
- ✅ 策略查询延迟 < 500ms（P99）
- ✅ 策略查询成功率 > 99.9%
- ✅ 策略缓存命中率 > 80%

**验收方法**：
- 性能基准测试
- 压力测试（500次/分钟）
- 监控指标统计

---

#### 标准15：系统吞吐量验收

**验收标准**：
- ✅ Token生成吞吐量 > 1000次/分钟
- ✅ 工具调用吞吐量 > 500次/分钟
- ✅ Cookie换取吞吐量 > 1000次/分钟
- ✅ 策略查询吞吐量 > 500次/分钟

**验收方法**：
- 压力测试
- 监控指标统计

---

### 16.3 安全验收标准

#### 标准16：权限安全验收

**验收标准**：
- ✅ Cookie不透传到Agent
- ✅ Token任务级绑定，防止Token滥用
- ✅ 环境维度强制注入，防止数据越权
- ✅ 委托链完整传递，防止权限泄露
- ✅ 数据范围强制限定
- ✅ Agent无法绕过权限限制

**验收方法**：
- 安全测试
- 渗透测试
- 权限绕过测试

---

#### 标准17：异常检测安全验收

**验收标准**：
- ✅ 异常调用频率检测成功（>100次/分钟）
- ✅ 异常调用模式检测成功
- ✅ Token滥用检测成功
- ✅ 安全防护触发正确
- ✅ Token立即失效
- ✅ Task立即终止
- ✅ 安全告警发送正确

**验收方法**：
- 异常调用测试
- Token滥用测试
- 安全防护测试

---

#### 标准18：审计安全验收

**验收标准**：
- ✅ 所有关键操作审计日志记录
- ✅ 审计日志包含完整信息
- ✅ 审计日志不可篡改
- ✅ 审计日志存储安全
- ✅ 审计日志查询可用

**验收方法**：
- 审计日志完整性检查
- 审计日志篡改测试
- 审计日志查询测试

---

#### 标准19：加密安全验收

**验收标准**：
- ✅ Cookie加密存储（AES-256-GCM）
- ✅ Cookie解密正确
- ✅ Token签名验证正确
- ✅ 加密算法符合安全标准

**验收方法**：
- 加密算法检查
- 解密正确性测试
- 签名验证测试

---

### 16.4 可用性验收标准

#### 标准20：系统可用性验收

**验收标准**：
- ✅ Agent Gateway可用性 > 99.9%
- ✅ MCP Gateway可用性 > 99.9%
- ✅ 策略中心可用性 > 99.9%
- ✅ 故障自动转移成功
- ✅ 故障恢复时间 < 5分钟

**验收方法**：
- 可用性监控
- 故障转移测试
- 故障恢复测试

---

#### 标准21：容错能力验收

**验收标准**：
- ✅ Agent服务不可用时降级处理
- ✅ 策略中心不可用时降级处理
- ✅ Session失效时正确处理
- ✅ Token失效时正确处理
- ✅ 数据库异常时正确处理

**验收方法**：
- 容错测试
- 降级策略测试
- 异常处理测试

---

#### 标准22：用户体验验收

**验收标准**：
- ✅ 委托配置界面易用
- ✅ 实时确认界面易用
- ✅ 错误提示清晰
- ✅ 权限不足提示引导正确
- ✅ 响应时间满足用户期望

**验收方法**：
- 用户体验测试
- 用户反馈收集
- 响应时间测试

---

### 16.5 可扩展性验收标准

#### 标准23：Agent扩展验收

**验收标准**：
- ✅ 新Agent注册无需修改代码
- ✅ Agent类型扩展支持
- ✅ Agent能力扩展支持
- ✅ Agent注册流程正确

**验收方法**：
- 新Agent注册测试
- Agent类型扩展测试
- Agent能力扩展测试

---

#### 标准24：工具扩展验收

**验收标准**：
- ✅ 新工具注册无需修改代码
- ✅ 工具标签扩展支持
- ✅ 工具分类扩展支持
- ✅ 工具元数据管理正确

**验收方法**：
- 新工具注册测试
- 工具标签扩展测试
- 工具分类扩展测试

---

#### 标准25：策略扩展验收

**验收标准**：
- ✅ 新策略模板支持
- ✅ 策略规则扩展支持
- ✅ 委托类型扩展支持
- ✅ 策略模板库管理正确

**验收方法**：
- 新策略模板测试
- 策略规则扩展测试
- 委托类型扩展测试

---

### 16.6 监控告警验收标准

#### 标准26：监控指标验收

**验收标准**：
- ✅ Agent Gateway监控指标正确
- ✅ MCP Gateway监控指标正确
- ✅ 策略中心监控指标正确
- ✅ 安全监控指标正确
- ✅ 监控数据实时准确

**验收方法**：
- 监控指标检查
- 监控数据准确性测试
- 监控实时性测试

---

#### 标准27：告警机制验收

**验收标准**：
- ✅ 安全告警触发正确
- ✅ 性能告警触发正确
- ✅ 可用性告警触发正确
- ✅ 告警级别分类正确
- ✅ 告警通知发送正确

**验收方法**：
- 告警触发测试
- 告警级别测试
- 告警通知测试

---

### 16.7 文档验收标准

#### 标准28：设计文档验收

**验收标准**：
- ✅ 设计文档完整性
- ✅ 设计文档准确性
- ✅ 设计文档可读性
- ✅ 设计文档更新及时

**验收方法**：
- 文档完整性检查
- 文档准确性检查
- 文档可读性检查

---

#### 标准29：接口文档验收

**验收标准**：
- ✅ 接口文档完整性
- ✅ 接口文档准确性
- ✅ 接口文档示例完整
- ✅ 接口文档更新及时

**验收方法**：
- 接口文档检查
- 接口示例测试
- 接口文档更新检查

---

#### 标准30：测试用例文档验收

**验收标准**：
- ✅ 测试用例覆盖所有功能
- ✅ 测试用例准确性
- ✅ 测试用例可执行性
- ✅ 测试用例验收标准明确

**验收方法**：
- 测试用例覆盖率检查
- 测试用例准确性检查
- 测试用例可执行性检查

---

## 17. 总结

### 17.1 设计完成度

**设计文档完成度**：
- ✅ 17个章节，完整覆盖架构设计
- ✅ 30个测试用例，覆盖所有核心场景
- ✅ 30个验收标准，明确验收要求
- ✅ 附录包含术语表、接口清单、配置示例

**设计覆盖范围**：
- ✅ 整体架构设计
- ✅ 核心数据结构设计
- ✅ 核心流程设计（11个流程）
- ✅ 工具标签策略设计
- ✅ A2A委托链传递设计
- ✅ 委托配置流程设计
- ✅ Agent注册管理设计
- ✅ 审计日志与监控设计
- ✅ 异常处理与安全防护设计
- ✅ 系统部署与扩展性设计
- ✅ 场景测试用例设计
- ✅ 验收标准设计

### 17.2 下一步实施建议

**实施阶段划分**：

**阶段1：核心组件开发**（优先级：HIGH）
- Agent Gateway开发：Token生成、Session管理、Cookie换取、A2A路由
- MCP Gateway开发：工具鉴权、Cookie还原、数据注入
- 策略中心开发：委托配置、策略存储、委托链解析

**阶段2：数据库与存储开发**（优先级：HIGH）
- Session表、Task表、Token表、Delegation表、Agent表、ToolMetadata表
- Redis缓存：Session缓存、Token缓存、策略缓存
- Elasticsearch审计日志存储

**阶段3：接口开发**（优先级：HIGH）
- Agent Gateway接口：注册、Token生成、Cookie换取、A2A调用
- MCP Gateway接口：工具调用、鉴权
- 策略中心接口：委托配置、策略查询、实时确认

**阶段4：测试与验收**（优先级：MEDIUM）
- 执行30个测试用例
- 性能基准测试
- 安全测试
- 可用性测试

**阶段5：监控与告警**（优先级：MEDIUM）
- 监控指标采集
- 告警规则配置
- 告警通知机制

**阶段6：文档与培训**（优先级：LOW）
- 用户手册编写
- 开发者文档编写
- 运维文档编写
- 用户培训

### 17.3 风险与挑战

**技术风险**：
- Cookie还原性能：需要优化Cookie换取性能
- 策略查询性能：需要优化策略缓存机制
- A2A委托链传递：需要处理多级委托链的复杂性

**安全风险**：
- Cookie安全：需要确保Cookie加密存储和传输安全
- Token滥用：需要完善Token滥用检测机制
- 权限泄露：需要完善委托链安全机制

**运维风险**：
- 系统可用性：需要确保高可用部署
- 故障恢复：需要完善故障转移和恢复机制
- 监控告警：需要完善监控告警机制

### 17.4 后续优化方向

#### 17.4.1 性能优化

**1. Token生成性能优化**

**优化目标**：
- Token生成延迟 < 10ms
- 高并发场景下Token生成吞吐量 > 10000 TPS
- Token生成服务可用性 > 99.99%

**优化方案**：

**方案1：预生成Token池**
```
实现思路：
1. 后台线程预生成Token池（1000个Token）
2. Token池水位低于30%时自动补充
3. Token生成时直接从池中获取，无需实时生成
4. Token池使用Redis存储，支持分布式场景

性能提升：
- Token获取延迟：从20ms降至2ms
- 吞吐量提升：从5000 TPS提升至15000 TPS

技术要点：
- Token池线程安全：使用Redis分布式锁
- Token池预热：服务启动时预加载Token池
- Token池监控：实时监控水位和补充速度
```

**方案2：JWT签名优化**
```
实现思路：
1. 使用RS256算法替代HS256，提升安全性
2. 使用JWT缓存，避免重复签名
3. 使用异步签名，避免阻塞主线程

性能提升：
- 签名延迟：从15ms降至5ms
- CPU占用降低30%

技术要点：
- JWT缓存Key：userId + sessionId + timestamp
- 缓存过期时间：5分钟
- 缓存命中率监控：目标 > 80%
```

**方案3：Token生成异步化**
```
实现思路：
1. Token生成请求进入异步队列
2. 后台线程池处理Token生成
3. 客户端通过WebSocket接收Token

性能提升：
- 接口响应时间：从20ms降至5ms
- 系统吞吐量提升50%

技术要点：
- 异步队列：使用Disruptor高性能队列
- 线程池配置：核心线程数 = CPU核心数 * 2
- 背压控制：队列满时拒绝请求
```

**性能基准测试**：
```
测试场景1：单用户Token生成
- 测试用例：连续生成1000个Token
- 预期结果：平均延迟 < 10ms，成功率100%

测试场景2：并发Token生成
- 测试用例：100个并发用户，每用户生成100个Token
- 预期结果：TPS > 10000，平均延迟 < 20ms

测试场景3：Token池压力测试
- 测试用例：Token池水位从100%降至0%
- 预期结果：补充速度 > 500 Token/s，无Token获取失败
```

---

**2. 策略查询性能优化**

**优化目标**：
- 策略查询延迟 < 5ms
- 策略缓存命中率 > 95%
- 策略变更生效延迟 < 1s

**优化方案**：

**方案1：多级缓存架构**
```
缓存层级：
L1缓存：本地内存缓存（Caffeine）
  - 容量：10000条策略
  - 过期时间：5分钟
  - 命中率目标：80%

L2缓存：Redis分布式缓存
  - 容量：无限制
  - 过期时间：30分钟
  - 命中率目标：95%

L3存储：MySQL数据库
  - 策略持久化存储
  - 命中率目标：5%

缓存查询流程：
1. 查询L1缓存，命中则返回
2. 未命中则查询L2缓存，命中则回填L1并返回
3. 未命中则查询数据库，回填L1、L2并返回

缓存更新策略：
- 主动更新：策略变更时主动刷新缓存
- 被动更新：缓存过期时重新加载
- 预加载：热点策略预加载到L1缓存
```

**方案2：策略索引优化**
```
索引设计：
1. 主键索引：delegationId
2. 唯一索引：userId + agentId
3. 组合索引：userId + agentId + status
4. 全文索引：工具标签（JSON字段索引）

查询优化：
- 使用索引覆盖查询，避免回表
- 使用分页查询，避免大结果集
- 使用延迟关联，优化深分页查询

性能提升：
- 查询延迟：从50ms降至5ms
- 数据库CPU占用降低40%
```

**方案3：策略预编译**
```
实现思路：
1. 策略配置时预编译为可执行规则
2. 规则引擎使用Drools或Aviator
3. 编译后的规则缓存到Redis

性能提升：
- 规则执行延迟：从10ms降至1ms
- 内存占用降低20%

技术要点：
- 规则编译缓存Key：策略版本号
- 规则更新时自动重新编译
- 编译失败回退到原始策略
```

**性能基准测试**：
```
测试场景1：策略查询缓存命中
- 测试用例：连续查询10000次相同策略
- 预期结果：L1命中率 > 80%，平均延迟 < 2ms

测试场景2：策略查询缓存未命中
- 测试用例：查询10000个不同策略
- 预期结果：L2命中率 > 95%，平均延迟 < 5ms

测试场景3：策略更新生效延迟
- 测试用例：更新策略后立即查询
- 预期结果：缓存刷新延迟 < 1s，查询结果正确
```

---

**3. Cookie换取性能优化**

**优化目标**：
- Cookie换取延迟 < 20ms
- Cookie缓存命中率 > 90%
- Cookie换取成功率 > 99.9%

**优化方案**：

**方案1：Cookie缓存机制**
```
缓存设计：
- 缓存Key：tokenId
- 缓存Value：加密后的Cookie
- 过期时间：与Token过期时间一致
- 缓存存储：Redis

缓存策略：
- 写入：Token生成时，同时缓存Cookie
- 读取：工具调用时，优先从缓存读取Cookie
- 更新：Cookie变更时，主动刷新缓存
- 删除：Token失效时，删除缓存

性能提升：
- Cookie获取延迟：从50ms降至5ms（缓存命中）
- Agent Gateway压力降低80%
```

**方案2：Cookie批量换取**
```
实现思路：
1. Agent Gateway提供批量换取接口
2. MCP Gateway缓存Token-Cookie映射
3. 批量预取Cookie，减少实时换取次数

批量换取流程：
1. MCP Gateway收集待换取的Token列表（最多100个）
2. 批量调用Agent Gateway换取接口
3. 缓存Token-Cookie映射关系
4. 后续请求直接从缓存获取

性能提升：
- 批量换取吞吐量：从100 TPS提升至1000 TPS
- 网络IO减少90%

技术要点：
- 批量大小：100个Token/次
- 批量超时：200ms
- 失败重试：单个Token失败不影响整体
```

**方案3：Cookie异步刷新**
```
实现思路：
1. Cookie即将过期时（剩余时间 < 30%），异步刷新
2. 后台线程定期扫描即将过期的Token
3. 主动刷新Cookie，避免实时换取

刷新策略：
- 扫描周期：每分钟扫描一次
- 刷新条件：剩余过期时间 < 30%
- 刷新方式：异步调用Agent Gateway刷新接口

性能提升：
- 实时换取请求减少70%
- Cookie获取成功率提升至99.9%

技术要点：
- 异步线程池：核心线程数 = 10
- 刷新失败重试：最多3次
- 刷新失败告警：连续失败3次触发告警
```

**性能基准测试**：
```
测试场景1：Cookie缓存命中
- 测试用例：连续换取10000次相同Token的Cookie
- 预期结果：缓存命中率 > 90%，平均延迟 < 10ms

测试场景2：Cookie批量换取
- 测试用例：批量换取100个Token的Cookie
- 预期结果：批量换取延迟 < 100ms，成功率100%

测试场景3：Cookie异步刷新
- 测试用例：1000个Token即将过期
- 预期结果：刷新成功率 > 99%，无实时换取请求
```

---

**4. 审计日志写入性能优化**

**优化目标**：
- 审计日志写入延迟 < 10ms
- 审计日志写入吞吐量 > 50000 TPS
- 审计日志查询延迟 < 100ms

**优化方案**：

**方案1：异步批量写入**
```
实现思路：
1. 审计日志先写入内存队列
2. 后台线程批量写入Elasticsearch
3. 批量大小：1000条/次，或100ms超时

写入流程：
1. 审计日志写入Disruptor队列（无锁队列）
2. 消费者线程从队列批量读取日志
3. 批量写入Elasticsearch
4. 写入失败则重试或降级到本地文件

性能提升：
- 写入延迟：从50ms降至5ms
- 吞吐量提升：从10000 TPS提升至50000 TPS

技术要点：
- 队列大小：100000条
- 批量大小：1000条或100ms
- 背压控制：队列满时降级到本地文件
```

**方案2：Elasticsearch索引优化**
```
索引设计：
1. 按日期分索引：audit_log_20260101
2. 使用别名：audit_log_alias指向最新索引
3. 设置副本数：0（写入时），1（查询时）
4. 设置刷新间隔：30s（默认1s）

索引模板：
{
  "index_patterns": ["audit_log_*"],
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 0,
    "refresh_interval": "30s"
  },
  "mappings": {
    "properties": {
      "timestamp": {"type": "date"},
      "userId": {"type": "keyword"},
      "agentId": {"type": "keyword"},
      "toolName": {"type": "keyword"},
      "action": {"type": "keyword"}
    }
  }
}

性能提升：
- 写入吞吐量提升200%
- 存储空间节省30%
```

**方案3：日志压缩与归档**
```
实现思路：
1. 30天内的日志保留在热索引
2. 30-90天的日志压缩后归档到冷索引
3. 90天以上的日志归档到对象存储

归档流程：
1. 定时任务每天凌晨执行归档
2. 将过期索引压缩为只读索引
3. 迁移到冷数据节点或对象存储
4. 删除原索引，释放存储空间

性能提升：
- 热索引大小减少70%
- 查询性能提升50%

技术要点：
- 压缩算法：LZ4
- 归档存储：OBS对象存储
- 查询路由：自动路由到冷热索引
```

**性能基准测试**：
```
测试场景1：审计日志写入吞吐量
- 测试用例：并发写入100万条审计日志
- 预期结果：吞吐量 > 50000 TPS，平均延迟 < 10ms

测试场景2：审计日志查询延迟
- 测试用例：查询最近1小时的审计日志
- 预期结果：查询延迟 < 100ms，结果准确

测试场景3：审计日志归档性能
- 测试用例：归档30天的审计日志（1000万条）
- 预期结果：归档时间 < 30分钟，无数据丢失
```

---

#### 17.4.2 安全增强

**1. Token滥用检测算法优化**

**优化目标**：
- Token滥用检测准确率 > 95%
- Token滥用检测误报率 < 5%
- Token滥用检测延迟 < 100ms

**优化方案**：

**方案1：基于行为分析的检测算法**
```
检测维度：
1. 调用频率异常
   - 正常：每分钟调用 < 100次
   - 异常：每分钟调用 > 500次
   - 检测算法：滑动窗口统计 + Z-Score异常检测

2. 调用时间异常
   - 正常：工作时间（9:00-18:00）调用
   - 异常：非工作时间频繁调用
   - 检测算法：时间窗口统计 + 偏差分析

3. 调用工具异常
   - 正常：调用用户常用工具
   - 异常：调用从未使用过的敏感工具
   - 检测算法：工具使用历史 + 协同过滤

4. 调用结果异常
   - 正常：成功率高（> 95%）
   - 异常：成功率低（< 50%）
   - 检测算法：成功率统计 + 阈值判断

综合评分：
- 每个维度评分0-100分
- 加权平均：频率(30%) + 时间(20%) + 工具(30%) + 结果(20%)
- 风险等级：低(0-30)、中(30-70)、高(70-100)
- 处理策略：低风险-记录日志，中风险-二次验证，高风险-拒绝请求
```

**方案2：基于机器学习的检测模型**
```
模型选择：
- 算法：Isolation Forest（孤立森林）
- 特征：调用频率、时间、工具、结果、用户画像
- 训练数据：历史正常调用数据（100万条）
- 异常比例：1%

模型训练：
1. 特征工程：
   - 调用频率特征：每分钟/每小时/每天调用次数
   - 时间特征：小时、星期、是否工作时间
   - 工具特征：工具类别、工具敏感度、工具使用频率
   - 用户特征：用户角色、用户权限、用户历史行为

2. 模型训练：
   - 训练集：80%正常数据 + 20%异常数据
   - 验证集：10%正常数据 + 10%异常数据
   - 参数调优：n_estimators=100, contamination=0.01

3. 模型评估：
   - 准确率：> 95%
   - 召回率：> 90%
   - F1-Score：> 0.92

模型部署：
- 在线预测：实时调用模型预测风险
- 离线训练：每天凌晨重新训练模型
- A/B测试：新模型先灰度发布，验证效果后全量发布
```

**方案3：实时风控规则引擎**
```
规则引擎设计：
- 规则类型：统计规则、序列规则、关联规则
- 规则执行：Drools规则引擎
- 规则更新：热更新，无需重启

规则示例：
规则1：单工具高频调用
- 条件：1分钟内调用同一工具 > 50次
- 动作：触发验证码验证

规则2：敏感工具连续失败
- 条件：连续调用敏感工具失败 > 3次
- 动作：锁定Token，发送告警

规则3：跨工具异常组合
- 条件：短时间内调用多个敏感工具组合
- 动作：触发人工审核

规则4：异地登录调用
- 条件：Token使用IP与登录IP不一致
- 动作：触发二次认证

规则5：权限越界调用
- 条件：调用工具超出委托权限范围
- 动作：拒绝请求，锁定Token
```

**安全基准测试**：
```
测试场景1：正常调用检测
- 测试用例：10000次正常调用
- 预期结果：误报率 < 5%，无正常请求被拒绝

测试场景2：异常调用检测
- 测试用例：1000次异常调用（高频、非工作时间、敏感工具）
- 预期结果：检测率 > 95%，异常请求被正确识别

测试场景3：检测延迟测试
- 测试用例：实时检测10000次调用
- 预期结果：平均检测延迟 < 100ms，无阻塞
```

---

**2. 异常调用模式识别算法**

**优化目标**：
- 异常模式识别准确率 > 90%
- 异常模式识别覆盖率 > 80%
- 异常模式识别延迟 < 200ms

**优化方案**：

**方案1：调用序列模式识别**
```
序列模式定义：
- 正常序列：用户历史调用序列的常见模式
- 异常序列：与历史模式偏差较大的序列

识别算法：
1. 序列挖掘：
   - 使用PrefixSpan算法挖掘频繁序列
   - 最小支持度：0.1（至少出现10%）
   - 序列长度：2-10个工具调用

2. 序列匹配：
   - 计算当前序列与历史序列的相似度
   - 相似度算法：编辑距离 + Jaccard相似度
   - 阈值：相似度 < 0.3视为异常

3. 序列预测：
   - 使用LSTM模型预测下一个工具调用
   - 预测准确率：> 70%
   - 实际调用与预测不符时触发告警

示例：
正常序列：登录 -> 查看数据源 -> 创建数据源 -> 配置数据源
异常序列：登录 -> 删除数据源 -> 删除用户 -> 删除租户（高风险序列）
```

**方案2：用户行为基线建模**
```
基线维度：
1. 时间基线：
   - 每日活跃时段：9:00-18:00
   - 每周活跃天数：周一至周五
   - 节假日活跃度：低

2. 频率基线：
   - 每分钟调用次数：10-50次
   - 每小时调用次数：100-500次
   - 每天调用次数：1000-5000次

3. 工具基线：
   - 常用工具：datasource_query、datasource_create
   - 敏感工具使用频率：低（< 5%）
   - 工具组合模式：固定组合

4. 结果基线：
   - 成功率：> 95%
   - 失败原因分布：参数错误(60%)、权限不足(30%)、其他(10%)

基线建模：
- 历史数据：最近30天的调用数据
- 更新周期：每天凌晨更新基线
- 偏差检测：实际行为与基线偏差 > 50%视为异常
```

**方案3：关联分析异常检测**
```
关联规则挖掘：
1. 工具关联：
   - 规则：调用工具A后，80%会调用工具B
   - 异常：调用工具A后，调用工具C（关联度 < 10%）

2. 用户关联：
   - 规则：用户A的调用模式与用户B相似度 > 80%
   - 异常：用户A突然改变调用模式

3. 时间关联：
   - 规则：工作时间调用工具A，非工作时间调用工具B
   - 异常：非工作时间调用敏感工具A

关联算法：
- Apriori算法挖掘关联规则
- 最小支持度：0.1
- 最小置信度：0.7
- 提升度：> 1.5

异常检测：
- 计算实际调用与关联规则的匹配度
- 匹配度 < 0.3视为异常
- 触发二次验证或人工审核
```

**安全基准测试**：
```
测试场景1：序列模式识别
- 测试用例：1000个正常序列 + 100个异常序列
- 预期结果：正常序列识别率 > 90%，异常序列识别率 > 80%

测试场景2：行为基线检测
- 测试用例：100个用户的行为基线建模
- 预期结果：基线建模准确率 > 90%，异常检测准确率 > 85%

测试场景3：关联分析检测
- 测试用例：挖掘100条关联规则，检测1000次调用
- 预期结果：关联规则准确率 > 80%，异常检测延迟 < 200ms
```

---

**3. 委托链伪造检测机制**

**优化目标**：
- 委托链伪造检测准确率 100%
- 委托链验证延迟 < 10ms
- 委托链完整性保证 100%

**优化方案**：

**方案1：委托链数字签名**
```
签名机制：
1. 签名算法：RSA-2048或ECDSA-P256
2. 签名内容：委托链完整JSON
3. 签名Key：每个Agent拥有唯一私钥
4. 验证Key：公钥存储在策略中心

签名流程：
1. 用户委托Agent-001：
   - 生成委托记录：{from: "user:l00867517", to: "agent:agent-001", timestamp: 1704067200}
   - 用户私钥签名：signature_user

2. Agent-001委托Agent-002：
   - 追加委托记录：{from: "agent:agent-001", to: "agent:agent-002", timestamp: 1704067260}
   - Agent-001私钥签名：signature_agent_001

3. 完整委托链：
   {
     "chain": [
       {from: "user:l00867517", to: "agent:agent-001", timestamp: 1704067200},
       {from: "agent:agent-001", to: "agent:agent-002", timestamp: 1704067260}
     ],
     "signatures": ["signature_user", "signature_agent_001"]
   }

验证流程：
1. 提取委托链和签名
2. 逐级验证签名：
   - 使用用户公钥验证第一级签名
   - 使用Agent-001公钥验证第二级签名
3. 验证时间戳连续性
4. 验证委托关系合法性
```

**方案2：委托链时间戳验证**
```
时间戳验证规则：
1. 时间戳连续性：
   - 后一级委托时间戳 >= 前一级委托时间戳
   - 时间戳间隔 < 委托有效期

2. 时间戳有效性：
   - 时间戳 > 当前时间 - 委托有效期
   - 时间戳 < 当前时间 + 时钟偏差容忍度（5分钟）

3. 时间戳防重放：
   - 每个委托记录的时间戳唯一
   - 已使用的时间戳记录在Redis中
   - 时间戳有效期：5分钟

验证流程：
1. 提取委托链中所有时间戳
2. 验证时间戳连续性
3. 验证时间戳有效性
4. 验证时间戳未重放
5. 任一验证失败则拒绝请求
```

**方案3：委托链完整性校验**
```
完整性校验规则：
1. 委托链完整性：
   - 委托链必须从用户开始
   - 委托链中间不能有断裂
   - 委托链终点必须是当前Agent

2. 委托权限一致性：
   - 每一级委托的权限范围 <= 上一级委托的权限范围
   - 权限范围逐级缩小或相等

3. 委托有效期一致性：
   - 每一级委托的有效期 <= 上一级委托的有效期
   - 有效期逐级缩短或相等

4. 委托关系合法性：
   - 委托人必须有委托权限
   - 受托人必须已注册
   - 委托关系必须在策略中心备案

校验流程：
1. 解析委托链JSON
2. 校验委托链完整性
3. 校验委托权限一致性
4. 校验委托有效期一致性
5. 校验委托关系合法性
6. 任一校验失败则拒绝请求
```

**安全基准测试**：
```
测试场景1：委托链伪造检测
- 测试用例：100个正常委托链 + 100个伪造委托链
- 预期结果：正常委托链通过率100%，伪造委托链检测率100%

测试场景2：委托链验证性能
- 测试用例：验证10000个委托链（平均长度5级）
- 预期结果：平均验证延迟 < 10ms，吞吐量 > 1000 TPS

测试场景3：委托链防重放
- 测试用例：重放1000个已使用的委托链
- 预期结果：重放检测率100%，所有重放请求被拒绝
```

---

**4. Cookie篡改检测机制**

**优化目标**：
- Cookie篡改检测准确率 100%
- Cookie完整性验证延迟 < 5ms
- Cookie加密强度符合安全标准

**优化方案**：

**方案1：Cookie数字签名**
```
签名机制：
1. 签名算法：HMAC-SHA256
2. 签名Key：系统密钥（定期轮换）
3. 签名内容：Cookie完整内容
4. 签名存储：Cookie值追加签名字段

签名流程：
1. 用户登录成功，生成Cookie
2. 计算Cookie的HMAC签名
3. 签名追加到Cookie值：cookie_value|signature
4. 返回签名后的Cookie给客户端

验证流程：
1. 接收客户端Cookie
2. 分离Cookie值和签名
3. 使用系统密钥重新计算签名
4. 比对签名是否一致
5. 签名不一致则拒绝请求
```

**方案2：Cookie加密存储**
```
加密机制：
1. 加密算法：AES-256-GCM
2. 加密Key：系统主密钥（定期轮换）
3. 加密模式：认证加密（AEAD）
4. 加密存储：Redis缓存

加密流程：
1. Token生成时，加密Cookie
2. 加密后存储到Redis：Key=tokenId, Value=encrypted_cookie
3. 设置过期时间：与Token过期时间一致
4. 由Agent Gateway在代理转发请求时注入Token给Agent

解密流程：
1. Agent调用工具，传入Token
2. MCP Gateway从Redis获取加密Cookie
3. 使用系统密钥解密Cookie
4. 验证Cookie完整性（GCM模式自带完整性校验）
5. 解密失败则拒绝请求
```

**方案3：Cookie完整性校验**
```
完整性校验规则：
1. Cookie结构校验：
   - 必须包含必要字段：userId、sessionId、tenantId、appName
   - 字段格式必须正确：userId格式、sessionId格式等
   - 字段值必须合法：tenantId存在、appName存在

2. Cookie时间戳校验：
   - Cookie创建时间戳必须有效
   - Cookie过期时间戳必须有效
   - Cookie未过期

3. Cookie来源校验：
   - Cookie必须来自IDaaS SSO
   - Cookie的签发者必须可信
   - Cookie的受众必须匹配当前系统

4. Cookie权限校验：
   - Cookie中的权限必须与Token中的权限一致
   - Cookie中的租户必须与Token中的租户一致
   - Cookie中的应用必须与Token中的应用一致

校验流程：
1. 解密Cookie
2. 校验Cookie结构
3. 校验Cookie时间戳
4. 校验Cookie来源
5. 校验Cookie权限
6. 任一校验失败则拒绝请求
```

**安全基准测试**：
```
测试场景1：Cookie篡改检测
- 测试用例：100个正常Cookie + 100个篡改Cookie
- 预期结果：正常Cookie通过率100%，篡改Cookie检测率100%

测试场景2：Cookie加密性能
- 测试用例：加密/解密10000个Cookie
- 预期结果：加密延迟 < 5ms，解密延迟 < 5ms

测试场景3：Cookie完整性验证
- 测试用例：验证10000个Cookie的完整性
- 预期结果：验证延迟 < 5ms，验证准确率100%
```

---

#### 17.4.3 用户体验优化

**1. 委托配置界面交互优化**

**优化目标**：
- 委托配置完成时间 < 2分钟
- 委托配置错误率 < 5%
- 用户满意度 > 90%

**优化方案**：

**方案1：智能推荐策略**
```
推荐维度：
1. 基于Agent类型推荐：
   - 数据集成Agent：推荐数据源管理、数据映射工具
   - 监控Agent：推荐监控配置、告警管理工具
   - 运维Agent：推荐系统配置、日志管理工具

2. 基于用户角色推荐：
   - 管理员：推荐全量工具权限
   - 开发者：推荐开发相关工具权限
   - 运维人员：推荐运维相关工具权限

3. 基于历史委托推荐：
   - 分析用户历史委托配置
   - 推荐相似用户常用的委托策略
   - 推荐用户自己常用的委托策略

推荐算法：
- 协同过滤：基于相似用户的委托配置推荐
- 内容推荐：基于Agent类型和工具标签推荐
- 混合推荐：结合协同过滤和内容推荐

推荐展示：
- 推荐策略卡片：显示策略名称、工具数量、适用场景
- 一键应用：点击即可应用推荐策略
- 自定义修改：应用后可自定义修改
```

**方案2：可视化权限配置**
```
可视化设计：
1. 权限树形结构：
   - 根节点：工具分类（数据源、集成任务、监控等）
   - 子节点：具体工具
   - 叶子节点：工具操作（查询、创建、删除等）

2. 权限矩阵视图：
   - 行：工具列表
   - 列：操作类型（查询、创建、更新、删除）
   - 单元格：权限状态（✓允许、✗禁止、?需确认）

3. 权限时间线：
   - 显示委托有效期
   - 可拖拽调整有效期
   - 显示委托历史

交互设计：
- 点击节点：展开/折叠子节点
- 右键节点：快捷操作（全选、全不选、反选）
- 拖拽节点：调整权限范围
- 悬停节点：显示权限说明
```

**方案3：委托配置向导**
```
向导流程：
步骤1：选择Agent
- 显示已注册Agent列表
- 显示Agent类型、描述、推荐场景
- 支持搜索和筛选

步骤2：选择委托类型
- 长期委托：适用于长期协作场景
- 临时委托：适用于临时任务场景
- 会话委托：适用于单次会话场景
- 显示每种类型的适用场景和有效期建议

步骤3：配置权限范围
- 显示推荐策略
- 显示工具树形结构
- 支持可视化权限配置
- 显示权限影响预览

步骤4：配置数据范围
- 显示数据范围配置界面
- 支持租户、应用、数据源维度配置
- 显示数据范围影响预览

步骤5：配置确认规则
- 显示敏感工具列表
- 配置确认规则（实时确认、批量确认、免确认）
- 显示确认规则影响预览

步骤6：预览和确认
- 显示委托配置摘要
- 显示权限范围摘要
- 显示数据范围摘要
- 支持修改和确认

向导优化：
- 进度条显示：显示当前步骤和总步骤
- 步骤跳转：支持跳转到任意步骤
- 配置保存：支持保存草稿，下次继续
- 配置导入：支持导入已有委托配置
```

**用户体验基准测试**：
```
测试场景1：委托配置效率
- 测试用例：10个用户完成委托配置
- 预期结果：平均配置时间 < 2分钟，配置错误率 < 5%

测试场景2：推荐策略准确性
- 测试用例：推荐策略与用户实际需求的匹配度
- 预期结果：推荐策略匹配度 > 80%，用户采纳率 > 60%

测试场景3：用户满意度
- 测试用例：用户满意度调查
- 预期结果：用户满意度 > 90%，NPS > 50
```

---

**2. 实时确认弹窗交互优化**

**优化目标**：
- 确认响应时间 < 3秒
- 确认弹窗误触发率 < 5%
- 用户确认满意度 > 85%

**优化方案**：

**方案1：智能确认策略**
```
确认策略：
1. 工具风险分级：
   - 低风险工具：免确认（如查询操作）
   - 中风险工具：批量确认（每10次操作确认一次）
   - 高风险工具：实时确认（每次操作都确认）
   - 极高风险工具：二次确认（输入操作说明）

2. 用户信任分级：
   - 高信任用户：降低确认频率
   - 中信任用户：标准确认频率
   - 低信任用户：提高确认频率

3. 场景感知确认：
   - 工作时间：降低确认频率
   - 非工作时间：提高确认频率
   - 敏感数据操作：强制确认

智能决策：
- 使用决策树模型预测确认需求
- 特征：工具类型、用户信任度、时间、数据敏感度
- 输出：确认策略（免确认、批量确认、实时确认、二次确认）
```

**方案2：确认弹窗优化**
```
弹窗设计：
1. 弹窗内容：
   - 操作摘要：Agent名称、工具名称、操作类型
   - 风险提示：操作风险等级、可能影响
   - 操作详情：操作参数、目标对象
   - 确认选项：允许、拒绝、允许本次会话、允许24小时

2. 弹窗样式：
   - 风险等级颜色：低风险-绿色、中风险-黄色、高风险-红色
   - 弹窗位置：右上角悬浮，不遮挡主要内容
   - 弹窗动画：淡入淡出，不突兀

3. 弹窗交互：
   - 倒计时：30秒内未响应，自动拒绝
   - 快捷键：支持快捷键确认（Y允许、N拒绝）
   - 批量操作：支持批量确认多个操作

交互优化：
- 弹窗预加载：提前加载弹窗内容，减少延迟
- 弹窗缓存：相同操作使用缓存弹窗，减少渲染
- 弹窗聚合：短时间内多个操作聚合为一个弹窗
```

**方案3：确认历史管理**
```
历史记录：
1. 确认记录：
   - 记录每次确认的时间、用户、Agent、工具、操作
   - 记录用户的确认决策（允许、拒绝）
   - 记录确认时的上下文信息

2. 历史查询：
   - 支持按时间、Agent、工具、决策查询
   - 支持分页和排序
   - 支持导出确认历史

3. 历史分析：
   - 分析用户确认模式
   - 识别误触发模式
   - 优化确认策略

历史应用：
- 学习用户确认习惯，优化确认策略
- 识别异常确认模式，触发安全告警
- 提供确认历史查询，便于审计
```

**用户体验基准测试**：
```
测试场景1：确认响应时间
- 测试用例：100次实时确认操作
- 预期结果：平均响应时间 < 3秒，弹窗显示延迟 < 500ms

测试场景2：确认弹窗误触发
- 测试用例：100次操作，统计误触发次数
- 预期结果：误触发率 < 5%，用户投诉率 < 1%

测试场景3：用户确认满意度
- 测试用例：用户满意度调查
- 预期结果：用户满意度 > 85%，确认流程合理性评分 > 80分
```

---

**3. Agent推荐策略智能匹配**

**优化目标**：
- Agent推荐准确率 > 85%
- Agent推荐覆盖率 > 90%
- 用户采纳率 > 70%

**优化方案**：

**方案1：基于任务类型的推荐**
```
任务类型分类：
1. 数据集成任务：
   - 推荐Agent：数据集成Agent
   - 推荐工具：数据源管理、数据映射、集成任务配置

2. 监控告警任务：
   - 推荐Agent：监控Agent
   - 推荐工具：监控配置、告警规则、指标查询

3. 系统运维任务：
   - 推荐Agent：运维Agent
   - 推荐工具：系统配置、日志管理、性能优化

4. 数据分析任务：
   - 推荐Agent：分析Agent
   - 推荐工具：数据查询、报表生成、数据导出

推荐算法：
- 任务类型识别：使用NLP识别用户任务意图
- Agent匹配：根据任务类型匹配Agent
- 工具推荐：根据Agent推荐相关工具

匹配流程：
1. 用户输入任务描述
2. NLP识别任务类型和关键词
3. 匹配Agent类型和工具标签
4. 计算推荐分数（0-100分）
5. 返回Top-3推荐Agent
```

**方案2：基于用户画像的推荐**
```
用户画像维度：
1. 角色画像：
   - 管理员：关注系统配置、权限管理
   - 开发者：关注数据集成、API开发
   - 运维人员：关注监控告警、系统运维
   - 分析师：关注数据查询、报表生成

2. 行为画像：
   - 常用Agent：统计用户最常用的Agent
   - 常用工具：统计用户最常用的工具
   - 活跃时间：统计用户活跃时间段
   - 任务类型：统计用户常见任务类型

3. 偏好画像：
   - Agent偏好：用户对Agent的评分和反馈
   - 工具偏好：用户对工具的使用频率
   - 确认偏好：用户的确认策略偏好

推荐算法：
- 协同过滤：推荐相似用户常用的Agent
- 内容推荐：推荐符合用户画像的Agent
- 混合推荐：结合协同过滤和内容推荐

推荐流程：
1. 构建用户画像
2. 计算用户相似度
3. 匹配Agent特征
4. 计算推荐分数
5. 返回个性化推荐结果
```

**方案3：基于场景的推荐**
```
场景分类：
1. 新手场景：
   - 用户特征：首次使用、无历史记录
   - 推荐策略：推荐通用Agent、提供使用向导
   - 推荐Agent：数据集成Agent（最常用）

2. 日常场景：
   - 用户特征：有历史记录、常用Agent明确
   - 推荐策略：推荐常用Agent、快捷操作
   - 推荐Agent：用户最常用的Agent

3. 复杂场景：
   - 用户特征：任务复杂、需要多个Agent协作
   - 推荐策略：推荐Agent组合、提供协作方案
   - 推荐Agent：多个相关Agent

4. 异常场景：
   - 用户特征：操作失败、任务异常
   - 推荐策略：推荐排查Agent、提供解决方案
   - 推荐Agent：运维Agent、监控Agent

推荐流程：
1. 识别用户场景
2. 匹配场景推荐策略
3. 计算推荐分数
4. 返回场景化推荐结果
```

**推荐效果基准测试**：
```
测试场景1：任务类型推荐
- 测试用例：100个不同类型的任务
- 预期结果：推荐准确率 > 85%，覆盖率 > 90%

测试场景2：用户画像推荐
- 测试用例：100个用户，每人推荐3个Agent
- 预期结果：推荐准确率 > 80%，用户采纳率 > 70%

测试场景3：场景化推荐
- 测试用例：4种场景，每种场景25个任务
- 预期结果：场景识别准确率 > 90%，推荐满意度 > 85%
```

---

**4. 权限不足提示引导优化**

**优化目标**：
- 权限不足提示清晰度 > 90%
- 权限申请成功率 > 70%
- 用户投诉率 < 5%

**优化方案**：

**方案1：智能诊断提示**
```
诊断维度：
1. 权限缺失诊断：
   - 识别缺失的权限：工具权限、数据权限、操作权限
   - 分析权限缺失原因：未委托、委托过期、权限不足

2. 权限申请建议：
   - 推荐申请的权限：基于任务需求推荐
   - 推荐申请对象：推荐有权委托的用户
   - 申请流程引导：提供申请流程指引

3. 替代方案建议：
   - 推荐有权限的Agent：推荐可完成任务的Agent
   - 推荐有权限的工具：推荐可替代的工具
   - 推荐简化任务：推荐可完成的简化任务

诊断流程：
1. 解析操作失败原因
2. 识别权限缺失类型
3. 分析权限缺失原因
4. 生成诊断报告
5. 提供申请建议和替代方案
```

**方案2：可视化权限对比**
```
对比设计：
1. 权限对比视图：
   - 左侧：当前权限范围
   - 右侧：任务所需权限
   - 中间：权限差异高亮显示

2. 权限差距分析：
   - 工具权限差距：缺少的工具权限
   - 数据权限差距：缺少的数据权限
   - 操作权限差距：缺少的操作权限

3. 权限申请路径：
   - 显示权限申请路径
   - 显示审批人信息
   - 显示预计审批时间

交互设计：
- 点击权限差距：显示详细说明
- 点击申请路径：跳转到申请页面
- 点击审批人：显示审批人联系方式
```

**方案3：权限申请流程优化**
```
申请流程：
步骤1：识别权限需求
- 自动识别任务所需的权限
- 显示权限需求清单
- 支持用户手动调整

步骤2：选择申请对象
- 推荐有权委托的用户
- 显示用户的权限范围
- 支持搜索和筛选

步骤3：填写申请理由
- 提供申请理由模板
- 自动填充任务上下文
- 支持附件上传

步骤4：提交申请
- 显示申请摘要
- 支持预览和修改
- 提交申请并通知审批人

申请优化：
- 一键申请：常用权限一键申请
- 申请模板：提供常用申请模板
- 申请历史：查看申请历史和状态
- 申请提醒：审批超时自动提醒
```

**用户体验基准测试**：
```
测试场景1：权限不足提示清晰度
- 测试用例：100次权限不足场景
- 预期结果：提示清晰度 > 90%，用户理解率 > 85%

测试场景2：权限申请成功率
- 测试用例：100次权限申请
- 预期结果：申请成功率 > 70%，平均审批时间 < 4小时

测试场景3：用户投诉率
- 测试用例：统计权限不足相关的用户投诉
- 预期结果：投诉率 < 5%，投诉处理满意度 > 80%
```

---

#### 17.4.4 监控完善

**1. A2A调用链可视化监控**

**优化目标**：
- A2A调用链可视化完整性 100%
- A2A调用链查询延迟 < 500ms
- A2A调用链监控准确率 > 99%

**优化方案**：

**方案1：调用链数据采集**
```
采集维度：
1. 调用链节点：
   - 调用者：用户ID、Agent ID
   - 被调用者：Agent ID
   - 调用时间：开始时间、结束时间、耗时
   - 调用结果：成功、失败、错误信息

2. 调用链路径：
   - 调用链ID：全局唯一的调用链标识
   - 调用层级：调用深度（如：user→agent-001→agent-002）
   - 调用顺序：调用序号（如：1.1, 1.2, 2.1）

3. 调用链上下文：
   - 委托信息：委托ID、委托类型、委托权限
   - 工具信息：工具名称、工具参数、工具结果
   - 环境信息：appName、tenantId、sessionId

采集方式：
- 日志埋点：在关键节点埋点记录调用信息
- 链路追踪：使用OpenTelemetry进行链路追踪
- 数据存储：Elasticsearch存储调用链数据

采集流程：
1. Agent调用Agent时，生成调用链ID
2. 记录调用者、被调用者、调用时间
3. 传递调用链ID到被调用Agent
4. 被调用Agent记录调用信息
5. 调用完成后，写入Elasticsearch
```

**方案2：调用链可视化展示**
```
可视化设计：
1. 调用链拓扑图：
   - 节点：用户、Agent、工具
   - 边：调用关系
   - 颜色：成功-绿色、失败-红色、进行中-蓝色
   - 粗细：调用频率

2. 调用链时间线：
   - 横轴：时间
   - 纵轴：调用层级
   - 条形：调用耗时
   - 颜色：调用结果

3. 调用链详情：
   - 调用者信息
   - 被调用者信息
   - 调用参数
   - 调用结果
   - 错误信息

交互设计：
- 点击节点：显示节点详情
- 点击边：显示调用详情
- 拖拽节点：调整布局
- 缩放：查看调用链细节
- 过滤：按时间、Agent、结果过滤
```

**方案3：调用链监控告警**
```
监控指标：
1. 调用链成功率：
   - 计算公式：成功调用次数 / 总调用次数
   - 告警阈值：< 95%
   - 告警级别：WARNING

2. 调用链平均耗时：
   - 计算公式：总耗时 / 调用次数
   - 告警阈值：> 5秒
   - 告警级别：WARNING

3. 调用链深度：
   - 计算公式：调用链的最大层级
   - 告警阈值：> 5层
   - 告警级别：INFO

4. 调用链异常：
   - 调用失败、超时、权限不足等异常
   - 告警阈值：任何异常
   - 告警级别：ERROR

告警规则：
- 调用链成功率 < 95%：发送WARNING告警
- 调用链平均耗时 > 5秒：发送WARNING告警
- 调用链深度 > 5层：发送INFO告警
- 调用链异常：发送ERROR告警

告警通知：
- 通知渠道：邮件、短信、企业微信
- 通知对象：运维人员、Agent负责人
- 通知频率：同类型告警1小时内只通知一次
```

**监控基准测试**：
```
测试场景1：调用链可视化完整性
- 测试用例：100个A2A调用链
- 预期结果：可视化完整性100%，所有节点和边正确显示

测试场景2：调用链查询延迟
- 测试用例：查询1000个调用链
- 预期结果：平均查询延迟 < 500ms，P99延迟 < 1秒

测试场景3：调用链监控准确率
- 测试用例：监控100个调用链的成功率、耗时、深度
- 预期结果：监控准确率 > 99%，告警触发准确率 > 95%
```

---

**2. 委托关系拓扑可视化**

**优化目标**：
- 委托关系拓扑可视化完整性 100%
- 委托关系拓扑查询延迟 < 1秒
- 委托关系拓扑更新延迟 < 5秒

**优化方案**：

**方案1：委托关系数据建模**
```
数据模型：
1. 委托节点：
   - 节点类型：用户、Agent
   - 节点ID：用户ID、Agent ID
   - 节点属性：名称、类型、状态、创建时间

2. 委托边：
   - 边类型：委托关系
   - 边属性：委托ID、委托类型、权限范围、有效期、状态

3. 委托图：
   - 图类型：有向图
   - 图存储：Neo4j图数据库
   - 图索引：节点ID索引、边类型索引

建模流程：
1. 用户创建委托时，创建委托边
2. 委托变更时，更新委托边属性
3. 委托失效时，删除委托边
4. 定期同步委托关系到图数据库
```

**方案2：委托关系拓扑可视化**
```
可视化设计：
1. 拓扑图布局：
   - 力导向布局：节点自动分布
   - 层次布局：按委托层级分布
   - 环形布局：按委托类型分布

2. 节点样式：
   - 用户节点：圆形、蓝色
   - Agent节点：方形、绿色
   - 活跃节点：高亮显示
   - 失效节点：灰色显示

3. 边样式：
   - 长期委托：实线、粗线
   - 临时委托：虚线、细线
   - 会话委托：点线、细线
   - 活跃委托：高亮显示

交互设计：
- 点击节点：显示节点详情
- 点击边：显示委托详情
- 拖拽节点：调整布局
- 缩放：查看拓扑细节
- 过滤：按委托类型、状态、时间过滤
- 搜索：按节点ID、名称搜索
```

**方案3：委托关系分析**
```
分析维度：
1. 委托路径分析：
   - 查找委托路径：从用户到Agent的委托路径
   - 路径长度：委托链的长度
   - 路径权限：委托链的权限范围

2. 委托影响分析：
   - 影响范围：委托变更影响的Agent和用户
   - 影响评估：评估委托变更的影响
   - 风险评估：评估委托变更的风险

3. 委托统计：
   - 委托数量：每个用户的委托数量
   - 委托类型分布：长期、临时、会话委托的分布
   - 委托活跃度：委托的使用频率

分析流程：
1. 构建委托图
2. 执行委托路径查询
3. 分析委托影响范围
4. 生成分析报告
5. 提供可视化展示
```

**监控基准测试**：
```
测试场景1：委托关系拓扑可视化完整性
- 测试用例：100个委托关系
- 预期结果：可视化完整性100%，所有节点和边正确显示

测试场景2：委托关系拓扑查询延迟
- 测试用例：查询1000个委托关系的拓扑
- 预期结果：平均查询延迟 < 1秒，P99延迟 < 2秒

测试场景3：委托关系拓扑更新延迟
- 测试用例：创建、更新、删除委托关系
- 预期结果：拓扑更新延迟 < 5秒，可视化实时同步
```

---

**3. Token生命周期监控**

**优化目标**：
- Token生命周期监控完整性 100%
- Token生命周期监控延迟 < 1秒
- Token异常检测准确率 > 95%

**优化方案**：

**方案1：Token生命周期状态机**
```
状态定义：
1. CREATED：Token已创建
2. ACTIVE：Token已激活，正在使用
3. EXPIRED：Token已过期
4. REVOKED：Token已撤销
5. ABUSED：Token滥用被锁定

状态转换：
- CREATED → ACTIVE：Token首次使用
- ACTIVE → EXPIRED：Token过期
- ACTIVE → REVOKED：Token被撤销
- ACTIVE → ABUSED：Token滥用被锁定
- EXPIRED → REVOKED：过期Token被撤销

状态监控：
- 状态转换事件：记录每次状态转换
- 状态转换时间：记录状态转换的时间戳
- 状态转换原因：记录状态转换的原因

监控指标：
- Token创建速率：每分钟创建的Token数量
- Token激活速率：每分钟激活的Token数量
- Token过期速率：每分钟过期的Token数量
- Token撤销速率：每分钟撤销的Token数量
- Token滥用速率：每分钟滥用的Token数量
```

**方案2：Token使用监控**
```
监控维度：
1. 使用频率监控：
   - 每分钟调用次数
   - 每小时调用次数
   - 每天调用次数
   - 异常频率检测

2. 使用时间监控：
   - 首次使用时间
   - 最后使用时间
   - 使用时长
   - 非活跃时长

3. 使用工具监控：
   - 使用的工具列表
   - 工具调用频率
   - 敏感工具使用频率
   - 工具调用成功率

4. 使用结果监控：
   - 成功率
   - 失败率
   - 失败原因分布
   - 错误类型分布

监控流程：
1. Token使用时，记录使用事件
2. 实时更新监控指标
3. 检测异常使用模式
4. 触发告警或自动处理
```

**方案3：Token异常检测**
```
异常类型：
1. 频率异常：
   - 调用频率过高：每分钟 > 100次
   - 调用频率突增：频率突增 > 10倍
   - 调用频率异常波动：频率波动 > 50%

2. 时间异常：
   - 非工作时间使用：22:00-6:00频繁使用
   - 使用时长异常：连续使用 > 8小时
   - 非活跃时长异常：非活跃 > 30天后突然使用

3. 工具异常：
   - 敏感工具使用：频繁使用敏感工具
   - 工具组合异常：异常的工具组合
   - 工具调用失败：连续失败 > 3次

4. 结果异常：
   - 成功率异常：成功率 < 50%
   - 错误类型异常：权限错误频繁

检测算法：
- 统计检测：基于统计阈值检测异常
- 机器学习：使用Isolation Forest检测异常
- 规则引擎：使用Drools规则引擎检测异常

处理策略：
- 低风险异常：记录日志，继续使用
- 中风险异常：二次验证，限制使用
- 高风险异常：锁定Token，发送告警
```

**监控基准测试**：
```
测试场景1：Token生命周期监控完整性
- 测试用例：100个Token的生命周期
- 预期结果：监控完整性100%，所有状态转换正确记录

测试场景2：Token生命周期监控延迟
- 测试用例：实时监控1000个Token的使用
- 预期结果：监控延迟 < 1秒，告警触发延迟 < 5秒

测试场景3：Token异常检测准确率
- 测试用例：100个正常Token + 100个异常Token
- 预期结果：异常检测准确率 > 95%，误报率 < 5%
```

---

**4. 权限变更影响分析**

**优化目标**：
- 权限变更影响分析完整性 100%
- 权限变更影响分析延迟 < 3秒
- 权限变更影响评估准确率 > 90%

**优化方案**：

**方案1：权限变更影响范围分析**
```
影响范围维度：
1. 用户影响：
   - 直接受影响用户：权限变更的用户
   - 间接受影响用户：通过委托链受影响的用户
   - 影响用户数量：统计受影响的用户数量

2. Agent影响：
   - 直接受影响Agent：权限变更的Agent
   - 间接受影响Agent：通过委托链受影响的Agent
   - 影响Agent数量：统计受影响的Agent数量

3. 任务影响：
   - 进行中任务：受影响的进行中任务
   - 待执行任务：受影响的待执行任务
   - 影响任务数量：统计受影响的任务数量

4. 工具影响：
   - 可用工具变更：权限变更导致的工具可用性变化
   - 工具权限变更：工具操作权限的变化
   - 影响工具数量：统计受影响的工具数量

分析流程：
1. 解析权限变更内容
2. 构建权限依赖图
3. 计算直接影响范围
4. 计算间接影响范围
5. 生成影响分析报告
```

**方案2：权限变更影响评估**
```
评估维度：
1. 风险评估：
   - 高风险变更：影响 > 100个用户或 > 10个Agent
   - 中风险变更：影响 10-100个用户或 1-10个Agent
   - 低风险变更：影响 < 10个用户或 < 1个Agent

2. 影响评估：
   - 正面影响：权限扩大，功能增强
   - 负面影响：权限缩小，功能受限
   - 中性影响：权限调整，功能不变

3. 紧急程度评估：
   - 紧急变更：安全漏洞修复、权限泄露修复
   - 重要变更：业务需求变更、合规要求变更
   - 一般变更：优化调整、常规维护

评估流程：
1. 分析权限变更类型
2. 计算影响范围
3. 评估风险等级
4. 评估影响类型
5. 评估紧急程度
6. 生成评估报告
```

**方案3：权限变更预警和通知**
```
预警机制：
1. 变更前预警：
   - 预警时间：变更前24小时
   - 预警内容：变更内容、影响范围、风险评估
   - 预警对象：受影响的用户、Agent负责人

2. 变更中预警：
   - 预警时间：变更执行时
   - 预警内容：变更执行进度、执行结果
   - 预警对象：变更执行人、受影响的用户

3. 变更后预警：
   - 预警时间：变更完成后
   - 预警内容：变更结果、影响确认、问题反馈
   - 预警对象：变更执行人、受影响的用户

通知机制：
1. 通知渠道：
   - 邮件通知：发送变更通知邮件
   - 站内信：发送站内信通知
   - 企业微信：发送企业微信通知
   - 短信通知：紧急变更发送短信通知

2. 通知内容：
   - 变更摘要：变更内容、变更时间、变更原因
   - 影响范围：受影响的用户、Agent、工具
   - 应对措施：用户需要采取的措施
   - 联系方式：变更负责人联系方式

3. 通知跟踪：
   - 通知发送状态：已发送、已送达、已读
   - 通知确认状态：已确认、未确认
   - 通知反馈：用户反馈的问题和建议
```

**监控基准测试**：
```
测试场景1：权限变更影响分析完整性
- 测试用例：100个权限变更操作
- 预期结果：影响分析完整性100%，所有影响范围正确识别

测试场景2：权限变更影响分析延迟
- 测试用例：分析1000个权限变更的影响
- 预期结果：分析延迟 < 3秒，报告生成延迟 < 5秒

测试场景3：权限变更影响评估准确率
- 测试用例：评估100个权限变更的风险和影响
- 预期结果：评估准确率 > 90%，误报率 < 10%
```

---

**文档结束**
