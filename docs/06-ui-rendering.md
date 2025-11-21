# UIレンダリングとChromiumエンジン

## 1. Chromiumレンダリングパイプライン

### 1.1 レンダリングフロー

```
HTML → DOM Tree → Render Tree → Layout → Paint → Composite → Display
  │       │           │            │        │         │
  │       │           │            │        │         └─ GPU
  │       │           │            │        └─ Rasterization
  │       │           │            └─ Box Model Calculation
  │       │           └─ Visual Tree
  │       └─ JavaScript Execution
  └─ Parsing
```

### 1.2 詳細なレンダリングステップ

```typescript
// 1. HTMLパース → DOMツリー構築
// index.html が読み込まれる
<html>
  <head>
    <link rel="stylesheet" href="styles.css">
  </head>
  <body>
    <div id="app">
      <h1>Hello Electron</h1>
    </div>
    <script src="renderer.js"></script>
  </body>
</html>

// 2. CSSパース → CSSOMツリー構築
// styles.css が適用される

// 3. DOM + CSSOM → Render Tree
// 表示に必要な要素のみがRender Treeに含まれる

// 4. Layout (Reflow)
// 各要素の位置とサイズを計算

// 5. Paint
// Render Treeを実際のピクセルに変換

// 6. Composite
// レイヤーを合成して最終画像を生成
```

## 2. Blinkレンダリングエンジン

### 2.1 Blinkの主要コンポーネント

| コンポーネント | 役割 |
|---------------|------|
| **HTML Parser** | HTMLをDOMツリーに変換 |
| **CSS Parser** | CSSをCSSOMに変換 |
| **Style Resolver** | 要素にスタイルを適用 |
| **Layout Engine** | 要素の位置・サイズを計算 |
| **Paint** | グラフィックス命令を生成 |
| **Compositor** | レイヤーを合成 |

### 2.2 レイヤーの最適化

```css
/* GPUアクセラレーションを強制 */
.accelerated {
  transform: translateZ(0);
  /* または */
  will-change: transform;
}

/* 3D transformでレイヤー生成 */
.layer {
  transform: translate3d(0, 0, 0);
}

/* アニメーションの最適化 */
.animated {
  will-change: transform, opacity;
  /* アニメーション終了後はwill-changeを削除 */
}
```

### 2.3 リペイント・リフローの最適化

```typescript
// 悪い例: 複数回のレイアウト計算
function badExample() {
  const element = document.getElementById('box');

  // レイアウトを読み取る
  const height = element.offsetHeight;

  // スタイルを変更（リフロー発生）
  element.style.height = `${height + 10}px`;

  // また読み取る（強制リフロー）
  const width = element.offsetWidth;

  // また変更（リフロー発生）
  element.style.width = `${width + 10}px`;
}

// 良い例: バッチ処理
function goodExample() {
  const element = document.getElementById('box');

  // 読み取りをまとめる
  const height = element.offsetHeight;
  const width = element.offsetWidth;

  // 書き込みをまとめる
  requestAnimationFrame(() => {
    element.style.height = `${height + 10}px`;
    element.style.width = `${width + 10}px`;
  });
}

// さらに良い例: CSSクラスで変更
function bestExample() {
  const element = document.getElementById('box');
  element.classList.add('enlarged');
}
```

## 3. V8 JavaScriptエンジン

### 3.1 V8の最適化

```typescript
// 隠しクラスの最適化
class Point {
  x: number;
  y: number;

  constructor(x: number, y: number) {
    // プロパティを常に同じ順序で初期化
    this.x = x;
    this.y = y;
  }
}

// 悪い例: 動的にプロパティを追加
const point1 = { x: 1 };
point1.y = 2; // 隠しクラスが変更される

// 良い例: 初期化時にすべてのプロパティを定義
const point2 = { x: 1, y: 2 };

// インライン展開の最適化
function add(a: number, b: number): number {
  return a + b;
}

// 頻繁に呼ばれる小さな関数はインライン展開される
for (let i = 0; i < 10000; i++) {
  const result = add(i, 1);
}
```

### 3.2 メモリ管理とガベージコレクション

```typescript
// メモリリークを避ける
class DataManager {
  private listeners: Map<string, Function[]> = new Map();

  // イベントリスナーの登録
  addEventListener(event: string, callback: Function): void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event)!.push(callback);
  }

  // 重要: リスナーの削除
  removeEventListener(event: string, callback: Function): void {
    const listeners = this.listeners.get(event);
    if (listeners) {
      const index = listeners.indexOf(callback);
      if (index !== -1) {
        listeners.splice(index, 1);
      }
    }
  }

  // クリーンアップ
  cleanup(): void {
    this.listeners.clear();
  }
}

// WeakMapを使用したメモリリーク防止
const cache = new WeakMap<object, any>();

function cacheData(obj: object, data: any): void {
  cache.set(obj, data);
}

// objが不要になると自動的にGCされる
```

## 4. ハードウェアアクセラレーション

### 4.1 GPU加速の有効化

```typescript
// main.ts
import { app, BrowserWindow } from 'electron';

app.commandLine.appendSwitch('enable-gpu-rasterization');
app.commandLine.appendSwitch('enable-zero-copy');
app.commandLine.appendSwitch('ignore-gpu-blacklist');

const win = new BrowserWindow({
  webPreferences: {
    // ハードウェアアクセラレーションを有効化
    offscreen: false
  }
});

// GPUの状態を確認
win.webContents.on('did-finish-load', async () => {
  const info = await win.webContents.executeJavaScript(`
    (async () => {
      const adapter = await navigator.gpu?.requestAdapter();
      return adapter ? 'WebGPU available' : 'WebGPU not available';
    })()
  `);
  console.log(info);
});
```

### 4.2 Canvas/WebGLの最適化

```typescript
// Canvas 2Dの最適化
const canvas = document.getElementById('canvas') as HTMLCanvasElement;
const ctx = canvas.getContext('2d', {
  alpha: false, // 透明度が不要な場合
  desynchronized: true // 低レイテンシ
});

// オフスクリーンキャンバスでバックグラウンド描画
const offscreen = canvas.transferControlToOffscreen();
const worker = new Worker('canvas-worker.js');
worker.postMessage({ canvas: offscreen }, [offscreen]);

// canvas-worker.js
self.onmessage = (e) => {
  const canvas = e.data.canvas;
  const ctx = canvas.getContext('2d');

  function render() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = 'blue';
    ctx.fillRect(10, 10, 100, 100);

    requestAnimationFrame(render);
  }

  render();
};

// WebGLの最適化
const gl = canvas.getContext('webgl2', {
  antialias: false, // 不要な場合は無効化
  depth: false,
  powerPreference: 'high-performance'
});
```

## 5. CSS最適化

### 5.1 パフォーマンスの良いCSS

```css
/* 悪い例: 複雑なセレクタ */
body div.container ul li a.link span {
  color: red;
}

/* 良い例: シンプルなセレクタ */
.link-text {
  color: red;
}

/* transformとopacityはGPU加速される */
.animated {
  transition: transform 0.3s, opacity 0.3s;
}

.animated:hover {
  transform: scale(1.1);
  opacity: 0.8;
}

/* containプロパティで分離 */
.isolated {
  contain: layout style paint;
}

/* content-visibilityで遅延レンダリング */
.lazy-section {
  content-visibility: auto;
  contain-intrinsic-size: 500px;
}
```

### 5.2 Flexbox vs Grid

```css
/* Flexboxは1次元レイアウトに最適 */
.flex-container {
  display: flex;
  gap: 10px;
}

/* Gridは2次元レイアウトに最適 */
.grid-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 20px;
}
```

## 6. 仮想スクロール・レンダリング

### 6.1 大量データの効率的表示

```typescript
class VirtualList {
  private container: HTMLElement;
  private items: any[];
  private itemHeight: number;
  private visibleItems: number;
  private scrollTop: number = 0;

  constructor(
    container: HTMLElement,
    items: any[],
    itemHeight: number
  ) {
    this.container = container;
    this.items = items;
    this.itemHeight = itemHeight;
    this.visibleItems = Math.ceil(container.clientHeight / itemHeight) + 1;

    this.setupScroll();
    this.render();
  }

  private setupScroll(): void {
    this.container.style.height = `${this.items.length * this.itemHeight}px`;
    this.container.style.position = 'relative';

    this.container.addEventListener('scroll', () => {
      this.scrollTop = this.container.scrollTop;
      this.render();
    });
  }

  private render(): void {
    const startIndex = Math.floor(this.scrollTop / this.itemHeight);
    const endIndex = Math.min(
      startIndex + this.visibleItems,
      this.items.length
    );

    // 既存の要素をクリア
    this.container.innerHTML = '';

    // 表示範囲の要素のみレンダリング
    for (let i = startIndex; i < endIndex; i++) {
      const item = this.items[i];
      const element = this.createItemElement(item, i);
      this.container.appendChild(element);
    }
  }

  private createItemElement(item: any, index: number): HTMLElement {
    const div = document.createElement('div');
    div.style.position = 'absolute';
    div.style.top = `${index * this.itemHeight}px`;
    div.style.height = `${this.itemHeight}px`;
    div.textContent = item.text;
    return div;
  }
}

// 使用例
const items = Array.from({ length: 10000 }, (_, i) => ({
  text: `Item ${i}`
}));

const virtualList = new VirtualList(
  document.getElementById('list')!,
  items,
  50 // 各アイテムの高さ
);
```

## 7. React/Vue等のフレームワーク統合

### 7.1 React with Electron

```typescript
// App.tsx
import React, { useEffect, useState } from 'react';

function App() {
  const [data, setData] = useState<string>('');

  useEffect(() => {
    // Electron APIを呼び出す
    window.electronAPI.getData()
      .then(result => setData(result));
  }, []);

  const handleSave = async () => {
    await window.electronAPI.saveData(data);
  };

  return (
    <div className="app">
      <h1>Electron + React</h1>
      <textarea
        value={data}
        onChange={(e) => setData(e.target.value)}
      />
      <button onClick={handleSave}>Save</button>
    </div>
  );
}

export default App;
```

### 7.2 パフォーマンス監視

```typescript
// renderer.ts
import { PerformanceObserver } from 'perf_hooks';

// レンダリングパフォーマンスの監視
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.duration}ms`);
  }
});

observer.observe({ entryTypes: ['measure', 'navigation', 'paint'] });

// カスタム計測
performance.mark('render-start');

// レンダリング処理
renderUI();

performance.mark('render-end');
performance.measure('render', 'render-start', 'render-end');

// FPS計測
let lastTime = performance.now();
let frames = 0;

function measureFPS() {
  frames++;
  const currentTime = performance.now();

  if (currentTime >= lastTime + 1000) {
    const fps = Math.round((frames * 1000) / (currentTime - lastTime));
    console.log(`FPS: ${fps}`);
    frames = 0;
    lastTime = currentTime;
  }

  requestAnimationFrame(measureFPS);
}

measureFPS();
```

## 8. DevTools活用

### 8.1 DevToolsの開き方

```typescript
// main.ts
const win = new BrowserWindow({ ... });

// 開発時は自動的にDevToolsを開く
if (process.env.NODE_ENV === 'development') {
  win.webContents.openDevTools({ mode: 'detach' });
}

// F12キーでDevToolsを切り替え
win.webContents.on('before-input-event', (event, input) => {
  if (input.key === 'F12') {
    win.webContents.toggleDevTools();
  }
});
```

### 8.2 パフォーマンスプロファイリング

```typescript
// Performanceタブで記録開始
console.profile('MyProfile');

// 重い処理
heavyComputation();

// 記録終了
console.profileEnd('MyProfile');

// メモリスナップショット
console.time('memory-snapshot');
const snapshot = (performance as any).memory;
console.log('Used JS Heap Size:', snapshot.usedJSHeapSize);
console.log('Total JS Heap Size:', snapshot.totalJSHeapSize);
console.timeEnd('memory-snapshot');
```

## 9. まとめ

- **Chromium**: Blink（レンダリング）+ V8（JavaScript）
- **最適化**: リフロー/リペイントを最小化
- **GPU加速**: transform, opacity を使用
- **仮想化**: 大量データは仮想スクロールで表示
- **監視**: DevTools と Performance API で計測

次のドキュメントでは、オフライン動作とデータ同期について解説します。
