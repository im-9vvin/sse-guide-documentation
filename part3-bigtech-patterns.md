***
**[← 이전: Part 2 - 작동 원리와 청킹 메커니즘](./part2-working-principles.md)** | **[목차](./README.md)** | **[다음: Part 4 - AI 챗봇 서비스의 SSE 구현 →](./part4-ai-chatbot-sse.md)**

***

# Part 3: 빅테크의 비공식 SSE 패턴

## 4장. 빅테크의 비공식 SSE 패턴

W3C 표준은 `data`, `event`, `id`, `retry` 필드만을 정의하지만, 실제로 주요 기술 기업들은 다양한 커스텀 이벤트 타입과 패턴을 사용합니다. 이 장에서는 실제 프로덕션 환경에서 널리 사용되는 비공식 패턴들을 살펴봅니다.

### 4.1 연결 유지 패턴 (Keep-Alive)

#### 4.1.1 Ping/Heartbeat 패턴

많은 서비스가 연결 상태를 유지하고 프록시 타임아웃을 방지하기 위해 주기적인 ping 메시지를 전송합니다.

**GitHub Actions 예제:**
```
event: ping
data: {"timestamp": "2024-01-15T10:30:00Z"}

: heartbeat

event: heartbeat
data: {"status": "alive", "uptime": 3600}
```

**구현 예제:**
```javascript
class SSEConnection {
  constructor(response) {
    this.response = response;
    this.pingInterval = 30000; // 30초
    this.startHeartbeat();
  }
  
  startHeartbeat() {
    this.heartbeatTimer = setInterval(() => {
      // 주석 형태의 heartbeat
      this.response.write(': heartbeat\n\n');
      
      // 또는 이벤트 형태
      this.response.write('event: ping\n');
      this.response.write(`data: ${Date.now()}\n\n`);
    }, this.pingInterval);
  }
  
  send(data, eventType) {
    if (eventType) {
      this.response.write(`event: ${eventType}\n`);
    }
    this.response.write(`data: ${JSON.stringify(data)}\n\n`);
  }
  
  close() {
    clearInterval(this.heartbeatTimer);
    this.response.end();
  }
}
```

#### 4.1.2 Keep-Alive 메타데이터

**Vercel 배포 모니터링:**
```
event: keep-alive
data: {"connectionId": "abc123", "duration": 120, "nextPing": 30}

event: connection-status
data: {"active": true, "latency": 15, "region": "us-east-1"}
```

### 4.2 스트림 제어 패턴

#### 4.2.1 시작/종료 신호

**OpenAI 스타일:**
```
event: stream-start
data: {"id": "chatcmpl-123", "model": "gpt-4", "created": 1704094800}

event: message
data: {"content": "스트리밍 응답 내용..."}

event: stream-end
data: {"finish_reason": "stop", "usage": {"total_tokens": 150}}
```

**구현 예제:**
```javascript
class StreamController {
  constructor(response) {
    this.response = response;
    this.streamId = generateId();
    this.startTime = Date.now();
  }
  
  async start(metadata = {}) {
    this.send({
      streamId: this.streamId,
      started: this.startTime,
      ...metadata
    }, 'stream-start');
  }
  
  async sendChunk(data) {
    this.send(data, 'message');
  }
  
  async end(reason = 'complete') {
    const duration = Date.now() - this.startTime;
    this.send({
      streamId: this.streamId,
      reason,
      duration,
      ended: Date.now()
    }, 'stream-end');
    
    this.response.end();
  }
  
  send(data, eventType) {
    if (eventType) {
      this.response.write(`event: ${eventType}\n`);
    }
    this.response.write(`data: ${JSON.stringify(data)}\n\n`);
  }
}
```

#### 4.2.2 일시정지/재개 패턴

**비디오 스트리밍 서비스 예제:**
```
event: stream-pause
data: {"reason": "buffer_full", "position": 1024}

event: stream-resume
data: {"position": 1024, "buffered": true}

event: stream-throttle
data: {"rate": 0.5, "reason": "network_congestion"}
```

### 4.3 상태 관리 패턴

#### 4.3.1 상태 업데이트

**배포 파이프라인 예제:**
```
event: status
data: {"state": "initializing", "step": 1, "total": 5}

event: status
data: {"state": "building", "step": 2, "total": 5}

event: status
data: {"state": "testing", "step": 3, "total": 5}

event: status
data: {"state": "deploying", "step": 4, "total": 5}

event: status
data: {"state": "complete", "step": 5, "total": 5}
```

**구현 예제:**
```javascript
class DeploymentStream {
  constructor(response) {
    this.response = response;
    this.states = ['initializing', 'building', 'testing', 'deploying', 'complete'];
    this.currentStep = 0;
  }
  
  async updateStatus(metadata = {}) {
    const state = this.states[this.currentStep];
    
    this.send({
      state,
      step: this.currentStep + 1,
      total: this.states.length,
      timestamp: new Date().toISOString(),
      ...metadata
    }, 'status');
    
    this.currentStep++;
  }
  
  async setCustomStatus(state, data = {}) {
    this.send({
      state,
      custom: true,
      timestamp: new Date().toISOString(),
      ...data
    }, 'status');
  }
  
  send(data, eventType) {
    this.response.write(`event: ${eventType}\n`);
    this.response.write(`data: ${JSON.stringify(data)}\n\n`);
  }
}
```

#### 4.3.2 상태 전환 이벤트

**워크플로우 엔진 예제:**
```
event: state-change
data: {"from": "pending", "to": "running", "trigger": "manual"}

event: state-transition
data: {
  "workflow": "ci-pipeline",
  "previousState": "testing",
  "currentState": "deploying",
  "duration": 145000,
  "success": true
}
```

### 4.4 진행 상황 패턴

#### 4.4.1 백분율 진행률

**파일 업로드 예제:**
```
event: progress
data: {"percent": 0, "bytes": 0, "total": 1048576}

event: progress
data: {"percent": 25, "bytes": 262144, "total": 1048576}

event: progress
data: {"percent": 50, "bytes": 524288, "total": 1048576}

event: progress
data: {"percent": 100, "bytes": 1048576, "total": 1048576}

event: complete
data: {"fileId": "file123", "size": 1048576, "duration": 2500}
```

**구현 예제:**
```javascript
class ProgressStream {
  constructor(response, total) {
    this.response = response;
    this.total = total;
    this.current = 0;
    this.startTime = Date.now();
  }
  
  update(processed) {
    this.current += processed;
    const percent = Math.round((this.current / this.total) * 100);
    
    this.send({
      percent,
      current: this.current,
      total: this.total,
      rate: this.calculateRate()
    }, 'progress');
    
    if (this.current >= this.total) {
      this.complete();
    }
  }
  
  calculateRate() {
    const elapsed = Date.now() - this.startTime;
    return elapsed > 0 ? Math.round(this.current / (elapsed / 1000)) : 0;
  }
  
  complete() {
    const duration = Date.now() - this.startTime;
    this.send({
      total: this.total,
      duration,
      averageRate: this.calculateRate()
    }, 'complete');
  }
  
  send(data, eventType) {
    this.response.write(`event: ${eventType}\n`);
    this.response.write(`data: ${JSON.stringify(data)}\n\n`);
  }
}
```

#### 4.4.2 단계별 진행률

**멀티스텝 프로세스 예제:**
```
event: step-progress
data: {
  "currentStep": "데이터 검증",
  "stepNumber": 1,
  "totalSteps": 4,
  "stepProgress": 100,
  "overallProgress": 25
}

event: step-progress
data: {
  "currentStep": "데이터 변환",
  "stepNumber": 2,
  "totalSteps": 4,
  "stepProgress": 50,
  "overallProgress": 37.5
}
```

### 4.5 에러 처리 패턴

#### 4.5.1 에러 이벤트

**구조화된 에러 메시지:**
```
event: error
data: {
  "code": "RATE_LIMIT_EXCEEDED",
  "message": "API 호출 한도 초과",
  "details": {
    "limit": 100,
    "used": 100,
    "resetAt": "2024-01-15T11:00:00Z"
  },
  "recoverable": true
}

event: warning
data: {
  "code": "SLOW_NETWORK",
  "message": "네트워크 속도가 느립니다",
  "suggestion": "연결 상태를 확인하세요"
}
```

**구현 예제:**
```javascript
class ErrorAwareStream {
  constructor(response) {
    this.response = response;
    this.errorCount = 0;
  }
  
  sendError(error, recoverable = true) {
    this.errorCount++;
    
    const errorData = {
      code: error.code || 'UNKNOWN_ERROR',
      message: error.message,
      timestamp: new Date().toISOString(),
      errorCount: this.errorCount,
      recoverable
    };
    
    if (error.details) {
      errorData.details = error.details;
    }
    
    this.send(errorData, 'error');
    
    if (!recoverable) {
      this.send({ reason: 'fatal_error' }, 'stream-end');
      this.response.end();
    }
  }
  
  sendWarning(code, message, data = {}) {
    this.send({
      code,
      message,
      level: 'warning',
      timestamp: new Date().toISOString(),
      ...data
    }, 'warning');
  }
  
  send(data, eventType) {
    this.response.write(`event: ${eventType}\n`);
    this.response.write(`data: ${JSON.stringify(data)}\n\n`);
  }
}
```

#### 4.5.2 재시도 힌트

**서버 과부하 처리:**
```
event: retry-hint
data: {"suggested": 5000, "reason": "server_busy"}

event: backoff
data: {"attempt": 3, "nextRetry": 8000, "maxAttempts": 5}
```

### 4.6 메타데이터 패턴

#### 4.6.1 시스템 정보

**서버 상태 브로드캐스트:**
```
event: system-info
data: {
  "serverId": "prod-us-east-1a",
  "version": "2.5.0",
  "uptime": 864000,
  "connections": 1523
}

event: performance-metrics
data: {
  "cpu": 45.2,
  "memory": 78.5,
  "activeStreams": 523,
  "queueLength": 12
}
```

#### 4.6.2 디버그 정보

**개발 환경 디버깅:**
```
event: debug
data: {
  "action": "query_executed",
  "duration": 123,
  "query": "SELECT * FROM users WHERE active = true",
  "affectedRows": 42
}

event: trace
data: {
  "requestId": "req-123",
  "spanId": "span-456",
  "operation": "database_query",
  "duration": 15
}
```

### 4.7 인증/세션 패턴

#### 4.7.1 토큰 갱신

**JWT 토큰 자동 갱신:**
```
event: token-refresh
data: {
  "newToken": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "expiresIn": 3600,
  "refreshToken": "refresh_token_here"
}

event: session-update
data: {
  "sessionId": "sess_123",
  "remainingTime": 1800,
  "autoExtended": true
}
```

**구현 예제:**
```javascript
class AuthenticatedStream {
  constructor(response, session) {
    this.response = response;
    this.session = session;
    this.startTokenRefreshTimer();
  }
  
  startTokenRefreshTimer() {
    // 토큰 만료 5분 전에 갱신
    const refreshTime = (this.session.expiresIn - 300) * 1000;
    
    this.tokenTimer = setTimeout(async () => {
      try {
        const newToken = await this.refreshToken();
        
        this.send({
          newToken: newToken.access_token,
          expiresIn: newToken.expires_in,
          refreshToken: newToken.refresh_token
        }, 'token-refresh');
        
        // 다음 갱신 예약
        this.startTokenRefreshTimer();
      } catch (error) {
        this.send({
          error: 'Token refresh failed',
          code: 'AUTH_REFRESH_FAILED'
        }, 'auth-error');
      }
    }, refreshTime);
  }
  
  async refreshToken() {
    // 토큰 갱신 로직
    return await authService.refresh(this.session.refreshToken);
  }
  
  send(data, eventType) {
    this.response.write(`event: ${eventType}\n`);
    this.response.write(`data: ${JSON.stringify(data)}\n\n`);
  }
  
  close() {
    clearTimeout(this.tokenTimer);
    this.response.end();
  }
}
```

#### 4.7.2 권한 변경 알림

**실시간 권한 업데이트:**
```
event: permission-change
data: {
  "userId": "user123",
  "changes": {
    "added": ["write", "delete"],
    "removed": ["admin"],
    "current": ["read", "write", "delete"]
  }
}

event: access-granted
data: {
  "resource": "/api/premium-features",
  "level": "full",
  "validUntil": "2024-12-31T23:59:59Z"
}
```

### 4.8 실제 기업 사례 종합

#### 4.8.1 OpenAI ChatGPT 패턴
```javascript
// 스트림 시작
`event: message_start
data: {"id": "msg_123", "role": "assistant"}

// 콘텐츠 델타
event: content_block_delta
data: {"delta": {"text": "안녕"}, "index": 0}

// 스트림 종료
event: message_stop
data: {"stop_reason": "end_turn"}`
```

#### 4.8.2 GitHub Actions 패턴
```javascript
// 작업 시작
`event: workflow_run
data: {"action": "started", "workflow": "CI Pipeline"}

// 단계 진행
event: job_step
data: {"name": "Run Tests", "status": "in_progress", "number": 3}

// 로그 스트리밍
event: log_line
data: {"line": "All tests passed!", "timestamp": "2024-01-15T10:30:00Z"}`
```

#### 4.8.3 Vercel 배포 패턴
```javascript
// 빌드 진행
`event: build-state
data: {"state": "building", "progress": 45}

// 함수 생성
event: creating-functions
data: {"functions": ["api/hello", "api/world"], "count": 2}

// 배포 완료
event: ready
data: {"url": "https://my-app.vercel.app", "duration": 35000}`
```

***

**다음 장 예고**: [Part 4: AI 챗봇 서비스의 SSE 구현](./part4-ai-chatbot-sse.md)에서는 OpenAI, Anthropic 등 AI 서비스들의 구체적인 SSE 구현 패턴과 스트리밍 응답 처리 방법을 자세히 살펴보겠습니다.

***
**[← 이전: Part 2 - 작동 원리와 청킹 메커니즘](./part2-working-principles.md)** | **[목차](./README.md)** | **[다음: Part 4 - AI 챗봇 서비스의 SSE 구현 →](./part4-ai-chatbot-sse.md)**

***