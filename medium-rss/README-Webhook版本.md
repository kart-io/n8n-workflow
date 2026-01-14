# n8n 漫画生成工作流 - Webhook 版本

使用 **Webhook 触发器**替代 Form Trigger 的版本，更加稳定和通用。

## 🎯 为什么使用 Webhook 版本？

### 问题：Form Trigger 无法使用

某些 n8n 环境中，Form Trigger 节点可能：
- 不可用或被禁用
- 版本不兼容
- 权限受限

### 解决方案：Webhook 触发器

**Webhook 的优势**：
- ✅ **稳定性高**：核心节点，所有 n8n 版本都支持
- ✅ **兼容性好**：适用于任何部署环境
- ✅ **更灵活**：可以被任何 HTTP 客户端调用
- ✅ **易于集成**：可以与前端表单、API、自动化工具集成
- ✅ **可编程**：支持脚本调用，方便批量操作

## 🏗️ 工作流架构

### 节点流程（8个节点）

```
Webhook 触发器 (POST)
    ↓
验证输入数据 (IF)
    ├─ True  → Gemini 生成图片
    └─ False → 缺少必填字段（错误响应）
                  ↓
            检查生成结果 (IF)
                ├─ True  → 提取图片数据 → 保存到本地
                └─ False → 生成失败（错误响应）
```

### 工作流模式

- **Pattern**: Webhook Processing（Webhook 处理模式）
- **触发器**: Webhook (POST)
- **验证**: IF 节点检查必填字段
- **处理**: Gemini API 调用 + 图片提取
- **输出**: 文件保存 + JSON 响应
- **错误处理**: 多级错误捕获和响应

## 📦 节点配置详解

### 1. Webhook 触发器

**配置**：
```json
{
  "httpMethod": "POST",
  "path": "comic-generator",
  "responseMode": "lastNode",
  "responseData": "firstEntryJson"
}
```

**关键参数**：
- `httpMethod`: POST - 接收 POST 请求
- `path`: comic-generator - Webhook 路径
- `responseMode`: lastNode - 等待工作流完成后响应
- `responseData`: firstEntryJson - 返回最后一个节点的 JSON 数据

**生成的 URL**（激活后）：
```
https://your-n8n-instance.com/webhook/comic-generator
```

### 2. 验证输入数据 (IF)

**验证条件**：
- 检查 `$json.body.书名` 是否非空
- 检查 `$json.body.章节` 是否非空

**数据路径说明**：
- Webhook 接收的 POST 数据位于 `$json.body`
- 例如：`$json.body.书名` 访问书名字段

### 3. Gemini 生成图片

与 Gemini 原生版本相同，但表达式路径改为：
```javascript
// Form Trigger 版本
{{ $json.书名 }}

// Webhook 版本
{{ $json.body.书名 }}
```

### 4-8. 其他节点

与 Gemini 原生版本基本相同，只需调整数据访问路径。

## 🔑 准备工作

### 1. 获取 Gemini API Key

参考 [README-Gemini版本.md](./README-Gemini版本.md) 中的步骤。

### 2. 在 n8n 中配置凭证

1. **创建 Header Auth 凭证**：
   - 在 n8n 界面 → Credentials → New Credential
   - 类型：**Header Auth**
   - Name: `x-goog-api-key`
   - Value: 你的 Gemini API Key
   - 保存为：**Gemini API Key**

2. **关联到工作流**：
   - 打开 "漫画生成工作流 - Webhook 版本"
   - 选择 "Gemini 生成图片" 节点
   - Credentials → 选择 **Gemini API Key**

### 3. 创建保存目录

```bash
mkdir -p /tmp/comics
```

## 📥 导入和使用

### 导入工作流

1. 打开 n8n
2. 点击 "+" → "Import from File"
3. 选择 `comic-generation-webhook-workflow.json`
4. 配置 Gemini API Key 凭证
5. **激活工作流**

### 获取 Webhook URL

激活工作流后，点击 "Webhook 触发器" 节点，会显示：
- **Production URL**: 生产环境 URL（激活后使用）
- **Test URL**: 测试 URL（调试时使用）

## 🚀 使用方法

### 方法 1: 使用 curl 命令

```bash
curl -X POST https://your-n8n-instance.com/webhook/comic-generator \
  -H "Content-Type: application/json" \
  -d '{
    "书名": "小岛经济学",
    "章节": "第一章",
    "提示词": "生成像《足球小将》那种专业分镜结构、漫画叙事节奏、对白气泡、拟声词、画格布局、视角变化、动态镜头，中文对白"
  }'
```

### 方法 2: 使用 JavaScript (fetch)

```javascript
const response = await fetch('https://your-n8n-instance.com/webhook/comic-generator', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    书名: '小岛经济学',
    章节: '第一章',
    提示词: '生成像《足球小将》那种专业分镜结构...'
  })
});

const result = await response.json();
console.log(result);
```

### 方法 3: 使用 Python (requests)

```python
import requests

response = requests.post(
    'https://your-n8n-instance.com/webhook/comic-generator',
    json={
        '书名': '小岛经济学',
        '章节': '第一章',
        '提示词': '生成像《足球小将》那种专业分镜结构...'
    }
)

print(response.json())
```

### 方法 4: 创建 HTML 表单

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>漫画生成器 - Webhook 版本</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 600px;
            margin: 50px auto;
            padding: 20px;
        }
        .form-group {
            margin-bottom: 15px;
        }
        label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        input, textarea {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
            box-sizing: border-box;
        }
        textarea {
            min-height: 100px;
        }
        button {
            background-color: #4CAF50;
            color: white;
            padding: 10px 20px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
        }
        button:hover {
            background-color: #45a049;
        }
        button:disabled {
            background-color: #ccc;
            cursor: not-allowed;
        }
        #result {
            margin-top: 20px;
            padding: 15px;
            border-radius: 4px;
            display: none;
        }
        #result.success {
            background-color: #d4edda;
            border: 1px solid #c3e6cb;
            color: #155724;
        }
        #result.error {
            background-color: #f8d7da;
            border: 1px solid #f5c6cb;
            color: #721c24;
        }
    </style>
</head>
<body>
    <h1>漫画生成器 (Gemini - Webhook 版本)</h1>
    <p>使用 Google Gemini 原生图片生成能力，输入书名和章节信息，自动生成专业漫画分镜</p>

    <form id="comicForm">
        <div class="form-group">
            <label for="bookName">书名 *</label>
            <input type="text" id="bookName" name="书名" required placeholder="例如：小岛经济学">
        </div>

        <div class="form-group">
            <label for="chapter">章节 *</label>
            <input type="text" id="chapter" name="章节" required placeholder="例如：第一章">
        </div>

        <div class="form-group">
            <label for="prompt">提示词（可选）</label>
            <textarea id="prompt" name="提示词" placeholder="例如：生成像《足球小将》那种专业分镜结构、漫画叙事节奏、对白气泡、拟声词、画格布局、视角变化、动态镜头，中文对白"></textarea>
        </div>

        <button type="submit" id="submitBtn">生成漫画</button>
    </form>

    <div id="result"></div>

    <script>
        const form = document.getElementById('comicForm');
        const resultDiv = document.getElementById('result');
        const submitBtn = document.getElementById('submitBtn');

        // 替换为你的实际 Webhook URL
        const WEBHOOK_URL = 'https://your-n8n-instance.com/webhook/comic-generator';

        form.addEventListener('submit', async (e) => {
            e.preventDefault();

            // 禁用提交按钮
            submitBtn.disabled = true;
            submitBtn.textContent = '生成中...';

            // 隐藏之前的结果
            resultDiv.style.display = 'none';

            // 收集表单数据
            const formData = new FormData(e.target);
            const data = Object.fromEntries(formData);

            try {
                const response = await fetch(WEBHOOK_URL, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify(data)
                });

                const result = await response.json();

                // 显示结果
                resultDiv.style.display = 'block';

                if (result.success) {
                    resultDiv.className = 'success';
                    resultDiv.innerHTML = `
                        <h3>✅ ${result.message}</h3>
                        <p><strong>书名：</strong>${result.书名}</p>
                        <p><strong>章节：</strong>${result.章节}</p>
                        <p><strong>文件大小：</strong>${(result.imageSize / 1024).toFixed(2)} KB</p>
                        <p>图片已保存到服务器 /tmp/comics/ 目录</p>
                    `;
                } else {
                    resultDiv.className = 'error';
                    resultDiv.innerHTML = `
                        <h3>❌ 生成失败</h3>
                        <p><strong>错误：</strong>${result.error}</p>
                        <p><strong>详情：</strong>${result.message || result.details || '未知错误'}</p>
                    `;
                }
            } catch (error) {
                resultDiv.style.display = 'block';
                resultDiv.className = 'error';
                resultDiv.innerHTML = `
                    <h3>❌ 请求失败</h3>
                    <p>${error.message}</p>
                `;
            } finally {
                // 恢复提交按钮
                submitBtn.disabled = false;
                submitBtn.textContent = '生成漫画';
            }
        });
    </script>
</body>
</html>
```

**使用步骤**：
1. 将上面的 HTML 保存为 `comic-form.html`
2. 修改 `WEBHOOK_URL` 为你的实际 Webhook URL
3. 在浏览器中打开 HTML 文件
4. 填写表单并提交

## 📋 请求和响应格式

### 请求格式

**必填字段**：
```json
{
  "书名": "小岛经济学",
  "章节": "第一章"
}
```

**可选字段**：
```json
{
  "书名": "小岛经济学",
  "章节": "第一章",
  "提示词": "自定义漫画风格描述"
}
```

### 响应格式

**成功响应** (HTTP 200):
```json
{
  "书名": "小岛经济学",
  "章节": "第一章",
  "mimeType": "image/jpeg",
  "imageSize": 245678,
  "success": true,
  "message": "漫画生成成功"
}
```

**错误响应 - 缺少必填字段** (HTTP 200):
```json
{
  "success": false,
  "error": "缺少必填字段",
  "message": "请提供书名和章节信息",
  "required": ["书名", "章节"],
  "received": {
    "书名": "小岛经济学"
  }
}
```

**错误响应 - 生成失败** (HTTP 200):
```json
{
  "success": false,
  "error": "图片生成失败",
  "details": "..."
}
```

## 🆚 版本对比

| 特性 | Form Trigger 版本 | Webhook 版本 |
|------|------------------|-------------|
| **触发器** | Form Trigger | Webhook (POST) |
| **可用性** | 部分环境不支持 | ✅ 所有环境支持 |
| **用户界面** | ✅ 自动生成表单 | 需要自己创建前端 |
| **API 集成** | ❌ 不便集成 | ✅ 易于集成 |
| **批量调用** | ❌ 不支持 | ✅ 支持脚本调用 |
| **验证逻辑** | 表单自带验证 | ✅ 工作流内验证 |
| **错误处理** | 基础错误提示 | ✅ 详细错误响应 |
| **节点数量** | 6 个 | 8 个 |
| **复杂度** | 简单 | 中等 |

## 🔧 高级配置

### 添加 CORS 支持

如果需要从浏览器直接调用，需要添加 CORS 头。在工作流最后添加 "Set" 节点：

```json
{
  "headers": {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "POST, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type"
  }
}
```

### 添加认证

在 Webhook 节点配置中启用 "Authentication"：
1. 选择认证类型（Basic Auth 或 Header Auth）
2. 配置凭证
3. 只有提供正确凭证的请求才能执行工作流

### 添加速率限制

使用 n8n 的 Rate Limit 节点限制请求频率，防止滥用。

## 🐛 常见问题

### 1. 404 Not Found

**原因**：工作流未激活或 Webhook 路径错误

**解决**：
- 确保工作流已激活
- 检查 Webhook URL 是否正确
- 使用 Production URL，不是 Test URL

### 2. 422 Validation Error

**原因**：请求数据格式错误

**解决**：
- 确保使用 `Content-Type: application/json`
- 检查 JSON 格式是否正确
- 确保提供了必填字段（书名、章节）

### 3. 数据访问错误

**问题**：表达式 `{{ $json.书名 }}` 返回 undefined

**解决**：
- Webhook 数据在 `$json.body` 下
- 使用 `{{ $json.body.书名 }}`
- 可以在 Code 节点中打印 `console.log($input.item.json)` 查看数据结构

### 4. Gemini API 调用失败

参考 [README-Gemini版本.md](./README-Gemini版本.md) 中的"常见问题"章节。

## 📚 参考资源

- [n8n Webhook 文档](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)
- [Gemini API 文档](https://ai.google.dev/docs)
- [n8n Expression 语法](https://docs.n8n.io/code-examples/expressions/)
- [Webhook Processing Pattern](https://docs.n8n.io/workflows/components/webhook/)

## 📁 工作流文件

- **Webhook 版本**: `comic-generation-webhook-workflow.json` (本工作流)
- **Gemini 版本**: `comic-generation-gemini-workflow.json` (Form Trigger)
- **Nano Banana 版本**: `comic-generation-workflow.json` (异步轮询)

## ⚖️ 选择建议

**选择 Webhook 版本，如果**：
- ✅ Form Trigger 在你的环境中不可用
- ✅ 需要通过 API 调用工作流
- ✅ 需要自定义前端界面
- ✅ 需要批量调用或脚本自动化
- ✅ 需要与其他系统集成

**选择 Form Trigger 版本，如果**：
- ✅ Form Trigger 在你的环境中可用
- ✅ 希望快速部署，无需创建前端
- ✅ 只需要简单的表单界面
- ✅ 不需要 API 集成

## 🎉 总结

Webhook 版本提供了更强大和灵活的触发方式，虽然需要自己创建前端界面，但在集成性、可扩展性和稳定性方面都优于 Form Trigger。

**关键优势**：
- 🔧 **稳定可靠**：核心节点，所有环境支持
- 🔌 **易于集成**：RESTful API，标准 HTTP 调用
- 🎨 **自定义前端**：完全控制用户界面
- 🤖 **自动化友好**：支持脚本、批量处理
- ✅ **验证完善**：工作流内验证，详细错误响应
