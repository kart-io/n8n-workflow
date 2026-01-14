# n8n 漫画生成工作流 - Gemini 原生版本

使用 **Google Gemini 2.0 Flash** 原生图片生成能力，无需中转服务，直接调用官方 API 生成专业漫画分镜。

## 🎯 为什么选择 Gemini 原生版本？

### 优势对比

| 特性 | Gemini 原生版本 | Nano Banana Pro (apimart) |
|------|----------------|---------------------------|
| **API 调用** | 直接调用 Google API | 需要中转服务 |
| **响应速度** | **同步返回**，无需轮询 | 异步轮询，需等待 30-120 秒 |
| **工作流复杂度** | **简单**（6 个节点） | 复杂（10 个节点 + 循环） |
| **稳定性** | 官方 API，高稳定性 | 依赖第三方中转 |
| **访问限制** | 需要科学上网或代理 | 国内可直接访问 |
| **成本** | 免费额度 + 按需付费 | $0.025/图 |
| **图片质量** | Gemini 2.0 最新模型 | Nano Banana 2 |

### 核心差异

**Gemini 版本的最大优势**：
- ✅ **同步生成**：API 直接返回图片的 base64 数据，无需轮询
- ✅ **简化流程**：去除了等待、查询、循环等复杂逻辑
- ✅ **官方支持**：Google 官方 API，长期稳定

**适用场景**：
- 如果你有科学上网条件 → 选择 **Gemini 原生版本**
- 如果在国内无法访问 Google API → 选择 **Nano Banana Pro 版本**

## 🏗️ 工作流架构

### 简化的同步流程

```
表单提交 → Gemini 生成图片 → 检查结果 → 提取图片数据 → 保存到本地
                                    ↓
                                处理错误
```

**对比异步轮询版本**（10 个节点）：
```
表单提交 → 创建任务 → 检查响应 → 轮询查询 → 下载保存
                              ↑         ↓
                              └─ 等待5秒 ←┘
```

**节省了**：
- ❌ 等待节点
- ❌ 查询任务状态节点
- ❌ 检查进度 Switch 节点
- ❌ Loop Back 循环节点
- ❌ 下载图片节点

## 📦 节点配置详解

### 1. 表单触发器 (formTrigger)

与 Nano Banana 版本完全相同，收集用户输入：

```json
{
  "formTitle": "漫画生成器 (Gemini)",
  "formFields": [
    {"fieldLabel": "书名", "fieldType": "text", "requiredField": true},
    {"fieldLabel": "章节", "fieldType": "text", "requiredField": true},
    {"fieldLabel": "提示词（可选）", "fieldType": "textarea", "requiredField": false}
  ],
  "responseMode": "lastNode"
}
```

### 2. Gemini 生成图片 (httpRequest)

**关键配置**：

**URL**:
```
https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=YOUR_API_KEY
```

**方法**: POST

**请求体**:
```json
{
  "contents": [{
    "parts": [{
      "text": "深刻理解《{{ $json.书名 }}》这本书，将{{ $json.章节 }}内容，{{ $json['提示词（可选）'] || '默认提示词...' }}"
    }]
  }],
  "generationConfig": {
    "response_modalities": ["IMAGE"],
    "response_mime_type": "image/jpeg"
  }
}
```

**重要参数说明**：

- `response_modalities: ["IMAGE"]` - **仅返回图片**，不返回文本
- `response_mime_type: "image/jpeg"` - 指定图片格式为 JPEG
- 如果需要同时返回文本和图片：`response_modalities: ["TEXT", "IMAGE"]`

**响应格式**：
```json
{
  "candidates": [{
    "content": {
      "parts": [{
        "inline_data": {
          "mime_type": "image/jpeg",
          "data": "base64_encoded_image_data..."
        }
      }]
    }
  }]
}
```

### 3. 检查生成结果 (if)

验证 API 是否成功返回图片：

```json
{
  "conditions": [{
    "leftValue": "={{ $json.candidates && $json.candidates.length > 0 }}",
    "rightValue": "true",
    "operator": {"type": "boolean", "operation": "true"}
  }]
}
```

### 4. 提取图片数据 (code)

**核心逻辑**：将 Gemini 返回的 base64 数据转换为 n8n 二进制格式

```javascript
// 提取 Gemini 返回的图片数据
const candidates = $input.item.json.candidates;

if (!candidates || candidates.length === 0) {
  throw new Error('No candidates found in response');
}

const parts = candidates[0].content.parts;
if (!parts || parts.length === 0) {
  throw new Error('No parts found in content');
}

// 查找 inline_data
let imageData = null;
let mimeType = 'image/jpeg';

for (const part of parts) {
  if (part.inline_data) {
    imageData = part.inline_data.data;
    mimeType = part.inline_data.mime_type || 'image/jpeg';
    break;
  }
}

if (!imageData) {
  throw new Error('No image data found in response');
}

// 将 base64 数据转换为二进制
const buffer = Buffer.from(imageData, 'base64');

// 获取原始表单数据
const formData = $('表单触发器').item.json;

return {
  json: {
    书名: formData.书名,
    章节: formData.章节,
    mimeType: mimeType,
    imageSize: buffer.length
  },
  binary: {
    data: {
      data: buffer.toString('base64'),
      mimeType: mimeType,
      fileName: `${formData.书名}_${formData.章节}_${new Date().getTime()}.jpg`
    }
  }
};
```

**代码说明**：
1. 从响应中提取 `candidates[0].content.parts`
2. 查找 `inline_data` 字段（包含 base64 图片）
3. 将 base64 解码为 Buffer
4. 转换为 n8n 的 binary 格式（用于后续保存）

### 5. 保存到本地 (writeBinaryFile)

```json
{
  "fileName": "=/tmp/comics/{{ $json.书名 }}_{{ $json.章节 }}_{{ $now.format('yyyyMMdd_HHmmss') }}.jpg",
  "dataPropertyName": "data"
}
```

### 6. 处理错误 (code)

```javascript
// 错误处理：返回错误信息
const error = $input.item.json;

return {
  json: {
    success: false,
    error: '图片生成失败',
    details: JSON.stringify(error, null, 2)
  }
};
```

## 🔑 准备工作

### 1. 获取 Gemini API Key

#### 方法一：通过 Google AI Studio（推荐）

1. 访问 [Google AI Studio](https://aistudio.google.com/app/apikey)
2. 登录你的 Google 账号
3. 点击 "Get API Key" 或 "Create API Key"
4. 选择一个 Google Cloud 项目（或创建新项目）
5. 复制生成的 API Key

#### 方法二：通过 Google Cloud Console

1. 访问 [Google Cloud Console](https://console.cloud.google.com/)
2. 创建或选择一个项目
3. 启用 "Generative Language API"
4. 创建凭据（API Key）
5. 复制 API Key

### 2. 在 n8n 中配置 Credential（推荐方式）

**使用 Header Auth 凭证**（最佳实践）

1. **创建新凭证**：
   - 在 n8n 界面中进入 **Credentials** 页面
   - 点击 **New Credential**
   - 选择类型：**Header Auth**

2. **配置凭证**：
   - Name: `x-goog-api-key`
   - Value: 你的 Gemini API Key（例如：`AIzaSyXXXXXXXXXXXXXX`）
   - 保存为：**Gemini API Key**

3. **关联到工作流**：
   - 打开 "漫画生成工作流 - Gemini 原生版本"
   - 点击 "Gemini 生成图片" 节点
   - 在 **Authentication** 部分选择 **Header Auth**
   - 选择刚才创建的 **Gemini API Key** 凭证
   - 保存工作流

**为什么使用凭证管理？**

✅ **安全性**：API Key 加密存储，不会明文出现在工作流 JSON 中
✅ **可维护性**：一处修改凭证，所有使用该凭证的工作流自动更新
✅ **可重用性**：多个工作流可以共享同一个凭证
✅ **最佳实践**：符合 n8n 官方推荐的凭证管理方式

### 3. 创建保存目录

```bash
mkdir -p /tmp/comics
```

## 📥 导入和使用

### 导入工作流

1. 打开 n8n
2. 点击右上角 "+" → "Import from File"
3. 选择 `comic-generation-gemini-workflow.json`
4. 配置 Gemini API Key（见上方）
5. 激活工作流

### 使用步骤

1. 获取表单 URL（激活工作流后自动生成）
2. 在浏览器中打开表单
3. 填写信息：
   - **书名**: 例如 "小岛经济学"
   - **章节**: 例如 "第一章"
   - **提示词**: 自定义漫画风格（可选）
4. 点击 "生成漫画"
5. **等待 5-15 秒**（同步生成，无需轮询）
6. 在 `/tmp/comics/` 查看生成的图片

## 🆚 两个版本对比

### Gemini 原生版本（本版本）

**优点**：
- ✅ 同步返回，响应快（5-15 秒）
- ✅ 工作流简单（6 个节点）
- ✅ 官方 API，稳定可靠
- ✅ 免费额度 + 按需付费
- ✅ Gemini 2.0 最新模型

**缺点**：
- ❌ 需要科学上网或代理
- ❌ 需要 Google 账号
- ❌ 可能受到地区限制

**适合**：
- 有稳定科学上网条件的用户
- 追求稳定性和官方支持
- 希望简化工作流逻辑

### Nano Banana Pro 版本（apimart.ai）

**优点**：
- ✅ 国内可直接访问
- ✅ 支持支付宝/微信充值
- ✅ 无需科学上网

**缺点**：
- ❌ 异步轮询，需等待 30-120 秒
- ❌ 工作流复杂（10 个节点 + 循环）
- ❌ 依赖第三方中转服务
- ❌ 需要付费（$0.025/图）

**适合**：
- 国内用户，无法访问 Google API
- 追求便捷性（支持国内支付）
- 不在意等待时间

## 💰 成本对比

### Gemini API 定价（截至 2026年1月）

**免费额度**（每天）：
- Gemini 2.0 Flash: 1500 requests/day
- 图片生成：包含在请求额度内

**付费定价**：
- 具体价格参考 [Google AI Pricing](https://ai.google.dev/pricing)
- 通常比第三方中转服务便宜

### Nano Banana Pro 定价

- $0.025/图（约 ¥0.18）
- 需通过 apimart.ai 充值

## 🔧 高级配置

### 1. 自定义图片参数

在 "Gemini 生成图片" 节点的 `generationConfig` 中添加：

```json
{
  "generationConfig": {
    "response_modalities": ["IMAGE"],
    "response_mime_type": "image/jpeg",
    "candidateCount": 1,
    "temperature": 0.9,
    "topP": 0.95
  }
}
```

### 2. 同时返回文本和图片

修改 `response_modalities`:

```json
{
  "generationConfig": {
    "response_modalities": ["TEXT", "IMAGE"],
    "response_mime_type": "image/jpeg"
  }
}
```

然后在 "提取图片数据" 节点中同时处理文本：

```javascript
// 提取文本
let textContent = '';
for (const part of parts) {
  if (part.text) {
    textContent += part.text;
  }
}

return {
  json: {
    书名: formData.书名,
    章节: formData.章节,
    generatedText: textContent, // 新增
    mimeType: mimeType,
    imageSize: buffer.length
  },
  binary: { /* ... */ }
};
```

### 3. 使用 Imagen 3 模型（更高质量）

更改 API URL:

```
https://generativelanguage.googleapis.com/v1beta/models/imagen-3.0-generate-001:generateContent
```

配置参数：

```json
{
  "prompt": "your prompt here",
  "number_of_images": 1,
  "aspect_ratio": "1:1",
  "safety_filter_level": "block_some",
  "person_generation": "allow_adult"
}
```

## 🐛 常见问题

### 1. API 调用失败 403/401

**原因**：API Key 无效或未启用 API

**解决**：
1. 检查 API Key 是否正确
2. 确认已启用 "Generative Language API"
3. 检查 API Key 是否有使用限制

### 2. 返回错误 "Region not supported"

**原因**：Gemini API 在某些地区不可用

**解决**：
1. 使用代理服务器
2. 更换 Google Cloud 项目的地区设置
3. 或使用 Nano Banana Pro 版本

### 3. 图片提取失败

**原因**：响应格式不符合预期

**解决**：
1. 检查 `response_modalities` 是否包含 `"IMAGE"`
2. 查看完整响应 JSON，确认结构
3. 在 Code 节点中添加调试日志

### 4. 生成速度慢

**原因**：复杂提示词或网络延迟

**优化**：
1. 简化提示词
2. 使用更快的模型（gemini-2.0-flash）
3. 检查网络连接

## 🚀 扩展功能

### 1. 批量生成

添加 Split In Batches 节点：

```
表单输入章节列表 → Split In Batches → Gemini 生成 → 保存
```

### 2. 多模型对比

并行调用多个模型：

```
表单提交 → [Gemini 分支]
         → [Imagen 3 分支]
         → [DALL-E 分支]
         → 合并结果
```

### 3. 云存储集成

上传到 Google Drive/S3：

```
保存到本地 → Upload to Google Drive → 获取分享链接
```

### 4. Notion 集成

记录生成历史：

```
保存到本地 → Notion: Create Page → 嵌入图片 + 元数据
```

## 📚 参考资源

- [Gemini API 官方文档](https://ai.google.dev/docs)
- [Gemini 图片生成指南](https://ai.google.dev/gemini-api/docs/vision)
- [Google AI Studio](https://aistudio.google.com/)
- [Gemini API Cookbook](https://github.com/google-gemini/cookbook)
- [n8n 官方文档](https://docs.n8n.io/)

## 📁 工作流文件

- **Gemini 版本**: `comic-generation-gemini-workflow.json`
- **Nano Banana 版本**: `comic-generation-workflow.json`
- **使用说明**: 本文档

## ⚖️ 选择建议

**选择 Gemini 原生版本，如果**：
- ✅ 你有稳定的科学上网条件
- ✅ 追求官方支持和长期稳定性
- ✅ 希望简化工作流逻辑
- ✅ 想要更快的响应速度（同步返回）

**选择 Nano Banana Pro 版本，如果**：
- ✅ 你在国内，无法访问 Google API
- ✅ 需要国内支付方式（支付宝/微信）
- ✅ 不在意等待时间和工作流复杂度
- ✅ 想要专门的漫画风格模型

## 🎉 总结

Gemini 原生版本将原本 10 个节点的复杂异步轮询流程，简化为 **6 个节点的同步流程**，大幅提升了：
- ⚡ **响应速度**：从 30-120 秒缩短到 5-15 秒
- 🧩 **维护性**：去除循环逻辑，易于理解和调试
- 💰 **成本**：免费额度 + 官方定价，更加透明

如果你有条件使用 Google API，强烈推荐选择 Gemini 原生版本！
