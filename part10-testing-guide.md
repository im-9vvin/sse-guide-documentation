***
**[← 이전: Part 9 - 보안과 인프라](./part9-security-infrastructure.md)** | **[목차](./README.md)** | **[다음: Part 11 - 프로덕션 예제 →](./part11-production-examples.md)**

***

# Part 10: 테스팅 가이드 (Testing Guide)

## 목차
- [단위 테스트 모범 사례](#단위-테스트-모범-사례)
- [EventSource 모킹](#eventsource-모킹)
- [Jest + React Testing Library](#jest--react-testing-library)
- [파서 로직 테스팅](#파서-로직-테스팅)
- [Storybook SSE 테스팅](#storybook-sse-테스팅)

## 단위 테스트 모범 사례

### SSE 클라이언트 테스트 전략

```javascript
// __tests__/SSEClient.test.js
import { SSEClient } from '../src/SSEClient';
import { EventSourcePolyfill } from 'event-source-polyfill';

// EventSource 모킹
jest.mock('event-source-polyfill');

describe('SSEClient', () => {
  let client;
  let mockEventSource;

  beforeEach(() => {
    // EventSource 인스턴스 모킹
    mockEventSource = {
      addEventListener: jest.fn(),
      removeEventListener: jest.fn(),
      close: jest.fn(),
      readyState: EventSource.CONNECTING
    };

    EventSourcePolyfill.mockImplementation(() => mockEventSource);
    client = new SSEClient('https://api.example.com/events');
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('연결 관리', () => {
    test('연결 시작 시 EventSource를 생성해야 함', () => {
      client.connect();

      expect(EventSourcePolyfill).toHaveBeenCalledWith(
        'https://api.example.com/events',
        expect.objectContaining({
          withCredentials: true
        })
      );
    });

    test('연결 종료 시 EventSource를 닫아야 함', () => {
      client.connect();
      client.disconnect();

      expect(mockEventSource.close).toHaveBeenCalled();
      expect(client.eventSource).toBeNull();
    });

    test('중복 연결 시도를 방지해야 함', () => {
      client.connect();
      client.connect();

      expect(EventSourcePolyfill).toHaveBeenCalledTimes(1);
    });
  });

  describe('이벤트 처리', () => {
    test('커스텀 이벤트 리스너를 등록해야 함', () => {
      const handler = jest.fn();
      client.on('message', handler);
      client.connect();

      expect(mockEventSource.addEventListener)
        .toHaveBeenCalledWith('message', expect.any(Function));
    });

    test('이벤트 수신 시 핸들러를 호출해야 함', () => {
      const handler = jest.fn();
      client.on('update', handler);
      client.connect();

      // addEventListener 호출 시 등록된 콜백 추출
      const [eventType, callback] = mockEventSource.addEventListener.mock.calls[0];
      
      // 이벤트 시뮬레이션
      const mockEvent = {
        type: 'update',
        data: JSON.stringify({ id: 1, status: 'completed' })
      };
      callback(mockEvent);

      expect(handler).toHaveBeenCalledWith({
        id: 1,
        status: 'completed'
      });
    });

    test('JSON 파싱 에러를 처리해야 함', () => {
      const errorHandler = jest.fn();
      client.on('error', errorHandler);
      client.connect();

      const [, callback] = mockEventSource.addEventListener.mock.calls.find(
        call => call[0] === 'message'
      );

      const mockEvent = {
        type: 'message',
        data: 'invalid json'
      };
      callback(mockEvent);

      expect(errorHandler).toHaveBeenCalledWith(
        expect.objectContaining({
          message: expect.stringContaining('JSON')
        })
      );
    });
  });

  describe('재연결 로직', () => {
    jest.useFakeTimers();

    test('연결 에러 시 재연결을 시도해야 함', () => {
      client.connect();

      // 에러 이벤트 시뮬레이션
      const [, errorCallback] = mockEventSource.addEventListener.mock.calls.find(
        call => call[0] === 'error'
      );
      errorCallback(new Event('error'));

      // 재연결 대기
      jest.advanceTimersByTime(1000);

      expect(EventSourcePolyfill).toHaveBeenCalledTimes(2);
    });

    test('최대 재시도 횟수를 초과하면 중단해야 함', () => {
      client = new SSEClient('https://api.example.com/events', {
        maxRetries: 3
      });
      client.connect();

      // 여러 번 에러 발생
      for (let i = 0; i < 4; i++) {
        const [, errorCallback] = mockEventSource.addEventListener.mock.calls.find(
          call => call[0] === 'error'
        );
        errorCallback(new Event('error'));
        jest.advanceTimersByTime(1000);
      }

      // 3번의 재시도 + 초기 연결 = 4번
      expect(EventSourcePolyfill).toHaveBeenCalledTimes(4);
    });

    test('백오프 전략을 적용해야 함', () => {
      client.connect();
      const connectSpy = jest.spyOn(client, 'connect');

      // 첫 번째 재시도 - 1초 후
      const [, errorCallback] = mockEventSource.addEventListener.mock.calls.find(
        call => call[0] === 'error'
      );
      errorCallback(new Event('error'));

      jest.advanceTimersByTime(999);
      expect(connectSpy).not.toHaveBeenCalled();

      jest.advanceTimersByTime(1);
      expect(connectSpy).toHaveBeenCalledTimes(1);

      // 두 번째 재시도 - 2초 후
      errorCallback(new Event('error'));
      jest.advanceTimersByTime(2000);
      expect(connectSpy).toHaveBeenCalledTimes(2);
    });
  });
});
```

### 서버 사이드 테스트

```javascript
// __tests__/SSEServer.test.js
import request from 'supertest';
import { createSSEApp } from '../src/server';
import { EventEmitter } from 'events';

describe('SSE Server', () => {
  let app;
  let eventBus;

  beforeEach(() => {
    eventBus = new EventEmitter();
    app = createSSEApp(eventBus);
  });

  describe('GET /events', () => {
    test('올바른 SSE 헤더를 반환해야 함', async () => {
      const response = await request(app)
        .get('/events')
        .expect(200);

      expect(response.headers['content-type']).toBe('text/event-stream');
      expect(response.headers['cache-control']).toBe('no-cache');
      expect(response.headers['connection']).toBe('keep-alive');
    });

    test('초기 연결 메시지를 전송해야 함', (done) => {
      request(app)
        .get('/events')
        .buffer(false)
        .parse((res, callback) => {
          let buffer = '';
          
          res.on('data', (chunk) => {
            buffer += chunk.toString();
            
            if (buffer.includes('\n\n')) {
              const messages = buffer.split('\n\n').filter(Boolean);
              const firstMessage = messages[0];
              
              expect(firstMessage).toContain('event: connected');
              expect(firstMessage).toContain('data: {"status":"connected"}');
              
              done();
              callback();
            }
          });
        })
        .end(() => {});
    });

    test('이벤트 버스의 이벤트를 전달해야 함', (done) => {
      let connection;
      
      request(app)
        .get('/events')
        .buffer(false)
        .parse((res, callback) => {
          let buffer = '';
          let messageCount = 0;
          
          res.on('data', (chunk) => {
            buffer += chunk.toString();
            const messages = buffer.split('\n\n').filter(Boolean);
            
            if (messages.length > messageCount) {
              messageCount = messages.length;
              
              if (messageCount === 2) {
                const secondMessage = messages[1];
                expect(secondMessage).toContain('event: update');
                expect(secondMessage).toContain('data: {"id":123,"status":"completed"}');
                
                done();
                callback();
              }
            }
          });
        })
        .end((err, res) => {
          if (!err) {
            connection = res;
            
            // 이벤트 발생
            setTimeout(() => {
              eventBus.emit('update', { id: 123, status: 'completed' });
            }, 100);
          }
        });
    });

    test('연결 종료 시 정리 작업을 수행해야 함', (done) => {
      const cleanupSpy = jest.fn();
      app.on('client:disconnect', cleanupSpy);

      const req = request(app)
        .get('/events')
        .buffer(false)
        .parse(() => {})
        .end(() => {
          setTimeout(() => {
            req.abort();
            
            setTimeout(() => {
              expect(cleanupSpy).toHaveBeenCalled();
              done();
            }, 100);
          }, 100);
        });
    });
  });

  describe('인증이 필요한 SSE', () => {
    test('인증 토큰이 없으면 401을 반환해야 함', async () => {
      await request(app)
        .get('/secure/events')
        .expect(401);
    });

    test('유효한 토큰으로 연결할 수 있어야 함', async () => {
      const token = 'valid-token';
      
      await request(app)
        .get('/secure/events')
        .set('Authorization', `Bearer ${token}`)
        .expect(200)
        .expect('Content-Type', 'text/event-stream');
    });
  });
});
```

## EventSource 모킹

### 커스텀 EventSource 모크

```javascript
// __mocks__/EventSourceMock.js
export class EventSourceMock {
  constructor(url, options) {
    this.url = url;
    this.options = options;
    this.readyState = EventSource.CONNECTING;
    this.listeners = new Map();
    this.onerror = null;
    this.onmessage = null;
    this.onopen = null;

    // 연결 시뮬레이션
    setTimeout(() => {
      this.readyState = EventSource.OPEN;
      this.dispatchEvent(new Event('open'));
    }, 0);
  }

  addEventListener(type, listener) {
    if (!this.listeners.has(type)) {
      this.listeners.set(type, new Set());
    }
    this.listeners.get(type).add(listener);
  }

  removeEventListener(type, listener) {
    const listeners = this.listeners.get(type);
    if (listeners) {
      listeners.delete(listener);
    }
  }

  dispatchEvent(event) {
    // 내장 이벤트 핸들러 호출
    const handler = this[`on${event.type}`];
    if (handler) {
      handler.call(this, event);
    }

    // 등록된 리스너 호출
    const listeners = this.listeners.get(event.type);
    if (listeners) {
      listeners.forEach(listener => listener.call(this, event));
    }
  }

  close() {
    this.readyState = EventSource.CLOSED;
  }

  // 테스트 헬퍼 메서드
  simulate(eventType, data) {
    const event = new MessageEvent(eventType, {
      data: typeof data === 'object' ? JSON.stringify(data) : data,
      origin: new URL(this.url).origin,
      lastEventId: ''
    });
    this.dispatchEvent(event);
  }

  simulateError(error = new Error('Connection failed')) {
    this.readyState = EventSource.CLOSED;
    const event = new Event('error');
    event.error = error;
    this.dispatchEvent(event);
  }
}

// 전역 모킹 설정
global.EventSource = EventSourceMock;
```

### 테스트 유틸리티

```javascript
// __tests__/utils/SSETestUtils.js
export class SSETestHelper {
  constructor() {
    this.mockEventSource = null;
    this.capturedEvents = [];
  }

  setupMockEventSource() {
    const self = this;
    
    class TestEventSource extends EventSourceMock {
      constructor(url, options) {
        super(url, options);
        self.mockEventSource = this;
      }

      dispatchEvent(event) {
        self.capturedEvents.push({
          type: event.type,
          data: event.data,
          timestamp: Date.now()
        });
        super.dispatchEvent(event);
      }
    }

    global.EventSource = TestEventSource;
    return TestEventSource;
  }

  async simulateSSESequence(events, delays = []) {
    for (let i = 0; i < events.length; i++) {
      const { type, data } = events[i];
      const delay = delays[i] || 0;

      await new Promise(resolve => setTimeout(resolve, delay));
      this.mockEventSource.simulate(type, data);
    }
  }

  expectEventSequence(expectedEvents) {
    expect(this.capturedEvents.length).toBe(expectedEvents.length);

    expectedEvents.forEach((expected, index) => {
      const actual = this.capturedEvents[index];
      expect(actual.type).toBe(expected.type);
      
      if (expected.data) {
        const actualData = JSON.parse(actual.data);
        expect(actualData).toEqual(expected.data);
      }
    });
  }

  reset() {
    this.capturedEvents = [];
    if (this.mockEventSource) {
      this.mockEventSource.close();
      this.mockEventSource = null;
    }
  }
}

// 사용 예제
describe('SSE Integration Test', () => {
  const helper = new SSETestHelper();

  beforeEach(() => {
    helper.setupMockEventSource();
  });

  afterEach(() => {
    helper.reset();
  });

  test('복잡한 이벤트 시퀀스를 처리해야 함', async () => {
    const client = new SSEClient('/events');
    const receivedEvents = [];

    client.on('update', data => receivedEvents.push(data));
    client.connect();

    // 이벤트 시퀀스 시뮬레이션
    await helper.simulateSSESequence([
      { type: 'update', data: { id: 1, status: 'started' } },
      { type: 'update', data: { id: 1, status: 'progress', percent: 50 } },
      { type: 'update', data: { id: 1, status: 'completed' } }
    ], [0, 100, 200]);

    // 검증
    expect(receivedEvents).toHaveLength(3);
    expect(receivedEvents[2].status).toBe('completed');
    
    helper.expectEventSequence([
      { type: 'open' },
      { type: 'update', data: { id: 1, status: 'started' } },
      { type: 'update', data: { id: 1, status: 'progress', percent: 50 } },
      { type: 'update', data: { id: 1, status: 'completed' } }
    ]);
  });
});
```

## Jest + React Testing Library

### React 컴포넌트 SSE 테스트

```javascript
// __tests__/StreamingChat.test.jsx
import React from 'react';
import { render, screen, waitFor, act } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { StreamingChat } from '../src/components/StreamingChat';
import { SSEProvider } from '../src/contexts/SSEContext';

// SSE 컨텍스트 모킹
jest.mock('../src/contexts/SSEContext', () => ({
  ...jest.requireActual('../src/contexts/SSEContext'),
  useSSE: jest.fn()
}));

describe('StreamingChat Component', () => {
  let mockSSE;

  beforeEach(() => {
    mockSSE = {
      connected: false,
      messages: [],
      connect: jest.fn(),
      disconnect: jest.fn(),
      sendMessage: jest.fn()
    };

    require('../src/contexts/SSEContext').useSSE.mockReturnValue(mockSSE);
  });

  test('연결 상태를 표시해야 함', () => {
    render(
      <SSEProvider>
        <StreamingChat />
      </SSEProvider>
    );

    expect(screen.getByText(/disconnected/i)).toBeInTheDocument();
  });

  test('연결 버튼 클릭 시 SSE 연결을 시작해야 함', async () => {
    const user = userEvent.setup();
    
    render(
      <SSEProvider>
        <StreamingChat />
      </SSEProvider>
    );

    const connectButton = screen.getByRole('button', { name: /connect/i });
    await user.click(connectButton);

    expect(mockSSE.connect).toHaveBeenCalled();
  });

  test('스트리밍 메시지를 실시간으로 렌더링해야 함', async () => {
    const { rerender } = render(
      <SSEProvider>
        <StreamingChat />
      </SSEProvider>
    );

    // 연결 상태로 변경
    mockSSE.connected = true;
    mockSSE.messages = [
      { id: '1', content: 'Hello', timestamp: Date.now() }
    ];

    rerender(
      <SSEProvider>
        <StreamingChat />
      </SSEProvider>
    );

    await waitFor(() => {
      expect(screen.getByText('Hello')).toBeInTheDocument();
    });

    // 새 메시지 추가
    act(() => {
      mockSSE.messages = [
        ...mockSSE.messages,
        { id: '2', content: 'World', timestamp: Date.now() }
      ];
    });

    rerender(
      <SSEProvider>
        <StreamingChat />
      </SSEProvider>
    );

    await waitFor(() => {
      expect(screen.getByText('World')).toBeInTheDocument();
    });
  });

  test('부분적인 메시지 업데이트를 처리해야 함', async () => {
    const { rerender } = render(
      <SSEProvider>
        <StreamingChat />
      </SSEProvider>
    );

    // 초기 메시지
    mockSSE.messages = [
      { id: '1', content: 'Loading', streaming: true }
    ];

    rerender(
      <SSEProvider>
        <StreamingChat />
      </SSEProvider>
    );

    expect(screen.getByText('Loading')).toBeInTheDocument();
    expect(screen.getByTestId('streaming-indicator')).toBeInTheDocument();

    // 메시지 업데이트
    act(() => {
      mockSSE.messages = [
        { id: '1', content: 'Loading... Complete!', streaming: false }
      ];
    });

    rerender(
      <SSEProvider>
        <StreamingChat />
      </SSEProvider>
    );

    await waitFor(() => {
      expect(screen.getByText('Loading... Complete!')).toBeInTheDocument();
      expect(screen.queryByTestId('streaming-indicator')).not.toBeInTheDocument();
    });
  });
});
```

### 커스텀 훅 테스트

```javascript
// __tests__/useSSE.test.js
import { renderHook, act, waitFor } from '@testing-library/react';
import { useSSE } from '../src/hooks/useSSE';
import { EventSourceMock } from '../__mocks__/EventSourceMock';

describe('useSSE Hook', () => {
  beforeEach(() => {
    global.EventSource = EventSourceMock;
  });

  test('초기 상태를 올바르게 설정해야 함', () => {
    const { result } = renderHook(() => useSSE('/events'));

    expect(result.current.status).toBe('disconnected');
    expect(result.current.data).toBeNull();
    expect(result.current.error).toBeNull();
  });

  test('연결 및 메시지 수신을 처리해야 함', async () => {
    const { result } = renderHook(() => useSSE('/events'));

    // 연결 시작
    act(() => {
      result.current.connect();
    });

    await waitFor(() => {
      expect(result.current.status).toBe('connected');
    });

    // 메시지 시뮬레이션
    const mockData = { message: 'Hello SSE' };
    act(() => {
      const eventSource = global.EventSource.mock.instances[0];
      eventSource.simulate('message', mockData);
    });

    expect(result.current.data).toEqual(mockData);
  });

  test('에러 상태를 처리해야 함', async () => {
    const { result } = renderHook(() => useSSE('/events'));

    act(() => {
      result.current.connect();
    });

    // 에러 시뮬레이션
    act(() => {
      const eventSource = global.EventSource.mock.instances[0];
      eventSource.simulateError(new Error('Connection failed'));
    });

    await waitFor(() => {
      expect(result.current.status).toBe('error');
      expect(result.current.error).toEqual(
        expect.objectContaining({
          message: 'Connection failed'
        })
      );
    });
  });

  test('재연결 로직을 실행해야 함', async () => {
    jest.useFakeTimers();
    
    const { result } = renderHook(() => 
      useSSE('/events', { 
        reconnect: true,
        reconnectInterval: 1000 
      })
    );

    act(() => {
      result.current.connect();
    });

    // 에러 발생
    act(() => {
      const eventSource = global.EventSource.mock.instances[0];
      eventSource.simulateError();
    });

    expect(result.current.status).toBe('reconnecting');

    // 재연결 대기
    act(() => {
      jest.advanceTimersByTime(1000);
    });

    await waitFor(() => {
      expect(global.EventSource).toHaveBeenCalledTimes(2);
    });

    jest.useRealTimers();
  });

  test('언마운트 시 연결을 정리해야 함', () => {
    const { result, unmount } = renderHook(() => useSSE('/events'));

    act(() => {
      result.current.connect();
    });

    const eventSource = global.EventSource.mock.instances[0];
    const closeSpy = jest.spyOn(eventSource, 'close');

    unmount();

    expect(closeSpy).toHaveBeenCalled();
  });
});
```

## 파서 로직 테스팅

### SSE 파서 단위 테스트

```javascript
// __tests__/SSEParser.test.js
import { SSEParser } from '../src/utils/SSEParser';

describe('SSEParser', () => {
  let parser;

  beforeEach(() => {
    parser = new SSEParser();
  });

  describe('기본 파싱', () => {
    test('단일 필드를 파싱해야 함', () => {
      const result = parser.parse('data: hello world\n\n');
      
      expect(result).toEqual([{
        data: 'hello world'
      }]);
    });

    test('여러 필드를 가진 이벤트를 파싱해야 함', () => {
      const result = parser.parse(
        'event: update\n' +
        'data: {"id": 1}\n' +
        'id: 123\n' +
        'retry: 5000\n\n'
      );

      expect(result).toEqual([{
        event: 'update',
        data: '{"id": 1}',
        id: '123',
        retry: 5000
      }]);
    });

    test('여러 데이터 필드를 연결해야 함', () => {
      const result = parser.parse(
        'data: first line\n' +
        'data: second line\n' +
        'data: third line\n\n'
      );

      expect(result).toEqual([{
        data: 'first line\nsecond line\nthird line'
      }]);
    });
  });

  describe('스트리밍 파싱', () => {
    test('부분적인 데이터를 버퍼링해야 함', () => {
      const chunk1 = 'data: hello';
      const chunk2 = ' world\n\n';

      const result1 = parser.parseChunk(chunk1);
      expect(result1).toEqual([]);

      const result2 = parser.parseChunk(chunk2);
      expect(result2).toEqual([{
        data: 'hello world'
      }]);
    });

    test('여러 이벤트를 포함한 청크를 처리해야 함', () => {
      const chunk = 
        'event: first\ndata: 1\n\n' +
        'event: second\ndata: 2\n\n' +
        'event: third\ndata: 3';

      const result = parser.parseChunk(chunk);
      
      expect(result).toEqual([
        { event: 'first', data: '1' },
        { event: 'second', data: '2' }
      ]);

      // 마지막 불완전한 이벤트는 버퍼에 남아있어야 함
      expect(parser.getBuffer()).toBe('event: third\ndata: 3');
    });

    test('줄바꿈이 분할된 경우를 처리해야 함', () => {
      const chunks = [
        'data: test\n',
        '\n',
        'event: complete\n\n'
      ];

      const results = chunks.map(chunk => parser.parseChunk(chunk));
      
      expect(results[0]).toEqual([]);
      expect(results[1]).toEqual([{ data: 'test' }]);
      expect(results[2]).toEqual([{ event: 'complete' }]);
    });
  });

  describe('엣지 케이스', () => {
    test('빈 줄을 무시해야 함', () => {
      const result = parser.parse('\n\n\ndata: test\n\n\n');
      
      expect(result).toEqual([{
        data: 'test'
      }]);
    });

    test('콜론이 없는 필드를 처리해야 함', () => {
      const result = parser.parse(
        'data\n' +
        'event: ping\n\n'
      );

      expect(result).toEqual([{
        event: 'ping'
      }]);
    });

    test('주석을 무시해야 함', () => {
      const result = parser.parse(
        ': this is a comment\n' +
        'data: actual data\n' +
        ': another comment\n\n'
      );

      expect(result).toEqual([{
        data: 'actual data'
      }]);
    });

    test('BOM을 제거해야 함', () => {
      const bomString = '\uFEFFdata: test\n\n';
      const result = parser.parse(bomString);

      expect(result).toEqual([{
        data: 'test'
      }]);
    });
  });

  describe('상태 관리', () => {
    test('파서를 리셋할 수 있어야 함', () => {
      parser.parseChunk('data: incomplete');
      expect(parser.getBuffer()).toBe('data: incomplete');

      parser.reset();
      expect(parser.getBuffer()).toBe('');
    });

    test('retry 값을 추적해야 함', () => {
      parser.parse('retry: 3000\n\n');
      expect(parser.getRetryTime()).toBe(3000);

      parser.parse('retry: invalid\n\n');
      expect(parser.getRetryTime()).toBe(3000); // 변경되지 않음
    });

    test('마지막 이벤트 ID를 추적해야 함', () => {
      parser.parse('id: 123\ndata: test\n\n');
      expect(parser.getLastEventId()).toBe('123');

      parser.parse('data: no id\n\n');
      expect(parser.getLastEventId()).toBe('123'); // 유지됨
    });
  });
});
```

### 통합 파서 테스트

```javascript
// __tests__/SSEStreamParser.integration.test.js
import { Readable } from 'stream';
import { SSEStreamParser } from '../src/utils/SSEStreamParser';

describe('SSEStreamParser Integration', () => {
  test('실제 스트림 데이터를 파싱해야 함', async () => {
    const events = [];
    const parser = new SSEStreamParser();

    parser.on('event', event => events.push(event));

    // 스트림 생성
    const stream = new Readable({
      read() {}
    });

    stream.pipe(parser);

    // 데이터 전송
    stream.push('event: start\n');
    stream.push('data: {"status": "initializing"}\n\n');
    
    await new Promise(resolve => setTimeout(resolve, 10));
    
    stream.push('event: progress\n');
    stream.push('data: {"percent": 50}\n\n');
    
    stream.push('event: complete\n');
    stream.push('data: {"status": "done"}\n\n');
    
    stream.push(null); // 스트림 종료

    await new Promise(resolve => parser.on('finish', resolve));

    expect(events).toEqual([
      { event: 'start', data: { status: 'initializing' } },
      { event: 'progress', data: { percent: 50 } },
      { event: 'complete', data: { status: 'done' } }
    ]);
  });

  test('네트워크 지연을 시뮬레이션해야 함', async () => {
    const events = [];
    const parser = new SSEStreamParser();

    parser.on('event', event => events.push(event));

    const simulateNetworkStream = async (parser) => {
      const chunks = [
        'ev',
        'ent: update\nda',
        'ta: {"id": 1, "mes',
        'sage": "Hello ',
        'World"}\n\n'
      ];

      for (const chunk of chunks) {
        parser.write(chunk);
        await new Promise(resolve => setTimeout(resolve, 50));
      }

      parser.end();
    };

    await simulateNetworkStream(parser);

    expect(events).toEqual([
      { 
        event: 'update', 
        data: { id: 1, message: 'Hello World' } 
      }
    ]);
  });
});
```

## Storybook SSE 테스팅

### SSE 컴포넌트 스토리

```javascript
// stories/StreamingComponents.stories.js
import React from 'react';
import { StreamingChat } from '../src/components/StreamingChat';
import { SSEProvider } from '../src/contexts/SSEContext';
import { MockSSEServer } from './mocks/MockSSEServer';

export default {
  title: 'SSE/StreamingChat',
  component: StreamingChat,
  decorators: [
    (Story) => (
      <SSEProvider>
        <Story />
      </SSEProvider>
    )
  ]
};

// 모의 SSE 서버 설정
const mockServer = new MockSSEServer();

export const Default = {
  args: {
    endpoint: mockServer.endpoint
  },
  play: async ({ canvasElement }) => {
    // 자동 시나리오 실행
    await mockServer.start();
    await mockServer.simulateTypingMessage('Hello from Storybook!', 50);
  }
};

export const WithErrors = {
  args: {
    endpoint: mockServer.endpoint
  },
  play: async () => {
    await mockServer.start();
    await mockServer.simulateError('Connection lost');
    await new Promise(resolve => setTimeout(resolve, 2000));
    await mockServer.simulateReconnection();
  }
};

export const StreamingResponse = {
  args: {
    endpoint: mockServer.endpoint
  },
  play: async () => {
    await mockServer.start();
    
    const longMessage = `This is a very long message that will be streamed 
    word by word to demonstrate the streaming capability of our SSE implementation. 
    Watch as each word appears one by one!`;
    
    await mockServer.streamMessage(longMessage, {
      chunkSize: 'word',
      delay: 100
    });
  }
};

export const MultipleClients = {
  render: () => (
    <div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr', gap: '20px' }}>
      <StreamingChat clientId="user1" />
      <StreamingChat clientId="user2" />
    </div>
  ),
  play: async () => {
    await mockServer.start();
    
    // 다중 클라이언트 시뮬레이션
    await mockServer.sendToClient('user1', { message: 'Hello User 1!' });
    await new Promise(resolve => setTimeout(resolve, 1000));
    await mockServer.sendToClient('user2', { message: 'Hello User 2!' });
    await new Promise(resolve => setTimeout(resolve, 1000));
    await mockServer.broadcast({ message: 'Hello Everyone!' });
  }
};
```

### Mock SSE 서버

```javascript
// stories/mocks/MockSSEServer.js
export class MockSSEServer {
  constructor() {
    this.clients = new Map();
    this.endpoint = '/mock-sse';
    this.messageQueue = [];
    this.isRunning = false;
  }

  start() {
    this.isRunning = true;
    this.setupInterceptor();
    return Promise.resolve();
  }

  stop() {
    this.isRunning = false;
    this.clients.clear();
  }

  setupInterceptor() {
    // Storybook에서 MSW(Mock Service Worker) 사용
    if (typeof window !== 'undefined' && window.msw) {
      const { rest } = window.msw;
      
      window.msw.worker.use(
        rest.get(this.endpoint, (req, res, ctx) => {
          const stream = new ReadableStream({
            start: (controller) => {
              const clientId = req.url.searchParams.get('clientId') || 'default';
              
              this.clients.set(clientId, {
                controller,
                encoder: new TextEncoder()
              });

              // 초기 연결 메시지
              this.sendToController(controller, {
                event: 'connected',
                data: { clientId, timestamp: Date.now() }
              });

              // 대기 중인 메시지 처리
              this.processMessageQueue(clientId);
            },
            cancel: () => {
              const clientId = req.url.searchParams.get('clientId') || 'default';
              this.clients.delete(clientId);
            }
          });

          return res(
            ctx.status(200),
            ctx.set({
              'Content-Type': 'text/event-stream',
              'Cache-Control': 'no-cache',
              'Connection': 'keep-alive'
            }),
            ctx.body(stream)
          );
        })
      );
    }
  }

  sendToController(controller, { event, data }) {
    const encoder = new TextEncoder();
    const message = this.formatSSEMessage(event, data);
    controller.enqueue(encoder.encode(message));
  }

  formatSSEMessage(event, data) {
    return `event: ${event}\ndata: ${JSON.stringify(data)}\n\n`;
  }

  async simulateTypingMessage(message, delay = 50) {
    const words = message.split(' ');
    let accumulated = '';

    for (const word of words) {
      accumulated += (accumulated ? ' ' : '') + word;
      this.broadcast({
        event: 'message',
        data: {
          content: accumulated,
          isTyping: true,
          timestamp: Date.now()
        }
      });
      await new Promise(resolve => setTimeout(resolve, delay));
    }

    // 타이핑 완료
    this.broadcast({
      event: 'message',
      data: {
        content: accumulated,
        isTyping: false,
        timestamp: Date.now()
      }
    });
  }

  async streamMessage(message, options = {}) {
    const { chunkSize = 'char', delay = 50 } = options;
    
    let chunks;
    if (chunkSize === 'word') {
      chunks = message.split(' ').map((word, i, arr) => 
        i === 0 ? word : ' ' + word
      );
    } else {
      chunks = message.split('');
    }

    let accumulated = '';
    for (const chunk of chunks) {
      accumulated += chunk;
      this.broadcast({
        event: 'stream',
        data: {
          content: accumulated,
          done: false
        }
      });
      await new Promise(resolve => setTimeout(resolve, delay));
    }

    this.broadcast({
      event: 'stream',
      data: {
        content: accumulated,
        done: true
      }
    });
  }

  simulateError(errorMessage) {
    this.broadcast({
      event: 'error',
      data: {
        error: errorMessage,
        timestamp: Date.now()
      }
    });

    // 모든 연결 종료
    this.clients.forEach(client => {
      client.controller.close();
    });
    this.clients.clear();
  }

  async simulateReconnection() {
    // 재연결 이벤트 시뮬레이션
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    this.broadcast({
      event: 'reconnected',
      data: {
        message: 'Connection restored',
        timestamp: Date.now()
      }
    });
  }

  broadcast(message) {
    this.clients.forEach(client => {
      this.sendToController(client.controller, message);
    });
  }

  sendToClient(clientId, data) {
    const client = this.clients.get(clientId);
    if (client) {
      this.sendToController(client.controller, {
        event: 'message',
        data
      });
    } else {
      // 클라이언트가 아직 연결되지 않은 경우 큐에 저장
      this.messageQueue.push({ clientId, data });
    }
  }

  processMessageQueue(clientId) {
    const messages = this.messageQueue.filter(m => m.clientId === clientId);
    messages.forEach(({ data }) => {
      this.sendToClient(clientId, data);
    });
    this.messageQueue = this.messageQueue.filter(m => m.clientId !== clientId);
  }
}
```

### 상호작용 테스트

```javascript
// stories/InteractiveSSE.stories.js
import React from 'react';
import { within, userEvent, waitFor } from '@storybook/testing-library';
import { expect } from '@storybook/jest';
import { ChatInterface } from '../src/components/ChatInterface';

export default {
  title: 'SSE/Interactive',
  component: ChatInterface
};

export const UserInteraction = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    const user = userEvent.setup();

    // 연결 버튼 찾기 및 클릭
    const connectButton = await canvas.findByRole('button', { name: /connect/i });
    await user.click(connectButton);

    // 연결 상태 확인
    await waitFor(() => {
      expect(canvas.getByText(/connected/i)).toBeInTheDocument();
    });

    // 메시지 입력
    const input = canvas.getByRole('textbox', { name: /message/i });
    await user.type(input, 'Hello SSE!');

    // 전송 버튼 클릭
    const sendButton = canvas.getByRole('button', { name: /send/i });
    await user.click(sendButton);

    // 응답 대기
    await waitFor(() => {
      expect(canvas.getByText(/Hello SSE!/)).toBeInTheDocument();
    }, { timeout: 5000 });

    // 스트리밍 응답 확인
    await waitFor(() => {
      const response = canvas.getByTestId('ai-response');
      expect(response).toHaveTextContent(/responding/i);
    });
  }
};

export const ErrorRecovery = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    
    // 에러 시뮬레이션 버튼
    const errorButton = await canvas.findByRole('button', { 
      name: /simulate error/i 
    });
    await userEvent.click(errorButton);

    // 에러 메시지 확인
    await waitFor(() => {
      expect(canvas.getByRole('alert')).toHaveTextContent(/connection lost/i);
    });

    // 자동 재연결 확인
    await waitFor(() => {
      expect(canvas.getByText(/reconnecting/i)).toBeInTheDocument();
    }, { timeout: 10000 });

    // 재연결 성공 확인
    await waitFor(() => {
      expect(canvas.getByText(/connected/i)).toBeInTheDocument();
    }, { timeout: 15000 });
  }
};
```

## 마무리

이 장에서는 SSE 애플리케이션의 포괄적인 테스트 전략을 다루었습니다. EventSource 모킹, Jest와 React Testing Library를 사용한 컴포넌트 테스트, 파서 로직 테스트, 그리고 Storybook을 활용한 시각적 테스트까지 모든 측면을 포함했습니다.

다음 장에서는 실제 프로덕션 환경에서 사용할 수 있는 완성된 예제들을 살펴보겠습니다.

***
**[← 이전: Part 9 - 보안과 인프라](./part9-security-infrastructure.md)** | **[목차](./README.md)** | **[다음: Part 11 - 프로덕션 예제 →](./part11-production-examples.md)**

***