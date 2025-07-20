# Gemini Balance 项目技术迁移文档

## 项目概述

**Gemini Balance** 是一个基于Python FastAPI的Gemini API代理和负载均衡服务，主要功能包括：
- 多API Key负载均衡
- OpenAI格式API兼容
- 图像生成和处理
- 文本转语音(TTS)
- 流式响应优化
- 密钥管理和监控

## 核心架构分析

### 1. 应用架构

```
Next.js 等价架构建议:
├── app/                    # Next.js App Router
│   ├── api/               # API Routes
│   │   ├── v1/           # OpenAI兼容接口
│   │   ├── gemini/       # Gemini原生接口
│   │   └── admin/        # 管理接口
│   ├── dashboard/        # 管理面板页面
│   └── components/       # React组件
├── lib/                  # 核心业务逻辑
│   ├── services/         # 业务服务层
│   ├── clients/          # API客户端
│   ├── database/         # 数据库操作
│   └── utils/           # 工具函数
└── types/               # TypeScript类型定义
```

### 2. 数据库设计

**表结构 (需要在Next.js中实现)**

```sql
-- 配置表
CREATE TABLE t_settings (
    id INT PRIMARY KEY AUTO_INCREMENT,
    key VARCHAR(100) NOT NULL UNIQUE,
    value TEXT,
    description VARCHAR(255),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 错误日志表
CREATE TABLE t_error_logs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    gemini_key VARCHAR(100),
    model_name VARCHAR(100),
    error_type VARCHAR(50),
    error_log TEXT,
    error_code INT,
    request_msg JSON,
    request_time DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 请求日志表
CREATE TABLE t_request_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    request_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    model_name VARCHAR(100),
    api_key VARCHAR(100),
    is_success BOOLEAN NOT NULL,
    status_code INT,
    latency_ms INT
);
```

## 核心功能实现详解

### 1. API Key管理系统

**核心逻辑 (KeyManager)**

```typescript
// lib/services/keyManager.ts
export class KeyManager {
  private apiKeys: string[];
  private keyFailureCounts: Map<string, number>;
  private keyIndex: number = 0;
  private maxFailures: number;

  constructor(apiKeys: string[], maxFailures: number = 3) {
    this.apiKeys = apiKeys;
    this.keyFailureCounts = new Map();
    this.maxFailures = maxFailures;
    
    // 初始化失败计数
    apiKeys.forEach(key => this.keyFailureCounts.set(key, 0));
  }

  // 获取下一个可用的API Key
  getNextWorkingKey(): string {
    const startIndex = this.keyIndex;
    
    do {
      const currentKey = this.apiKeys[this.keyIndex];
      const failureCount = this.keyFailureCounts.get(currentKey) || 0;
      
      if (failureCount < this.maxFailures) {
        this.keyIndex = (this.keyIndex + 1) % this.apiKeys.length;
        return currentKey;
      }
      
      this.keyIndex = (this.keyIndex + 1) % this.apiKeys.length;
    } while (this.keyIndex !== startIndex);
    
    // 如果所有key都失效，返回第一个
    return this.apiKeys[0];
  }

  // 处理API调用失败
  handleApiFailure(apiKey: string): void {
    const currentCount = this.keyFailureCounts.get(apiKey) || 0;
    this.keyFailureCounts.set(apiKey, currentCount + 1);
  }

  // 重置失败计数
  resetFailureCount(apiKey: string): void {
    this.keyFailureCounts.set(apiKey, 0);
  }
}
```
##
# 2. HTTP客户端封装

**Gemini API客户端**

```typescript
// lib/clients/geminiClient.ts
export class GeminiApiClient {
  private baseUrl: string;
  private timeout: number;

  constructor(baseUrl: string, timeout: number = 300000) {
    this.baseUrl = baseUrl;
    this.timeout = timeout;
  }

  async generateContent(
    payload: any,
    model: string,
    apiKey: string
  ): Promise<any> {
    const url = `${this.baseUrl}/models/${model}:generateContent?key=${apiKey}`;
    
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(payload),
      signal: AbortSignal.timeout(this.timeout),
    });

    if (!response.ok) {
      throw new Error(`API call failed: ${response.status} ${await response.text()}`);
    }

    return response.json();
  }

  async *streamGenerateContent(
    payload: any,
    model: string,
    apiKey: string
  ): AsyncGenerator<string> {
    const url = `${this.baseUrl}/models/${model}:streamGenerateContent?alt=sse&key=${apiKey}`;
    
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(payload),
      signal: AbortSignal.timeout(this.timeout),
    });

    if (!response.ok) {
      throw new Error(`Stream API call failed: ${response.status}`);
    }

    const reader = response.body?.getReader();
    const decoder = new TextDecoder();

    if (!reader) throw new Error('No response body');

    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const chunk = decoder.decode(value);
        const lines = chunk.split('\n');
        
        for (const line of lines) {
          if (line.startsWith('data:')) {
            yield line;
          }
        }
      }
    } finally {
      reader.releaseLock();
    }
  }
}
```

### 3. OpenAI兼容层实现

**消息格式转换**

```typescript
// lib/converters/messageConverter.ts
export class OpenAIMessageConverter {
  convert(messages: any[]): { contents: any[], systemInstruction?: any } {
    const contents: any[] = [];
    let systemInstruction: any = null;

    for (const message of messages) {
      if (message.role === 'system') {
        systemInstruction = {
          role: 'system',
          parts: [{ text: message.content }]
        };
        continue;
      }

      const content: any = {
        role: message.role === 'assistant' ? 'model' : 'user',
        parts: []
      };

      if (typeof message.content === 'string') {
        content.parts.push({ text: message.content });
      } else if (Array.isArray(message.content)) {
        for (const part of message.content) {
          if (part.type === 'text') {
            content.parts.push({ text: part.text });
          } else if (part.type === 'image_url') {
            // 处理图片URL
            content.parts.push({
              inline_data: {
                mime_type: 'image/jpeg',
                data: part.image_url.url.split(',')[1] // base64数据
              }
            });
          }
        }
      }

      contents.push(content);
    }

    return { contents, systemInstruction };
  }
}
```

**响应格式转换**

```typescript
// lib/converters/responseConverter.ts
export class OpenAIResponseHandler {
  handleResponse(
    geminiResponse: any,
    model: string,
    stream: boolean = false,
    finishReason: string = 'stop'
  ): any {
    if (stream) {
      return this.handleStreamResponse(geminiResponse, model, finishReason);
    }
    
    const candidate = geminiResponse.candidates?.[0];
    const content = candidate?.content?.parts?.[0]?.text || '';
    
    return {
      id: `chatcmpl-${Date.now()}`,
      object: 'chat.completion',
      created: Math.floor(Date.now() / 1000),
      model,
      choices: [{
        index: 0,
        message: {
          role: 'assistant',
          content
        },
        finish_reason: finishReason
      }],
      usage: {
        prompt_tokens: geminiResponse.usageMetadata?.promptTokenCount || 0,
        completion_tokens: geminiResponse.usageMetadata?.candidatesTokenCount || 0,
        total_tokens: geminiResponse.usageMetadata?.totalTokenCount || 0
      }
    };
  }

  private handleStreamResponse(
    geminiChunk: any,
    model: string,
    finishReason?: string
  ): any {
    const candidate = geminiChunk.candidates?.[0];
    const content = candidate?.content?.parts?.[0]?.text || '';
    
    return {
      id: `chatcmpl-${Date.now()}`,
      object: 'chat.completion.chunk',
      created: Math.floor(Date.now() / 1000),
      model,
      choices: [{
        index: 0,
        delta: {
          content: content || null
        },
        finish_reason: finishReason || null
      }]
    };
  }
}
```#
## 4. Next.js API路由实现

**OpenAI兼容聊天接口**

```typescript
// app/api/v1/chat/completions/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { OpenAIChatService } from '@/lib/services/openaiChatService';
import { verifyAuthorization } from '@/lib/auth/security';

export async function POST(request: NextRequest) {
  try {
    // 验证授权
    const authResult = await verifyAuthorization(request);
    if (!authResult.success) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }

    const body = await request.json();
    const chatService = new OpenAIChatService();
    
    if (body.stream) {
      // 流式响应
      const stream = await chatService.createStreamCompletion(body);
      
      return new Response(
        new ReadableStream({
          async start(controller) {
            try {
              for await (const chunk of stream) {
                controller.enqueue(new TextEncoder().encode(chunk));
              }
            } catch (error) {
              controller.error(error);
            } finally {
              controller.close();
            }
          }
        }),
        {
          headers: {
            'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
          },
        }
      );
    } else {
      // 普通响应
      const response = await chatService.createCompletion(body);
      return NextResponse.json(response);
    }
  } catch (error) {
    console.error('Chat completion error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

**Gemini原生接口**

```typescript
// app/api/gemini/v1beta/models/[model]/generateContent/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { GeminiChatService } from '@/lib/services/geminiChatService';

export async function POST(
  request: NextRequest,
  { params }: { params: { model: string } }
) {
  try {
    const body = await request.json();
    const chatService = new GeminiChatService();
    
    const response = await chatService.generateContent(
      params.model,
      body,
      request.headers.get('authorization')?.replace('Bearer ', '') || ''
    );
    
    return NextResponse.json(response);
  } catch (error) {
    console.error('Gemini generate content error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### 5. 配置管理系统

**动态配置**

```typescript
// lib/config/settings.ts
export interface AppSettings {
  apiKeys: string[];
  allowedTokens: string[];
  baseUrl: string;
  maxFailures: number;
  maxRetries: number;
  timeout: number;
  searchModels: string[];
  imageModels: string[];
  filteredModels: string[];
  safetySettings: any[];
}

export class SettingsManager {
  private static instance: SettingsManager;
  private settings: AppSettings;

  private constructor() {
    this.settings = this.loadFromEnv();
  }

  static getInstance(): SettingsManager {
    if (!SettingsManager.instance) {
      SettingsManager.instance = new SettingsManager();
    }
    return SettingsManager.instance;
  }

  private loadFromEnv(): AppSettings {
    return {
      apiKeys: process.env.API_KEYS?.split(',') || [],
      allowedTokens: process.env.ALLOWED_TOKENS?.split(',') || [],
      baseUrl: process.env.BASE_URL || 'https://generativelanguage.googleapis.com/v1beta',
      maxFailures: parseInt(process.env.MAX_FAILURES || '3'),
      maxRetries: parseInt(process.env.MAX_RETRIES || '3'),
      timeout: parseInt(process.env.TIME_OUT || '300000'),
      searchModels: process.env.SEARCH_MODELS?.split(',') || [],
      imageModels: process.env.IMAGE_MODELS?.split(',') || [],
      filteredModels: process.env.FILTERED_MODELS?.split(',') || [],
      safetySettings: JSON.parse(process.env.SAFETY_SETTINGS || '[]'),
    };
  }

  getSettings(): AppSettings {
    return this.settings;
  }

  async updateSettings(newSettings: Partial<AppSettings>): Promise<void> {
    this.settings = { ...this.settings, ...newSettings };
    // 保存到数据库
    await this.saveToDatabase();
  }

  private async saveToDatabase(): Promise<void> {
    // 实现数据库保存逻辑
  }
}
```

### 6. 数据库操作层

**使用Prisma或Drizzle ORM**

```typescript
// lib/database/operations.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export async function addErrorLog(data: {
  geminiKey: string;
  modelName: string;
  errorType: string;
  errorLog: string;
  errorCode: number;
  requestMsg: any;
}): Promise<void> {
  await prisma.errorLog.create({
    data: {
      geminiKey: data.geminiKey,
      modelName: data.modelName,
      errorType: data.errorType,
      errorLog: data.errorLog,
      errorCode: data.errorCode,
      requestMsg: data.requestMsg,
      requestTime: new Date(),
    },
  });
}

export async function addRequestLog(data: {
  modelName: string;
  apiKey: string;
  isSuccess: boolean;
  statusCode: number;
  latencyMs: number;
}): Promise<void> {
  await prisma.requestLog.create({
    data: {
      modelName: data.modelName,
      apiKey: data.apiKey,
      isSuccess: data.isSuccess,
      statusCode: data.statusCode,
      latencyMs: data.latencyMs,
      requestTime: new Date(),
    },
  });
}
```###
 7. 管理面板实现

**React组件示例**

```tsx
// app/dashboard/keys/page.tsx
'use client';

import { useState, useEffect } from 'react';

interface KeyStatus {
  key: string;
  failureCount: number;
  isValid: boolean;
}

export default function KeysManagement() {
  const [keys, setKeys] = useState<KeyStatus[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchKeysStatus();
  }, []);

  const fetchKeysStatus = async () => {
    try {
      const response = await fetch('/api/admin/keys/status');
      const data = await response.json();
      setKeys(data.keys);
    } catch (error) {
      console.error('Failed to fetch keys status:', error);
    } finally {
      setLoading(false);
    }
  };

  const resetKeyFailureCount = async (key: string) => {
    try {
      await fetch(`/api/admin/keys/${key}/reset`, {
        method: 'POST',
      });
      fetchKeysStatus(); // 刷新状态
    } catch (error) {
      console.error('Failed to reset key:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold mb-6">API Keys Management</h1>
      
      <div className="grid gap-4">
        {keys.map((keyStatus) => (
          <div
            key={keyStatus.key}
            className={`p-4 border rounded-lg ${
              keyStatus.isValid ? 'border-green-200 bg-green-50' : 'border-red-200 bg-red-50'
            }`}
          >
            <div className="flex justify-between items-center">
              <div>
                <span className="font-mono text-sm">
                  {keyStatus.key.substring(0, 8)}...{keyStatus.key.slice(-4)}
                </span>
                <span className={`ml-2 px-2 py-1 rounded text-xs ${
                  keyStatus.isValid ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'
                }`}>
                  {keyStatus.isValid ? 'Valid' : 'Invalid'}
                </span>
              </div>
              
              <div className="flex items-center gap-4">
                <span className="text-sm text-gray-600">
                  Failures: {keyStatus.failureCount}
                </span>
                <button
                  onClick={() => resetKeyFailureCount(keyStatus.key)}
                  className="px-3 py-1 bg-blue-500 text-white rounded text-sm hover:bg-blue-600"
                >
                  Reset
                </button>
              </div>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## 网络协议和数据格式

### 1. OpenAI兼容接口

**请求格式**
```json
POST /v1/chat/completions
Content-Type: application/json
Authorization: Bearer your-token

{
  "model": "gemini-1.5-pro",
  "messages": [
    {
      "role": "user",
      "content": "Hello, how are you?"
    }
  ],
  "stream": false,
  "temperature": 0.7,
  "max_tokens": 1000
}
```

**响应格式**
```json
{
  "id": "chatcmpl-1234567890",
  "object": "chat.completion",
  "created": 1677652288,
  "model": "gemini-1.5-pro",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! I'm doing well, thank you for asking."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 12,
    "total_tokens": 22
  }
}
```

### 2. Gemini原生接口

**请求格式**
```json
POST /gemini/v1beta/models/gemini-1.5-pro:generateContent?key=your-api-key
Content-Type: application/json

{
  "contents": [
    {
      "role": "user",
      "parts": [
        {
          "text": "Hello, how are you?"
        }
      ]
    }
  ],
  "generationConfig": {
    "temperature": 0.7,
    "maxOutputTokens": 1000
  }
}
```

**响应格式**
```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "text": "Hello! I'm doing well, thank you for asking."
          }
        ],
        "role": "model"
      },
      "finishReason": "STOP"
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 10,
    "candidatesTokenCount": 12,
    "totalTokenCount": 22
  }
}
```

### 3. 流式响应格式

**Server-Sent Events (SSE)**
```
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gemini-1.5-pro","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gemini-1.5-pro","choices":[{"index":0,"delta":{"content":"!"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gemini-1.5-pro","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```