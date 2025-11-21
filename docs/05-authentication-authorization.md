# 認証認可とアカウント管理

## 1. Electronアプリでの認証の概要

Electronアプリはデスクトップアプリケーションのため、Web APIサーバーと連携して認証を行います。

### 1.1 認証フロー

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Electron   │         │  Auth Server │         │  API Server  │
│     App      │         │              │         │              │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │  1. Login Request      │                        │
       │───────────────────────>│                        │
       │                        │                        │
       │  2. Access Token +     │                        │
       │     Refresh Token      │                        │
       │<───────────────────────│                        │
       │                        │                        │
       │  3. API Request        │                        │
       │     (with token)       │                        │
       │────────────────────────┼───────────────────────>│
       │                        │                        │
       │  4. Response           │                        │
       │<────────────────────────────────────────────────│
       │                        │                        │
       │  5. Token Expired      │                        │
       │<────────────────────────────────────────────────│
       │                        │                        │
       │  6. Refresh Token      │                        │
       │───────────────────────>│                        │
       │                        │                        │
       │  7. New Access Token   │                        │
       │<───────────────────────│                        │
```

## 2. OAuth 2.0 認証

### 2.1 OAuth 2.0 フロー実装

```typescript
// main.ts
import { BrowserWindow, ipcMain } from 'electron';
import * as crypto from 'crypto';

class OAuth2Service {
  private authWindow: BrowserWindow | null = null;

  async authenticate(
    authorizationUrl: string,
    tokenUrl: string,
    clientId: string,
    clientSecret: string,
    redirectUri: string
  ): Promise<{ accessToken: string; refreshToken: string }> {
    // PKCE用のコードチャレンジを生成
    const codeVerifier = this.generateCodeVerifier();
    const codeChallenge = this.generateCodeChallenge(codeVerifier);

    // 認証URLを構築
    const authUrl = new URL(authorizationUrl);
    authUrl.searchParams.append('client_id', clientId);
    authUrl.searchParams.append('redirect_uri', redirectUri);
    authUrl.searchParams.append('response_type', 'code');
    authUrl.searchParams.append('scope', 'read write');
    authUrl.searchParams.append('code_challenge', codeChallenge);
    authUrl.searchParams.append('code_challenge_method', 'S256');

    // 認証ウィンドウを開く
    const authCode = await this.openAuthWindow(authUrl.toString(), redirectUri);

    // 認証コードをトークンに交換
    const tokens = await this.exchangeCodeForToken(
      tokenUrl,
      clientId,
      clientSecret,
      authCode,
      redirectUri,
      codeVerifier
    );

    return tokens;
  }

  private openAuthWindow(url: string, redirectUri: string): Promise<string> {
    return new Promise((resolve, reject) => {
      this.authWindow = new BrowserWindow({
        width: 800,
        height: 600,
        webPreferences: {
          nodeIntegration: false,
          contextIsolation: true
        }
      });

      this.authWindow.loadURL(url);

      // リダイレクトを監視
      this.authWindow.webContents.on('will-redirect', (event, newUrl) => {
        this.handleCallback(newUrl, redirectUri, resolve, reject);
      });

      this.authWindow.on('closed', () => {
        reject(new Error('Auth window was closed'));
      });
    });
  }

  private handleCallback(
    url: string,
    redirectUri: string,
    resolve: (value: string) => void,
    reject: (reason: Error) => void
  ): void {
    if (url.startsWith(redirectUri)) {
      const urlObj = new URL(url);
      const code = urlObj.searchParams.get('code');
      const error = urlObj.searchParams.get('error');

      if (this.authWindow) {
        this.authWindow.close();
      }

      if (error) {
        reject(new Error(error));
      } else if (code) {
        resolve(code);
      } else {
        reject(new Error('No code in callback'));
      }
    }
  }

  private async exchangeCodeForToken(
    tokenUrl: string,
    clientId: string,
    clientSecret: string,
    code: string,
    redirectUri: string,
    codeVerifier: string
  ): Promise<{ accessToken: string; refreshToken: string }> {
    const response = await fetch(tokenUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        grant_type: 'authorization_code',
        code,
        redirect_uri: redirectUri,
        client_id: clientId,
        client_secret: clientSecret,
        code_verifier: codeVerifier
      })
    });

    const data = await response.json();

    return {
      accessToken: data.access_token,
      refreshToken: data.refresh_token
    };
  }

  private generateCodeVerifier(): string {
    return crypto.randomBytes(32).toString('base64url');
  }

  private generateCodeChallenge(verifier: string): string {
    return crypto
      .createHash('sha256')
      .update(verifier)
      .digest('base64url');
  }
}

// IPCハンドラーの登録
const oauth2Service = new OAuth2Service();

ipcMain.handle('oauth2-login', async (event) => {
  try {
    const tokens = await oauth2Service.authenticate(
      'https://auth.example.com/oauth/authorize',
      'https://auth.example.com/oauth/token',
      'YOUR_CLIENT_ID',
      'YOUR_CLIENT_SECRET',
      'myapp://callback'
    );

    return tokens;
  } catch (error) {
    console.error('OAuth2 error:', error);
    throw error;
  }
});
```

### 2.2 preload.ts

```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('authAPI', {
  oauth2Login: () => ipcRenderer.invoke('oauth2-login'),
  logout: () => ipcRenderer.invoke('logout')
});
```

### 2.3 renderer.ts（ログイン画面）

```typescript
// renderer.ts
const loginButton = document.getElementById('login-btn');

loginButton?.addEventListener('click', async () => {
  try {
    const tokens = await window.authAPI.oauth2Login();
    console.log('Login successful:', tokens);

    // トークンを保存
    await saveTokens(tokens);

    // メイン画面に遷移
    window.location.href = 'main.html';
  } catch (error) {
    console.error('Login failed:', error);
    alert('ログインに失敗しました');
  }
});
```

## 3. トークンの安全な保存

### 3.1 keytar を使用したセキュアストレージ

```typescript
// main.ts
import * as keytar from 'keytar';

const SERVICE_NAME = 'MyElectronApp';
const ACCESS_TOKEN_KEY = 'accessToken';
const REFRESH_TOKEN_KEY = 'refreshToken';

class TokenStorage {
  async saveTokens(accessToken: string, refreshToken: string): Promise<void> {
    await keytar.setPassword(SERVICE_NAME, ACCESS_TOKEN_KEY, accessToken);
    await keytar.setPassword(SERVICE_NAME, REFRESH_TOKEN_KEY, refreshToken);
  }

  async getAccessToken(): Promise<string | null> {
    return await keytar.getPassword(SERVICE_NAME, ACCESS_TOKEN_KEY);
  }

  async getRefreshToken(): Promise<string | null> {
    return await keytar.getPassword(SERVICE_NAME, REFRESH_TOKEN_KEY);
  }

  async clearTokens(): Promise<void> {
    await keytar.deletePassword(SERVICE_NAME, ACCESS_TOKEN_KEY);
    await keytar.deletePassword(SERVICE_NAME, REFRESH_TOKEN_KEY);
  }
}

const tokenStorage = new TokenStorage();

ipcMain.handle('save-tokens', async (event, accessToken, refreshToken) => {
  await tokenStorage.saveTokens(accessToken, refreshToken);
});

ipcMain.handle('get-access-token', async () => {
  return await tokenStorage.getAccessToken();
});

ipcMain.handle('logout', async () => {
  await tokenStorage.clearTokens();
});
```

### 3.2 暗号化ストレージ（electron-store）

```typescript
import Store from 'electron-store';
import { app } from 'electron';

const store = new Store({
  name: 'secure-config',
  encryptionKey: 'your-encryption-key' // 実際は環境変数等から取得
});

class SecureStorage {
  saveToken(token: string): void {
    store.set('accessToken', token);
  }

  getToken(): string | undefined {
    return store.get('accessToken') as string | undefined;
  }

  clear(): void {
    store.clear();
  }
}
```

## 4. 認証状態の管理

### 4.1 認証サービスクラス

```typescript
// auth-service.ts (Main Process)
class AuthService {
  private accessToken: string | null = null;
  private refreshToken: string | null = null;
  private tokenExpiry: number | null = null;
  private tokenStorage: TokenStorage;

  constructor() {
    this.tokenStorage = new TokenStorage();
    this.loadTokensFromStorage();
  }

  private async loadTokensFromStorage(): Promise<void> {
    this.accessToken = await this.tokenStorage.getAccessToken();
    this.refreshToken = await this.tokenStorage.getRefreshToken();
  }

  async login(username: string, password: string): Promise<void> {
    const response = await fetch('https://api.example.com/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password })
    });

    if (!response.ok) {
      throw new Error('Login failed');
    }

    const data = await response.json();
    await this.setTokens(data.accessToken, data.refreshToken, data.expiresIn);
  }

  private async setTokens(
    accessToken: string,
    refreshToken: string,
    expiresIn: number
  ): Promise<void> {
    this.accessToken = accessToken;
    this.refreshToken = refreshToken;
    this.tokenExpiry = Date.now() + expiresIn * 1000;

    await this.tokenStorage.saveTokens(accessToken, refreshToken);
  }

  async getValidAccessToken(): Promise<string | null> {
    if (!this.accessToken) {
      return null;
    }

    // トークンの有効期限をチェック
    if (this.tokenExpiry && Date.now() >= this.tokenExpiry - 60000) {
      // 有効期限の1分前に更新
      await this.refreshAccessToken();
    }

    return this.accessToken;
  }

  private async refreshAccessToken(): Promise<void> {
    if (!this.refreshToken) {
      throw new Error('No refresh token available');
    }

    const response = await fetch('https://api.example.com/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken: this.refreshToken })
    });

    if (!response.ok) {
      await this.logout();
      throw new Error('Token refresh failed');
    }

    const data = await response.json();
    await this.setTokens(data.accessToken, data.refreshToken, data.expiresIn);
  }

  async makeAuthenticatedRequest(url: string, options: RequestInit = {}): Promise<any> {
    const token = await this.getValidAccessToken();

    if (!token) {
      throw new Error('Not authenticated');
    }

    const response = await fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        'Authorization': `Bearer ${token}`
      }
    });

    if (response.status === 401) {
      // トークンが無効な場合は再ログインが必要
      await this.logout();
      throw new Error('Authentication required');
    }

    return await response.json();
  }

  async logout(): Promise<void> {
    this.accessToken = null;
    this.refreshToken = null;
    this.tokenExpiry = null;
    await this.tokenStorage.clearTokens();

    // すべてのウィンドウにログアウトを通知
    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.send('auth-state-changed', { isAuthenticated: false });
    });
  }

  isAuthenticated(): boolean {
    return this.accessToken !== null;
  }
}

// グローバルインスタンス
const authService = new AuthService();

// IPCハンドラー
ipcMain.handle('login', async (event, username, password) => {
  await authService.login(username, password);
  return { success: true };
});

ipcMain.handle('logout', async () => {
  await authService.logout();
});

ipcMain.handle('is-authenticated', async () => {
  return authService.isAuthenticated();
});

ipcMain.handle('api-request', async (event, url, options) => {
  return await authService.makeAuthenticatedRequest(url, options);
});
```

## 5. ユーザー情報の管理

### 5.1 ユーザープロファイル

```typescript
interface UserProfile {
  id: string;
  username: string;
  email: string;
  displayName: string;
  avatar?: string;
  roles: string[];
}

class UserService {
  private currentUser: UserProfile | null = null;

  async loadUserProfile(): Promise<UserProfile> {
    const response = await authService.makeAuthenticatedRequest(
      'https://api.example.com/users/me'
    );

    this.currentUser = response;
    return this.currentUser;
  }

  getCurrentUser(): UserProfile | null {
    return this.currentUser;
  }

  hasRole(role: string): boolean {
    return this.currentUser?.roles.includes(role) ?? false;
  }

  async updateProfile(updates: Partial<UserProfile>): Promise<void> {
    const response = await authService.makeAuthenticatedRequest(
      'https://api.example.com/users/me',
      {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updates)
      }
    );

    this.currentUser = { ...this.currentUser, ...response };
  }
}

const userService = new UserService();

ipcMain.handle('get-user-profile', async () => {
  return await userService.loadUserProfile();
});

ipcMain.handle('update-user-profile', async (event, updates) => {
  await userService.updateProfile(updates);
});
```

## 6. 認可（Authorization）

### 6.1 ロールベースアクセス制御（RBAC）

```typescript
enum Permission {
  READ_USERS = 'read:users',
  WRITE_USERS = 'write:users',
  DELETE_USERS = 'delete:users',
  ADMIN = 'admin'
}

class PermissionService {
  private userPermissions: Set<Permission> = new Set();

  async loadPermissions(): Promise<void> {
    const user = userService.getCurrentUser();

    if (!user) {
      this.userPermissions.clear();
      return;
    }

    // ロールに基づいて権限を設定
    if (user.roles.includes('admin')) {
      this.userPermissions.add(Permission.ADMIN);
      this.userPermissions.add(Permission.READ_USERS);
      this.userPermissions.add(Permission.WRITE_USERS);
      this.userPermissions.add(Permission.DELETE_USERS);
    } else if (user.roles.includes('editor')) {
      this.userPermissions.add(Permission.READ_USERS);
      this.userPermissions.add(Permission.WRITE_USERS);
    } else {
      this.userPermissions.add(Permission.READ_USERS);
    }
  }

  hasPermission(permission: Permission): boolean {
    return this.userPermissions.has(permission);
  }

  requirePermission(permission: Permission): void {
    if (!this.hasPermission(permission)) {
      throw new Error(`Permission denied: ${permission}`);
    }
  }
}

const permissionService = new PermissionService();

ipcMain.handle('check-permission', async (event, permission: Permission) => {
  return permissionService.hasPermission(permission);
});

// 使用例
ipcMain.handle('delete-user', async (event, userId: string) => {
  permissionService.requirePermission(Permission.DELETE_USERS);

  // ユーザーの削除処理
  await authService.makeAuthenticatedRequest(
    `https://api.example.com/users/${userId}`,
    { method: 'DELETE' }
  );
});
```

## 7. セッション管理

### 7.1 自動ログアウト

```typescript
class SessionManager {
  private inactivityTimer: NodeJS.Timeout | null = null;
  private readonly INACTIVITY_TIMEOUT = 30 * 60 * 1000; // 30分

  startMonitoring(): void {
    this.resetTimer();

    // ユーザーのアクティビティを監視
    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.on('did-navigate', () => this.resetTimer());
      win.on('focus', () => this.resetTimer());
    });
  }

  private resetTimer(): void {
    if (this.inactivityTimer) {
      clearTimeout(this.inactivityTimer);
    }

    this.inactivityTimer = setTimeout(() => {
      this.handleInactivity();
    }, this.INACTIVITY_TIMEOUT);
  }

  private async handleInactivity(): Promise<void> {
    console.log('User inactive, logging out...');
    await authService.logout();

    // ログイン画面にリダイレクト
    BrowserWindow.getAllWindows().forEach((win) => {
      win.loadFile('login.html');
    });
  }

  stopMonitoring(): void {
    if (this.inactivityTimer) {
      clearTimeout(this.inactivityTimer);
    }
  }
}

const sessionManager = new SessionManager();

app.on('ready', () => {
  if (authService.isAuthenticated()) {
    sessionManager.startMonitoring();
  }
});
```

## 8. まとめ

- **OAuth 2.0**: PKCE拡張を使用した安全な認証
- **トークン管理**: keytar または暗号化ストレージで安全に保存
- **自動更新**: アクセストークンの有効期限を監視して自動更新
- **セッション管理**: 非アクティブタイムアウトを実装
- **認可**: RBAC（ロールベースアクセス制御）で権限管理

次のドキュメントでは、UIレンダリングとChromiumエンジンについて解説します。
