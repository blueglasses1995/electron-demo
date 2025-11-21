# ネットワーク通信 (HTTP & WebSocket)

## 1. Electronのネットワークスタック

### 1.1 ネットワークアーキテクチャ

```
┌──────────────────────────────────────┐
│    Electron Application Layer       │
├──────────────────────────────────────┤
│  Node.js APIs     │  Chromium APIs   │
│  - https          │  - fetch()       │
│  - net            │  - XMLHttpRequest│
│  - http           │  - WebSocket     │
├──────────────────────────────────────┤
│     Chromium Network Stack           │
│  - HTTP/1.1, HTTP/2, HTTP/3 (QUIC)  │
│  - TLS 1.2, TLS 1.3                  │
│  - DNS Resolution                    │
│  - Cookie Management                 │
│  - Cache Control                     │
├──────────────────────────────────────┤
│          OS Network Layer            │
│  - TCP/UDP Sockets                   │
│  - System DNS                        │
└──────────────────────────────────────┘
```

## 2. HTTP通信

### 2.1 Mainプロセスでの通信

#### 方法1: Electron net モジュール（推奨）

```typescript
import { net } from 'electron';

// GETリクエスト
async function fetchData(url: string): Promise<any> {
  return new Promise((resolve, reject) => {
    const request = net.request({
      method: 'GET',
      url: url,
      // プロキシ、証明書の設定を継承
      session: session.defaultSession
    });

    let body = '';

    request.on('response', (response) => {
      console.log(`Status: ${response.statusCode}`);
      console.log(`Headers:`, response.headers);

      response.on('data', (chunk) => {
        body += chunk.toString();
      });

      response.on('end', () => {
        if (response.statusCode >= 200 && response.statusCode < 300) {
          resolve(JSON.parse(body));
        } else {
          reject(new Error(`HTTP ${response.statusCode}`));
        }
      });
    });

    request.on('error', (error) => {
      reject(error);
    });

    request.end();
  });
}

// POSTリクエスト
async function postData(url: string, data: any): Promise<any> {
  return new Promise((resolve, reject) => {
    const payload = JSON.stringify(data);

    const request = net.request({
      method: 'POST',
      url: url
    });

    request.setHeader('Content-Type', 'application/json');
    request.setHeader('Content-Length', Buffer.byteLength(payload).toString());

    let responseBody = '';

    request.on('response', (response) => {
      response.on('data', (chunk) => {
        responseBody += chunk.toString();
      });

      response.on('end', () => {
        resolve(JSON.parse(responseBody));
      });
    });

    request.on('error', reject);

    request.write(payload);
    request.end();
  });
}
```

#### 方法2: Node.js https モジュール

```typescript
import * as https from 'https';

async function fetchWithHttps(url: string): Promise<any> {
  return new Promise((resolve, reject) => {
    https.get(url, (res) => {
      let data = '';

      res.on('data', (chunk) => {
        data += chunk;
      });

      res.on('end', () => {
        resolve(JSON.parse(data));
      });
    }).on('error', reject);
  });
}
```

#### 方法3: サードパーティライブラリ（axios, node-fetch）

```typescript
import axios from 'axios';

// axiosの使用
async function fetchWithAxios(url: string): Promise<any> {
  const response = await axios.get(url, {
    headers: {
      'User-Agent': 'My Electron App/1.0'
    },
    timeout: 10000
  });

  return response.data;
}

// POSTリクエスト
async function postWithAxios(url: string, data: any): Promise<any> {
  const response = await axios.post(url, data, {
    headers: {
      'Content-Type': 'application/json'
    }
  });

  return response.data;
}
```

### 2.2 Rendererプロセスでの通信

```typescript
// renderer.ts

// 方法1: fetch API（モダンな方法）
async function fetchData(url: string): Promise<any> {
  const response = await fetch(url, {
    method: 'GET',
    headers: {
      'Content-Type': 'application/json'
    }
  });

  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

  return await response.json();
}

// POSTリクエスト
async function postData(url: string, data: any): Promise<any> {
  const response = await fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(data)
  });

  return await response.json();
}

// ファイルアップロード
async function uploadFile(url: string, file: File): Promise<any> {
  const formData = new FormData();
  formData.append('file', file);

  const response = await fetch(url, {
    method: 'POST',
    body: formData
  });

  return await response.json();
}
```

### 2.3 リクエストのインターセプト

```typescript
// main.ts
import { session } from 'electron';

session.defaultSession.webRequest.onBeforeSendHeaders((details, callback) => {
  // リクエストヘッダーを変更
  details.requestHeaders['User-Agent'] = 'My Electron App';
  details.requestHeaders['X-Custom-Header'] = 'CustomValue';

  callback({ requestHeaders: details.requestHeaders });
});

// レスポンスヘッダーの変更
session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
  const responseHeaders = details.responseHeaders || {};

  // CSPヘッダーを追加
  responseHeaders['Content-Security-Policy'] = [
    "default-src 'self'; script-src 'self'"
  ];

  callback({ responseHeaders });
});

// リクエストのブロック
session.defaultSession.webRequest.onBeforeRequest((details, callback) => {
  // 特定のURLをブロック
  if (details.url.includes('tracking.example.com')) {
    callback({ cancel: true });
  } else {
    callback({});
  }
});
```

## 3. WebSocket通信

### 3.1 基本的なWebSocket実装

```typescript
// renderer.ts
class WebSocketClient {
  private ws: WebSocket | null = null;
  private reconnectInterval: number = 5000;
  private reconnectTimer: NodeJS.Timeout | null = null;

  constructor(private url: string) {}

  connect(): void {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.onConnected();
    };

    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.onMessage(data);
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      this.onError(error);
    };

    this.ws.onclose = () => {
      console.log('WebSocket closed');
      this.onDisconnected();
      this.scheduleReconnect();
    };
  }

  send(data: any): void {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    } else {
      console.warn('WebSocket is not connected');
    }
  }

  close(): void {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
    }

    if (this.ws) {
      this.ws.close();
    }
  }

  private scheduleReconnect(): void {
    this.reconnectTimer = setTimeout(() => {
      console.log('Reconnecting...');
      this.connect();
    }, this.reconnectInterval);
  }

  // フック関数（継承してオーバーライド）
  protected onConnected(): void {}
  protected onMessage(data: any): void {}
  protected onError(error: Event): void {}
  protected onDisconnected(): void {}
}

// 使用例
class ChatWebSocket extends WebSocketClient {
  protected onConnected(): void {
    console.log('Chat connected');
    this.send({ type: 'auth', token: 'user-token' });
  }

  protected onMessage(data: any): void {
    console.log('Received:', data);

    if (data.type === 'message') {
      displayMessage(data.message);
    }
  }
}

const chat = new ChatWebSocket('wss://example.com/chat');
chat.connect();
```

### 3.2 MainプロセスでのWebSocket

```typescript
// main.ts
import WebSocket from 'ws';

class MainWebSocketClient {
  private ws: WebSocket | null = null;

  connect(url: string): void {
    this.ws = new WebSocket(url);

    this.ws.on('open', () => {
      console.log('Main WebSocket connected');
    });

    this.ws.on('message', (data) => {
      const message = JSON.parse(data.toString());
      console.log('Received:', message);

      // すべてのRendererに送信
      BrowserWindow.getAllWindows().forEach((win) => {
        win.webContents.send('websocket-message', message);
      });
    });

    this.ws.on('error', (error) => {
      console.error('Main WebSocket error:', error);
    });

    this.ws.on('close', () => {
      console.log('Main WebSocket closed');
    });
  }

  send(data: any): void {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    }
  }
}

const mainWs = new MainWebSocketClient();
mainWs.connect('wss://example.com/api');
```

### 3.3 WebSocketの状態管理

```typescript
enum ConnectionState {
  DISCONNECTED = 'disconnected',
  CONNECTING = 'connecting',
  CONNECTED = 'connected',
  RECONNECTING = 'reconnecting'
}

class ManagedWebSocket {
  private ws: WebSocket | null = null;
  private state: ConnectionState = ConnectionState.DISCONNECTED;
  private messageQueue: any[] = [];
  private heartbeatInterval: NodeJS.Timeout | null = null;

  constructor(private url: string) {}

  get connectionState(): ConnectionState {
    return this.state;
  }

  connect(): void {
    if (this.state === ConnectionState.CONNECTED ||
        this.state === ConnectionState.CONNECTING) {
      return;
    }

    this.state = ConnectionState.CONNECTING;
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      this.state = ConnectionState.CONNECTED;
      this.startHeartbeat();
      this.flushMessageQueue();
    };

    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.handleMessage(data);
    };

    this.ws.onclose = () => {
      this.state = ConnectionState.DISCONNECTED;
      this.stopHeartbeat();
      this.reconnect();
    };
  }

  send(data: any): void {
    if (this.state === ConnectionState.CONNECTED && this.ws) {
      this.ws.send(JSON.stringify(data));
    } else {
      // オフライン時はキューに保存
      this.messageQueue.push(data);
    }
  }

  private startHeartbeat(): void {
    this.heartbeatInterval = setInterval(() => {
      this.send({ type: 'ping' });
    }, 30000); // 30秒ごと
  }

  private stopHeartbeat(): void {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
    }
  }

  private flushMessageQueue(): void {
    while (this.messageQueue.length > 0) {
      const message = this.messageQueue.shift();
      this.send(message);
    }
  }

  private reconnect(): void {
    this.state = ConnectionState.RECONNECTING;
    setTimeout(() => this.connect(), 5000);
  }

  private handleMessage(data: any): void {
    if (data.type === 'pong') {
      // Heartbeat受信
      return;
    }

    // 通常のメッセージ処理
    console.log('Message:', data);
  }
}
```

## 4. ネットワーク状態の監視

### 4.1 オンライン/オフライン検出

```typescript
// renderer.ts
window.addEventListener('online', () => {
  console.log('Network is online');
  updateUIState('online');
  reconnectServices();
});

window.addEventListener('offline', () => {
  console.log('Network is offline');
  updateUIState('offline');
  pauseServices();
});

// 初期状態の確認
if (navigator.onLine) {
  console.log('Currently online');
} else {
  console.log('Currently offline');
}

// main.ts
import { app } from 'electron';

app.on('ready', () => {
  // ネットワーク状態の変更を監視
  app.on('network-changed', () => {
    console.log('Network changed');
  });
});
```

### 4.2 ネットワーク品質の測定

```typescript
class NetworkMonitor {
  private latency: number = 0;

  async measureLatency(url: string): Promise<number> {
    const start = performance.now();

    try {
      await fetch(url, { method: 'HEAD' });
      const end = performance.now();
      this.latency = end - start;
      return this.latency;
    } catch (error) {
      console.error('Latency measurement failed:', error);
      return -1;
    }
  }

  async measureBandwidth(url: string, sizeBytes: number): Promise<number> {
    const start = performance.now();

    try {
      const response = await fetch(url);
      await response.arrayBuffer();
      const end = performance.now();

      const durationSeconds = (end - start) / 1000;
      const mbps = (sizeBytes * 8) / (durationSeconds * 1000000);

      return mbps;
    } catch (error) {
      console.error('Bandwidth measurement failed:', error);
      return -1;
    }
  }

  getConnectionType(): string {
    const connection = (navigator as any).connection;

    if (connection) {
      return connection.effectiveType; // '4g', '3g', '2g', 'slow-2g'
    }

    return 'unknown';
  }
}
```

## 5. セキュアな通信

### 5.1 HTTPS/TLS設定

```typescript
// main.ts
import { app, session } from 'electron';

app.on('ready', () => {
  // 証明書検証のカスタマイズ
  session.defaultSession.setCertificateVerifyProc((request, callback) => {
    const { hostname, certificate, verificationResult } = request;

    // 本番環境では必ず証明書を検証
    if (process.env.NODE_ENV === 'production') {
      callback(verificationResult === 'net::OK' ? 0 : -2);
    } else {
      // 開発環境でのみlocalhostを許可
      if (hostname === 'localhost' || hostname === '127.0.0.1') {
        callback(0);
      } else {
        callback(verificationResult === 'net::OK' ? 0 : -2);
      }
    }
  });
});

// 証明書エラーのハンドリング
app.on('certificate-error', (event, webContents, url, error, certificate, callback) => {
  console.error('Certificate error:', error);

  // 開発環境でのみ自己署名証明書を許可
  if (process.env.NODE_ENV === 'development' && url.startsWith('https://localhost')) {
    event.preventDefault();
    callback(true);
  } else {
    callback(false);
  }
});
```

### 5.2 認証とトークン管理

```typescript
class AuthService {
  private accessToken: string | null = null;
  private refreshToken: string | null = null;

  async login(username: string, password: string): Promise<void> {
    const response = await fetch('https://api.example.com/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password })
    });

    const data = await response.json();
    this.accessToken = data.accessToken;
    this.refreshToken = data.refreshToken;

    // セキュアに保存（keytar等を使用）
    await this.saveTokens();
  }

  async request(url: string, options: RequestInit = {}): Promise<any> {
    const headers = {
      ...options.headers,
      'Authorization': `Bearer ${this.accessToken}`
    };

    let response = await fetch(url, { ...options, headers });

    // トークンが期限切れの場合
    if (response.status === 401) {
      await this.refreshAccessToken();

      // リトライ
      headers['Authorization'] = `Bearer ${this.accessToken}`;
      response = await fetch(url, { ...options, headers });
    }

    return await response.json();
  }

  private async refreshAccessToken(): Promise<void> {
    const response = await fetch('https://api.example.com/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken: this.refreshToken })
    });

    const data = await response.json();
    this.accessToken = data.accessToken;
    await this.saveTokens();
  }

  private async saveTokens(): Promise<void> {
    // keytar等を使用してセキュアに保存
    // await keytar.setPassword('my-app', 'accessToken', this.accessToken);
  }
}
```

## 6. まとめ

- **Mainプロセス**: `electron.net` または `https` モジュールを使用
- **Rendererプロセス**: `fetch()` API を使用（モダン）
- **WebSocket**: リアルタイム通信に使用、再接続ロジックを実装
- **ネットワーク監視**: オンライン/オフライン検出、品質測定
- **セキュリティ**: HTTPS/TLS、証明書検証、トークン管理

次のドキュメントでは、認証認可とアカウント管理について詳しく解説します。
