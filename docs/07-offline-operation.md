# オフライン動作とデータ同期

## 1. オフライン動作の概要

Electronアプリは、ネットワーク接続がない環境でも動作する必要があります。

### 1.1 オフラインファースト設計

```
┌────────────────────────────────────┐
│      Electron Application         │
├────────────────────────────────────┤
│  ┌──────────────┐  ┌────────────┐ │
│  │  Local DB    │  │   Cache    │ │
│  │  (IndexedDB  │  │  (LocalDB) │ │
│  │   SQLite)    │  │            │ │
│  └──────┬───────┘  └─────┬──────┘ │
│         │                 │        │
│         └────────┬────────┘        │
│                  │                 │
│         ┌────────▼────────┐        │
│         │  Sync Engine    │        │
│         └────────┬────────┘        │
│                  │                 │
│         ┌────────▼────────┐        │
│         │  API Client     │        │
│         └────────┬────────┘        │
└──────────────────┼─────────────────┘
                   │
         ┌─────────▼──────────┐
         │  Remote API Server │
         └────────────────────┘
```

## 2. ローカルデータストレージ

### 2.1 IndexedDB（Rendererプロセス）

```typescript
// db.ts
class DatabaseService {
  private db: IDBDatabase | null = null;
  private readonly DB_NAME = 'MyAppDB';
  private readonly DB_VERSION = 1;

  async open(): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.DB_NAME, this.DB_VERSION);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => {
        this.db = request.result;
        resolve();
      };

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;

        // オブジェクトストアの作成
        if (!db.objectStoreNames.contains('documents')) {
          const objectStore = db.createObjectStore('documents', {
            keyPath: 'id',
            autoIncrement: true
          });

          // インデックスの作成
          objectStore.createIndex('title', 'title', { unique: false });
          objectStore.createIndex('createdAt', 'createdAt', { unique: false });
          objectStore.createIndex('syncStatus', 'syncStatus', { unique: false });
        }
      };
    });
  }

  async save(storeName: string, data: any): Promise<number> {
    if (!this.db) throw new Error('Database not opened');

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readwrite');
      const objectStore = transaction.objectStore(storeName);

      // 同期状態を追加
      const record = {
        ...data,
        syncStatus: 'pending',
        updatedAt: new Date().toISOString()
      };

      const request = objectStore.add(record);

      request.onsuccess = () => resolve(request.result as number);
      request.onerror = () => reject(request.error);
    });
  }

  async get(storeName: string, id: number): Promise<any> {
    if (!this.db) throw new Error('Database not opened');

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readonly');
      const objectStore = transaction.objectStore(storeName);
      const request = objectStore.get(id);

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async getAll(storeName: string): Promise<any[]> {
    if (!this.db) throw new Error('Database not opened');

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readonly');
      const objectStore = transaction.objectStore(storeName);
      const request = objectStore.getAll();

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async update(storeName: string, data: any): Promise<void> {
    if (!this.db) throw new Error('Database not opened');

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readwrite');
      const objectStore = transaction.objectStore(storeName);

      const record = {
        ...data,
        syncStatus: 'pending',
        updatedAt: new Date().toISOString()
      };

      const request = objectStore.put(record);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async delete(storeName: string, id: number): Promise<void> {
    if (!this.db) throw new Error('Database not opened');

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readwrite');
      const objectStore = transaction.objectStore(storeName);
      const request = objectStore.delete(id);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async query(
    storeName: string,
    indexName: string,
    value: any
  ): Promise<any[]> {
    if (!this.db) throw new Error('Database not opened');

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readonly');
      const objectStore = transaction.objectStore(storeName);
      const index = objectStore.index(indexName);
      const request = index.getAll(value);

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }
}

const db = new DatabaseService();
await db.open();

// 使用例
await db.save('documents', {
  title: 'My Document',
  content: 'Lorem ipsum...',
  createdAt: new Date().toISOString()
});

const documents = await db.getAll('documents');
console.log(documents);
```

### 2.2 SQLite（Mainプロセス）

```typescript
// main.ts
import Database from 'better-sqlite3';
import * as path from 'path';
import { app } from 'electron';

class SQLiteService {
  private db: Database.Database;

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'app.db');
    this.db = new Database(dbPath);
    this.initialize();
  }

  private initialize(): void {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS documents (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        content TEXT,
        sync_status TEXT DEFAULT 'pending',
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
      );

      CREATE INDEX IF NOT EXISTS idx_sync_status
        ON documents(sync_status);

      CREATE TABLE IF NOT EXISTS sync_queue (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        table_name TEXT NOT NULL,
        record_id INTEGER NOT NULL,
        operation TEXT NOT NULL,
        data TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
      );
    `);
  }

  save(table: string, data: any): number {
    const stmt = this.db.prepare(`
      INSERT INTO ${table} (title, content)
      VALUES (@title, @content)
    `);

    const result = stmt.run(data);
    return result.lastInsertRowid as number;
  }

  getAll(table: string): any[] {
    const stmt = this.db.prepare(`SELECT * FROM ${table}`);
    return stmt.all();
  }

  get(table: string, id: number): any {
    const stmt = this.db.prepare(`SELECT * FROM ${table} WHERE id = ?`);
    return stmt.get(id);
  }

  update(table: string, id: number, data: any): void {
    const stmt = this.db.prepare(`
      UPDATE ${table}
      SET title = @title,
          content = @content,
          sync_status = 'pending',
          updated_at = CURRENT_TIMESTAMP
      WHERE id = @id
    `);

    stmt.run({ ...data, id });
  }

  delete(table: string, id: number): void {
    const stmt = this.db.prepare(`DELETE FROM ${table} WHERE id = ?`);
    stmt.run(id);
  }

  getPendingSync(table: string): any[] {
    const stmt = this.db.prepare(`
      SELECT * FROM ${table}
      WHERE sync_status = 'pending'
    `);
    return stmt.all();
  }

  markAsSynced(table: string, id: number): void {
    const stmt = this.db.prepare(`
      UPDATE ${table}
      SET sync_status = 'synced'
      WHERE id = ?
    `);
    stmt.run(id);
  }

  close(): void {
    this.db.close();
  }
}

const sqlite = new SQLiteService();

// IPCハンドラー
ipcMain.handle('db-save', async (event, table, data) => {
  return sqlite.save(table, data);
});

ipcMain.handle('db-get-all', async (event, table) => {
  return sqlite.getAll(table);
});

ipcMain.handle('db-get', async (event, table, id) => {
  return sqlite.get(table, id);
});

ipcMain.handle('db-update', async (event, table, id, data) => {
  sqlite.update(table, id, data);
});

ipcMain.handle('db-delete', async (event, table, id) => {
  sqlite.delete(table, id);
});
```

## 3. データ同期エンジン

### 3.1 同期戦略

```typescript
interface SyncRecord {
  id: number;
  localId: number;
  remoteId?: string;
  operation: 'create' | 'update' | 'delete';
  data: any;
  syncStatus: 'pending' | 'syncing' | 'synced' | 'conflict';
  retryCount: number;
  lastAttempt?: Date;
}

class SyncEngine {
  private isOnline: boolean = navigator.onLine;
  private syncInterval: NodeJS.Timeout | null = null;
  private isSyncing: boolean = false;

  constructor() {
    this.setupNetworkListeners();
    this.startPeriodicSync();
  }

  private setupNetworkListeners(): void {
    window.addEventListener('online', () => {
      console.log('Network online');
      this.isOnline = true;
      this.sync();
    });

    window.addEventListener('offline', () => {
      console.log('Network offline');
      this.isOnline = false;
    });
  }

  private startPeriodicSync(): void {
    // 5分ごとに同期
    this.syncInterval = setInterval(() => {
      if (this.isOnline) {
        this.sync();
      }
    }, 5 * 60 * 1000);
  }

  async sync(): Promise<void> {
    if (this.isSyncing || !this.isOnline) {
      return;
    }

    this.isSyncing = true;

    try {
      console.log('Starting sync...');

      // 1. ローカルの変更をサーバーにプッシュ
      await this.pushLocalChanges();

      // 2. サーバーの変更をローカルにプル
      await this.pullRemoteChanges();

      console.log('Sync completed');
    } catch (error) {
      console.error('Sync failed:', error);
    } finally {
      this.isSyncing = false;
    }
  }

  private async pushLocalChanges(): Promise<void> {
    const pendingRecords = await db.query('documents', 'syncStatus', 'pending');

    for (const record of pendingRecords) {
      try {
        if (!record.remoteId) {
          // 新規作成
          const response = await fetch('https://api.example.com/documents', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(record)
          });

          const data = await response.json();

          // ローカルレコードを更新
          await db.update('documents', {
            ...record,
            remoteId: data.id,
            syncStatus: 'synced'
          });
        } else {
          // 更新
          await fetch(`https://api.example.com/documents/${record.remoteId}`, {
            method: 'PUT',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(record)
          });

          await db.update('documents', {
            ...record,
            syncStatus: 'synced'
          });
        }
      } catch (error) {
        console.error(`Failed to sync record ${record.id}:`, error);
        // リトライカウントを増やす
      }
    }
  }

  private async pullRemoteChanges(): Promise<void> {
    // 最後の同期時刻を取得
    const lastSyncTime = localStorage.getItem('lastSyncTime') || '1970-01-01';

    const response = await fetch(
      `https://api.example.com/documents?since=${lastSyncTime}`
    );

    const remoteRecords = await response.json();

    for (const remoteRecord of remoteRecords) {
      // ローカルに存在するか確認
      const localRecords = await db.query(
        'documents',
        'remoteId',
        remoteRecord.id
      );

      if (localRecords.length === 0) {
        // 新規レコードを追加
        await db.save('documents', {
          ...remoteRecord,
          remoteId: remoteRecord.id,
          syncStatus: 'synced'
        });
      } else {
        const localRecord = localRecords[0];

        // コンフリクトチェック
        if (localRecord.syncStatus === 'pending') {
          // コンフリクト: ローカルとリモートの両方が変更されている
          await this.resolveConflict(localRecord, remoteRecord);
        } else {
          // リモートの変更を適用
          await db.update('documents', {
            ...remoteRecord,
            id: localRecord.id,
            remoteId: remoteRecord.id,
            syncStatus: 'synced'
          });
        }
      }
    }

    // 同期時刻を更新
    localStorage.setItem('lastSyncTime', new Date().toISOString());
  }

  private async resolveConflict(
    localRecord: any,
    remoteRecord: any
  ): Promise<void> {
    // コンフリクト解決戦略:
    // 1. 最後の更新時刻で判定（Last Write Wins）
    // 2. ユーザーに選択させる
    // 3. 両方を保持してマージ

    // ここでは Last Write Wins を実装
    const localTime = new Date(localRecord.updatedAt).getTime();
    const remoteTime = new Date(remoteRecord.updatedAt).getTime();

    if (remoteTime > localTime) {
      // リモートの方が新しい
      await db.update('documents', {
        ...remoteRecord,
        id: localRecord.id,
        remoteId: remoteRecord.id,
        syncStatus: 'synced'
      });
    } else {
      // ローカルの方が新しい → 再度プッシュ
      await fetch(`https://api.example.com/documents/${remoteRecord.id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(localRecord)
      });

      await db.update('documents', {
        ...localRecord,
        syncStatus: 'synced'
      });
    }
  }

  stop(): void {
    if (this.syncInterval) {
      clearInterval(this.syncInterval);
    }
  }
}

const syncEngine = new SyncEngine();

// 手動同期
document.getElementById('sync-btn')?.addEventListener('click', async () => {
  await syncEngine.sync();
});
```

## 4. オフラインキャッシュ戦略

### 4.1 Service Workerの使用（Rendererプロセス）

```typescript
// service-worker.ts
const CACHE_NAME = 'my-app-v1';
const urlsToCache = [
  '/',
  '/index.html',
  '/styles.css',
  '/renderer.js',
  '/assets/logo.png'
];

self.addEventListener('install', (event: any) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(urlsToCache);
    })
  );
});

self.addEventListener('fetch', (event: any) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      // キャッシュがあれば返す
      if (response) {
        return response;
      }

      // なければネットワークから取得
      return fetch(event.request).then((response) => {
        // 成功したらキャッシュに保存
        if (response.status === 200) {
          const responseClone = response.clone();
          caches.open(CACHE_NAME).then((cache) => {
            cache.put(event.request, responseClone);
          });
        }

        return response;
      });
    })
  );
});

// 古いキャッシュを削除
self.addEventListener('activate', (event: any) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames.map((cacheName) => {
          if (cacheName !== CACHE_NAME) {
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});
```

### 4.2 キャッシュファーストストラテジー

```typescript
class CacheManager {
  private cache: Map<string, { data: any; expiry: number }> = new Map();
  private readonly DEFAULT_TTL = 5 * 60 * 1000; // 5分

  async get(key: string): Promise<any | null> {
    const cached = this.cache.get(key);

    if (!cached) {
      return null;
    }

    // 有効期限チェック
    if (Date.now() > cached.expiry) {
      this.cache.delete(key);
      return null;
    }

    return cached.data;
  }

  set(key: string, data: any, ttl: number = this.DEFAULT_TTL): void {
    this.cache.set(key, {
      data,
      expiry: Date.now() + ttl
    });
  }

  clear(): void {
    this.cache.clear();
  }

  async fetchWithCache(
    url: string,
    options?: RequestInit,
    ttl?: number
  ): Promise<any> {
    // キャッシュをチェック
    const cached = await this.get(url);
    if (cached) {
      console.log('Cache hit:', url);
      return cached;
    }

    // ネットワークから取得
    try {
      const response = await fetch(url, options);
      const data = await response.json();

      // キャッシュに保存
      this.set(url, data, ttl);

      return data;
    } catch (error) {
      console.error('Fetch failed:', error);

      // オフライン時は IndexedDB から取得
      const offlineData = await db.get('cache', url);
      if (offlineData) {
        return offlineData.data;
      }

      throw error;
    }
  }
}

const cacheManager = new CacheManager();

// 使用例
const data = await cacheManager.fetchWithCache(
  'https://api.example.com/data',
  { method: 'GET' },
  10 * 60 * 1000 // 10分間キャッシュ
);
```

## 5. バックグラウンド同期

### 5.1 定期的なバックグラウンド同期

```typescript
// main.ts
class BackgroundSyncService {
  private syncTimer: NodeJS.Timeout | null = null;

  start(): void {
    // 5分ごとに同期
    this.syncTimer = setInterval(() => {
      this.performSync();
    }, 5 * 60 * 1000);

    // 初回実行
    this.performSync();
  }

  private async performSync(): Promise<void> {
    console.log('Background sync started');

    try {
      // 同期処理
      const pendingRecords = sqlite.getPendingSync('documents');

      for (const record of pendingRecords) {
        await this.syncRecord(record);
      }

      // 全ウィンドウに通知
      BrowserWindow.getAllWindows().forEach((win) => {
        win.webContents.send('sync-completed', {
          timestamp: new Date().toISOString()
        });
      });
    } catch (error) {
      console.error('Background sync failed:', error);
    }
  }

  private async syncRecord(record: any): Promise<void> {
    // API呼び出し
    await authService.makeAuthenticatedRequest(
      'https://api.example.com/documents',
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(record)
      }
    );

    // 同期済みとしてマーク
    sqlite.markAsSynced('documents', record.id);
  }

  stop(): void {
    if (this.syncTimer) {
      clearInterval(this.syncTimer);
    }
  }
}

const backgroundSync = new BackgroundSyncService();

app.on('ready', () => {
  backgroundSync.start();
});

app.on('quit', () => {
  backgroundSync.stop();
});
```

## 6. まとめ

- **オフラインファースト**: ローカルDBを優先、バックグラウンドで同期
- **IndexedDB**: Rendererプロセスでの構造化データ保存
- **SQLite**: Mainプロセスでの大量データ保存
- **同期エンジン**: 双方向同期、コンフリクト解決
- **キャッシュ**: ネットワークリクエストのキャッシュ

次のドキュメントでは、配布・インストール・自動更新について解説します。
