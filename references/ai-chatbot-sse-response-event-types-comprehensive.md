# AI 챗봇 서비스의 SSE 응답 이벤트 유형 총정리

## 1. W3C SSE 표준 필드

### 공식 표준 필드 (서버→클라이언트)

- **`data:`** - 실제 이벤트 데이터
- **`event:`** - 이벤트 타입 이름 (기본값: "message")
- **`id:`** - 재연결을 위한 이벤트 ID
- **`retry:`** - 재연결 시간 설정 (밀리초)
- **`:`** - 주석 (무시됨)

### 구분 규칙

- **필드 구분**: 단일 개행 (`\n`)
- **이벤트 구분**: 이중 개행 (`\n\n`)

## 2. OpenAI ChatGPT SSE 메시지 형식

### 2.1 일반 텍스트 응답

```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","created":1679168243,"model":"gpt-4","choices":[{"delta":{"content":"안녕하세요"},"index":0,"finish_reason":null}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","created":1679168243,"model":"gpt-4","choices":[{"delta":{"content":" 무엇을"},"index":0,"finish_reason":null}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","created":1679168243,"model":"gpt-4","choices":[{"delta":{},"index":0,"finish_reason":"stop"}]}

data: [DONE]
```

#### 주요 필드 구조

- `id`: 대화 완성 ID
- `object`: "chat.completion.chunk"
- `created`: 타임스탬프
- `model`: 사용된 모델명
- `choices`: 응답 선택지 배열
  - `delta`: 스트리밍된 콘텐츠 조각
    - `content`: 실제 텍스트 내용
    - `role`: 역할 (첫 청크에만 "assistant")
  - `index`: 선택지 인덱스
  - `finish_reason`: 완료 사유 (stop, length, content_filter, function_call 등)

### 2.2 Tool Calling (함수 호출)

#### Tool Call 시작

```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","created":1721075653,"model":"gpt-4","choices":[{"delta":{"role":"assistant","content":"","tool_calls":[{"index":0,"id":"call_abc123","type":"function","function":{"name":"get_weather","arguments":""}}]},"index":0,"finish_reason":null}]}
```

#### Tool Call 인자 스트리밍

```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","created":1721075653,"model":"gpt-4","choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"{\""}}]},"index":0,"finish_reason":null}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","created":1721075653,"model":"gpt-4","choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"location"}}]},"index":0,"finish_reason":null}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","created":1721075653,"model":"gpt-4","choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"\": \"서울\"}"}}]},"index":0,"finish_reason":null}]}
```

#### 병렬 Tool Calls (parallel_tool_calls=true)

```
data: {"choices":[{"delta":{"tool_calls":[{"index":0,"id":"call_123","type":"function","function":{"name":"search","arguments":""}},{"index":1,"id":"call_456","type":"function","function":{"name":"calculate","arguments":""}}]}}]}
```

### 2.3 구조화된 출력 (Structured Output)

```
data: {"choices":[{"delta":{"content":"{\"name\": \""},"index":0}]}
data: {"choices":[{"delta":{"content":"홍길동"},"index":0}]}
data: {"choices":[{"delta":{"content":"\", \"age\": "},"index":0}]}
data: {"choices":[{"delta":{"content":"30"},"index":0}]}
data: {"choices":[{"delta":{"content":"}"},"index":0}]}
```

### 2.4 에러 및 특수 상황

#### 콘텐츠 필터링 (Azure OpenAI)

```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","created":1721075653,"model":"gpt-4","choices":[],"content_filter_results":{"hate":{"filtered":false}}}
```

#### 로그 확률 (Log Probabilities)

```
data: {"choices":[{"delta":{"content":"안녕","logprobs":{"tokens":["안녕"],"token_logprobs":[-0.123],"top_logprobs":[{"안녕":-0.123,"hello":-2.456}]}},"index":0}]}
```

### 2.5 Assistant API 스트리밍 이벤트

```
event: thread.run.created
data: {"id":"run_xxx","object":"thread.run","status":"queued"}

event: thread.run.in_progress
data: {"id":"run_xxx","object":"thread.run","status":"in_progress"}

event: thread.message.delta
data: {"id":"msg_xxx","delta":{"content":[{"index":0,"type":"text","text":{"value":"응답"}}]}}

event: thread.run.completed
data: {"id":"run_xxx","object":"thread.run","status":"completed"}
```

## 3. Anthropic Claude SSE 메시지 형식

### 3.1 기본 메시지 스트리밍

```
event: message_start
data: {"type": "message_start", "message": {"id": "msg_01...", "type": "message", "role": "assistant", "content": [], "model": "claude-3-opus-20240229", "stop_reason": null, "stop_sequence": null}}

event: content_block_start
data: {"type": "content_block_start", "index": 0, "content_block": {"type": "text", "text": ""}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "안녕하세요"}}

event: content_block_stop
data: {"type": "content_block_stop", "index": 0}

event: message_delta
data: {"type": "message_delta", "delta": {"stop_reason": "end_turn", "stop_sequence": null}}

event: message_stop
data: {"type": "message_stop"}
```

### 3.2 Thinking/Reasoning 스트리밍

```
event: content_block_start
data: {"type": "content_block_start", "index": 0, "content_block": {"type": "thinking", "thinking": ""}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "thinking_delta", "thinking": "Let me solve this step by step:\n\n1. First break down 27 * 453"}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "thinking_delta", "thinking": "\n2. 453 = 400 + 50 + 3"}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "thinking_delta", "thinking": "\n3. 27 * 400 = 10,800"}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "signature_delta", "signature": "EqQBCgIYAhIM1gbcDa9GJwZA2b3hGgxBdjrkzLoky3dl1pkiMOYds..."}}

event: content_block_stop
data: {"type": "content_block_stop", "index": 0}
```

### 3.3 Tool Use

```
event: content_block_start
data: {"type": "content_block_start", "index": 1, "content_block": {"type": "tool_use", "id": "toolu_xxx", "name": "get_weather", "input": {}}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 1, "delta": {"type": "input_json_delta", "partial_json": "{\"location\":"}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 1, "delta": {"type": "input_json_delta", "partial_json": " \"서울\"}"}}

event: content_block_stop
data: {"type": "content_block_stop", "index": 1}
```

## 4. 빅테크 기업들의 비공식 SSE 이벤트 패턴

### 4.1 연결 유지 패턴 (Keep-Alive)

#### ping

```
event: ping
data: {"timestamp": 1704067200}

// 또는 단순 형태
data: ping
```

#### heartbeat

```
event: heartbeat
data: {"type": "heartbeat", "interval": 30000, "sequence": 1}

// 주석 형태
: heartbeat

// 빈 데이터 형태
event: heartbeat
data:
```

#### keep-alive

```
event: keep-alive
data: {"timestamp": "2024-01-01T00:00:00Z"}

// Twitter/X 스타일 (10-20초마다)
\r\n

// 또는
: keep-alive
```

### 4.2 스트림 제어 패턴

#### start/init

```
event: start
data: {
  "stream_id": "abc-123",
  "version": "2.0",
  "created_at": "2024-01-01T00:00:00Z",
  "capabilities": ["compression", "binary", "multiplexing"]
}

event: init
data: {
  "protocol": "sse/1.0",
  "encoding": "utf-8",
  "compression": "gzip",
  "max_reconnect_attempts": 5
}
```

#### done/complete

```
// OpenAI 스타일
data: [DONE]

// 이벤트 타입 사용
event: done
data: {"status": "completed", "total_events": 1500, "duration_ms": 45000}

event: complete
data: {
  "reason": "end_of_stream",
  "stats": {
    "messages_sent": 150,
    "bytes_transferred": 45678,
    "connection_duration": "PT5M30S"
  }
}
```

#### ready

```
event: ready
data: {"buffer_size": 1000, "client_version": "2.1.0"}
```

### 4.3 상태 관리 패턴

#### status

```
event: status
data: {
  "connection": "active",
  "health": "healthy",
  "queue_depth": 0,
  "active_connections": 1523,
  "server_time": "2024-01-01T00:00:00Z"
}
```

#### system

```
event: system
data: {
  "message": "Scheduled maintenance at 02:00 UTC",
  "severity": "warning",
  "action_required": false
}

// Twitter/X 스타일
event: system
data: {
  "message": "Rate limit approaching",
  "current": 4500,
  "limit": 5000,
  "reset_at": "2024-01-01T01:00:00Z"
}
```

#### config

```
event: config
data: {
  "rate_limit": 100,
  "batch_size": 50,
  "compression": true,
  "features": {
    "realtime_updates": true,
    "binary_support": false
  }
}
```

### 4.4 진행 상황 패턴

#### progress

```
event: progress
data: {
  "task_id": "export-123",
  "percentage": 45,
  "current": 4500,
  "total": 10000,
  "eta_seconds": 120,
  "message": "Processing records..."
}
```

#### update

```
event: update
data: {
  "type": "incremental",
  "delta": {
    "added": 5,
    "removed": 2,
    "modified": 3
  },
  "timestamp": "2024-01-01T00:00:00Z"
}
```

#### checkpoint

```
event: checkpoint
data: {
  "id": "ckpt-123",
  "position": 5000,
  "can_resume": true,
  "state_hash": "abc123..."
}
```

### 4.5 에러 처리 패턴

#### error

```
event: error
data: {
  "code": "RATE_LIMIT_EXCEEDED",
  "message": "Too many requests",
  "retry_after": 60,
  "details": {
    "limit": 100,
    "window": "1m",
    "current": 150
  }
}
```

#### warning

```
event: warning
data: {
  "code": "APPROACHING_LIMIT",
  "message": "80% of rate limit consumed",
  "threshold": 0.8
}
```

#### operational-disconnect

```
event: operational-disconnect
data: {
  "title": "operational-disconnect",
  "disconnect_type": "UpstreamOperationalDisconnect",
  "detail": "This stream has been disconnected upstream for operational reasons",
  "reconnect": true,
  "backoff_ms": 5000
}
```

### 4.6 메타데이터 패턴

#### meta/metadata

```
event: metadata
data: {
  "stream_version": "2.0",
  "server_version": "1.5.0",
  "capabilities": ["filtering", "replay", "compression"],
  "limits": {
    "max_connections": 1000,
    "max_events_per_second": 100
  }
}
```

#### info

```
event: info
data: {
  "server_region": "us-east-1",
  "latency_ms": 23,
  "connected_since": "2024-01-01T00:00:00Z"
}
```

### 4.7 인증/세션 패턴

#### auth

```
event: auth
data: {
  "status": "authenticated",
  "user_id": "user-123",
  "expires_at": "2024-01-02T00:00:00Z",
  "refresh_required": false
}
```

#### session

```
event: session
data: {
  "session_id": "sess-abc123",
  "created_at": "2024-01-01T00:00:00Z",
  "expires_in": 3600,
  "renewable": true
}
```

## 5. 실제 구현 예시

### 5.1 GitHub Actions

```javascript
// 작업 시작
event: job-start
data: {"job_id": "123", "name": "Build", "runner": "ubuntu-latest"}

// 로그 스트리밍
event: log
data: {"timestamp": "2024-01-01T00:00:00Z", "level": "info", "message": "Building..."}

// 진행 상황
event: step-complete
data: {"step": "build", "status": "success", "duration_ms": 1234}

// 완료
event: job-complete
data: {"job_id": "123", "status": "success", "total_duration_ms": 5678}
```

### 5.2 Vercel 배포

```javascript
// 배포 시작
event: deployment-created
data: {"id": "dpl_123", "url": "https://my-app-git-abc123.vercel.app"}

// 빌드 진행
event: build-progress
data: {"phase": "building", "progress": 0.45}

// 로그
event: build-log
data: {"timestamp": 1704067200, "text": "Installing dependencies..."}

// 준비 완료
event: ready
data: {"url": "https://my-app.vercel.app", "duration_ms": 45000}
```

### 5.3 Shopify Live Map

```
event: stream-start
data: {
  "session_id": "bfcm-2024",
  "region": "north-america",
  "shard": "na-east-1"
}
```

## 6. o1 모델의 Reasoning (추측)

**참고**: OpenAI o1 모델의 reasoning은 API에서 직접 볼 수 없으며, 다음은 ChatGPT UI에서의 추측되는 형식입니다.

```
event: reasoning
data: {"status": "thinking", "message": "문제를 분석하고 있습니다..."}

event: reasoning_step
data: {"step": 1, "content": "먼저 이 문제를 작은 부분으로 나누어보겠습니다"}

event: reasoning_complete
data: {"status": "complete", "token_count": 2500}
```

## 7. 핵심 특징 요약

1. **모든 데이터는 `data:` 필드에**: ChatGPT, Claude 등 모든 서비스의 복잡한 메시지는 SSE 표준의 `data:` 필드 안에 JSON으로 인코딩되어 전송

2. **이벤트 타입 활용**: 일부 서비스는 `event:` 필드를 활용하여 메시지 타입을 구분

3. **스트리밍 특성**: 대부분의 서비스가 점진적 스트리밍을 지원하여 사용자 경험 향상

4. **연결 관리**: heartbeat, ping 등을 통한 연결 상태 확인 및 유지

5. **에러 처리**: 구조화된 에러 정보와 재시도 로직 포함
