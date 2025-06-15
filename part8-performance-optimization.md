---
**[← 이전: Part 7 - 에러 처리와 재연결 메커니즘](./part7-error-reconnection.md)** | **[목차](./README.md)** | **[다음: Part 9 - 보안과 인프라 →](./part9-security-infrastructure.md)**

---

# Part 8: 성능 특성과 최적화

## 9장. 성능 특성과 최적화

SSE는 실시간 데이터 스트리밍에 효율적이지만, 대규모 서비스에서는 세심한 성능 최적화가 필요합니다. 이 장에서는 SSE의 성능 특성과 최적화 전략을 다룹니다.

### 9.1 대기 시간 및 처리량

#### 9.1.1 지연 시간 분석

```typescript
class SSELatencyAnalyzer {
  private metrics = {
    firstByteTime: 0,
    totalTransferTime: 0,
    chunkCount: 0,
    chunkSizes: [] as number[],
    timestamps: [] as number[]
  };
  
  startMeasurement() {
    this.metrics.timestamps.push(performance.now());
  }
  
  measureFirstByte() {
    const now = performance.now();
    this.metrics.firstByteTime = now - this.metrics.timestamps[0];
  }
  
  measureChunk(size: number) {
    const now = performance.now();
    this.metrics.chunkCount++;
    this.metrics.chunkSizes.push(size);
    this.metrics.timestamps.push(now);
  }  
  calculateMetrics() {
    const lastTimestamp = this.metrics.timestamps[this.metrics.timestamps.length - 1];
    this.metrics.totalTransferTime = lastTimestamp - this.metrics.timestamps[0];
    
    const avgChunkSize = this.metrics.chunkSizes.reduce((a, b) => a + b, 0) / this.metrics.chunkSizes.length;
    const totalDataSize = this.metrics.chunkSizes.reduce((a, b) => a + b, 0);
    const throughput = totalDataSize / (this.metrics.totalTransferTime / 1000); // bytes/sec
    
    // 청크 간 지연 계산
    const interChunkDelays: number[] = [];
    for (let i = 1; i < this.metrics.timestamps.length; i++) {
      interChunkDelays.push(this.metrics.timestamps[i] - this.metrics.timestamps[i - 1]);
    }
    
    return {
      firstByteLatency: this.metrics.firstByteTime,
      totalTime: this.metrics.totalTransferTime,
      averageChunkSize: avgChunkSize,
      throughput: throughput,
      chunksPerSecond: this.metrics.chunkCount / (this.metrics.totalTransferTime / 1000),
      averageInterChunkDelay: interChunkDelays.reduce((a, b) => a + b, 0) / interChunkDelays.length
    };
  }
}
```#### 9.1.2 처리량 최적화

```typescript
class ThroughputOptimizer {
  private bufferSize = 4096; // 4KB
  private flushThreshold = 0.8; // 80% full
  private buffer: Uint8Array;
  private position = 0;
  
  constructor(private response: Response) {
    this.buffer = new Uint8Array(this.bufferSize);
  }
  
  write(data: string) {
    const encoded = new TextEncoder().encode(data);
    
    // 버퍼가 충분히 크면 직접 전송
    if (encoded.length > this.bufferSize * 0.5) {
      this.flush();
      this.response.write(data);
      return;
    }
    
    // 버퍼에 공간이 부족하면 flush
    if (this.position + encoded.length > this.bufferSize * this.flushThreshold) {
      this.flush();
    }
    
    // 버퍼에 추가
    this.buffer.set(encoded, this.position);
    this.position += encoded.length;
  }  
  flush() {
    if (this.position > 0) {
      const data = new TextDecoder().decode(this.buffer.slice(0, this.position));
      this.response.write(data);
      this.position = 0;
    }
  }
  
  // Nagle 알고리즘 비활성화 (Node.js)
  setNoDelay(socket: any) {
    socket.setNoDelay(true);
  }
}
```

#### 9.1.3 동시 연결 관리

```typescript
class ConnectionPool {
  private connections = new Map<string, SSEConnection>();
  private maxConnections = 10000;
  private connectionStats = {
    active: 0,
    total: 0,
    rejected: 0,
    avgDuration: 0
  };
  
  async addConnection(id: string, connection: SSEConnection): Promise<boolean> {
    if (this.connections.size >= this.maxConnections) {
      this.connectionStats.rejected++;
      return false;
    }    
    this.connections.set(id, connection);
    this.connectionStats.active++;
    this.connectionStats.total++;
    
    connection.on('close', () => {
      this.removeConnection(id);
    });
    
    return true;
  }
  
  removeConnection(id: string) {
    if (this.connections.delete(id)) {
      this.connectionStats.active--;
    }
  }
  
  broadcast(message: string) {
    const encoder = new TextEncoder();
    const data = encoder.encode(`data: ${message}\n\n`);
    
    // 병렬 전송
    const promises = Array.from(this.connections.values()).map(
      conn => conn.sendRaw(data)
    );
    
    return Promise.allSettled(promises);
  }
  
  getStats() {
    return {
      ...this.connectionStats,
      utilization: this.connections.size / this.maxConnections
    };
  }
}
```### 9.2 HTTP/2 및 HTTP/3의 영향

#### 9.2.1 HTTP/2 멀티플렉싱

```typescript
// HTTP/2 서버 설정 (Node.js)
import http2 from 'http2';
import fs from 'fs';

const server = http2.createSecureServer({
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt'),
  // HTTP/2 최적화 설정
  settings: {
    headerTableSize: 4096,
    enablePush: false, // SSE는 서버 푸시 불필요
    initialWindowSize: 1048576, // 1MB
    maxFrameSize: 16384,
    maxConcurrentStreams: 1000,
    maxHeaderListSize: 8192
  }
});

server.on('stream', (stream, headers) => {
  if (headers[':path'] === '/events') {
    // HTTP/2 스트림별 우선순위 설정
    stream.priority({
      parent: 0,
      weight: 16,
      exclusive: false
    });
    
    stream.respond({
      ':status': 200,
      'content-type': 'text/event-stream',
      'cache-control': 'no-cache'
    });    
    // HTTP/2 flow control
    stream.on('drain', () => {
      // 버퍼가 비었을 때 더 많은 데이터 전송
      sendNextChunk(stream);
    });
  }
});

// HTTP/2 성능 모니터링
class HTTP2PerformanceMonitor {
  private streamMetrics = new Map<number, {
    startTime: number;
    bytesWritten: number;
    writeCount: number;
  }>();
  
  trackStream(streamId: number) {
    this.streamMetrics.set(streamId, {
      startTime: Date.now(),
      bytesWritten: 0,
      writeCount: 0
    });
  }
  
  updateMetrics(streamId: number, bytes: number) {
    const metrics = this.streamMetrics.get(streamId);
    if (metrics) {
      metrics.bytesWritten += bytes;
      metrics.writeCount++;
    }
  }
  
  getStreamStats(streamId: number) {
    const metrics = this.streamMetrics.get(streamId);
    if (!metrics) return null;    
    const duration = Date.now() - metrics.startTime;
    return {
      throughput: metrics.bytesWritten / (duration / 1000),
      avgChunkSize: metrics.bytesWritten / metrics.writeCount,
      duration
    };
  }
}
```

#### 9.2.2 HTTP/3 QUIC 최적화

```typescript
// HTTP/3 설정 (실험적)
interface HTTP3Config {
  maxStreamDataBidiLocal: number;
  maxStreamDataBidiRemote: number;
  maxStreamDataUni: number;
  maxData: number;
  maxBidiStreams: number;
  maxUniStreams: number;
  idleTimeout: number;
}

const http3Config: HTTP3Config = {
  maxStreamDataBidiLocal: 1048576,  // 1MB
  maxStreamDataBidiRemote: 1048576,
  maxStreamDataUni: 1048576,
  maxData: 10485760,                 // 10MB
  maxBidiStreams: 100,
  maxUniStreams: 100,
  idleTimeout: 30000                 // 30s
};// QUIC 특성을 활용한 SSE 최적화
class QUICOptimizedSSE {
  // 0-RTT 연결 활용
  async establish0RTTConnection(sessionTicket: Buffer) {
    // 이전 연결의 세션 티켓 사용
    // 첫 번째 요청과 함께 데이터 전송 가능
  }
  
  // 연결 마이그레이션 지원
  handleConnectionMigration(oldConnection: any, newConnection: any) {
    // 네트워크 변경 시 (WiFi → LTE) 연결 유지
    // 진행 중인 SSE 스트림 중단 없이 계속
  }
  
  // 향상된 혼잡 제어
  optimizeCongestionControl() {
    // QUIC의 더 나은 RTT 측정 활용
    // 패킷 손실에 대한 빠른 반응
  }
}
```

### 9.3 서버 측 최적화 전략

#### 9.3.1 이벤트 루프 최적화

```typescript
// Node.js 이벤트 루프 최적화
class EventLoopOptimizer {
  private pendingWrites: Array<{
    connection: SSEConnection;
    data: string;
  }> = [];  
  private batchSize = 100;
  private isProcessing = false;
  
  queueWrite(connection: SSEConnection, data: string) {
    this.pendingWrites.push({ connection, data });
    
    if (!this.isProcessing) {
      setImmediate(() => this.processBatch());
    }
  }
  
  private async processBatch() {
    this.isProcessing = true;
    
    while (this.pendingWrites.length > 0) {
      const batch = this.pendingWrites.splice(0, this.batchSize);
      
      // 병렬 처리
      await Promise.all(
        batch.map(({ connection, data }) => 
          connection.send(data).catch(err => 
            console.error('Write failed:', err)
          )
        )
      );
      
      // 이벤트 루프에 양보
      await new Promise(resolve => setImmediate(resolve));
    }
    
    this.isProcessing = false;
  }
}
```#### 9.3.2 메모리 최적화

```typescript
class MemoryEfficientSSE {
  // Object Pool 패턴
  private messagePool: Array<{
    id: string;
    data: string;
    used: boolean;
  }> = [];
  
  private poolSize = 1000;
  
  getPooledMessage(): any {
    let message = this.messagePool.find(m => !m.used);
    
    if (!message) {
      if (this.messagePool.length < this.poolSize) {
        message = { id: '', data: '', used: false };
        this.messagePool.push(message);
      } else {
        // 풀이 가득 참 - 가장 오래된 메시지 재사용
        message = this.messagePool[0];
      }
    }
    
    message.used = true;
    return message;
  }
  
  releaseMessage(message: any) {
    message.used = false;
    message.id = '';
    message.data = '';
  }  
  // WeakMap을 사용한 메모리 누수 방지
  private connectionMetadata = new WeakMap<SSEConnection, {
    created: number;
    lastActivity: number;
    messageCount: number;
  }>();
  
  trackConnection(connection: SSEConnection) {
    this.connectionMetadata.set(connection, {
      created: Date.now(),
      lastActivity: Date.now(),
      messageCount: 0
    });
  }
  
  // 메모리 압력 모니터링
  monitorMemoryPressure() {
    const usage = process.memoryUsage();
    const heapUsedPercent = usage.heapUsed / usage.heapTotal;
    
    if (heapUsedPercent > 0.9) {
      // 90% 이상 사용 시 경고
      console.warn('High memory pressure:', usage);
      
      // 오래된 연결 정리
      this.cleanupIdleConnections();
      
      // 가비지 컬렉션 강제 실행 (주의해서 사용)
      if (global.gc) {
        global.gc();
      }
    }
  }
}```

#### 9.3.3 CPU 최적화

```typescript
import cluster from 'cluster';
import os from 'os';

// 멀티 프로세스 클러스터링
class ClusteredSSEServer {
  private numWorkers = os.cpus().length;
  
  start() {
    if (cluster.isPrimary) {
      console.log(`Master ${process.pid} is running`);
      
      // 워커 프로세스 생성
      for (let i = 0; i < this.numWorkers; i++) {
        cluster.fork();
      }
      
      // 워커 재시작
      cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died`);
        cluster.fork();
      });
      
      // 로드 밸런싱 메트릭
      this.setupLoadBalancing();
    } else {
      // 워커 프로세스
      this.startWorker();
    }
  }  
  private setupLoadBalancing() {
    const workerLoad = new Map<number, number>();
    
    // 각 워커의 부하 추적
    cluster.on('message', (worker, message) => {
      if (message.type === 'load-report') {
        workerLoad.set(worker.id, message.connections);
      }
    });
    
    // 주기적 부하 재분배
    setInterval(() => {
      const avgLoad = Array.from(workerLoad.values())
        .reduce((a, b) => a + b, 0) / workerLoad.size;
      
      // 부하가 높은 워커 식별
      for (const [workerId, load] of workerLoad) {
        if (load > avgLoad * 1.5) {
          // 새 연결을 다른 워커로 리다이렉트
          cluster.workers[workerId]?.send({ 
            type: 'reduce-load' 
          });
        }
      }
    }, 5000);
  }
  
  private startWorker() {
    // 워커별 SSE 서버 시작
    const server = new SSEServer();
    server.start();    
    // 부하 보고
    setInterval(() => {
      process.send!({
        type: 'load-report',
        connections: server.getConnectionCount()
      });
    }, 1000);
  }
}
```

### 9.4 메시지 배치 및 압축

#### 9.4.1 메시지 배치 처리

```typescript
class MessageBatcher {
  private batch: Array<{
    timestamp: number;
    data: any;
  }> = [];
  
  private batchSize = 10;
  private batchTimeout = 100; // ms
  private lastFlush = Date.now();
  
  add(data: any) {
    this.batch.push({
      timestamp: Date.now(),
      data
    });
    
    if (this.shouldFlush()) {
      return this.flush();
    }
    
    return null;
  }  
  private shouldFlush(): boolean {
    return (
      this.batch.length >= this.batchSize ||
      Date.now() - this.lastFlush >= this.batchTimeout
    );
  }
  
  flush(): string | null {
    if (this.batch.length === 0) return null;
    
    const batchData = {
      type: 'batch',
      count: this.batch.length,
      items: this.batch
    };
    
    this.batch = [];
    this.lastFlush = Date.now();
    
    return `data: ${JSON.stringify(batchData)}\n\n`;
  }
  
  // 적응형 배치 크기
  adjustBatchSize(throughput: number) {
    if (throughput < 1000) {
      // 낮은 처리량 - 배치 크기 감소
      this.batchSize = Math.max(5, this.batchSize - 1);
    } else if (throughput > 10000) {
      // 높은 처리량 - 배치 크기 증가
      this.batchSize = Math.min(50, this.batchSize + 1);
    }
  }
}```

#### 9.4.2 압축 전략

```typescript
import zlib from 'zlib';

class CompressionSSE {
  // Brotli 압축 (최고 압축률)
  async compressBrotli(data: string): Promise<Buffer> {
    return new Promise((resolve, reject) => {
      zlib.brotliCompress(data, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  }
  
  // 스트리밍 압축
  createCompressedStream(response: any) {
    // gzip 압축 스트림
    const gzip = zlib.createGzip({
      level: zlib.Z_BEST_SPEED, // 속도 우선
      memLevel: 8,
      strategy: zlib.Z_DEFAULT_STRATEGY
    });
    
    // 압축 헤더 설정
    response.setHeader('Content-Encoding', 'gzip');
    response.setHeader('Vary', 'Accept-Encoding');
    
    return {
      write: (data: string) => {
        gzip.write(data);
      },
      end: () => {
        gzip.end();
      }
    };  }
  
  // 선택적 압축
  shouldCompress(data: string, acceptEncoding: string): boolean {
    // 작은 데이터는 압축하지 않음
    if (data.length < 1000) return false;
    
    // 클라이언트 지원 확인
    if (!acceptEncoding.includes('gzip')) return false;
    
    // 이미 압축된 형식 제외
    try {
      JSON.parse(data);
      return true; // JSON은 압축 효과 좋음
    } catch {
      return false; // 바이너리나 이미 압축된 데이터
    }
  }
}
```

### 9.5 스트리밍 버퍼링 문제와 해결

#### 9.5.1 프록시 버퍼링 해결

```typescript
class ProxyBufferingMitigation {
  // nginx 버퍼링 비활성화
  setNginxHeaders(response: any) {
    response.setHeader('X-Accel-Buffering', 'no');
    response.setHeader('Cache-Control', 'no-cache');
  }  
  // CloudFlare 버퍼링 회피
  setCloudflareHeaders(response: any) {
    // 충분한 데이터 전송으로 버퍼 플러시 유도
    response.write(':' + ' '.repeat(2048) + '\n\n');
  }
  
  // 패딩을 통한 버퍼 플러시
  forcefulFlush(response: any) {
    const padding = ':' + ' '.repeat(4096) + '\n\n';
    response.write(padding);
  }
  
  // 청크 크기 최적화
  optimizeChunkSize(data: string): string {
    // 최소 청크 크기 보장 (일부 프록시는 작은 청크 버퍼링)
    const minChunkSize = 1024;
    
    if (data.length < minChunkSize) {
      const padding = ' '.repeat(minChunkSize - data.length);
      return data + `\n:${padding}\n`;
    }
    
    return data;
  }
}
```#### 9.5.2 브라우저 버퍼링 최적화

```typescript
class BrowserBufferingOptimization {
  // 초기 플러시로 연결 확립
  async establishConnection(response: any) {
    // 2KB 초기 데이터로 브라우저 버퍼 플러시
    const initialData = ':' + ' '.repeat(2048) + '\n\n';
    response.write(initialData);
    
    // 연결 확인 이벤트
    response.write('event: connected\n');
    response.write('data: {"status": "ready"}\n\n');
  }
  
  // 점진적 렌더링 유도
  enableProgressiveRendering(response: any) {
    // Content-Type에 charset 명시
    response.setHeader('Content-Type', 'text/event-stream; charset=utf-8');
    
    // 빠른 시작을 위한 힌트
    response.setHeader('Link', '</styles.css>; rel=preload; as=style');
  }
  
  // 청크 경계 최적화
  alignChunkBoundaries(message: string): string {
    // SSE 메시지가 청크 경계에 걸치지 않도록
    const boundary = '\n\n';
    if (!message.endsWith(boundary)) {
      message += boundary;
    }
    
    return message;
  }
}```

#### 9.5.3 통합 성능 모니터링

```typescript
class SSEPerformanceMonitor {
  private metrics = {
    connections: new Map<string, ConnectionMetrics>(),
    global: {
      totalConnections: 0,
      activeConnections: 0,
      messagesPerSecond: 0,
      bytesPerSecond: 0,
      avgLatency: 0
    }
  };
  
  trackConnection(id: string, connection: SSEConnection) {
    this.metrics.connections.set(id, {
      created: Date.now(),
      messagesSent: 0,
      bytesSent: 0,
      lastActivity: Date.now(),
      errors: 0
    });
    
    this.metrics.global.totalConnections++;
    this.metrics.global.activeConnections++;
  }
  
  recordMessage(id: string, size: number) {
    const conn = this.metrics.connections.get(id);
    if (conn) {
      conn.messagesSent++;
      conn.bytesSent += size;
      conn.lastActivity = Date.now();
    }
  }  
  generateReport(): PerformanceReport {
    const now = Date.now();
    const activeConns = Array.from(this.metrics.connections.values())
      .filter(conn => now - conn.lastActivity < 60000);
    
    const totalMessages = activeConns.reduce((sum, conn) => sum + conn.messagesSent, 0);
    const totalBytes = activeConns.reduce((sum, conn) => sum + conn.bytesSent, 0);
    
    return {
      timestamp: now,
      connections: {
        total: this.metrics.global.totalConnections,
        active: activeConns.length,
        idle: this.metrics.connections.size - activeConns.length
      },
      throughput: {
        messagesPerSecond: totalMessages / (now / 1000),
        bytesPerSecond: totalBytes / (now / 1000),
        avgMessageSize: totalBytes / totalMessages
      },
      health: {
        errorRate: this.calculateErrorRate(),
        memoryUsage: process.memoryUsage(),
        cpuUsage: process.cpuUsage()
      }
    };
  }
  
  private calculateErrorRate(): number {
    const totalErrors = Array.from(this.metrics.connections.values())
      .reduce((sum, conn) => sum + conn.errors, 0);
    
    return totalErrors / this.metrics.global.totalConnections;
  }
}

interface ConnectionMetrics {
  created: number;
  messagesSent: number;
  bytesSent: number;
  lastActivity: number;
  errors: number;
}

interface PerformanceReport {
  timestamp: number;
  connections: {
    total: number;
    active: number;
    idle: number;
  };
  throughput: {
    messagesPerSecond: number;
    bytesPerSecond: number;
    avgMessageSize: number;
  };
  health: {
    errorRate: number;
    memoryUsage: NodeJS.MemoryUsage;
    cpuUsage: NodeJS.CpuUsage;
  };
}
```

---

**다음 장 예고**: [Part 9: 보안과 인프라](./part9-security-infrastructure.md)에서는 SSE 구현 시 고려해야 할 보안 사항과 인프라 구성을 다루겠습니다.

---
**[← 이전: Part 7 - 에러 처리와 재연결 메커니즘](./part7-error-reconnection.md)** | **[목차](./README.md)** | **[다음: Part 9 - 보안과 인프라 →](./part9-security-infrastructure.md)**

---