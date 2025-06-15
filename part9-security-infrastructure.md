***
**[← 이전: Part 8 - 성능 특성과 최적화](./part8-performance-optimization.md)** | **[목차](./README.md)** | **[다음: Part 10 - 테스팅 가이드 →](./part10-testing-guide.md)**

***

# Part 9: 보안과 인프라 (Security and Infrastructure)

## 목차
- [CORS 설정](#cors-설정)
- [인증과 권한 부여](#인증과-권한-부여)
- [SSL/TLS 요구사항](#ssltls-요구사항)
- [XSS와 CSRF 방어 전략](#xss와-csrf-방어-전략)
- [로드 밸런싱 설정](#로드-밸런싱-설정)
- [모니터링과 디버깅](#모니터링과-디버깅)

## CORS 설정

### 기본 CORS 헤더 설정

```javascript
// Express.js CORS 설정
app.use((req, res, next) => {
  // SSE 엔드포인트를 위한 CORS 헤더
  if (req.path.startsWith('/events')) {
    res.setHeader('Access-Control-Allow-Origin', process.env.CLIENT_ORIGIN || '*');
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    res.setHeader('Access-Control-Allow-Methods', 'GET, OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-Requested-With');
    res.setHeader('Access-Control-Max-Age', '86400');
  }
  
  if (req.method === 'OPTIONS') {
    return res.sendStatus(204);
  }
  
  next();
});
```

### 프로덕션 CORS 전략

```javascript
class CorsManager {
  constructor(config) {
    this.allowedOrigins = config.allowedOrigins || [];
    this.credentials = config.credentials || false;
    this.maxAge = config.maxAge || 86400;
  }

  validateOrigin(origin) {
    if (!origin) return false;
    
    // 정확한 매칭
    if (this.allowedOrigins.includes(origin)) {
      return true;
    }
    
    // 와일드카드 패턴 매칭
    return this.allowedOrigins.some(allowed => {
      if (allowed.includes('*')) {
        const pattern = allowed.replace(/\*/g, '.*');
        const regex = new RegExp(`^${pattern}$`);
        return regex.test(origin);
      }
      return false;
    });
  }

  applyCorsHeaders(req, res) {
    const origin = req.headers.origin;
    
    if (this.validateOrigin(origin)) {
      res.setHeader('Access-Control-Allow-Origin', origin);
      
      if (this.credentials) {
        res.setHeader('Access-Control-Allow-Credentials', 'true');
      }
    }
    
    res.setHeader('Access-Control-Allow-Methods', 'GET, OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    res.setHeader('Access-Control-Max-Age', this.maxAge.toString());
    res.setHeader('Vary', 'Origin');
  }
}

// 사용 예제
const corsManager = new CorsManager({
  allowedOrigins: [
    'https://app.example.com',
    'https://*.example.com',
    'https://localhost:3000'
  ],
  credentials: true
});
```

## 인증과 권한 부여

### JWT 기반 SSE 인증

```javascript
class SSEAuthenticator {
  constructor(jwtSecret) {
    this.jwtSecret = jwtSecret;
    this.activeSessions = new Map();
  }

  async authenticate(req) {
    // 1. Authorization 헤더에서 토큰 추출
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      throw new Error('Missing or invalid authorization header');
    }

    const token = authHeader.substring(7);
    
    try {
      // 2. JWT 검증
      const decoded = jwt.verify(token, this.jwtSecret);
      
      // 3. 추가 검증 (만료, 권한 등)
      if (decoded.exp < Date.now() / 1000) {
        throw new Error('Token expired');
      }
      
      // 4. 세션 정보 저장
      const sessionId = crypto.randomUUID();
      this.activeSessions.set(sessionId, {
        userId: decoded.userId,
        permissions: decoded.permissions,
        connectedAt: new Date(),
        lastActivity: new Date()
      });
      
      return {
        sessionId,
        userId: decoded.userId,
        permissions: decoded.permissions
      };
    } catch (error) {
      throw new Error(`Authentication failed: ${error.message}`);
    }
  }

  validatePermissions(sessionId, requiredPermission) {
    const session = this.activeSessions.get(sessionId);
    if (!session) {
      return false;
    }
    
    session.lastActivity = new Date();
    
    return session.permissions.includes(requiredPermission) ||
           session.permissions.includes('admin');
  }

  cleanupSessions(maxInactivityMs = 30 * 60 * 1000) {
    const now = Date.now();
    
    for (const [sessionId, session] of this.activeSessions) {
      if (now - session.lastActivity.getTime() > maxInactivityMs) {
        this.activeSessions.delete(sessionId);
      }
    }
  }
}

// SSE 엔드포인트에 인증 적용
app.get('/events/:channel', async (req, res) => {
  try {
    const auth = await authenticator.authenticate(req);
    
    // 채널별 권한 검증
    const channel = req.params.channel;
    const requiredPermission = `read:${channel}`;
    
    if (!authenticator.validatePermissions(auth.sessionId, requiredPermission)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    
    // SSE 연결 설정
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'X-Session-Id': auth.sessionId
    });
    
    // 인증된 사용자에게만 이벤트 전송
    const eventStream = new AuthenticatedEventStream(auth.userId, auth.permissions);
    eventStream.pipe(res);
    
  } catch (error) {
    res.status(401).json({ error: error.message });
  }
});
```

### API 키 기반 인증

```javascript
class APIKeyAuthenticator {
  constructor(datastore) {
    this.datastore = datastore;
    this.rateLimiter = new Map();
  }

  async authenticate(req) {
    const apiKey = req.headers['x-api-key'] || req.query.apiKey;
    
    if (!apiKey) {
      throw new Error('API key required');
    }
    
    // API 키 검증
    const keyData = await this.datastore.getAPIKey(apiKey);
    if (!keyData || !keyData.active) {
      throw new Error('Invalid or inactive API key');
    }
    
    // Rate limiting 체크
    if (!this.checkRateLimit(apiKey, keyData.rateLimit)) {
      throw new Error('Rate limit exceeded');
    }
    
    return {
      apiKey,
      accountId: keyData.accountId,
      permissions: keyData.permissions,
      metadata: keyData.metadata
    };
  }

  checkRateLimit(apiKey, limit) {
    const now = Date.now();
    const window = 60 * 1000; // 1분 윈도우
    
    if (!this.rateLimiter.has(apiKey)) {
      this.rateLimiter.set(apiKey, []);
    }
    
    const requests = this.rateLimiter.get(apiKey);
    const recentRequests = requests.filter(time => now - time < window);
    
    if (recentRequests.length >= limit) {
      return false;
    }
    
    recentRequests.push(now);
    this.rateLimiter.set(apiKey, recentRequests);
    
    return true;
  }
}
```

## SSL/TLS 요구사항

### HTTPS 강제 적용

```javascript
// Express.js HTTPS 리다이렉션
app.use((req, res, next) => {
  if (req.header('x-forwarded-proto') !== 'https') {
    return res.redirect(`https://${req.header('host')}${req.url}`);
  }
  next();
});

// HSTS 헤더 설정
app.use((req, res, next) => {
  res.setHeader(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
  );
  next();
});
```

### TLS 설정 최적화

```javascript
const https = require('https');
const fs = require('fs');

// TLS 옵션 설정
const tlsOptions = {
  cert: fs.readFileSync('./ssl/cert.pem'),
  key: fs.readFileSync('./ssl/key.pem'),
  ca: fs.readFileSync('./ssl/ca.pem'),
  
  // TLS 버전 제한
  secureProtocol: 'TLSv1_2_method',
  
  // 약한 암호화 제외
  ciphers: [
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-RSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES128-SHA256',
    'ECDHE-RSA-AES256-SHA384'
  ].join(':'),
  
  honorCipherOrder: true,
  
  // 세션 재사용 설정
  sessionTimeout: 300,
  
  // OCSP Stapling
  requestCert: false,
  rejectUnauthorized: true
};

const server = https.createServer(tlsOptions, app);
```

## XSS와 CSRF 방어 전략

### XSS 방어

```javascript
class XSSProtection {
  constructor() {
    this.encoder = new TextEncoder();
    this.decoder = new TextDecoder();
  }

  // HTML 엔티티 인코딩
  encodeHTML(str) {
    const replacements = {
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
      "'": '&#x27;',
      '/': '&#x2F;'
    };
    
    return str.replace(/[&<>"'/]/g, char => replacements[char]);
  }

  // JSON 데이터 새니타이징
  sanitizeJSON(data) {
    if (typeof data === 'string') {
      return this.encodeHTML(data);
    }
    
    if (Array.isArray(data)) {
      return data.map(item => this.sanitizeJSON(item));
    }
    
    if (typeof data === 'object' && data !== null) {
      const sanitized = {};
      for (const [key, value] of Object.entries(data)) {
        sanitized[this.encodeHTML(key)] = this.sanitizeJSON(value);
      }
      return sanitized;
    }
    
    return data;
  }

  // SSE 메시지 새니타이징
  sanitizeSSEMessage(event, data) {
    const sanitizedEvent = this.encodeHTML(event);
    const sanitizedData = typeof data === 'object' 
      ? JSON.stringify(this.sanitizeJSON(data))
      : this.encodeHTML(String(data));
    
    return `event: ${sanitizedEvent}\ndata: ${sanitizedData}\n\n`;
  }
}

// 사용 예제
const xssProtection = new XSSProtection();

class SecureEventStream extends EventEmitter {
  constructor(res) {
    super();
    this.res = res;
    this.xss = new XSSProtection();
  }

  sendEvent(event, data) {
    try {
      // 데이터 검증
      if (!this.validateEventData(data)) {
        throw new Error('Invalid event data');
      }
      
      // XSS 방어 적용
      const safeMessage = this.xss.sanitizeSSEMessage(event, data);
      this.res.write(safeMessage);
      
    } catch (error) {
      console.error('Event send error:', error);
      this.sendError('Invalid data');
    }
  }

  validateEventData(data) {
    // 데이터 크기 제한
    const dataStr = JSON.stringify(data);
    if (dataStr.length > 64 * 1024) { // 64KB 제한
      return false;
    }
    
    // 의심스러운 패턴 검사
    const suspiciousPatterns = [
      /<script[^>]*>/gi,
      /javascript:/gi,
      /on\w+\s*=/gi,
      /<iframe/gi,
      /<object/gi,
      /<embed/gi
    ];
    
    return !suspiciousPatterns.some(pattern => pattern.test(dataStr));
  }
}
```

### CSRF 방어

```javascript
class CSRFProtection {
  constructor(secret) {
    this.secret = secret;
    this.tokens = new Map();
  }

  generateToken(sessionId) {
    const token = crypto
      .createHmac('sha256', this.secret)
      .update(`${sessionId}-${Date.now()}-${Math.random()}`)
      .digest('hex');
    
    this.tokens.set(token, {
      sessionId,
      createdAt: Date.now(),
      used: false
    });
    
    // 토큰 만료 처리
    setTimeout(() => {
      this.tokens.delete(token);
    }, 3600 * 1000); // 1시간
    
    return token;
  }

  validateToken(token, sessionId) {
    const tokenData = this.tokens.get(token);
    
    if (!tokenData) {
      return false;
    }
    
    if (tokenData.used) {
      return false;
    }
    
    if (tokenData.sessionId !== sessionId) {
      return false;
    }
    
    // 토큰 사용 표시
    tokenData.used = true;
    
    return true;
  }
}

// SSE 연결 시 CSRF 토큰 검증
app.get('/events', (req, res) => {
  const csrfToken = req.headers['x-csrf-token'];
  const sessionId = req.session.id;
  
  if (!csrfProtection.validateToken(csrfToken, sessionId)) {
    return res.status(403).json({ error: 'Invalid CSRF token' });
  }
  
  // SSE 연결 진행
  setupSSEConnection(req, res);
});
```

## 로드 밸런싱 설정

### Nginx 로드 밸런싱

```nginx
upstream sse_backend {
    # IP 해시로 스티키 세션 구현
    ip_hash;
    
    # 백엔드 서버들
    server backend1.example.com:3001 max_fails=3 fail_timeout=30s;
    server backend2.example.com:3002 max_fails=3 fail_timeout=30s;
    server backend3.example.com:3003 max_fails=3 fail_timeout=30s;
    
    # 연결 유지 설정
    keepalive 32;
    keepalive_requests 100;
    keepalive_timeout 60s;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    
    # SSE 엔드포인트
    location /events {
        proxy_pass http://sse_backend;
        
        # SSE 필수 헤더
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_cache off;
        
        # 타임아웃 설정
        proxy_read_timeout 24h;
        proxy_connect_timeout 5s;
        proxy_send_timeout 5s;
        
        # 헤더 전달
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 에러 처리
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 3;
    }
}
```

### HAProxy 설정

```
global
    maxconn 100000
    log /dev/log local0
    tune.ssl.default-dh-param 2048

defaults
    mode http
    timeout connect 5s
    timeout client 24h
    timeout server 24h
    timeout tunnel 24h
    option httplog
    option dontlognull

frontend sse_frontend
    bind *:443 ssl crt /etc/ssl/certs/cert.pem
    
    # ACL 정의
    acl is_sse path_beg /events
    
    # SSE 트래픽 라우팅
    use_backend sse_backend if is_sse
    default_backend web_backend

backend sse_backend
    balance source  # 소스 IP 기반 밸런싱
    
    # 헬스 체크
    option httpchk GET /health
    
    # SSE 서버들
    server sse1 10.0.1.10:3001 check fall 3 rise 2 weight 100
    server sse2 10.0.1.11:3001 check fall 3 rise 2 weight 100
    server sse3 10.0.1.12:3001 check fall 3 rise 2 weight 100
    
    # HTTP/1.1 강제
    option http-server-close
    option forwardfor
    
    # 버퍼링 비활성화
    option http-no-delay
```

### 분산 이벤트 브로드캐스팅

```javascript
class DistributedEventBroadcaster {
  constructor(redisConfig) {
    this.publisher = redis.createClient(redisConfig);
    this.subscriber = redis.createClient(redisConfig);
    this.localClients = new Map();
    
    this.setupSubscriber();
  }

  setupSubscriber() {
    this.subscriber.on('message', (channel, message) => {
      const event = JSON.parse(message);
      this.broadcastLocal(channel, event);
    });
  }

  // 로컬 클라이언트에게 브로드캐스트
  broadcastLocal(channel, event) {
    const clients = this.localClients.get(channel) || [];
    
    clients.forEach(client => {
      try {
        client.send(event);
      } catch (error) {
        console.error('Failed to send to client:', error);
        this.removeClient(channel, client);
      }
    });
  }

  // 전체 클러스터에 브로드캐스트
  broadcastGlobal(channel, event) {
    const message = JSON.stringify({
      ...event,
      serverId: process.env.SERVER_ID,
      timestamp: Date.now()
    });
    
    this.publisher.publish(channel, message);
  }

  // 클라이언트 등록
  addClient(channel, client) {
    if (!this.localClients.has(channel)) {
      this.localClients.set(channel, new Set());
      this.subscriber.subscribe(channel);
    }
    
    this.localClients.get(channel).add(client);
  }

  // 클라이언트 제거
  removeClient(channel, client) {
    const clients = this.localClients.get(channel);
    if (clients) {
      clients.delete(client);
      
      if (clients.size === 0) {
        this.localClients.delete(channel);
        this.subscriber.unsubscribe(channel);
      }
    }
  }
}
```

## 모니터링과 디버깅

### 메트릭 수집

```javascript
class SSEMetrics {
  constructor() {
    this.metrics = {
      connections: {
        active: 0,
        total: 0,
        failed: 0
      },
      messages: {
        sent: 0,
        failed: 0,
        totalBytes: 0
      },
      errors: new Map(),
      performance: {
        messageLatency: [],
        connectionDuration: []
      }
    };
  }

  recordConnection() {
    this.metrics.connections.active++;
    this.metrics.connections.total++;
  }

  recordDisconnection(duration) {
    this.metrics.connections.active--;
    this.metrics.performance.connectionDuration.push(duration);
    
    // 평균 계산 (최근 1000개만 유지)
    if (this.metrics.performance.connectionDuration.length > 1000) {
      this.metrics.performance.connectionDuration.shift();
    }
  }

  recordMessage(size, latency) {
    this.metrics.messages.sent++;
    this.metrics.messages.totalBytes += size;
    this.metrics.performance.messageLatency.push(latency);
    
    if (this.metrics.performance.messageLatency.length > 1000) {
      this.metrics.performance.messageLatency.shift();
    }
  }

  recordError(type, message) {
    const key = `${type}:${message}`;
    const count = this.metrics.errors.get(key) || 0;
    this.metrics.errors.set(key, count + 1);
  }

  getReport() {
    const avgLatency = this.calculateAverage(this.metrics.performance.messageLatency);
    const avgDuration = this.calculateAverage(this.metrics.performance.connectionDuration);
    
    return {
      connections: this.metrics.connections,
      messages: {
        ...this.metrics.messages,
        averageSize: this.metrics.messages.totalBytes / this.metrics.messages.sent
      },
      performance: {
        averageMessageLatency: avgLatency,
        averageConnectionDuration: avgDuration
      },
      errors: Array.from(this.metrics.errors.entries())
        .map(([key, count]) => ({ error: key, count }))
        .sort((a, b) => b.count - a.count)
        .slice(0, 10)
    };
  }

  calculateAverage(arr) {
    if (arr.length === 0) return 0;
    return arr.reduce((a, b) => a + b, 0) / arr.length;
  }
}
```

### 디버깅 도구

```javascript
class SSEDebugger {
  constructor(options = {}) {
    this.enabled = options.debug || process.env.SSE_DEBUG === 'true';
    this.logLevel = options.logLevel || 'info';
    this.eventLog = [];
    this.maxLogSize = options.maxLogSize || 1000;
  }

  log(level, category, message, data = {}) {
    if (!this.enabled) return;
    
    const logEntry = {
      timestamp: new Date().toISOString(),
      level,
      category,
      message,
      data,
      stack: level === 'error' ? new Error().stack : undefined
    };
    
    this.eventLog.push(logEntry);
    
    if (this.eventLog.length > this.maxLogSize) {
      this.eventLog.shift();
    }
    
    // 콘솔 출력
    if (this.shouldLog(level)) {
      console.log(JSON.stringify(logEntry, null, 2));
    }
  }

  shouldLog(level) {
    const levels = ['debug', 'info', 'warn', 'error'];
    const currentLevelIndex = levels.indexOf(this.logLevel);
    const messageLevelIndex = levels.indexOf(level);
    
    return messageLevelIndex >= currentLevelIndex;
  }

  // SSE 연결 추적
  traceConnection(clientId, event, data) {
    this.log('debug', 'connection', `${event} for client ${clientId}`, data);
  }

  // 메시지 추적
  traceMessage(clientId, event, data) {
    this.log('debug', 'message', `Sending ${event} to ${clientId}`, {
      event,
      dataSize: JSON.stringify(data).length,
      preview: JSON.stringify(data).substring(0, 100)
    });
  }

  // 에러 추적
  traceError(error, context) {
    this.log('error', 'error', error.message, {
      ...context,
      stack: error.stack,
      name: error.name
    });
  }

  // 디버그 정보 내보내기
  exportDebugInfo() {
    return {
      logs: this.eventLog,
      stats: {
        totalLogs: this.eventLog.length,
        errorCount: this.eventLog.filter(log => log.level === 'error').length,
        categories: this.getCategoryCounts()
      }
    };
  }

  getCategoryCounts() {
    const counts = {};
    this.eventLog.forEach(log => {
      counts[log.category] = (counts[log.category] || 0) + 1;
    });
    return counts;
  }
}

// 프로덕션 모니터링 통합
class ProductionMonitor {
  constructor(config) {
    this.metrics = new SSEMetrics();
    this.debugger = new SSEDebugger(config.debug);
    this.healthCheck = new HealthChecker(config.health);
    this.alerting = new AlertManager(config.alerts);
  }

  // 주기적 상태 보고
  startReporting(interval = 60000) {
    setInterval(() => {
      const report = this.generateReport();
      
      // 메트릭 전송
      this.sendToMonitoringService(report);
      
      // 알림 체크
      this.checkAlerts(report);
      
    }, interval);
  }

  generateReport() {
    return {
      timestamp: new Date().toISOString(),
      metrics: this.metrics.getReport(),
      health: this.healthCheck.getStatus(),
      debug: this.debugger.exportDebugInfo()
    };
  }

  sendToMonitoringService(report) {
    // Prometheus, DataDog, CloudWatch 등으로 전송
    if (process.env.PROMETHEUS_GATEWAY) {
      this.sendToPrometheus(report);
    }
    
    if (process.env.DATADOG_API_KEY) {
      this.sendToDataDog(report);
    }
  }

  checkAlerts(report) {
    // 임계값 체크
    if (report.metrics.connections.active > 10000) {
      this.alerting.trigger('high_connection_count', {
        count: report.metrics.connections.active
      });
    }
    
    if (report.metrics.errors.length > 0) {
      const criticalErrors = report.metrics.errors
        .filter(e => e.count > 100);
      
      if (criticalErrors.length > 0) {
        this.alerting.trigger('high_error_rate', {
          errors: criticalErrors
        });
      }
    }
  }
}
```

### 성능 프로파일링

```javascript
class SSEProfiler {
  constructor() {
    this.profiles = new Map();
    this.enabled = process.env.NODE_ENV === 'development';
  }

  startProfile(name) {
    if (!this.enabled) return () => {};
    
    const start = process.hrtime.bigint();
    const startMemory = process.memoryUsage();
    
    return () => {
      const end = process.hrtime.bigint();
      const endMemory = process.memoryUsage();
      
      const profile = {
        name,
        duration: Number(end - start) / 1e6, // 밀리초
        memory: {
          heapUsed: endMemory.heapUsed - startMemory.heapUsed,
          external: endMemory.external - startMemory.external
        },
        timestamp: new Date().toISOString()
      };
      
      this.addProfile(name, profile);
    };
  }

  addProfile(name, profile) {
    if (!this.profiles.has(name)) {
      this.profiles.set(name, []);
    }
    
    const profiles = this.profiles.get(name);
    profiles.push(profile);
    
    // 최대 100개 프로파일만 유지
    if (profiles.length > 100) {
      profiles.shift();
    }
  }

  getAnalysis(name) {
    const profiles = this.profiles.get(name);
    if (!profiles || profiles.length === 0) {
      return null;
    }
    
    const durations = profiles.map(p => p.duration);
    const memories = profiles.map(p => p.memory.heapUsed);
    
    return {
      name,
      samples: profiles.length,
      duration: {
        min: Math.min(...durations),
        max: Math.max(...durations),
        avg: durations.reduce((a, b) => a + b) / durations.length,
        p50: this.percentile(durations, 0.5),
        p95: this.percentile(durations, 0.95),
        p99: this.percentile(durations, 0.99)
      },
      memory: {
        min: Math.min(...memories),
        max: Math.max(...memories),
        avg: memories.reduce((a, b) => a + b) / memories.length
      }
    };
  }

  percentile(arr, p) {
    const sorted = arr.slice().sort((a, b) => a - b);
    const index = Math.ceil(sorted.length * p) - 1;
    return sorted[index];
  }
}
```

## 마무리

이 장에서는 SSE 애플리케이션의 보안과 인프라 측면을 다루었습니다. CORS 설정부터 인증/권한 부여, SSL/TLS 요구사항, XSS/CSRF 방어, 로드 밸런싱, 그리고 모니터링까지 프로덕션 환경에서 필요한 모든 요소를 포함했습니다.

다음 장에서는 SSE 애플리케이션의 테스트 전략과 도구들을 살펴보겠습니다.

***
**[← 이전: Part 8 - 성능 특성과 최적화](./part8-performance-optimization.md)** | **[목차](./README.md)** | **[다음: Part 10 - 테스팅 가이드 →](./part10-testing-guide.md)**

***