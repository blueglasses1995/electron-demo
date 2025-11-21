# ハンズオンカリキュラム Part 1: 環境構築

## 目標

このレッスンでは、Electronアプリ開発に必要な環境を構築します。

## 前提条件

- 基本的なTypeScriptの知識
- ターミナル/コマンドラインの基本操作
- テキストエディタ（VS Code推奨）

## Step 1: Node.jsのインストール

### 1.1 Node.jsのバージョン確認

```bash
node --version
npm --version
```

**推奨バージョン**: Node.js 18.x以上、npm 9.x以上

### 1.2 Node.jsのインストール（必要な場合）

#### macOS / Linux (nvm使用)

```bash
# nvmのインストール
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# シェルの再起動後
nvm install 18
nvm use 18
nvm alias default 18
```

#### Windows (nvm-windows使用)

1. https://github.com/coreybutler/nvm-windows/releases からインストーラーをダウンロード
2. インストール後、コマンドプロンプトで:

```cmd
nvm install 18
nvm use 18
```

## Step 2: プロジェクトの初期化

### 2.1 プロジェクトディレクトリの作成

```bash
mkdir my-electron-app
cd my-electron-app
```

### 2.2 npm初期化

```bash
npm init -y
```

### 2.3 package.jsonの編集

```json
{
  "name": "my-electron-app",
  "version": "1.0.0",
  "description": "My first Electron app",
  "main": "dist/main.js",
  "scripts": {
    "start": "electron .",
    "dev": "tsc && electron .",
    "build": "tsc",
    "watch": "tsc --watch"
  },
  "keywords": ["electron", "typescript"],
  "author": "Your Name",
  "license": "MIT"
}
```

## Step 3: 依存パッケージのインストール

### 3.1 Electronのインストール

```bash
npm install --save-dev electron
```

### 3.2 TypeScriptのインストール

```bash
npm install --save-dev typescript
npm install --save-dev @types/node
```

### 3.3 開発ツールのインストール

```bash
# electron-reloadで自動リロード
npm install --save-dev electron-reload

# electron-builderでパッケージング
npm install --save-dev electron-builder

# その他の便利なツール
npm install --save-dev concurrently
npm install --save-dev wait-on
```

## Step 4: TypeScriptの設定

### 4.1 tsconfig.jsonの作成

```bash
npx tsc --init
```

### 4.2 tsconfig.jsonの編集

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020", "DOM"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node",
    "types": ["node"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## Step 5: プロジェクト構造の作成

### 5.1 ディレクトリ構造

```
my-electron-app/
├── src/
│   ├── main/
│   │   └── main.ts
│   ├── preload/
│   │   └── preload.ts
│   └── renderer/
│       ├── index.html
│       ├── renderer.ts
│       └── styles.css
├── dist/           (ビルド後に生成)
├── package.json
├── tsconfig.json
└── .gitignore
```

### 5.2 ディレクトリの作成

```bash
mkdir -p src/main src/preload src/renderer
```

### 5.3 .gitignoreの作成

```gitignore
# 依存パッケージ
node_modules/

# ビルド成果物
dist/
release/
out/

# ログ
*.log
npm-debug.log*

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/*
!.vscode/extensions.json
!.vscode/settings.json
.idea/
*.swp
*.swo

# 環境変数
.env
.env.local
```

## Step 6: 最小限のElectronアプリの作成

### 6.1 src/main/main.ts

```typescript
import { app, BrowserWindow } from 'electron';
import * as path from 'path';

function createWindow(): void {
  const mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, '../preload/preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  });

  mainWindow.loadFile(path.join(__dirname, '../renderer/index.html'));

  // 開発時はDevToolsを開く
  if (process.env.NODE_ENV === 'development') {
    mainWindow.webContents.openDevTools();
  }
}

app.whenReady().then(() => {
  createWindow();

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow();
    }
  });
});

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});
```

### 6.2 src/preload/preload.ts

```typescript
import { contextBridge } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  platform: process.platform,
  versions: {
    node: process.versions.node,
    chrome: process.versions.chrome,
    electron: process.versions.electron
  }
});
```

### 6.3 src/renderer/index.html

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>My Electron App</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div class="container">
    <h1>Welcome to Electron!</h1>
    <div id="info">
      <p>Platform: <span id="platform"></span></p>
      <p>Node.js: <span id="node-version"></span></p>
      <p>Chromium: <span id="chrome-version"></span></p>
      <p>Electron: <span id="electron-version"></span></p>
    </div>
  </div>
  <script src="renderer.js"></script>
</body>
</html>
```

### 6.4 src/renderer/renderer.ts

```typescript
// グローバルに定義されたelectronAPIを使用
declare global {
  interface Window {
    electronAPI: {
      platform: string;
      versions: {
        node: string;
        chrome: string;
        electron: string;
      };
    };
  }
}

// バージョン情報を表示
document.getElementById('platform')!.textContent = window.electronAPI.platform;
document.getElementById('node-version')!.textContent = window.electronAPI.versions.node;
document.getElementById('chrome-version')!.textContent = window.electronAPI.versions.chrome;
document.getElementById('electron-version')!.textContent = window.electronAPI.versions.electron;

console.log('Renderer process initialized');
```

### 6.5 src/renderer/styles.css

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #333;
}

.container {
  background: white;
  padding: 40px;
  border-radius: 12px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
  max-width: 600px;
  width: 90%;
}

h1 {
  color: #667eea;
  margin-bottom: 24px;
  font-size: 32px;
  text-align: center;
}

#info {
  background: #f7f7f7;
  padding: 20px;
  border-radius: 8px;
}

#info p {
  margin: 12px 0;
  font-size: 16px;
}

#info span {
  font-weight: bold;
  color: #667eea;
}
```

## Step 7: ビルドとテスト

### 7.1 TypeScriptのコンパイル

```bash
npm run build
```

### 7.2 アプリの起動

```bash
npm start
```

成功すると、Electronウィンドウが開き、バージョン情報が表示されます。

## Step 8: 開発環境の改善

### 8.1 package.jsonにスクリプト追加

```json
{
  "scripts": {
    "start": "electron .",
    "dev": "npm run build && electron .",
    "build": "tsc",
    "watch": "tsc --watch",
    "clean": "rm -rf dist"
  }
}
```

### 8.2 VS Code設定（.vscode/launch.json）

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Main Process",
      "type": "node",
      "request": "launch",
      "cwd": "${workspaceFolder}",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
      "windows": {
        "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron.cmd"
      },
      "args": ["."],
      "outputCapture": "std",
      "preLaunchTask": "npm: build"
    }
  ]
}
```

### 8.3 VS Code設定（.vscode/settings.json）

```json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll": true
  }
}
```

## 演習

1. **カスタマイズ**: index.htmlのタイトルと見出しを変更してみましょう
2. **スタイル変更**: styles.cssで背景色やフォントを変更してみましょう
3. **新しい情報追加**: preload.tsで`process.arch`（CPUアーキテクチャ）を追加してみましょう

## トラブルシューティング

### エラー: "Cannot find module 'electron'"

```bash
# node_modulesを削除して再インストール
rm -rf node_modules package-lock.json
npm install
```

### エラー: "tsc: command not found"

```bash
# TypeScriptをグローバルにインストール
npm install -g typescript

# またはnpxを使用
npx tsc
```

### Windowsで起動しない

```bash
# Windowsの場合、スクリプトを変更
"start": "electron.cmd ."
```

## まとめ

このレッスンで学んだこと:
- Node.jsとnpmの環境構築
- Electronプロジェクトの初期化
- TypeScriptの設定
- 最小限のElectronアプリの作成
- プロジェクト構造のベストプラクティス

次のレッスンでは、基本的なアプリケーション機能を追加していきます。
