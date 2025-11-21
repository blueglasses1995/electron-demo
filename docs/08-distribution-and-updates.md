# 配布・インストール・自動更新

## 1. アプリケーションのパッケージング

### 1.1 electron-builderの設定

```json
// package.json
{
  "name": "my-electron-app",
  "version": "1.0.0",
  "main": "dist/main.js",
  "scripts": {
    "start": "electron .",
    "build": "tsc",
    "pack": "electron-builder --dir",
    "dist": "electron-builder",
    "dist:win": "electron-builder --win",
    "dist:mac": "electron-builder --mac",
    "dist:linux": "electron-builder --linux"
  },
  "build": {
    "appId": "com.example.myapp",
    "productName": "My Electron App",
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
          "target": "portable"
        }
      ],
      "icon": "build/icon.ico",
      "certificateFile": "cert.pfx",
      "certificatePassword": "${env.CERTIFICATE_PASSWORD}"
    },
    "nsis": {
      "oneClick": false,
      "allowToChangeInstallationDirectory": true,
      "createDesktopShortcut": true,
      "createStartMenuShortcut": true,
      "shortcutName": "My Electron App",
      "perMachine": false,
      "installerIcon": "build/installer.ico",
      "uninstallerIcon": "build/uninstaller.ico"
    },
    "mac": {
      "target": [
        {
          "target": "dmg",
          "arch": ["x64", "arm64", "universal"]
        },
        "zip"
      ],
      "category": "public.app-category.productivity",
      "icon": "build/icon.icns",
      "hardenedRuntime": true,
      "gatekeeperAssess": false,
      "entitlements": "build/entitlements.mac.plist",
      "entitlementsInherit": "build/entitlements.mac.plist"
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
      ]
    },
    "linux": {
      "target": ["AppImage", "deb", "rpm", "snap"],
      "category": "Utility",
      "icon": "build/icons",
      "maintainer": "your-email@example.com"
    },
    "publish": {
      "provider": "github",
      "owner": "your-username",
      "repo": "your-repo"
    }
  },
  "devDependencies": {
    "electron": "^27.0.0",
    "electron-builder": "^24.0.0",
    "typescript": "^5.0.0"
  },
  "dependencies": {
    "electron-updater": "^6.1.0"
  }
}
```

## 2. プラットフォーム別パッケージング

### 2.1 Windows

```bash
# NSISインストーラー (.exe)
npm run dist:win

# ポータブル版 (.exe)
electron-builder --win portable

# Microsoft Store用 (.appx)
electron-builder --win appx
```

#### Windows用の設定

```json
// electron-builder.json
{
  "win": {
    "target": ["nsis", "portable"],
    "icon": "build/icon.ico",
    "publisherName": "Your Company Name",
    "verifyUpdateCodeSignature": true
  },
  "nsis": {
    "differentialPackage": true,
    "oneClick": false,
    "allowElevation": true,
    "runAfterFinish": true,
    "deleteAppDataOnUninstall": false
  }
}
```

#### コード署名（Windows）

```bash
# 証明書の準備
# 1. EV Code Signing証明書を取得
# 2. 証明書をPFX形式で保存

# 環境変数に証明書情報を設定
export CERTIFICATE_FILE=path/to/cert.pfx
export CERTIFICATE_PASSWORD=your_password

# ビルド時に自動的に署名される
npm run dist:win
```

### 2.2 macOS

```bash
# DMGファイル (.dmg)
npm run dist:mac

# Universal Binary（Intel + Apple Silicon）
electron-builder --mac --universal
```

#### macOS用の設定

```json
{
  "mac": {
    "target": {
      "target": "dmg",
      "arch": ["universal"]
    },
    "category": "public.app-category.productivity",
    "hardenedRuntime": true,
    "gatekeeperAssess": false,
    "entitlements": "build/entitlements.mac.plist",
    "notarize": {
      "teamId": "YOUR_TEAM_ID"
    }
  }
}
```

#### コード署名とNotarization（macOS）

```xml
<!-- entitlements.mac.plist -->
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
</dict>
</plist>
```

```bash
# Apple Developer IDでサインとNotarization
export APPLE_ID=your@email.com
export APPLE_ID_PASSWORD=app-specific-password
export APPLE_TEAM_ID=XXXXXXXXXX

npm run dist:mac
```

### 2.3 Linux

```bash
# AppImage (.AppImage)
npm run dist:linux

# Debian パッケージ (.deb)
electron-builder --linux deb

# RPM パッケージ (.rpm)
electron-builder --linux rpm

# Snap パッケージ (.snap)
electron-builder --linux snap
```

#### Linux用の設定

```json
{
  "linux": {
    "target": ["AppImage", "deb", "rpm"],
    "category": "Utility",
    "desktop": {
      "Name": "My Electron App",
      "Comment": "My awesome Electron app",
      "Categories": "Utility;Development;"
    }
  }
}
```

## 3. 自動更新の実装

### 3.1 electron-updaterの設定

```typescript
// main.ts
import { app, BrowserWindow, ipcMain } from 'electron';
import { autoUpdater } from 'electron-updater';
import log from 'electron-log';

// ロギング設定
autoUpdater.logger = log;
log.transports.file.level = 'info';

class AutoUpdateService {
  private mainWindow: BrowserWindow | null = null;

  initialize(window: BrowserWindow): void {
    this.mainWindow = window;
    this.setupAutoUpdater();
    this.setupIPC();

    // アプリ起動時に更新をチェック
    if (!app.isPackaged) {
      log.info('Dev mode: Auto-update disabled');
      return;
    }

    setTimeout(() => {
      this.checkForUpdates();
    }, 5000); // 5秒後にチェック
  }

  private setupAutoUpdater(): void {
    // 更新が利用可能
    autoUpdater.on('update-available', (info) => {
      log.info('Update available:', info);
      this.sendToRenderer('update-available', info);
    });

    // 更新がない
    autoUpdater.on('update-not-available', (info) => {
      log.info('Update not available:', info);
      this.sendToRenderer('update-not-available', info);
    });

    // ダウンロード進捗
    autoUpdater.on('download-progress', (progress) => {
      log.info('Download progress:', progress);
      this.sendToRenderer('download-progress', progress);
    });

    // ダウンロード完了
    autoUpdater.on('update-downloaded', (info) => {
      log.info('Update downloaded:', info);
      this.sendToRenderer('update-downloaded', info);

      // 自動的に再起動してインストール（オプション）
      // setTimeout(() => {
      //   autoUpdater.quitAndInstall(false, true);
      // }, 5000);
    });

    // エラー発生
    autoUpdater.on('error', (error) => {
      log.error('Update error:', error);
      this.sendToRenderer('update-error', error);
    });
  }

  private setupIPC(): void {
    ipcMain.handle('check-for-updates', async () => {
      return await this.checkForUpdates();
    });

    ipcMain.handle('download-update', async () => {
      return await autoUpdater.downloadUpdate();
    });

    ipcMain.handle('install-update', () => {
      autoUpdater.quitAndInstall(false, true);
    });

    ipcMain.handle('get-version', () => {
      return app.getVersion();
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

  private sendToRenderer(channel: string, data: any): void {
    if (this.mainWindow) {
      this.mainWindow.webContents.send(channel, data);
    }
  }
}

// 使用例
const autoUpdateService = new AutoUpdateService();

app.whenReady().then(() => {
  const mainWindow = new BrowserWindow({
    // ...
  });

  autoUpdateService.initialize(mainWindow);
});
```

### 3.2 Rendererでの更新UI

```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('updater', {
  checkForUpdates: () => ipcRenderer.invoke('check-for-updates'),
  downloadUpdate: () => ipcRenderer.invoke('download-update'),
  installUpdate: () => ipcRenderer.invoke('install-update'),
  getVersion: () => ipcRenderer.invoke('get-version'),

  onUpdateAvailable: (callback: (info: any) => void) => {
    ipcRenderer.on('update-available', (_, info) => callback(info));
  },
  onDownloadProgress: (callback: (progress: any) => void) => {
    ipcRenderer.on('download-progress', (_, progress) => callback(progress));
  },
  onUpdateDownloaded: (callback: (info: any) => void) => {
    ipcRenderer.on('update-downloaded', (_, info) => callback(info));
  },
  onUpdateError: (callback: (error: any) => void) => {
    ipcRenderer.on('update-error', (_, error) => callback(error));
  }
});

// renderer.ts
class UpdateUI {
  private updateBanner: HTMLElement | null = null;
  private progressBar: HTMLElement | null = null;

  initialize(): void {
    this.createUpdateBanner();
    this.setupListeners();

    // バージョン表示
    window.updater.getVersion().then((version: string) => {
      console.log('Current version:', version);
      document.getElementById('version')!.textContent = version;
    });
  }

  private createUpdateBanner(): void {
    const banner = document.createElement('div');
    banner.id = 'update-banner';
    banner.style.cssText = `
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      background: #4CAF50;
      color: white;
      padding: 16px;
      text-align: center;
      display: none;
      z-index: 1000;
    `;

    const message = document.createElement('span');
    message.id = 'update-message';
    banner.appendChild(message);

    const downloadBtn = document.createElement('button');
    downloadBtn.id = 'download-update-btn';
    downloadBtn.textContent = 'ダウンロード';
    downloadBtn.style.marginLeft = '16px';
    downloadBtn.onclick = () => this.downloadUpdate();
    banner.appendChild(downloadBtn);

    const installBtn = document.createElement('button');
    installBtn.id = 'install-update-btn';
    installBtn.textContent = '再起動してインストール';
    installBtn.style.marginLeft = '16px';
    installBtn.style.display = 'none';
    installBtn.onclick = () => this.installUpdate();
    banner.appendChild(installBtn);

    const progress = document.createElement('div');
    progress.id = 'update-progress';
    progress.style.cssText = `
      width: 100%;
      height: 4px;
      background: rgba(255,255,255,0.3);
      margin-top: 8px;
      display: none;
    `;

    const progressBar = document.createElement('div');
    progressBar.style.cssText = `
      height: 100%;
      background: white;
      width: 0%;
      transition: width 0.3s;
    `;
    progress.appendChild(progressBar);
    banner.appendChild(progress);

    document.body.appendChild(banner);

    this.updateBanner = banner;
    this.progressBar = progressBar;
  }

  private setupListeners(): void {
    window.updater.onUpdateAvailable((info) => {
      console.log('Update available:', info);
      this.showBanner(`新しいバージョン ${info.version} が利用可能です`);
      document.getElementById('download-update-btn')!.style.display = 'inline';
    });

    window.updater.onDownloadProgress((progress) => {
      console.log('Download progress:', progress);
      const percent = Math.round(progress.percent);

      document.getElementById('update-message')!.textContent =
        `ダウンロード中... ${percent}%`;

      document.getElementById('update-progress')!.style.display = 'block';
      this.progressBar!.style.width = `${percent}%`;
    });

    window.updater.onUpdateDownloaded((info) => {
      console.log('Update downloaded:', info);
      document.getElementById('update-message')!.textContent =
        'アップデートがダウンロードされました';

      document.getElementById('download-update-btn')!.style.display = 'none';
      document.getElementById('install-update-btn')!.style.display = 'inline';
      document.getElementById('update-progress')!.style.display = 'none';
    });

    window.updater.onUpdateError((error) => {
      console.error('Update error:', error);
      this.showBanner('アップデートエラーが発生しました', '#f44336');
    });
  }

  private showBanner(message: string, color: string = '#4CAF50'): void {
    if (this.updateBanner) {
      this.updateBanner.style.background = color;
      this.updateBanner.style.display = 'block';
      document.getElementById('update-message')!.textContent = message;
    }
  }

  private async downloadUpdate(): Promise<void> {
    document.getElementById('download-update-btn')!.setAttribute('disabled', 'true');
    await window.updater.downloadUpdate();
  }

  private installUpdate(): void {
    window.updater.installUpdate();
  }
}

const updateUI = new UpdateUI();
updateUI.initialize();

// 手動チェックボタン
document.getElementById('check-updates-btn')?.addEventListener('click', async () => {
  await window.updater.checkForUpdates();
});
```

## 4. 配布チャネル

### 4.1 GitHub Releasesでの配布

```yaml
# .github/workflows/build.yml
name: Build/release

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
      - name: Check out Git repository
        uses: actions/checkout@v3

      - name: Install Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Publish
        env:
          GH_TOKEN: ${{ secrets.GH_TOKEN }}
        run: npm run dist

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: ${{ matrix.os }}
          path: release/*
```

### 4.2 更新サーバーのセットアップ

```typescript
// update-server.ts (例: Express)
import express from 'express';
import * as fs from 'fs';
import * as path from 'path';

const app = express();
const PORT = 3000;

// 最新バージョン情報
app.get('/update/:platform/:version', (req, res) => {
  const { platform, version } = req.params;

  // latest.ymlを返す
  const updateFile = path.join(__dirname, 'releases', platform, 'latest.yml');

  if (fs.existsSync(updateFile)) {
    res.sendFile(updateFile);
  } else {
    res.status(404).send('No update available');
  }
});

// ダウンロード
app.get('/download/:platform/:filename', (req, res) => {
  const { platform, filename } = req.params;
  const filePath = path.join(__dirname, 'releases', platform, filename);

  if (fs.existsSync(filePath)) {
    res.download(filePath);
  } else {
    res.status(404).send('File not found');
  }
});

app.listen(PORT, () => {
  console.log(`Update server running on port ${PORT}`);
});
```

## 5. インストール時の挙動

### 5.1 初回起動時の処理

```typescript
// main.ts
import Store from 'electron-store';

const store = new Store();

app.on('ready', () => {
  const isFirstRun = !store.has('firstRunCompleted');

  if (isFirstRun) {
    // 初回起動時の処理
    showWelcomeWindow();
    setupDefaultSettings();
    store.set('firstRunCompleted', true);
  } else {
    // 通常起動
    createMainWindow();
  }

  // バージョンチェック
  const lastVersion = store.get('lastVersion') as string;
  const currentVersion = app.getVersion();

  if (lastVersion !== currentVersion) {
    // アップデート後の処理
    onVersionUpdate(lastVersion, currentVersion);
    store.set('lastVersion', currentVersion);
  }
});

function showWelcomeWindow(): void {
  const welcomeWindow = new BrowserWindow({
    width: 600,
    height: 400,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true
    }
  });

  welcomeWindow.loadFile('welcome.html');
}

function setupDefaultSettings(): void {
  store.set('settings', {
    theme: 'light',
    language: 'ja',
    autoStart: false
  });
}

function onVersionUpdate(oldVersion: string, newVersion: string): void {
  console.log(`Updated from ${oldVersion} to ${newVersion}`);

  // マイグレーション処理
  migrateData(oldVersion, newVersion);
}
```

### 5.2 アンインストール時のクリーンアップ

```typescript
// main.ts
app.on('will-quit', async (event) => {
  event.preventDefault();

  // クリーンアップ処理
  await cleanupTempFiles();
  await closeAllConnections();

  // ユーザーデータを残すか確認
  const shouldKeepData = store.get('keepDataOnUninstall', true);

  if (!shouldKeepData) {
    await cleanupUserData();
  }

  app.exit(0);
});

async function cleanupTempFiles(): Promise<void> {
  const tempDir = app.getPath('temp');
  // 一時ファイルを削除
}

async function cleanupUserData(): Promise<void> {
  const userDataPath = app.getPath('userData');
  // ユーザーデータを削除
}
```

## 6. まとめ

- **electron-builder**: クロスプラットフォームパッケージング
- **コード署名**: Windows（証明書）、macOS（Apple Developer ID）
- **自動更新**: electron-updater を使用
- **配布**: GitHub Releases、独自サーバー、ストア
- **インストール**: 初回起動処理、アップデート処理

次はハンズオンカリキュラムで実際に開発していきます。
