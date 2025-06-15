# SSE의 두 레이어와 청킹 메커니즘 상세 설명

## 목차

1. [SSE의 두 독립적인 레이어](#1-sse의-두-독립적인-레이어)
2. [데이터 변환 과정](#2-데이터-변환-과정)
3. [청킹의 역할과 위치](#3-청킹의-역할과-위치)
4. [청킹이 필요한 이유](#4-청킹이-필요한-이유)
5. [실제 구현 예시](#5-실제-구현-예시)

## 1. SSE의 두 독립적인 레이어

SSE는 완전히 독립적인 두 개의 프로토콜 레이어에서 작동합니다:

### 전송 레이어 (HTTP 청킹)

- **역할**: 네트워크상에서 데이터를 어떻게 전송할지 결정
- **주체**: HTTP 프로토콜 (서버와 브라우저의 HTTP 파서)
- **목적**: 크기를 알 수 없는 데이터를 스트리밍으로 전송
- **경계 구분**: CRLF (`\r\n`)
- **데이터 단위**: 청크 (임의 크기)

### 애플리케이션 레이어 (SSE 프로토콜)

- **역할**: 이벤트를 어떻게 구조화하고 파싱할지 결정
- **주체**: EventSource API
- **목적**: 서버에서 보낸 이벤트를 JavaScript 이벤트로 변환
- **경계 구분**: 빈 줄 (`\n\n`)
- **데이터 단위**: 이벤트 (논리적 단위)

### 두 레이어의 독립성 예시

#### 예시 1: 하나의 SSE 이벤트가 여러 HTTP 청크로 전송

```
서버가 보내는 SSE 이벤트:
data: 매우 긴 이벤트 데이터...(1000자)\n\n

네트워크 전송 시 HTTP 청킹:
[청크 1] 100\r\ndata: 매우 긴 이벤...\r\n
[청크 2] 100\r\n트 데이터의 중간 부분...\r\n
[청크 3] 50\r\n...나머지 데이터\n\n\r\n
```

#### 예시 2: 여러 SSE 이벤트가 하나의 HTTP 청크로 전송

```
서버가 보내는 SSE 이벤트들:
data: 첫 번째\n\n
data: 두 번째\n\n
data: 세 번째\n\n

네트워크 전송 시 HTTP 청킹:
[청크 1] 2A\r\ndata: 첫 번째\n\ndata: 두 번째\n\ndata: 세 번째\n\n\r\n
```

## 2. 데이터 변환 과정

### 전체 데이터 흐름

```
1. [서버 애플리케이션]
   res.write('data: 안녕하세요\n\n');
   ↓
2. [SSE 텍스트 스트림] - 원본 데이터
   data: 안녕하세요\n\n
   ↓
3. [HTTP 청킹 레이어] - 전송을 위한 포장
   14\r\ndata: 안녕하세요\n\n\r\n
   ↓
4. [네트워크 전송]
   ↓
5. [브라우저 HTTP 파서] - 포장 제거
   청킹 정보(14\r\n, \r\n) 제거
   ↓
6. [순수 텍스트 스트림 복원]
   data: 안녕하세요\n\n
   ↓
7. [EventSource API]
   MessageEvent { data: "안녕하세요", type: "message", ... }
```

### 데이터 변환의 핵심

#### 전송 레이어에서 애플리케이션 레이어로의 변환

**질문**: "전송 레이어에서 온 '어떤 것'이 애플리케이션 레이어에서 EventSource API에 의해 '무엇'으로 변하는가?"

**답변**:

- **전송 레이어에서 온 것** = HTTP 청킹이 제거된 순수한 UTF-8 텍스트 스트림
- **EventSource API가 변환한 결과** = JavaScript의 MessageEvent 객체

#### 구체적인 변환 예시

**입력 (텍스트 스트림)**:

```
event: userJoin\n
data: {"userId": 123}\n
data: {"name": "김철수"}\n
id: evt-001\n
retry: 5000\n
\n
```

**출력 (Event 객체)**:

```javascript
{
  type: "userJoin",              // event: 필드 → type 속성
  data: '{"userId": 123}\n{"name": "김철수"}',  // data: 필드들 → data 속성
  lastEventId: "evt-001",        // id: 필드 → lastEventId 속성
  origin: "https://example.com",
  source: null,
  ports: []
  // retry는 Event 객체에 포함되지 않고 재연결 설정에 사용됨
}
```

## 3. 청킹의 역할과 위치

### 청킹은 "포장지" 개념

```
원본 편지 (SSE 텍스트)
    ↓
봉투에 넣기 (HTTP 청킹 추가)
    ↓
우편 배송 (네트워크 전송)
    ↓
봉투 제거 (HTTP 청킹 제거)
    ↓
원본 편지 읽기 (EventSource 파싱)
```

### 시간 순서로 본 데이터 변환

**T1: 서버에서 SSE 이벤트 생성**

```javascript
res.write('data: {"temp": 25}\n\n');
```

**T2: HTTP 레이어가 청킹 추가**

```
12\r\ndata: {"temp": 25}\n\n\r\n
```

**T3: 네트워크 전송**

```
0x31 0x32 0x0D 0x0A 0x64 0x61 0x74 0x61 ...
```

**T4: 브라우저가 청킹 제거**

```
data: {"temp": 25}\n\n
```

**T5: EventSource가 이벤트로 변환**

```javascript
{
  type: "message",
  data: '{"temp": 25}',
  lastEventId: "",
  origin: "https://api.example.com"
}
```

## 4. 청킹이 필요한 이유

### HTTP/1.1의 근본적인 제약: Content-Length 문제

**일반적인 HTTP 응답 (크기를 아는 경우)**:

```http
HTTP/1.1 200 OK
Content-Length: 1234  ← 미리 전체 크기를 알려줘야 함
Content-Type: text/html

<html>...</html>
```

**SSE의 딜레마**:

```http
HTTP/1.1 200 OK
Content-Length: ???  ← 몇 시간 동안 이벤트를 보낼지 모름!
Content-Type: text/event-stream

data: 실시간 이벤트들...
```

### 청킹: "크기를 모를 때의 해결책"

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked  ← Content-Length 대신 사용
Content-Type: text/event-stream

5\r\n
data:\r\n
C\r\n
첫 이벤트\n\n\r\n
0\r\n
\r\n
```

### 청킹을 사용하는 구체적인 이유

#### 1. 실시간 전송 가능

```javascript
// 청킹 없이는 불가능한 시나리오
setInterval(() => {
  res.write(`data: ${new Date()}\n\n`); // 매초 전송
}, 1000);
// 언제 끝날지 모르므로 Content-Length 설정 불가
```

#### 2. 메모리 효율성

```javascript
// 나쁜 예: 모든 데이터를 모아서 전송
let allData = '';
for (let i = 0; i < 1000000; i++) {
  allData += `data: 이벤트 ${i}\n\n`;
}
res.end(allData); // 메모리 폭발!

// 좋은 예: 청킹으로 즉시 전송
for (let i = 0; i < 1000000; i++) {
  res.write(`data: 이벤트 ${i}\n\n`); // 즉시 전송
}
```

#### 3. 연결 유지 확인

```
10\r\n[데이터]\r\n  ← 클라이언트가 받으면 연결 정상
10\r\n[데이터]\r\n  ← 계속 주고받으며 연결 확인
```

### HTTP 버전별 스트리밍 방식

| HTTP 버전 | 스트리밍 방식              | 설명                       |
| --------- | -------------------------- | -------------------------- |
| HTTP/1.1  | Transfer-Encoding: chunked | 청킹 필수, CRLF로 구분     |
| HTTP/2    | DATA 프레임                | 청킹 불필요, 프레임 기반   |
| HTTP/3    | QUIC 스트림                | 청킹 불필요, QUIC 프로토콜 |

## 5. 실제 구현 예시

### 서버 측 구현 (Node.js/Express)

```javascript
app.get('/events', (req, res) => {
  // HTTP/1.1에서는 chunked 자동 설정
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no', // 프록시 버퍼링 비활성화
  });

  // 초기 연결 확인
  res.write('data: 연결됨\n\n');

  // 실시간 이벤트 전송
  const interval = setInterval(() => {
    const event = {
      timestamp: new Date().toISOString(),
      value: Math.random(),
    };

    res.write(`data: ${JSON.stringify(event)}\n\n`);
  }, 2000);

  // 클라이언트 연결 해제 처리
  req.on('close', () => {
    clearInterval(interval);
  });
});
```

### 클라이언트 측 구현 (JavaScript)

```javascript
const eventSource = new EventSource('/events');

// 연결 성공
eventSource.onopen = function (event) {
  console.log('SSE 연결 성공');
};

// 메시지 수신
eventSource.onmessage = function (event) {
  // event.data는 서버가 보낸 순수한 데이터
  // HTTP 청킹 정보는 이미 제거된 상태
  const data = JSON.parse(event.data);
  console.log('수신:', data);
};

// 커스텀 이벤트 처리
eventSource.addEventListener('userJoin', function (event) {
  console.log('사용자 참여:', event.data);
});

// 에러 처리
eventSource.onerror = function (event) {
  if (eventSource.readyState === EventSource.CLOSED) {
    console.log('연결 종료됨');
  } else {
    console.log('연결 에러, 재연결 시도 중...');
  }
};
```

## 핵심 요약

1. **SSE는 두 개의 완전히 독립적인 레이어에서 작동**

   - 전송 레이어: HTTP 청킹 (네트워크 전송 메커니즘)
   - 애플리케이션 레이어: SSE 프로토콜 (이벤트 스트림)

2. **청킹은 SSE와 무관한 HTTP/1.1의 스트리밍 메커니즘**

   - SSE가 아닌 다른 스트리밍에서도 사용
   - HTTP/2, HTTP/3에서는 다른 방식 사용

3. **개발자는 SSE 프로토콜만 신경쓰면 됨**

   - 서버: `data:`, `event:`, `id:` 등의 SSE 형식
   - 클라이언트: EventSource API 사용
   - HTTP 청킹은 프레임워크가 자동 처리

4. **청킹이 필요한 이유는 HTTP/1.1의 제약 때문**
   - Content-Length를 미리 알 수 없는 무한 스트림
   - 실시간 전송과 메모리 효율성 확보
