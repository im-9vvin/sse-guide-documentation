***
**[목차](./README.md)** | **[다음: Part 2 - 작동 원리와 청킹 메커니즘 →](./part2-working-principles.md)**

***

# Part 1: SSE 개요와 표준 스펙

## 1장. SSE 기술 개요

### 1.1 SSE의 정의와 핵심 개념

Server-Sent Events(SSE)는 서버에서 클라이언트로 단방향 실시간 데이터 스트리밍을 가능하게 하는 W3C 표준 기술입니다. 2004년 Ian Hickson이 제안한 이후, 2015년 W3C Recommendation으로 표준화되었습니다.

#### SSE의 핵심 정의
```
SSE는 HTTP 프로토콜을 통해 서버가 웹 브라우저로 자동 업데이트를 
푸시할 수 있도록 하는 기술입니다. 클라이언트가 한 번 연결을 
설정하면, 서버는 필요할 때마다 데이터를 전송할 수 있습니다.
```

#### 기본 작동 원리
```javascript
// 클라이언트 측 연결
const eventSource = new EventSource('/api/stream');

// 서버에서 보낸 메시지 수신
eventSource.onmessage = (event) => {
  console.log('받은 데이터:', event.data);
};

// 서버 측 응답 (Node.js 예제)
res.writeHead(200, {
  'Content-Type': 'text/event-stream',
  'Cache-Control': 'no-cache',
  'Connection': 'keep-alive'
});

res.write('data: Hello, SSE!\n\n');
```

### 1.2 핵심 특성

#### 1.2.1 단방향 통신
- **서버 → 클라이언트**: 서버에서 클라이언트로만 데이터 전송
- **지속적 연결**: 한 번 연결되면 서버가 필요할 때마다 데이터 푸시
- **HTTP 기반**: 별도의 프로토콜이 아닌 표준 HTTP 위에서 동작

#### 1.2.2 자동 재연결
```javascript
// EventSource는 연결이 끊어지면 자동으로 재연결을 시도합니다
eventSource.onerror = (error) => {
  console.log('연결 오류 발생, 자동 재연결 시도 중...');
  // 브라우저가 자동으로 처리, 개발자가 별도 구현 불필요
};
```

#### 1.2.3 텍스트 기반 프로토콜
```
data: 첫 번째 메시지\n\n
data: {"type": "update", "content": "JSON 데이터도 전송 가능"}\n\n
event: custom-event\n
data: 커스텀 이벤트와 함께 전송\n\n
```

### 1.3 SSE와 기존 기술의 차이점

#### 1.3.1 폴링(Polling) vs SSE

**전통적인 폴링:**
```javascript
// 클라이언트가 주기적으로 서버에 요청
setInterval(async () => {
  const response = await fetch('/api/updates');
  const data = await response.json();
  updateUI(data);
}, 5000); // 5초마다 요청
```

**SSE 방식:**
```javascript
// 서버가 필요할 때만 데이터 전송
const eventSource = new EventSource('/api/updates');
eventSource.onmessage = (event) => {
  updateUI(JSON.parse(event.data));
};
```

#### 1.3.2 WebSocket vs SSE

| 특성 | WebSocket | SSE |
|------|-----------|-----|
| 통신 방향 | 양방향 (Full-duplex) | 단방향 (서버 → 클라이언트) |
| 프로토콜 | ws:// 또는 wss:// | http:// 또는 https:// |
| 복잡도 | 높음 | 낮음 |
| 자동 재연결 | 수동 구현 필요 | 내장 기능 |
| 방화벽 친화성 | 제한적 | 우수 (HTTP 사용) |
| 브라우저 지원 | 광범위 | 광범위 (IE 제외) |

**WebSocket 예제:**
```javascript
// 양방향 통신이 필요한 경우
const ws = new WebSocket('wss://example.com/socket');

ws.onopen = () => {
  ws.send('클라이언트에서 서버로 메시지 전송');
};

ws.onmessage = (event) => {
  console.log('서버로부터 받은 메시지:', event.data);
};

// 재연결 로직을 직접 구현해야 함
ws.onclose = () => {
  setTimeout(() => reconnect(), 5000);
};
```

**SSE 예제:**
```javascript
// 서버에서 클라이언트로만 데이터 전송
const eventSource = new EventSource('/api/stream');

eventSource.onmessage = (event) => {
  console.log('서버로부터 받은 메시지:', event.data);
};

// 자동 재연결 - 추가 구현 불필요
```

#### 1.3.3 각 기술의 사용 사례

**폴링이 적합한 경우:**
- 실시간성이 중요하지 않은 경우
- 간단한 상태 체크
- 레거시 시스템 지원

**SSE가 적합한 경우:**
- 서버에서 클라이언트로의 단방향 실시간 업데이트
- 실시간 알림, 뉴스 피드
- 실시간 차트/대시보드
- AI 챗봇 응답 스트리밍
- 진행 상황 업데이트

**WebSocket이 적합한 경우:**
- 양방향 실시간 통신이 필요한 경우
- 온라인 게임
- 협업 도구 (구글 독스 등)
- 실시간 채팅 (사용자 입력 전송 필요)

## 2장. W3C 표준 스펙 상세 분석

### 2.1 표준 문서 현황

#### 2.1.1 공식 표준 문서
- **W3C Recommendation**: https://www.w3.org/TR/eventsource/
- **WHATWG Living Standard**: https://html.spec.whatwg.org/multipage/server-sent-events.html
- **상태**: 2015년 2월 3일 W3C Recommendation 승인

#### 2.1.2 표준화 역사
```
2004년: Ian Hickson이 최초 제안
2009년: W3C Working Draft 발표
2011년: Last Call Working Draft
2013년: Candidate Recommendation
2015년: W3C Recommendation (표준 확정)
현재: WHATWG Living Standard로 지속적 업데이트
```

### 2.2 EventSource 인터페이스 정의

#### 2.2.1 IDL 정의
```idl
[Exposed=(Window,Worker)]
interface EventSource : EventTarget {
  constructor(USVString url, optional EventSourceInit eventSourceInitDict = {});

  readonly attribute USVString url;
  readonly attribute boolean withCredentials;

  // ready state
  const unsigned short CONNECTING = 0;
  const unsigned short OPEN = 1;
  const unsigned short CLOSED = 2;
  readonly attribute unsigned short readyState;

  // networking
  attribute EventHandler onopen;
  attribute EventHandler onmessage;
  attribute EventHandler onerror;
  undefined close();
};

dictionary EventSourceInit {
  boolean withCredentials = false;
};
```

#### 2.2.2 생성자와 초기화
```javascript
// 기본 생성자
const eventSource = new EventSource('/api/stream');

// 인증 정보 포함
const eventSourceWithAuth = new EventSource('/api/stream', {
  withCredentials: true  // 쿠키 포함
});

// 절대 URL 사용
const eventSourceAbsolute = new EventSource('https://api.example.com/stream');
```

#### 2.2.3 readyState 상태
```javascript
// EventSource.CONNECTING (0): 연결 중
// EventSource.OPEN (1): 연결됨
// EventSource.CLOSED (2): 연결 종료

eventSource.onopen = () => {
  console.log('연결 상태:', eventSource.readyState); // 1 (OPEN)
};

eventSource.onerror = () => {
  if (eventSource.readyState === EventSource.CLOSED) {
    console.log('연결이 종료되었습니다');
  } else {
    console.log('연결 중 오류 발생, 재연결 시도 중...');
  }
};
```

### 2.3 프로토콜 형식 명세

#### 2.3.1 메시지 형식
```
field: value\n
field: value\n
\n
```

#### 2.3.2 표준 필드

**data 필드:**
```
data: 단순 텍스트 메시지\n\n

data: 첫 번째 줄\n
data: 두 번째 줄\n
data: 세 번째 줄\n\n
```

**event 필드:**
```
event: user-login\n
data: {"userId": "12345", "timestamp": "2024-01-15T10:30:00Z"}\n\n

event: system-alert\n
data: 시스템 점검이 예정되어 있습니다\n\n
```

**id 필드:**
```
id: 1\n
data: 첫 번째 메시지\n\n

id: 2\n
data: 두 번째 메시지\n\n

// 클라이언트가 재연결 시 Last-Event-ID: 2 헤더 전송
```

**retry 필드:**
```
retry: 10000\n
data: 재연결 간격을 10초로 설정\n\n

retry: 30000\n
data: 재연결 간격을 30초로 변경\n\n
```

**주석 (comment):**
```
: 이것은 주석입니다. 클라이언트에서 무시됩니다\n
: Keep-alive를 위해 주기적으로 전송할 수 있습니다\n\n

: heartbeat\n\n
```

#### 2.3.3 메시지 파싱 규칙

1. **줄 단위 처리**: `\n`, `\r`, 또는 `\r\n`으로 구분
2. **필드 구분**: 첫 번째 콜론(`:`)을 기준으로 필드명과 값 구분
3. **공백 처리**: 콜론 뒤의 첫 번째 공백은 무시
4. **빈 줄**: 메시지 종료를 의미

```javascript
// 서버에서 보낸 데이터
`data: {"type": "message"}\n\n`

// 파싱 결과
{
  data: '{"type": "message"}'
}

// 여러 줄 데이터
`data: {\n
data:   "type": "multiline",\n
data:   "content": "여러 줄에 걸친 JSON"\n
data: }\n\n`

// 파싱 결과 (줄바꿈으로 연결됨)
{
  data: '{\n  "type": "multiline",\n  "content": "여러 줄에 걸친 JSON"\n}'
}
```

### 2.4 API 명세와 이벤트 처리

#### 2.4.1 이벤트 핸들러

**onopen 이벤트:**
```javascript
eventSource.onopen = (event) => {
  console.log('SSE 연결 성공');
  console.log('readyState:', eventSource.readyState); // 1 (OPEN)
};
```

**onmessage 이벤트:**
```javascript
// 기본 메시지 수신 (event 필드가 없는 경우)
eventSource.onmessage = (event) => {
  console.log('받은 데이터:', event.data);
  console.log('이벤트 ID:', event.lastEventId);
  console.log('Origin:', event.origin);
};
```

**onerror 이벤트:**
```javascript
eventSource.onerror = (event) => {
  console.error('SSE 오류 발생');
  
  if (eventSource.readyState === EventSource.CLOSED) {
    console.log('연결이 완전히 종료됨');
  } else if (eventSource.readyState === EventSource.CONNECTING) {
    console.log('재연결 시도 중...');
  }
};
```

#### 2.4.2 커스텀 이벤트 리스너
```javascript
// 서버에서 event 필드를 지정한 경우
eventSource.addEventListener('user-login', (event) => {
  const userData = JSON.parse(event.data);
  console.log('사용자 로그인:', userData);
});

eventSource.addEventListener('notification', (event) => {
  showNotification(event.data);
});

eventSource.addEventListener('progress', (event) => {
  updateProgressBar(JSON.parse(event.data).percentage);
});
```

#### 2.4.3 연결 종료
```javascript
// 명시적 연결 종료
eventSource.close();

// 연결 종료 후 상태 확인
console.log(eventSource.readyState); // 2 (CLOSED)

// 종료 후에는 재사용 불가, 새로운 인스턴스 생성 필요
eventSource = new EventSource('/api/stream');
```

#### 2.4.4 에러 처리와 재연결

**자동 재연결 메커니즘:**
```javascript
let reconnectTime = 1000; // 초기 재연결 시간 (1초)

eventSource.onerror = (event) => {
  console.log(`재연결 시도까지 ${reconnectTime}ms 대기`);
  
  // 브라우저가 자동으로 재연결을 시도합니다
  // retry 필드로 서버에서 재연결 간격을 제어할 수 있습니다
};

// 서버에서 재연결 간격 제어
res.write('retry: 5000\n'); // 5초로 설정
res.write('data: 재연결 간격이 5초로 변경되었습니다\n\n');
```

**수동 재연결 구현 (필요한 경우):**
```javascript
class ReconnectingEventSource {
  constructor(url, options = {}) {
    this.url = url;
    this.options = options;
    this.reconnectInterval = 1000;
    this.maxReconnectInterval = 30000;
    this.reconnectAttempts = 0;
    
    this.connect();
  }
  
  connect() {
    this.eventSource = new EventSource(this.url, this.options);
    
    this.eventSource.onopen = (event) => {
      this.reconnectAttempts = 0;
      this.reconnectInterval = 1000;
      if (this.onopen) this.onopen(event);
    };
    
    this.eventSource.onmessage = (event) => {
      if (this.onmessage) this.onmessage(event);
    };
    
    this.eventSource.onerror = (event) => {
      if (this.onerror) this.onerror(event);
      
      if (this.eventSource.readyState === EventSource.CLOSED) {
        // 지수 백오프로 재연결
        this.reconnectAttempts++;
        const timeout = Math.min(
          this.reconnectInterval * Math.pow(2, this.reconnectAttempts),
          this.maxReconnectInterval
        );
        
        setTimeout(() => this.connect(), timeout);
      }
    };
  }
  
  close() {
    if (this.eventSource) {
      this.eventSource.close();
    }
  }
}
```

#### 2.4.5 브라우저 호환성

| 브라우저 | 지원 버전 |
|---------|----------|
| Chrome | 6+ |
| Firefox | 6+ |
| Safari | 5+ |
| Edge | 79+ |
| Opera | 11+ |
| IE | 미지원 |

**폴리필 사용 (IE 지원):**
```html
<script src="https://cdn.jsdelivr.net/npm/event-source-polyfill@1.0.31/src/eventsource.min.js"></script>
<script>
  // 폴리필이 적용된 EventSource 사용
  const eventSource = new EventSourcePolyfill('/api/stream', {
    headers: {
      'Authorization': 'Bearer token'
    }
  });
</script>
```

***

**[목차](./README.md)** | **[다음: Part 2 - 작동 원리와 청킹 메커니즘 →](./part2-working-principles.md)**

**다음 장 예고**: [Part 2: 작동 원리와 청킹 메커니즘](./part2-working-principles.md)에서는 SSE가 HTTP 레벨에서 어떻게 작동하는지, 청킹 메커니즘의 역할과 HTTP 버전별 차이점을 살펴보겠습니다.