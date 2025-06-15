# 빅테크 기업들의 SSE 비공식 이벤트 타입과 패턴 가이드

## 개요

Server-Sent Events(SSE)의 W3C 표준은 `data`, `event`, `id`, `retry` 네 가지 필드만을 정의하지만, 실제 프로덕션 환경에서는 다양한 커스텀 이벤트 타입과 패턴이 사용됩니다. 이 문서는 주요 빅테크 기업들이 사용하는 비공식 SSE 패턴을 정리합니다.

## 1. 연결 유지 패턴 (Keep-Alive Patterns)

### ping

가장 널리 사용되는 연결 확인 패턴입니다.

```
event: ping
data: {"timestamp": 1704067200}

// 또는 단순 형태
data: ping
```

**사용 예시:**

- GitHub Actions 실시간 로그
- Vercel 배포 상태 스트리밍
- ChatGPT 대화 연결 유지

### heartbeat

주기적인 연결 상태 확인에 사용됩니다.

```
event: heartbeat
data: {"type": "heartbeat", "interval": 30000, "sequence": 1}

// 주석 형태 (Play Framework, Express.js)
: heartbeat

// 빈 데이터 형태
event: heartbeat
data:
```

**Twitter/X 구현:**

```
// 10-20초마다 전송
\r\n

// 또는
: keep-alive signal
```

### keep-alive

연결 타임아웃 방지를 위한 신호입니다.

```
event: keep-alive
data: {"timestamp": "2024-01-01T00:00:00Z"}

// Nginx 프록시 통과용
: keep-alive
```

## 2. 스트림 제어 패턴 (Stream Control Patterns)

### start/init

스트림 시작과 초기화를 알립니다.

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

**Shopify Live Map 예시:**

```
event: stream-start
data: {
  "session_id": "bfcm-2024",
  "region": "north-america",
  "shard": "na-east-1"
}
```

### done/complete

스트림 종료를 알립니다.

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

### ready

클라이언트가 데이터를 받을 준비가 되었음을 알립니다.

```
event: ready
data: {"buffer_size": 1000, "client_version": "2.1.0"}
```

## 3. 상태 관리 패턴 (State Management Patterns)

### status

현재 시스템 또는 연결 상태를 전달합니다.

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

### system

시스템 레벨 메시지를 전달합니다.

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

### config

런타임 설정 변경을 알립니다.

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

## 4. 진행 상황 패턴 (Progress Patterns)

### progress

작업 진행 상황을 보고합니다.

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

### update

증분 업데이트를 전달합니다.

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

### checkpoint

중간 저장 지점을 표시합니다.

```
event: checkpoint
data: {
  "id": "ckpt-123",
  "position": 5000,
  "can_resume": true,
  "state_hash": "abc123..."
}
```

## 5. 에러 처리 패턴 (Error Handling Patterns)

### error

일반적인 에러 메시지입니다.

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

### warning

경고 메시지입니다.

```
event: warning
data: {
  "code": "APPROACHING_LIMIT",
  "message": "80% of rate limit consumed",
  "threshold": 0.8
}
```

### operational-disconnect

Twitter/X의 운영상 연결 해제 패턴입니다.

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

## 6. 메타데이터 패턴 (Metadata Patterns)

### meta/metadata

스트림 관련 메타데이터를 전송합니다.

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

### info

추가 정보를 제공합니다.

```
event: info
data: {
  "server_region": "us-east-1",
  "latency_ms": 23,
  "connected_since": "2024-01-01T00:00:00Z"
}
```

## 7. 인증/세션 패턴 (Auth/Session Patterns)

### auth

인증 관련 이벤트입니다.

```
event: auth
data: {
  "status": "authenticated",
  "user_id": "user-123",
  "expires_at": "2024-01-02T00:00:00Z",
  "refresh_required": false
}
```

### session

세션 관련 정보입니다.

```
event: session
data: {
  "session_id": "sess-abc123",
  "created_at": "2024-01-01T00:00:00Z",
  "expires_in": 3600,
  "renewable": true
}
```

## 8. 실제 구현 예시

### OpenAI ChatGPT

```javascript
// 스트리밍 시작
data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","choices":[{"delta":{"role":"assistant"}}]}

// 콘텐츠 스트리밍
data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","choices":[{"delta":{"content":"Hello"}}]}

// 완료
data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","choices":[{"finish_reason":"stop"}]}
data: [DONE]
```

### GitHub Actions

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

### Vercel 배포

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

## 9. 복합 패턴 예시

### 실시간 분석 대시보드

```javascript
// 초기화
event: init
data: {"dashboard_id": "analytics-2024", "refresh_rate": 5000}

// 주기적 업데이트
event: metrics
data: {
  "timestamp": "2024-01-01T00:00:00Z",
  "visitors": 1523,
  "page_views": 4521,
  "avg_duration": 156.7
}

// 알림
event: alert
data: {
  "type": "spike_detected",
  "metric": "error_rate",
  "value": 0.05,
  "threshold": 0.01
}

// Heartbeat
: heartbeat

// 종료
event: done
data: {"reason": "client_disconnect"}
```

## 10. 모범 사례

### 이벤트 네이밍 규칙

- 소문자와 하이픈 사용: `user-connected`, `stream-start`
- 동사-명사 형태: `update-progress`, `send-notification`
- 네임스페이스 사용: `auth:login`, `db:update`

### 데이터 형식

- JSON 사용 (파싱 용이성)
- ISO 8601 타임스탬프
- 일관된 필드명
- 버전 정보 포함

### 에러 처리

- 명확한 에러 코드
- 재시도 정보 포함
- 사용자 친화적 메시지

### 연결 관리

- 30-60초 간격 heartbeat
- 자동 재연결 로직
- 백오프 전략
- 연결 상태 모니터링

## 결론

이러한 비공식 패턴들은 SSE의 기본 기능을 확장하여 복잡한 실시간 애플리케이션 구축을 가능하게 합니다. 각 기업과 서비스의 요구사항에 따라 적절한 패턴을 선택하고 조합하여 사용하면, 안정적이고 확장 가능한 실시간 통신 시스템을 구축할 수 있습니다.
