---
**[← 이전: Part 1 - SSE 개요와 표준 스펙](./part1-overview-and-specs.md)** | **[목차](./README.md)** | **[다음: Part 3 - 빅테크의 비공식 SSE 패턴 →](./part3-bigtech-patterns.md)**

---

# Part 2: 작동 원리와 청킹 메커니즘

## 3장. SSE의 작동 원리

### 3.1 SSE의 두 독립적인 레이어

SSE는 두 개의 독립적인 레이어로 구성되어 있습니다:

1. **전송 계층 (Transport Layer)**: HTTP 청킹을 통한 데이터 전송
2. **애플리케이션 계층 (Application Layer)**: SSE 프로토콜 형식

이 두 레이어는 서로 독립적으로 작동하며, 각각의 역할이 명확히 구분됩니다.

#### 3.1.1 전송 계층: HTTP 청킹

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Transfer-Encoding: chunked

1c\r\n
data: First message\n\n\r\n
1d\r\n
data: Second message\n\n\r\n
0\r\n
\r\n
```

#### 3.1.2 애플리케이션 계층: SSE 프로토콜

```
data: First message\n\n
data: Second message\n\n
event: custom-event\n
data: Custom event data\n\n
```

### 3.2 데이터 변환 과정

#### 3.2.1 서버에서 클라이언트까지의 여정

```javascript
// 1. 애플리케이션 데이터
const message = {
  type: "chat",
  content: "안녕하세요"
};

// 2. JSON 직렬화
const jsonString = JSON.stringify(message);
// '{"type":"chat","content":"안녕하세요"}'

// 3. SSE 형식으로 포맷팅
const sseMessage = `data: ${jsonString}\n\n`;
// 'data: {"type":"chat","content":"안녕하세요"}\n\n'

// 4. HTTP 청킹 (HTTP/1.1의 경우)
const chunk = Buffer.from(sseMessage);
const chunkSize = chunk.length.toString(16);
const httpChunk = `${chunkSize}\r\n${sseMessage}\r\n`;
// '37\r\ndata: {"type":"chat","content":"안녕하세요"}\n\n\r\n'

// 5. TCP/IP 패킷으로 전송
// 네트워크 레이어에서 처리
```

#### 3.2.2 클라이언트에서의 역변환

```javascript
// 1. TCP/IP 패킷 수신 및 조립
// 브라우저가 자동 처리

// 2. HTTP 청킹 해제
// 브라우저가 자동 처리

// 3. SSE 파싱
// EventSource API가 자동 처리

// 4. 이벤트 객체 생성
const event = new MessageEvent('message', {
  data: '{"type":"chat","content":"안녕하세요"}',
  origin: 'https://example.com',
  lastEventId: ''
});

// 5. 애플리케이션에서 사용
eventSource.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log(message); // {type: "chat", content: "안녕하세요"}
};
```

### 3.3 청킹의 역할과 위치

#### 3.3.1 청킹은 전송 계층의 메커니즘

청킹(Chunking)은 SSE 프로토콜의 일부가 아니라 HTTP 전송 메커니즘입니다:

```
┌─────────────────────┐
│  애플리케이션 데이터  │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│   SSE 프로토콜 형식  │  ← SSE 레이어
│  data: ...\n\n      │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│    HTTP 청킹        │  ← HTTP 레이어
│  1c\r\n...\r\n      │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│   TCP/IP 패킷       │  ← 네트워크 레이어
└─────────────────────┘
```

#### 3.3.2 HTTP 버전별 청킹 방식

**HTTP/1.1:**
```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: text/event-stream

1c\r\n
data: First chunk\n\n\r\n
1d\r\n
data: Second chunk\n\n\r\n
```

**HTTP/2:**
```
// HTTP/2는 바이너리 프레이밍 사용
HEADERS frame:
  :status: 200
  content-type: text/event-stream

DATA frame (not END_STREAM):
  data: First chunk\n\n

DATA frame (not END_STREAM):
  data: Second chunk\n\n
```

**HTTP/3:**
```
// HTTP/3는 QUIC 위에서 동작
HEADERS frame:
  :status: 200
  content-type: text/event-stream

DATA frame:
  data: First chunk\n\n

DATA frame:
  data: Second chunk\n\n
```

### 3.4 청킹이 필요한 이유

#### 3.4.1 무한 스트림 지원

SSE는 연결을 유지하면서 계속 데이터를 전송해야 합니다:

```javascript
// 서버 측 구현
const streamHandler = (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    // Content-Length를 지정하지 않음 - 무한 스트림
  });

  // 주기적으로 데이터 전송
  const interval = setInterval(() => {
    const data = {
      timestamp: new Date().toISOString(),
      value: Math.random()
    };
    
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  }, 1000);

  // 클라이언트 연결 종료 시 정리
  req.on('close', () => {
    clearInterval(interval);
  });
};
```

#### 3.4.2 실시간 데이터 전송

청킹을 통해 부분적인 데이터를 즉시 전송할 수 있습니다:

```javascript
// AI 응답 스트리밍 예제
async function* generateAIResponse(prompt) {
  const tokens = await getAITokens(prompt);
  
  for (const token of tokens) {
    yield token;
    // 각 토큰이 생성되는 즉시 전송
  }
}

// SSE로 스트리밍
app.get('/ai/stream', async (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache'
  });

  const generator = generateAIResponse(req.query.prompt);
  
  for await (const token of generator) {
    res.write(`data: ${JSON.stringify({ token })}\n\n`);
    // 각 청크가 즉시 클라이언트로 전송됨
  }
  
  res.write('event: complete\ndata: {"finished": true}\n\n');
  res.end();
});
```

### 3.5 HTTP 버전별 스트리밍 방식

#### 3.5.1 HTTP/1.1의 Transfer-Encoding: chunked

```javascript
// Node.js에서 HTTP/1.1 청킹 예제
const http = require('http');

http.createServer((req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Transfer-Encoding': 'chunked' // 명시적 설정 (자동 설정됨)
  });

  // res.write()를 호출할 때마다 자동으로 청킹됨
  res.write('data: First message\n\n');
  // 실제 전송: "14\r\ndata: First message\n\n\r\n"
  
  setTimeout(() => {
    res.write('data: Second message\n\n');
    // 실제 전송: "15\r\ndata: Second message\n\n\r\n"
  }, 1000);
}).listen(3000);
```

#### 3.5.2 HTTP/2의 바이너리 프레이밍

```javascript
// Node.js에서 HTTP/2 SSE 구현
const http2 = require('http2');
const fs = require('fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt')
});

server.on('stream', (stream, headers) => {
  if (headers[':path'] === '/stream') {
    stream.respond({
      ':status': 200,
      'content-type': 'text/event-stream'
    });

    // HTTP/2는 자동으로 DATA 프레임으로 분할
    stream.write('data: First message\n\n');
    
    setTimeout(() => {
      stream.write('data: Second message\n\n');
    }, 1000);
  }
});
```

#### 3.5.3 HTTP/3의 QUIC 스트림

```javascript
// HTTP/3는 아직 Node.js에서 실험적 지원
// 개념적 예제
const http3Server = createHTTP3Server({
  cert: 'server.crt',
  key: 'server.key'
});

http3Server.on('stream', (stream) => {
  stream.respond({
    ':status': 200,
    'content-type': 'text/event-stream'
  });

  // QUIC 스트림을 통해 전송
  stream.write('data: Message over QUIC\n\n');
});
```

### 3.6 SSE 데이터의 완전한 여정

#### 3.6.1 송신 측 (서버)

```javascript
// 1단계: 애플리케이션 데이터 생성
const appData = {
  id: 123,
  message: "실시간 업데이트",
  timestamp: Date.now()
};

// 2단계: JSON 직렬화
const jsonData = JSON.stringify(appData);
console.log('JSON:', jsonData);
// {"id":123,"message":"실시간 업데이트","timestamp":1704094800000}

// 3단계: SSE 포맷팅
const sseFormatted = `id: ${appData.id}\ndata: ${jsonData}\n\n`;
console.log('SSE 형식:', sseFormatted);
// id: 123\n
// data: {"id":123,"message":"실시간 업데이트","timestamp":1704094800000}\n\n

// 4단계: HTTP 응답으로 전송
res.write(sseFormatted);

// 5단계: HTTP/1.1 청킹 (자동)
// 네트워크로 전송되는 실제 데이터:
// 6e\r\n
// id: 123\n
// data: {"id":123,"message":"실시간 업데이트","timestamp":1704094800000}\n\n
// \r\n
```

#### 3.6.2 수신 측 (브라우저)

```javascript
// 1단계: EventSource 연결
const eventSource = new EventSource('/api/stream');

// 2단계: 네트워크 데이터 수신 (브라우저가 처리)
// TCP/IP 패킷 → HTTP 청킹 해제 → SSE 스트림

// 3단계: SSE 파싱 (EventSource API가 처리)
// 브라우저가 자동으로:
// - 청크 경계 해제
// - SSE 필드 파싱
// - 이벤트 객체 생성

// 4단계: 이벤트 핸들러 호출
eventSource.onmessage = (event) => {
  console.log('받은 이벤트:', event);
  // MessageEvent {
  //   data: '{"id":123,"message":"실시간 업데이트","timestamp":1704094800000}',
  //   lastEventId: '123',
  //   origin: 'https://example.com'
  // }
  
  // 5단계: 애플리케이션에서 사용
  const appData = JSON.parse(event.data);
  updateUI(appData);
};
```

#### 3.6.3 디버깅과 모니터링

```javascript
// 서버 측 디버깅
class SSEDebugger {
  constructor(response) {
    this.response = response;
    this.bytesSent = 0;
    this.messageCount = 0;
  }
  
  send(data, event = null, id = null) {
    let message = '';
    
    if (id) message += `id: ${id}\n`;
    if (event) message += `event: ${event}\n`;
    message += `data: ${JSON.stringify(data)}\n\n`;
    
    // 전송 전 로깅
    console.log(`[SSE] 전송 메시지 #${++this.messageCount}:`);
    console.log(`[SSE] 크기: ${Buffer.byteLength(message)} bytes`);
    console.log(`[SSE] 내용: ${message.replace(/\n/g, '\\n')}`);
    
    this.response.write(message);
    this.bytesSent += Buffer.byteLength(message);
    
    console.log(`[SSE] 총 전송량: ${this.bytesSent} bytes`);
  }
}

// 클라이언트 측 디버깅
class EventSourceDebugger {
  constructor(url) {
    this.eventSource = new EventSource(url);
    this.messageCount = 0;
    this.startTime = Date.now();
    
    this.setupDebugHandlers();
  }
  
  setupDebugHandlers() {
    this.eventSource.onopen = (event) => {
      console.log('[SSE] 연결 성공');
      console.log(`[SSE] readyState: ${this.eventSource.readyState}`);
    };
    
    this.eventSource.onmessage = (event) => {
      const elapsed = Date.now() - this.startTime;
      console.log(`[SSE] 메시지 #${++this.messageCount} (${elapsed}ms):`);
      console.log(`[SSE] data: ${event.data}`);
      console.log(`[SSE] lastEventId: ${event.lastEventId}`);
      
      // 원래 핸들러 호출
      if (this.onmessage) this.onmessage(event);
    };
    
    this.eventSource.onerror = (event) => {
      console.error('[SSE] 오류 발생');
      console.log(`[SSE] readyState: ${this.eventSource.readyState}`);
      
      if (this.onerror) this.onerror(event);
    };
  }
}
```

#### 3.6.4 네트워크 레벨 분석

```javascript
// Wireshark나 Chrome DevTools로 캡처한 실제 HTTP/1.1 트래픽
/*
GET /api/stream HTTP/1.1
Host: example.com
Accept: text/event-stream
Cache-Control: no-cache

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Transfer-Encoding: chunked

1c\r\n
data: First message\n\n\r\n
1d\r\n
data: Second message\n\n\r\n
21\r\n
event: status\ndata: active\n\n\r\n
*/

// Chrome DevTools에서 SSE 모니터링
// 1. Network 탭 열기
// 2. EventStream 필터 선택
// 3. SSE 연결 클릭
// 4. EventStream 탭에서 실시간 메시지 확인
```

#### 3.6.5 성능 고려사항

```javascript
// 버퍼링 최적화
class OptimizedSSEStream {
  constructor(response) {
    this.response = response;
    this.buffer = [];
    this.bufferSize = 0;
    this.maxBufferSize = 1024; // 1KB
    this.flushInterval = 100; // 100ms
    
    this.startFlushTimer();
  }
  
  send(data) {
    const message = `data: ${JSON.stringify(data)}\n\n`;
    this.buffer.push(message);
    this.bufferSize += Buffer.byteLength(message);
    
    // 버퍼가 가득 차면 즉시 전송
    if (this.bufferSize >= this.maxBufferSize) {
      this.flush();
    }
  }
  
  flush() {
    if (this.buffer.length > 0) {
      const combined = this.buffer.join('');
      this.response.write(combined);
      
      this.buffer = [];
      this.bufferSize = 0;
    }
  }
  
  startFlushTimer() {
    this.flushTimer = setInterval(() => {
      this.flush();
    }, this.flushInterval);
  }
  
  close() {
    clearInterval(this.flushTimer);
    this.flush();
    this.response.end();
  }
}

// 압축 설정 (SSE는 압축과 함께 사용 가능)
app.use(compression({
  filter: (req, res) => {
    // SSE도 압축 적용
    if (res.getHeader('Content-Type') === 'text/event-stream') {
      return true;
    }
    return compression.filter(req, res);
  }
}));
```

---

**[← 이전: Part 1 - SSE 개요와 표준 스펙](./part1-overview-and-specs.md)** | **[목차](./README.md)** | **[다음: Part 3 - 빅테크의 비공식 SSE 패턴 →](./part3-bigtech-patterns.md)**

**다음 장 예고**: [Part 3: 빅테크의 비공식 SSE 패턴](./part3-bigtech-patterns.md)에서는 OpenAI, GitHub, Vercel 등 주요 기업들이 SSE를 활용하는 실제 패턴과 커스텀 이벤트 타입들을 살펴보겠습니다.