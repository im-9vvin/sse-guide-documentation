---
**[← 이전: Part 4 - AI 챗봇 서비스의 SSE 구현](./part4-ai-chatbot-sse.md)** | **[목차](./README.md)** | **[다음: Part 6 - 브라우저에서의 불완전한 데이터 처리 →](./part6-browser-data-processing.md)**

---

# Part 5: Next.js와 Vercel AI SDK 활용

## 6장. Next.js와 Vercel AI SDK 활용

Vercel AI SDK는 Next.js에서 AI 스트리밍을 구현하는 가장 효율적인 방법을 제공합니다. 이 장에서는 최신 버전의 AI SDK를 활용한 프로덕션 레벨 구현을 다룹니다.

### 6.1 Vercel AI SDK 최신 버전 활용법

#### 6.1.1 설치 및 기본 설정

```bash
npm install ai openai
# 또는
npm install ai @anthropic-ai/sdk
```

```typescript
// app/api/chat/route.ts
import { OpenAI } from 'openai';
import { OpenAIStream, StreamingTextResponse } from 'ai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export const runtime = 'edge';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const response = await openai.chat.completions.create({
    model: 'gpt-4-turbo-preview',
    stream: true,
    messages,
  });

  const stream = OpenAIStream(response);
  return new StreamingTextResponse(stream);
}
```

#### 6.1.2 클라이언트 측 구현

```typescript
// app/chat/page.tsx
'use client';

import { useChat } from 'ai/react';

export default function Chat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat();

  return (
    <div className="flex flex-col h-screen">
      <div className="flex-1 overflow-y-auto p-4">
        {messages.map((message) => (
          <div
            key={message.id}
            className={`mb-4 ${
              message.role === 'user' ? 'text-right' : 'text-left'
            }`}
          >
            <div
              className={`inline-block p-3 rounded-lg ${
                message.role === 'user'
                  ? 'bg-blue-500 text-white'
                  : 'bg-gray-200 text-black'
              }`}
            >
              {message.content}
            </div>
          </div>
        ))}
      </div>
      
      <form onSubmit={handleSubmit} className="p-4 border-t">
        <div className="flex gap-2">
          <input
            type="text"
            value={input}
            onChange={handleInputChange}
            placeholder="메시지를 입력하세요..."
            className="flex-1 p-2 border rounded"
            disabled={isLoading}
          />
          <button
            type="submit"
            disabled={isLoading}
            className="px-4 py-2 bg-blue-500 text-white rounded disabled:opacity-50"
          >
            전송
          </button>
        </div>
      </form>
    </div>
  );
}
```

#### 6.1.3 고급 옵션과 커스터마이징

```typescript
// 고급 useChat 옵션
const {
  messages,
  input,
  handleInputChange,
  handleSubmit,
  isLoading,
  error,
  reload,
  stop,
  append,
  setMessages,
} = useChat({
  api: '/api/chat',
  id: 'unique-chat-id',
  initialMessages: [
    {
      id: '1',
      role: 'system',
      content: '당신은 도움이 되는 AI 어시스턴트입니다.',
    },
  ],
  body: {
    temperature: 0.7,
    max_tokens: 1000,
  },
  headers: {
    'x-custom-header': 'value',
  },
  onResponse: (response) => {
    console.log('응답 수신:', response);
  },
  onFinish: (message) => {
    console.log('메시지 완료:', message);
  },
  onError: (error) => {
    console.error('에러 발생:', error);
  },
});
```

### 6.2 App Router SSE 구현

#### 6.2.1 스트리밍 Route Handler

```typescript
// app/api/stream/route.ts
import { ReadableStream } from 'web-streams-polyfill';

export const runtime = 'edge';

function iteratorToStream(iterator: AsyncIterator<any>) {
  return new ReadableStream({
    async pull(controller) {
      const { value, done } = await iterator.next();
      
      if (done) {
        controller.close();
      } else {
        controller.enqueue(value);
      }
    },
  });
}

async function* generateData() {
  const phrases = ['안녕하세요', ' ', '저는', ' ', 'AI', ' ', '어시스턴트입니다', '.'];
  
  for (const phrase of phrases) {
    await new Promise(resolve => setTimeout(resolve, 100));
    yield new TextEncoder().encode(`data: ${JSON.stringify({ text: phrase })}\n\n`);
  }
}

export async function GET() {
  const iterator = generateData();
  const stream = iteratorToStream(iterator);
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

#### 6.2.2 Transform Stream 활용

```typescript
// app/api/transform-stream/route.ts
export async function POST(req: Request) {
  const { prompt } = await req.json();
  
  const encoder = new TextEncoder();
  const decoder = new TextDecoder();
  
  const transformStream = new TransformStream({
    async transform(chunk, controller) {
      // OpenAI 응답 변환
      const text = decoder.decode(chunk);
      const lines = text.split('\n');
      
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6);
          if (data === '[DONE]') {
            controller.enqueue(encoder.encode('event: done\ndata: {}\n\n'));
            return;
          }
          
          try {
            const parsed = JSON.parse(data);
            const content = parsed.choices[0]?.delta?.content || '';
            
            if (content) {
              controller.enqueue(
                encoder.encode(`data: ${JSON.stringify({ text: content })}\n\n`)
              );
            }
          } catch (e) {
            // 파싱 에러 무시
          }
        }
      }
    },
  });
  
  const openaiResponse = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
    },
    body: JSON.stringify({
      model: 'gpt-3.5-turbo',
      messages: [{ role: 'user', content: prompt }],
      stream: true,
    }),
  });
  
  const stream = openaiResponse.body!.pipeThrough(transformStream);
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
    },
  });
}
```

#### 6.2.3 Server Actions와 스트리밍

```typescript
// app/actions/stream-action.ts
'use server';

import { createStreamableValue } from 'ai/rsc';

export async function streamData(prompt: string) {
  const stream = createStreamableValue('');
  
  (async () => {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      },
      body: JSON.stringify({
        model: 'gpt-3.5-turbo',
        messages: [{ role: 'user', content: prompt }],
        stream: true,
      }),
    });
    
    const reader = response.body!.getReader();
    const decoder = new TextDecoder();
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');
      
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6);
          if (data === '[DONE]') {
            stream.done();
            return;
          }
          
          try {
            const parsed = JSON.parse(data);
            const content = parsed.choices[0]?.delta?.content || '';
            stream.update(content);
          } catch (e) {
            // 무시
          }
        }
      }
    }
  })();
  
  return { output: stream.value };
}

// 클라이언트 컴포넌트
'use client';

import { useState } from 'react';
import { readStreamableValue } from 'ai/rsc';
import { streamData } from './actions/stream-action';

export default function StreamingServerAction() {
  const [output, setOutput] = useState('');
  
  async function handleSubmit(prompt: string) {
    const { output } = await streamData(prompt);
    
    for await (const delta of readStreamableValue(output)) {
      setOutput(current => current + delta);
    }
  }
  
  return (
    <div>
      <button onClick={() => handleSubmit('안녕하세요!')}>
        스트림 시작
      </button>
      <div>{output}</div>
    </div>
  );
}
```

### 6.3 Pages Router SSE 구현

#### 6.3.1 API Route 구현

```typescript
// pages/api/sse.ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache, no-transform',
    'Connection': 'keep-alive',
  });
  
  const sendEvent = (data: any) => {
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };
  
  // 초기 연결 확인
  sendEvent({ type: 'connected', timestamp: Date.now() });
  
  // 주기적 업데이트
  const interval = setInterval(() => {
    sendEvent({
      type: 'update',
      value: Math.random(),
      timestamp: Date.now(),
    });
  }, 1000);
  
  // 연결 종료 처리
  req.on('close', () => {
    clearInterval(interval);
    res.end();
  });
  
  // Keep-alive
  const keepAlive = setInterval(() => {
    res.write(': keep-alive\n\n');
  }, 30000);
  
  req.on('close', () => {
    clearInterval(keepAlive);
  });
}
```

#### 6.3.2 클라이언트 훅 구현

```typescript
// hooks/useSSE.ts
import { useEffect, useState, useCallback, useRef } from 'react';

interface UseSSEOptions {
  onMessage?: (data: any) => void;
  onError?: (error: Event) => void;
  onOpen?: () => void;
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
}

export function useSSE(url: string, options: UseSSEOptions = {}) {
  const [data, setData] = useState<any>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const eventSourceRef = useRef<EventSource | null>(null);
  const reconnectAttemptsRef = useRef(0);
  
  const connect = useCallback(() => {
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
    }
    
    const eventSource = new EventSource(url);
    eventSourceRef.current = eventSource;
    
    eventSource.onopen = () => {
      setIsConnected(true);
      setError(null);
      reconnectAttemptsRef.current = 0;
      options.onOpen?.();
    };
    
    eventSource.onmessage = (event) => {
      try {
        const parsedData = JSON.parse(event.data);
        setData(parsedData);
        options.onMessage?.(parsedData);
      } catch (e) {
        console.error('Failed to parse SSE data:', e);
      }
    };
    
    eventSource.onerror = (event) => {
      setIsConnected(false);
      setError(new Error('SSE connection error'));
      options.onError?.(event);
      
      // 재연결 로직
      if (
        reconnectAttemptsRef.current < (options.maxReconnectAttempts || 5)
      ) {
        reconnectAttemptsRef.current++;
        setTimeout(() => {
          connect();
        }, options.reconnectInterval || 1000);
      }
    };
  }, [url, options]);
  
  const disconnect = useCallback(() => {
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
      eventSourceRef.current = null;
      setIsConnected(false);
    }
  }, []);
  
  useEffect(() => {
    connect();
    return () => disconnect();
  }, [connect, disconnect]);
  
  return { data, error, isConnected, reconnect: connect, disconnect };
}
```

### 6.4 TypeScript 타입 정의

#### 6.4.1 메시지 타입 정의

```typescript
// types/chat.ts
export interface Message {
  id: string;
  role: 'system' | 'user' | 'assistant' | 'function';
  content: string;
  name?: string;
  function_call?: {
    name: string;
    arguments: string;
  };
  createdAt?: Date;
}

export interface ChatCompletionChunk {
  id: string;
  object: string;
  created: number;
  model: string;
  choices: Array<{
    index: number;
    delta: {
      role?: string;
      content?: string;
      function_call?: {
        name?: string;
        arguments?: string;
      };
    };
    finish_reason: string | null;
  }>;
}

export interface StreamingResponse {
  text: string;
  isComplete: boolean;
  error?: Error;
}
```

#### 6.4.2 이벤트 타입 정의

```typescript
// types/events.ts
export enum SSEEventType {
  Message = 'message',
  Error = 'error',
  Done = 'done',
  Ping = 'ping',
  Metadata = 'metadata',
}

export interface SSEEvent<T = any> {
  type: SSEEventType;
  data: T;
  id?: string;
  retry?: number;
}

export interface StreamMetadata {
  model: string;
  usage?: {
    prompt_tokens: number;
    completion_tokens: number;
    total_tokens: number;
  };
  finish_reason?: string;
  duration?: number;
}
```

#### 6.4.3 Provider 타입 정의

```typescript
// types/providers.ts
export interface AIProvider {
  name: string;
  streamChat(
    messages: Message[],
    options?: StreamOptions
  ): AsyncIterable<string>;
}

export interface StreamOptions {
  model?: string;
  temperature?: number;
  maxTokens?: number;
  topP?: number;
  frequencyPenalty?: number;
  presencePenalty?: number;
  stop?: string[];
  user?: string;
  functions?: Array<{
    name: string;
    description: string;
    parameters: any;
  }>;
  functionCall?: 'none' | 'auto' | { name: string };
}

export class OpenAIProvider implements AIProvider {
  name = 'openai';
  
  async *streamChat(
    messages: Message[],
    options?: StreamOptions
  ): AsyncIterable<string> {
    // 구현
  }
}
```

### 6.5 AI SDK의 Data Stream Protocol

#### 6.5.1 스트림 프로토콜 이해

```typescript
// Vercel AI SDK의 내부 프로토콜
interface DataStreamProtocol {
  // 텍스트 청크
  '0': string;
  
  // 함수 호출
  '1': {
    function_call: {
      name: string;
      arguments: string;
    };
  };
  
  // 에러
  '3': {
    error: string;
  };
  
  // 메타데이터
  '8': {
    model?: string;
    usage?: {
      prompt_tokens: number;
      completion_tokens: number;
    };
  };
  
  // 완료
  'd': undefined;
}
```

#### 6.5.2 커스텀 스트림 프로토콜 구현

```typescript
// lib/custom-stream-protocol.ts
export class CustomStreamProtocol {
  private encoder = new TextEncoder();
  
  // 타입별 인코딩
  encodeText(text: string): Uint8Array {
    return this.encoder.encode(`0:"${text}"\n`);
  }
  
  encodeFunction(name: string, args: string): Uint8Array {
    const data = JSON.stringify({ function_call: { name, arguments: args } });
    return this.encoder.encode(`1:${data}\n`);
  }
  
  encodeError(error: string): Uint8Array {
    return this.encoder.encode(`3:${JSON.stringify({ error })}\n`);
  }
  
  encodeMetadata(metadata: any): Uint8Array {
    return this.encoder.encode(`8:${JSON.stringify(metadata)}\n`);
  }
  
  encodeDone(): Uint8Array {
    return this.encoder.encode(`d\n`);
  }
  
  // 디코딩
  decode(line: string): { type: string; data: any } | null {
    const match = line.match(/^([0-9a-z]):(.*)$/);
    if (!match) return null;
    
    const [, type, dataStr] = match;
    
    switch (type) {
      case '0':
        return { type: 'text', data: JSON.parse(dataStr) };
      case '1':
        return { type: 'function', data: JSON.parse(dataStr) };
      case '3':
        return { type: 'error', data: JSON.parse(dataStr) };
      case '8':
        return { type: 'metadata', data: JSON.parse(dataStr) };
      case 'd':
        return { type: 'done', data: null };
      default:
        return null;
    }
  }
}
```

#### 6.5.3 통합 스트림 핸들러

```typescript
// lib/unified-stream-handler.ts
import { OpenAIStream, AnthropicStream, StreamingTextResponse } from 'ai';

export type AIProvider = 'openai' | 'anthropic' | 'custom';

export async function createAIStream(
  provider: AIProvider,
  response: Response | ReadableStream,
  callbacks?: {
    onStart?: () => void;
    onToken?: (token: string) => void;
    onCompletion?: (completion: string) => void;
    onFinal?: (message: Message) => void;
  }
) {
  let stream: ReadableStream;
  
  switch (provider) {
    case 'openai':
      stream = OpenAIStream(response as Response, {
        onStart: callbacks?.onStart,
        onToken: callbacks?.onToken,
        onCompletion: callbacks?.onCompletion,
        onFinal: callbacks?.onFinal,
      });
      break;
      
    case 'anthropic':
      stream = AnthropicStream(response as Response, {
        onStart: callbacks?.onStart,
        onToken: callbacks?.onToken,
        onCompletion: callbacks?.onCompletion,
        onFinal: callbacks?.onFinal,
      });
      break;
      
    case 'custom':
      stream = createCustomStream(response as ReadableStream, callbacks);
      break;
      
    default:
      throw new Error(`Unsupported provider: ${provider}`);
  }
  
  return new StreamingTextResponse(stream);
}

function createCustomStream(
  input: ReadableStream,
  callbacks?: any
): ReadableStream {
  const reader = input.getReader();
  const decoder = new TextDecoder();
  let buffer = '';
  
  return new ReadableStream({
    async start(controller) {
      callbacks?.onStart?.();
    },
    
    async pull(controller) {
      const { done, value } = await reader.read();
      
      if (done) {
        callbacks?.onCompletion?.(buffer);
        controller.close();
        return;
      }
      
      const chunk = decoder.decode(value, { stream: true });
      buffer += chunk;
      
      callbacks?.onToken?.(chunk);
      controller.enqueue(new TextEncoder().encode(chunk));
    },
  });
}
```

---

**다음 장 예고**: [Part 6: 브라우저에서의 불완전한 데이터 처리](./part6-browser-data-processing.md)에서는 SSE 스트리밍 중 발생할 수 있는 청크 분할 문제와 안전한 파싱 방법을 다루겠습니다.

---
**[← 이전: Part 4 - AI 챗봇 서비스의 SSE 구현](./part4-ai-chatbot-sse.md)** | **[목차](./README.md)** | **[다음: Part 6 - 브라우저에서의 불완전한 데이터 처리 →](./part6-browser-data-processing.md)**

---