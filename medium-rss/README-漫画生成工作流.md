# n8n 漫画生成工作流

基于文章《【21天精通n8n】day14：封神！n8n生成漫画工作流（调用最新Gemini3 + Nano Banana Pro）》实现的自动化漫画生成工作流。

## 功能概述

通过 n8n 工作流自动调用 Nano Banana Pro API（通过 apimart.ai 中转），将书籍章节内容转换为专业漫画分镜图。支持自定义漫画风格和叙事方式。

## 工作流架构

### 核心模式：异步轮询（Async Polling）

本工作流采用 **Webhook Processing** 和 **HTTP API Integration** 混合模式，实现异步图片生成：

```
表单提交 → 创建任务 → 检查响应 → 轮询状态 → 下载图片 → 保存文件
                              ↑              ↓
                              └─── 等待5秒 ←──┘
```

### 工作流程详解

1. **表单触发器** (Form Trigger)
   - 用户输入：书名、章节、提示词（可选）
   - 触发工作流执行

2. **创建生成任务** (HTTP POST)
   - 调用 apimart.ai API
   - 发送提示词到 Nano Banana Pro
   - 获取任务 ID (task_id)

3. **检查响应状态** (IF 节点)
   - 判断 HTTP 状态码是否为 200
   - 成功 → 进入轮询流程
   - 失败 → 进入错误处理

4. **等待5秒** (Wait 节点)
   - 给 AI 生成时间
   - 避免频繁请求

5. **查询任务状态** (HTTP GET)
   - 通过 task_id 查询生成进度
   - 获取 progress 值

6. **检查生成进度** (Switch 节点)
   - progress = 100 → 生成完成，下载图片
   - progress ≠ 100 → 继续等待，循环查询

7. **下载图片** (HTTP GET)
   - 从返回的 image_url 下载图片
   - 以二进制格式获取

8. **保存到本地** (Write Binary File)
   - 保存到 `/tmp/comics/` 目录
   - 文件名格式：`书名_章节_时间戳.png`

## 节点配置详情

### 1. 表单触发器 (formTrigger)

```json
{
  "formTitle": "漫画生成器",
  "formFields": [
    {"fieldLabel": "书名", "fieldType": "text", "requiredField": true},
    {"fieldLabel": "章节", "fieldType": "text", "requiredField": true},
    {"fieldLabel": "提示词（可选）", "fieldType": "textarea", "requiredField": false}
  ],
  "responseMode": "lastNode"
}
```

### 2. 创建生成任务 (httpRequest)

**URL**: `https://apimart.ai/api/v1/images/generations`

**方法**: POST

**认证**: Header Auth (需要配置 Apimart API Key)

**请求体**:
```json
{
  "model": "nano-banana-2",
  "prompt": "深刻理解《{{ $json.书名 }}》这本书，将{{ $json.章节 }}内容，{{ $json['提示词（可选）'] || '生成像《足球小将》那种专业分镜结构、漫画叙事节奏、对白气泡、拟声词、画格布局、视角变化、动态镜头，中文对白' }}",
  "n": 1,
  "size": "1024x1024",
  "quality": "hd"
}
```

**表达式说明**:
- `{{ $json.书名 }}` - 引用表单提交的书名
- `{{ $json.章节 }}` - 引用表单提交的章节
- `{{ $json['提示词（可选）'] || 'default...' }}` - 可选提示词，有则用，无则用默认值

### 3. 检查响应状态 (if)

```json
{
  "conditions": {
    "conditions": [
      {
        "leftValue": "={{ $json.statusCode }}",
        "rightValue": "200",
        "operator": {"type": "number", "operation": "equals"}
      }
    ]
  }
}
```

### 4. 等待5秒 (wait)

```json
{
  "resume": "timeInterval",
  "amount": 5,
  "unit": "seconds"
}
```

### 5. 查询任务状态 (httpRequest)

**URL**: `https://apimart.ai/api/v1/tasks/{{ $('创建生成任务').item.json.task_id }}`

**方法**: GET

**认证**: Header Auth

**表达式说明**:
- `$('创建生成任务').item.json.task_id` - 引用之前节点返回的任务 ID

### 6. 检查生成进度 (switch)

```json
{
  "rules": [
    {
      "outputKey": "completed",
      "conditions": [{
        "leftValue": "={{ $json.progress }}",
        "rightValue": "100",
        "operator": {"type": "number", "operation": "equals"}
      }]
    },
    {
      "outputKey": "processing",
      "conditions": [{
        "leftValue": "={{ $json.progress }}",
        "rightValue": "100",
        "operator": {"type": "number", "operation": "notEquals"}
      }]
    }
  ]
}
```

**输出分支**:
- `completed` → 下载图片
- `processing` → Loop Back → 等待5秒 → 继续查询

### 7. 下载图片 (httpRequest)

**URL**: `={{ $json.result.image_url }}`

**方法**: GET

**选项**: `responseFormat: "file"` (以二进制文件格式返回)

### 8. 保存到本地 (writeBinaryFile)

**文件名**: `=/tmp/comics/{{ $('表单触发器').item.json.书名 }}_{{ $('表单触发器').item.json.章节 }}_{{ $now.format('yyyyMMdd_HHmmss') }}.png`

**数据属性**: `data`

## 准备工作

### 1. 注册并配置 Apimart.ai

1. 访问 [apimart.ai](https://apimart.ai)
2. 使用 Google 或 GitHub 账号登录
3. 创建 API Key
4. 充值（支持支付宝/微信，建议先充值 $1 测试）

### 2. 在 n8n 中配置 Credential

1. 打开 n8n
2. 进入 Credentials 页面
3. 创建新凭证：**Header Auth**
4. 配置：
   - Name: `Authorization`
   - Value: `Bearer YOUR_API_KEY`（替换为你的 API Key）
5. 保存为 "Apimart API Key"

### 3. 创建保存目录

```bash
mkdir -p /tmp/comics
```

## 导入工作流

### 方法一：通过 n8n UI 导入

1. 打开 n8n
2. 点击右上角 "+" → "Import from File"
3. 选择 `comic-generation-workflow.json`
4. 配置 Credentials（关联到上面创建的 Apimart API Key）
5. 激活工作流

### 方法二：通过 n8n API 导入

```bash
# 使用 n8n MCP 工具
# 见下方 MCP 工具使用示例
```

## 使用说明

### 基本使用

1. 激活工作流后，获取表单 URL
2. 在浏览器中打开表单
3. 填写信息：
   - **书名**: 例如 "小岛经济学"
   - **章节**: 例如 "第一章"
   - **提示词（可选）**: 自定义漫画风格（留空则使用默认）
4. 点击"生成漫画"
5. 等待工作流执行完成
6. 在 `/tmp/comics/` 目录查看生成的图片

### 自定义提示词示例

**默认提示词**（足球小将风格）:
```
生成像《足球小将》那种专业分镜结构、漫画叙事节奏、对白气泡、拟声词、画格布局、视角变化、动态镜头，中文对白
```

**其他风格示例**:

1. **灌篮高手风格**:
```
模仿《灌篮高手》的经典漫画风格，写实人物比例，细腻的表情刻画，动态的运动场景，热血的氛围，中文对白
```

2. **龙珠风格**:
```
参考《龙珠》的分镜手法，夸张的动作表现，速度线和冲击波效果，紧凑的格子布局，强烈的战斗张力，中文对白
```

3. **名侦探柯南风格**:
```
采用《名侦探柯南》的叙事节奏，推理场景的细节刻画，悬念氛围营造，清晰的人物对话，中文对白
```

## 轮询机制说明

### 为什么需要轮询？

生成 4K 高质量漫画需要时间（通常 30-120 秒），API 采用异步模式：

1. POST 请求创建任务 → 立即返回 task_id
2. 后台开始生成图片
3. 客户端定期查询进度（轮询）
4. progress = 100 时下载完成的图片

### 轮询优化

**当前配置**: 每 5 秒查询一次

**可优化方向**:
- 增加轮询间隔到 10 秒（减少 API 调用）
- 添加最大轮询次数限制（防止无限循环）
- 添加超时处理（生成失败时停止）

## 常见问题

### 1. 循环依赖错误

**问题**: 验证时提示 "Workflow contains a cycle"

**原因**: 轮询逻辑形成了环路（等待5秒 → 查询状态 → 检查进度 → Loop Back → 等待5秒）

**解决**: 这是**设计内的循环**，不是错误。n8n 的 Wait 节点会暂停执行，不会造成真正的无限循环。在实际执行中，当 progress = 100 时会跳出循环。

### 2. 表达式警告

**问题**: 验证时提示表达式格式问题

**原因**: 中文属性名需要使用 `$json['属性名']` 格式

**说明**: 当前配置已正确处理，警告可忽略。

### 3. 图片 URL 只能访问一次

**问题**: 返回的 image_url 只能下载一次

**解决方案**:
- 工作流已配置自动保存到本地
- 后续可扩展：上传到云存储（S3、OSS 等）
- 后续可扩展：写入 Notion 数据库

### 4. API 调用失败

**可能原因**:
- API Key 未配置或错误
- apimart.ai 余额不足
- API 中转服务不稳定

**检查步骤**:
1. 验证 Credential 配置正确
2. 登录 apimart.ai 检查余额
3. 查看工作流执行日志中的错误信息

## 成本说明

**Apimart.ai 定价** (截至文章发布时):
- Nano Banana Pro: $0.025/图
- 充值 $1 可生成约 40 张图片

**其他中转平台**:
- OpenRouter.ai
- kie.ai: $0.12/图（更贵但可能更稳定）

## 扩展功能建议

### 1. 批量生成

添加循环节点，一次生成多个章节：

```
表单输入章节列表 → Split In Batches → 逐个生成 → 合并结果
```

### 2. 结果通知

生成完成后发送通知：

```
保存到本地 → Slack/邮件/飞书通知
```

### 3. 云存储集成

上传到云端而非本地：

```
下载图片 → Upload to S3/OSS → 获取公开 URL → 写入数据库
```

### 4. Notion 集成

记录生成历史：

```
保存到本地 → Notion: Create Page → 附加图片 + 元数据
```

### 5. 定时批量生成

配合 Schedule Trigger：

```
每日定时 → 从数据库读取待生成列表 → 批量生成 → 结果归档
```

## 技术要点

### n8n 表达式语法

1. **访问当前节点数据**:
   - `{{ $json.字段名 }}`
   - `{{ $json['中文字段'] }}`

2. **访问其他节点数据**:
   - `{{ $('节点名称').item.json.字段名 }}`

3. **条件默认值**:
   - `{{ $json.可选字段 || '默认值' }}`

4. **时间格式化**:
   - `{{ $now.format('yyyyMMdd_HHmmss') }}`

### 异步轮询最佳实践

1. **设置合理的等待时间**: 太短浪费 API 调用，太长用户体验差
2. **添加超时机制**: 防止永久等待
3. **错误处理**: 处理 API 失败、网络问题
4. **状态记录**: 记录每次查询的结果便于调试

### 循环控制技巧

使用 **Switch 节点** 而非 **If 节点**:
- Switch 支持多分支
- 可明确命名输出（completed、processing）
- 避免直接循环连接，使用中间 NoOp 节点（Loop Back）

## 参考资源

- [Nano Banana 2 API 文档](https://apimart.ai/model/nano-banana-2-api)
- [Apimart 任务管理文档](https://docs.apimart.ai/en/api-reference/tasks/status)
- [n8n 官方文档](https://docs.n8n.io/)
- [n8n 表达式语法](https://docs.n8n.io/code-examples/expressions/)

## 工作流文件

- **工作流 JSON**: `comic-generation-workflow.json`
- **使用说明**: 本文档

## 作者信息

基于张卫泉的文章《【21天精通n8n】day14》实现。

原文链接: https://mp.weixin.qq.com/s/P-B4TqBDyia2FqdajMMoWg

## 致谢

感谢 Gemini 团队和 Nano Banana Pro 团队提供强大的 AI 图片生成能力！
