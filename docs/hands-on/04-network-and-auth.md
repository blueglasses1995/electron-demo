# ハンズオンカリキュラム Part 4: ネットワーク通信と認証

## 目標

クラウド同期機能を追加し、REST APIとの通信、簡易認証を実装します。

## 前提条件

Part 1-3を完了していること

## Step 1: 依存パッケージの追加

```bash
npm install axios
npm install --save-dev @types/axios
```

## Step 2: APIクライアントの実装

### 2.1 src/main/api-client.ts を作成

```typescript
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import { configStore } from './config-store';

class APIClient {
  private client: AxiosInstance;
  private baseURL: string = 'https://api.example.com'; // 本番環境では環境変数から取得

  constructor() {
    this.client = axios.create({
      baseURL: this.baseURL,
      timeout: 10000,
      headers: {
        'Content-Type': 'application/json'
      }
    });

    this.setupInterceptors();
  }

  private setupInterceptors(): void {
    // リクエストインターセプター
    this.client.interceptors.request.use(
      (config) => {
        // トークンを追加
        const token = this.getAuthToken();
        if (token) {
          config.headers.Authorization = `Bearer ${token}`;
        }
        return config;
      },
      (error) => {
        return Promise.reject(error);
      }
    );

    // レスポンスインターセプター
    this.client.interceptors.response.use(
      (response) => response,
      async (error) => {
        if (error.response?.status === 401) {
          // トークンが無効な場合、再認証を試みる
          await this.refreshToken();
          // リクエストを再実行
          return this.client.request(error.config);
        }
        return Promise.reject(error);
      }
    );
  }

  private getAuthToken(): string | null {
    return configStore.get('authToken' as any) || null;
  }

  private async refreshToken(): Promise<void> {
    const refreshToken = configStore.get('refreshToken' as any);
    if (!refreshToken) {
      throw new Error('No refresh token available');
    }

    try {
      const response = await axios.post(`${this.baseURL}/auth/refresh`, {
        refreshToken
      });

      configStore.set('authToken' as any, response.data.accessToken);
      configStore.set('refreshToken' as any, response.data.refreshToken);
    } catch (error) {
      // リフレッシュトークンも無効な場合、ログアウト
      this.clearAuthTokens();
      throw new Error('Token refresh failed');
    }
  }

  private clearAuthTokens(): void {
    configStore.set('authToken' as any, null);
    configStore.set('refreshToken' as any, null);
    configStore.set('user' as any, null);
  }

  // 認証API
  async login(email: string, password: string): Promise<any> {
    const response = await this.client.post('/auth/login', {
      email,
      password
    });

    const { accessToken, refreshToken, user } = response.data;

    configStore.set('authToken' as any, accessToken);
    configStore.set('refreshToken' as any, refreshToken);
    configStore.set('user' as any, user);

    return user;
  }

  async logout(): Promise<void> {
    try {
      await this.client.post('/auth/logout');
    } finally {
      this.clearAuthTokens();
    }
  }

  async getCurrentUser(): Promise<any> {
    const response = await this.client.get('/users/me');
    return response.data;
  }

  // ドキュメントAPI
  async getDocuments(): Promise<any[]> {
    const response = await this.client.get('/documents');
    return response.data;
  }

  async getDocument(id: string): Promise<any> {
    const response = await this.client.get(`/documents/${id}`);
    return response.data;
  }

  async createDocument(data: { title: string; content: string }): Promise<any> {
    const response = await this.client.post('/documents', data);
    return response.data;
  }

  async updateDocument(id: string, data: { title?: string; content?: string }): Promise<any> {
    const response = await this.client.put(`/documents/${id}`, data);
    return response.data;
  }

  async deleteDocument(id: string): Promise<void> {
    await this.client.delete(`/documents/${id}`);
  }
}

export const apiClient = new APIClient();
```

### 2.2 src/main/auth-manager.ts を作成

```typescript
import { ipcMain, BrowserWindow } from 'electron';
import { apiClient } from './api-client';
import { configStore } from './config-store';

export function setupAuthManager(): void {
  // ログイン
  ipcMain.handle('auth-login', async (event, email: string, password: string) => {
    try {
      const user = await apiClient.login(email, password);

      // すべてのウィンドウに認証状態を通知
      BrowserWindow.getAllWindows().forEach((win) => {
        win.webContents.send('auth-state-changed', { isAuthenticated: true, user });
      });

      return { success: true, user };
    } catch (error: any) {
      console.error('Login failed:', error);
      return {
        success: false,
        error: error.response?.data?.message || 'ログインに失敗しました'
      };
    }
  });

  // ログアウト
  ipcMain.handle('auth-logout', async () => {
    try {
      await apiClient.logout();

      BrowserWindow.getAllWindows().forEach((win) => {
        win.webContents.send('auth-state-changed', { isAuthenticated: false, user: null });
      });

      return { success: true };
    } catch (error) {
      console.error('Logout failed:', error);
      return { success: false };
    }
  });

  // 現在のユーザー情報を取得
  ipcMain.handle('auth-get-current-user', async () => {
    try {
      const user = await apiClient.getCurrentUser();
      return { success: true, user };
    } catch (error) {
      return { success: false, user: null };
    }
  });

  // 認証状態の確認
  ipcMain.handle('auth-check', () => {
    const token = configStore.get('authToken' as any);
    const user = configStore.get('user' as any);
    return {
      isAuthenticated: !!token,
      user
    };
  });
}
```

### 2.3 src/main/document-sync.ts を作成

```typescript
import { ipcMain } from 'electron';
import { apiClient } from './api-client';

export function setupDocumentSync(): void {
  // ドキュメント一覧を取得
  ipcMain.handle('documents-list', async () => {
    try {
      const documents = await apiClient.getDocuments();
      return { success: true, documents };
    } catch (error: any) {
      return {
        success: false,
        error: error.message
      };
    }
  });

  // ドキュメントを取得
  ipcMain.handle('documents-get', async (event, id: string) => {
    try {
      const document = await apiClient.getDocument(id);
      return { success: true, document };
    } catch (error: any) {
      return {
        success: false,
        error: error.message
      };
    }
  });

  // ドキュメントを作成
  ipcMain.handle('documents-create', async (event, data: { title: string; content: string }) => {
    try {
      const document = await apiClient.createDocument(data);
      return { success: true, document };
    } catch (error: any) {
      return {
        success: false,
        error: error.message
      };
    }
  });

  // ドキュメントを更新
  ipcMain.handle('documents-update', async (event, id: string, data: any) => {
    try {
      const document = await apiClient.updateDocument(id, data);
      return { success: true, document };
    } catch (error: any) {
      return {
        success: false,
        error: error.message
      };
    }
  });

  // ドキュメントを削除
  ipcMain.handle('documents-delete', async (event, id: string) => {
    try {
      await apiClient.deleteDocument(id);
      return { success: true };
    } catch (error: any) {
      return {
        success: false,
        error: error.message
      };
    }
  });
}
```

## Step 3: 簡易認証UIの実装

### 3.1 src/renderer/login.html を作成

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>ログイン</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
    }

    .login-container {
      background: white;
      padding: 40px;
      border-radius: 12px;
      box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
      width: 400px;
    }

    h1 {
      color: #667eea;
      margin-bottom: 30px;
      text-align: center;
    }

    .form-group {
      margin-bottom: 20px;
    }

    label {
      display: block;
      margin-bottom: 8px;
      color: #555;
      font-weight: 500;
    }

    input {
      width: 100%;
      padding: 12px;
      border: 1px solid #ddd;
      border-radius: 6px;
      font-size: 14px;
    }

    input:focus {
      outline: none;
      border-color: #667eea;
    }

    button {
      width: 100%;
      padding: 12px;
      background: #667eea;
      color: white;
      border: none;
      border-radius: 6px;
      font-size: 16px;
      font-weight: 500;
      cursor: pointer;
      transition: background 0.2s;
    }

    button:hover {
      background: #5568d3;
    }

    button:disabled {
      background: #ccc;
      cursor: not-allowed;
    }

    .error {
      color: #f44336;
      font-size: 14px;
      margin-top: 10px;
      display: none;
    }

    .error.show {
      display: block;
    }

    .demo-note {
      margin-top: 20px;
      padding: 15px;
      background: #f0f0f0;
      border-radius: 6px;
      font-size: 13px;
      color: #666;
    }
  </style>
</head>
<body>
  <div class="login-container">
    <h1>ログイン</h1>

    <form id="login-form">
      <div class="form-group">
        <label for="email">メールアドレス</label>
        <input type="email" id="email" required>
      </div>

      <div class="form-group">
        <label for="password">パスワード</label>
        <input type="password" id="password" required>
      </div>

      <button type="submit" id="login-btn">ログイン</button>

      <div class="error" id="error-message"></div>
    </form>

    <div class="demo-note">
      <strong>デモモード:</strong><br>
      このアプリはデモ用です。実際のAPIサーバーに接続する場合は、
      src/main/api-client.ts のbaseURLを変更してください。
    </div>
  </div>

  <script>
    const form = document.getElementById('login-form');
    const emailInput = document.getElementById('email');
    const passwordInput = document.getElementById('password');
    const loginBtn = document.getElementById('login-btn');
    const errorMessage = document.getElementById('error-message');

    form.addEventListener('submit', async (e) => {
      e.preventDefault();

      const email = emailInput.value;
      const password = passwordInput.value;

      loginBtn.disabled = true;
      loginBtn.textContent = 'ログイン中...';
      errorMessage.classList.remove('show');

      try {
        const result = await window.electronAPI.authLogin(email, password);

        if (result.success) {
          // メイン画面に遷移
          window.location.href = 'index.html';
        } else {
          errorMessage.textContent = result.error || 'ログインに失敗しました';
          errorMessage.classList.add('show');
        }
      } catch (error) {
        errorMessage.textContent = 'ログインに失敗しました';
        errorMessage.classList.add('show');
      } finally {
        loginBtn.disabled = false;
        loginBtn.textContent = 'ログイン';
      }
    });
  </script>
</body>
</html>
```

## Step 4: クラウド同期機能の追加

### 4.1 メインアプリの更新（renderer.ts）

```typescript
class NotepadApp {
  // ... 既存のコード

  private currentDocumentId: string | null = null;
  private isAuthenticated: boolean = false;
  private user: any = null;

  constructor() {
    // ... 既存の初期化
    this.checkAuthStatus();
  }

  private async checkAuthStatus(): Promise<void> {
    const result = await window.electronAPI.authCheck();
    this.isAuthenticated = result.isAuthenticated;
    this.user = result.user;

    if (!this.isAuthenticated) {
      // ログイン画面に遷移
      window.location.href = 'login.html';
    } else {
      this.updateUserInfo();
      this.loadCloudDocuments();
    }
  }

  private updateUserInfo(): void {
    if (this.user) {
      // ツールバーにユーザー情報を表示
      const userInfo = document.createElement('span');
      userInfo.textContent = `👤 ${this.user.name || this.user.email}`;
      userInfo.style.marginLeft = 'auto';
      userInfo.style.marginRight = '10px';
      document.querySelector('.toolbar')!.appendChild(userInfo);

      // ログアウトボタンを追加
      const logoutBtn = document.createElement('button');
      logoutBtn.textContent = 'ログアウト';
      logoutBtn.addEventListener('click', () => this.logout());
      document.querySelector('.toolbar')!.appendChild(logoutBtn);
    }
  }

  private async logout(): Promise<void> {
    await window.electronAPI.authLogout();
    window.location.href = 'login.html';
  }

  private async loadCloudDocuments(): Promise<void> {
    try {
      const result = await window.electronAPI.documentsList();
      if (result.success) {
        this.showDocumentsList(result.documents);
      }
    } catch (error) {
      console.error('Failed to load cloud documents:', error);
    }
  }

  private showDocumentsList(documents: any[]): void {
    // サイドバーにドキュメント一覧を表示（UIは省略）
    console.log('Cloud documents:', documents);
  }

  private async syncToCloud(): Promise<void> {
    if (!this.isAuthenticated) {
      alert('ログインしてください');
      return;
    }

    try {
      const data = {
        title: this.currentFileName,
        content: this.editor.value
      };

      let result;
      if (this.currentDocumentId) {
        // 既存ドキュメントを更新
        result = await window.electronAPI.documentsUpdate(this.currentDocumentId, data);
      } else {
        // 新規ドキュメントを作成
        result = await window.electronAPI.documentsCreate(data);
        if (result.success) {
          this.currentDocumentId = result.document.id;
        }
      }

      if (result.success) {
        this.showStatus('クラウドに同期しました');
      } else {
        throw new Error(result.error);
      }
    } catch (error) {
      alert('クラウド同期に失敗しました');
      console.error(error);
    }
  }
}
```

## Step 5: Preloadスクリプトの更新

```typescript
// preload.ts に追加
contextBridge.exposeInMainWorld('electronAPI', {
  // ... 既存のAPI

  // 認証API
  authLogin: (email: string, password: string) =>
    ipcRenderer.invoke('auth-login', email, password),
  authLogout: () => ipcRenderer.invoke('auth-logout'),
  authGetCurrentUser: () => ipcRenderer.invoke('auth-get-current-user'),
  authCheck: () => ipcRenderer.invoke('auth-check'),

  // ドキュメントAPI
  documentsList: () => ipcRenderer.invoke('documents-list'),
  documentsGet: (id: string) => ipcRenderer.invoke('documents-get', id),
  documentsCreate: (data: any) => ipcRenderer.invoke('documents-create', data),
  documentsUpdate: (id: string, data: any) => ipcRenderer.invoke('documents-update', id, data),
  documentsDelete: (id: string) => ipcRenderer.invoke('documents-delete', id),

  // 認証状態変更イベント
  onAuthStateChanged: (callback: (state: any) => void) => {
    ipcRenderer.on('auth-state-changed', (_, state) => callback(state));
  }
});
```

## 演習問題

### 基礎編

1. **オフライン対応**: ネットワークがない時のエラーハンドリング
2. **同期状態表示**: 同期中/同期完了のステータス表示
3. **コンフリクト検出**: ローカルとリモートの変更を比較

### 応用編

4. **自動同期**: 編集後30秒で自動的にクラウド同期
5. **バージョン履歴**: ドキュメントの変更履歴を表示
6. **リアルタイム共同編集**: WebSocketで複数ユーザーの編集を同期

## まとめ

このレッスンで学んだこと:
- REST API通信の実装
- 認証フロー（ログイン/ログアウト/トークンリフレッシュ）
- クラウド同期機能
- エラーハンドリング

次のレッスンでは、アプリのパッケージングとデプロイを学びます。
