***
**[← 이전: Part 3 - 빅테크의 비공식 SSE 패턴](./part3-bigtech-patterns.md)** | **[목차](./README.md)** | **[다음: Part 5 - Next.js와 Vercel AI SDK 활용 →](./part5-nextjs-vercel-ai.md)**

***

# Part 4: AI 챗봇 서비스의 SSE 구현

## 5장. AI 챗봇 서비스의 SSE 구현

AI 챗봇 서비스들은 SSE를 활용하여 대규모 언어 모델(LLM)의 응답을 실시간으로 스트리밍합니다. 이 장에서는 OpenAI, Anthropic Claude 등 주요 AI 서비스의 SSE 구현 패턴을 상세히 분석합니다.

### 5.1 OpenAI ChatGPT SSE 메시지 형식

#### 5.1.1 기본 스트리밍 구조

OpenAI의 Chat Completions API는 다음과 같은 SSE 형식을 사용합니다:

```
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-3.5-turbo","choices":[{"index":0,"delta":{"role":"assistant","content":""},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-3.5-turbo","choices":[{"index":0,"delta":{"content":"안녕"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-3.5-turbo","choices":[{"index":0,"delta":{"content":"하세요"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-3.5-turbo","choices":[{"index":0,"delta":{"content":"!"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-3.5-turbo","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

#### 5.1.2 OpenAI 스트리밍 응답 파서

```typescript
interface OpenAIStreamChunk {
  id: string;
  object: string;
  created: number;
  model: string;
  choices: Array<{
    index: number;
    delta: {
      role?: string;
      content?: string;
      function_call?: {
        name?: string;
        arguments?: string;
      };
    };
    finish_reason: string | null;
  }>;
  usage?: {
    prompt_tokens: number;
    completion_tokens: number;
    total_tokens: number;
  };
}

class OpenAIStreamParser {
  private buffer: string = '';
  
  parseSSEMessage(data: string): OpenAIStreamChunk | null {
    if (data === '[DONE]') {
      return null;
    }
    
    try {
      return JSON.parse(data);
    } catch (error) {
      console.error('Failed to parse OpenAI chunk:', error);
      return null;
    }
  }
  
  extractContent(chunk: OpenAIStreamChunk): string {
    return chunk.choices[0]?.delta?.content || '';
  }
  
  isComplete(chunk: OpenAIStreamChunk): boolean {
    return chunk.choices[0]?.finish_reason !== null;
  }
  
  getFinishReason(chunk: OpenAIStreamChunk): string | null {
    return chunk.choices[0]?.finish_reason;
  }
}
```

#### 5.1.3 스트리밍 클라이언트 구현

```typescript
class OpenAIStreamingClient {
  private apiKey: string;
  private baseURL: string = 'https://api.openai.com/v1';
  
  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }
  
  async *streamChatCompletion(
    messages: Array<{role: string; content: string}>,
    options: {
      model?: string;
      temperature?: number;
      max_tokens?: number;
    } = {}
  ): AsyncGenerator<string> {
    const response = await fetch(`${this.baseURL}/chat/completions`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.apiKey}`,
      },
      body: JSON.stringify({
        model: options.model || 'gpt-3.5-turbo',
        messages,
        stream: true,
        temperature: options.temperature,
        max_tokens: options.max_tokens,
      }),
    });
    
    if (!response.ok) {
      throw new Error(`OpenAI API error: ${response.status}`);
    }
    
    const reader = response.body!.getReader();
    const decoder = new TextDecoder();
    const parser = new OpenAIStreamParser();
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
          const chunk = parser.parseSSEMessage(data);
          
          if (chunk === null) {
            // [DONE] 메시지
            return;
          }
          
          const content = parser.extractContent(chunk);
          if (content) {
            yield content;
          }
          
          if (parser.isComplete(chunk)) {
            return;
          }
        }
      }
    }
  }
}
```

### 5.2 Anthropic Claude SSE 메시지 형식

#### 5.2.1 Claude의 이벤트 기반 스트리밍

Anthropic Claude는 더 구조화된 이벤트 기반 접근을 사용합니다:

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_123","type":"message","role":"assistant","content":[],"model":"claude-3-sonnet-20240229","stop_reason":null,"stop_sequence":null,"usage":{"input_tokens":25,"output_tokens":1}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: ping
data: {"type":"ping"}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"안녕"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"하세요"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"!"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":12}}

event: message_stop
data: {"type":"message_stop"}
```

#### 5.2.2 Claude 이벤트 타입 정의

```typescript
// Claude 이벤트 타입들
type ClaudeEventType = 
  | 'message_start'
  | 'content_block_start'
  | 'content_block_delta'
  | 'content_block_stop'
  | 'message_delta'
  | 'message_stop'
  | 'ping'
  | 'error';

interface ClaudeMessageStart {
  type: 'message_start';
  message: {
    id: string;
    type: 'message';
    role: 'assistant';
    content: any[];
    model: string;
    stop_reason: string | null;
    stop_sequence: string | null;
    usage: {
      input_tokens: number;
      output_tokens: number;
    };
  };
}

interface ClaudeContentBlockDelta {
  type: 'content_block_delta';
  index: number;
  delta: {
    type: 'text_delta';
    text: string;
  };
}

interface ClaudeMessageStop {
  type: 'message_stop';
}
```

#### 5.2.3 Claude 스트리밍 파서

```typescript
class ClaudeStreamParser {
  private eventHandlers: Map<ClaudeEventType, Function> = new Map();
  
  constructor() {
    this.setupDefaultHandlers();
  }
  
  private setupDefaultHandlers() {
    this.on('message_start', (data: ClaudeMessageStart) => {
      console.log('Message started:', data.message.id);
    });
    
    this.on('ping', () => {
      // Keep-alive ping, no action needed
    });
  }
  
  on(eventType: ClaudeEventType, handler: Function) {
    this.eventHandlers.set(eventType, handler);
  }
  
  parseSSEEvent(eventLine: string, dataLine: string) {
    const eventType = eventLine.replace('event: ', '') as ClaudeEventType;
    const data = JSON.parse(dataLine.replace('data: ', ''));
    
    const handler = this.eventHandlers.get(eventType);
    if (handler) {
      handler(data);
    }
    
    return { eventType, data };
  }
  
  async *parseStream(reader: ReadableStreamDefaultReader<Uint8Array>): AsyncGenerator<string> {
    const decoder = new TextDecoder();
    let buffer = '';
    let currentEvent = '';
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';
      
      for (const line of lines) {
        if (line.startsWith('event: ')) {
          currentEvent = line;
        } else if (line.startsWith('data: ') && currentEvent) {
          const { eventType, data } = this.parseSSEEvent(currentEvent, line);
          
          if (eventType === 'content_block_delta') {
            yield data.delta.text;
          } else if (eventType === 'message_stop') {
            return;
          }
          
          currentEvent = '';
        }
      }
    }
  }
}
```

### 5.3 Tool Calling과 함수 호출

#### 5.3.1 OpenAI Function Calling 스트리밍

```
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-4","choices":[{"index":0,"delta":{"role":"assistant","content":null,"function_call":{"name":"get_weather","arguments":""}},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-4","choices":[{"index":0,"delta":{"function_call":{"arguments":"{\n"}},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-4","choices":[{"index":0,"delta":{"function_call":{"arguments":"  \"location\":"}},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-4","choices":[{"index":0,"delta":{"function_call":{"arguments":" \"Seoul\"\n}"}},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1677652288,"model":"gpt-4","choices":[{"index":0,"delta":{},"finish_reason":"function_call"}]}
```

#### 5.3.2 Tool Call 스트리밍 처리

```typescript
class ToolCallStreamHandler {
  private functionCallBuffer: {
    name?: string;
    arguments: string;
  } = { arguments: '' };
  
  handleFunctionCallDelta(delta: any): boolean {
    if (delta.function_call) {
      if (delta.function_call.name) {
        this.functionCallBuffer.name = delta.function_call.name;
      }
      if (delta.function_call.arguments) {
        this.functionCallBuffer.arguments += delta.function_call.arguments;
      }
      return true;
    }
    return false;
  }
  
  getFunctionCall(): { name: string; arguments: any } | null {
    if (!this.functionCallBuffer.name) {
      return null;
    }
    
    try {
      const args = JSON.parse(this.functionCallBuffer.arguments);
      return {
        name: this.functionCallBuffer.name,
        arguments: args
      };
    } catch {
      // 아직 완전한 JSON이 아님
      return null;
    }
  }
  
  reset() {
    this.functionCallBuffer = { arguments: '' };
  }
}

// 사용 예제
const toolHandler = new ToolCallStreamHandler();

for await (const chunk of streamResponse) {
  const delta = chunk.choices[0].delta;
  
  if (toolHandler.handleFunctionCallDelta(delta)) {
    // 함수 호출 처리 중
    const functionCall = toolHandler.getFunctionCall();
    if (functionCall) {
      console.log('Function call complete:', functionCall);
      // 함수 실행
      const result = await executeFunction(functionCall.name, functionCall.arguments);
      // 결과를 다시 스트림에 포함
    }
  } else if (delta.content) {
    // 일반 텍스트 콘텐츠
    yield delta.content;
  }
}
```

### 5.4 구조화된 출력

#### 5.4.1 JSON 모드 스트리밍

```typescript
interface StructuredOutputHandler {
  onPartialJSON(partial: string): void;
  onCompleteJSON(data: any): void;
  onError(error: Error): void;
}

class JSONStreamParser implements StructuredOutputHandler {
  private buffer: string = '';
  private depth: number = 0;
  
  processChunk(chunk: string) {
    this.buffer += chunk;
    
    // 간단한 JSON 유효성 검사
    for (const char of chunk) {
      if (char === '{' || char === '[') this.depth++;
      if (char === '}' || char === ']') this.depth--;
    }
    
    // 부분 JSON 콜백
    this.onPartialJSON(this.buffer);
    
    // 완전한 JSON인지 확인
    if (this.depth === 0 && this.buffer.trim()) {
      try {
        const parsed = JSON.parse(this.buffer);
        this.onCompleteJSON(parsed);
        this.buffer = '';
      } catch (error) {
        // 아직 완전하지 않음
      }
    }
  }
  
  onPartialJSON(partial: string) {
    // UI에 부분적인 JSON 표시
    console.log('Partial JSON:', partial);
  }
  
  onCompleteJSON(data: any) {
    console.log('Complete JSON:', data);
  }
  
  onError(error: Error) {
    console.error('JSON parsing error:', error);
  }
}
```

#### 5.4.2 스키마 기반 검증

```typescript
import { z } from 'zod';

// 응답 스키마 정의
const ResponseSchema = z.object({
  summary: z.string(),
  keyPoints: z.array(z.string()),
  sentiment: z.enum(['positive', 'negative', 'neutral']),
  confidence: z.number().min(0).max(1)
});

class SchemaValidatedStream {
  private schema: z.ZodType<any>;
  private buffer: string = '';
  
  constructor(schema: z.ZodType<any>) {
    this.schema = schema;
  }
  
  async *validateStream(stream: AsyncGenerator<string>): AsyncGenerator<{
    partial: string;
    valid: boolean;
    data?: any;
    errors?: z.ZodError;
  }> {
    for await (const chunk of stream) {
      this.buffer += chunk;
      
      try {
        // 부분 JSON 파싱 시도
        const partialData = JSON.parse(this.buffer);
        
        // 스키마 검증
        const result = this.schema.safeParse(partialData);
        
        yield {
          partial: this.buffer,
          valid: result.success,
          data: result.success ? result.data : undefined,
          errors: !result.success ? result.error : undefined
        };
      } catch {
        // 아직 유효한 JSON이 아님
        yield {
          partial: this.buffer,
          valid: false
        };
      }
    }
  }
}
```

### 5.5 Thinking/Reasoning 스트리밍

#### 5.5.1 Claude의 Thinking 블록

```typescript
// Claude의 thinking 프로세스 스트리밍
interface ClaudeThinkingBlock {
  type: 'thinking';
  content: string;
}

interface ClaudeResponseBlock {
  type: 'response';
  content: string;
}

class ClaudeThinkingParser {
  private currentBlock: 'thinking' | 'response' | null = null;
  private thinkingBuffer: string = '';
  private responseBuffer: string = '';
  
  parseContentDelta(delta: any): {
    thinking?: string;
    response?: string;
  } {
    const text = delta.text || '';
    
    // <thinking> 태그 감지
    if (text.includes('<thinking>')) {
      this.currentBlock = 'thinking';
      const startIdx = text.indexOf('<thinking>') + 10;
      this.thinkingBuffer = text.substring(startIdx);
      return {};
    }
    
    // </thinking> 태그 감지
    if (text.includes('</thinking>')) {
      const endIdx = text.indexOf('</thinking>');
      this.thinkingBuffer += text.substring(0, endIdx);
      this.currentBlock = 'response';
      
      const remainingText = text.substring(endIdx + 11);
      if (remainingText) {
        this.responseBuffer += remainingText;
        return {
          thinking: this.thinkingBuffer,
          response: remainingText
        };
      }
      
      return { thinking: this.thinkingBuffer };
    }
    
    // 현재 블록에 따라 버퍼링
    if (this.currentBlock === 'thinking') {
      this.thinkingBuffer += text;
      return {};
    } else {
      this.responseBuffer += text;
      return { response: text };
    }
  }
}

// UI에서 thinking 프로세스 표시
class ThinkingUI {
  private thinkingElement: HTMLElement;
  private responseElement: HTMLElement;
  
  constructor() {
    this.thinkingElement = document.getElementById('thinking')!;
    this.responseElement = document.getElementById('response')!;
  }
  
  showThinking(text: string) {
    this.thinkingElement.style.display = 'block';
    this.thinkingElement.innerHTML = `
      <div class="thinking-process">
        <div class="thinking-header">
          <span class="thinking-icon">🤔</span>
          <span>AI가 생각하는 중...</span>
        </div>
        <div class="thinking-content">${text}</div>
      </div>
    `;
  }
  
  appendResponse(text: string) {
    this.responseElement.innerHTML += text;
  }
  
  completeThinking() {
    this.thinkingElement.classList.add('thinking-complete');
    setTimeout(() => {
      this.thinkingElement.style.display = 'none';
    }, 2000);
  }
}
```

#### 5.5.2 스트리밍 상태 관리

```typescript
enum StreamingState {
  Idle = 'idle',
  Connecting = 'connecting',
  Thinking = 'thinking',
  Responding = 'responding',
  FunctionCalling = 'function_calling',
  Complete = 'complete',
  Error = 'error'
}

class StreamStateManager {
  private state: StreamingState = StreamingState.Idle;
  private listeners: Set<(state: StreamingState) => void> = new Set();
  
  setState(newState: StreamingState) {
    this.state = newState;
    this.notifyListeners();
  }
  
  getState(): StreamingState {
    return this.state;
  }
  
  onStateChange(listener: (state: StreamingState) => void) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }
  
  private notifyListeners() {
    this.listeners.forEach(listener => listener(this.state));
  }
}

// React Hook 예제
function useStreamingState() {
  const [state, setState] = useState<StreamingState>(StreamingState.Idle);
  const stateManager = useRef(new StreamStateManager());
  
  useEffect(() => {
    return stateManager.current.onStateChange(setState);
  }, []);
  
  const streamMessage = useCallback(async (message: string) => {
    stateManager.current.setState(StreamingState.Connecting);
    
    try {
      const stream = await createStream(message);
      stateManager.current.setState(StreamingState.Responding);
      
      for await (const chunk of stream) {
        // 청크 처리
        if (chunk.type === 'thinking') {
          stateManager.current.setState(StreamingState.Thinking);
        } else if (chunk.type === 'function_call') {
          stateManager.current.setState(StreamingState.FunctionCalling);
        }
      }
      
      stateManager.current.setState(StreamingState.Complete);
    } catch (error) {
      stateManager.current.setState(StreamingState.Error);
    }
  }, []);
  
  return { state, streamMessage };
}
```

#### 5.5.3 멀티모달 스트리밍

```typescript
// 이미지 생성과 텍스트를 함께 스트리밍
interface MultimodalChunk {
  type: 'text' | 'image' | 'code' | 'table';
  content: string;
  metadata?: {
    language?: string;
    imageUrl?: string;
    mimeType?: string;
  };
}

class MultimodalStreamHandler {
  private handlers: Map<string, (chunk: MultimodalChunk) => void> = new Map();
  
  constructor() {
    this.setupHandlers();
  }
  
  private setupHandlers() {
    this.handlers.set('text', (chunk) => {
      document.getElementById('output')!.innerHTML += chunk.content;
    });
    
    this.handlers.set('image', async (chunk) => {
      const img = document.createElement('img');
      img.src = chunk.metadata?.imageUrl || '';
      img.alt = chunk.content;
      document.getElementById('output')!.appendChild(img);
    });
    
    this.handlers.set('code', (chunk) => {
      const pre = document.createElement('pre');
      const code = document.createElement('code');
      code.className = `language-${chunk.metadata?.language || 'plaintext'}`;
      code.textContent = chunk.content;
      pre.appendChild(code);
      document.getElementById('output')!.appendChild(pre);
      
      // 구문 강조 적용
      if (window.Prism) {
        window.Prism.highlightElement(code);
      }
    });
    
    this.handlers.set('table', (chunk) => {
      const tableData = JSON.parse(chunk.content);
      const table = this.createTable(tableData);
      document.getElementById('output')!.appendChild(table);
    });
  }
  
  handleChunk(chunk: MultimodalChunk) {
    const handler = this.handlers.get(chunk.type);
    if (handler) {
      handler(chunk);
    }
  }
  
  private createTable(data: any[][]): HTMLTableElement {
    const table = document.createElement('table');
    // 테이블 생성 로직
    return table;
  }
}
```

***

**다음 장 예고**: [Part 5: Next.js와 Vercel AI SDK 활용](./part5-nextjs-vercel-ai.md)에서는 Next.js 환경에서 Vercel AI SDK를 사용하여 프로덕션 레벨의 AI 챗봇을 구현하는 방법을 자세히 다루겠습니다.

***
**[← 이전: Part 3 - 빅테크의 비공식 SSE 패턴](./part3-bigtech-patterns.md)** | **[목차](./README.md)** | **[다음: Part 5 - Next.js와 Vercel AI SDK 활용 →](./part5-nextjs-vercel-ai.md)**

***