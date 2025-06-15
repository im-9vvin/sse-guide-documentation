# W3C 표준 SSE(Server-Sent Events) 기술 스펙 포괄적 연구 보고서 (개선판)

## SSE는 서버에서 클라이언트로 단방향 실시간 데이터를 전송하는 HTTP 기반 웹 표준

W3C Server-Sent Events(SSE)는 HTTP 연결을 통해 서버가 웹 페이지로 데이터를 푸시할 수 있게 하는 표준 기술입니다. 2015년 W3C 권고안으로 발표되었으며, 현재는 WHATWG HTML Living Standard의 일부로 지속적으로 개발되고 있습니다. SSE는 `text/event-stream` 미디어 타입을 사용하여 서버에서 클라이언트로의 단방향 통신 채널을 제공하며, 자동 재연결, 이벤트 ID 추적, 간단한 API를 특징으로 합니다. 현재 전 세계 브라우저의 92% 이상이 SSE를 지원하며, 실시간 대시보드, 알림 시스템, 주식 시세 표시 등 다양한 분야에서 활용되고 있습니다.

## 1. SSE 기술의 정의와 핵심 개념

### SSE의 공식 정의

Server-Sent Events(SSE)는 **"서버가 초기 클라이언트 연결이 확립된 후 클라이언트를 향해 데이터 전송을 시작할 수 있는 방법을 설명하는 서버 푸시 기술"**로 정의됩니다. 이는 HTTP 연결을 통해 클라이언트가 서버로부터 자동 업데이트를 받을 수 있게 하는 메커니즘입니다.

### 핵심 특성

SSE 기술의 주요 특성은 다음과 같습니다:

- **단방향 통신(Unidirectional Communication)**: 데이터는 서버에서 클라이언트로만 흐름
- **HTTP 기반(HTTP-Based)**: 특별한 프로토콜 없이 표준 HTTP 연결 사용
- **지속 연결(Persistent Connection)**: 장기간 유지되는 HTTP 연결
- **이벤트 주도(Event-Driven)**: 데이터가 DOM 이벤트로 클라이언트 애플리케이션에 전달
- **UTF-8 인코딩**: 모든 이벤트 스트림은 UTF-8로 인코딩되어야 함
- **자동 재연결(Automatic Reconnection)**: 지수 백오프를 사용한 내장 재연결 로직

### SSE와 기존 기술의 차이점

전통적인 HTTP 요청-응답 모델과 달리, SSE는 서버가 클라이언트의 요청 없이도 지속적으로 데이터를 전송할 수 있습니다. 폴링(Polling)이나 롱 폴링(Long Polling)에 비해 효율적이며, WebSocket보다 구현이 간단하고 HTTP 인프라와의 호환성이 뛰어납니다.

## 2. W3C 표준 스펙 상세 분석

### 표준 문서 현황

SSE 기술의 표준화는 다음과 같은 경로를 거쳤습니다:

**현재 활성 표준**:

- **WHATWG HTML Living Standard**: https://html.spec.whatwg.org/multipage/server-sent-events.html
- 상태: Living Standard (지속적으로 업데이트)
- 관리 기관: WHATWG (Web Hypertext Application Technology Working Group)

**역사적 표준 문서**:

- **W3C Server-Sent Events Recommendation**: 2015년 2월 3일 발표
- 상태: W3C 권고안 (아카이브)
- 주의: 현재 개발은 WHATWG HTML Living Standard로 이전됨

### EventSource 인터페이스 정의 (클라이언트 사이드)

SSE의 핵심인 EventSource 인터페이스는 **브라우저(클라이언트 사이드)**에서 사용되며 다음과 같이 정의됩니다:

```webidl
[Exposed=(Window,Worker)]
interface EventSource : EventTarget {
  constructor(USVString url, optional EventSourceInit eventSourceInitDict = {});

  readonly attribute USVString url;
  readonly attribute boolean withCredentials;

  // 준비 상태 상수
  const unsigned short CONNECTING = 0;
  const unsigned short OPEN = 1;
  const unsigned short CLOSED = 2;
  readonly attribute unsigned short readyState;

  // 네트워킹
  attribute EventHandler onopen;
  attribute EventHandler onmessage;
  attribute EventHandler onerror;

  undefined close();
};

// 클라이언트 사이드: EventSource 생성자 옵션
dictionary EventSourceInit {
  boolean withCredentials = false;
};
```

### 프로토콜 형식 명세 (서버→클라이언트 통신)

SSE는 `text/event-stream` MIME 타입을 사용하며, **서버가 클라이언트로 전송하는 데이터**는 다음과 같은 형식을 따릅니다:

```abnf
stream = [ bom ] *event
event = *( comment / field ) end-of-line
comment = colon *any-char end-of-line
field = 1*name-char [ colon [ space ] *any-char ] end-of-line
end-of-line = ( cr lf / cr / lf )
```

**인식되는 필드 타입 (표준 필드)**:

- `event:` - 이벤트 타입 설정 (기본값: "message") **(표준)**
- `data:` - 이벤트 페이로드 데이터 **(표준)**
- `id:` - 재연결을 위한 마지막 이벤트 ID 설정 **(표준)**
- `retry:` - 재연결 시간 설정 (밀리초 단위, 서버가 전송하면 클라이언트가 적용) **(표준)**
- `:` - 주석 라인 (무시됨) **(표준)**

## 3. API 명세와 이벤트 처리 (클라이언트 사이드)

### EventSource 생성자 (클라이언트 사이드)

```javascript
const eventSource = new EventSource(url [, eventSourceInitDict]);
```

**매개변수**:

- `url` (USVString): 이벤트 스트림을 제공하는 URL
- `eventSourceInitDict` (EventSourceInit, 선택사항):
  - `withCredentials` (boolean, 기본값: false): CORS 자격 증명 모드 설정

### EventSource 객체의 속성(Properties) - 클라이언트 사이드

**readyState** (읽기 전용)

- `CONNECTING` (0): 연결 미확립 또는 재연결 중
- `OPEN` (1): 연결 확립 및 이벤트 수신 중
- `CLOSED` (2): 연결 종료, 재연결 시도 안 함

**url** (읽기 전용): EventSource 객체의 URL 직렬화 반환

**withCredentials** (읽기 전용): CORS 자격 증명 사용 여부 표시

### 이벤트 핸들러 (클라이언트 사이드)

```javascript
eventSource.onopen = function (event) {
  console.log('연결 열림');
};

eventSource.onmessage = function (event) {
  console.log('메시지 수신:', event.data);
};

eventSource.onerror = function (event) {
  console.log('오류 발생');
};

// 커스텀 이벤트 타입
eventSource.addEventListener('userJoined', function (event) {
  console.log('사용자 참여:', event.data);
});
```

## 4. 보안 고려사항

### CORS (Cross-Origin Resource Sharing)

SSE는 동일 출처 정책을 따르며, 교차 출처 요청 시 **서버에서** 적절한 CORS 헤더를 설정해야 합니다:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

### 인증 및 권한 부여

SSE의 네이티브 EventSource API는 커스텀 헤더를 지원하지 않으므로, 다음과 같은 방법을 사용합니다:

- **쿠키 기반 인증**: 동일 출처 요청에 권장 (클라이언트가 자동으로 쿠키 전송)
- **URL 쿼리 매개변수**: 토큰 전달 (로깅 노출 위험 주의)
- **세션 기반 인증**: 서버 측에서 세션 검증

### SSL/TLS 요구사항

프로덕션 환경에서는 다음을 준수해야 합니다:

- HTTPS 사용 필수
- 최소 TLS 1.2, TLS 1.3 권장
- 혼합 콘텐츠 방지 (HTTPS 페이지에서 HTTPS SSE 엔드포인트 사용)

### XSS 및 CSRF 방어 전략

**필수 보안 헤더 (서버 사이드 설정)**:

```
Content-Type: text/event-stream
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Cache-Control: no-cache
```

**데이터 검증**: 서버에서 전송하는 모든 데이터를 적절히 이스케이프하고, 클라이언트에서 처리 시 추가 검증을 수행해야 합니다.

## 5. 브라우저 호환성 현황

### 데스크톱 브라우저 (2025년 기준)

- **Chrome**: 6+ (2010년부터 지원, 현재: 135+)
- **Firefox**: 6+ (2011년부터 지원, 현재: 137+)
- **Safari**: 5+ (2010년부터 지원, 현재: 18.4+)
- **Edge**: 79+ (2020년 1월부터 지원, 현재: 135+)
- **Opera**: 11+ (현재: 117+)
- **Internet Explorer**: 지원하지 않음

### 모바일 브라우저

- **Chrome for Android**: 완전 지원 (현재: 135)
- **Safari on iOS**: iOS 4+부터 완전 지원 (현재: 18.4+)
- **Firefox for Android**: 완전 지원 (현재: 137)
- **Samsung Internet**: 완전 지원 (현재: 27)

**전체 호환성 점수**: 92/100 (전 세계 브라우저의 92% 이상이 SSE 지원)

### 폴리필 솔루션 (클라이언트 사이드)

IE 및 레거시 브라우저를 위한 클라이언트 사이드 폴리필:

- `event-source-polyfill`: 커스텀 헤더 지원 포함
- `eventsource` (npm): Node.js 및 브라우저 폴리필
- Yaffle의 EventSource 폴리필: 경량 브라우저 폴리필

## 6. WebSocket과의 비교 분석

### 기술적 차이점

| 측면            | SSE                           | WebSocket                          |
| --------------- | ----------------------------- | ---------------------------------- |
| **통신 방향**   | 단방향 (서버→클라이언트)      | 양방향 (전이중)                    |
| **프로토콜**    | HTTP 기반 (text/event-stream) | HTTP 업그레이드 후 커스텀 프로토콜 |
| **데이터 형식** | UTF-8 텍스트만                | 바이너리 및 텍스트 지원            |
| **연결 설정**   | 단일 HTTP 요청                | HTTP 업그레이드 핸드셰이크         |
| **재연결**      | 자동 (이벤트 ID 추적 포함)    | 수동 구현 필요                     |
| **보안 모델**   | 표준 HTTP 보안                | 추가 보안 조치 필요                |

### 사용 사례별 적합성

**SSE가 적합한 경우**:

- 실시간 대시보드
- 주식 시세 표시
- 뉴스 피드
- 알림 시스템
- 진행 상황 추적

**WebSocket이 적합한 경우**:

- 채팅 애플리케이션
- 실시간 게임
- 협업 편집 도구
- 거래 플랫폼
- 바이너리 데이터 전송

### 성능 비교

연구 결과, SSE와 WebSocket의 성능은 유사한 수준을 보입니다:

- **최대 처리량**: SSE ~270만 이벤트/초, WebSocket ~240만 이벤트/초
- **실용적 시나리오**: 10만 이벤트/초, 50ms 목표 대기 시간에서 동등한 성능
- **CPU 활용도**: WebSocket이 약간 낮음 (10-15% 차이)

## 7. 성능 특성과 최적화 방법

### 대기 시간 및 처리량

**실제 성능 메트릭**:

- **Shopify BFCM**: 데이터 가용성 후 밀리초 내 전달
- **LinkedIn 메시징**: 서버당 10만+ 동시 연결 성공적 확장
- **금융 플랫폼**: 압축 및 배치 처리로 데이터 전송량 90% 감소

### HTTP/2 및 HTTP/3의 영향

**HTTP/2 이점**:

- 다중화로 6개 연결 제한 해결
- 바이너리 프로토콜로 파싱 효율성 향상
- 스트림 우선순위로 중요 이벤트 리소스 할당 개선
- 캐시 가능 요청의 경우 최대 28% 처리 시간 단축

**HTTP/3 장점**:

- QUIC 전송으로 head-of-line 차단 제거
- 내장 TLS로 빠른 초기 연결
- 연결 마이그레이션으로 모바일 성능 향상

### 서버 측 최적화 전략 (서버 사이드)

**프레임워크 선택**:

- 권장: 비동기 프레임워크 (Node.js, Python FastAPI, Go, Scala Play)
- 피해야 할 것: 동기식 요청당 프로세스 모델 (전통적인 LAMP 스택)

**연결 관리 최적화 (서버 사이드)**:

```javascript
// 서버 사이드: Node.js/Express 최적화 예제
app.get('/events', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no',
  });

  connections.add(res);

  req.on('close', () => {
    connections.delete(res);
  });
});
```

### 메시지 배치 및 압축 (서버 사이드)

**서버 측 성능 향상 기법**:

- gzip 압축 활성화 (60-80% 크기 감소)
- 효율적인 JSON 직렬화
- 빈번한 업데이트를 위한 델타 압축
- 높은 빈도 업데이트를 위한 배치 처리

## 8. 실제 사용 사례와 적용 시나리오

### 실시간 대시보드 (서버 사이드)

```javascript
// 서버 사이드: Node.js/Express 메트릭 스트리밍 서버
app.get('/metrics-stream', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
  });

  const sendMetrics = () => {
    const metrics = {
      cpu: Math.random() * 100,
      memory: Math.random() * 100,
      timestamp: Date.now(),
    };

    res.write(`data: ${JSON.stringify(metrics)}\n\n`);
  };

  const interval = setInterval(sendMetrics, 2000);

  req.on('close', () => {
    clearInterval(interval);
  });
});
```

### 알림 시스템 (서버 사이드)

```javascript
// 서버 사이드: 알림 서비스 클래스
class NotificationService {
  notify(userId, notification) {
    const response = this.subscribers.get(userId);
    if (response) {
      response.write(
        `data: ${JSON.stringify({
          type: 'notification',
          title: notification.title,
          message: notification.message,
          timestamp: new Date().toISOString(),
        })}\n\n`
      );
    }
  }
}
```

### 실제 기업 사례

**Shopify Black Friday Cyber Monday (BFCM) 라이브 맵**:

- Go 기반 SSE 서버로 100% 가동 시간 달성
- 엔드투엔드 21초 대기 시간으로 실시간 시각화
- 이전 10초 폴링 시스템 대비 현저한 개선

**LinkedIn 인스턴트 메시징**:

- Play Framework와 Akka Actor 모델 사용
- 서버당 10만+ 동시 연결 지원
- 타이핑 표시 및 읽음 확인을 위한 1초 미만 메시지 전달

## 9. 구현 예제와 코드 샘플

### 서버 구현 (Node.js/Express - 서버 사이드)

```javascript
// 서버 사이드: Node.js/Express SSE 엔드포인트
const express = require('express');
const app = express();

// SSE 엔드포인트
app.get('/events', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  });

  // 초기 데이터 전송
  res.write(`data: ${JSON.stringify({ message: '연결됨' })}\n\n`);

  // 주기적 업데이트
  const interval = setInterval(() => {
    res.write(
      `data: ${JSON.stringify({
        timestamp: new Date().toISOString(),
        message: '서버 업데이트',
      })}\n\n`
    );
  }, 5000);

  // 클라이언트 연결 해제 처리
  req.on('close', () => {
    clearInterval(interval);
  });
});
```

### 클라이언트 구현 (JavaScript/React - 클라이언트 사이드)

```javascript
// 클라이언트 사이드: React Hook 예제
function useSSE(url) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [status, setStatus] = useState('connecting');

  useEffect(() => {
    const eventSource = new EventSource(url);

    eventSource.onopen = () => {
      setStatus('connected');
      setError(null);
    };

    eventSource.onmessage = (event) => {
      try {
        const parsedData = JSON.parse(event.data);
        setData(parsedData);
      } catch (err) {
        setError(err.message);
      }
    };

    eventSource.onerror = () => {
      setStatus('error');
      setError('연결 실패');
    };

    return () => {
      eventSource.close();
    };
  }, [url]);

  return { data, error, status };
}
```

## 10. 에러 처리와 재연결 메커니즘

### 자동 재연결 동작 (클라이언트 사이드)

SSE의 내장 재연결 메커니즘 (클라이언트 동작):

- 기본 동작: 연결 손실 시 3초 후 자동 재연결
- 서버 제어: `retry: 15000` 필드로 재연결 간격 지정 (밀리초)
- 브라우저별 동작: Chrome은 무한 재연결, Firefox는 제한된 재연결 시도

### 강력한 클라이언트 구현 (클라이언트 사이드)

```javascript
// 클라이언트 사이드: 재연결 기능이 강화된 SSE 클래스
class RobustSSE {
  constructor(url, options = {}) {
    this.url = url;
    this.retryCount = 0;
    this.maxRetries = options.maxRetries || 5;
    this.retryDelay = options.retryDelay || 3000;
  }

  connect() {
    this.eventSource = new EventSource(this.url);

    this.eventSource.onerror = () => {
      this.handleError();
    };
  }

  handleError() {
    this.close();

    if (this.retryCount < this.maxRetries) {
      this.retryCount++;
      const delay = this.retryDelay * Math.pow(2, this.retryCount - 1);

      setTimeout(() => {
        this.connect();
      }, delay);
    }
  }
}
```

## 11. HTTP 스트리밍과 청킹 메커니즘

### 레이어별 데이터 흐름 이해

SSE는 두 개의 독립적인 프로토콜 레이어에서 작동합니다:

1. **전송 레이어 (HTTP/1.1 청킹)**: 네트워크 상의 실제 데이터 전송
2. **애플리케이션 레이어 (SSE 프로토콜)**: EventSource가 처리하는 이벤트 스트림

### HTTP 청킹(Chunking)과 SSE - 레이어별 구분

#### 1. 네트워크 레이어 (HTTP/1.1 청킹)

네트워크 상에서 실제로 전송되는 데이터 형태:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Transfer-Encoding: chunked
Cache-Control: no-cache

1f\r\n                              ← 청크 크기 (16진수) + CRLF
data: 첫 번째 이벤트\n\n\r\n        ← 청크 데이터 + CRLF
21\r\n                              ← 다음 청크 크기 + CRLF
data: 두 번째 이벤트\n\n\r\n        ← 청크 데이터 + CRLF
0\r\n                               ← 마지막 청크 표시 + CRLF
\r\n                                ← 최종 CRLF
```

**핵심 포인트**:

- HTTP/1.1 표준(RFC 7230)에 따라 **반드시 CRLF(`\r\n`)** 사용
- 청크 크기는 16진수로 표현
- 각 청크는 크기 헤더와 데이터로 구성

#### 2. 애플리케이션 레이어 (SSE 이벤트 스트림)

브라우저의 EventSource가 받는 데이터 (HTTP 청킹이 제거된 후):

```
data: 첫 번째 이벤트\n\n            ← 순수 SSE 데이터
data: 두 번째 이벤트\n\n            ← 순수 SSE 데이터
```

**핵심 포인트**:

- HTTP 청킹 정보(`\r\n`, 청크 크기)는 완전히 제거됨
- SSE 표준에 따라 **이벤트 경계는 빈 줄(`\n\n`)**로 구분
- EventSource는 청킹을 인식하지 못하고 순수 이벤트 스트림만 처리

### 실제 데이터 처리 과정

#### 서버 → 네트워크 전송

```javascript
// 서버 코드 (Node.js)
res.write('data: 이벤트\n\n'); // 개발자가 작성하는 코드

// ↓ HTTP 서버가 자동으로 청킹 처리

// 네트워크에 실제 전송되는 형태:
// 10\r\n
// data: 이벤트\n\n\r\n
```

#### 네트워크 → 클라이언트 수신

```javascript
// 1. 브라우저가 네트워크에서 받는 데이터:
// 10\r\ndata: 이벤트\n\n\r\n

// 2. HTTP 파서가 청킹 제거:
// data: 이벤트\n\n

// 3. EventSource가 받는 최종 데이터:
eventSource.onmessage = (event) => {
  console.log(event.data); // "이벤트" (청킹 정보 없음)
};
```

### 청크와 이벤트의 관계 - 명확한 구분

| 구분              | HTTP 청킹 레이어           | SSE 이벤트 레이어      |
| ----------------- | -------------------------- | ---------------------- |
| **역할**          | 데이터 전송 메커니즘       | 이벤트 스트림 프로토콜 |
| **경계 구분**     | CRLF (`\r\n`)              | 빈 줄 (`\n\n`)         |
| **데이터 단위**   | 청크 (임의 크기)           | 이벤트 (논리적 단위)   |
| **처리 주체**     | HTTP 파서                  | EventSource API        |
| **개발자 가시성** | 숨겨짐 (프레임워크가 처리) | 직접 처리              |

### 스트리밍 버퍼링 문제와 해결

**문제**: 프록시나 중간 서버가 응답을 버퍼링하여 실시간성 저하

**해결책 (서버 사이드)**:

```javascript
// Node.js에서 버퍼링 비활성화
res.writeHead(200, {
  'Content-Type': 'text/event-stream',
  'Cache-Control': 'no-cache',
  'X-Accel-Buffering': 'no', // Nginx 버퍼링 비활성화
  'Connection': 'keep-alive',
});

// 즉시 플러시
res.write('data: connected\n\n');
res.flushHeaders(); // 헤더 즉시 전송
```

**Nginx 설정 (서버 사이드)**:

```nginx
location /events {
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 24h;
    chunked_transfer_encoding on;
    tcp_nodelay on;  # 작은 패킷도 즉시 전송
    tcp_nopush off;  # 버퍼링 비활성화
}
```

### 청크 크기 최적화

**성능 고려사항**:

- **작은 청크 (수십 바이트)**: 낮은 지연시간, 높은 오버헤드
- **큰 청크 (수 KB)**: 높은 처리량, 지연시간 증가
- **권장**: 이벤트 크기에 따라 동적 조정

```javascript
// 서버 사이드: 청크 크기 제어 예제
class SSEStream {
  constructor(response) {
    this.response = response;
    this.buffer = [];
    this.bufferSize = 0;
    this.maxBufferSize = 1024; // 1KB
  }

  write(data) {
    this.buffer.push(data);
    this.bufferSize += data.length;

    // 버퍼가 가득 차거나 중요한 이벤트일 때 플러시
    if (this.bufferSize >= this.maxBufferSize || data.includes('priority:high')) {
      this.flush();
    }
  }

  flush() {
    if (this.buffer.length > 0) {
      this.response.write(this.buffer.join(''));
      this.buffer = [];
      this.bufferSize = 0;
    }
  }
}
```

### HTTP/2에서의 스트리밍

HTTP/2에서는 청킹 방식이 다릅니다:

- `Transfer-Encoding: chunked` 헤더 사용 안 함
- DATA 프레임으로 스트림 전송
- 프레임 단위로 흐름 제어
- 멀티플렉싱으로 여러 SSE 연결 동시 처리

```javascript
// HTTP/2 감지 및 최적화 (서버 사이드)
app.get('/events', (req, res) => {
  const headers = {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
  };

  // HTTP/2는 Transfer-Encoding 헤더 불필요
  if (req.httpVersion !== '2.0') {
    headers['Transfer-Encoding'] = 'chunked';
  }

  res.writeHead(200, headers);
});
```

### 클라이언트의 청크 처리 - 상세 과정

브라우저의 EventSource는 청크 처리를 자동으로 수행합니다:

1. **청크 수신**: 네트워크에서 HTTP 청크 수신

   ```
   10\r\ndata: 테스트\n\n\r\n
   ```

2. **HTTP 파싱**: 브라우저의 HTTP 파서가 청킹 제거

   ```
   data: 테스트\n\n
   ```

3. **버퍼 축적**: EventSource 내부 버퍼에 데이터 추가

4. **이벤트 파싱**: 빈 줄(`\n\n`) 발견 시 이벤트 경계 인식

5. **이벤트 발생**: 완전한 이벤트를 JavaScript 이벤트로 변환
   ```javascript
   // 최종적으로 개발자가 받는 데이터
   eventSource.onmessage = (event) => {
     console.log(event.data); // "테스트"
   };
   ```

**주의사항**:

- 클라이언트는 HTTP 청크 경계를 인식하지 못함
- 오직 SSE 이벤트 경계(`\n\n`)만 인식
- 하나의 이벤트가 여러 HTTP 청크에 걸쳐 전송될 수 있음

## 12. 프로토콜 상세 사항

### 메시지 형식 (서버→클라이언트)

**기본 메시지 (서버가 전송)**:

```
data: Hello World

```

**여러 줄 메시지 (서버가 전송)**:

```
data: 첫 번째 줄
data: 두 번째 줄
data: 세 번째 줄

```

**명명된 이벤트와 ID (서버가 전송)**:

```
event: userJoined
data: {"userId": 12345, "name": "John"}
id: msg-001

```

### 필드 처리 규칙 (클라이언트의 파싱 동작)

클라이언트(EventSource)가 서버로부터 받은 데이터를 처리하는 규칙:

1. **빈 줄**: 축적된 이벤트 디스패치 및 버퍼 초기화
2. **주석 줄 (`:` 시작)**: 완전히 무시
3. **콜론이 있는 필드**: 첫 번째 `:`에서 분할, 콜론 뒤 선택적 공백 제거
4. **콜론이 없는 필드**: 전체 줄이 빈 값을 가진 필드 이름

## 13. 프로덕션 배포 가이드

### 로드 밸런싱 구성 (서버 사이드)

**NGINX 구성 (서버 사이드)**:

```nginx
upstream sse_backend {
    ip_hash;  # SSE 연결을 위한 스티키 세션
    server backend1.example.com:3000;
    server backend2.example.com:3000;
}

location /events {
    proxy_pass http://sse_backend;
    proxy_http_version 1.1;
    proxy_buffering off;
    proxy_read_timeout 24h;
    add_header X-Accel-Buffering no;
}
```

### 모니터링 및 디버깅 (서버 사이드)

**연결 모니터링 (서버 사이드)**:

```javascript
// 서버 사이드: SSE 연결 모니터링 클래스
class SSEMonitor {
  constructor() {
    this.metrics = {
      totalConnections: 0,
      activeConnections: 0,
      messagesSent: 0,
      errors: 0,
    };
  }

  trackConnection(clientId, metadata) {
    this.metrics.activeConnections++;
    this.metrics.totalConnections++;
  }

  getMetrics() {
    return this.metrics;
  }
}
```

## 결론

Server-Sent Events(SSE)는 단방향 실시간 통신을 위한 강력하고 간단한 웹 표준입니다. WebSocket에 비해 구현이 간단하면서도 유사한 성능을 제공하며, HTTP 인프라와의 뛰어난 호환성을 가지고 있습니다. 자동 재연결, 이벤트 ID 추적, 간단한 텍스트 기반 프로토콜 등의 특징으로 인해 실시간 대시보드, 알림 시스템, 라이브 피드 등 다양한 애플리케이션에 적합합니다.

HTTP/2의 도입으로 과거의 연결 제한 문제가 해결되었으며, 현재 전 세계 브라우저의 92% 이상이 SSE를 지원합니다. 적절한 서버 아키텍처와 최적화를 통해 수십만 개의 동시 연결을 처리할 수 있으며, 실제 기업 사례에서도 그 효용성이 입증되었습니다. SSE는 양방향 통신이나 바이너리 데이터 전송이 필요하지 않은 대부분의 실시간 웹 애플리케이션에서 WebSocket의 훌륭한 대안이 될 수 있습니다.
