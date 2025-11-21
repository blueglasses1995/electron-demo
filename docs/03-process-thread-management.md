# プロセス・スレッド管理の詳細

## 1. Electronのマルチプロセスアーキテクチャ

### 1.1 プロセスモデル概要

```
┌────────────────────────────────────────────────┐
│             OS Process Layer                  │
├────────────────────────────────────────────────┤
│  Main Process (PID: 1000)                     │
│  └─ Node.js + Electron Main API               │
│                                                │
│  GPU Process (PID: 1001)                      │
│  └─ Hardware Acceleration, WebGL              │
│                                                │
│  Utility Process (PID: 1002)                  │
│  └─ Network Service, Storage Service          │
│                                                │
│  Renderer Process 1 (PID: 1003)               │
│  └─ Chromium + Blink + V8 (Window 1)          │
│                                                │
│  Renderer Process 2 (PID: 1004)               │
│  └─ Chromium + Blink + V8 (Window 2)          │
└────────────────────────────────────────────────┘
```

### 1.2 プロセスの種類と役割

| プロセスタイプ | 数 | 役割 | メモリ使用量目安 |
|--------------|---|------|----------------|
| **Browser (Main)** | 1 | アプリケーション管理、ウィンドウ制御 | 50-100MB |
| **Renderer** | N | UI表示、JavaScript実行 | 50-200MB/プロセス |
| **GPU** | 1 | グラフィックス処理 | 50-500MB |
| **Utility** | 1+ | ネットワーク、ストレージ等のサービス | 20-50MB/プロセス |

## 2. Mainプロセスの詳細

### 2.1 Mainプロセスのライフサイクル

```typescript
// main.ts
import { app, BrowserWindow } from 'electron';

console.log('Main process PID:', process.pid);

// アプリケーションの初期化
app.whenReady().then(() => {
  console.log('App is ready');
  createWindow();
});

// すべてのウィンドウが閉じられた
app.on('window-all-closed', () => {
  console.log('All windows closed');
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

// アプリケーション終了前
app.on('before-quit', (event) => {
  console.log('Before quit');
  // 非同期処理がある場合はevent.preventDefault()で待つ
});

// アプリケーション終了時
app.on('will-quit', () => {
  console.log('Will quit');
});

// アプリケーション終了完了
app.on('quit', (event, exitCode) => {
  console.log('Quit with exit code:', exitCode);
});
```

### 2.2 Mainプロセスのスレッド構成

Mainプロセスは複数のスレッドで構成:

```typescript
import { app } from 'electron';
import * as os from 'os';

app.whenReady().then(() => {
  // Node.jsのイベントループ情報
  console.log('CPUs:', os.cpus().length);
  console.log('Platform:', process.platform);

  // タイマーとイベントループ
  setImmediate(() => {
    console.log('Immediate callback (next tick)');
  });

  process.nextTick(() => {
    console.log('Next tick callback (microtask)');
  });

  setTimeout(() => {
    console.log('Timeout callback (macrotask)');
  }, 0);
});
```

**Mainプロセスのスレッド**:
1. **Main Thread**: JavaScriptコードの実行
2. **libuv Worker Thread Pool**: ファイルI/O、DNS解決（デフォルト4スレッド）
3. **V8 Background Threads**: ガベージコレクション、最適化

### 2.3 Mainプロセスでの重い処理

```typescript
import { utilityProcess } from 'electron';

// 方法1: Utility Processで処理
function runHeavyTaskInUtilityProcess() {
  const child = utilityProcess.fork(
    path.join(__dirname, 'heavy-task.js'),
    ['arg1', 'arg2'],
    {
      serviceName: 'Heavy Task Worker',
      stdio: 'pipe'
    }
  );

  child.on('message', (message) => {
    console.log('Result from utility process:', message);
  });

  child.postMessage({ data: 'some data' });
}

// 方法2: Worker Threadsで処理（Node.js機能）
import { Worker } from 'worker_threads';

function runHeavyTaskInWorkerThread() {
  const worker = new Worker('./worker.js', {
    workerData: { input: 'data' }
  });

  worker.on('message', (result) => {
    console.log('Result from worker:', result);
  });

  worker.on('error', (error) => {
    console.error('Worker error:', error);
  });

  worker.on('exit', (code) => {
    console.log('Worker exited with code:', code);
  });
}
```

## 3. Rendererプロセスの詳細

### 3.1 Rendererプロセスの作成

```typescript
import { BrowserWindow } from 'electron';

function createWindow() {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      // プロセス分離の設定
      nodeIntegration: false,
      contextIsolation: true,
      sandbox: true,

      // プロセス再利用の設定
      additionalArguments: ['--process-per-site'],

      preload: path.join(__dirname, 'preload.js')
    }
  });

  // Rendererプロセスの情報取得
  win.webContents.on('did-finish-load', () => {
    console.log('Renderer Process ID:', win.webContents.getOSProcessId());
    console.log('Renderer Type:', win.webContents.getType()); // 'window'
  });

  win.loadFile('index.html');
}
```

### 3.2 Rendererプロセスのスレッド構成

```
Renderer Process
├── Main Thread (Blink Main Thread)
│   ├── JavaScript実行 (V8)
│   ├── DOMツリー構築
│   ├── CSSスタイル計算
│   ├── Layout計算
│   └── Paint処理
│
├── Compositor Thread
│   ├── Layer合成
│   ├── スクロール処理
│   └── アニメーション
│
├── Raster Threads (複数)
│   └── ビットマップ生成
│
└── Web Worker Threads
    └── バックグラウンドJavaScript実行
```

### 3.3 Web Workersの活用

```typescript
// renderer.ts - メインスレッド
const worker = new Worker('worker.js');

// 重い計算をWorkerに委譲
worker.postMessage({
  type: 'calculate',
  data: largeDataSet
});

worker.onmessage = (event) => {
  const result = event.data;
  console.log('Calculation result:', result);
  updateUI(result);
};

worker.onerror = (error) => {
  console.error('Worker error:', error);
};

// worker.js - Workerスレッド
self.onmessage = (event) => {
  const { type, data } = event.data;

  if (type === 'calculate') {
    // CPU集約的な処理
    const result = heavyCalculation(data);
    self.postMessage(result);
  }
};

function heavyCalculation(data: any[]): number {
  let sum = 0;
  for (let i = 0; i < 1000000; i++) {
    sum += data[i % data.length];
  }
  return sum;
}
```

### 3.4 Shared Workers（複数Rendererで共有）

```typescript
// renderer1.ts
const sharedWorker = new SharedWorker('shared-worker.js');

sharedWorker.port.onmessage = (event) => {
  console.log('Message from shared worker:', event.data);
};

sharedWorker.port.postMessage('Hello from renderer 1');

// shared-worker.js
const connections: MessagePort[] = [];

onconnect = (event) => {
  const port = event.ports[0];
  connections.push(port);

  port.onmessage = (e) => {
    console.log('Received:', e.data);

    // すべての接続にブロードキャスト
    connections.forEach((conn) => {
      conn.postMessage(`Broadcast: ${e.data}`);
    });
  };

  port.start();
};
```

## 4. プロセス間通信 (IPC)

### 4.1 IPCの種類

| IPC方式 | 方向 | 用途 |
|---------|------|------|
| **ipcRenderer.invoke** | Renderer → Main | 非同期リクエスト/レスポンス |
| **ipcRenderer.send** | Renderer → Main | 一方向メッセージ |
| **ipcMain.handle** | Main受信 | invokeのハンドラ |
| **ipcMain.on** | Main受信 | sendのハンドラ |
| **webContents.send** | Main → Renderer | 一方向メッセージ |
| **MessagePort** | 双方向 | 構造化複製による通信 |

### 4.2 IPC実装パターン

#### パターン1: Request/Response（推奨）

```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  // 非同期リクエスト
  getData: (id: string) => ipcRenderer.invoke('get-data', id),

  // 同期的に見えるがPromiseを返す
  saveData: (data: any) => ipcRenderer.invoke('save-data', data)
});

// main.ts
import { ipcMain } from 'electron';

ipcMain.handle('get-data', async (event, id: string) => {
  console.log('Request from renderer:', event.processId);

  // データベースやファイルから取得
  const data = await fetchDataFromDB(id);
  return data;
});

ipcMain.handle('save-data', async (event, data: any) => {
  await saveDataToDB(data);
  return { success: true };
});

// renderer.ts
const data = await window.electronAPI.getData('user-123');
console.log(data);

await window.electronAPI.saveData({ name: 'John' });
```

#### パターン2: イベント駆動（一方向）

```typescript
// preload.ts
contextBridge.exposeInMainWorld('electronAPI', {
  // イベント送信
  notify: (message: string) => ipcRenderer.send('notification', message),

  // イベント受信
  onUpdate: (callback: (data: any) => void) => {
    ipcRenderer.on('update', (event, data) => callback(data));
  }
});

// main.ts
ipcMain.on('notification', (event, message) => {
  console.log('Notification:', message);

  // 通知を表示
  new Notification({ title: 'Notification', body: message }).show();
});

// すべてのRendererに更新を送信
function broadcastUpdate(data: any) {
  BrowserWindow.getAllWindows().forEach((win) => {
    win.webContents.send('update', data);
  });
}

// renderer.ts
window.electronAPI.notify('Something happened');

window.electronAPI.onUpdate((data) => {
  console.log('Update received:', data);
  updateUI(data);
});
```

#### パターン3: MessagePort（双方向チャネル）

```typescript
// main.ts
import { BrowserWindow, MessageChannelMain } from 'electron';

const win1 = new BrowserWindow({});
const win2 = new BrowserWindow({});

const { port1, port2 } = new MessageChannelMain();

// port1をwin1に、port2をwin2に送信
win1.webContents.postMessage('port', null, [port1]);
win2.webContents.postMessage('port', null, [port2]);

// preload.ts
import { ipcRenderer } from 'electron';

ipcRenderer.on('port', (event) => {
  const [port] = event.ports;

  port.onmessage = (event) => {
    console.log('Message from other window:', event.data);
  };

  port.postMessage('Hello from this window');
  port.start();
});
```

### 4.3 IPCのパフォーマンス最適化

```typescript
// 悪い例: 大量の小さなメッセージ
for (let i = 0; i < 1000; i++) {
  ipcRenderer.invoke('process-item', items[i]);
}

// 良い例: バッチ処理
ipcRenderer.invoke('process-items', items);

// main.ts
ipcMain.handle('process-items', async (event, items: any[]) => {
  const results = await Promise.all(
    items.map(item => processItem(item))
  );
  return results;
});

// さらに良い例: Streaming（大量データの場合）
ipcMain.handle('process-items-stream', async (event, items: any[]) => {
  const sender = event.sender;

  for (let i = 0; i < items.length; i++) {
    const result = await processItem(items[i]);

    // 進捗を送信
    sender.send('progress', {
      current: i + 1,
      total: items.length,
      result
    });
  }

  return { success: true };
});
```

## 5. プロセスのライフサイクル管理

### 5.1 Rendererプロセスのクラッシュハンドリング

```typescript
// main.ts
const win = new BrowserWindow({});

// Rendererがクラッシュした
win.webContents.on('render-process-gone', (event, details) => {
  console.error('Renderer crashed:', details.reason);
  // reason: 'clean-exit', 'abnormal-exit', 'killed', 'crashed', etc.

  if (details.reason !== 'clean-exit') {
    // ウィンドウを再作成
    win.reload();
  }
});

// Rendererが応答しない
win.webContents.on('unresponsive', () => {
  console.warn('Renderer is unresponsive');

  dialog.showMessageBox(win, {
    type: 'warning',
    title: 'アプリが応答していません',
    message: '待機しますか、それとも終了しますか?',
    buttons: ['待機', '終了']
  }).then((result) => {
    if (result.response === 1) {
      win.close();
    }
  });
});

// Rendererが応答を再開
win.webContents.on('responsive', () => {
  console.log('Renderer is responsive again');
});
```

### 5.2 プロセスメモリの監視

```typescript
import { app } from 'electron';

// 定期的にメモリ使用量を監視
setInterval(() => {
  const metrics = app.getAppMetrics();

  metrics.forEach((metric) => {
    const memoryMB = metric.memory.workingSetSize / 1024 / 1024;

    console.log(`Process ${metric.pid} (${metric.type}):`);
    console.log(`  Memory: ${memoryMB.toFixed(2)} MB`);
    console.log(`  CPU: ${metric.cpu.percentCPUUsage.toFixed(2)}%`);

    // メモリ使用量が多い場合は警告
    if (memoryMB > 500) {
      console.warn(`Process ${metric.pid} is using too much memory!`);

      // Rendererプロセスの場合は再読み込みを検討
      if (metric.type === 'Renderer') {
        // const win = BrowserWindow.fromWebContents(webContents);
        // win?.reload();
      }
    }
  });
}, 30000); // 30秒ごと
```

### 5.3 プロセスのグレースフルシャットダウン

```typescript
// main.ts
let isQuitting = false;

app.on('before-quit', async (event) => {
  if (!isQuitting) {
    event.preventDefault();
    isQuitting = true;

    console.log('Graceful shutdown starting...');

    // すべてのRendererにクリーンアップを通知
    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.send('app-closing');
    });

    // クリーンアップ処理を待つ
    await Promise.all([
      closeDatabase(),
      saveApplicationState(),
      stopBackgroundTasks()
    ]);

    console.log('Cleanup complete, quitting...');
    app.quit();
  }
});

// renderer.ts
window.electronAPI.onAppClosing(async () => {
  console.log('App is closing, cleaning up...');

  // 保存されていないデータを保存
  await saveUnsavedData();

  // リソースの解放
  cleanupResources();
});
```

## 6. スレッドの活用

### 6.1 Node.js Worker Threads（Mainプロセス）

```typescript
// main.ts
import { Worker } from 'worker_threads';
import * as path from 'path';

class WorkerPool {
  private workers: Worker[] = [];
  private availableWorkers: Worker[] = [];
  private taskQueue: Array<{ data: any; resolve: (value: any) => void }> = [];

  constructor(size: number, workerScript: string) {
    for (let i = 0; i < size; i++) {
      const worker = new Worker(workerScript);
      this.workers.push(worker);
      this.availableWorkers.push(worker);

      worker.on('message', (result) => {
        // Workerを再利用可能にする
        this.availableWorkers.push(worker);

        // 次のタスクを処理
        this.processNextTask();
      });
    }
  }

  async execute(data: any): Promise<any> {
    return new Promise((resolve) => {
      this.taskQueue.push({ data, resolve });
      this.processNextTask();
    });
  }

  private processNextTask() {
    if (this.taskQueue.length === 0 || this.availableWorkers.length === 0) {
      return;
    }

    const task = this.taskQueue.shift()!;
    const worker = this.availableWorkers.shift()!;

    worker.once('message', (result) => {
      task.resolve(result);
    });

    worker.postMessage(task.data);
  }

  async terminate() {
    await Promise.all(this.workers.map(w => w.terminate()));
  }
}

// 使用例
const pool = new WorkerPool(4, path.join(__dirname, 'cpu-worker.js'));

ipcMain.handle('heavy-task', async (event, data) => {
  const result = await pool.execute(data);
  return result;
});
```

### 6.2 Web Workers（Rendererプロセス）

```typescript
// renderer.ts
class WorkerManager {
  private worker: Worker;

  constructor(workerPath: string) {
    this.worker = new Worker(workerPath);
  }

  async execute<T>(task: string, data: any): Promise<T> {
    return new Promise((resolve, reject) => {
      const messageId = Math.random().toString(36);

      const handler = (event: MessageEvent) => {
        if (event.data.id === messageId) {
          this.worker.removeEventListener('message', handler);
          if (event.data.error) {
            reject(new Error(event.data.error));
          } else {
            resolve(event.data.result);
          }
        }
      };

      this.worker.addEventListener('message', handler);
      this.worker.postMessage({ id: messageId, task, data });
    });
  }

  terminate() {
    this.worker.terminate();
  }
}

// 使用例
const workerManager = new WorkerManager('worker.js');

async function processImage(imageData: ImageData) {
  const result = await workerManager.execute<ImageData>('process-image', imageData);
  return result;
}
```

## 7. まとめ

- Electronは **マルチプロセス・マルチスレッド** アーキテクチャ
- **Main Process**: アプリケーション全体の管理（1つ）
- **Renderer Process**: UI表示（ウィンドウごとに1つ）
- **IPC**: `invoke/handle` パターンが推奨
- **重い処理**: Worker Threads/Utility Processを活用
- **クラッシュハンドリング**: Rendererの再起動を実装
- **パフォーマンス**: バッチ処理、プロセスプールを活用

次のドキュメントでは、メモリとストレージ管理について詳しく解説します。
