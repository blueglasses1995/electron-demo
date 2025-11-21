# ハンズオンカリキュラム Part 2: 基本アプリケーション開発

## 目標

メモ帳アプリを作成しながら、Electronの基本機能を学びます。

## Step 1: アプリケーションメニューの追加

### 1.1 src/main/menu.ts を作成

```typescript
import { Menu, MenuItemConstructorOptions, app, BrowserWindow, dialog } from 'electron';

export function createApplicationMenu(mainWindow: BrowserWindow): void {
  const template: MenuItemConstructorOptions[] = [
    {
      label: 'ファイル',
      submenu: [
        {
          label: '新規作成',
          accelerator: 'CmdOrCtrl+N',
          click: () => {
            mainWindow.webContents.send('menu-new-file');
          }
        },
        {
          label: '開く',
          accelerator: 'CmdOrCtrl+O',
          click: async () => {
            const result = await dialog.showOpenDialog(mainWindow, {
              properties: ['openFile'],
              filters: [
                { name: 'Text Files', extensions: ['txt', 'md'] },
                { name: 'All Files', extensions: ['*'] }
              ]
            });

            if (!result.canceled && result.filePaths.length > 0) {
              mainWindow.webContents.send('menu-open-file', result.filePaths[0]);
            }
          }
        },
        {
          label: '保存',
          accelerator: 'CmdOrCtrl+S',
          click: () => {
            mainWindow.webContents.send('menu-save-file');
          }
        },
        {
          label: '名前を付けて保存',
          accelerator: 'CmdOrCtrl+Shift+S',
          click: () => {
            mainWindow.webContents.send('menu-save-file-as');
          }
        },
        { type: 'separator' },
        {
          label: '終了',
          accelerator: 'CmdOrCtrl+Q',
          click: () => {
            app.quit();
          }
        }
      ]
    },
    {
      label: '編集',
      submenu: [
        { label: '元に戻す', accelerator: 'CmdOrCtrl+Z', role: 'undo' },
        { label: 'やり直す', accelerator: 'CmdOrCtrl+Shift+Z', role: 'redo' },
        { type: 'separator' },
        { label: '切り取り', accelerator: 'CmdOrCtrl+X', role: 'cut' },
        { label: 'コピー', accelerator: 'CmdOrCtrl+C', role: 'copy' },
        { label: '貼り付け', accelerator: 'CmdOrCtrl+V', role: 'paste' },
        { label: 'すべて選択', accelerator: 'CmdOrCtrl+A', role: 'selectAll' }
      ]
    },
    {
      label: '表示',
      submenu: [
        { label: '再読み込み', accelerator: 'CmdOrCtrl+R', role: 'reload' },
        { label: '開発者ツール', accelerator: 'CmdOrCtrl+Shift+I', role: 'toggleDevTools' },
        { type: 'separator' },
        { label: '実際のサイズ', accelerator: 'CmdOrCtrl+0', role: 'resetZoom' },
        { label: '拡大', accelerator: 'CmdOrCtrl+Plus', role: 'zoomIn' },
        { label: '縮小', accelerator: 'CmdOrCtrl+-', role: 'zoomOut' },
        { type: 'separator' },
        { label: '全画面表示', accelerator: 'F11', role: 'togglefullscreen' }
      ]
    },
    {
      label: 'ヘルプ',
      submenu: [
        {
          label: 'バージョン情報',
          click: () => {
            dialog.showMessageBox(mainWindow, {
              type: 'info',
              title: 'バージョン情報',
              message: `My Electron App`,
              detail: `Version: ${app.getVersion()}\nElectron: ${process.versions.electron}\nNode: ${process.versions.node}\nChrome: ${process.versions.chrome}`
            });
          }
        }
      ]
    }
  ];

  const menu = Menu.buildFromTemplate(template);
  Menu.setApplicationMenu(menu);
}
```

### 1.2 main.tsを更新

```typescript
import { app, BrowserWindow } from 'electron';
import * as path from 'path';
import { createApplicationMenu } from './menu';

function createWindow(): void {
  const mainWindow = new BrowserWindow({
    width: 1000,
    height: 700,
    webPreferences: {
      preload: path.join(__dirname, '../preload/preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  });

  // メニューを作成
  createApplicationMenu(mainWindow);

  mainWindow.loadFile(path.join(__dirname, '../renderer/index.html'));

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

## Step 2: ファイル操作の実装

### 2.1 src/main/file-manager.ts を作成

```typescript
import { ipcMain, dialog, BrowserWindow } from 'electron';
import * as fs from 'fs/promises';
import * as path from 'path';

export function setupFileOperations(): void {
  // ファイルを開く
  ipcMain.handle('open-file', async (event, filePath?: string) => {
    try {
      let targetPath = filePath;

      if (!targetPath) {
        const result = await dialog.showOpenDialog({
          properties: ['openFile'],
          filters: [
            { name: 'Text Files', extensions: ['txt', 'md'] },
            { name: 'All Files', extensions: ['*'] }
          ]
        });

        if (result.canceled || result.filePaths.length === 0) {
          return null;
        }

        targetPath = result.filePaths[0];
      }

      const content = await fs.readFile(targetPath, 'utf-8');
      return {
        content,
        filePath: targetPath,
        fileName: path.basename(targetPath)
      };
    } catch (error) {
      console.error('Failed to open file:', error);
      throw error;
    }
  });

  // ファイルを保存
  ipcMain.handle('save-file', async (event, filePath: string, content: string) => {
    try {
      await fs.writeFile(filePath, content, 'utf-8');
      return { success: true, filePath };
    } catch (error) {
      console.error('Failed to save file:', error);
      throw error;
    }
  });

  // 名前を付けて保存
  ipcMain.handle('save-file-as', async (event, content: string, defaultName?: string) => {
    try {
      const result = await dialog.showSaveDialog({
        defaultPath: defaultName || 'untitled.txt',
        filters: [
          { name: 'Text Files', extensions: ['txt'] },
          { name: 'Markdown Files', extensions: ['md'] },
          { name: 'All Files', extensions: ['*'] }
        ]
      });

      if (result.canceled || !result.filePath) {
        return null;
      }

      await fs.writeFile(result.filePath, content, 'utf-8');
      return {
        success: true,
        filePath: result.filePath,
        fileName: path.basename(result.filePath)
      };
    } catch (error) {
      console.error('Failed to save file:', error);
      throw error;
    }
  });
}
```

### 2.2 main.tsに追加

```typescript
import { setupFileOperations } from './file-manager';

app.whenReady().then(() => {
  setupFileOperations();
  createWindow();
  // ...
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

  // メニューイベント
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
  }
});
```

## Step 4: UIの実装

### 4.1 src/renderer/index.html を更新

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">
  <title>メモ帳 - Electron</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div class="app-container">
    <div class="toolbar">
      <button id="new-btn" title="新規作成 (Ctrl+N)">📄 新規</button>
      <button id="open-btn" title="開く (Ctrl+O)">📂 開く</button>
      <button id="save-btn" title="保存 (Ctrl+S)">💾 保存</button>
      <button id="save-as-btn" title="名前を付けて保存 (Ctrl+Shift+S)">💾 名前を付けて保存</button>
      <span class="file-name" id="file-name">untitled.txt</span>
      <span class="status" id="status"></span>
    </div>

    <div class="editor-container">
      <textarea id="editor" placeholder="ここに入力してください..."></textarea>
    </div>

    <div class="statusbar">
      <span id="char-count">0文字</span>
      <span id="line-count">1行</span>
    </div>
  </div>

  <script src="renderer.js"></script>
</body>
</html>
```

### 4.2 src/renderer/styles.css を更新

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  height: 100vh;
  overflow: hidden;
}

.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

.toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  background: #f5f5f5;
  border-bottom: 1px solid #ddd;
  -webkit-app-region: drag;
}

.toolbar button {
  -webkit-app-region: no-drag;
  padding: 6px 12px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
}

.toolbar button:hover {
  background: #e8e8e8;
}

.toolbar button:active {
  background: #d8d8d8;
}

.file-name {
  -webkit-app-region: no-drag;
  margin-left: auto;
  font-weight: 500;
  color: #666;
}

.status {
  margin-left: 16px;
  color: #4CAF50;
  font-size: 12px;
}

.editor-container {
  flex: 1;
  overflow: hidden;
}

#editor {
  width: 100%;
  height: 100%;
  padding: 20px;
  border: none;
  outline: none;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 14px;
  line-height: 1.6;
  resize: none;
}

.statusbar {
  display: flex;
  justify-content: flex-end;
  gap: 20px;
  padding: 4px 16px;
  background: #f5f5f5;
  border-top: 1px solid #ddd;
  font-size: 12px;
  color: #666;
}
```

### 4.3 src/renderer/renderer.ts を実装

```typescript
declare global {
  interface Window {
    electronAPI: {
      platform: string;
      versions: { node: string; chrome: string; electron: string };
      openFile: (filePath?: string) => Promise<any>;
      saveFile: (filePath: string, content: string) => Promise<any>;
      saveFileAs: (content: string, defaultName?: string) => Promise<any>;
      onNewFile: (callback: () => void) => void;
      onOpenFile: (callback: (filePath: string) => void) => void;
      onSaveFile: (callback: () => void) => void;
      onSaveFileAs: (callback: () => void) => void;
    };
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

  constructor() {
    this.editor = document.getElementById('editor') as HTMLTextAreaElement;
    this.fileNameDisplay = document.getElementById('file-name')!;
    this.statusDisplay = document.getElementById('status')!;
    this.charCountDisplay = document.getElementById('char-count')!;
    this.lineCountDisplay = document.getElementById('line-count')!;

    this.setupEventListeners();
    this.setupMenuListeners();
    this.updateStats();
  }

  private setupEventListeners(): void {
    // ボタンイベント
    document.getElementById('new-btn')!.addEventListener('click', () => this.newFile());
    document.getElementById('open-btn')!.addEventListener('click', () => this.openFile());
    document.getElementById('save-btn')!.addEventListener('click', () => this.saveFile());
    document.getElementById('save-as-btn')!.addEventListener('click', () => this.saveFileAs());

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
    window.electronAPI.onOpenFile((filePath) => this.openFile(filePath));
    window.electronAPI.onSaveFile(() => this.saveFile());
    window.electronAPI.onSaveFileAs(() => this.saveFileAs());
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

## Step 5: ビルドとテスト

```bash
npm run build
npm start
```

## 演習

1. **文字数制限**: エディタに1000文字の制限を追加
2. **検索機能**: Ctrl+Fで文字列検索機能を追加
3. **ダークモード**: テーマ切り替え機能を追加

## まとめ

このレッスンで学んだこと:
- アプリケーションメニューの作成
- ファイルの読み書き操作
- IPC通信の実装
- UIとロジックの統合

次のレッスンでは、ローカルストレージとデータ管理を学びます。
