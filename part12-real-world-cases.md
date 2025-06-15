---
**[← 이전: Part 11 - 프로덕션 예제](./part11-production-examples.md)** | **[목차](./README.md)** | **[다음: Part 13 - 부록 →](./part13-appendix.md)**

---

# Part 12: 실제 기업 사례

## 목차
- [OpenAI ChatGPT 사례 연구](#openai-chatgpt-사례-연구)
- [GitHub Actions 사례 연구](#github-actions-사례-연구)
- [Vercel 배포 사례 연구](#vercel-배포-사례-연구)
- [Shopify BFCM Live Map 사례 연구](#shopify-bfcm-live-map-사례-연구)
- [LinkedIn 인스턴트 메시징 사례 연구](#linkedin-인스턴트-메시징-사례-연구)

## OpenAI ChatGPT 사례 연구

### 개요

OpenAI의 ChatGPT는 SSE를 활용한 가장 대표적인 사례 중 하나입니다. 대규모 언어 모델의 응답을 실시간으로 스트리밍하여 사용자 경험을 크게 개선했습니다.

### 기술적 구현

```typescript
// OpenAI 스타일 스트리밍 구현
interface ChatCompletionChunk {
  id: string;
  object: 'chat.completion.chunk';
  created: number;
  model: string;
  choices: {
    index: number;
    delta: {
      role?: string;
      content?: string;
    };
    finish_reason: string | null;
  }[];
}

class OpenAIStreamingClient {
  private apiKey: string;
  private baseURL: string;

  constructor(apiKey: string) {
    this.apiKey = apiKey;
    this.baseURL = 'https://api.openai.com/v1';
  }

  async createChatCompletionStream(
    messages: Array<{ role: string; content: string }>,
    options: {
      model?: string;
      temperature?: number;
      max_tokens?: number;
    }
  ): AsyncGenerator<ChatCompletionChunk> {
    const response = await fetch(`${this.baseURL}/chat/completions`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.apiKey}`,
        'Accept': 'text/event-stream',
      },
      body: JSON.stringify({
        model: options.model || 'gpt-4',
        messages,
        stream: true,
        temperature: options.temperature,
        max_tokens: options.max_tokens,
      }),
    });

    const reader = response.body!.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6);
          if (data === '[DONE]') return;
          
          try {
            yield JSON.parse(data);
          } catch (e) {
            console.error('Failed to parse SSE data:', e);
          }
        }
      }
    }
  }
}

// UI 컴포넌트에서 사용
const StreamingChatMessage: React.FC = () => {
  const [content, setContent] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);

  const handleStream = async () => {
    setIsStreaming(true);
    const client = new OpenAIStreamingClient(API_KEY);
    
    const stream = client.createChatCompletionStream(
      [{ role: 'user', content: 'Hello!' }],
      { model: 'gpt-4' }
    );

    for await (const chunk of stream) {
      if (chunk.choices[0]?.delta?.content) {
        setContent(prev => prev + chunk.choices[0].delta.content);
      }
    }
    
    setIsStreaming(false);
  };

  return (
    <div className="chat-message">
      <div className="content">{content}</div>
      {isStreaming && <div className="typing-indicator">AI가 입력 중...</div>}
    </div>
  );
};
```

### 주요 교훈

1. **스트리밍 UX**: 첫 번째 토큰까지의 시간(TTFT) 최적화로 체감 속도 향상
2. **에러 처리**: 네트워크 중단 시 graceful degradation
3. **토큰 단위 렌더링**: 자연스러운 타이핑 효과로 사용자 경험 개선
4. **백프레셔 관리**: 클라이언트 렌더링 속도에 맞춘 스트림 제어

## GitHub Actions 사례 연구

### 개요

GitHub Actions는 워크플로우 실행 로그를 실시간으로 스트리밍하기 위해 SSE를 활용합니다. 대용량 로그 데이터를 효율적으로 전송하면서도 실시간성을 보장합니다.

### 기술적 구현

```typescript
// GitHub Actions 로그 스트리밍
interface LogLine {
  timestamp: string;
  level: 'info' | 'warning' | 'error' | 'debug';
  message: string;
  stepId?: string;
  lineNumber: number;
}

class GitHubActionsLogStreamer {
  private eventSource: EventSource;
  private logBuffer: LogLine[] = [];
  private subscribers: Set<(logs: LogLine[]) => void> = new Set();

  constructor(runId: string, token: string) {
    const url = `https://api.github.com/repos/{owner}/{repo}/actions/runs/${runId}/logs`;
    
    this.eventSource = new EventSource(url, {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Accept': 'text/event-stream'
      }
    });

    this.setupEventHandlers();
  }

  private setupEventHandlers() {
    this.eventSource.addEventListener('log', (event) => {
      const logData = JSON.parse(event.data);
      const logLine: LogLine = {
        timestamp: new Date().toISOString(),
        level: this.detectLogLevel(logData.message),
        message: this.formatMessage(logData.message),
        stepId: logData.stepId,
        lineNumber: this.logBuffer.length + 1
      };

      this.logBuffer.push(logLine);
      this.notifySubscribers();
    });

    this.eventSource.addEventListener('step-start', (event) => {
      const stepData = JSON.parse(event.data);
      this.logBuffer.push({
        timestamp: new Date().toISOString(),
        level: 'info',
        message: `=== Step Started: ${stepData.name} ===`,
        stepId: stepData.id,
        lineNumber: this.logBuffer.length + 1
      });
      this.notifySubscribers();
    });

    this.eventSource.addEventListener('step-complete', (event) => {
      const stepData = JSON.parse(event.data);
      const status = stepData.conclusion === 'success' ? '✓' : '✗';
      
      this.logBuffer.push({
        timestamp: new Date().toISOString(),
        level: stepData.conclusion === 'success' ? 'info' : 'error',
        message: `=== Step Completed: ${stepData.name} ${status} (${stepData.duration}ms) ===`,
        stepId: stepData.id,
        lineNumber: this.logBuffer.length + 1
      });
      this.notifySubscribers();
    });
  }

  private detectLogLevel(message: string): LogLine['level'] {
    if (message.includes('ERROR') || message.includes('FAIL')) return 'error';
    if (message.includes('WARN')) return 'warning';
    if (message.includes('DEBUG')) return 'debug';
    return 'info';
  }

  private formatMessage(message: string): string {
    // ANSI 컬러 코드를 HTML로 변환
    return message
      .replace(/\x1b\[31m/g, '<span class="text-red">')
      .replace(/\x1b\[32m/g, '<span class="text-green">')
      .replace(/\x1b\[33m/g, '<span class="text-yellow">')
      .replace(/\x1b\[0m/g, '</span>');
  }

  subscribe(callback: (logs: LogLine[]) => void): () => void {
    this.subscribers.add(callback);
    callback(this.logBuffer); // 초기 데이터 전송

    return () => {
      this.subscribers.delete(callback);
    };
  }

  private notifySubscribers() {
    this.subscribers.forEach(callback => callback(this.logBuffer));
  }

  // 가상 스크롤링을 위한 범위 조회
  getLogsInRange(start: number, end: number): LogLine[] {
    return this.logBuffer.slice(start, end);
  }

  close() {
    this.eventSource.close();
  }
}

// React 컴포넌트
const ActionsLogViewer: React.FC<{ runId: string }> = ({ runId }) => {
  const [logs, setLogs] = useState<LogLine[]>([]);
  const [filter, setFilter] = useState<string>('');
  const logContainerRef = useRef<HTMLDivElement>(null);
  const [autoScroll, setAutoScroll] = useState(true);

  useEffect(() => {
    const streamer = new GitHubActionsLogStreamer(runId, process.env.GITHUB_TOKEN!);
    
    const unsubscribe = streamer.subscribe((newLogs) => {
      setLogs(newLogs);
      
      if (autoScroll && logContainerRef.current) {
        logContainerRef.current.scrollTop = logContainerRef.current.scrollHeight;
      }
    });

    streamer.startStreaming();

    return () => {
      unsubscribe();
    };
  }, [runId, autoScroll]);

  const filteredLogs = logs.filter(log => 
    log.message.toLowerCase().includes(filter.toLowerCase())
  );

  return (
    <div className="actions-log-viewer">
      <div className="log-controls">
        <input
          type="text"
          placeholder="Filter logs..."
          value={filter}
          onChange={(e) => setFilter(e.target.value)}
        />
        <label>
          <input
            type="checkbox"
            checked={autoScroll}
            onChange={(e) => setAutoScroll(e.target.checked)}
          />
          Auto-scroll
        </label>
      </div>

      <div 
        ref={logContainerRef}
        className="log-container"
      >
        {filteredLogs.map((log, index) => (
          <div 
            key={index}
            className={`log-line log-${log.level}`}
          >
            <span className="log-timestamp">{log.timestamp}</span>
            <span 
              className="log-message"
              dangerouslySetInnerHTML={{ __html: log.message }}
            />
          </div>
        ))}
      </div>
    </div>
  );
};
```

### 주요 교훈

1. **대용량 로그 처리**: 효율적인 버퍼링과 가상 스크롤링으로 성능 유지
2. **실시간 필터링**: 클라이언트 사이드 필터링으로 즉각적인 반응성
3. **ANSI 코드 지원**: 터미널 출력을 웹에서 정확히 재현
4. **단계별 추적**: 워크플로우 단계별 진행 상황 실시간 업데이트

## Vercel 배포 사례 연구

### 개요

Vercel은 배포 프로세스의 실시간 피드백을 제공하기 위해 SSE를 활용합니다. 빌드 로그, 상태 업데이트, 프리뷰 URL 생성 등을 스트리밍합니다.

### 기술적 구현

```typescript
// Vercel 스타일 배포 스트리밍
interface DeploymentEvent {
  type: 'build' | 'deploy' | 'ready' | 'error' | 'log';
  timestamp: number;
  data: any;
}

class VercelDeploymentStreamer {
  private eventSource: EventSource | null = null;
  private deploymentId: string;
  private state: DeploymentState = {
    status: 'initializing',
    progress: 0,
    logs: [],
    url: null,
    error: null
  };

  constructor(deploymentId: string) {
    this.deploymentId = deploymentId;
  }

  async startStreaming(): Promise<void> {
    const url = `https://api.vercel.com/v1/deployments/${this.deploymentId}/events`;
    
    this.eventSource = new EventSource(url, {
      headers: {
        'Authorization': `Bearer ${process.env.VERCEL_TOKEN}`
      }
    });

    this.setupEventHandlers();
  }

  private setupEventHandlers() {
    if (!this.eventSource) return;

    // 빌드 이벤트
    this.eventSource.addEventListener('build-state', (event) => {
      const data = JSON.parse(event.data);
      this.updateState({
        status: 'building',
        progress: data.progress,
        currentStep: data.step
      });
    });

    // 로그 스트리밍
    this.eventSource.addEventListener('log', (event) => {
      const logEntry = JSON.parse(event.data);
      this.state.logs.push({
        timestamp: Date.now(),
        level: logEntry.level,
        message: logEntry.message,
        source: logEntry.source
      });
      this.notifySubscribers();
    });

    // 함수 생성
    this.eventSource.addEventListener('creating-functions', (event) => {
      const data = JSON.parse(event.data);
      this.updateState({
        status: 'creating-functions',
        functions: data.functions
      });
    });

    // 배포 준비 완료
    this.eventSource.addEventListener('ready', (event) => {
      const data = JSON.parse(event.data);
      this.updateState({
        status: 'ready',
        progress: 100,
        url: data.url,
        aliasUrls: data.aliasUrls
      });
    });

    // 에러 처리
    this.eventSource.addEventListener('error', (event) => {
      const error = JSON.parse(event.data);
      this.updateState({
        status: 'error',
        error: {
          code: error.code,
          message: error.message,
          details: error.details
        }
      });
    });
  }

  // 프로그레시브 UI 업데이트
  private updateState(update: Partial<DeploymentState>) {
    this.state = { ...this.state, ...update };
    this.notifySubscribers();
  }

  // React Hook으로 사용
  static useDeploymentStream(deploymentId: string) {
    const [state, setState] = useState<DeploymentState>({
      status: 'initializing',
      progress: 0,
      logs: [],
      url: null,
      error: null
    });

    useEffect(() => {
      const streamer = new VercelDeploymentStreamer(deploymentId);
      
      streamer.subscribe((newState) => {
        setState(newState);
      });

      streamer.startStreaming();

      return () => {
        streamer.close();
      };
    }, [deploymentId]);

    return state;
  }
}

// UI 컴포넌트
const DeploymentProgress: React.FC<{ deploymentId: string }> = ({ deploymentId }) => {
  const deployment = VercelDeploymentStreamer.useDeploymentStream(deploymentId);

  return (
    <div className="deployment-progress">
      <div className="status-header">
        <h3>Deployment {deploymentId}</h3>
        <span className={`status-badge status-${deployment.status}`}>
          {deployment.status}
        </span>
      </div>

      <div className="progress-bar">
        <div 
          className="progress-fill"
          style={{ width: `${deployment.progress}%` }}
        />
      </div>

      {deployment.currentStep && (
        <div className="current-step">
          {deployment.currentStep}
        </div>
      )}

      <div className="log-viewer">
        {deployment.logs.map((log, index) => (
          <div key={index} className={`log-entry log-${log.level}`}>
            <span className="log-time">
              {new Date(log.timestamp).toLocaleTimeString()}
            </span>
            <span className="log-source">[{log.source}]</span>
            <span className="log-message">{log.message}</span>
          </div>
        ))}
      </div>

      {deployment.status === 'ready' && deployment.url && (
        <div className="deployment-result">
          <h4>Deployment Ready!</h4>
          <a href={deployment.url} target="_blank" rel="noopener noreferrer">
            {deployment.url}
          </a>
        </div>
      )}

      {deployment.error && (
        <div className="deployment-error">
          <h4>Deployment Failed</h4>
          <p>{deployment.error.message}</p>
          {deployment.error.details && (
            <pre>{JSON.stringify(deployment.error.details, null, 2)}</pre>
          )}
        </div>
      )}
    </div>
  );
};
```

### 주요 교훈

1. **진행률 시각화**: 단계별 진행 상황을 명확하게 표시
2. **실시간 로그**: 빌드 로그를 스트리밍하여 즉각적인 피드백
3. **상태 전환**: 배포 상태 변화를 실시간으로 반영
4. **에러 복구**: 실패 시 상세한 정보 제공과 재시도 옵션

## Shopify BFCM Live Map 사례 연구

### 개요

Shopify의 Black Friday Cyber Monday (BFCM) Live Map은 전 세계 실시간 거래 데이터를 시각화합니다. SSE를 통해 초당 수천 건의 거래를 효율적으로 스트리밍합니다.

### 기술적 구현

```typescript
// Shopify BFCM 실시간 데이터 스트리밍
interface SaleEvent {
  id: string;
  timestamp: number;
  location: {
    lat: number;
    lng: number;
    city: string;
    country: string;
  };
  amount: number;
  currency: string;
  products: number;
}

class BFCMLiveMapStream {
  private eventSource: EventSource;
  private salesBuffer: SaleEvent[] = [];
  private stats = {
    totalSales: 0,
    totalRevenue: 0,
    salesPerSecond: 0,
    topCountries: new Map<string, number>()
  };

  constructor() {
    this.eventSource = new EventSource('https://bfcm-api.shopify.com/live-sales');
    this.setupEventHandlers();
    this.startStatsCalculation();
  }

  private setupEventHandlers() {
    // 개별 판매 이벤트
    this.eventSource.addEventListener('sale', (event) => {
      const sale: SaleEvent = JSON.parse(event.data);
      
      // 버퍼에 추가 (최대 1000개 유지)
      this.salesBuffer.push(sale);
      if (this.salesBuffer.length > 1000) {
        this.salesBuffer.shift();
      }

      // 통계 업데이트
      this.updateStats(sale);
      
      // 구독자에게 알림
      this.notifySaleEvent(sale);
    });

    // 집계 데이터
    this.eventSource.addEventListener('stats-update', (event) => {
      const stats = JSON.parse(event.data);
      this.stats = { ...this.stats, ...stats };
      this.notifyStatsUpdate();
    });

    // 특별 이벤트 (대량 구매 등)
    this.eventSource.addEventListener('milestone', (event) => {
      const milestone = JSON.parse(event.data);
      this.notifyMilestone(milestone);
    });
  }

  private updateStats(sale: SaleEvent) {
    this.stats.totalSales++;
    this.stats.totalRevenue += sale.amount;
    
    // 국가별 통계
    const countryCount = this.stats.topCountries.get(sale.location.country) || 0;
    this.stats.topCountries.set(sale.location.country, countryCount + 1);
  }

  private startStatsCalculation() {
    setInterval(() => {
      // 초당 판매량 계산
      const recentSales = this.salesBuffer.filter(
        sale => Date.now() - sale.timestamp < 1000
      );
      this.stats.salesPerSecond = recentSales.length;
      
      this.notifyStatsUpdate();
    }, 1000);
  }

  // 지도 시각화를 위한 데이터 스트림
  streamToMap(callback: (sale: SaleEvent) => void): () => void {
    const handler = (sale: SaleEvent) => {
      // 애니메이션을 위한 약간의 지연
      setTimeout(() => callback(sale), Math.random() * 500);
    };

    this.saleListeners.add(handler);
    
    return () => {
      this.saleListeners.delete(handler);
    };
  }

  // 실시간 통계 구독
  subscribeToStats(callback: (stats: typeof this.stats) => void): () => void {
    this.statsListeners.add(callback);
    callback(this.stats); // 초기 데이터
    
    return () => {
      this.statsListeners.delete(callback);
    };
  }

  private saleListeners = new Set<(sale: SaleEvent) => void>();
  private statsListeners = new Set<(stats: typeof this.stats) => void>();
  private milestoneListeners = new Set<(milestone: any) => void>();

  private notifySaleEvent(sale: SaleEvent) {
    this.saleListeners.forEach(listener => listener(sale));
  }

  private notifyStatsUpdate() {
    this.statsListeners.forEach(listener => listener(this.stats));
  }

  private notifyMilestone(milestone: any) {
    this.milestoneListeners.forEach(listener => listener(milestone));
  }
}

// React 컴포넌트 (지도 시각화)
const BFCMLiveMap: React.FC = () => {
  const mapRef = useRef<mapboxgl.Map | null>(null);
  const [stats, setStats] = useState({
    totalSales: 0,
    totalRevenue: 0,
    salesPerSecond: 0
  });

  useEffect(() => {
    // Mapbox 초기화
    mapboxgl.accessToken = process.env.MAPBOX_TOKEN!;
    
    const map = new mapboxgl.Map({
      container: mapRef.current!,
      style: 'mapbox://styles/mapbox/dark-v10',
      center: [0, 20],
      zoom: 2
    });

    mapRef.current = map;

    // SSE 스트림 연결
    const stream = new BFCMLiveMapStream();
    
    // 판매 이벤트를 지도에 표시
    const unsubscribeSales = stream.streamToMap((sale) => {
      addSaleToMap(map, sale);
    });

    // 통계 업데이트
    const unsubscribeStats = stream.subscribeToStats((newStats) => {
      setStats({
        totalSales: newStats.totalSales,
        totalRevenue: newStats.totalRevenue,
        salesPerSecond: newStats.salesPerSecond
      });
    });

    return () => {
      unsubscribeSales();
      unsubscribeStats();
      map.remove();
    };
  }, []);

  const addSaleToMap = (map: mapboxgl.Map, sale: SaleEvent) => {
    // 판매 위치에 애니메이션 마커 추가
    const el = document.createElement('div');
    el.className = 'sale-pulse';
    el.style.width = `${Math.min(sale.amount / 100, 50)}px`;
    el.style.height = el.style.width;

    new mapboxgl.Marker(el)
      .setLngLat([sale.location.lng, sale.location.lat])
      .addTo(map);

    // 애니메이션 후 제거
    setTimeout(() => {
      marker.remove();
    }, 3000);

    // 금액에 따른 원 크기 조정
    const size = Math.min(Math.max(sale.amount / 100, 10), 50);
    
    map.addSource(`sale-${sale.id}`, {
      type: 'geojson',
      data: {
        type: 'Feature',
        geometry: {
          type: 'Point',
          coordinates: [sale.location.lng, sale.location.lat]
        }
      }
    });

    map.addLayer({
      id: `sale-circle-${sale.id}`,
      type: 'circle',
      source: `sale-${sale.id}`,
      paint: {
        'circle-radius': {
          stops: [
            [0, size],
            [1, size * 3]
          ]
        },
        'circle-opacity': {
          stops: [
            [0, 0.8],
            [1, 0]
          ]
        },
        'circle-color': '#00ff00'
      }
    });

    // 애니메이션
    let progress = 0;
    const animateCircle = () => {
      progress += 0.02;
      
      if (progress <= 1) {
        map.setPaintProperty(
          `sale-circle-${sale.id}`,
          'circle-radius-transition',
          { duration: 0 }
        );
        
        requestAnimationFrame(animateCircle);
      } else {
        // 애니메이션 완료 후 레이어 제거
        map.removeLayer(`sale-circle-${sale.id}`);
        map.removeSource(`sale-${sale.id}`);
      }
    };
    
    animateCircle();
  };

  return (
    <div className="bfcm-live-map">
      <div className="stats-overlay">
        <div className="stat-card">
          <h3>Total Sales</h3>
          <div className="stat-value">
            {stats.totalSales?.toLocaleString() || '0'}
          </div>
        </div>
        <div className="stat-card">
          <h3>Revenue</h3>
          <div className="stat-value">
            ${stats.totalRevenue?.toLocaleString() || '0'}
          </div>
        </div>
        <div className="stat-card">
          <h3>Sales/Second</h3>
          <div className="stat-value">
            {stats.salesPerSecond?.toFixed(1) || '0'}
          </div>
        </div>
      </div>

      <div 
        ref={mapContainer} 
        className="map-container"
        style={{ width: '100%', height: '100vh' }}
      />
    </div>
  );
};
```

### 주요 교훈

1. **대규모 데이터 처리**: 초당 수천 건의 이벤트를 효율적으로 처리
2. **시각적 최적화**: 애니메이션과 집계를 통한 부드러운 시각화
3. **글로벌 스케일**: 전 세계 데이터를 실시간으로 수집 및 표시
4. **성능 최적화**: 버퍼링과 샘플링으로 클라이언트 부하 관리

## LinkedIn 인스턴트 메시징 사례 연구

### 개요

LinkedIn의 메시징 시스템은 SSE를 활용하여 실시간 메시지 전달, 타이핑 인디케이터, 읽음 확인 등을 구현합니다.

### 기술적 구현

```typescript
// LinkedIn 스타일 메시징 스트림
interface Message {
  id: string;
  conversationId: string;
  senderId: string;
  content: string;
  timestamp: number;
  status: 'sending' | 'sent' | 'delivered' | 'read';
  attachments?: Array<{
    type: string;
    url: string;
    name: string;
  }>;
}

interface TypingIndicator {
  conversationId: string;
  userId: string;
  isTyping: boolean;
}

class LinkedInMessagingStream {
  private eventSource: EventSource;
  private conversations: Map<string, Conversation> = new Map();
  private presenceMap: Map<string, PresenceStatus> = new Map();
  private messageQueue: MessageQueue;

  constructor(private userId: string, private token: string) {
    this.messageQueue = new MessageQueue();
    this.connect();
  }

  private connect() {
    this.eventSource = new EventSource('/api/messaging/stream', {
      headers: {
        'Authorization': `Bearer ${this.token}`
      }
    });

    this.setupEventHandlers();
  }

  private setupEventHandlers() {
    // 새 메시지 수신
    this.eventSource.addEventListener('message', (event) => {
      const message: Message = JSON.parse(event.data);
      this.handleNewMessage(message);
    });

    // 타이핑 인디케이터
    this.eventSource.addEventListener('typing', (event) => {
      const indicator: TypingIndicator = JSON.parse(event.data);
      this.updateTypingStatus(
        indicator.conversationId,
        indicator.userId,
        indicator.isTyping
      );
    });

    // 메시지 상태 업데이트
    this.eventSource.addEventListener('message-status', (event) => {
      const update = JSON.parse(event.data);
      this.updateMessageStatus(
        update.messageId,
        update.status,
        update.conversationId
      );
    });

    // 프레즌스 업데이트
    this.eventSource.addEventListener('presence', (event) => {
      const presence = JSON.parse(event.data);
      this.presenceMap.set(presence.userId, presence.status);
      this.onPresenceUpdate?.(presence.userId, presence.status);
    });

    // 읽음 확인
    this.eventSource.addEventListener('read-receipt', (event) => {
      const receipt = JSON.parse(event.data);
      this.markMessagesAsRead(
        receipt.conversationId,
        receipt.messageId,
        receipt.userId
      );
    });
  }

  private handleNewMessage(message: Message) {
    const conversation = this.getOrCreateConversation(message.conversationId);
    conversation.addMessage(message);
    
    // UI 업데이트 알림
    this.onNewMessage?.(message);
    
    // 자동 읽음 확인 (활성 대화인 경우)
    if (this.activeConversationId === message.conversationId && 
        message.senderId !== this.userId) {
      this.sendReadReceipt(message.conversationId, message.id);
    }
  }

  private updateTypingStatus(
    conversationId: string, 
    userId: string, 
    isTyping: boolean
  ): void {
    const conversation = this.conversations.get(conversationId);
    if (!conversation) return;

    if (isTyping) {
      conversation.typingUsers.add(userId);
      
      // 3초 후 자동 제거
      setTimeout(() => {
        conversation.typingUsers.delete(userId);
        this.onTypingUpdate?.(conversationId, Array.from(conversation.typingUsers));
      }, 3000);
    } else {
      conversation.typingUsers.delete(userId);
    }

    this.onTypingUpdate?.(conversationId, Array.from(conversation.typingUsers));
  }

  sendMessage(conversationId: string, content: string, attachments?: any[]): void {
    const message = {
      conversationId,
      content,
      attachments,
      clientId: generateClientId(),
      timestamp: Date.now()
    };

    // 낙관적 업데이트
    const optimisticMessage: Message = {
      ...message,
      id: message.clientId,
      senderId: this.userId,
      status: 'sending'
    };

    const conversation = this.getOrCreateConversation(conversationId);
    conversation.addMessage(optimisticMessage);

    // 큐에 추가
    this.messageQueue.enqueue(message);
    this.processMessageQueue();
  }

  private async processMessageQueue(): Promise<void> {
    if (this.messageQueue.isProcessing) return;

    this.messageQueue.isProcessing = true;

    while (!this.messageQueue.isEmpty()) {
      const message = this.messageQueue.dequeue();
      
      try {
        const response = await fetch('/api/messages/send', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${this.token}`
          },
          body: JSON.stringify(message)
        });

        if (!response.ok) throw new Error('Failed to send message');

        const sentMessage = await response.json();
        
        // 낙관적 메시지 업데이트
        this.updateMessageStatus(
          message.clientId,
          'delivered',
          message.conversationId,
          sentMessage.id
        );
      } catch (error) {
        // 재시도 로직
        this.messageQueue.enqueue(message);
        this.updateMessageStatus(
          message.clientId,
          'failed',
          message.conversationId
        );
        break;
      }
    }

    this.messageQueue.isProcessing = false;
  }

  // 타이핑 인디케이터 전송
  sendTypingIndicator(conversationId: string): void {
    // 디바운싱
    if (this.typingTimers.has(conversationId)) {
      clearTimeout(this.typingTimers.get(conversationId));
    }

    fetch('/api/typing', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.token}`
      },
      body: JSON.stringify({
        conversationId,
        isTyping: true
      })
    });

    // 3초 후 자동으로 타이핑 중지
    const timer = setTimeout(() => {
      this.stopTyping(conversationId);
    }, 3000);

    this.typingTimers.set(conversationId, timer);
  }

  onNewMessage?: (message: Message) => void;
  onTypingUpdate?: (conversationId: string, typingUsers: string[]) => void;
  onPresenceUpdate?: (userId: string, presence: PresenceStatus) => void;
}

// React Hook
function useLinkedInMessaging(conversationId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [typingUsers, setTypingUsers] = useState<string[]>([]);
  const [connectionStatus, setConnectionStatus] = useState<'connected' | 'disconnected'>('disconnected');
  const streamRef = useRef<LinkedInMessagingStream | null>(null);

  useEffect(() => {
    const stream = new LinkedInMessagingStream(
      getCurrentUserId(),
      getAuthToken()
    );

    stream.onNewMessage = (message) => {
      if (message.conversationId === conversationId) {
        setMessages(prev => [...prev, message]);
      }
    };

    stream.onTypingUpdate = (convId, users) => {
      if (convId === conversationId) {
        setTypingUsers(users);
      }
    };

    streamRef.current = stream;
    stream.connect();

    return () => {
      stream.disconnect();
    };
  }, [conversationId]);

  const sendMessage = useCallback((content: string) => {
    streamRef.current?.sendMessage(conversationId, content);
  }, [conversationId]);

  const sendTypingIndicator = useCallback(() => {
    streamRef.current?.sendTypingIndicator(conversationId);
  }, [conversationId]);

  return {
    messages,
    typingUsers,
    connectionStatus,
    sendMessage,
    sendTypingIndicator
  };
}
```

### 주요 교훈

1. **메시지 큐잉**: 오프라인 지원과 안정적인 메시지 전달
2. **낙관적 UI**: 즉각적인 피드백으로 반응성 향상
3. **프레즌스 추적**: 효율적인 온라인 상태 관리
4. **타이핑 인디케이터**: 실시간 상호작용 피드백

---

**다음 장 예고**: [Part 13: 부록](./part13-appendix.md)에서는 브라우저 호환성, 참고 문서, 용어집, 그리고 유용한 코드 스니펫들을 제공합니다.

---
**[← 이전: Part 11 - 프로덕션 예제](./part11-production-examples.md)** | **[목차](./README.md)** | **[다음: Part 13 - 부록 →](./part13-appendix.md)**

---