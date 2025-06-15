***
**[← 이전: Part 10 - 테스팅 가이드](./part10-testing-guide.md)** | **[목차](./README.md)** | **[다음: Part 12 - 실제 기업 사례 →](./part12-real-world-cases.md)**

***

# Part 11: 프로덕션 예제 (Production Examples)

## 목차
- [완성된 스트리밍 채팅 컴포넌트](#완성된-스트리밍-채팅-컴포넌트)
- [커스텀 프로바이더 구현](#커스텀-프로바이더-구현)
- [재사용 가능한 UI 컴포넌트 라이브러리](#재사용-가능한-ui-컴포넌트-라이브러리)
- [상태 머신 기반 UI 관리](#상태-머신-기반-ui-관리)

## 완성된 스트리밍 채팅 컴포넌트

### 전체 구조

```typescript
// types/chat.types.ts
export interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: number;
  status: 'pending' | 'streaming' | 'completed' | 'error';
  metadata?: {
    model?: string;
    tokens?: number;
    latency?: number;
  };
}

export interface ChatState {
  messages: Message[];
  isConnected: boolean;
  isStreaming: boolean;
  error: Error | null;
}

export interface ChatConfig {
  endpoint: string;
  apiKey?: string;
  model?: string;
  maxRetries?: number;
  reconnectDelay?: number;
  streamingOptions?: {
    throttleMs?: number;
    bufferSize?: number;
  };
}
```

### 메인 채팅 컴포넌트

```tsx
// components/StreamingChat.tsx
import React, { useCallback, useEffect, useRef, useState } from 'react';
import { ChatConfig, ChatState, Message } from '../types/chat.types';
import { SSEManager } from '../services/SSEManager';
import { MessageList } from './MessageList';
import { ChatInput } from './ChatInput';
import { ConnectionStatus } from './ConnectionStatus';
import { useAutoScroll } from '../hooks/useAutoScroll';
import { useChatState } from '../hooks/useChatState';

export const StreamingChat: React.FC<{ config: ChatConfig }> = ({ config }) => {
  const [state, dispatch] = useChatState();
  const sseManagerRef = useRef<SSEManager | null>(null);
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const { shouldAutoScroll, handleScroll } = useAutoScroll();

  // SSE Manager 초기화
  useEffect(() => {
    sseManagerRef.current = new SSEManager(config);
    
    const manager = sseManagerRef.current;
    
    // 이벤트 리스너 등록
    manager.on('connected', () => {
      dispatch({ type: 'SET_CONNECTED', payload: true });
    });

    manager.on('message', (data: Partial<Message>) => {
      dispatch({ type: 'UPDATE_MESSAGE', payload: data });
    });

    manager.on('stream_start', (data: { messageId: string }) => {
      dispatch({ 
        type: 'START_STREAMING', 
        payload: data.messageId 
      });
    });

    manager.on('stream_chunk', (data: { messageId: string; chunk: string }) => {
      dispatch({ 
        type: 'APPEND_CHUNK', 
        payload: data 
      });
    });

    manager.on('stream_end', (data: { messageId: string; metadata?: any }) => {
      dispatch({ 
        type: 'END_STREAMING', 
        payload: data 
      });
    });

    manager.on('error', (error: Error) => {
      dispatch({ type: 'SET_ERROR', payload: error });
    });

    manager.on('disconnected', () => {
      dispatch({ type: 'SET_CONNECTED', payload: false });
    });

    // 연결 시작
    manager.connect();

    return () => {
      manager.disconnect();
    };
  }, [config, dispatch]);

  // 메시지 전송
  const sendMessage = useCallback(async (content: string) => {
    if (!sseManagerRef.current || !state.isConnected) {
      return;
    }

    const userMessage: Message = {
      id: `msg_${Date.now()}_user`,
      role: 'user',
      content,
      timestamp: Date.now(),
      status: 'completed'
    };

    dispatch({ type: 'ADD_MESSAGE', payload: userMessage });

    const assistantMessage: Message = {
      id: `msg_${Date.now()}_assistant`,
      role: 'assistant',
      content: '',
      timestamp: Date.now(),
      status: 'pending'
    };

    dispatch({ type: 'ADD_MESSAGE', payload: assistantMessage });

    try {
      await sseManagerRef.current.sendMessage({
        messageId: assistantMessage.id,
        content,
        context: state.messages.slice(-10) // 최근 10개 메시지를 컨텍스트로
      });
    } catch (error) {
      dispatch({ 
        type: 'UPDATE_MESSAGE', 
        payload: {
          id: assistantMessage.id,
          status: 'error',
          content: 'Failed to send message'
        }
      });
    }
  }, [state.isConnected, state.messages, dispatch]);

  // 자동 스크롤
  useEffect(() => {
    if (shouldAutoScroll) {
      messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
    }
  }, [state.messages, shouldAutoScroll]);

  // 재연결
  const handleReconnect = useCallback(() => {
    sseManagerRef.current?.reconnect();
  }, []);

  return (
    <div className="streaming-chat">
      <ConnectionStatus
        isConnected={state.isConnected}
        error={state.error}
        onReconnect={handleReconnect}
      />
      
      <div 
        className="chat-messages"
        onScroll={handleScroll}
      >
        <MessageList 
          messages={state.messages}
          isStreaming={state.isStreaming}
        />
        <div ref={messagesEndRef} />
      </div>

      <ChatInput
        onSendMessage={sendMessage}
        disabled={!state.isConnected || state.isStreaming}
        placeholder={
          !state.isConnected 
            ? "Connecting..." 
            : state.isStreaming 
            ? "AI is responding..." 
            : "Type your message..."
        }
      />
    </div>
  );
};
```

### SSE 매니저 서비스

```typescript
// services/SSEManager.ts
import { EventEmitter } from 'eventemitter3';
import { ChatConfig } from '../types/chat.types';

interface SSEManagerEvents {
  connected: () => void;
  disconnected: () => void;
  message: (data: any) => void;
  stream_start: (data: { messageId: string }) => void;
  stream_chunk: (data: { messageId: string; chunk: string }) => void;
  stream_end: (data: { messageId: string; metadata?: any }) => void;
  error: (error: Error) => void;
}

export class SSEManager extends EventEmitter<SSEManagerEvents> {
  private eventSource: EventSource | null = null;
  private config: ChatConfig;
  private reconnectTimer: NodeJS.Timeout | null = null;
  private reconnectAttempts = 0;
  private messageBuffer: Map<string, string[]> = new Map();

  constructor(config: ChatConfig) {
    super();
    this.config = config;
  }

  connect(): void {
    if (this.eventSource) {
      return;
    }

    try {
      const url = new URL(this.config.endpoint);
      if (this.config.apiKey) {
        url.searchParams.append('apiKey', this.config.apiKey);
      }

      this.eventSource = new EventSource(url.toString(), {
        withCredentials: true
      });

      this.setupEventHandlers();
    } catch (error) {
      this.emit('error', new Error(`Failed to connect: ${error.message}`));
    }
  }

  private setupEventHandlers(): void {
    if (!this.eventSource) return;

    this.eventSource.onopen = () => {
      this.reconnectAttempts = 0;
      this.emit('connected');
    };

    this.eventSource.onerror = (event) => {
      this.emit('error', new Error('Connection error'));
      this.scheduleReconnect();
    };

    // 스트리밍 시작
    this.eventSource.addEventListener('stream_start', (event) => {
      const data = JSON.parse(event.data);
      this.messageBuffer.set(data.messageId, []);
      this.emit('stream_start', data);
    });

    // 스트리밍 청크
    this.eventSource.addEventListener('stream_chunk', (event) => {
      const data = JSON.parse(event.data);
      
      // 버퍼에 청크 추가
      const buffer = this.messageBuffer.get(data.messageId) || [];
      buffer.push(data.chunk);
      this.messageBuffer.set(data.messageId, buffer);

      // 스로틀링 적용
      if (this.config.streamingOptions?.throttleMs) {
        this.throttleEmit('stream_chunk', data);
      } else {
        this.emit('stream_chunk', data);
      }
    });

    // 스트리밍 종료
    this.eventSource.addEventListener('stream_end', (event) => {
      const data = JSON.parse(event.data);
      
      // 버퍼 정리
      this.messageBuffer.delete(data.messageId);
      
      this.emit('stream_end', data);
    });

    // 일반 메시지
    this.eventSource.addEventListener('message', (event) => {
      try {
        const data = JSON.parse(event.data);
        this.emit('message', data);
      } catch (error) {
        this.emit('error', new Error('Invalid message format'));
      }
    });

    // 하트비트
    this.eventSource.addEventListener('heartbeat', () => {
      // 연결 상태 유지
    });
  }

  private throttleEmit = (() => {
    const throttleMap = new Map<string, NodeJS.Timeout>();
    
    return (event: string, data: any) => {
      const key = `${event}_${data.messageId}`;
      const delay = this.config.streamingOptions?.throttleMs || 0;

      if (throttleMap.has(key)) {
        clearTimeout(throttleMap.get(key)!);
      }

      const timeout = setTimeout(() => {
        this.emit(event as any, data);
        throttleMap.delete(key);
      }, delay);

      throttleMap.set(key, timeout);
    };
  })();

  disconnect(): void {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null;
    }

    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
      this.emit('disconnected');
    }

    this.messageBuffer.clear();
  }

  reconnect(): void {
    this.disconnect();
    this.connect();
  }

  private scheduleReconnect(): void {
    if (this.reconnectTimer || !this.config.maxRetries) {
      return;
    }

    if (this.reconnectAttempts >= this.config.maxRetries) {
      this.emit('error', new Error('Max reconnection attempts reached'));
      return;
    }

    const delay = this.config.reconnectDelay || 1000;
    const backoffDelay = delay * Math.pow(2, this.reconnectAttempts);

    this.reconnectTimer = setTimeout(() => {
      this.reconnectAttempts++;
      this.reconnectTimer = null;
      this.connect();
    }, backoffDelay);
  }

  async sendMessage(payload: any): Promise<void> {
    const response = await fetch(this.config.endpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...(this.config.apiKey && { 'Authorization': `Bearer ${this.config.apiKey}` })
      },
      body: JSON.stringify({
        ...payload,
        model: this.config.model || 'gpt-3.5-turbo',
        stream: true
      })
    });

    if (!response.ok) {
      throw new Error(`Request failed: ${response.statusText}`);
    }
  }
}
```

### 상태 관리 훅

```typescript
// hooks/useChatState.ts
import { useReducer, useCallback } from 'react';
import { ChatState, Message } from '../types/chat.types';

type ChatAction =
  | { type: 'SET_CONNECTED'; payload: boolean }
  | { type: 'ADD_MESSAGE'; payload: Message }
  | { type: 'UPDATE_MESSAGE'; payload: Partial<Message> & { id: string } }
  | { type: 'START_STREAMING'; payload: string }
  | { type: 'APPEND_CHUNK'; payload: { messageId: string; chunk: string } }
  | { type: 'END_STREAMING'; payload: { messageId: string; metadata?: any } }
  | { type: 'SET_ERROR'; payload: Error | null }
  | { type: 'CLEAR_MESSAGES' };

const initialState: ChatState = {
  messages: [],
  isConnected: false,
  isStreaming: false,
  error: null
};

function chatReducer(state: ChatState, action: ChatAction): ChatState {
  switch (action.type) {
    case 'SET_CONNECTED':
      return {
        ...state,
        isConnected: action.payload,
        error: action.payload ? null : state.error
      };

    case 'ADD_MESSAGE':
      return {
        ...state,
        messages: [...state.messages, action.payload]
      };

    case 'UPDATE_MESSAGE':
      return {
        ...state,
        messages: state.messages.map(msg =>
          msg.id === action.payload.id
            ? { ...msg, ...action.payload }
            : msg
        )
      };

    case 'START_STREAMING':
      return {
        ...state,
        isStreaming: true,
        messages: state.messages.map(msg =>
          msg.id === action.payload
            ? { ...msg, status: 'streaming' }
            : msg
        )
      };

    case 'APPEND_CHUNK':
      return {
        ...state,
        messages: state.messages.map(msg =>
          msg.id === action.payload.messageId
            ? { ...msg, content: msg.content + action.payload.chunk }
            : msg
        )
      };

    case 'END_STREAMING':
      return {
        ...state,
        isStreaming: false,
        messages: state.messages.map(msg =>
          msg.id === action.payload.messageId
            ? { 
                ...msg, 
                status: 'completed',
                metadata: action.payload.metadata 
              }
            : msg
        )
      };

    case 'SET_ERROR':
      return {
        ...state,
        error: action.payload
      };

    case 'CLEAR_MESSAGES':
      return {
        ...state,
        messages: []
      };

    default:
      return state;
  }
}

export function useChatState() {
  const [state, dispatch] = useReducer(chatReducer, initialState);

  const clearMessages = useCallback(() => {
    dispatch({ type: 'CLEAR_MESSAGES' });
  }, []);

  return [state, dispatch, { clearMessages }] as const;
}
```

## 커스텀 프로바이더 구현

### SSE 프로바이더 컨텍스트

```tsx
// contexts/SSEProvider.tsx
import React, { createContext, useContext, useEffect, useRef, useState } from 'react';
import { SSEConnection } from '../services/SSEConnection';

interface SSEContextValue {
  subscribe: (channel: string, handler: (data: any) => void) => () => void;
  publish: (channel: string, data: any) => Promise<void>;
  getConnectionState: () => ConnectionState;
  reconnect: () => void;
}

interface ConnectionState {
  status: 'connecting' | 'connected' | 'disconnected' | 'error';
  error?: Error;
  retries: number;
}

const SSEContext = createContext<SSEContextValue | null>(null);

export interface SSEProviderProps {
  endpoint: string;
  options?: {
    reconnect?: boolean;
    reconnectDelay?: number;
    maxReconnectAttempts?: number;
    heartbeatInterval?: number;
    headers?: Record<string, string>;
  };
  children: React.ReactNode;
}

export const SSEProvider: React.FC<SSEProviderProps> = ({
  endpoint,
  options = {},
  children
}) => {
  const connectionRef = useRef<SSEConnection | null>(null);
  const [connectionState, setConnectionState] = useState<ConnectionState>({
    status: 'connecting',
    retries: 0
  });

  useEffect(() => {
    const connection = new SSEConnection(endpoint, {
      ...options,
      onStateChange: setConnectionState
    });

    connectionRef.current = connection;
    connection.connect();

    return () => {
      connection.disconnect();
    };
  }, [endpoint, options]);

  const value: SSEContextValue = {
    subscribe: (channel, handler) => {
      if (!connectionRef.current) {
        throw new Error('SSE connection not initialized');
      }
      return connectionRef.current.subscribe(channel, handler);
    },

    publish: async (channel, data) => {
      if (!connectionRef.current) {
        throw new Error('SSE connection not initialized');
      }
      return connectionRef.current.publish(channel, data);
    },

    getConnectionState: () => connectionState,

    reconnect: () => {
      connectionRef.current?.reconnect();
    }
  };

  return (
    <SSEContext.Provider value={value}>
      {children}
    </SSEContext.Provider>
  );
};

export function useSSE() {
  const context = useContext(SSEContext);
  if (!context) {
    throw new Error('useSSE must be used within SSEProvider');
  }
  return context;
}

// 특정 채널 구독을 위한 훅
export function useSSEChannel<T = any>(
  channel: string,
  handler: (data: T) => void
) {
  const { subscribe, getConnectionState } = useSSE();
  const [lastData, setLastData] = useState<T | null>(null);

  useEffect(() => {
    const unsubscribe = subscribe(channel, (data: T) => {
      setLastData(data);
      handler(data);
    });

    return unsubscribe;
  }, [channel, handler, subscribe]);

  return {
    lastData,
    connectionState: getConnectionState()
  };
}
```

### SSE 연결 서비스

```typescript
// services/SSEConnection.ts
import { EventEmitter } from 'eventemitter3';

interface ChannelHandlers {
  [channel: string]: Set<(data: any) => void>;
}

interface SSEConnectionOptions {
  reconnect?: boolean;
  reconnectDelay?: number;
  maxReconnectAttempts?: number;
  heartbeatInterval?: number;
  headers?: Record<string, string>;
  onStateChange?: (state: any) => void;
}

export class SSEConnection extends EventEmitter {
  private endpoint: string;
  private options: SSEConnectionOptions;
  private eventSource: EventSource | null = null;
  private channelHandlers: ChannelHandlers = {};
  private reconnectAttempts = 0;
  private heartbeatTimer: NodeJS.Timeout | null = null;
  private lastEventTime = Date.now();

  constructor(endpoint: string, options: SSEConnectionOptions = {}) {
    super();
    this.endpoint = endpoint;
    this.options = {
      reconnect: true,
      reconnectDelay: 1000,
      maxReconnectAttempts: 5,
      heartbeatInterval: 30000,
      ...options
    };
  }

  connect(): void {
    if (this.eventSource) {
      return;
    }

    this.updateState({ status: 'connecting' });

    try {
      const url = new URL(this.endpoint);
      
      // 헤더를 쿼리 파라미터로 전달 (EventSource 제약)
      if (this.options.headers) {
        Object.entries(this.options.headers).forEach(([key, value]) => {
          url.searchParams.append(`_header_${key}`, value);
        });
      }

      this.eventSource = new EventSource(url.toString());
      this.setupEventHandlers();
      this.startHeartbeat();
    } catch (error) {
      this.updateState({ 
        status: 'error', 
        error: new Error(`Connection failed: ${error.message}`) 
      });
    }
  }

  private setupEventHandlers(): void {
    if (!this.eventSource) return;

    this.eventSource.onopen = () => {
      this.reconnectAttempts = 0;
      this.updateState({ status: 'connected' });
      this.emit('connected');
    };

    this.eventSource.onerror = () => {
      this.updateState({ status: 'error' });
      this.handleReconnect();
    };

    this.eventSource.onmessage = (event) => {
      this.lastEventTime = Date.now();
      
      try {
        const { channel, data } = JSON.parse(event.data);
        this.dispatchToChannel(channel || 'default', data);
      } catch (error) {
        console.error('Failed to parse SSE message:', error);
      }
    };

    // 커스텀 이벤트 타입 처리
    ['notification', 'update', 'delete'].forEach(eventType => {
      this.eventSource!.addEventListener(eventType, (event) => {
        this.lastEventTime = Date.now();
        
        try {
          const data = JSON.parse(event.data);
          this.dispatchToChannel(eventType, data);
        } catch (error) {
          console.error(`Failed to parse ${eventType} event:`, error);
        }
      });
    });
  }

  private dispatchToChannel(channel: string, data: any): void {
    const handlers = this.channelHandlers[channel];
    if (handlers) {
      handlers.forEach(handler => {
        try {
          handler(data);
        } catch (error) {
          console.error(`Handler error in channel ${channel}:`, error);
        }
      });
    }

    // 글로벌 이벤트 발생
    this.emit('message', { channel, data });
  }

  subscribe(channel: string, handler: (data: any) => void): () => void {
    if (!this.channelHandlers[channel]) {
      this.channelHandlers[channel] = new Set();
    }

    this.channelHandlers[channel].add(handler);

    // 언서브스크라이브 함수 반환
    return () => {
      const handlers = this.channelHandlers[channel];
      if (handlers) {
        handlers.delete(handler);
        if (handlers.size === 0) {
          delete this.channelHandlers[channel];
        }
      }
    };
  }

  async publish(channel: string, data: any): Promise<void> {
    const response = await fetch(`${this.endpoint}/publish`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...this.options.headers
      },
      body: JSON.stringify({ channel, data })
    });

    if (!response.ok) {
      throw new Error(`Publish failed: ${response.statusText}`);
    }
  }

  private startHeartbeat(): void {
    if (!this.options.heartbeatInterval) return;

    this.heartbeatTimer = setInterval(() => {
      const timeSinceLastEvent = Date.now() - this.lastEventTime;
      
      if (timeSinceLastEvent > this.options.heartbeatInterval! * 2) {
        console.warn('No events received, connection might be stale');
        this.reconnect();
      }
    }, this.options.heartbeatInterval);
  }

  private stopHeartbeat(): void {
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
  }

  private handleReconnect(): void {
    if (!this.options.reconnect) return;

    if (this.reconnectAttempts >= this.options.maxReconnectAttempts!) {
      this.updateState({ 
        status: 'disconnected',
        error: new Error('Max reconnection attempts reached')
      });
      return;
    }

    const delay = this.options.reconnectDelay! * Math.pow(2, this.reconnectAttempts);
    
    setTimeout(() => {
      this.reconnectAttempts++;
      this.updateState({ 
        status: 'connecting',
        retries: this.reconnectAttempts 
      });
      this.disconnect();
      this.connect();
    }, delay);
  }

  reconnect(): void {
    this.reconnectAttempts = 0;
    this.disconnect();
    this.connect();
  }

  disconnect(): void {
    this.stopHeartbeat();
    
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
    }

    this.updateState({ status: 'disconnected' });
    this.emit('disconnected');
  }

  private updateState(state: any): void {
    if (this.options.onStateChange) {
      this.options.onStateChange({
        ...state,
        retries: this.reconnectAttempts
      });
    }
  }
}
```

## 재사용 가능한 UI 컴포넌트 라이브러리

### 스트리밍 인디케이터

```tsx
// components/ui/StreamingIndicator.tsx
import React from 'react';
import { motion, AnimatePresence } from 'framer-motion';

interface StreamingIndicatorProps {
  isStreaming: boolean;
  variant?: 'dots' | 'pulse' | 'typing' | 'wave';
  size?: 'sm' | 'md' | 'lg';
  color?: string;
}

export const StreamingIndicator: React.FC<StreamingIndicatorProps> = ({
  isStreaming,
  variant = 'dots',
  size = 'md',
  color = 'currentColor'
}) => {
  const sizeMap = {
    sm: { dot: 4, spacing: 6 },
    md: { dot: 6, spacing: 8 },
    lg: { dot: 8, spacing: 10 }
  };

  const { dot, spacing } = sizeMap[size];

  const renderIndicator = () => {
    switch (variant) {
      case 'dots':
        return (
          <div className="flex items-center" style={{ gap: `${spacing}px` }}>
            {[0, 1, 2].map((i) => (
              <motion.div
                key={i}
                style={{
                  width: dot,
                  height: dot,
                  backgroundColor: color,
                  borderRadius: '50%'
                }}
                animate={{
                  y: [0, -dot, 0],
                  opacity: [0.5, 1, 0.5]
                }}
                transition={{
                  duration: 0.6,
                  repeat: Infinity,
                  delay: i * 0.1
                }}
              />
            ))}
          </div>
        );

      case 'pulse':
        return (
          <motion.div
            style={{
              width: dot * 3,
              height: dot * 3,
              backgroundColor: color,
              borderRadius: '50%'
            }}
            animate={{
              scale: [1, 1.2, 1],
              opacity: [0.6, 0.3, 0.6]
            }}
            transition={{
              duration: 1.5,
              repeat: Infinity
            }}
          />
        );

      case 'typing':
        return (
          <div className="flex items-end" style={{ gap: `${spacing / 2}px` }}>
            {[0, 1, 2].map((i) => (
              <motion.div
                key={i}
                style={{
                  width: dot / 2,
                  backgroundColor: color
                }}
                animate={{
                  height: [dot / 2, dot * 2, dot / 2]
                }}
                transition={{
                  duration: 0.8,
                  repeat: Infinity,
                  delay: i * 0.15
                }}
              />
            ))}
          </div>
        );

      case 'wave':
        return (
          <svg
            width={dot * 8}
            height={dot * 3}
            viewBox={`0 0 ${dot * 8} ${dot * 3}`}
          >
            <motion.path
              d={`M0,${dot * 1.5} Q${dot * 2},${dot * 0.5} ${dot * 4},${dot * 1.5} T${dot * 8},${dot * 1.5}`}
              fill="none"
              stroke={color}
              strokeWidth={2}
              animate={{
                d: [
                  `M0,${dot * 1.5} Q${dot * 2},${dot * 0.5} ${dot * 4},${dot * 1.5} T${dot * 8},${dot * 1.5}`,
                  `M0,${dot * 1.5} Q${dot * 2},${dot * 2.5} ${dot * 4},${dot * 1.5} T${dot * 8},${dot * 1.5}`,
                  `M0,${dot * 1.5} Q${dot * 2},${dot * 0.5} ${dot * 4},${dot * 1.5} T${dot * 8},${dot * 1.5}`
                ]
              }}
              transition={{
                duration: 1,
                repeat: Infinity
              }}
            />
          </svg>
        );

      default:
        return null;
    }
  };

  return (
    <AnimatePresence>
      {isStreaming && (
        <motion.div
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          exit={{ opacity: 0 }}
          transition={{ duration: 0.2 }}
          className="streaming-indicator"
        >
          {renderIndicator()}
        </motion.div>
      )}
    </AnimatePresence>
  );
};
```

### 메시지 컴포넌트

```tsx
// components/ui/Message.tsx
import React, { memo } from 'react';
import { motion } from 'framer-motion';
import { StreamingIndicator } from './StreamingIndicator';
import { Markdown } from './Markdown';
import { CodeBlock } from './CodeBlock';
import { formatTimestamp } from '../../utils/formatters';

interface MessageProps {
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: number;
  status: 'pending' | 'streaming' | 'completed' | 'error';
  metadata?: {
    model?: string;
    tokens?: number;
    latency?: number;
  };
  onRetry?: () => void;
}

export const Message = memo<MessageProps>(({
  role,
  content,
  timestamp,
  status,
  metadata,
  onRetry
}) => {
  const isAssistant = role === 'assistant';
  const isStreaming = status === 'streaming';
  const isError = status === 'error';

  return (
    <motion.div
      initial={{ opacity: 0, y: 10 }}
      animate={{ opacity: 1, y: 0 }}
      className={`message message-${role}`}
    >
      <div className="message-header">
        <span className="message-role">
          {role === 'user' ? 'You' : role === 'assistant' ? 'AI' : 'System'}
        </span>
        <span className="message-time">
          {formatTimestamp(timestamp)}
        </span>
      </div>

      <div className="message-content">
        {isAssistant && isStreaming && content === '' ? (
          <StreamingIndicator isStreaming={true} variant="dots" />
        ) : (
          <Markdown 
            content={content}
            components={{
              code: CodeBlock
            }}
          />
        )}
        
        {isAssistant && isStreaming && content !== '' && (
          <StreamingIndicator 
            isStreaming={true} 
            variant="pulse" 
            size="sm" 
          />
        )}
      </div>

      {metadata && (
        <div className="message-metadata">
          {metadata.model && (
            <span className="metadata-item">
              Model: {metadata.model}
            </span>
          )}
          {metadata.tokens && (
            <span className="metadata-item">
              Tokens: {metadata.tokens}
            </span>
          )}
          {metadata.latency && (
            <span className="metadata-item">
              Latency: {metadata.latency}ms
            </span>
          )}
        </div>
      )}

      {isError && onRetry && (
        <button 
          className="message-retry"
          onClick={onRetry}
        >
          Retry
        </button>
      )}
    </motion.div>
  );
});

Message.displayName = 'Message';
```

### 연결 상태 컴포넌트

```tsx
// components/ui/ConnectionStatus.tsx
import React from 'react';
import { motion, AnimatePresence } from 'framer-motion';

interface ConnectionStatusProps {
  status: 'connecting' | 'connected' | 'disconnected' | 'error';
  error?: Error;
  retries?: number;
  onReconnect?: () => void;
}

export const ConnectionStatus: React.FC<ConnectionStatusProps> = ({
  status,
  error,
  retries = 0,
  onReconnect
}) => {
  const getStatusConfig = () => {
    switch (status) {
      case 'connecting':
        return {
          color: 'orange',
          text: retries > 0 ? `Reconnecting... (${retries})` : 'Connecting...',
          icon: '🔄'
        };
      case 'connected':
        return {
          color: 'green',
          text: 'Connected',
          icon: '✅'
        };
      case 'disconnected':
        return {
          color: 'gray',
          text: 'Disconnected',
          icon: '🔌'
        };
      case 'error':
        return {
          color: 'red',
          text: error?.message || 'Connection error',
          icon: '❌'
        };
    }
  };

  const config = getStatusConfig();

  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={status}
        initial={{ opacity: 0, x: -20 }}
        animate={{ opacity: 1, x: 0 }}
        exit={{ opacity: 0, x: 20 }}
        className="connection-status"
        style={{ 
          backgroundColor: `var(--color-${config.color}-light)`,
          color: `var(--color-${config.color}-dark)`
        }}
      >
        <span className="status-icon">{config.icon}</span>
        <span className="status-text">{config.text}</span>
        
        {(status === 'disconnected' || status === 'error') && onReconnect && (
          <button
            className="reconnect-button"
            onClick={onReconnect}
          >
            Reconnect
          </button>
        )}
      </motion.div>
    </AnimatePresence>
  );
};
```

## 상태 머신 기반 UI 관리

### SSE 상태 머신

```typescript
// machines/sseMachine.ts
import { createMachine, assign, interpret } from 'xstate';

interface SSEContext {
  endpoint: string;
  eventSource: EventSource | null;
  messages: any[];
  error: Error | null;
  retries: number;
  maxRetries: number;
}

type SSEEvent =
  | { type: 'CONNECT' }
  | { type: 'DISCONNECT' }
  | { type: 'RETRY' }
  | { type: 'MESSAGE'; data: any }
  | { type: 'ERROR'; error: Error }
  | { type: 'OPEN' }
  | { type: 'CLOSE' };

export const sseMachine = createMachine<SSEContext, SSEEvent>({
  id: 'sse',
  initial: 'disconnected',
  context: {
    endpoint: '',
    eventSource: null,
    messages: [],
    error: null,
    retries: 0,
    maxRetries: 5
  },
  states: {
    disconnected: {
      on: {
        CONNECT: {
          target: 'connecting',
          actions: 'resetError'
        }
      }
    },
    connecting: {
      invoke: {
        src: 'createEventSource',
        onDone: {
          target: 'connected',
          actions: assign({
            eventSource: (_, event) => event.data,
            retries: 0
          })
        },
        onError: {
          target: 'error',
          actions: assign({
            error: (_, event) => event.data
          })
        }
      },
      on: {
        OPEN: 'connected',
        ERROR: {
          target: 'error',
          actions: assign({
            error: (_, event) => event.error
          })
        }
      }
    },
    connected: {
      on: {
        MESSAGE: {
          actions: assign({
            messages: (context, event) => [...context.messages, event.data]
          })
        },
        ERROR: 'reconnecting',
        DISCONNECT: {
          target: 'disconnected',
          actions: 'cleanup'
        },
        CLOSE: 'reconnecting'
      }
    },
    reconnecting: {
      entry: assign({
        retries: (context) => context.retries + 1
      }),
      always: [
        {
          target: 'error',
          cond: (context) => context.retries >= context.maxRetries
        }
      ],
      after: {
        RECONNECT_DELAY: {
          target: 'connecting',
          actions: 'cleanup'
        }
      },
      on: {
        DISCONNECT: {
          target: 'disconnected',
          actions: 'cleanup'
        }
      }
    },
    error: {
      on: {
        RETRY: {
          target: 'connecting',
          actions: assign({
            retries: 0,
            error: null
          })
        },
        DISCONNECT: {
          target: 'disconnected',
          actions: 'cleanup'
        }
      }
    }
  }
}, {
  actions: {
    resetError: assign({
      error: null
    }),
    cleanup: assign({
      eventSource: (context) => {
        context.eventSource?.close();
        return null;
      }
    })
  },
  services: {
    createEventSource: (context) => {
      return new Promise((resolve, reject) => {
        try {
          const eventSource = new EventSource(context.endpoint);
          
          eventSource.onopen = () => resolve(eventSource);
          eventSource.onerror = () => reject(new Error('Connection failed'));
          
        } catch (error) {
          reject(error);
        }
      });
    }
  },
  delays: {
    RECONNECT_DELAY: (context) => {
      return Math.min(1000 * Math.pow(2, context.retries), 30000);
    }
  }
});
```

### 상태 머신 React 통합

```tsx
// hooks/useSSEMachine.ts
import { useMachine } from '@xstate/react';
import { useEffect, useCallback } from 'react';
import { sseMachine } from '../machines/sseMachine';

export interface UseSSEMachineOptions {
  endpoint: string;
  onMessage?: (data: any) => void;
  onError?: (error: Error) => void;
  maxRetries?: number;
}

export function useSSEMachine({
  endpoint,
  onMessage,
  onError,
  maxRetries = 5
}: UseSSEMachineOptions) {
  const [state, send, service] = useMachine(sseMachine, {
    context: {
      endpoint,
      eventSource: null,
      messages: [],
      error: null,
      retries: 0,
      maxRetries
    }
  });

  // EventSource 이벤트 핸들러 설정
  useEffect(() => {
    const subscription = service.subscribe((state) => {
      const { eventSource } = state.context;
      
      if (eventSource && state.matches('connected')) {
        eventSource.onmessage = (event) => {
          const data = JSON.parse(event.data);
          send({ type: 'MESSAGE', data });
          onMessage?.(data);
        };

        eventSource.onerror = () => {
          send({ type: 'ERROR', error: new Error('Connection error') });
        };

        eventSource.addEventListener('close', () => {
          send({ type: 'CLOSE' });
        });
      }
    });

    return () => {
      subscription.unsubscribe();
    };
  }, [service, send, onMessage]);

  // 에러 핸들링
  useEffect(() => {
    if (state.context.error && onError) {
      onError(state.context.error);
    }
  }, [state.context.error, onError]);

  const connect = useCallback(() => {
    send({ type: 'CONNECT' });
  }, [send]);

  const disconnect = useCallback(() => {
    send({ type: 'DISCONNECT' });
  }, [send]);

  const retry = useCallback(() => {
    send({ type: 'RETRY' });
  }, [send]);

  return {
    state: state.value,
    context: state.context,
    isConnected: state.matches('connected'),
    isConnecting: state.matches('connecting'),
    isReconnecting: state.matches('reconnecting'),
    hasError: state.matches('error'),
    messages: state.context.messages,
    error: state.context.error,
    actions: {
      connect,
      disconnect,
      retry
    }
  };
}
```

### UI 컴포넌트에서 상태 머신 사용

```tsx
// components/SSEMachineDemo.tsx
import React from 'react';
import { useSSEMachine } from '../hooks/useSSEMachine';
import { ConnectionStatus } from './ui/ConnectionStatus';
import { MessageList } from './ui/MessageList';

export const SSEMachineDemo: React.FC = () => {
  const {
    state,
    context,
    isConnected,
    isConnecting,
    isReconnecting,
    hasError,
    messages,
    error,
    actions
  } = useSSEMachine({
    endpoint: 'https://api.example.com/events',
    onMessage: (data) => {
      console.log('Received message:', data);
    },
    onError: (error) => {
      console.error('SSE error:', error);
    }
  });

  const getStatusFromState = () => {
    if (isConnecting || isReconnecting) return 'connecting';
    if (isConnected) return 'connected';
    if (hasError) return 'error';
    return 'disconnected';
  };

  return (
    <div className="sse-machine-demo">
      <div className="demo-header">
        <h2>SSE State Machine Demo</h2>
        <ConnectionStatus
          status={getStatusFromState()}
          error={error}
          retries={context.retries}
          onReconnect={hasError ? actions.retry : actions.connect}
        />
      </div>

      <div className="demo-controls">
        <button
          onClick={actions.connect}
          disabled={isConnected || isConnecting}
        >
          Connect
        </button>
        <button
          onClick={actions.disconnect}
          disabled={!isConnected}
        >
          Disconnect
        </button>
        <button
          onClick={actions.retry}
          disabled={!hasError}
        >
          Retry
        </button>
      </div>

      <div className="demo-state">
        <h3>Current State: {String(state)}</h3>
        <pre>{JSON.stringify(context, null, 2)}</pre>
      </div>

      <div className="demo-messages">
        <h3>Messages ({messages.length})</h3>
        <MessageList messages={messages} />
      </div>
    </div>
  );
};
```

## 마무리

이 장에서는 프로덕션 환경에서 바로 사용할 수 있는 완성도 높은 SSE 구현 예제들을 제공했습니다. 스트리밍 채팅 컴포넌트, 커스텀 프로바이더, 재사용 가능한 UI 컴포넌트 라이브러리, 그리고 상태 머신 기반 관리까지 실제 애플리케이션에서 필요한 모든 요소를 포함했습니다.

다음 장에서는 실제 기업들의 SSE 활용 사례와 부록을 다루겠습니다.

***
**[← 이전: Part 10 - 테스팅 가이드](./part10-testing-guide.md)** | **[목차](./README.md)** | **[다음: Part 12 - 실제 기업 사례 →](./part12-real-world-cases.md)**

***