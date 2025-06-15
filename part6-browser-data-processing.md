***
**[← 이전: Part 5 - Next.js와 Vercel AI SDK 활용](./part5-nextjs-vercel-ai.md)** | **[목차](./README.md)** | **[다음: Part 7 - 에러 처리와 재연결 메커니즘 →](./part7-error-reconnection.md)**

***

# Part 6: 브라우저에서의 불완전한 데이터 처리

## 7장. 브라우저에서의 불완전한 데이터 처리

SSE 스트리밍에서 가장 중요한 과제 중 하나는 네트워크를 통해 전송되는 데이터가 예측할 수 없는 지점에서 분할될 수 있다는 점입니다. 이 장에서는 이러한 문제를 해결하는 방법을 다룹니다.

### 7.1 청크 분할 문제와 현실

#### 7.1.1 문제의 본질

SSE 데이터는 네트워크 계층에서 임의로 분할될 수 있습니다:

```javascript
// 원본 데이터
"data: {\"message\": \"안녕하세요\"}\n\n"

// 실제로 수신될 수 있는 청크들
청크 1: "data: {\"mes"
청크 2: "sage\": \"안녕"
청크 3: "하세요\"}\n\n"
```#### 7.1.2 청크 분할 시나리오

```typescript
// 다양한 분할 시나리오
interface ChunkScenario {
  description: string;
  chunks: string[];
  expected: any[];
}

const scenarios: ChunkScenario[] = [
  {
    description: "JSON 중간에서 분할",
    chunks: [
      'data: {"type": "messag',
      'e", "content": "Hello"}\n\n'
    ],
    expected: [{ type: "message", content: "Hello" }]
  },
  {
    description: "이벤트 경계에서 분할",
    chunks: [
      'data: {"first": true}\n',
      '\ndata: {"second": true}\n\n'
    ],
    expected: [{ first: true }, { second: true }]
  },
  {
    description: "유니코드 문자 중간 분할",
    chunks: [
      'data: "안',
      '녕하세요"\n\n'
    ],
    expected: ["안녕하세요"]
  }
];
```#### 7.1.3 실제 문제 사례

```typescript
// 실제 프로덕션에서 발생한 문제
class RealWorldIssues {
  // 문제 1: 긴 JSON 응답의 분할
  async handleLongJSON() {
    const response = await fetch('/api/stream');
    const reader = response.body!.getReader();
    const decoder = new TextDecoder();
    
    let buffer = '';
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      // 문제: 이 청크가 JSON 중간을 자를 수 있음
      const chunk = decoder.decode(value, { stream: true });
      
      try {
        // 이 시도는 자주 실패함
        const data = JSON.parse(chunk);
        console.log(data);
      } catch (e) {
        // JSON.parse 에러 발생
        console.error('Incomplete JSON:', chunk);
      }
    }
  }
}
```### 7.2 안전한 파싱 구현

#### 7.2.1 버퍼 기반 파서

```typescript
class SafeSSEParser {
  private buffer: string = '';
  private decoder = new TextDecoder();
  
  parseChunk(chunk: Uint8Array): Array<{ event?: string; data: string; id?: string }> {
    // 청크를 버퍼에 추가
    this.buffer += this.decoder.decode(chunk, { stream: true });
    
    const events: Array<{ event?: string; data: string; id?: string }> = [];
    
    // 완전한 이벤트 찾기 (두 개의 줄바꿈으로 끝남)
    const eventEndIndex = this.buffer.indexOf('\n\n');
    
    while (eventEndIndex !== -1) {
      // 이벤트 추출
      const eventText = this.buffer.substring(0, eventEndIndex);
      this.buffer = this.buffer.substring(eventEndIndex + 2);
      
      // 이벤트 파싱
      const event = this.parseEvent(eventText);
      if (event) {
        events.push(event);
      }
      
      // 다음 이벤트 찾기
      const nextEventEndIndex = this.buffer.indexOf('\n\n');
      if (nextEventEndIndex === -1) break;
    }
    
    return events;
  }  
  private parseEvent(eventText: string): { event?: string; data: string; id?: string } | null {
    const lines = eventText.split('\n');
    const event: any = {};
    
    for (const line of lines) {
      if (line.startsWith('data: ')) {
        event.data = (event.data || '') + line.substring(6);
      } else if (line.startsWith('event: ')) {
        event.event = line.substring(7);
      } else if (line.startsWith('id: ')) {
        event.id = line.substring(4);
      } else if (line.startsWith('retry: ')) {
        // retry는 일반적으로 무시
      } else if (line.startsWith(':')) {
        // 주석 무시
      }
    }
    
    return event.data ? event : null;
  }
  
  // 남은 버퍼 내용 확인
  getBuffer(): string {
    return this.buffer;
  }
  
  // 버퍼 초기화
  reset(): void {
    this.buffer = '';
  }
}```

#### 7.2.2 JSON 안전 파싱

```typescript
class JSONSafeParser {
  private jsonBuffer: string = '';
  
  tryParseJSON(text: string): { success: boolean; data?: any; partial?: string } {
    try {
      const data = JSON.parse(text);
      return { success: true, data };
    } catch (e) {
      // 부분적인 JSON인지 확인
      if (this.isPartialJSON(text)) {
        return { success: false, partial: text };
      }
      
      // 완전히 잘못된 JSON
      return { success: false };
    }
  }
  
  private isPartialJSON(text: string): boolean {
    // 간단한 휴리스틱: 열린 괄호/따옴표 수 계산
    let depth = 0;
    let inString = false;
    let escape = false;
    
    for (let i = 0; i < text.length; i++) {
      const char = text[i];
      
      if (escape) {
        escape = false;
        continue;
      }      
      if (char === '\\') {
        escape = true;
        continue;
      }
      
      if (char === '"' && !inString) {
        inString = true;
      } else if (char === '"' && inString) {
        inString = false;
      }
      
      if (!inString) {
        if (char === '{' || char === '[') depth++;
        if (char === '}' || char === ']') depth--;
      }
    }
    
    // depth가 0보다 크면 아직 닫히지 않은 구조가 있음
    return depth > 0 || inString;
  }
  
  accumulateAndParse(chunk: string): { complete: any[]; partial: string } {
    this.jsonBuffer += chunk;
    const complete: any[] = [];
    
    // 가능한 JSON 객체들을 찾아서 파싱
    let startIndex = 0;
    let depth = 0;
    let inString = false;
    let escape = false;    
    for (let i = 0; i < this.jsonBuffer.length; i++) {
      const char = this.jsonBuffer[i];
      
      if (escape) {
        escape = false;
        continue;
      }
      
      if (char === '\\') {
        escape = true;
        continue;
      }
      
      if (char === '"' && !inString) {
        inString = true;
      } else if (char === '"' && inString) {
        inString = false;
      }
      
      if (!inString) {
        if (char === '{' || char === '[') {
          if (depth === 0) startIndex = i;
          depth++;
        }
        if (char === '}' || char === ']') {
          depth--;
          if (depth === 0) {
            // 완전한 JSON 객체 발견
            const jsonStr = this.jsonBuffer.substring(startIndex, i + 1);
            try {
              const obj = JSON.parse(jsonStr);
              complete.push(obj);
              startIndex = i + 1;
            } catch (e) {
              // 파싱 실패, 계속 진행
            }
          }
        }
      }
    }    
    // 남은 부분 저장
    this.jsonBuffer = this.jsonBuffer.substring(startIndex);
    
    return { complete, partial: this.jsonBuffer };
  }
}
```

#### 7.2.3 통합 SSE 스트림 파서

```typescript
class RobustSSEStreamParser {
  private sseParser = new SafeSSEParser();
  private jsonParser = new JSONSafeParser();
  
  async *parseStream(
    reader: ReadableStreamDefaultReader<Uint8Array>
  ): AsyncGenerator<any> {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      // SSE 이벤트 파싱
      const events = this.sseParser.parseChunk(value);
      
      for (const event of events) {
        if (event.data) {
          // JSON 파싱 시도
          const result = this.jsonParser.tryParseJSON(event.data);
          
          if (result.success) {
            yield {
              type: 'data',
              event: event.event,
              data: result.data,
              id: event.id
            };
          } else if (result.partial) {
            // 부분적인 JSON은 다음 청크를 기다림
            yield {
              type: 'partial',
              data: result.partial
            };
          } else {
            // JSON이 아닌 일반 텍스트
            yield {
              type: 'text',
              event: event.event,
              data: event.data,
              id: event.id
            };
          }
        }
      }
    }    
    // 스트림 종료 시 버퍼 확인
    const remainingSSE = this.sseParser.getBuffer();
    if (remainingSSE) {
      console.warn('Incomplete SSE event in buffer:', remainingSSE);
    }
    
    const { partial } = this.jsonParser.accumulateAndParse('');
    if (partial) {
      console.warn('Incomplete JSON in buffer:', partial);
    }
  }
}
```

### 7.3 검증된 UX 패턴과 전략

#### 7.3.1 Progressive Disclosure 패턴

```typescript
interface ProgressiveUIState {
  status: 'idle' | 'loading' | 'streaming' | 'complete' | 'error';
  partialContent: string;
  fullContent: string;
  metadata: {
    tokensReceived: number;
    startTime: number;
    lastUpdateTime: number;
  };
}

class ProgressiveDisclosureUI {
  private state: ProgressiveUIState = {
    status: 'idle',
    partialContent: '',
    fullContent: '',
    metadata: {
      tokensReceived: 0,
      startTime: 0,
      lastUpdateTime: 0
    }
  };  
  private updateHandlers: Set<(state: ProgressiveUIState) => void> = new Set();
  
  startStreaming() {
    this.state = {
      ...this.state,
      status: 'streaming',
      metadata: {
        ...this.state.metadata,
        startTime: Date.now(),
        lastUpdateTime: Date.now()
      }
    };
    this.notifyUpdate();
  }
  
  appendContent(chunk: string) {
    this.state = {
      ...this.state,
      partialContent: this.state.partialContent + chunk,
      fullContent: this.state.fullContent + chunk,
      metadata: {
        ...this.state.metadata,
        tokensReceived: this.state.metadata.tokensReceived + 1,
        lastUpdateTime: Date.now()
      }
    };
    
    // 부드러운 업데이트를 위한 디바운싱
    this.debouncedUpdate();
  }
  
  private debouncedUpdate = this.debounce(() => {
    this.notifyUpdate();
  }, 16); // 약 60fps
  
  private debounce(func: Function, wait: number) {
    let timeout: NodeJS.Timeout;
    return (...args: any[]) => {
      clearTimeout(timeout);
      timeout = setTimeout(() => func.apply(this, args), wait);
    };
  }  
  private notifyUpdate() {
    this.updateHandlers.forEach(handler => handler(this.state));
  }
  
  onUpdate(handler: (state: ProgressiveUIState) => void) {
    this.updateHandlers.add(handler);
    return () => this.updateHandlers.delete(handler);
  }
}
```

#### 7.3.2 Intent-based Buffering

```typescript
class IntentBasedBuffer {
  private buffer: string = '';
  private intentDetector = new IntentDetector();
  private minBufferTime = 100; // ms
  private maxBufferTime = 500; // ms
  private lastFlushTime = Date.now();
  
  shouldFlush(chunk: string): boolean {
    this.buffer += chunk;
    const currentTime = Date.now();
    const timeSinceLastFlush = currentTime - this.lastFlushTime;
    
    // 의도 감지 기반 플러시
    const intent = this.intentDetector.detect(this.buffer);
    
    if (intent.type === 'sentence_end' || intent.type === 'paragraph_end') {
      // 문장이나 단락이 끝나면 즉시 플러시
      return true;
    }
    
    if (intent.type === 'code_block' && intent.complete) {
      // 완전한 코드 블록이면 플러시
      return true;
    }    
    if (timeSinceLastFlush > this.maxBufferTime) {
      // 최대 버퍼 시간 초과
      return true;
    }
    
    if (timeSinceLastFlush < this.minBufferTime) {
      // 최소 버퍼 시간 미달
      return false;
    }
    
    // 단어 경계에서 플러시
    return intent.type === 'word_boundary';
  }
  
  flush(): string {
    const content = this.buffer;
    this.buffer = '';
    this.lastFlushTime = Date.now();
    return content;
  }
}

class IntentDetector {
  detect(text: string): { type: string; complete?: boolean } {
    // 문장 끝 감지
    if (text.match(/[.!?]\s*$/)) {
      return { type: 'sentence_end' };
    }
    
    // 단락 끝 감지
    if (text.match(/\n\n$/)) {
      return { type: 'paragraph_end' };
    }    
    // 코드 블록 감지
    const codeBlockMatch = text.match(/```[\s\S]*```$/);
    if (codeBlockMatch) {
      return { type: 'code_block', complete: true };
    }
    
    // 불완전한 코드 블록
    if (text.match(/```[\s\S]*$/)) {
      return { type: 'code_block', complete: false };
    }
    
    // 단어 경계 감지
    if (text.match(/\s$/)) {
      return { type: 'word_boundary' };
    }
    
    return { type: 'partial' };
  }
}
```

#### 7.3.3 Skeleton UI와 Placeholder

```typescript
interface SkeletonUIConfig {
  showSkeleton: boolean;
  skeletonLines: number;
  animationType: 'pulse' | 'wave' | 'none';
  transitionDuration: number;
}

class StreamingSkeletonUI {
  private config: SkeletonUIConfig = {
    showSkeleton: true,
    skeletonLines: 3,
    animationType: 'pulse',
    transitionDuration: 300
  };  
  renderSkeleton(): string {
    const lines = Array(this.config.skeletonLines)
      .fill(0)
      .map((_, i) => {
        const width = 80 + Math.random() * 20; // 80-100% 너비
        return `<div class="skeleton-line" style="width: ${width}%"></div>`;
      })
      .join('');
    
    return `
      <div class="skeleton-container ${this.config.animationType}">
        ${lines}
      </div>
    `;
  }
  
  renderStreamingContent(content: string, isComplete: boolean): string {
    if (!content && this.config.showSkeleton && !isComplete) {
      return this.renderSkeleton();
    }
    
    const className = isComplete ? 'content-complete' : 'content-streaming';
    const cursor = isComplete ? '' : '<span class="typing-cursor">▋</span>';
    
    return `
      <div class="${className}" style="transition: all ${this.config.transitionDuration}ms">
        ${this.formatContent(content)}${cursor}
      </div>
    `;
  }  
  private formatContent(content: string): string {
    // 마크다운 렌더링, 코드 하이라이팅 등
    return content
      .replace(/\n/g, '<br>')
      .replace(/```(\w+)?\n([\s\S]*?)```/g, (match, lang, code) => {
        return `<pre><code class="language-${lang || 'text'}">${this.escapeHtml(code.trim())}</code></pre>`;
      });
  }
  
  private escapeHtml(text: string): string {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
  }
}

// CSS 스타일
const skeletonStyles = `
  .skeleton-container {
    padding: 1rem;
  }
  
  .skeleton-line {
    height: 1.2em;
    background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
    background-size: 200% 100%;
    margin-bottom: 0.5em;
    border-radius: 4px;
  }  
  .skeleton-container.pulse .skeleton-line {
    animation: pulse 1.5s ease-in-out infinite;
  }
  
  .skeleton-container.wave .skeleton-line {
    animation: wave 1.5s linear infinite;
  }
  
  @keyframes pulse {
    0% { opacity: 0.6; }
    50% { opacity: 1; }
    100% { opacity: 0.6; }
  }
  
  @keyframes wave {
    0% { background-position: -200% 0; }
    100% { background-position: 200% 0; }
  }
  
  .typing-cursor {
    animation: blink 1s infinite;
  }
  
  @keyframes blink {
    0%, 50% { opacity: 1; }
    51%, 100% { opacity: 0; }
  }
`;
```

### 7.4 Progressive Enhancement 전략#### 7.4.1 점진적 렌더링

```typescript
class ProgressiveRenderer {
  private renderQueue: Array<() => void> = [];
  private isRendering = false;
  private batchSize = 5;
  
  scheduleRender(renderFn: () => void) {
    this.renderQueue.push(renderFn);
    
    if (!this.isRendering) {
      this.processQueue();
    }
  }
  
  private async processQueue() {
    this.isRendering = true;
    
    while (this.renderQueue.length > 0) {
      const batch = this.renderQueue.splice(0, this.batchSize);
      
      // requestAnimationFrame을 사용하여 부드러운 렌더링
      await new Promise(resolve => {
        requestAnimationFrame(() => {
          batch.forEach(fn => fn());
          resolve(undefined);
        });
      });
      
      // 짧은 대기로 UI 반응성 유지
      await new Promise(resolve => setTimeout(resolve, 0));
    }
    
    this.isRendering = false;
  }
}// 사용 예제
class StreamingMessageRenderer {
  private renderer = new ProgressiveRenderer();
  private container: HTMLElement;
  
  constructor(containerId: string) {
    this.container = document.getElementById(containerId)!;
  }
  
  appendChunk(chunk: string) {
    this.renderer.scheduleRender(() => {
      const span = document.createElement('span');
      span.textContent = chunk;
      span.className = 'chunk-fade-in';
      this.container.appendChild(span);
    });
  }
}
```

#### 7.4.2 가상 스크롤링

```typescript
class VirtualScrollingStream {
  private items: string[] = [];
  private container: HTMLElement;
  private itemHeight = 24;
  private visibleCount = 20;
  private scrollTop = 0;
  
  constructor(container: HTMLElement) {
    this.container = container;
    this.setupScrollListener();
  }  
  addItem(text: string) {
    this.items.push(text);
    this.render();
    
    // 자동 스크롤
    if (this.isAtBottom()) {
      this.scrollToBottom();
    }
  }
  
  private setupScrollListener() {
    this.container.addEventListener('scroll', () => {
      this.scrollTop = this.container.scrollTop;
      this.render();
    });
  }
  
  private render() {
    const startIndex = Math.floor(this.scrollTop / this.itemHeight);
    const endIndex = Math.min(
      startIndex + this.visibleCount,
      this.items.length
    );
    
    // 가상 높이 설정
    const totalHeight = this.items.length * this.itemHeight;
    const offsetY = startIndex * this.itemHeight;
    
    this.container.innerHTML = `
      <div style="height: ${totalHeight}px; position: relative;">
        <div style="transform: translateY(${offsetY}px);">
          ${this.items
            .slice(startIndex, endIndex)
            .map(item => `<div style="height: ${this.itemHeight}px">${item}</div>`)
            .join('')}
        </div>
      </div>
    `;
  }  
  private isAtBottom(): boolean {
    const { scrollTop, scrollHeight, clientHeight } = this.container;
    return scrollTop + clientHeight >= scrollHeight - 10;
  }
  
  private scrollToBottom() {
    this.container.scrollTop = this.container.scrollHeight;
  }
}
```

#### 7.4.3 오프스크린 렌더링

```typescript
class OffscreenStreamRenderer {
  private offscreenCanvas: OffscreenCanvas;
  private ctx: OffscreenCanvasRenderingContext2D;
  private mainCanvas: HTMLCanvasElement;
  private mainCtx: CanvasRenderingContext2D;
  
  constructor(canvasId: string) {
    this.mainCanvas = document.getElementById(canvasId) as HTMLCanvasElement;
    this.mainCtx = this.mainCanvas.getContext('2d')!;
    
    // 오프스크린 캔버스 생성
    this.offscreenCanvas = new OffscreenCanvas(
      this.mainCanvas.width,
      this.mainCanvas.height
    );
    this.ctx = this.offscreenCanvas.getContext('2d')!;
  }  
  async renderStreamingText(text: string, x: number, y: number) {
    // 오프스크린에서 텍스트 렌더링
    this.ctx.font = '16px monospace';
    this.ctx.fillStyle = '#000';
    
    // 텍스트를 한 글자씩 렌더링
    for (let i = 0; i < text.length; i++) {
      this.ctx.fillText(text[i], x + i * 10, y);
      
      // 주기적으로 메인 캔버스에 전송
      if (i % 10 === 0) {
        await this.transferToMain();
      }
    }
    
    // 최종 전송
    await this.transferToMain();
  }
  
  private async transferToMain() {
    // 오프스크린 캔버스의 비트맵을 메인 캔버스로 전송
    const bitmap = await createImageBitmap(this.offscreenCanvas);
    this.mainCtx.drawImage(bitmap, 0, 0);
    bitmap.close();
  }
}
```

***

**다음 장 예고**: [Part 7: 에러 처리와 재연결 메커니즘](./part7-error-reconnection.md)에서는 SSE 연결의 안정성을 보장하는 에러 처리와 자동 재연결 구현 방법을 다루겠습니다.

***
**[← 이전: Part 5 - Next.js와 Vercel AI SDK 활용](./part5-nextjs-vercel-ai.md)** | **[목차](./README.md)** | **[다음: Part 7 - 에러 처리와 재연결 메커니즘 →](./part7-error-reconnection.md)**

***