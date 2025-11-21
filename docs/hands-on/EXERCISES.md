# Electronアプリ開発 - 総合演習問題集

このドキュメントには、各レッスンの演習問題と解答例が含まれています。

## Part 1: 環境構築編

### 演習1-1: カスタマイズ（基礎）

**問題**: index.htmlのタイトルと見出しを「私のElectronアプリ」に変更してみましょう。

**解答**:
```html
<!-- src/renderer/index.html -->
<head>
  <title>私のElectronアプリ</title>
</head>
<body>
  <h1>私のElectronアプリへようこそ！</h1>
</body>
```

### 演習1-2: スタイル変更（基礎）

**問題**: styles.cssで背景色を青系のグラデーションに変更してみましょう。

**解答**:
```css
/* src/renderer/styles.css */
body {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  /* または */
  background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
}
```

### 演習1-3: 新しい情報追加（応用）

**問題**: preload.tsで`process.arch`（CPUアーキテクチャ）を追加してHTMLに表示してみましょう。

**解答**:
```typescript
// src/preload/preload.ts
contextBridge.exposeInMainWorld('electronAPI', {
  platform: process.platform,
  arch: process.arch, // 追加
  versions: {
    node: process.versions.node,
    chrome: process.versions.chrome,
    electron: process.versions.electron
  }
});

// src/renderer/index.html
<p>Architecture: <span id="arch"></span></p>

// src/renderer/renderer.ts
document.getElementById('arch')!.textContent = window.electronAPI.arch;
```

## Part 2: 基本アプリケーション開発編

### 演習2-1: 文字数制限（基礎）

**問題**: エディタに1000文字の制限を追加してください。

**解答**:
```typescript
// src/renderer/renderer.ts
class NotepadApp {
  private MAX_CHARS = 1000;

  private setupEventListeners(): void {
    this.editor.addEventListener('input', () => {
      const content = this.editor.value;

      if (content.length > this.MAX_CHARS) {
        this.editor.value = content.substring(0, this.MAX_CHARS);
        this.showStatus(`文字数制限: ${this.MAX_CHARS}文字まで`);
      }

      this.isModified = true;
      this.updateStats();
      this.updateTitle();
    });
  }
}
```

### 演習2-2: 検索機能（応用）

**問題**: Ctrl+Fで文字列検索機能を追加してください。

**解答**:
```typescript
// src/renderer/renderer.ts
class NotepadApp {
  private searchDialog: HTMLElement | null = null;

  constructor() {
    this.createSearchDialog();
    this.setupSearchShortcut();
  }

  private createSearchDialog(): void {
    const dialog = document.createElement('div');
    dialog.id = 'search-dialog';
    dialog.style.cssText = `
      position: fixed;
      top: 60px;
      right: 20px;
      background: white;
      padding: 15px;
      border-radius: 6px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.2);
      display: none;
      z-index: 1000;
    `;

    dialog.innerHTML = `
      <input type="text" id="search-input" placeholder="検索..."
             style="padding: 6px; border: 1px solid #ddd; border-radius: 4px; width: 200px;">
      <button id="search-next" style="margin-left: 8px; padding: 6px 12px;">次へ</button>
      <button id="search-prev" style="padding: 6px 12px;">前へ</button>
      <button id="search-close" style="padding: 6px 12px;">×</button>
    `;

    document.body.appendChild(dialog);
    this.searchDialog = dialog;

    // イベントリスナー
    document.getElementById('search-input')!.addEventListener('keyup', (e) => {
      if (e.key === 'Enter') {
        this.searchNext();
      }
    });

    document.getElementById('search-next')!.addEventListener('click', () => {
      this.searchNext();
    });

    document.getElementById('search-prev')!.addEventListener('click', () => {
      this.searchPrevious();
    });

    document.getElementById('search-close')!.addEventListener('click', () => {
      this.hideSearch();
    });
  }

  private setupSearchShortcut(): void {
    document.addEventListener('keydown', (e) => {
      if ((e.ctrlKey || e.metaKey) && e.key === 'f') {
        e.preventDefault();
        this.showSearch();
      }
    });
  }

  private showSearch(): void {
    if (this.searchDialog) {
      this.searchDialog.style.display = 'block';
      (document.getElementById('search-input') as HTMLInputElement).focus();
    }
  }

  private hideSearch(): void {
    if (this.searchDialog) {
      this.searchDialog.style.display = 'none';
    }
  }

  private searchNext(): void {
    const query = (document.getElementById('search-input') as HTMLInputElement).value;
    if (!query) return;

    const content = this.editor.value;
    const start = this.editor.selectionEnd;
    const index = content.indexOf(query, start);

    if (index !== -1) {
      this.editor.setSelectionRange(index, index + query.length);
      this.editor.focus();
    } else {
      // 最初から検索
      const firstIndex = content.indexOf(query);
      if (firstIndex !== -1) {
        this.editor.setSelectionRange(firstIndex, firstIndex + query.length);
        this.editor.focus();
      } else {
        this.showStatus('見つかりませんでした');
      }
    }
  }

  private searchPrevious(): void {
    const query = (document.getElementById('search-input') as HTMLInputElement).value;
    if (!query) return;

    const content = this.editor.value;
    const start = this.editor.selectionStart - 1;
    const index = content.lastIndexOf(query, start);

    if (index !== -1) {
      this.editor.setSelectionRange(index, index + query.length);
      this.editor.focus();
    } else {
      this.showStatus('見つかりませんでした');
    }
  }
}
```

### 演習2-3: ダークモード（応用）

**問題**: テーマ切り替え機能を追加してください。

**解答**:
```typescript
// src/renderer/renderer.ts
class NotepadApp {
  private theme: 'light' | 'dark' = 'light';

  constructor() {
    this.loadTheme();
    this.addThemeToggle();
  }

  private addThemeToggle(): void {
    const toggleBtn = document.createElement('button');
    toggleBtn.textContent = '🌓 テーマ切替';
    toggleBtn.addEventListener('click', () => this.toggleTheme());
    document.querySelector('.toolbar')!.appendChild(toggleBtn);
  }

  private toggleTheme(): void {
    this.theme = this.theme === 'light' ? 'dark' : 'light';
    this.applyTheme();
    localStorage.setItem('theme', this.theme);
  }

  private loadTheme(): void {
    const saved = localStorage.getItem('theme') as 'light' | 'dark' | null;
    if (saved) {
      this.theme = saved;
    }
    this.applyTheme();
  }

  private applyTheme(): void {
    document.body.setAttribute('data-theme', this.theme);
  }
}

// src/renderer/styles.css
:root {
  --bg-color: white;
  --text-color: #333;
  --toolbar-bg: #f5f5f5;
  --border-color: #ddd;
}

[data-theme="dark"] {
  --bg-color: #2b2b2b;
  --text-color: #e0e0e0;
  --toolbar-bg: #1e1e1e;
  --border-color: #404040;
}

body {
  background: var(--bg-color);
  color: var(--text-color);
}

.toolbar {
  background: var(--toolbar-bg);
  border-bottom: 1px solid var(--border-color);
}

#editor {
  background: var(--bg-color);
  color: var(--text-color);
}
```

## Part 3: IPC通信とストレージ編

### 演習3-1: 設定エクスポート/インポート（応用）

**問題**: 設定をJSON形式でエクスポート/インポートする機能を追加してください。

**解答**:
```typescript
// src/main/config-manager.ts
import { dialog } from 'electron';
import * as fs from 'fs/promises';

ipcMain.handle('config-export', async () => {
  try {
    const config = configStore.getAll();
    const result = await dialog.showSaveDialog({
      defaultPath: 'my-notepad-config.json',
      filters: [
        { name: 'JSON Files', extensions: ['json'] },
        { name: 'All Files', extensions: ['*'] }
      ]
    });

    if (!result.canceled && result.filePath) {
      await fs.writeFile(
        result.filePath,
        JSON.stringify(config, null, 2),
        'utf-8'
      );
      return { success: true, path: result.filePath };
    }

    return { success: false };
  } catch (error: any) {
    return { success: false, error: error.message };
  }
});

ipcMain.handle('config-import', async () => {
  try {
    const result = await dialog.showOpenDialog({
      properties: ['openFile'],
      filters: [
        { name: 'JSON Files', extensions: ['json'] },
        { name: 'All Files', extensions: ['*'] }
      ]
    });

    if (!result.canceled && result.filePaths.length > 0) {
      const content = await fs.readFile(result.filePaths[0], 'utf-8');
      const config = JSON.parse(content);

      // 設定を検証
      if (typeof config !== 'object') {
        throw new Error('Invalid config format');
      }

      configStore.setAll(config);

      // すべてのウィンドウに通知
      BrowserWindow.getAllWindows().forEach((win) => {
        win.webContents.send('config-changed-all', config);
      });

      return { success: true, config };
    }

    return { success: false };
  } catch (error: any) {
    return { success: false, error: error.message };
  }
});

// src/renderer/settings.ts
class SettingsManager {
  private setupEventListeners(): void {
    // エクスポートボタン
    const exportBtn = document.createElement('button');
    exportBtn.textContent = 'エクスポート';
    exportBtn.addEventListener('click', () => this.exportSettings());
    document.querySelector('.settings-actions')!.prepend(exportBtn);

    // インポートボタン
    const importBtn = document.createElement('button');
    importBtn.textContent = 'インポート';
    importBtn.addEventListener('click', () => this.importSettings());
    document.querySelector('.settings-actions')!.prepend(importBtn);
  }

  private async exportSettings(): Promise<void> {
    try {
      const result = await window.electronAPI.configExport();
      if (result.success) {
        this.showStatus(`設定をエクスポートしました: ${result.path}`, 'success');
      }
    } catch (error) {
      this.showStatus('エクスポートに失敗しました', 'error');
    }
  }

  private async importSettings(): Promise<void> {
    try {
      const result = await window.electronAPI.configImport();
      if (result.success) {
        this.currentConfig = result.config;
        this.populateForm();
        this.showStatus('設定をインポートしました', 'success');
      }
    } catch (error) {
      this.showStatus('インポートに失敗しました', 'error');
    }
  }
}
```

### 演習3-2: キーボードショートカット設定（上級）

**問題**: ユーザーがショートカットをカスタマイズできる機能を追加してください。

**解答**:
```typescript
// src/main/config-store.ts
export interface AppConfig {
  // ... 既存の設定
  shortcuts: {
    newFile: string;
    openFile: string;
    saveFile: string;
    saveFileAs: string;
  };
}

const schema = {
  // ... 既存のスキーマ
  shortcuts: {
    type: 'object' as const,
    default: {
      newFile: 'CmdOrCtrl+N',
      openFile: 'CmdOrCtrl+O',
      saveFile: 'CmdOrCtrl+S',
      saveFileAs: 'CmdOrCtrl+Shift+S'
    }
  }
};

// src/main/menu.ts
import { configStore } from './config-store';

export function createApplicationMenu(mainWindow: BrowserWindow): void {
  const shortcuts = configStore.get('shortcuts');

  const template: MenuItemConstructorOptions[] = [
    {
      label: 'ファイル',
      submenu: [
        {
          label: '新規作成',
          accelerator: shortcuts.newFile,
          click: () => mainWindow.webContents.send('menu-new-file')
        },
        {
          label: '開く',
          accelerator: shortcuts.openFile,
          click: async () => { /* ... */ }
        },
        // ... 残りのメニュー項目
      ]
    }
  ];

  const menu = Menu.buildFromTemplate(template);
  Menu.setApplicationMenu(menu);
}

// src/renderer/settings.html に追加
<div class="settings-section">
  <h2>キーボードショートカット</h2>

  <div class="setting-item">
    <label>新規作成</label>
    <input type="text" id="shortcut-new-file" placeholder="CmdOrCtrl+N">
  </div>

  <div class="setting-item">
    <label>開く</label>
    <input type="text" id="shortcut-open-file" placeholder="CmdOrCtrl+O">
  </div>

  <div class="setting-item">
    <label>保存</label>
    <input type="text" id="shortcut-save-file" placeholder="CmdOrCtrl+S">
  </div>

  <div class="setting-item">
    <label>名前を付けて保存</label>
    <input type="text" id="shortcut-save-file-as" placeholder="CmdOrCtrl+Shift+S">
  </div>
</div>

// src/renderer/settings.ts
class SettingsManager {
  private populateForm(): void {
    // ... 既存のコード

    // ショートカット
    const shortcuts = this.currentConfig.shortcuts || {};
    (document.getElementById('shortcut-new-file') as HTMLInputElement).value =
      shortcuts.newFile || 'CmdOrCtrl+N';
    (document.getElementById('shortcut-open-file') as HTMLInputElement).value =
      shortcuts.openFile || 'CmdOrCtrl+O';
    (document.getElementById('shortcut-save-file') as HTMLInputElement).value =
      shortcuts.saveFile || 'CmdOrCtrl+S';
    (document.getElementById('shortcut-save-file-as') as HTMLInputElement).value =
      shortcuts.saveFileAs || 'CmdOrCtrl+Shift+S';
  }

  private async saveSettings(): Promise<void> {
    try {
      const config = {
        // ... 既存の設定
        shortcuts: {
          newFile: (document.getElementById('shortcut-new-file') as HTMLInputElement).value,
          openFile: (document.getElementById('shortcut-open-file') as HTMLInputElement).value,
          saveFile: (document.getElementById('shortcut-save-file') as HTMLInputElement).value,
          saveFileAs: (document.getElementById('shortcut-save-file-as') as HTMLInputElement).value
        }
      };

      await window.electronAPI.configSetAll(config);
      this.showStatus('設定を保存しました。再起動が必要です。', 'success');
    } catch (error) {
      this.showStatus('設定の保存に失敗しました', 'error');
    }
  }
}
```

## Part 4: ネットワークと認証編

### 演習4-1: オフライン対応（基礎）

**問題**: ネットワークがない時のエラーハンドリングを実装してください。

**解答**:
```typescript
// src/renderer/renderer.ts
class NotepadApp {
  private isOnline: boolean = navigator.onLine;

  constructor() {
    this.setupNetworkListeners();
  }

  private setupNetworkListeners(): void {
    window.addEventListener('online', () => {
      this.isOnline = true;
      this.showStatus('オンラインになりました', 'success');
      this.enableCloudFeatures();
    });

    window.addEventListener('offline', () => {
      this.isOnline = false;
      this.showStatus('オフラインモードです', 'warning');
      this.disableCloudFeatures();
    });
  }

  private async syncToCloud(): Promise<void> {
    if (!this.isOnline) {
      this.showStatus('オフラインのため同期できません', 'error');
      return;
    }

    try {
      // クラウド同期処理
      await this.performCloudSync();
      this.showStatus('同期完了', 'success');
    } catch (error: any) {
      if (error.message.includes('Network')) {
        this.showStatus('ネットワークエラー: 後で再試行します', 'error');
        // 同期キューに追加
        this.addToSyncQueue();
      } else {
        this.showStatus(`同期エラー: ${error.message}`, 'error');
      }
    }
  }

  private addToSyncQueue(): void {
    const queue = JSON.parse(localStorage.getItem('syncQueue') || '[]');
    queue.push({
      fileName: this.currentFileName,
      content: this.editor.value,
      timestamp: Date.now()
    });
    localStorage.setItem('syncQueue', JSON.stringify(queue));
  }

  private async processSyncQueue(): Promise<void> {
    if (!this.isOnline) return;

    const queue = JSON.parse(localStorage.getItem('syncQueue') || '[]');
    if (queue.length === 0) return;

    this.showStatus(`${queue.length}件の変更を同期中...`);

    for (const item of queue) {
      try {
        await this.performCloudSync(item);
        // 成功したらキューから削除
        queue.shift();
        localStorage.setItem('syncQueue', JSON.stringify(queue));
      } catch (error) {
        console.error('Sync queue processing failed:', error);
        break;
      }
    }

    this.showStatus('キューの同期完了');
  }
}
```

## 総合課題

### 課題1: マークダウンプレビュー機能（中級）

エディタにマークダウンのプレビュー機能を追加してください。
- エディタとプレビューを並べて表示
- リアルタイムプレビュー更新
- HTMLエクスポート機能

### 課題2: タブ機能（上級）

複数のファイルをタブで切り替えられるようにしてください。
- タブの追加/削除
- タブのドラッグ&ドロップ並び替え
- 未保存タブの警告

### 課題3: Gitインテグレーション（上級）

簡易的なGit連携機能を追加してください。
- ファイルのコミット
- 変更履歴の表示
- 差分表示

### 課題4: プラグインシステム（エキスパート）

カスタムプラグインを読み込める仕組みを実装してください。
- プラグインAPI の設計
- プラグインのホットリロード
- サンドボックス実行

## まとめ

これらの演習問題を通じて、Electronアプリ開発の実践的なスキルを身につけることができます。
まずは基礎編から始めて、徐々に応用編、上級編にチャレンジしてください！
