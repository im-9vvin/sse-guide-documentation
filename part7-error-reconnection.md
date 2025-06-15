***
**[← 이전: Part 6 - 브라우저에서의 불완전한 데이터 처리](./part6-browser-data-processing.md)** | **[목차](./README.md)** | **[다음: Part 8 - 성능 특성과 최적화 →](./part8-performance-optimization.md)**

***

# Part 7: 에러 처리와 재연결 메커니즘

## 8장. 에러 처리와 재연결 메커니즘

SSE의 가장 큰 장점 중 하나는 자동 재연결 기능입니다. 하지만 프로덕션 환경에서는 더 정교한 에러 처리와 재연결 전략이 필요합니다.

### 8.1 자동 재연결 동작

#### 8.1.1 브라우저의 기본 재연결 동작

```typescript
// EventSource의 기본 재연결 동작
const eventSource = new EventSource('/api/stream');

eventSource.onopen = () => {
  console.log('연결 성공');
};

eventSource.onerror = (event) => {
  console.log('에러 발생, 자동 재연결 시도');
  // 브라우저가 자동으로:
  // 1. 3초 후 재연결 시도 (기본값)
  // 2. 실패 시 다시 시도
  // 3. Last-Event-ID 헤더 포함
};
```#### 8.1.2 재연결 간격 제어

```typescript
// 서버에서 재연결 간격 제어
class SSEResponseWithRetry {
  constructor(private response: Response) {}
  
  setRetryInterval(milliseconds: number) {
    // retry 필드로 재연결 간격 설정
    this.response.write(`retry: ${milliseconds}\n\n`);
  }
  
  sendWithRetry(data: any, retryMs: number) {
    this.response.write(`retry: ${retryMs}\n`);
    this.response.write(`data: ${JSON.stringify(data)}\n\n`);
  }
}

// 동적 재연결 간격
class DynamicRetryStrategy {
  private baseRetry = 1000;
  private maxRetry = 30000;
  
  calculateRetry(errorCount: number): number {
    // 지수 백오프
    const retry = Math.min(
      this.baseRetry * Math.pow(2, errorCount),
      this.maxRetry
    );
    return retry;
  }
}```

#### 8.1.3 Last-Event-ID를 활용한 재연결

```typescript
// 서버 측: 이벤트 ID 추적
class EventIDTracker {
  private lastEventId: string = '0';
  private eventHistory: Map<string, any> = new Map();
  private maxHistorySize = 1000;
  
  generateId(): string {
    this.lastEventId = Date.now().toString();
    return this.lastEventId;
  }
  
  sendWithId(response: Response, data: any) {
    const eventId = this.generateId();
    
    // 히스토리에 저장
    this.eventHistory.set(eventId, data);
    if (this.eventHistory.size > this.maxHistorySize) {
      const firstKey = this.eventHistory.keys().next().value;
      this.eventHistory.delete(firstKey);
    }
    
    response.write(`id: ${eventId}\n`);
    response.write(`data: ${JSON.stringify(data)}\n\n`);
  }
  
  getEventsSince(lastEventId: string): any[] {
    const events: any[] = [];
    let found = false;    
    for (const [id, data] of this.eventHistory) {
      if (found) {
        events.push({ id, data });
      }
      if (id === lastEventId) {
        found = true;
      }
    }
    
    return events;
  }
}

// 서버 엔드포인트
app.get('/api/stream', (req, res) => {
  const lastEventId = req.headers['last-event-id'];
  
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });
  
  if (lastEventId) {
    // 놓친 이벤트 재전송
    const missedEvents = eventTracker.getEventsSince(lastEventId);
    for (const event of missedEvents) {
      res.write(`id: ${event.id}\n`);
      res.write(`data: ${JSON.stringify(event.data)}\n\n`);
    }
  }
  
  // 정상 스트리밍 시작
});
```### 8.2 지수 백오프를 활용한 재연결 구현

#### 8.2.1 커스텀 재연결 로직

```typescript
class ExponentialBackoffReconnection {
  private baseDelay = 1000;
  private maxDelay = 60000;
  private factor = 2;
  private jitter = 0.3;
  private attempt = 0;
  
  getNextDelay(): number {
    const exponentialDelay = Math.min(
      this.baseDelay * Math.pow(this.factor, this.attempt),
      this.maxDelay
    );
    
    // 지터 추가 (동시 재연결 방지)
    const jitterValue = exponentialDelay * this.jitter * Math.random();
    const delay = exponentialDelay + jitterValue;
    
    this.attempt++;
    return Math.round(delay);
  }
  
  reset() {
    this.attempt = 0;
  }
  
  // 재연결 성공률 기반 조정
  adjustBasedOnSuccess(successRate: number) {
    if (successRate < 0.5) {
      // 성공률이 낮으면 기본 지연 증가
      this.baseDelay = Math.min(this.baseDelay * 1.5, 5000);
    } else if (successRate > 0.8) {
      // 성공률이 높으면 기본 지연 감소
      this.baseDelay = Math.max(this.baseDelay * 0.8, 1000);
    }
  }
}```

#### 8.2.2 회로 차단기 패턴

```typescript
enum CircuitState {
  Closed = 'closed',  // 정상 작동
  Open = 'open',      // 차단됨
  HalfOpen = 'half-open' // 테스트 중
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.Closed;
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime?: number;
  
  private readonly failureThreshold = 5;
  private readonly successThreshold = 3;
  private readonly timeout = 60000; // 1분
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.Open) {
      if (this.shouldAttemptReset()) {
        this.state = CircuitState.HalfOpen;
      } else {
        throw new Error('Circuit breaker is open');
      }
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }  
  private onSuccess() {
    this.failureCount = 0;
    
    if (this.state === CircuitState.HalfOpen) {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.state = CircuitState.Closed;
        this.successCount = 0;
      }
    }
  }
  
  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.state === CircuitState.HalfOpen) {
      this.state = CircuitState.Open;
      this.successCount = 0;
    } else if (this.failureCount >= this.failureThreshold) {
      this.state = CircuitState.Open;
    }
  }
  
  private shouldAttemptReset(): boolean {
    return (
      this.lastFailureTime &&
      Date.now() - this.lastFailureTime >= this.timeout
    );
  }
  
  getState(): CircuitState {
    return this.state;
  }
}```

#### 8.2.3 적응형 재연결 전략

```typescript
class AdaptiveReconnectionStrategy {
  private metrics = {
    totalAttempts: 0,
    successfulConnections: 0,
    averageConnectionDuration: 0,
    lastConnectionDuration: 0,
    networkQuality: 1.0 // 0-1 scale
  };
  
  private backoff = new ExponentialBackoffReconnection();
  private circuitBreaker = new CircuitBreaker();
  
  async connect(url: string): Promise<EventSource> {
    return this.circuitBreaker.execute(async () => {
      const startTime = Date.now();
      const eventSource = new EventSource(url);
      
      return new Promise<EventSource>((resolve, reject) => {
        eventSource.onopen = () => {
          this.onConnectionSuccess(startTime);
          resolve(eventSource);
        };
        
        eventSource.onerror = () => {
          this.onConnectionFailure();
          reject(new Error('Connection failed'));
        };
      });
    });
  }  
  private onConnectionSuccess(startTime: number) {
    this.metrics.totalAttempts++;
    this.metrics.successfulConnections++;
    this.metrics.lastConnectionDuration = Date.now() - startTime;
    
    // 평균 연결 시간 업데이트
    this.metrics.averageConnectionDuration = 
      (this.metrics.averageConnectionDuration * (this.metrics.successfulConnections - 1) + 
       this.metrics.lastConnectionDuration) / this.metrics.successfulConnections;
    
    // 네트워크 품질 계산
    this.updateNetworkQuality();
    
    // 백오프 리셋
    this.backoff.reset();
  }
  
  private onConnectionFailure() {
    this.metrics.totalAttempts++;
    this.updateNetworkQuality();
  }
  
  private updateNetworkQuality() {
    const successRate = this.metrics.successfulConnections / this.metrics.totalAttempts;
    const connectionSpeed = 1000 / Math.max(this.metrics.averageConnectionDuration, 1);
    
    this.metrics.networkQuality = (successRate * 0.7 + Math.min(connectionSpeed, 1) * 0.3);
  }
  
  getNextRetryDelay(): number {
    const baseDelay = this.backoff.getNextDelay();
    
    // 네트워크 품질에 따라 조정
    return Math.round(baseDelay / this.metrics.networkQuality);
  }
}```

### 8.3 OpenAI API 에러 처리

#### 8.3.1 OpenAI 특화 에러 처리

```typescript
interface OpenAIError {
  error: {
    message: string;
    type: string;
    param?: string;
    code?: string;
  };
}

class OpenAISSEErrorHandler {
  private readonly retryableErrors = new Set([
    'rate_limit_exceeded',
    'timeout',
    'server_error',
    'service_unavailable'
  ]);
  
  isRetryable(error: OpenAIError): boolean {
    return this.retryableErrors.has(error.error.type);
  }
  
  getRetryDelay(error: OpenAIError): number {
    switch (error.error.type) {
      case 'rate_limit_exceeded':
        // Rate limit 헤더에서 재시도 시간 추출
        return this.parseRateLimitHeaders() || 60000;
      case 'timeout':
        return 5000;
      case 'server_error':
      case 'service_unavailable':
        return 30000;
      default:
        return 10000;
    }
  }  
  private parseRateLimitHeaders(): number | null {
    // 실제 구현에서는 response headers 참조
    // X-RateLimit-Reset-Requests
    // X-RateLimit-Reset-Tokens
    return null;
  }
  
  async handleStreamError(error: any): Promise<void> {
    if (error.response) {
      const errorData: OpenAIError = await error.response.json();
      
      if (this.isRetryable(errorData)) {
        const delay = this.getRetryDelay(errorData);
        console.log(`Retrying after ${delay}ms due to ${errorData.error.type}`);
        await new Promise(resolve => setTimeout(resolve, delay));
      } else {
        throw new Error(`Non-retryable error: ${errorData.error.message}`);
      }
    } else {
      // 네트워크 에러
      throw error;
    }
  }
}
```

#### 8.3.2 스트림 중 에러 복구

```typescript
class StreamErrorRecovery {
  private buffer: string[] = [];
  private lastSuccessfulPosition = 0;  
  async *recoveryStream(
    createStream: () => Promise<AsyncIterable<string>>,
    maxRetries = 3
  ): AsyncGenerator<string> {
    let retries = 0;
    
    while (retries < maxRetries) {
      try {
        const stream = await createStream();
        
        for await (const chunk of stream) {
          this.buffer.push(chunk);
          this.lastSuccessfulPosition = this.buffer.length;
          yield chunk;
        }
        
        // 성공적으로 완료
        return;
      } catch (error) {
        retries++;
        console.error(`Stream error (attempt ${retries}/${maxRetries}):`, error);
        
        if (retries < maxRetries) {
          // 버퍼에서 이미 전송된 내용 제거
          yield* this.resumeFromBuffer();
        } else {
          throw error;
        }
      }
    }
  }  
  private *resumeFromBuffer(): Generator<string> {
    // 중복 전송 방지를 위해 새로운 마커 추가
    yield '\n[RESUMED]\n';
    
    // 버퍼의 마지막 몇 개 항목만 재전송
    const replayCount = Math.min(5, this.buffer.length);
    const startIndex = Math.max(0, this.buffer.length - replayCount);
    
    for (let i = startIndex; i < this.buffer.length; i++) {
      yield this.buffer[i];
    }
  }
  
  clearBuffer() {
    this.buffer = [];
    this.lastSuccessfulPosition = 0;
  }
}
```

### 8.4 강력한 클라이언트 구현

#### 8.4.1 완전한 SSE 클라이언트

```typescript
class RobustSSEClient {
  private eventSource?: EventSource;
  private url: string;
  private reconnectStrategy: AdaptiveReconnectionStrategy;
  private errorHandler: OpenAISSEErrorHandler;
  private listeners: Map<string, Set<Function>> = new Map();
  private state: 'disconnected' | 'connecting' | 'connected' = 'disconnected';
  private reconnectTimer?: NodeJS.Timeout;  
  constructor(url: string, options: {
    reconnectStrategy?: AdaptiveReconnectionStrategy;
    errorHandler?: OpenAISSEErrorHandler;
  } = {}) {
    this.url = url;
    this.reconnectStrategy = options.reconnectStrategy || new AdaptiveReconnectionStrategy();
    this.errorHandler = options.errorHandler || new OpenAISSEErrorHandler();
  }
  
  async connect(): Promise<void> {
    if (this.state !== 'disconnected') {
      return;
    }
    
    this.state = 'connecting';
    
    try {
      this.eventSource = await this.reconnectStrategy.connect(this.url);
      this.setupEventHandlers();
      this.state = 'connected';
      this.emit('connect', {});
    } catch (error) {
      this.state = 'disconnected';
      this.scheduleReconnect();
      throw error;
    }
  }
  
  private setupEventHandlers() {
    if (!this.eventSource) return;    
    this.eventSource.onopen = () => {
      console.log('SSE connection opened');
      this.emit('open', {});
    };
    
    this.eventSource.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        this.emit('message', data);
      } catch (e) {
        // 일반 텍스트 메시지
        this.emit('message', event.data);
      }
    };
    
    this.eventSource.onerror = async (event) => {
      console.error('SSE error:', event);
      this.state = 'disconnected';
      
      if (this.eventSource?.readyState === EventSource.CLOSED) {
        this.emit('close', {});
        this.scheduleReconnect();
      } else {
        // 일시적 오류
        this.emit('error', { temporary: true });
      }
    };
    
    // 커스텀 이벤트 리스너 등록
    for (const [eventType, handlers] of this.listeners) {
      if (eventType !== 'message' && eventType !== 'open' && 
          eventType !== 'error' && eventType !== 'close') {
        this.eventSource.addEventListener(eventType, (event: any) => {
          handlers.forEach(handler => handler(event.data));
        });
      }
    }
  }  
  private scheduleReconnect() {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
    }
    
    const delay = this.reconnectStrategy.getNextRetryDelay();
    console.log(`Scheduling reconnect in ${delay}ms`);
    
    this.reconnectTimer = setTimeout(() => {
      this.connect().catch(error => {
        console.error('Reconnection failed:', error);
      });
    }, delay);
  }
  
  on(event: string, handler: Function) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);
    
    // 이미 연결된 경우 즉시 리스너 등록
    if (this.eventSource && this.state === 'connected' &&
        event !== 'message' && event !== 'open' && 
        event !== 'error' && event !== 'close') {
      this.eventSource.addEventListener(event, (e: any) => handler(e.data));
    }
  }
  
  off(event: string, handler: Function) {
    this.listeners.get(event)?.delete(handler);
  }  
  private emit(event: string, data: any) {
    this.listeners.get(event)?.forEach(handler => handler(data));
  }
  
  disconnect() {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = undefined;
    }
    
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = undefined;
    }
    
    this.state = 'disconnected';
    this.emit('disconnect', {});
  }
  
  getState() {
    return this.state;
  }
}
```

#### 8.4.2 React Hook 구현

```typescript
import { useEffect, useRef, useState, useCallback } from 'react';

interface UseRobustSSEOptions {
  url: string;
  reconnect?: boolean;
  reconnectAttempts?: number;
  reconnectInterval?: number;}

function useRobustSSE<T = any>(options: UseRobustSSEOptions) {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [state, setState] = useState<'disconnected' | 'connecting' | 'connected'>('disconnected');
  const clientRef = useRef<RobustSSEClient | null>(null);
  
  const connect = useCallback(() => {
    if (clientRef.current) {
      clientRef.current.disconnect();
    }
    
    const client = new RobustSSEClient(options.url);
    clientRef.current = client;
    
    client.on('connect', () => setState('connected'));
    client.on('disconnect', () => setState('disconnected'));
    client.on('message', (data: T) => setData(data));
    client.on('error', (error: any) => setError(error));
    
    setState('connecting');
    client.connect().catch(err => {
      setState('disconnected');
      setError(err);
    });
  }, [options.url]);
  
  const disconnect = useCallback(() => {
    clientRef.current?.disconnect();
  }, []);  
  useEffect(() => {
    connect();
    return () => disconnect();
  }, [connect, disconnect]);
  
  return {
    data,
    error,
    state,
    reconnect: connect,
    disconnect
  };
}

// 사용 예제
function ChatComponent() {
  const { data, error, state, reconnect } = useRobustSSE<ChatMessage>({
    url: '/api/chat/stream',
    reconnect: true,
    reconnectAttempts: 5,
    reconnectInterval: 1000
  });
  
  if (state === 'connecting') {
    return <div>연결 중...</div>;
  }
  
  if (error) {
    return (
      <div>
        <p>에러: {error.message}</p>
        <button onClick={reconnect}>재연결</button>
      </div>
    );
  }
  
  return <div>{data?.content}</div>;
}
```

***

**다음 장 예고**: [Part 8: 성능 특성과 최적화](./part8-performance-optimization.md)에서는 SSE의 성능 특성과 대규모 서비스를 위한 최적화 전략을 다루겠습니다.

***
**[← 이전: Part 6 - 브라우저에서의 불완전한 데이터 처리](./part6-browser-data-processing.md)** | **[목차](./README.md)** | **[다음: Part 8 - 성능 특성과 최적화 →](./part8-performance-optimization.md)**

***