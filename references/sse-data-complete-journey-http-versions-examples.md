# SSE 데이터의 완전한 여정: HTTP 버전별 상세 예시

## 🚀 HTTP/1.1에서의 SSE 데이터 여정

### 시나리오: 주식 가격 업데이트 이벤트

**1️⃣ 서버 애플리케이션에서 작성하는 원본 코드**

```javascript
// 개발자가 작성한 서버 코드
res.write('event: stockPrice\n');
res.write('data: {"symbol":"AAPL","price":150.25}\n');
res.write('id: msg-1234\n');
res.write('\n');
```

**2️⃣ SSE 프로토콜 형식의 텍스트 스트림**

```
event: stockPrice\ndata: {"symbol":"AAPL","price":150.25}\nid: msg-1234\n\n
```

**3️⃣ HTTP/1.1 청킹 추가 (네트워크 전송 형태)**

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Transfer-Encoding: chunked
Cache-Control: no-cache

3D\r\n
event: stockPrice\ndata: {"symbol":"AAPL","price":150.25}\nid: msg-1234\n\n\r\n
```

_참고: 3D는 16진수로 61바이트를 의미_

**4️⃣ 실제 네트워크 바이트 스트림**

```
0x33 0x44 0x0D 0x0A 0x65 0x76 0x65 0x6E 0x74 0x3A 0x20 0x73 0x74 0x6F...
```

**5️⃣ 브라우저 HTTP 파서가 청킹 제거**

```
event: stockPrice\ndata: {"symbol":"AAPL","price":150.25}\nid: msg-1234\n\n
```

**6️⃣ EventSource API 파싱**

```javascript
// 최종적으로 JavaScript가 받는 Event 객체
{
  type: "stockPrice",           // event: 필드에서 추출
  data: '{"symbol":"AAPL","price":150.25}',  // data: 필드에서 추출
  lastEventId: "msg-1234",      // id: 필드에서 추출
  origin: "https://api.trading.com",
  source: null,
  ports: [],
  bubbles: false,
  cancelBubble: false,
  cancelable: false,
  composed: false,
  currentTarget: EventSource {...},
  defaultPrevented: false,
  eventPhase: 0,
  isTrusted: true,
  returnValue: true,
  srcElement: EventSource {...},
  target: EventSource {...},
  timeStamp: 1234567890,
  path: []
}
```

**7️⃣ 클라이언트 애플리케이션에서 사용**

```javascript
eventSource.addEventListener('stockPrice', function (event) {
  const data = JSON.parse(event.data);
  console.log(`${data.symbol}: $${data.price}`); // "AAPL: $150.25"
});
```

---

## 🚀 HTTP/2에서의 SSE 데이터 여정

### 동일한 시나리오: 주식 가격 업데이트

**1️⃣ 서버 애플리케이션 (동일한 코드)**

```javascript
res.write('event: stockPrice\n');
res.write('data: {"symbol":"AAPL","price":150.25}\n');
res.write('id: msg-1234\n');
res.write('\n');
```

**2️⃣ SSE 프로토콜 형식 (동일)**

```
event: stockPrice\ndata: {"symbol":"AAPL","price":150.25}\nid: msg-1234\n\n
```

**3️⃣ HTTP/2 DATA 프레임 (청킹 대신 사용)**

```
HTTP/2 200 OK
content-type: text/event-stream
cache-control: no-cache

DATA Frame (Stream ID: 5)
├─ Flags: END_STREAM=0, PADDED=0
├─ Length: 61
└─ Payload: event: stockPrice\ndata: {"symbol":"AAPL","price":150.25}\nid: msg-1234\n\n
```

**4️⃣ HTTP/2 바이너리 프레임 형식**

```
+-----------------------------------------------+
| Length (24)  | Type (8) | Flags (8) |
+---------------+----------+-----------+
| Stream Identifier (32)              |
+-------------------------------------+
| Frame Payload (가변 길이)            |
+-------------------------------------+

실제 바이트:
0x00003D 0x00 0x00 0x00000005 [페이로드 데이터...]
```

**5️⃣ 브라우저 HTTP/2 파서가 프레임 제거**

```
event: stockPrice\ndata: {"symbol":"AAPL","price":150.25}\nid: msg-1234\n\n
```

**6️⃣ EventSource API 파싱 (HTTP/1.1과 동일)**

```javascript
{
  type: "stockPrice",
  data: '{"symbol":"AAPL","price":150.25}',
  lastEventId: "msg-1234",
  // ... 나머지 속성들 동일
}
```

---

## 🚀 HTTP/3에서의 SSE 데이터 여정

**1️⃣-2️⃣ 서버 코드와 SSE 형식 (동일)**

**3️⃣ HTTP/3 QUIC 스트림**

```
HTTP/3 200 OK
content-type: text/event-stream
cache-control: no-cache

QUIC STREAM Frame
├─ Stream ID: 0x04
├─ Offset: 0
├─ Length: 61
├─ FIN: 0
└─ Data: event: stockPrice\ndata: {"symbol":"AAPL","price":150.25}\nid: msg-1234\n\n
```

**4️⃣ QUIC 패킷 형식**

```
QUIC Packet
├─ Header
│   ├─ Type: 1-RTT
│   ├─ Packet Number: 42
│   └─ ...
└─ Frames
    └─ STREAM Frame (위의 내용)
```

**5️⃣-7️⃣ 이후 과정은 동일**

---

## 📊 복잡한 시나리오: 여러 이벤트가 청크로 분할되는 경우

### HTTP/1.1에서 대용량 데이터 전송

**서버가 연속으로 보내는 이벤트들**

```javascript
// 빠르게 연속으로 전송
res.write('data: 이벤트1\n\n');
res.write('data: 이벤트2\n\n');
res.write('event: bigData\ndata: ' + veryLongData + '\n\n');
```

**HTTP/1.1 청킹 시나리오**

```
[청크 1: 작은 이벤트들이 하나로]
1C\r\n
data: 이벤트1\n\ndata: 이벤트2\n\n\r\n

[청크 2: 큰 이벤트의 시작 부분]
100\r\n
event: bigData\ndata: {"items":[{"id":1,"name":"상품1","price":...\r\n

[청크 3: 큰 이벤트의 중간 부분]
100\r\n
...12000},{"id":251,"name":"상품251","price":25100},...\r\n

[청크 4: 큰 이벤트의 끝 부분]
50\r\n
...{"id":500,"name":"상품500","price":50000}]}\n\n\r\n
```

**EventSource의 처리**

```javascript
// 청크 1 도착 → 두 개의 완전한 이벤트 감지
onmessage: { data: "이벤트1" }
onmessage: { data: "이벤트2" }

// 청크 2,3 도착 → 아직 \n\n 없음, 버퍼에 축적
// (이벤트 발생 안 함)

// 청크 4 도착 → \n\n 감지, 전체 이벤트 조합
addEventListener('bigData'): {
  data: '{"items":[{"id":1,...},{"id":500,...}]}'
}
```

---

## 🎯 핵심 포인트

1. **모든 HTTP 버전에서 SSE 텍스트 형식은 동일**

   - 서버 코드도 동일
   - EventSource API의 동작도 동일
   - 오직 전송 메커니즘만 다름

2. **청킹은 HTTP/1.1의 특징**

   - HTTP/2: DATA 프레임 사용
   - HTTP/3: QUIC STREAM 프레임 사용

3. **이벤트 경계와 전송 경계는 무관**

   - 하나의 이벤트가 여러 청크/프레임에 걸칠 수 있음
   - 여러 이벤트가 하나의 청크/프레임에 포함될 수 있음

4. **최종 결과는 항상 동일**
   - HTTP 버전과 관계없이 동일한 JavaScript Event 객체
   - 개발자는 전송 메커니즘을 신경 쓸 필요 없음
