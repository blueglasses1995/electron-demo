# ハンズオンカリキュラム Part 3: IPC通信とローカルストレージ

## 目標

設定機能を追加し、electron-storeによる永続化とIPCパターンを学びます。

## 前提条件

Part 1-2を完了していること

## Step 1: 依存パッケージの追加

### 1.1 electron-storeのインストール

```bash
npm install electron-store
npm install --save-dev @types/electron-store
```

### 1.2 package.jsonの確認

```json
{
  "dependencies": {
    "electron-store": "^8.1.0"
  }
}
```

## Step 2: 設定ストアの実装

### 2.1 src/main/config-store.ts を作成

```typescript
import Store from 'electron-store';

export interface AppConfig {
  theme: 'light' | 'dark';
  fontSize: number;
  autoSave: boolean;
  autoSaveInterval: number; // 秒
  recentFiles: string[];
  windowBounds?: {
    x: number;
    y: number;
    width: number;
    height: number;
  };
}

const schema = {
  theme: {
    type: 'string' as const,
    enum: ['light', 'dark'],
    default: 'light'
  },
  fontSize: {
    type: 'number' as const,
    minimum: 10,
    maximum: 30,
    default: 14
  },
  autoSave: {
    type: 'boolean' as const,
    default: false
  },
  autoSaveInterval: {
    type: 'number' as const,
    minimum: 10,
    maximum: 300,
    default: 30
  },
  recentFiles: {
    type: 'array' as const,
    default: []
  }
};

class ConfigStore {
  private store: Store<AppConfig>;

  constructor() {
    this.store = new Store<AppConfig>({
      name: 'config',
      schema: schema as any,
      clearInvalidConfig: true
    });
  }

  get<K extends keyof AppConfig>(key: K): AppConfig[K] {
    return this.store.get(key);
  }

  set<K extends keyof AppConfig>(key: K, value: AppConfig[K]): void {
    this.store.set(key, value);
  }

  getAll(): AppConfig {
    return this.store.store;
  }

  setAll(config: Partial<AppConfig>): void {
    Object.entries(config).forEach(([key, value]) => {
      this.store.set(key as keyof AppConfig, value as any);
    });
  }

  addRecentFile(filePath: string): void {
    const recentFiles = this.get('recentFiles');
    const updated = [
      filePath,
      ...recentFiles.filter(f => f !== filePath)
    ].slice(0, 10); // 最大10件

    this.set('recentFiles', updated);
  }

  clearRecentFiles(): void {
    this.set('recentFiles', []);
  }

  reset(): void {
    this.store.clear();
  }
}

export const configStore = new ConfigStore();
```

### 2.2 src/main/config-manager.ts を作成

```typescript
import { ipcMain, BrowserWindow } from 'electron';
import { configStore, AppConfig } from './config-store';

export function setupConfigManager(): void {
  // 設定の取得
  ipcMain.handle('config-get', (event, key?: keyof AppConfig) => {
    if (key) {
      return configStore.get(key);
    }
    return configStore.getAll();
  });

  // 設定の保存
  ipcMain.handle('config-set', (event, key: keyof AppConfig, value: any) => {
    configStore.set(key, value);

    // すべてのウィンドウに変更を通知
    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.send('config-changed', key, value);
    });

    return { success: true };
  });

  // 複数設定の一括保存
  ipcMain.handle('config-set-all', (event, config: Partial<AppConfig>) => {
    configStore.setAll(config);

    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.send('config-changed-all', config);
    });

    return { success: true };
  });

  // 最近使ったファイルの追加
  ipcMain.handle('config-add-recent-file', (event, filePath: string) => {
    configStore.addRecentFile(filePath);
    return configStore.get('recentFiles');
  });

  // 設定のリセット
  ipcMain.handle('config-reset', () => {
    configStore.reset();
    return configStore.getAll();
  });
}

// ウィンドウの位置とサイズを保存
export function saveWindowBounds(window: BrowserWindow): void {
  const bounds = window.getBounds();
  configStore.set('windowBounds', bounds);
}

// ウィンドウの位置とサイズを復元
export function restoreWindowBounds(window: BrowserWindow): void {
  const bounds = configStore.get('windowBounds');
  if (bounds) {
    window.setBounds(bounds);
  }
}
```

### 2.3 main.ts を更新

```typescript
import { app, BrowserWindow } from 'electron';
import * as path from 'path';
import { createApplicationMenu } from './menu';
import { setupFileOperations } from './file-manager';
import { setupConfigManager, saveWindowBounds, restoreWindowBounds } from './config-manager';

let mainWindow: BrowserWindow | null = null;

function createWindow(): void {
  mainWindow = new BrowserWindow({
    width: 1000,
    height: 700,
    webPreferences: {
      preload: path.join(__dirname, '../preload/preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  });

  // ウィンドウの位置とサイズを復元
  restoreWindowBounds(mainWindow);

  createApplicationMenu(mainWindow);
  mainWindow.loadFile(path.join(__dirname, '../renderer/index.html'));

  if (process.env.NODE_ENV === 'development') {
    mainWindow.webContents.openDevTools();
  }

  // ウィンドウが閉じられる前に位置とサイズを保存
  mainWindow.on('close', () => {
    if (mainWindow) {
      saveWindowBounds(mainWindow);
    }
  });

  mainWindow.on('closed', () => {
    mainWindow = null;
  });
}

app.whenReady().then(() => {
  setupFileOperations();
  setupConfigManager();
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

## Step 3: Preloadスクリプトの拡張

### 3.1 src/preload/preload.ts を更新

```typescript
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  // バージョン情報
  platform: process.platform,
  versions: {
    node: process.versions.node,
    chrome: process.versions.chrome,
    electron: process.versions.electron
  },

  // ファイル操作
  openFile: (filePath?: string) => ipcRenderer.invoke('open-file', filePath),
  saveFile: (filePath: string, content: string) =>
    ipcRenderer.invoke('save-file', filePath, content),
  saveFileAs: (content: string, defaultName?: string) =>
    ipcRenderer.invoke('save-file-as', content, defaultName),

  // 設定管理
  configGet: (key?: string) => ipcRenderer.invoke('config-get', key),
  configSet: (key: string, value: any) => ipcRenderer.invoke('config-set', key, value),
  configSetAll: (config: any) => ipcRenderer.invoke('config-set-all', config),
  configAddRecentFile: (filePath: string) =>
    ipcRenderer.invoke('config-add-recent-file', filePath),
  configReset: () => ipcRenderer.invoke('config-reset'),

  // イベントリスナー
  onNewFile: (callback: () => void) => {
    ipcRenderer.on('menu-new-file', () => callback());
  },
  onOpenFile: (callback: (filePath: string) => void) => {
    ipcRenderer.on('menu-open-file', (_, filePath) => callback(filePath));
  },
  onSaveFile: (callback: () => void) => {
    ipcRenderer.on('menu-save-file', () => callback());
  },
  onSaveFileAs: (callback: () => void) => {
    ipcRenderer.on('menu-save-file-as', () => callback());
  },
  onConfigChanged: (callback: (key: string, value: any) => void) => {
    ipcRenderer.on('config-changed', (_, key, value) => callback(key, value));
  },
  onConfigChangedAll: (callback: (config: any) => void) => {
    ipcRenderer.on('config-changed-all', (_, config) => callback(config));
  }
});
```

## Step 4: 設定画面の実装

### 4.1 src/renderer/settings.html を作成

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">
  <title>設定</title>
  <link rel="stylesheet" href="settings-styles.css">
</head>
<body>
  <div class="settings-container">
    <h1>設定</h1>

    <div class="settings-section">
      <h2>外観</h2>

      <div class="setting-item">
        <label for="theme">テーマ</label>
        <select id="theme">
          <option value="light">ライト</option>
          <option value="dark">ダーク</option>
        </select>
      </div>

      <div class="setting-item">
        <label for="font-size">フォントサイズ</label>
        <input type="range" id="font-size" min="10" max="30" step="1">
        <span id="font-size-value">14px</span>
      </div>
    </div>

    <div class="settings-section">
      <h2>自動保存</h2>

      <div class="setting-item">
        <label for="auto-save">
          <input type="checkbox" id="auto-save">
          自動保存を有効にする
        </label>
      </div>

      <div class="setting-item">
        <label for="auto-save-interval">自動保存間隔（秒）</label>
        <input type="number" id="auto-save-interval" min="10" max="300" step="10">
      </div>
    </div>

    <div class="settings-section">
      <h2>最近使ったファイル</h2>
      <div id="recent-files-list"></div>
      <button id="clear-recent-files">履歴をクリア</button>
    </div>

    <div class="settings-actions">
      <button id="reset-settings" class="danger">設定をリセット</button>
      <button id="save-settings" class="primary">保存</button>
      <button id="cancel-settings">キャンセル</button>
    </div>

    <div id="status-message"></div>
  </div>

  <script src="settings.js"></script>
</body>
</html>
```

### 4.2 src/renderer/settings-styles.css を作成

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: #f5f5f5;
  padding: 20px;
}

.settings-container {
  max-width: 800px;
  margin: 0 auto;
  background: white;
  padding: 30px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

h1 {
  color: #333;
  margin-bottom: 30px;
  font-size: 28px;
}

.settings-section {
  margin-bottom: 30px;
  padding-bottom: 20px;
  border-bottom: 1px solid #eee;
}

.settings-section:last-of-type {
  border-bottom: none;
}

h2 {
  color: #555;
  font-size: 18px;
  margin-bottom: 15px;
}

.setting-item {
  margin-bottom: 15px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.setting-item label {
  flex: 0 0 200px;
  color: #666;
}

.setting-item input[type="checkbox"] {
  margin-right: 8px;
}

.setting-item select,
.setting-item input[type="number"] {
  padding: 6px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

.setting-item input[type="range"] {
  flex: 1;
}

#font-size-value {
  flex: 0 0 50px;
  text-align: right;
  color: #666;
}

#recent-files-list {
  background: #f9f9f9;
  padding: 15px;
  border-radius: 4px;
  max-height: 200px;
  overflow-y: auto;
  margin-bottom: 10px;
}

.recent-file-item {
  padding: 8px;
  margin-bottom: 5px;
  background: white;
  border-radius: 4px;
  font-size: 13px;
  color: #666;
  cursor: pointer;
  transition: background 0.2s;
}

.recent-file-item:hover {
  background: #e8e8e8;
}

.settings-actions {
  display: flex;
  gap: 10px;
  margin-top: 30px;
  justify-content: flex-end;
}

button {
  padding: 10px 20px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
}

button:hover {
  background: #f5f5f5;
}

button.primary {
  background: #4CAF50;
  color: white;
  border-color: #4CAF50;
}

button.primary:hover {
  background: #45a049;
}

button.danger {
  background: #f44336;
  color: white;
  border-color: #f44336;
}

button.danger:hover {
  background: #da190b;
}

#status-message {
  margin-top: 15px;
  padding: 10px;
  border-radius: 4px;
  text-align: center;
  display: none;
}

#status-message.success {
  background: #d4edda;
  color: #155724;
  display: block;
}

#status-message.error {
  background: #f8d7da;
  color: #721c24;
  display: block;
}
```

### 4.3 src/renderer/settings.ts を作成

```typescript
declare global {
  interface Window {
    electronAPI: any;
  }
}

class SettingsManager {
  private currentConfig: any = {};

  constructor() {
    this.loadSettings();
    this.setupEventListeners();
  }

  private async loadSettings(): Promise<void> {
    try {
      this.currentConfig = await window.electronAPI.configGet();
      this.populateForm();
      await this.loadRecentFiles();
    } catch (error) {
      console.error('Failed to load settings:', error);
      this.showStatus('設定の読み込みに失敗しました', 'error');
    }
  }

  private populateForm(): void {
    // テーマ
    const themeSelect = document.getElementById('theme') as HTMLSelectElement;
    themeSelect.value = this.currentConfig.theme || 'light';

    // フォントサイズ
    const fontSizeInput = document.getElementById('font-size') as HTMLInputElement;
    const fontSizeValue = document.getElementById('font-size-value')!;
    fontSizeInput.value = String(this.currentConfig.fontSize || 14);
    fontSizeValue.textContent = `${fontSizeInput.value}px`;

    // 自動保存
    const autoSaveCheckbox = document.getElementById('auto-save') as HTMLInputElement;
    autoSaveCheckbox.checked = this.currentConfig.autoSave || false;

    // 自動保存間隔
    const autoSaveInterval = document.getElementById('auto-save-interval') as HTMLInputElement;
    autoSaveInterval.value = String(this.currentConfig.autoSaveInterval || 30);
    autoSaveInterval.disabled = !autoSaveCheckbox.checked;
  }

  private async loadRecentFiles(): Promise<void> {
    const recentFiles = this.currentConfig.recentFiles || [];
    const container = document.getElementById('recent-files-list')!;

    if (recentFiles.length === 0) {
      container.innerHTML = '<p style="color: #999;">最近使ったファイルはありません</p>';
      return;
    }

    container.innerHTML = recentFiles
      .map((file: string) => `
        <div class="recent-file-item" data-path="${file}">
          ${this.truncatePath(file)}
        </div>
      `)
      .join('');

    // ファイルクリックで開く
    container.querySelectorAll('.recent-file-item').forEach((item) => {
      item.addEventListener('click', () => {
        const filePath = item.getAttribute('data-path');
        if (filePath) {
          window.electronAPI.openFile(filePath);
        }
      });
    });
  }

  private truncatePath(path: string): string {
    const maxLength = 60;
    if (path.length <= maxLength) return path;

    const parts = path.split(/[\\/]/);
    const filename = parts[parts.length - 1];
    const dir = parts.slice(0, -1).join('/');

    if (dir.length + filename.length + 4 > maxLength) {
      return `...${path.slice(-(maxLength - 3))}`;
    }

    return `${dir.slice(0, maxLength - filename.length - 4)}.../${filename}`;
  }

  private setupEventListeners(): void {
    // フォントサイズスライダー
    const fontSizeInput = document.getElementById('font-size') as HTMLInputElement;
    const fontSizeValue = document.getElementById('font-size-value')!;
    fontSizeInput.addEventListener('input', () => {
      fontSizeValue.textContent = `${fontSizeInput.value}px`;
    });

    // 自動保存チェックボックス
    const autoSaveCheckbox = document.getElementById('auto-save') as HTMLInputElement;
    const autoSaveInterval = document.getElementById('auto-save-interval') as HTMLInputElement;
    autoSaveCheckbox.addEventListener('change', () => {
      autoSaveInterval.disabled = !autoSaveCheckbox.checked;
    });

    // 保存ボタン
    document.getElementById('save-settings')!.addEventListener('click', () => {
      this.saveSettings();
    });

    // キャンセルボタン
    document.getElementById('cancel-settings')!.addEventListener('click', () => {
      window.close();
    });

    // リセットボタン
    document.getElementById('reset-settings')!.addEventListener('click', () => {
      if (confirm('すべての設定をリセットしますか？')) {
        this.resetSettings();
      }
    });

    // 履歴クリアボタン
    document.getElementById('clear-recent-files')!.addEventListener('click', () => {
      if (confirm('最近使ったファイルの履歴をクリアしますか？')) {
        this.clearRecentFiles();
      }
    });
  }

  private async saveSettings(): Promise<void> {
    try {
      const config = {
        theme: (document.getElementById('theme') as HTMLSelectElement).value,
        fontSize: parseInt((document.getElementById('font-size') as HTMLInputElement).value),
        autoSave: (document.getElementById('auto-save') as HTMLInputElement).checked,
        autoSaveInterval: parseInt((document.getElementById('auto-save-interval') as HTMLInputElement).value)
      };

      await window.electronAPI.configSetAll(config);
      this.showStatus('設定を保存しました', 'success');

      // 2秒後にウィンドウを閉じる
      setTimeout(() => {
        window.close();
      }, 2000);
    } catch (error) {
      console.error('Failed to save settings:', error);
      this.showStatus('設定の保存に失敗しました', 'error');
    }
  }

  private async resetSettings(): Promise<void> {
    try {
      this.currentConfig = await window.electronAPI.configReset();
      this.populateForm();
      await this.loadRecentFiles();
      this.showStatus('設定をリセットしました', 'success');
    } catch (error) {
      console.error('Failed to reset settings:', error);
      this.showStatus('設定のリセットに失敗しました', 'error');
    }
  }

  private async clearRecentFiles(): Promise<void> {
    try {
      await window.electronAPI.configSet('recentFiles', []);
      await this.loadRecentFiles();
      this.showStatus('履歴をクリアしました', 'success');
    } catch (error) {
      console.error('Failed to clear recent files:', error);
      this.showStatus('履歴のクリアに失敗しました', 'error');
    }
  }

  private showStatus(message: string, type: 'success' | 'error'): void {
    const statusMessage = document.getElementById('status-message')!;
    statusMessage.textContent = message;
    statusMessage.className = type;

    setTimeout(() => {
      statusMessage.style.display = 'none';
    }, 3000);
  }
}

// 初期化
new SettingsManager();
```

## Step 5: メインアプリからの設定適用

### 5.1 src/renderer/renderer.ts を更新

```typescript
declare global {
  interface Window {
    electronAPI: any;
  }
}

class NotepadApp {
  private editor: HTMLTextAreaElement;
  private fileNameDisplay: HTMLElement;
  private statusDisplay: HTMLElement;
  private charCountDisplay: HTMLElement;
  private lineCountDisplay: HTMLElement;

  private currentFilePath: string | null = null;
  private currentFileName: string = 'untitled.txt';
  private isModified: boolean = false;

  private config: any = {};
  private autoSaveTimer: NodeJS.Timeout | null = null;

  constructor() {
    this.editor = document.getElementById('editor') as HTMLTextAreaElement;
    this.fileNameDisplay = document.getElementById('file-name')!;
    this.statusDisplay = document.getElementById('status')!;
    this.charCountDisplay = document.getElementById('char-count')!;
    this.lineCountDisplay = document.getElementById('line-count')!;

    this.loadConfig();
    this.setupEventListeners();
    this.setupMenuListeners();
    this.setupConfigListeners();
    this.updateStats();
  }

  private async loadConfig(): Promise<void> {
    try {
      this.config = await window.electronAPI.configGet();
      this.applyConfig();
    } catch (error) {
      console.error('Failed to load config:', error);
    }
  }

  private applyConfig(): void {
    // テーマ適用
    document.body.setAttribute('data-theme', this.config.theme || 'light');

    // フォントサイズ適用
    this.editor.style.fontSize = `${this.config.fontSize || 14}px`;

    // 自動保存の設定
    this.setupAutoSave();
  }

  private setupAutoSave(): void {
    // 既存のタイマーをクリア
    if (this.autoSaveTimer) {
      clearInterval(this.autoSaveTimer);
      this.autoSaveTimer = null;
    }

    if (this.config.autoSave && this.currentFilePath) {
      const interval = (this.config.autoSaveInterval || 30) * 1000;
      this.autoSaveTimer = setInterval(() => {
        if (this.isModified && this.currentFilePath) {
          this.saveFile();
        }
      }, interval);
    }
  }

  private setupConfigListeners(): void {
    window.electronAPI.onConfigChangedAll((config: any) => {
      this.config = config;
      this.applyConfig();
    });
  }

  private setupEventListeners(): void {
    // ボタンイベント
    document.getElementById('new-btn')!.addEventListener('click', () => this.newFile());
    document.getElementById('open-btn')!.addEventListener('click', () => this.openFile());
    document.getElementById('save-btn')!.addEventListener('click', () => this.saveFile());
    document.getElementById('save-as-btn')!.addEventListener('click', () => this.saveFileAs());

    // 設定ボタンを追加
    const settingsBtn = document.createElement('button');
    settingsBtn.id = 'settings-btn';
    settingsBtn.title = '設定';
    settingsBtn.textContent = '⚙️ 設定';
    settingsBtn.addEventListener('click', () => this.openSettings());
    document.querySelector('.toolbar')!.appendChild(settingsBtn);

    // エディタの変更を監視
    this.editor.addEventListener('input', () => {
      this.isModified = true;
      this.updateStats();
      this.updateTitle();
    });

    // キーボードショートカット
    this.editor.addEventListener('keydown', (e) => {
      if (e.ctrlKey || e.metaKey) {
        if (e.key === 's') {
          e.preventDefault();
          if (e.shiftKey) {
            this.saveFileAs();
          } else {
            this.saveFile();
          }
        }
      }
    });
  }

  private setupMenuListeners(): void {
    window.electronAPI.onNewFile(() => this.newFile());
    window.electronAPI.onOpenFile((filePath: string) => this.openFile(filePath));
    window.electronAPI.onSaveFile(() => this.saveFile());
    window.electronAPI.onSaveFileAs(() => this.saveFileAs());
  }

  private openSettings(): void {
    // BrowserWindowを開く処理はMainプロセスで実装
    window.electronAPI.openSettings?.();
  }

  private newFile(): void {
    if (this.isModified) {
      if (!confirm('保存されていない変更があります。新しいファイルを作成しますか?')) {
        return;
      }
    }

    this.editor.value = '';
    this.currentFilePath = null;
    this.currentFileName = 'untitled.txt';
    this.isModified = false;
    this.updateTitle();
    this.updateStats();
    this.setupAutoSave(); // 自動保存をリセット
  }

  private async openFile(filePath?: string): Promise<void> {
    try {
      const result = await window.electronAPI.openFile(filePath);

      if (result) {
        this.editor.value = result.content;
        this.currentFilePath = result.filePath;
        this.currentFileName = result.fileName;
        this.isModified = false;
        this.updateTitle();
        this.updateStats();
        this.showStatus('ファイルを開きました');

        // 最近使ったファイルに追加
        await window.electronAPI.configAddRecentFile(result.filePath);

        // 自動保存を設定
        this.setupAutoSave();
      }
    } catch (error) {
      alert('ファイルを開けませんでした');
      console.error(error);
    }
  }

  private async saveFile(): Promise<void> {
    if (!this.currentFilePath) {
      await this.saveFileAs();
      return;
    }

    try {
      await window.electronAPI.saveFile(this.currentFilePath, this.editor.value);
      this.isModified = false;
      this.updateTitle();
      this.showStatus('保存しました');
    } catch (error) {
      alert('ファイルを保存できませんでした');
      console.error(error);
    }
  }

  private async saveFileAs(): Promise<void> {
    try {
      const result = await window.electronAPI.saveFileAs(
        this.editor.value,
        this.currentFileName
      );

      if (result) {
        this.currentFilePath = result.filePath;
        this.currentFileName = result.fileName;
        this.isModified = false;
        this.updateTitle();
        this.showStatus('保存しました');

        // 最近使ったファイルに追加
        await window.electronAPI.configAddRecentFile(result.filePath);

        // 自動保存を設定
        this.setupAutoSave();
      }
    } catch (error) {
      alert('ファイルを保存できませんでした');
      console.error(error);
    }
  }

  private updateTitle(): void {
    const modified = this.isModified ? '*' : '';
    this.fileNameDisplay.textContent = `${modified}${this.currentFileName}`;
    document.title = `${modified}${this.currentFileName} - メモ帳`;
  }

  private updateStats(): void {
    const content = this.editor.value;
    const charCount = content.length;
    const lineCount = content.split('\n').length;

    this.charCountDisplay.textContent = `${charCount}文字`;
    this.lineCountDisplay.textContent = `${lineCount}行`;
  }

  private showStatus(message: string): void {
    this.statusDisplay.textContent = message;
    setTimeout(() => {
      this.statusDisplay.textContent = '';
    }, 3000);
  }
}

// アプリを初期化
new NotepadApp();
```

### 5.2 src/renderer/styles.css にテーマ対応を追加

```css
/* ライトテーマ */
body[data-theme="light"] {
  --bg-color: white;
  --text-color: #333;
  --toolbar-bg: #f5f5f5;
  --border-color: #ddd;
}

/* ダークテーマ */
body[data-theme="dark"] {
  --bg-color: #2b2b2b;
  --text-color: #e0e0e0;
  --toolbar-bg: #1e1e1e;
  --border-color: #404040;
}

body {
  background: var(--bg-color);
  color: var(--text-color);
  transition: background-color 0.3s, color 0.3s;
}

.toolbar {
  background: var(--toolbar-bg);
  border-bottom: 1px solid var(--border-color);
}

#editor {
  background: var(--bg-color);
  color: var(--text-color);
}

.statusbar {
  background: var(--toolbar-bg);
  border-top: 1px solid var(--border-color);
  color: var(--text-color);
}
```

## Step 6: ビルドとテスト

```bash
npm run build
npm start
```

## 演習問題

### 基礎編

1. **エクスポート機能**: 設定をJSON形式でエクスポート/インポートする機能を追加
2. **キーボードショートカット設定**: ユーザーがショートカットをカスタマイズできる機能
3. **ウィンドウの透明度**: 設定画面でウィンドウの透明度を調整できる機能

### 応用編

4. **プラグインシステム**: 簡単なプラグインを読み込める仕組み
5. **マルチウィンドウ同期**: 複数ウィンドウ間で設定をリアルタイム同期
6. **設定の検証**: 不正な設定値を入力したときのエラーハンドリング

### 解答例（演習1）

```typescript
// 設定のエクスポート
ipcMain.handle('config-export', async () => {
  const config = configStore.getAll();
  const result = await dialog.showSaveDialog({
    defaultPath: 'config.json',
    filters: [{ name: 'JSON', extensions: ['json'] }]
  });

  if (!result.canceled && result.filePath) {
    await fs.writeFile(
      result.filePath,
      JSON.stringify(config, null, 2),
      'utf-8'
    );
    return { success: true };
  }

  return { success: false };
});

// 設定のインポート
ipcMain.handle('config-import', async () => {
  const result = await dialog.showOpenDialog({
    properties: ['openFile'],
    filters: [{ name: 'JSON', extensions: ['json'] }]
  });

  if (!result.canceled && result.filePaths.length > 0) {
    const content = await fs.readFile(result.filePaths[0], 'utf-8');
    const config = JSON.parse(content);
    configStore.setAll(config);
    return { success: true, config };
  }

  return { success: false };
});
```

## まとめ

このレッスンで学んだこと:
- electron-storeによる設定の永続化
- IPC通信のパターン（invoke/handle、イベント通知）
- 設定画面の実装
- テーマシステムの実装
- 自動保存機能

次のレッスンでは、ネットワーク通信と認証を学びます。
