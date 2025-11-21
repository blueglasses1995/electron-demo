# Electronアーキテクチャの基礎

## 1. Electronとは

Electronは、Chromium（レンダリングエンジン）とNode.js（JavaScriptランタイム）を組み合わせたクロスプラットフォームのデスクトップアプリケーション開発フレームワークです。

### 1.1 主要コンポーネント

```
┌─────────────────────────────────────────┐
│         Electronアプリケーション           │
├─────────────────────────────────────────┤
│  ┌─────────────┐    ┌───────────────┐  │
│  │ Main Process │◄───┤ IPC (通信)     │  │
│  │  (Node.js)   │    │               │  │
│  └──────┬───────┘    │               │  │
│         │            │               │  │
│         │ 管理        │               │  │
│         ▼            │               │  │
│  ┌─────────────────┐ │               │  │
│  │ Renderer Process│◄┤               │  │
│  │   (Chromium)    │ │               │  │
│  │  ・HTML         │ │               │  │
│  │  ・CSS          │ │               │  │
│  │  ・JavaScript   │ │               │  │
│  └─────────────────┘ └───────────────┘  │
├─────────────────────────────────────────┤
│         ┌──────────┐                    │
│         │ Node.js  │                    │
│         │ Runtime  │                    │
│         └──────────┘                    │
├─────────────────────────────────────────┤
│         ┌──────────┐                    │
│         │ Chromium │                    │
│         │  Engine  │                    │
│         └──────────┘                    │
├─────────────────────────────────────────┤
│         ┌──────────┐                    │
│         │    OS    │                    │
│         │  (Win/   │                    │
│         │ Mac/Lin) │                    │
│         └──────────┘                    │
└─────────────────────────────────────────┘
```

### 1.2 コアテクノロジー

#### Chromium（レンダリングエンジン）
- **バージョン**: Electronのバージョンに依存（例: Electron 27.x → Chromium 118.x）
- **役割**: HTML/CSS/JavaScriptのレンダリング、DOM操作、Webブラウザ機能
- **コンポーネント**:
  - **Blink**: HTMLとCSSのレンダリングエンジン
  - **V8**: JavaScriptエンジン
  - **Skia**: 2Dグラフィックスライブラリ

#### Node.js（サーバーサイドランタイム）
- **バージョン**: Electronのバージョンに依存（例: Electron 27.x → Node.js 18.x）
- **役割**: ファイルシステムアクセス、ネットワーク通信、OS APIへのアクセス
- **特徴**:
  - イベント駆動型、非同期I/O
  - npm/yarnエコシステムへのアクセス

## 2. プロセスモデル

### 2.1 Mainプロセス

**役割と責任**:
```typescript
// main.ts の例
import { app, BrowserWindow } from 'electron';

// アプリケーションのライフサイクル管理
app.on('ready', () => {
  // ウィンドウの作成
  const mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, 'preload.js')
    }
  });
});

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});
```

**Mainプロセスの責務**:
1. **ウィンドウ管理**: BrowserWindowの作成・管理
2. **アプリケーションライフサイクル**: 起動、終了、イベント処理
3. **システムリソースアクセス**: ファイルシステム、ネットワーク、ネイティブAPI
4. **メニュー・トレイ**: アプリケーションメニュー、システムトレイの管理
5. **自動更新**: アプリケーションの更新処理

### 2.2 Rendererプロセス

**役割と責任**:
```typescript
// renderer.ts の例
// DOMの操作
document.getElementById('button')?.addEventListener('click', async () => {
  // Preload経由でMainプロセスと通信
  const result = await window.electronAPI.doSomething();
  console.log(result);
});
```

**Rendererプロセスの責務**:
1. **UI表示**: HTML/CSS/JavaScriptによる画面表示
2. **ユーザーインタラクション**: イベント処理、フォーム入力
3. **フロントエンドロジック**: ビジネスロジックの実行
4. **Webコンテンツのレンダリング**: ブラウザと同等の機能

### 2.3 Preloadスクリプト

**セキュリティブリッジ**:
```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';

// セキュアなAPIの公開
contextBridge.exposeInMainWorld('electronAPI', {
  doSomething: () => ipcRenderer.invoke('do-something'),
  onUpdate: (callback) => ipcRenderer.on('update', callback)
});
```

**Preloadの役割**:
1. **セキュリティ境界**: RendererとMainの間の安全な通信路
2. **APIの限定公開**: 必要最小限のAPIのみを公開
3. **コンテキスト分離**: nodeIntegrationを無効化した状態でNode.js機能を使用

## 3. セキュリティモデル

### 3.1 セキュリティ設定

```typescript
const window = new BrowserWindow({
  webPreferences: {
    // Node.jsの統合を無効化（重要）
    nodeIntegration: false,

    // コンテキストの分離（重要）
    contextIsolation: true,

    // リモートモジュールを無効化
    enableRemoteModule: false,

    // サンドボックスモードを有効化
    sandbox: true,

    // Preloadスクリプトの指定
    preload: path.join(__dirname, 'preload.js')
  }
});
```

### 3.2 Content Security Policy (CSP)

```typescript
// index.htmlのmetaタグ
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self';
               style-src 'self' 'unsafe-inline'">
```

## 4. アプリケーションのライフサイクル

```typescript
import { app } from 'electron';

// アプリケーション準備完了
app.on('ready', () => {
  console.log('アプリ起動');
});

// 全ウィンドウが閉じられた
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

// アプリケーションがアクティブ化（macOS）
app.on('activate', () => {
  if (BrowserWindow.getAllWindows().length === 0) {
    createWindow();
  }
});

// アプリケーション終了前
app.on('before-quit', (event) => {
  // クリーンアップ処理
});

// アプリケーション終了
app.on('will-quit', () => {
  // 最終クリーンアップ
});
```

## 5. Electronとモバイルアプリの違い

### 5.1 実行環境の違い

| 項目 | Electron | モバイルアプリ |
|------|----------|---------------|
| **プラットフォーム** | Windows, macOS, Linux | iOS, Android |
| **ランタイム** | Node.js + Chromium | ネイティブ (Swift/Kotlin) または React Native |
| **配布方法** | 直接ダウンロード、Microsoft Store、Mac App Store | App Store, Google Play Store |
| **サンドボックス** | OS依存（macOS Sandboxあり） | 厳格なサンドボックス |
| **システムアクセス** | 広範なOS APIアクセス可能 | 制限されたAPI、権限システム |

### 5.2 アプリケーションサイズ

- **Electron**: 通常 50-150MB（ChromiumとNode.jsを含む）
- **モバイル**: 通常 10-50MB（ネイティブバイナリ）

### 5.3 配布モデル

**Electronの配布**:
- 自己ホスト型（独自サーバーからダウンロード）
- Microsoft Store（Windows）
- Mac App Store（macOS）
- Snap Store（Linux）

**モバイルの配布**:
- App Store（iOS）必須
- Google Play Store（Android）推奨

## 6. Electronアプリの起動フロー

```
1. ユーザーがアプリを起動
   ↓
2. OSがElectronバイナリを実行
   ↓
3. Node.jsランタイムの初期化
   ↓
4. Mainプロセスの起動（main.js実行）
   ↓
5. app.ready イベント発火
   ↓
6. BrowserWindowの作成
   ↓
7. Chromiumの初期化
   ↓
8. Rendererプロセスの起動
   ↓
9. preload.jsの実行（コンテキスト分離環境）
   ↓
10. HTMLの読み込み
   ↓
11. renderer.jsの実行
   ↓
12. アプリケーション準備完了
```

## 7. まとめ

- Electronは **Chromium + Node.js** のハイブリッドフレームワーク
- **Mainプロセス** がアプリケーション全体を管理
- **Rendererプロセス** が個々のウィンドウのUIを表示
- **IPC** でプロセス間通信を行う
- **セキュリティ** は `contextIsolation` と `preload` で確保
- デスクトップアプリケーションに特化（モバイルは非対応）

次のドキュメントでは、ハードウェアとOS基盤について詳しく解説します。
