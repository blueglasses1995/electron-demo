# ハンズオンカリキュラム Part 5: パッケージングとデプロイ

## 目標

electron-builderを使用してアプリをパッケージングし、自動更新機能を実装します。

## 前提条件

Part 1-4を完了していること

## Step 1: electron-builderの設定

### 1.1 依存パッケージのインストール

```bash
npm install --save-dev electron-builder
npm install electron-updater
```

### 1.2 package.jsonの更新

```json
{
  "name": "my-notepad-app",
  "version": "1.0.0",
  "description": "クラウド同期対応メモ帳",
  "main": "dist/main.js",
  "author": "Your Name <your.email@example.com>",
  "license": "MIT",
  "scripts": {
    "start": "electron .",
    "dev": "npm run build && electron .",
    "build": "tsc",
    "watch": "tsc --watch",
    "clean": "rm -rf dist release",
    "pack": "electron-builder --dir",
    "dist": "npm run build && electron-builder",
    "dist:win": "npm run build && electron-builder --win",
    "dist:mac": "npm run build && electron-builder --mac",
    "dist:linux": "npm run build && electron-builder --linux",
    "publish": "npm run build && electron-builder --publish always"
  },
  "build": {
    "appId": "com.example.mynotepad",
    "productName": "My Notepad",
    "copyright": "Copyright © 2024 ${author}",
    "directories": {
      "output": "release",
      "buildResources": "build"
    },
    "files": [
      "dist/**/*",
      "node_modules/**/*",
      "package.json"
    ],
    "extraResources": [
      {
        "from": "resources",
        "to": "resources",
        "filter": ["**/*"]
      }
    ],
    "win": {
      "target": [
        {
          "target": "nsis",
          "arch": ["x64", "ia32"]
        },
        {
          "target": "portable",
          "arch": ["x64"]
        }
      ],
      "icon": "build/icon.ico",
      "artifactName": "${productName}-${version}-${os}-${arch}.${ext}"
    },
    "nsis": {
      "oneClick": false,
      "allowToChangeInstallationDirectory": true,
      "allowElevation": true,
      "createDesktopShortcut": true,
      "createStartMenuShortcut": true,
      "shortcutName": "My Notepad",
      "perMachine": false,
      "runAfterFinish": true,
      "installerIcon": "build/icon.ico",
      "uninstallerIcon": "build/icon.ico",
      "installerHeader": "build/installerHeader.bmp",
      "installerSidebar": "build/installerSidebar.bmp"
    },
    "mac": {
      "target": [
        {
          "target": "dmg",
          "arch": ["universal"]
        },
        "zip"
      ],
      "category": "public.app-category.productivity",
      "icon": "build/icon.icns",
      "hardenedRuntime": true,
      "gatekeeperAssess": false,
      "entitlements": "build/entitlements.mac.plist",
      "entitlementsInherit": "build/entitlements.mac.plist",
      "darkModeSupport": true
    },
    "dmg": {
      "contents": [
        {
          "x": 130,
          "y": 220
        },
        {
          "x": 410,
          "y": 220,
          "type": "link",
          "path": "/Applications"
        }
      ],
      "title": "${productName} ${version}",
      "background": "build/dmg-background.png"
    },
    "linux": {
      "target": [
        "AppImage",
        "deb",
        "rpm"
      ],
      "category": "Utility",
      "icon": "build/icons",
      "maintainer": "your.email@example.com",
      "synopsis": "クラウド同期対応メモ帳アプリ"
    },
    "publish": {
      "provider": "github",
      "owner": "your-username",
      "repo": "my-notepad-app",
      "releaseType": "release"
    }
  }
}
```

## Step 2: アイコンとリソースの準備

### 2.1 ディレクトリ構造

```
project-root/
├── build/
│   ├── icon.ico        (Windows, 256x256)
│   ├── icon.icns       (macOS)
│   ├── icons/          (Linux, 複数サイズ)
│   │   ├── 16x16.png
│   │   ├── 32x32.png
│   │   ├── 48x48.png
│   │   ├── 64x64.png
│   │   ├── 128x128.png
│   │   ├── 256x256.png
│   │   └── 512x512.png
│   └── entitlements.mac.plist
└── resources/
    └── (追加リソースファイル)
```

### 2.2 entitlements.mac.plist の作成

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.security.cs.allow-jit</key>
  <true/>
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
  <true/>
  <key>com.apple.security.cs.disable-library-validation</key>
  <true/>
  <key>com.apple.security.files.user-selected.read-write</key>
  <true/>
  <key>com.apple.security.network.client</key>
  <true/>
</dict>
</plist>
```

## Step 3: 自動更新の実装

### 3.1 src/main/auto-updater.ts を作成

```typescript
import { autoUpdater } from 'electron-updater';
import { BrowserWindow, dialog, ipcMain } from 'electron';
import log from 'electron-log';

// ロギング設定
autoUpdater.logger = log;
log.transports.file.level = 'info';

class AutoUpdater {
  private mainWindow: BrowserWindow | null = null;
  private updateDownloaded: boolean = false;

  initialize(window: BrowserWindow): void {
    this.mainWindow = window;
    this.setupUpdater();
    this.setupIPC();

    // 開発モードでは自動更新を無効化
    if (process.env.NODE_ENV === 'development') {
      log.info('Development mode: auto-update disabled');
      return;
    }

    // 起動後5秒後に更新をチェック
    setTimeout(() => {
      this.checkForUpdates();
    }, 5000);

    // 1時間ごとに更新をチェック
    setInterval(() => {
      this.checkForUpdates();
    }, 60 * 60 * 1000);
  }

  private setupUpdater(): void {
    // 更新が利用可能
    autoUpdater.on('update-available', (info) => {
      log.info('Update available:', info);
      this.sendToRenderer('update-available', {
        version: info.version,
        releaseDate: info.releaseDate,
        releaseNotes: info.releaseNotes
      });

      // ユーザーに通知
      dialog.showMessageBox(this.mainWindow!, {
        type: 'info',
        title: '更新が利用可能です',
        message: `新しいバージョン ${info.version} が利用可能です。`,
        buttons: ['ダウンロード', '後で']
      }).then((result) => {
        if (result.response === 0) {
          this.downloadUpdate();
        }
      });
    });

    // 更新がない
    autoUpdater.on('update-not-available', (info) => {
      log.info('Update not available:', info);
      this.sendToRenderer('update-not-available', info);
    });

    // ダウンロード進捗
    autoUpdater.on('download-progress', (progress) => {
      log.info('Download progress:', progress);
      this.sendToRenderer('download-progress', {
        percent: Math.round(progress.percent),
        transferred: Math.round(progress.transferred / 1024 / 1024), // MB
        total: Math.round(progress.total / 1024 / 1024) // MB
      });
    });

    // ダウンロード完了
    autoUpdater.on('update-downloaded', (info) => {
      log.info('Update downloaded:', info);
      this.updateDownloaded = true;
      this.sendToRenderer('update-downloaded', info);

      // ユーザーに再起動を促す
      dialog.showMessageBox(this.mainWindow!, {
        type: 'info',
        title: '更新をインストール',
        message: 'アップデートがダウンロードされました。再起動してインストールしますか？',
        buttons: ['再起動', '後で']
      }).then((result) => {
        if (result.response === 0) {
          this.quitAndInstall();
        }
      });
    });

    // エラー発生
    autoUpdater.on('error', (error) => {
      log.error('Update error:', error);
      this.sendToRenderer('update-error', {
        message: error.message
      });
    });
  }

  private setupIPC(): void {
    ipcMain.handle('updater-check', async () => {
      return await this.checkForUpdates();
    });

    ipcMain.handle('updater-download', async () => {
      return await this.downloadUpdate();
    });

    ipcMain.handle('updater-install', () => {
      this.quitAndInstall();
    });

    ipcMain.handle('updater-get-version', () => {
      return {
        current: require('../../../package.json').version,
        updateDownloaded: this.updateDownloaded
      };
    });
  }

  async checkForUpdates(): Promise<any> {
    try {
      return await autoUpdater.checkForUpdates();
    } catch (error) {
      log.error('Check for updates failed:', error);
      return null;
    }
  }

  async downloadUpdate(): Promise<any> {
    try {
      return await autoUpdater.downloadUpdate();
    } catch (error) {
      log.error('Download update failed:', error);
      return null;
    }
  }

  quitAndInstall(): void {
    // すべての変更を保存してから終了
    autoUpdater.quitAndInstall(false, true);
  }

  private sendToRenderer(channel: string, data: any): void {
    if (this.mainWindow && !this.mainWindow.isDestroyed()) {
      this.mainWindow.webContents.send(channel, data);
    }
  }
}

export const updater = new AutoUpdater();
```

### 3.2 main.ts を更新

```typescript
import { updater } from './auto-updater';

function createWindow(): void {
  const mainWindow = new BrowserWindow({
    // ... 既存の設定
  });

  // 自動更新を初期化
  updater.initialize(mainWindow);

  // ... 既存のコード
}
```

### 3.3 更新UIの追加（renderer.ts）

```typescript
class UpdateNotification {
  private notification: HTMLElement | null = null;

  constructor() {
    this.createNotification();
    this.setupListeners();
  }

  private createNotification(): void {
    const div = document.createElement('div');
    div.id = 'update-notification';
    div.style.cssText = `
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      background: #4CAF50;
      color: white;
      padding: 12px 20px;
      text-align: center;
      display: none;
      z-index: 9999;
      font-size: 14px;
    `;

    const message = document.createElement('span');
    message.id = 'update-message';
    div.appendChild(message);

    const downloadBtn = document.createElement('button');
    downloadBtn.id = 'update-download-btn';
    downloadBtn.textContent = 'ダウンロード';
    downloadBtn.style.cssText = `
      margin-left: 16px;
      padding: 4px 12px;
      background: white;
      color: #4CAF50;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    `;
    downloadBtn.onclick = () => this.downloadUpdate();
    div.appendChild(downloadBtn);

    const installBtn = document.createElement('button');
    installBtn.id = 'update-install-btn';
    installBtn.textContent = '再起動してインストール';
    installBtn.style.cssText = downloadBtn.style.cssText;
    installBtn.style.display = 'none';
    installBtn.onclick = () => this.installUpdate();
    div.appendChild(installBtn);

    const progressBar = document.createElement('div');
    progressBar.id = 'update-progress';
    progressBar.style.cssText = `
      margin-top: 8px;
      height: 4px;
      background: rgba(255,255,255,0.3);
      border-radius: 2px;
      overflow: hidden;
      display: none;
    `;

    const progress = document.createElement('div');
    progress.id = 'update-progress-bar';
    progress.style.cssText = `
      height: 100%;
      background: white;
      width: 0%;
      transition: width 0.3s;
    `;
    progressBar.appendChild(progress);
    div.appendChild(progressBar);

    document.body.appendChild(div);
    this.notification = div;
  }

  private setupListeners(): void {
    window.electronAPI.onUpdateAvailable?.((info: any) => {
      this.show(`新しいバージョン ${info.version} が利用可能です`);
      document.getElementById('update-download-btn')!.style.display = 'inline-block';
    });

    window.electronAPI.onDownloadProgress?.((progress: any) => {
      document.getElementById('update-message')!.textContent =
        `ダウンロード中... ${progress.percent}%`;
      document.getElementById('update-progress')!.style.display = 'block';
      document.getElementById('update-progress-bar')!.style.width = `${progress.percent}%`;
    });

    window.electronAPI.onUpdateDownloaded?.(() => {
      document.getElementById('update-message')!.textContent =
        'アップデートの準備ができました';
      document.getElementById('update-download-btn')!.style.display = 'none';
      document.getElementById('update-install-btn')!.style.display = 'inline-block';
      document.getElementById('update-progress')!.style.display = 'none';
    });
  }

  private show(message: string): void {
    if (this.notification) {
      document.getElementById('update-message')!.textContent = message;
      this.notification.style.display = 'block';
    }
  }

  private async downloadUpdate(): Promise<void> {
    await window.electronAPI.updaterDownload?.();
  }

  private installUpdate(): void {
    window.electronAPI.updaterInstall?.();
  }
}

// 初期化
new UpdateNotification();
```

## Step 4: ビルド実行

### 4.1 開発用ビルド

```bash
# TypeScriptをコンパイル
npm run build

# パッケージング（インストーラー作成なし）
npm run pack
```

### 4.2 本番用ビルド

```bash
# Windows向け
npm run dist:win

# macOS向け
npm run dist:mac

# Linux向け
npm run dist:linux

# すべてのプラットフォーム
npm run dist
```

### 4.3 ビルド成果物

```
release/
├── win-unpacked/                    # Windows展開版
├── My Notepad-1.0.0-win-x64.exe    # Windowsインストーラー
├── My Notepad-1.0.0-mac.dmg        # macOS DMG
├── My Notepad-1.0.0-mac.zip        # macOS ZIP
├── My Notepad-1.0.0.AppImage       # Linux AppImage
├── My Notepad-1.0.0.deb            # Debian パッケージ
└── My Notepad-1.0.0.rpm            # RPM パッケージ
```

## Step 5: GitHub Releasesでの配布

### 5.1 GitHub Personal Access Tokenの作成

1. GitHub → Settings → Developer settings → Personal access tokens
2. "Generate new token" をクリック
3. `repo` スコープを選択
4. トークンをコピー

### 5.2 環境変数の設定

```bash
# macOS/Linux
export GH_TOKEN="your_github_token"

# Windows
set GH_TOKEN=your_github_token
```

### 5.3 リリースの公開

```bash
# package.jsonのバージョンを更新
npm version 1.0.1

# ビルドして GitHub Releases に公開
npm run publish
```

### 5.4 GitHub Actions での自動ビルド

`.github/workflows/build.yml` を作成:

```yaml
name: Build and Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [macos-latest, ubuntu-latest, windows-latest]

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Package and Publish
        env:
          GH_TOKEN: ${{ secrets.GH_TOKEN }}
        run: npm run publish

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: ${{ matrix.os }}
          path: release/*
```

## 演習問題

### 基礎編

1. **アプリアイコンの作成**: 独自のアプリアイコンを作成してビルド
2. **バージョン管理**: package.jsonのバージョンを自動更新するスクリプト
3. **ビルドスクリプト**: クリーン→ビルド→パッケージを一括実行

### 応用編

4. **コード署名**: Windows/macOSでのコード署名実装
5. **差分更新**: 大きなアップデートの差分配信
6. **ベータチャネル**: 安定版とベータ版の配布チャネル分け

### 解答例（演習3）

```json
// package.json
{
  "scripts": {
    "prebuild": "npm run clean",
    "build": "tsc",
    "postbuild": "npm run copy-resources",
    "copy-resources": "cp -r src/renderer/*.html dist/renderer/",
    "release": "npm run build && npm run dist"
  }
}
```

```bash
# シェルスクリプト: scripts/release.sh
#!/bin/bash

# バージョンをインクリメント
npm version patch

# ビルド
npm run build

# パッケージング
npm run dist

# Git push
git push && git push --tags

echo "Release completed!"
```

## トラブルシューティング

### エラー: "Application entry file does not exist"

```bash
# TypeScriptをビルド
npm run build

# dist/main.jsが存在することを確認
ls -la dist/
```

### エラー: "Failed to sign package"

```bash
# Windowsの場合: 証明書を確認
# macOSの場合: Xcodeがインストールされているか確認
xcode-select --install
```

### macOS Notarization が失敗

```bash
# Apple ID認証情報を確認
export APPLE_ID="your@email.com"
export APPLE_ID_PASSWORD="app-specific-password"

# ビルド
npm run dist:mac
```

## まとめ

このレッスンで学んだこと:
- electron-builderによるパッケージング
- プラットフォーム別ビルド設定
- 自動更新の実装（electron-updater）
- GitHub Releasesでの配布
- CI/CDパイプライン構築

これでElectronアプリの開発からデプロイまでの一連の流れを習得しました！
