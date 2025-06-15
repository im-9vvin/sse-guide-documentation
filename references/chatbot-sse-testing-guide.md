# 챗봇 SSE 응답 파싱 및 컴포넌트 렌더링 테스트 가이드

## 🧪 단위 테스트 Best Practices

### 1. SSE EventSource 모킹

Mock Service Worker (MSW)를 사용하여 SSE 이벤트를 모킹할 수 있습니다. MSW는 Storybook, 단위 테스트, 기타 개발 환경에서 재사용 가능한 JavaScript API 모킹 도구로 인기가 높습니다.

```javascript
// MSW를 사용한 SSE 모킹
import { rest } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  rest.get('/api/chat/stream', (req, res, ctx) => {
    return res(
      ctx.set('Content-Type', 'text/event-stream'),
      ctx.set('Cache-Control', 'no-cache'),
      ctx.set('Connection', 'keep-alive'),
      ctx.body(
        `data: {"type": "message", "content": "Hello"}\n\ndata: {"type": "message", "content": " World!"}\n\ndata: [DONE]\n\n`
      )
    );
  })
);

// 테스트 설정
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### 2. Jest + React Testing Library 조합

Jest와 React Testing Library는 상호 보완적인 역할을 합니다. Jest는 테스트 러너와 내장 어설션, 모킹, 테스트 구성, 커버리지 리포팅을 제공하고, React Testing Library는 사용자 관점에서 컴포넌트를 테스트하는 경량 라이브러리입니다.

```javascript
// 챗봇 스트리밍 컴포넌트 테스트
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ChatBot } from './ChatBot';

// EventSource 전역 모킹
const mockEventSource = {
  addEventListener: jest.fn(),
  removeEventListener: jest.fn(),
  close: jest.fn(),
  readyState: 1, // EventSource.OPEN
};

beforeEach(() => {
  global.EventSource = jest.fn(() => mockEventSource);
});

afterEach(() => {
  jest.restoreAllMocks();
});

test('should display streaming messages from SSE', async () => {
  const user = userEvent.setup();
  render(<ChatBot />);

  // 메시지 입력
  const input = screen.getByRole('textbox', { name: /message input/i });
  await user.type(input, 'Hello AI');
  await user.click(screen.getByRole('button', { name: /send/i }));

  // SSE 이벤트 시뮬레이션
  const messageHandler = mockEventSource.addEventListener.mock.calls.find(
    (call) => call[0] === 'message'
  )[1];

  // 첫 번째 청크
  messageHandler({ data: '{"content": "Hello"}' });
  expect(screen.getByText(/Hello/)).toBeInTheDocument();

  // 두 번째 청크 (누적)
  messageHandler({ data: '{"content": " World!"}' });
  expect(screen.getByText(/Hello World!/)).toBeInTheDocument();

  // 스트림 완료
  const errorHandler = mockEventSource.addEventListener.mock.calls.find(
    (call) => call[0] === 'error'
  )[1];

  expect(mockEventSource.close).not.toHaveBeenCalled();
});

test('should handle SSE connection errors gracefully', async () => {
  const user = userEvent.setup();
  render(<ChatBot />);

  const input = screen.getByRole('textbox', { name: /message input/i });
  await user.type(input, 'Hello AI');
  await user.click(screen.getByRole('button', { name: /send/i }));

  // 에러 이벤트 시뮬레이션
  const errorHandler = mockEventSource.addEventListener.mock.calls.find(
    (call) => call[0] === 'error'
  )[1];

  errorHandler({ target: { readyState: EventSource.CLOSED } });

  expect(screen.getByText(/connection error/i)).toBeInTheDocument();
  expect(mockEventSource.close).toHaveBeenCalled();
});
```

### 3. 파싱 로직 별도 테스트

테스트 가능한 작은 단위로 분리하여 특정 컴포넌트보다는 구체적인 기능을 대상으로 해야 합니다.

```javascript
// sseParser.js
export function parseSSEMessage(data) {
  try {
    if (data === '[DONE]') {
      return { type: 'done' };
    }
    return JSON.parse(data);
  } catch (error) {
    console.warn('Failed to parse SSE message:', data);
    return null;
  }
}

export function accumulateStreamContent(previous, newChunk) {
  if (!newChunk || newChunk.type === 'done') {
    return previous;
  }
  return previous + (newChunk.content || '');
}

// sseParser.test.js
import { parseSSEMessage, accumulateStreamContent } from './sseParser';

describe('SSE Parser', () => {
  test('should parse valid JSON message', () => {
    const result = parseSSEMessage('{"type": "message", "content": "Hello"}');
    expect(result).toEqual({ type: 'message', content: 'Hello' });
  });

  test('should handle [DONE] signal', () => {
    const result = parseSSEMessage('[DONE]');
    expect(result).toEqual({ type: 'done' });
  });

  test('should handle malformed JSON gracefully', () => {
    const result = parseSSEMessage('invalid json');
    expect(result).toBeNull();
  });

  test('should accumulate stream content correctly', () => {
    let content = '';
    content = accumulateStreamContent(content, { content: 'Hello' });
    content = accumulateStreamContent(content, { content: ' World' });
    content = accumulateStreamContent(content, { content: '!' });

    expect(content).toBe('Hello World!');
  });
});
```

## 📚 Storybook 활용 방법

### Storybook에서 SSE 테스트

MSW는 Storybook에서도 사용할 수 있어 개발 중에 SSE 동작을 시각적으로 확인할 수 있습니다.

```javascript
// .storybook/preview.js
import { initialize, mswDecorator } from 'msw-storybook-addon';

initialize({
  onUnhandledRequest: 'warn',
});

export const decorators = [mswDecorator];

// ChatBot.stories.js
import { rest } from 'msw';
import { ChatBot } from './ChatBot';

export default {
  title: 'Components/ChatBot',
  component: ChatBot,
  parameters: {
    msw: {
      handlers: [
        rest.get('/api/chat/stream', (req, res, ctx) => {
          return res(
            ctx.set('Content-Type', 'text/event-stream'),
            ctx.set('Cache-Control', 'no-cache'),
            ctx.body(
              `data: {"content": "안녕하세요! "}\n\ndata: {"content": "무엇을 도와드릴까요?"}\n\ndata: [DONE]\n\n`
            )
          );
        }),
      ],
    },
  },
};

export const Default = {
  name: '기본 챗봇',
};

export const StreamingResponse = {
  name: '스트리밍 응답',
  parameters: {
    msw: {
      handlers: [
        rest.get('/api/chat/stream', (req, res, ctx) => {
          // 긴 응답을 시뮬레이션
          const chunks = [
            '안녕하세요! ',
            '저는 AI ',
            '어시스턴트입니다. ',
            '오늘 무엇을 ',
            '도와드릴까요?',
          ];

          let responseBody = '';
          chunks.forEach((chunk) => {
            responseBody += `data: {"content": "${chunk}"}\n\n`;
          });
          responseBody += `data: [DONE]\n\n`;

          return res(ctx.set('Content-Type', 'text/event-stream'), ctx.body(responseBody));
        }),
      ],
    },
  },
};

export const ErrorHandling = {
  name: '에러 처리',
  parameters: {
    msw: {
      handlers: [
        rest.get('/api/chat/stream', (req, res, ctx) => {
          return res(ctx.status(500));
        }),
      ],
    },
  },
};
```

## 🎯 Best Practices 요약

### 테스트 구조

1. **AAA 패턴 사용**: Arrange, Act, Assert 패턴으로 테스트를 구조화
2. **사용자 중심 테스트**: \*ByRole 쿼리를 기본으로 사용하여 접근성도 함께 확보
3. **비동기 처리**: findBy\* 쿼리와 waitFor를 사용하여 비동기 UI 변경사항을 기다림

### 모킹 전략

외부 의존성을 모킹하여 테스트를 빠르고 안정적으로 만들고, 특히 네트워크 활동이 포함된 테스트에서 모킹은 테스트 속도를 크게 향상시킵니다.

```javascript
// 효율적인 모킹 설정
describe('ChatBot Component', () => {
  let mockEventSource;

  beforeEach(() => {
    mockEventSource = {
      addEventListener: jest.fn(),
      removeEventListener: jest.fn(),
      close: jest.fn(),
      readyState: EventSource.OPEN,
      url: '/api/chat/stream',
    };

    global.EventSource = jest.fn(() => mockEventSource);

    // fetch도 함께 모킹 (POST 요청용)
    global.fetch = jest.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve({ success: true }),
      })
    );
  });

  afterEach(() => {
    jest.restoreAllMocks();
  });

  // 테스트 케이스들...
});
```

### 컴포넌트 렌더링 테스트

스트리밍 응답을 받을 때 상태 변수와 refs를 적절히 업데이트하여 UI 컴포넌트가 각각의 상태 변수에 바인딩되도록 하면, 새로운 데이터 청크가 수신될 때마다 웹사이트의 출력이 업데이트됩니다.

```javascript
// 실제 컴포넌트 구조를 테스트
test('should update UI incrementally during streaming', async () => {
  const user = userEvent.setup();
  render(<ChatBot />);

  // 메시지 전송
  await user.type(screen.getByRole('textbox'), 'Hello');
  await user.click(screen.getByRole('button', { name: /send/i }));

  // 로딩 상태 확인
  expect(screen.getByText(/thinking/i)).toBeInTheDocument();

  // 스트리밍 시뮬레이션
  const messageHandler = mockEventSource.addEventListener.mock.calls.find(
    (call) => call[0] === 'message'
  )[1];

  // 점진적 업데이트 테스트
  messageHandler({ data: '{"content": "안녕"}' });
  expect(screen.getByText('안녕')).toBeInTheDocument();

  messageHandler({ data: '{"content": "하세요!"}' });
  expect(screen.getByText('안녕하세요!')).toBeInTheDocument();

  // 완료 상태 확인
  messageHandler({ data: '[DONE]' });
  expect(screen.queryByText(/thinking/i)).not.toBeInTheDocument();
});
```

### 성능 최적화

얕은 렌더링 사용, 무거운 의존성 모킹, 병렬 테스트 실행, 스냅샷 테스트 제한으로 테스트 성능을 향상시킬 수 있습니다.

```javascript
// 성능 최적화 예시
describe('ChatBot Performance Tests', () => {
  // 자식 컴포넌트 모킹으로 렌더링 최적화
  jest.mock('./MessageList', () => {
    return function MockMessageList(props) {
      return <div data-testid="message-list">{props.messages.length} messages</div>;
    };
  });

  // 무거운 의존성 모킹
  jest.mock('./MarkdownRenderer', () => {
    return function MockMarkdownRenderer({ content }) {
      return <div data-testid="markdown">{content}</div>;
    };
  });

  test('should handle large message volumes efficiently', async () => {
    render(<ChatBot />);

    // 대량 메시지 테스트를 위한 빠른 시뮬레이션
    const messageHandler = mockEventSource.addEventListener.mock.calls.find(
      (call) => call[0] === 'message'
    )[1];

    // 100개 청크 빠르게 처리
    for (let i = 0; i < 100; i++) {
      messageHandler({ data: `{"content": "chunk ${i} "}` });
    }

    expect(screen.getByTestId('message-list')).toBeInTheDocument();
  });
});
```

## 🔧 권장 도구 스택

1. **테스트 프레임워크**: Jest + React Testing Library
2. **모킹**: Mock Service Worker (MSW)
3. **시각적 테스트**: Storybook + MSW addon
4. **타입 안전성**: TypeScript로 이벤트 데이터 구조 정의
5. **추가 도구**:
   - `@testing-library/user-event` - 더 현실적인 사용자 상호작용
   - `@testing-library/jest-dom` - 추가 DOM 매처

## 📝 타입 정의 예시

```typescript
// types/sse.ts
export interface SSEMessage {
  type: 'message' | 'error' | 'done';
  content?: string;
  error?: string;
  timestamp?: number;
}

export interface ChatMessage {
  id: string;
  content: string;
  sender: 'user' | 'assistant';
  timestamp: Date;
  isStreaming?: boolean;
}

// 테스트에서 타입 안전성 확보
const mockMessage: SSEMessage = {
  type: 'message',
  content: 'Hello',
  timestamp: Date.now(),
};
```

이 가이드를 따르면 SSE 스트리밍 챗봇의 파싱 로직과 렌더링을 효과적으로 테스트할 수 있으며, Storybook을 통해 개발 과정에서도 시각적으로 확인할 수 있습니다.
