# Next.js와 Vercel AI SDK를 활용한 OpenAI ChatGPT SSE 스트리밍 구현 완전 가이드

## SSE W3C 표준 스펙과 OpenAI 구현의 이해

Server-Sent Events(SSE)는 WHATWG HTML Living Standard에서 정의하는 서버 푸시 기술로, HTTP를 통해 서버에서 클라이언트로 실시간 데이터를 전송할 수 있게 해줍니다. W3C 표준은 네 가지 핵심 필드를 정의합니다:

**`data` 필드**: 실제 메시지 데이터를 담는 필드입니다. 여러 연속된 `data:` 라인은 개행 문자로 연결됩니다.

```
data: 첫 번째 줄
data: 두 번째 줄
```

**`event` 필드**: 이벤트 타입 이름을 지정합니다. 지정하지 않으면 기본값은 "message"입니다.

```
event: userconnect
data: {"username": "bobby", "time": "02:33:48"}
```

**`id` 필드**: 재연결 시 사용되는 이벤트 ID를 설정합니다. 연결이 끊어지면 `Last-Event-ID` 헤더로 전송됩니다.

```
id: 123
data: Important message
```

**`retry` 필드**: 재연결 시간을 밀리초 단위로 설정합니다.

```
retry: 5000
data: 5초 후 재연결 시도
```

### OpenAI API의 SSE 구현 방식

OpenAI는 W3C 표준의 단순화된 버전을 사용합니다. **data-only** 형식으로 `event`, `id`, `retry` 필드를 사용하지 않습니다:

```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{"content":"Hello"}}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"finish_reason":"stop"}]}

data: [DONE]
```

OpenAI의 delta 객체는 증분 변경사항을 담습니다:

- **`role`**: 첫 번째 청크에만 포함 ("assistant")
- **`content`**: 스트리밍되는 텍스트 토큰
- **`finish_reason`**: 완료 이유 (stop, length, function_call 등)

## 빅테크 기업들의 비공식 SSE 이벤트 패턴

실제 프로덕션 환경에서는 표준 필드 외에도 다양한 커스텀 이벤트 타입들이 사용됩니다:

### 연결 유지 패턴

**ping/heartbeat**: 연결 상태 확인을 위한 가장 일반적인 패턴

```
event: ping
data: {"timestamp": 1704067200}

event: heartbeat
data: {"type": "heartbeat", "interval": 30000}

: heartbeat (주석 형태)
```

**keep-alive**: Twitter/X는 매 10-20초마다 `\r\n` 전송

```
: keep-alive
:\n\n
data: keep-alive\n\n
```

### 스트림 제어 패턴

**done/complete**: 스트림 종료 신호

```
event: done
data: {"status": "completed", "total_events": 150}

event: complete
data: [DONE]
```

**start/init**: 스트림 시작 및 초기화

```
event: start
data: {"stream_id": "abc123", "version": "2.0"}

event: init
data: {"protocol": "sse/1.0", "features": ["compression", "binary"]}
```

### 상태 및 메타데이터 패턴

**status/system**: 시스템 상태 알림

```
event: status
data: {"connection": "active", "queue_size": 0}

event: system
data: {"message": "Scheduled maintenance at 02:00 UTC"}
```

**progress/update**: 진행 상황 업데이트

```
event: progress
data: {"percentage": 75, "current": 750, "total": 1000}

event: update
data: {"type": "incremental", "delta": 5}
```

### 에러 처리 패턴

```
event: error
data: {
  "code": "RATE_LIMIT_EXCEEDED",
  "message": "Too many requests",
  "retry_after": 60
}

event: operational-disconnect
data: {
  "reason": "upstream_failure",
  "reconnect": true,
  "backoff": 5000
}
```

이러한 패턴들은 공식 표준은 아니지만, 대규모 실시간 시스템에서 연결 관리, 디버깅, 모니터링을 위해 필수적으로 사용됩니다.

## Vercel AI SDK 최신 버전 활용법

### 핵심 패키지 설치와 설정

```bash
npm install ai @ai-sdk/openai
```

Vercel AI SDK 4.3.16 버전과 @ai-sdk/openai 1.3.22는 OpenAI 스트리밍을 위한 포괄적인 솔루션을 제공합니다:

```typescript
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

// OpenAI 프로바이더 설정
const openai = createOpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  compatibility: 'strict', // OpenAI API 엄격 모드
  organization: 'your-org', // 선택사항
  project: 'your-project', // 선택사항
});

// 스트리밍 텍스트 생성
const result = streamText({
  model: openai('gpt-4o'),
  system: '당신은 도움이 되는 어시스턴트입니다.',
  messages,
  maxTokens: 1000,
  temperature: 0.7,
  onFinish: (result) => {
    // 완료 콜백
  },
  onError: (error) => {
    // 에러 처리
  },
});

return result.toDataStreamResponse();
```

### AI SDK의 Data Stream Protocol

Vercel AI SDK는 풍부한 메타데이터를 지원하는 커스텀 프로토콜을 사용합니다:

- **형식**: `TYPE_ID:CONTENT_JSON\n`
- **헤더**: `x-vercel-ai-data-stream: v1`
- **지원 타입**: 텍스트, 도구 호출, 도구 결과, 추론 단계, 에러 메시지

## Next.js App Router와 Pages Router 구현

### App Router SSE 구현 (권장)

```typescript
// app/api/stream/route.ts
export const dynamic = 'force-dynamic';

interface SSEEvent {
  event?: string;
  data: any;
  id?: string;
  retry?: number;
}

function formatSSE(event: SSEEvent): string {
  let formatted = '';

  if (event.event) formatted += `event: ${event.event}\n`;
  if (event.id) formatted += `id: ${event.id}\n`;
  if (event.retry) formatted += `retry: ${event.retry}\n`;

  const dataString = typeof event.data === 'string' ? event.data : JSON.stringify(event.data);

  formatted += `data: ${dataString}\n\n`;

  return formatted;
}

export async function POST(request: Request) {
  const { messages } = await request.json();
  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    async start(controller) {
      try {
        // 연결 시작 이벤트
        controller.enqueue(
          encoder.encode(
            formatSSE({
              event: 'start',
              data: { stream_id: crypto.randomUUID(), timestamp: Date.now() },
            })
          )
        );

        // Heartbeat 인터벌 설정 (30초마다)
        const heartbeatInterval = setInterval(() => {
          controller.enqueue(
            encoder.encode(
              formatSSE({
                event: 'heartbeat',
                data: { timestamp: Date.now(), type: 'heartbeat' },
              })
            )
          );
        }, 30000);

        const result = await streamText({
          model: openai('gpt-4o'),
          messages,
        });

        let totalTokens = 0;

        for await (const textPart of result.textStream) {
          totalTokens++;

          // 텍스트 스트리밍
          controller.enqueue(
            encoder.encode(
              formatSSE({
                data: { content: textPart },
                id: Date.now().toString(),
              })
            )
          );

          // 진행 상황 업데이트 (10토큰마다)
          if (totalTokens % 10 === 0) {
            controller.enqueue(
              encoder.encode(
                formatSSE({
                  event: 'progress',
                  data: { tokens: totalTokens },
                })
              )
            );
          }
        }

        // 완료 이벤트
        controller.enqueue(
          encoder.encode(
            formatSSE({
              event: 'done',
              data: { status: 'completed', total_tokens: totalTokens },
            })
          )
        );

        // OpenAI 스타일 [DONE] 메시지
        controller.enqueue(encoder.encode('data: [DONE]\n\n'));

        clearInterval(heartbeatInterval);
        controller.close();
      } catch (error) {
        // 에러 이벤트
        controller.enqueue(
          encoder.encode(
            formatSSE({
              event: 'error',
              data: {
                code: error.code || 'UNKNOWN_ERROR',
                message: error.message,
                retry_after: 60,
              },
            })
          )
        );
        controller.close();
      }
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache, no-transform',
      'Connection': 'keep-alive',
      'Access-Control-Allow-Origin': '*',
      'X-Accel-Buffering': 'no', // Nginx 버퍼링 비활성화
    },
  });
}
```

### Pages Router SSE 구현

```typescript
// pages/api/stream.ts
import type { NextApiRequest, NextApiResponse } from 'next';

interface SSEResponse extends NextApiResponse {
  writeSSE: (event: string, data: any, id?: string) => void;
}

function addSSEHelpers(res: NextApiResponse): SSEResponse {
  const sseRes = res as SSEResponse;

  sseRes.writeSSE = (event: string, data: any, id?: string) => {
    if (id) res.write(`id: ${id}\n`);
    res.write(`event: ${event}\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  return sseRes;
}

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache, no-transform',
    'Connection': 'keep-alive',
  });

  const sseRes = addSSEHelpers(res);

  try {
    const result = await streamText({
      model: openai('gpt-4o'),
      messages: req.body.messages,
    });

    for await (const textPart of result.textStream) {
      sseRes.writeSSE('message', { content: textPart });
    }

    res.write('data: [DONE]\n\n');
    res.end();
  } catch (error) {
    sseRes.writeSSE('error', { message: error.message });
    res.end();
  }
}
```

## TypeScript 타입 정의

### 핵심 SSE 타입

```typescript
// Server-Sent Event 인터페이스
interface ServerSentEvent {
  event?: string;
  data: any;
  id?: string;
  retry?: number;
}

// EventSource 준비 상태
enum EventSourceReadyState {
  CONNECTING = 0,
  OPEN = 1,
  CLOSED = 2,
}

// 커스텀 이벤트 타입 맵
interface EventSourceEventMap {
  open: Event;
  message: MessageEvent;
  error: Event;
  userConnected: CustomEvent<{ userId: string; username: string }>;
  heartbeat: CustomEvent<{ timestamp: number }>;
}

// 빅테크 스타일 커스텀 이벤트 타입
type CustomEventType =
  | 'ping'
  | 'heartbeat'
  | 'keep-alive'
  | 'done'
  | 'complete'
  | 'start'
  | 'init'
  | 'status'
  | 'system'
  | 'progress'
  | 'update'
  | 'error'
  | 'operational-disconnect';

// 커스텀 이벤트 페이로드 타입
interface CustomEventPayloads {
  'ping': { timestamp: number };
  'heartbeat': { type: 'heartbeat'; interval?: number };
  'keep-alive': string | null;
  'done': { status: string; total?: number };
  'complete': { message?: string } | '[DONE]';
  'start': { stream_id: string; version?: string };
  'init': { protocol: string; features?: string[] };
  'status': { connection: string; queue_size?: number };
  'system': { message: string; severity?: 'info' | 'warning' | 'error' };
  'progress': { percentage: number; current: number; total: number };
  'update': { type: string; delta: any };
  'error': { code: string; message: string; retry_after?: number };
  'operational-disconnect': { reason: string; reconnect: boolean; backoff: number };
}

// SSE 연결 설정
interface SSEConfig {
  url: string;
  withCredentials?: boolean;
  headers?: Record<string, string>;
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
}
```

### React 훅 타입

```typescript
// useSSE 훅 반환 타입
interface SSEHookReturn<T = any> {
  data: T | null;
  isConnected: boolean;
  error: string | null;
  close: () => void;
  reconnect: () => void;
}

// useChat 훅 사용 예시
import { useChat } from '@ai-sdk/react';

const { messages, input, handleSubmit, handleInputChange, status, append, reload, stop } =
  useChat({
    api: '/api/chat',
    maxSteps: 5,
    onToolCall: async (toolCall) => {},
    onFinish: (message) => {},
    onError: (error) => {},
  });
```

## 프로덕션 레벨 에러 처리와 재연결 로직

### 지수 백오프를 활용한 재연결 구현

```typescript
class SSEManager {
  constructor(url: string, options: SSEManagerOptions = {}) {
    this.url = url;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = options.maxReconnectAttempts || 10;
    this.initialReconnectDelay = options.initialReconnectDelay || 1000;
    this.maxReconnectDelay = options.maxReconnectDelay || 30000;
    this.reconnectDecay = options.reconnectDecay || 1.5;
    this.lastHeartbeat = Date.now();
    this.heartbeatTimeout = options.heartbeatTimeout || 45000;
  }

  connect() {
    this.eventSource = new EventSource(this.url);

    // 표준 이벤트 핸들러
    this.eventSource.onopen = () => {
      this.reconnectAttempts = 0;
      this.emit('connected');
    };

    this.eventSource.onmessage = (event) => {
      this.lastHeartbeat = Date.now();
      this.handleMessage(event);
    };

    // 커스텀 이벤트 핸들러
    this.eventSource.addEventListener('ping', (event) => {
      this.lastHeartbeat = Date.now();
      this.emit('ping', JSON.parse(event.data));
    });

    this.eventSource.addEventListener('heartbeat', (event) => {
      this.lastHeartbeat = Date.now();
      // heartbeat는 보통 로깅만 하고 UI에는 표시하지 않음
      console.debug('Heartbeat received:', event.data);
    });

    this.eventSource.addEventListener('done', (event) => {
      this.emit('streamComplete', JSON.parse(event.data));
      this.disconnect();
    });

    this.eventSource.addEventListener('error', (event: Event) => {
      const errorData = event instanceof MessageEvent ? JSON.parse(event.data) : null;
      this.handleStreamError(errorData);
    });

    // Heartbeat 모니터링
    this.startHeartbeatMonitor();
  }

  startHeartbeatMonitor() {
    this.heartbeatInterval = setInterval(() => {
      const timeSinceLastHeartbeat = Date.now() - this.lastHeartbeat;

      if (timeSinceLastHeartbeat > this.heartbeatTimeout) {
        console.warn('Heartbeat timeout detected, reconnecting...');
        this.handleConnectionError();
      }
    }, 10000); // 10초마다 체크
  }

  handleConnectionError() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('최대 재연결 시도 횟수 도달');
      this.emit('maxReconnectAttemptsReached');
      return;
    }

    const delay = Math.min(
      this.initialReconnectDelay * Math.pow(this.reconnectDecay, this.reconnectAttempts),
      this.maxReconnectDelay
    );

    // 썬더링 허드 방지를 위한 지터 추가
    const jitter = delay * 0.1 * Math.random();
    const actualDelay = delay + jitter;

    console.log(`${actualDelay}ms 후 재연결 시도 (시도 ${this.reconnectAttempts + 1})`);

    this.reconnectAttempts++;

    setTimeout(() => {
      this.disconnect();
      this.connect();
    }, actualDelay);
  }

  disconnect() {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
    }
    if (this.eventSource) {
      this.eventSource.close();
    }
  }
}
```

### OpenAI API 에러 처리

```typescript
const handleOpenAIError = (error: any) => {
  switch (error.type) {
    case 'insufficient_quota':
      return {
        retryable: false,
        message: 'API 할당량 초과',
        userMessage: '일시적으로 서비스를 사용할 수 없습니다',
      };
    case 'rate_limit_exceeded':
      return {
        retryable: true,
        message: '속도 제한 초과',
        retryAfter: error.retryAfter || 60,
      };
    case 'invalid_request_error':
      return {
        retryable: false,
        message: '잘못된 요청',
        userMessage: '요청을 다시 확인해주세요',
      };
    default:
      return {
        retryable: true,
        message: '알 수 없는 오류',
        retryAfter: 60,
      };
  }
};
```

## 실제 구현 예제와 베스트 프랙티스

### 완전한 스트리밍 채팅 컴포넌트

```typescript
// components/StreamingChat.tsx
import React, { useState, useEffect, useRef } from 'react';
import { useSSEConnection } from '../hooks/useSSEConnection';

const StreamingChat: React.FC<{ apiEndpoint: string }> = ({ apiEndpoint }) => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const sendMessage = async () => {
    if (!input.trim()) return;

    const userMessage = { role: 'user', content: input };
    setMessages((prev) => [...prev, userMessage]);
    setInput('');
    setIsLoading(true);

    try {
      const response = await fetch(apiEndpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
        },
        body: JSON.stringify({ message: input }),
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const reader = response.body!.getReader();
      let assistantMessage = { role: 'assistant', content: '' };
      setMessages((prev) => [...prev, assistantMessage]);

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = new TextDecoder().decode(value);
        const lines = chunk.split('\n');

        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = line.slice(6);
            if (data === '[DONE]') {
              setIsLoading(false);
              return;
            }

            try {
              const parsed = JSON.parse(data);
              if (parsed.content) {
                setMessages((prev) => {
                  const updated = [...prev];
                  updated[updated.length - 1].content += parsed.content;
                  return updated;
                });
              }
            } catch (parseError) {
              console.error('스트리밍 데이터 파싱 실패:', parseError);
            }
          }
        }
      }
    } catch (error) {
      console.error('스트리밍 오류:', error);
      setMessages((prev) => [
        ...prev,
        {
          role: 'system',
          content: '요청 처리 중 오류가 발생했습니다.',
        },
      ]);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="streaming-chat">
      <div className="messages-container">
        {messages.map((message, index) => (
          <div
            key={index}
            className={`message ${message.role}`}
          >
            <strong>{message.role}:</strong> {message.content}
          </div>
        ))}
        {isLoading && <div className="typing-indicator">AI가 입력 중...</div>}
        <div ref={messagesEndRef} />
      </div>

      <div className="input-container">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
          placeholder="메시지를 입력하세요..."
          disabled={isLoading}
        />
        <button
          onClick={sendMessage}
          disabled={isLoading || !input.trim()}
        >
          전송
        </button>
      </div>
    </div>
  );
};
```

### 보안 모범 사례

**API 키 보호**:

```typescript
// next.config.js
module.exports = {
  serverRuntimeConfig: {
    openaiApiKey: process.env.OPENAI_API_KEY, // 서버 사이드에서만 접근 가능
  },
};
```

**CORS 설정**:

```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith('/api/stream')) {
    const response = NextResponse.next();

    const allowedOrigins =
      process.env.NODE_ENV === 'production'
        ? ['https://yourdomain.com']
        : ['http://localhost:3000'];

    const origin = request.headers.get('origin');
    if (origin && allowedOrigins.includes(origin)) {
      response.headers.set('Access-Control-Allow-Origin', origin);
    }

    return response;
  }
}
```

### 성능 최적화

**연결 풀링**으로 동시 연결 수를 관리하고, **메모리 누수 방지**를 위해 컴포넌트 언마운트 시 반드시 연결을 정리합니다:

```typescript
useEffect(() => {
  return () => {
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
      eventSourceRef.current = null;
    }
  };
}, []);
```

## 결론

Next.js와 Vercel AI SDK를 활용한 OpenAI ChatGPT SSE 스트리밍 구현은 W3C 표준을 기반으로 하되, OpenAI의 단순화된 data-only 형식을 이해하고 활용하는 것이 핵심입니다. Vercel AI SDK는 이러한 복잡성을 추상화하여 개발자 친화적인 API를 제공하며, App Router의 ReadableStream API와 완벽하게 통합됩니다. 프로덕션 환경에서는 지수 백오프를 통한 재연결, 포괄적인 에러 처리, 그리고 적절한 보안 설정이 필수적입니다.
