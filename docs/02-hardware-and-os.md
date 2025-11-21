# ハードウェアとOS基盤

## 1. Electronアプリが動作するハードウェア

### 1.1 必要なハードウェアスペック

#### 最小要件
```
CPU: x86_64 / ARM64 (Apple Silicon)
RAM: 2GB以上
ストレージ: 200MB以上（アプリサイズによる）
ディスプレイ: 1024x768以上
ネットワーク: オプション（オフライン動作可能）
```

#### 推奨スペック
```
CPU: 4コア以上、2.0GHz+
RAM: 8GB以上
ストレージ: SSD、1GB以上の空き容量
ディスプレイ: 1920x1080以上、GPU加速対応
ネットワーク: 高速インターネット接続
```

### 1.2 CPUアーキテクチャ

Electronは以下のアーキテクチャをサポート:

| アーキテクチャ | プラットフォーム | 備考 |
|---------------|-----------------|------|
| **x64 (x86_64)** | Windows, macOS, Linux | 最も一般的 |
| **arm64** | macOS (M1/M2/M3), Linux | Apple Silicon対応 |
| **ia32 (x86)** | Windows, Linux | 32bitサポート（非推奨） |
| **armv7l** | Linux | Raspberry Pi等 |

### 1.3 GPU（グラフィックス処理）

Chromiumは **ハードウェアアクセラレーション** を使用:

```typescript
// GPUの有効化/無効化
import { app } from 'electron';

// GPUアクセラレーションを無効化
app.disableHardwareAcceleration();

// 特定のGPU機能を無効化
app.commandLine.appendSwitch('disable-gpu');
app.commandLine.appendSwitch('disable-software-rasterizer');
```

**GPU機能**:
- **Canvas/WebGL描画**: ゲーム、データビジュアライゼーション
- **ビデオデコード**: H.264、VP8/VP9、AV1
- **CSSアニメーション**: transform、transitionの高速化

### 1.4 ストレージとI/O

```typescript
// ストレージパスの取得
import { app } from 'electron';

// アプリケーションデータディレクトリ
const userDataPath = app.getPath('userData');
// Windows: C:\Users\<user>\AppData\Roaming\<app-name>
// macOS: ~/Library/Application Support/<app-name>
// Linux: ~/.config/<app-name>

const documentsPath = app.getPath('documents');
const tempPath = app.getPath('temp');
```

## 2. オペレーティングシステム (OS)

### 2.1 対応OS

#### Windows
```
サポートバージョン:
- Windows 10 (1809+)
- Windows 11
- Windows Server 2019+

アーキテクチャ:
- x64 (64bit)
- ia32 (32bit, 非推奨)
- arm64 (Surface Pro X等)
```

#### macOS
```
サポートバージョン:
- macOS 10.15 (Catalina) 以降
- macOS 11 (Big Sur)
- macOS 12 (Monterey)
- macOS 13 (Ventura)
- macOS 14 (Sonoma)

アーキテクチャ:
- x64 (Intel Mac)
- arm64 (Apple Silicon: M1/M2/M3)
- Universal Binary (両対応)
```

#### Linux
```
サポートディストリビューション:
- Ubuntu 18.04+
- Debian 10+
- Fedora 32+
- openSUSE 15.2+

デスクトップ環境:
- GNOME
- KDE Plasma
- XFCE
- その他（X11/Wayland対応）
```

### 2.2 OS APIとの統合

#### ファイルシステムアクセス

```typescript
import { dialog, app } from 'electron';
import * as fs from 'fs/promises';
import * as path from 'path';

// ファイル選択ダイアログ
async function openFile() {
  const result = await dialog.showOpenDialog({
    properties: ['openFile', 'multiSelections'],
    filters: [
      { name: 'Images', extensions: ['jpg', 'png', 'gif'] },
      { name: 'All Files', extensions: ['*'] }
    ]
  });

  if (!result.canceled) {
    const filePath = result.filePaths[0];
    const content = await fs.readFile(filePath, 'utf-8');
    return content;
  }
}

// ファイル保存
async function saveFile(content: string) {
  const result = await dialog.showSaveDialog({
    defaultPath: path.join(app.getPath('documents'), 'untitled.txt'),
    filters: [{ name: 'Text', extensions: ['txt'] }]
  });

  if (!result.canceled && result.filePath) {
    await fs.writeFile(result.filePath, content, 'utf-8');
  }
}
```

#### システムトレイ

```typescript
import { Tray, Menu, nativeImage } from 'electron';

let tray: Tray | null = null;

function createTray() {
  const icon = nativeImage.createFromPath('/path/to/icon.png');
  tray = new Tray(icon);

  const contextMenu = Menu.buildFromTemplate([
    { label: '開く', click: () => { /* ... */ } },
    { label: '終了', click: () => app.quit() }
  ]);

  tray.setContextMenu(contextMenu);
  tray.setToolTip('My Electron App');
}
```

#### 通知 (Notifications)

```typescript
import { Notification } from 'electron';

function showNotification() {
  const notification = new Notification({
    title: 'タイトル',
    body: 'メッセージ本文',
    icon: '/path/to/icon.png',
    silent: false
  });

  notification.on('click', () => {
    console.log('通知がクリックされました');
  });

  notification.show();
}
```

### 2.3 プロセス管理

#### OSレベルのプロセス

```
Electronアプリの起動時のプロセスツリー:

electron (Main Process)
├── electron --type=gpu-process
├── electron --type=utility
└── electron --type=renderer (BrowserWindow毎に1つ)
    ├── electron --type=renderer (別ウィンドウ)
    └── electron --type=renderer (別ウィンドウ)
```

#### プロセス情報の取得

```typescript
import { app } from 'electron';

// プロセスメトリクスの取得
const metrics = app.getAppMetrics();

metrics.forEach((metric) => {
  console.log(`Process ID: ${metric.pid}`);
  console.log(`Type: ${metric.type}`); // Browser, Tab, GPU, etc.
  console.log(`CPU: ${metric.cpu.percentCPUUsage}%`);
  console.log(`Memory: ${metric.memory.workingSetSize / 1024}KB`);
});

// システム情報
console.log(`Platform: ${process.platform}`); // win32, darwin, linux
console.log(`Arch: ${process.arch}`); // x64, arm64, ia32
console.log(`Node Version: ${process.versions.node}`);
console.log(`Electron Version: ${process.versions.electron}`);
console.log(`Chrome Version: ${process.versions.chrome}`);
```

### 2.4 スレッド管理

Electronは複数のスレッドを使用:

#### Mainプロセスのスレッド
1. **Main Thread**: JavaScriptの実行
2. **File I/O Thread**: ファイルシステム操作
3. **Network Thread**: ネットワーク通信

#### Rendererプロセスのスレッド
1. **Main Thread**: UIレンダリング、JavaScript実行
2. **Compositor Thread**: 画面合成
3. **Web Worker Threads**: バックグラウンド処理

#### Web Workerの使用

```typescript
// renderer.ts
const worker = new Worker('worker.js');

worker.postMessage({ type: 'heavy-calculation', data: [1, 2, 3] });

worker.onmessage = (event) => {
  console.log('結果:', event.data);
};

// worker.js
self.onmessage = (event) => {
  const { type, data } = event.data;

  if (type === 'heavy-calculation') {
    const result = data.reduce((a, b) => a + b, 0);
    self.postMessage(result);
  }
};
```

## 3. ネットワークプロトコル

### 3.1 ネットワークスタック

Electronは **Chromiumのネットワークスタック** を使用:

```
┌─────────────────────────────────┐
│   アプリケーション層              │
│   (HTTP/HTTPS, WebSocket)       │
├─────────────────────────────────┤
│   Chromium Network Stack        │
│   - HTTP/1.1, HTTP/2, HTTP/3    │
│   - TLS 1.2, 1.3                │
│   - QUIC                        │
├─────────────────────────────────┤
│   TCP/UDP                       │
├─────────────────────────────────┤
│   IP (IPv4/IPv6)                │
├─────────────────────────────────┤
│   OS Network Layer              │
└─────────────────────────────────┘
```

### 3.2 プロキシ設定

```typescript
import { session } from 'electron';

// プロキシの設定
await session.defaultSession.setProxy({
  proxyRules: 'http://proxy.example.com:8080',
  proxyBypassRules: 'localhost,127.0.0.1'
});

// プロキシ設定の取得
const proxyUrl = await session.defaultSession.resolveProxy('https://example.com');
console.log(proxyUrl); // 'PROXY proxy.example.com:8080'
```

### 3.3 証明書管理

```typescript
import { app } from 'electron';

// 証明書エラーのハンドリング
app.on('certificate-error', (event, webContents, url, error, certificate, callback) => {
  // 開発環境でのみ自己署名証明書を許可
  if (url.startsWith('https://localhost')) {
    event.preventDefault();
    callback(true);
  } else {
    callback(false);
  }
});

// カスタムCA証明書の追加
app.on('ready', () => {
  session.defaultSession.setCertificateVerifyProc((request, callback) => {
    // カスタム検証ロジック
    callback(0); // 0 = 成功, -2 = 失敗
  });
});
```

### 3.4 DNSとホスト解決

```typescript
import { net } from 'electron';

// DNS解決
async function resolveDNS(hostname: string) {
  const request = net.request({
    method: 'HEAD',
    url: `https://${hostname}`
  });

  request.on('response', (response) => {
    console.log(`IP: ${response.socket.remoteAddress}`);
  });

  request.end();
}
```

## 4. メモリ管理

### 4.1 メモリモデル

```
┌──────────────────────────────────────┐
│  Electronアプリのメモリ空間           │
├──────────────────────────────────────┤
│  Main Process Memory                │
│  - Node.js Heap (V8)                │
│  - Native Memory (C++ Objects)      │
├──────────────────────────────────────┤
│  Renderer Process 1 Memory          │
│  - JavaScript Heap                  │
│  - DOM Objects                      │
│  - WebGL Textures                   │
├──────────────────────────────────────┤
│  Renderer Process 2 Memory          │
│  ...                                │
├──────────────────────────────────────┤
│  GPU Process Memory                 │
│  - GPU Buffers                      │
│  - Textures                         │
└──────────────────────────────────────┘
```

### 4.2 メモリ使用量の監視

```typescript
import { app, BrowserWindow } from 'electron';

// プロセスメモリ情報
function getMemoryInfo() {
  const metrics = app.getAppMetrics();

  metrics.forEach((metric) => {
    console.log(`Process ${metric.pid} (${metric.type}):`);
    console.log(`  Working Set: ${metric.memory.workingSetSize / 1024 / 1024}MB`);
    console.log(`  Private Bytes: ${metric.memory.privateBytes / 1024 / 1024}MB`);
  });
}

// Rendererプロセスのメモリ情報
const win = new BrowserWindow({});
win.webContents.on('did-finish-load', async () => {
  const info = await win.webContents.executeJavaScript(`
    performance.memory
  `);

  console.log(`JS Heap Size: ${info.usedJSHeapSize / 1024 / 1024}MB`);
  console.log(`Total Heap Size: ${info.totalJSHeapSize / 1024 / 1024}MB`);
});
```

### 4.3 メモリリーク対策

```typescript
class WindowManager {
  private windows: Set<BrowserWindow> = new Set();

  createWindow() {
    const win = new BrowserWindow({
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true
      }
    });

    this.windows.add(win);

    // ウィンドウが閉じられたら参照を削除
    win.on('closed', () => {
      this.windows.delete(win);
    });

    return win;
  }

  closeAll() {
    this.windows.forEach(win => {
      if (!win.isDestroyed()) {
        win.close();
      }
    });
    this.windows.clear();
  }
}
```

## 5. ストレージ管理

### 5.1 ストレージの種類

| ストレージタイプ | 用途 | パス例 |
|-----------------|------|--------|
| **userData** | アプリ設定、データベース | `~/.config/app-name` |
| **temp** | 一時ファイル | `/tmp` |
| **cache** | キャッシュデータ | `~/.cache/app-name` |
| **logs** | ログファイル | `~/Library/Logs/app-name` |
| **documents** | ユーザードキュメント | `~/Documents` |

### 5.2 永続化ストレージ

```typescript
import Store from 'electron-store';

// electron-storeの使用
const store = new Store({
  name: 'config',
  defaults: {
    windowBounds: { width: 800, height: 600 }
  }
});

// データの保存
store.set('user.name', 'John Doe');
store.set('user.preferences', { theme: 'dark' });

// データの読み込み
const userName = store.get('user.name');
const preferences = store.get('user.preferences');

// データの削除
store.delete('user.name');
```

### 5.3 IndexedDB

```typescript
// renderer.ts
const request = indexedDB.open('MyDatabase', 1);

request.onupgradeneeded = (event) => {
  const db = (event.target as IDBOpenDBRequest).result;
  const objectStore = db.createObjectStore('users', { keyPath: 'id' });
  objectStore.createIndex('name', 'name', { unique: false });
};

request.onsuccess = (event) => {
  const db = (event.target as IDBOpenDBRequest).result;

  // データの追加
  const transaction = db.transaction(['users'], 'readwrite');
  const objectStore = transaction.objectStore('users');
  objectStore.add({ id: 1, name: 'John', email: 'john@example.com' });
};
```

### 5.4 ファイルシステムの監視

```typescript
import * as fs from 'fs';
import * as path from 'path';

// ファイル変更の監視
const watcher = fs.watch(
  path.join(app.getPath('userData'), 'config.json'),
  (eventType, filename) => {
    console.log(`File ${filename} changed: ${eventType}`);
  }
);

// 監視の停止
watcher.close();
```

## 6. まとめ

- Electronは **x64/arm64** などの主要CPUアーキテクチャをサポート
- **Windows/macOS/Linux** で動作するクロスプラットフォーム対応
- OSの **ネイティブAPI** にアクセス可能（ファイル、通知、トレイ等）
- Chromiumの **ネットワークスタック** を使用（HTTP/2、HTTP/3、WebSocket）
- **マルチプロセス・マルチスレッド** アーキテクチャ
- **メモリとストレージ** の効率的な管理が重要

次のドキュメントでは、プロセスとスレッド管理について詳しく解説します。
