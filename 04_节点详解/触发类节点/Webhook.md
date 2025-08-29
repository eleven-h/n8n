# Webhook 节点详解

## 🎯 什么是 Webhook？

Webhook 是一种**HTTP 回调机制**，允许外部系统在特定事件发生时向 n8n 发送 HTTP 请求，从而触发工作流执行。

## 🔗 工作原理

```
外部系统 → HTTP 请求 → n8n Webhook → 触发工作流 → 执行后续节点
```

### 典型应用场景
- **表单提交**：用户填写表单后自动处理
- **支付通知**：支付平台回调确认支付状态
- **Git 事件**：代码推送、合并请求等触发自动化
- **第三方集成**：其他系统的事件通知

## ⚙️ 节点配置

### 1. 基础配置
```json
{
  "name": "Webhook",
  "type": "n8n-nodes-base.webhook",
  "position": [0, 0],
  "parameters": {
    "httpMethod": "POST",
    "path": "webhook",
    "responseMode": "responseNode",
    "options": {}
  }
}
```

### 2. 配置参数详解

#### HTTP 方法
- **GET**：获取数据，适合简单查询
- **POST**：提交数据，最常用的触发方式
- **PUT**：更新数据
- **DELETE**：删除数据
- **PATCH**：部分更新数据

#### 路径设置
```bash
# 完整 URL 示例
http://your-n8n-domain:5678/webhook/your-custom-path

# 路径参数
/webhook/user/{userId}/action/{actionType}
```

#### 响应模式
- **responseNode**：需要专门的响应节点
- **onReceived**：自动响应，适合简单场景
- **responseCode**：返回指定 HTTP 状态码

## 🚀 实际应用示例

### 示例 1：简单的表单处理
```json
{
  "name": "表单提交 Webhook",
  "type": "n8n-nodes-base.webhook",
  "parameters": {
    "httpMethod": "POST",
    "path": "form-submit",
    "responseMode": "onReceived",
    "responseCode": 200,
    "responseData": "firstEntryJson"
  }
}
```

**工作流设计**：
```
Webhook → 数据验证 → 保存到数据库 → 发送确认邮件
```

### 示例 2：支付回调处理
```json
{
  "name": "支付回调 Webhook",
  "type": "n8n-nodes-base.webhook",
  "parameters": {
    "httpMethod": "POST",
    "path": "payment-callback",
    "responseMode": "responseNode",
    "authentication": "headerAuth",
    "headerAuthName": "X-API-Key",
    "headerAuthValue": "your-secret-key"
  }
}
```

**工作流设计**：
```
Webhook → 验证签名 → 更新订单状态 → 发送通知 → 返回响应
```

### 示例 3：Git 事件处理
```json
{
  "name": "Git Push Webhook",
  "type": "n8n-nodes-base.webhook",
  "parameters": {
    "httpMethod": "POST",
    "path": "git-push",
    "responseMode": "onReceived",
    "responseCode": 200
  }
}
```

**工作流设计**：
```
Webhook → 解析 Git 事件 → 自动部署 → 发送通知
```

## 🔐 安全配置

### 1. 身份验证
```json
{
  "authentication": "headerAuth",
  "headerAuthName": "Authorization",
  "headerAuthValue": "Bearer your-token"
}
```

### 2. 路径安全
```bash
# 使用随机路径
/webhook/abc123-def456-ghi789

# 使用时间戳
/webhook/2024-01-15-14-30-00

# 使用 UUID
/webhook/550e8400-e29b-41d4-a716-446655440000
```

### 3. 请求验证
```javascript
// 在 Code 节点中验证请求
const signature = $input.first().json.headers['x-signature'];
const payload = $input.first().json.body;
const expectedSignature = crypto
  .createHmac('sha256', 'your-secret')
  .update(JSON.stringify(payload))
  .digest('hex');

if (signature !== expectedSignature) {
  throw new Error('Invalid signature');
}
```

## 📊 数据处理

### 1. 请求数据结构
```json
{
  "headers": {
    "content-type": "application/json",
    "user-agent": "Mozilla/5.0...",
    "authorization": "Bearer token"
  },
  "body": {
    "event": "user.created",
    "data": {
      "userId": "12345",
      "email": "user@example.com"
    }
  },
  "query": {
    "source": "website",
    "campaign": "winter2024"
  }
}
```

### 2. 数据提取
```javascript
// 提取请求体数据
const eventData = $input.first().json.body;

// 提取查询参数
const source = $input.first().json.query.source;

// 提取请求头
const userAgent = $input.first().json.headers['user-agent'];
```

### 3. 数据验证
```javascript
// 验证必需字段
const requiredFields = ['userId', 'email', 'action'];
for (const field of requiredFields) {
  if (!eventData[field]) {
    throw new Error(`Missing required field: ${field}`);
  }
}

// 验证数据类型
if (typeof eventData.userId !== 'string') {
  throw new Error('userId must be a string');
}
```

## 🔄 响应处理

### 1. 成功响应
```json
{
  "status": "success",
  "message": "Data processed successfully",
  "timestamp": "2024-01-15T14:30:00Z",
  "data": {
    "processedId": "abc123"
  }
}
```

### 2. 错误响应
```json
{
  "status": "error",
  "message": "Invalid input data",
  "errorCode": "VALIDATION_ERROR",
  "details": [
    "Email format is invalid",
    "UserId is required"
  ]
}
```

### 3. 响应节点配置
```json
{
  "name": "Respond to Webhook",
  "type": "n8n-nodes-base.respondToWebhook",
  "parameters": {
    "respondWith": "json",
    "responseBody": "={{ $json }}",
    "responseCode": 200,
    "responseHeaders": {
      "content-type": "application/json",
      "x-processed-by": "n8n"
    }
  }
}
```

## 🧪 测试方法

### 1. 使用 curl 测试
```bash
# 测试 GET 请求
curl -X GET "http://localhost:5678/webhook/test"

# 测试 POST 请求
curl -X POST "http://localhost:5678/webhook/test" \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'

# 测试带认证的请求
curl -X POST "http://localhost:5678/webhook/secure" \
  -H "Authorization: Bearer your-token" \
  -H "Content-Type: application/json" \
  -d '{"event": "test"}'
```

### 2. 使用 Postman 测试
1. 创建新的请求
2. 设置请求方法和 URL
3. 添加请求头和请求体
4. 发送请求并查看响应

### 3. 使用在线工具测试
- **Webhook.site**：生成临时 webhook URL 进行测试
- **RequestBin**：创建请求接收器
- **ngrok**：本地开发时的公网访问

## 🚨 常见问题和解决方案

### 1. Webhook 不触发
**可能原因**：
- 工作流未激活
- 路径配置错误
- 防火墙阻止请求

**解决方案**：
```bash
# 检查工作流状态
# 确认路径配置
# 检查网络连接
# 查看 n8n 日志
```

### 2. 请求超时
**可能原因**：
- 工作流执行时间过长
- 网络延迟
- 服务器负载过高

**解决方案**：
```javascript
// 设置超时时间
const timeout = setTimeout(() => {
  // 处理超时逻辑
}, 30000);

// 异步处理
setImmediate(() => {
  // 立即返回响应，后台处理
});
```

### 3. 数据格式错误
**可能原因**：
- 请求体格式不匹配
- 编码问题
- 数据类型错误

**解决方案**：
```javascript
// 数据格式验证
try {
  const data = JSON.parse($input.first().json.body);
} catch (error) {
  throw new Error('Invalid JSON format');
}

// 数据类型转换
const userId = String($input.first().json.body.userId);
```

## 🎯 最佳实践

### 1. 设计原则
- **幂等性**：多次调用产生相同结果
- **安全性**：验证请求来源和内容
- **可靠性**：处理异常和错误情况
- **性能**：快速响应，避免阻塞

### 2. 错误处理
```javascript
try {
  // 处理业务逻辑
  const result = await processData($input.first().json.body);
  return { success: true, data: result };
} catch (error) {
  // 记录错误日志
  console.error('Webhook processing error:', error);
  
  // 返回错误响应
  return { 
    success: false, 
    error: error.message,
    timestamp: new Date().toISOString()
  };
}
```

### 3. 监控和日志
```javascript
// 记录请求信息
console.log('Webhook received:', {
  timestamp: new Date().toISOString(),
  path: $input.first().json.path,
  method: $input.first().json.method,
  headers: $input.first().json.headers,
  body: $input.first().json.body
});
```

## 📚 进阶技巧

### 1. 批量处理
```javascript
// 处理批量数据
const items = $input.first().json.body.items;
const results = [];

for (const item of items) {
  try {
    const result = await processItem(item);
    results.push({ id: item.id, success: true, result });
  } catch (error) {
    results.push({ id: item.id, success: false, error: error.message });
  }
}

return { processed: results.length, results };
```

### 2. 异步处理
```javascript
// 立即返回响应，后台处理
setImmediate(async () => {
  try {
    await processDataAsync($input.first().json.body);
    console.log('Background processing completed');
  } catch (error) {
    console.error('Background processing failed:', error);
  }
});

// 返回立即响应
return { status: 'accepted', message: 'Processing started' };
```

### 3. 条件处理
```javascript
// 根据请求内容选择处理方式
const eventType = $input.first().json.body.event;

switch (eventType) {
  case 'user.created':
    return await handleUserCreated($input.first().json.body.data);
  case 'user.updated':
    return await handleUserUpdated($input.first().json.body.data);
  case 'user.deleted':
    return await handleUserDeleted($input.first().json.body.data);
  default:
    throw new Error(`Unknown event type: ${eventType}`);
}
```

## 🎉 总结

Webhook 节点是 n8n 中最常用的触发节点，它让您的工作流能够响应外部事件，实现真正的自动化。

**关键要点**：
1. **理解原理**：HTTP 回调机制
2. **安全配置**：身份验证和路径安全
3. **数据处理**：请求解析和验证
4. **响应处理**：成功和错误响应
5. **最佳实践**：幂等性、安全性、可靠性

**下一步**：学习其他触发节点，如 Schedule、Email Trigger 等，构建更复杂的自动化流程。
