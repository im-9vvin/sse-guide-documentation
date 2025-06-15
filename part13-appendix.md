***
**[← 이전: Part 12 - 실제 기업 사례](./part12-real-world-cases.md)** | **[목차](./README.md)**

***

# Part 13: 부록

## 부록 A: 브라우저 호환성

### EventSource API 지원 현황

| 브라우저 | 최소 버전 | 참고사항 |
|---------|----------|---------|
| Chrome | 6+ | 완전 지원 |
| Firefox | 6+ | 완전 지원 |
| Safari | 5+ | 완전 지원 |
| Edge | 79+ | Chromium 기반 |
| IE | 미지원 | 폴리필 필요 |
| Opera | 11+ | 완전 지원 |
| iOS Safari | 4.2+ | 완전 지원 |
| Android Browser | 4.4+ | 완전 지원 |

### 폴리필 구현

```javascript
// EventSource 폴리필 (IE 지원)
if (!window.EventSource) {
  window.EventSource = function(url, options) {
    const self = this;
    self.url = url;
    self.readyState = 0; // CONNECTING
    self.withCredentials = options?.withCredentials || false;
    
    const xhr = new XMLHttpRequest();
    let position = 0;
    let buffer = '';
    
    xhr.open('GET', url, true);
    xhr.setRequestHeader('Accept', 'text/event-stream');
    xhr.setRequestHeader('Cache-Control', 'no-cache');
    
    if (self.withCredentials) {
      xhr.withCredentials = true;
    }
    
    xhr.onreadystatechange = function() {
      if (xhr.readyState === 3 || xhr.readyState === 4) {
        const newData = xhr.responseText.substring(position);
        position = xhr.responseText.length;
        buffer += newData;
        
        const messages = buffer.split('\n\n');
        buffer = messages.pop() || '';
        
        messages.forEach(message => {
          if (message.trim()) {
            self.dispatchEvent(parseMessage(message));
          }
        });
      }
      
      if (xhr.readyState === 4) {
        self.readyState = 2; // CLOSED
        self.dispatchEvent(new Event('error'));
      }
    };
    
    xhr.send();
    
    self.close = function() {
      xhr.abort();
      self.readyState = 2;
    };
  };
}
```

## 부록 B: 참고 문서

### 공식 명세

1. **W3C Server-Sent Events**
   - https://www.w3.org/TR/eventsource/
   - EventSource API의 공식 명세

2. **WHATWG HTML Living Standard - Server-sent events**
   - https://html.spec.whatwg.org/multipage/server-sent-events.html
   - 최신 HTML 표준의 SSE 섹션

3. **MDN Web Docs - Server-sent events**
   - https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
   - 포괄적인 가이드와 예제

### 도구 및 라이브러리

1. **EventSource Polyfill**
   - https://github.com/EventSource/eventsource
   - Node.js와 구형 브라우저 지원

2. **SSE.js**
   - https://github.com/mpetazzoni/sse.js
   - 향상된 기능을 제공하는 EventSource 래퍼

3. **Mercure**
   - https://mercure.rocks/
   - SSE 기반 실시간 API 게이트웨이

### 관련 기술

1. **WebSocket Protocol**
   - RFC 6455: https://tools.ietf.org/html/rfc6455
   - 양방향 통신 프로토콜

2. **HTTP/2 Server Push**
   - RFC 7540: https://tools.ietf.org/html/rfc7540#section-8.2
   - 서버 주도 리소스 전송

3. **WebTransport**
   - https://w3c.github.io/webtransport/
   - 차세대 양방향 통신 API

## 부록 C: 용어집

### 기본 용어

**EventSource**: 서버-전송 이벤트를 수신하기 위한 브라우저 API

**SSE (Server-Sent Events)**: 서버에서 클라이언트로 단방향 실시간 이벤트를 전송하는 웹 기술

**Event Stream**: SSE 프로토콜로 전송되는 이벤트의 연속적인 흐름

**Long Polling**: 서버가 응답을 보류하다가 이벤트 발생 시 응답하는 기법

### 기술 용어

**TTFB (Time To First Byte)**: 요청 후 첫 번째 바이트를 받기까지의 시간

**TTFT (Time To First Token)**: 스트리밍 응답에서 첫 번째 의미 있는 토큰까지의 시간

**Chunked Transfer Encoding**: HTTP에서 데이터를 청크 단위로 전송하는 방식

**Keep-Alive**: HTTP 연결을 재사용하여 여러 요청/응답을 처리하는 메커니즘

### 패턴 용어

**Backpressure**: 소비자가 생산자의 속도를 제어하는 메커니즘

**Circuit Breaker**: 장애 시 시스템을 보호하기 위한 패턴

**Exponential Backoff**: 재시도 간격을 지수적으로 증가시키는 전략

**Graceful Degradation**: 일부 기능 실패 시에도 핵심 기능을 유지하는 설계

## 부록 D: 코드 스니펫 모음

### 기본 SSE 서버 (Node.js)

```javascript
// 최소 SSE 서버
const http = require('http');

http.createServer((req, res) => {
  if (req.url === '/events') {
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'Access-Control-Allow-Origin': '*'
    });

    const sendEvent = (data) => {
      res.write(`data: ${JSON.stringify(data)}\n\n`);
    };

    const interval = setInterval(() => {
      sendEvent({ time: new Date().toISOString() });
    }, 1000);

    req.on('close', () => {
      clearInterval(interval);
    });
  }
}).listen(3000);
```

### 재사용 가능한 SSE 클라이언트

```typescript
// 타입 안전 SSE 클라이언트
class TypedEventSource<T extends Record<string, any>> {
  private eventSource: EventSource;
  private handlers = new Map<keyof T, Set<(data: any) => void>>();

  constructor(url: string, options?: EventSourceInit) {
    this.eventSource = new EventSource(url, options);
  }

  on<K extends keyof T>(event: K, handler: (data: T[K]) => void): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
      
      this.eventSource.addEventListener(event as string, (e) => {
        const data = JSON.parse(e.data);
        this.handlers.get(event)?.forEach(h => h(data));
      });
    }

    this.handlers.get(event)!.add(handler);

    return () => {
      this.handlers.get(event)?.delete(handler);
    };
  }

  close(): void {
    this.eventSource.close();
  }
}

// 사용 예
interface MyEvents {
  update: { id: string; status: string };
  notification: { message: string; level: 'info' | 'warning' | 'error' };
}

const events = new TypedEventSource<MyEvents>('/api/events');

events.on('update', (data) => {
  console.log(`Update ${data.id}: ${data.status}`);
});
```

### 서버 측 이벤트 브로드캐스터

```javascript
// 다중 클라이언트 브로드캐스터
class SSEBroadcaster {
  constructor() {
    this.clients = new Set();
  }

  addClient(res) {
    this.clients.add(res);
    
    res.on('close', () => {
      this.clients.delete(res);
    });
  }

  broadcast(event, data) {
    const message = event 
      ? `event: ${event}\ndata: ${JSON.stringify(data)}\n\n`
      : `data: ${JSON.stringify(data)}\n\n`;

    this.clients.forEach(client => {
      client.write(message);
    });
  }

  getClientCount() {
    return this.clients.size;
  }
}
```

### React SSE Hook

```typescript
// 범용 SSE Hook
function useEventSource<T = any>(
  url: string,
  options?: {
    withCredentials?: boolean;
    events?: string[];
    reconnect?: boolean;
    reconnectDelay?: number;
  }
) {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [status, setStatus] = useState<'connecting' | 'open' | 'closed'>('connecting');

  useEffect(() => {
    const eventSource = new EventSource(url, {
      withCredentials: options?.withCredentials
    });

    eventSource.onopen = () => {
      setStatus('open');
      setError(null);
    };

    eventSource.onerror = (e) => {
      setStatus('closed');
      setError(new Error('Connection failed'));
      
      if (options?.reconnect) {
        setTimeout(() => {
          setStatus('connecting');
        }, options.reconnectDelay || 5000);
      }
    };

    const handleMessage = (event: MessageEvent) => {
      try {
        const parsed = JSON.parse(event.data);
        setData(parsed);
      } catch (e) {
        setError(new Error('Failed to parse data'));
      }
    };

    if (options?.events) {
      options.events.forEach(eventName => {
        eventSource.addEventListener(eventName, handleMessage);
      });
    } else {
      eventSource.onmessage = handleMessage;
    }

    return () => {
      eventSource.close();
    };
  }, [url, options]);

  return { data, error, status };
}
```

## 마무리

이 가이드를 통해 Server-Sent Events(SSE)의 모든 측면을 다루었습니다. 기본 개념부터 고급 구현 패턴, 실제 기업들의 활용 사례까지 포괄적으로 살펴보았습니다.

SSE는 단순하면서도 강력한 기술로, 적절히 활용하면 뛰어난 실시간 웹 애플리케이션을 구축할 수 있습니다. 이 가이드가 여러분의 SSE 구현에 도움이 되기를 바랍니다.

Happy Streaming! 🚀

***
**[← 이전: Part 12 - 실제 기업 사례](./part12-real-world-cases.md)** | **[목차](./README.md)**

***