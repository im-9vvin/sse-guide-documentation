# AI 챗봇 SSE 스트리밍 구현 완벽 가이드

## 목차

1. [SSE와 데이터 청킹의 이해](#1-sse와-데이터-청킹의-이해)
2. [프론트엔드에서의 불완전한 데이터 처리](#2-프론트엔드에서의-불완전한-데이터-처리)
3. [검증된 UX 패턴과 전략](#3-검증된-ux-패턴과-전략)
4. [Vercel AI SDK 활용법](#4-vercel-ai-sdk-활용법)
5. [커스텀 프로바이더 구현 가이드](#5-커스텀-프로바이더-구현-가이드)
6. [프로덕션 레벨 구현 예제](#6-프로덕션-레벨-구현-예제)

## 1. SSE와 데이터 청킹의 이해

### SSE의 두 독립적인 레이어

SSE는 완전히 독립적인 두 개의 프로토콜 레이어에서 작동합니다:

#### 전송 레이어 (HTTP 청킹)

- **역할**: 네트워크상에서 데이터를 어떻게 전송할지 결정
- **주체**: HTTP 프로토콜 (서버와 브라우저의 HTTP 파서)
- **목적**: 크기를 알 수 없는 데이터를 스트리밍으로 전송
- **경계 구분**: CRLF (`\r\n`)
- **데이터 단위**: 청크 (임의 크기)

#### 애플리케이션 레이어 (SSE 프로토콜)

- **역할**: 이벤트를 어떻게 구조화하고 파싱할지 결정
- **주체**: EventSource API
- **목적**: 서버에서 보낸 이벤트를 JavaScript 이벤트로 변환
- **경계 구분**: 빈 줄 (`\n\n`)
- **데이터 단위**: 이벤트 (논리적 단위)

### W3C 표준 SSE 필드

- **`data:`** - 실제 이벤트 데이터
- **`event:`** - 이벤트 타입 이름 (기본값: "message")
- **`id:`** - 재연결을 위한 이벤트 ID
- **`retry:`** - 재연결 시간 설정 (밀리초)
- **`:`** - 주석 (무시됨)

## 2. 프론트엔드에서의 불완전한 데이터 처리

### 브라우저에서 받는 데이터 조각의 현실

AI 챗봇 프론트엔드에서 `Response.body.getReader()`로 읽는 청크는 다음과 같은 상황이 발생할 수 있습니다:

#### 1. 불완전한 data: 필드 수신

```javascript
// 첫 번째 청크
"data: {\"content\": \"안녕";

// 두 번째 청크
"하세요\"}\n\n";
```

#### 2. 하나의 청크에 여러 data: 필드

```javascript
// 한 번에 여러 이벤트가 도착
"data: {\"content\": \"안녕\"}\n\ndata: {\"content\": \"하세요\"}\n\n";
```

#### 3. 경계가 애매한 경우

```javascript
// 청크가 정확히 \n 중간에서 끊어진 경우
"data: {\"content\": \"안녕하세요\"}\n";
// 다음 청크
"\ndata: {\"content\": \"무엇을\"}\n\n";
```

### 안전한 파싱 구현

```javascript
// 버퍼를 사용한 안전한 파싱
let buffer = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  // 청크를 버퍼에 추가
  buffer += new TextDecoder().decode(value);

  // 완전한 이벤트만 처리
  const lines = buffer.split('\n');

  // 마지막 불완전한 라인은 버퍼에 남김
  buffer = lines.pop() || '';

  for (const line of lines) {
    if (line.startsWith('data: ')) {
      // 완전한 data 필드 처리
      const data = line.slice(6);
      handleData(data);
    }
  }
}
```

## 3. 검증된 UX 패턴과 전략

### 1. Progressive Disclosure (점진적 공개)

불완전한 데이터라도 사용자에게 "뭔가 일어나고 있음"을 계속 보여줍니다.

```jsx
function StreamingMessage() {
  return (
    <>
      {/* 1단계: 타이핑 인디케이터 */}
      {isConnecting && <TypingIndicator />}

      {/* 2단계: 메타데이터 먼저 표시 */}
      {messageStarted && (
        <MessageHeader
          model={model}
          timestamp={timestamp}
        />
      )}

      {/* 3단계: 콘텐츠 점진적 표시 */}
      {content && <MessageContent text={content} />}

      {/* 4단계: 완료 상태 */}
      {isComplete && <MessageFooter tokens={tokenCount} />}
    </>
  );
}
```

### 2. Skeleton UI with Graceful Transitions

```jsx
function ToolCallDisplay({ toolCall }) {
  // 불완전한 상태에서도 스켈레톤 표시
  if (!toolCall.name) {
    return <ToolCallSkeleton />;
  }

  return (
    <ToolCallCard>
      <ToolName>{toolCall.name}</ToolName>

      {/* 인자가 아직 파싱 중이어도 UI는 안정적 */}
      <ToolArguments>
        {toolCall.arguments ? (
          <JsonViewer data={safeParseJSON(toolCall.arguments)} />
        ) : (
          <PulsingPlaceholder>인자 수신 중...</PulsingPlaceholder>
        )}
      </ToolArguments>
    </ToolCallCard>
  );
}
```

### 3. Intent-Based Buffering (의도 기반 버퍼링)

특정 패턴을 감지할 때까지 버퍼링하여 UI 깜빡임을 방지합니다.

```javascript
class IntentDetector {
  detectEventType(buffer) {
    // 이벤트 타입을 미리 감지
    if (buffer.includes('"thinking":')) return 'reasoning';
    if (buffer.includes('"tool_calls":')) return 'tool_call';
    if (buffer.includes('"content":')) return 'message';
    return 'unknown';
  }

  shouldRenderPartial(eventType, buffer) {
    // 이벤트 타입별로 렌더링 시점 결정
    switch (eventType) {
      case 'reasoning':
        // reasoning은 문장 단위로 표시
        return buffer.includes('\n');
      case 'tool_call':
        // 툴콜은 이름이 완성되면 표시
        return buffer.includes('"name"') && buffer.includes('}');
      case 'message':
        // 일반 메시지는 즉시 표시
        return true;
    }
  }
}
```

### 4. Optimistic UI with Error Recovery

```jsx
function ThinkingBlock({ stream }) {
  const [state, setState] = useState('buffering');
  const [content, setContent] = useState('');

  return (
    <CollapsibleCard
      icon={state === 'buffering' ? <SpinnerIcon /> : <BrainIcon />}
      title={state === 'buffering' ? '생각 중...' : '사고 과정'}
    >
      {/* 불완전한 상태에서도 안정적인 UI */}
      <ThinkingContent>{content || <PulsingLine />}</ThinkingContent>

      {/* 에러 시 graceful fallback */}
      {state === 'error' && <ErrorHint>일부 내용이 누락되었을 수 있습니다</ErrorHint>}
    </CollapsibleCard>
  );
}
```

### 5. Adaptive Rendering Strategy

```jsx
const RenderingStrategies = {
  // 텍스트: 즉시 렌더링
  text: {
    bufferSize: 0,
    renderer: (data) => <TextSpan>{data}</TextSpan>,
  },

  // 코드: 구문 단위로 버퍼링
  code: {
    bufferUntil: /[\n;{}]/,
    renderer: (data) => <CodeBlock language={detectLanguage(data)}>{data}</CodeBlock>,
  },

  // JSON: 유효한 JSON까지 버퍼링
  json: {
    validator: isValidJSON,
    fallback: (data) => <PreformattedText>{data}</PreformattedText>,
    renderer: (data) => <JsonViewer data={JSON.parse(data)} />,
  },

  // 수식: LaTeX 블록 완성까지 버퍼링
  math: {
    bufferUntil: /\$\$/,
    placeholder: <MathPlaceholder />,
    renderer: (data) => <MathDisplay>{data}</MathDisplay>,
  },
};
```

### 6. Visual Stability Patterns

```css
/* 컨테이너 높이 예약으로 점프 방지 */
.message-container {
  min-height: 2.5rem;
  transition: height 0.2s ease-out;
}

/* 툴콜 카드의 확장 애니메이션 */
.tool-call-card {
  overflow: hidden;
  animation: slideDown 0.3s ease-out;
}

@keyframes slideDown {
  from {
    max-height: 3rem;
    opacity: 0.7;
  }
  to {
    max-height: 500px;
    opacity: 1;
  }
}
```

### 7. State Machine 기반 UI 관리

```typescript
type StreamState =
  | 'idle'
  | 'connecting'
  | 'streaming-metadata'
  | 'streaming-content'
  | 'streaming-tool'
  | 'completing'
  | 'complete'
  | 'error';

const streamStateTransitions = {
  'idle': ['connecting'],
  'connecting': ['streaming-metadata', 'error'],
  'streaming-metadata': ['streaming-content', 'streaming-tool'],
  'streaming-content': ['completing', 'streaming-tool'],
  'streaming-tool': ['streaming-content', 'completing'],
  'completing': ['complete'],
  'complete': ['idle'],
  'error': ['idle'],
};

// UI는 상태에 따라 안정적으로 렌더링
function MessageUI({ state, data }) {
  switch (state) {
    case 'connecting':
      return <PulsingDots />;
    case 'streaming-metadata':
      return <MessageHeader {...data} />;
    case 'streaming-content':
      return <StreamingText content={data.content} />;
    case 'streaming-tool':
      return <ToolCallAnimation tool={data.tool} />;
    // ...
  }
}
```

## 4. Vercel AI SDK 활용법

### 기본 제공 기능

#### 자동 스트리밍 UI 업데이트

```typescript
const { messages, input, handleSubmit, isLoading, error } = useChat({
  api: '/api/chat',
  // 툴 콜 자동 처리
  onToolCall: async ({ toolCall }) => {
    // 자동 클라이언트 사이드 툴 실행
    return await executeToolOnClient(toolCall);
  },
});
```

#### Data Stream Protocol

AI SDK는 다양한 타입의 스트림 파트를 지원합니다:

- 텍스트 콘텐츠
- 툴 콜 (시작, 델타, 완료)
- 리즈닝 스텝
- 메타데이터 (usage, finish reason)

### Progressive Enhancement 기본 지원

```typescript
// messages 배열이 자동으로 업데이트되어 점진적 UI 렌더링
{
  messages.map((message) => (
    <Message
      key={message.id}
      content={message.content}
      toolInvocations={message.toolInvocations}
      attachments={message.attachments}
    />
  ));
}
```

## 5. 커스텀 프로바이더 구현 가이드

### Custom Stream Parts 처리

```typescript
import { createDataStreamResponse, pipeDataStreamToResponse } from 'ai';

export async function POST(req: Request) {
  const dataStream = createDataStreamResponse();

  // 커스텀 프로바이더 응답 처리
  customProvider.on('thinking', (data) => {
    // Data Stream Protocol 형식으로 변환
    dataStream.write({
      type: 'custom',
      value: { type: 'thinking', content: data },
    });
  });

  return dataStream.response;
}
```

### Adapter 패턴 구현

```typescript
// Custom Provider Adapter
export function createCustomProviderAdapter(customProvider) {
  return {
    async streamText(params) {
      const response = await customProvider.generate(params);

      return {
        async *textStream() {
          for await (const chunk of response) {
            yield chunk.text;
          }
        },

        async *fullStream() {
          for await (const chunk of response) {
            // 커스텀 이벤트를 AI SDK 형식으로 매핑
            if (chunk.type === 'thinking') {
              yield { type: 'text-delta', textDelta: `[Thinking: ${chunk.content}]` };
            } else if (chunk.type === 'tool_call') {
              yield { type: 'tool-call', ...chunk };
            }
          }
        },
      };
    },
  };
}
```

### 재사용 가능한 UI 컴포넌트 라이브러리

```tsx
export const StreamingComponents = {
  ThinkingIndicator: ({ isThinking }) => (
    <AnimatePresence>
      {isThinking && (
        <motion.div
          initial={{ opacity: 0, height: 0 }}
          animate={{ opacity: 1, height: 'auto' }}
          exit={{ opacity: 0, height: 0 }}
        >
          <PulsingBrain />
        </motion.div>
      )}
    </AnimatePresence>
  ),

  ToolCallCard: ({ toolCall, status }) => (
    <Card className={`tool-call-${status}`}>
      <CardHeader>
        <ToolIcon name={toolCall.name} />
        <ToolName>{toolCall.name}</ToolName>
      </CardHeader>
      <CardContent>
        {status === 'streaming' ? (
          <JsonSkeleton lines={3} />
        ) : (
          <JsonViewer data={toolCall.args} />
        )}
      </CardContent>
    </Card>
  ),
};
```

## 6. 프로덕션 레벨 구현 예제

### 완전한 스트리밍 채팅 컴포넌트

```typescript
import React, { useState, useEffect, useRef } from 'react';
import { useChat } from '@ai-sdk/react';
import { StreamingComponents } from './components/streaming';

const StreamingChat: React.FC = () => {
  const { messages, input, handleInputChange, handleSubmit, isLoading, error } = useChat({
    api: '/api/chat',
    onError: (error) => {
      console.error('Chat error:', error);
    },
  });

  const messagesEndRef = useRef<HTMLDivElement>(null);

  // 자동 스크롤
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  return (
    <div className="streaming-chat">
      <div className="messages-container">
        {messages.map((message) => (
          <MessageRenderer
            key={message.id}
            message={message}
            isStreaming={isLoading && message.id === messages[messages.length - 1]?.id}
          />
        ))}

        {isLoading && <StreamingComponents.ThinkingIndicator isThinking={true} />}

        <div ref={messagesEndRef} />
      </div>

      <form
        onSubmit={handleSubmit}
        className="input-container"
      >
        <input
          type="text"
          value={input}
          onChange={handleInputChange}
          placeholder="메시지를 입력하세요..."
          disabled={isLoading}
        />
        <button
          type="submit"
          disabled={isLoading || !input.trim()}
        >
          전송
        </button>
      </form>
    </div>
  );
};

// 메시지 렌더러 컴포넌트
function MessageRenderer({ message, isStreaming }) {
  // 툴 콜 렌더링
  if (message.toolInvocations && message.toolInvocations.length > 0) {
    return (
      <div className="message-with-tools">
        {message.content && <div className="message-content">{message.content}</div>}
        {message.toolInvocations.map((invocation) => (
          <StreamingComponents.ToolCallCard
            key={invocation.toolCallId}
            toolCall={invocation}
            status={isStreaming ? 'streaming' : 'complete'}
          />
        ))}
      </div>
    );
  }

  // 일반 메시지 렌더링
  return (
    <div className={`message ${message.role}`}>
      <div className="message-header">
        <span className="role">{message.role}</span>
        <span className="timestamp">{new Date(message.createdAt).toLocaleTimeString()}</span>
      </div>
      <div className="message-content">
        {isStreaming ? (
          <StreamingText text={message.content} />
        ) : (
          <MarkdownRenderer content={message.content} />
        )}
      </div>
    </div>
  );
}

// 스트리밍 텍스트 컴포넌트
function StreamingText({ text }) {
  const [displayText, setDisplayText] = useState('');
  const [showCursor, setShowCursor] = useState(true);

  useEffect(() => {
    setDisplayText(text);
  }, [text]);

  useEffect(() => {
    const interval = setInterval(() => {
      setShowCursor((prev) => !prev);
    }, 500);
    return () => clearInterval(interval);
  }, []);

  return (
    <span>
      {displayText}
      {showCursor && <span className="cursor">▊</span>}
    </span>
  );
}
```

### 서버 측 구현 (Next.js App Router)

```typescript
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai('gpt-4'),
    messages,
    maxSteps: 5, // 툴 콜 연속 실행 허용
    async onStepFinish({ text, toolCalls, toolResults, finishReason, usage }) {
      // 각 스텝 완료 시 로깅
      console.log('Step finished:', {
        hasToolCalls: toolCalls.length > 0,
        finishReason,
      });
    },
  });

  return result.toDataStreamResponse();
}
```

## 핵심 요약

### ✅ Vercel AI SDK 기본 제공

- 텍스트 스트리밍
- 기본 툴 콜
- 메시지 상태 관리
- 에러 처리
- 자동 UI 업데이트

### 🔧 커스텀 구현 필요

- 커스텀 이벤트 타입 (thinking, status 등)
- 시각적 전환 효과
- 의도 기반 버퍼링
- 특수 UI 패턴

### 🎯 Best Practices

1. **Never Break**: 불완전한 데이터로 UI가 깨지지 않도록
2. **Always Respond**: 항상 무언가 일어나고 있음을 표시
3. **Graceful Degradation**: 에러 시에도 우아하게 처리
4. **Predictable Behavior**: 사용자가 예측 가능한 동작
5. **Progressive Enhancement**: 데이터가 도착할수록 UI가 풍부해짐
