# Electronアプリ技術ドキュメント＆ハンズオンカリキュラム

このドキュメント集は、TypeScriptを使用したElectronアプリケーションの開発について、技術的原理から実践的な開発手法まで体系的に学習できるように構成されています。

## 📚 技術ドキュメント

### 1. [Electronアーキテクチャの基礎](./01-electron-architecture.md)
- Electronとは何か
- Chromium + Node.jsのハイブリッドアーキテクチャ
- Mainプロセス vs Rendererプロセス
- IPCによるプロセス間通信
- セキュリティモデル

### 2. [ハードウェアとOS基盤](./02-hardware-and-os.md)
- 動作環境（Windows/macOS/Linux）
- CPUアーキテクチャ（x64, ARM64）
- OS APIとの統合
- ネットワークプロトコル（TCP/IP, HTTP, WebSocket）
- メモリ・ストレージ管理

### 3. [プロセス・スレッド管理の詳細](./03-process-thread-management.md)
- マルチプロセスアーキテクチャ
- Mainプロセスのスレッド構成
- Rendererプロセスのスレッド構成
- Web Workers/Utility Processの活用
- プロセスライフサイクル管理
- クラッシュハンドリング

### 4. [ネットワーク通信](./04-network-communication.md)
- HTTP/HTTPS通信（fetch, axios, electron.net）
- WebSocket通信
- リアルタイム通信の実装
- ネットワーク状態の監視
- セキュアな通信（TLS/SSL）

### 5. [認証認可とアカウント管理](./05-authentication-authorization.md)
- OAuth 2.0認証フロー
- トークン管理（keytar, electron-store）
- セッション管理
- ロールベースアクセス制御（RBAC）
- セキュアストレージ

### 6. [UIレンダリングとChromiumエンジン](./06-ui-rendering.md)
- Blinkレンダリングパイプライン
- V8 JavaScriptエンジン
- ハードウェアアクセラレーション（GPU）
- パフォーマンス最適化
- React/Vue等のフレームワーク統合

### 7. [オフライン動作とデータ同期](./07-offline-operation.md)
- オフラインファースト設計
- IndexedDB（Rendererプロセス）
- SQLite（Mainプロセス）
- データ同期エンジン
- コンフリクト解決

### 8. [配布・インストール・自動更新](./08-distribution-and-updates.md)
- electron-builderによるパッケージング
- プラットフォーム別ビルド（Windows/macOS/Linux）
- コード署名（Windows証明書、macOS Notarization）
- electron-updaterによる自動更新
- GitHub Releasesでの配布

## 🛠️ ハンズオンカリキュラム

### Part 1: [環境構築](./hands-on/01-environment-setup.md)
**所要時間**: 30分

- Node.js/npmのインストール
- プロジェクトの初期化
- TypeScript設定
- 最小限のElectronアプリの作成
- 開発環境の構築

**成果物**: Hello World Electronアプリ

### Part 2: [基本アプリケーション開発](./hands-on/02-basic-app-development.md)
**所要時間**: 2時間

- アプリケーションメニューの作成
- ファイル操作（開く/保存）
- IPC通信の実装
- テキストエディタUIの構築

**成果物**: シンプルなメモ帳アプリ

### Part 3: [IPC通信とローカルストレージ](./hands-on/03-ipc-and-storage.md)
**所要時間**: 2時間

- IPCのパターン（invoke/handle, send/on）
- electron-storeによる設定保存
- 設定画面の実装
- テーマシステムの構築
- 自動保存機能

**成果物**: 設定を保存できるメモ帳アプリ

### Part 4: [ネットワーク通信と認証](./hands-on/04-network-and-auth.md)
**所要時間**: 3時間

- REST API通信
- 認証フロー（ログイン/ログアウト/トークンリフレッシュ）
- クラウド同期機能
- オフライン対応

**成果物**: クラウド同期機能付きメモ帳アプリ

### Part 5: [パッケージングとデプロイ](./hands-on/05-packaging-and-deployment.md)
**所要時間**: 2時間

- electron-builderの設定
- Windows/macOS/Linuxビルド
- アイコン・リソースの設定
- 自動更新の実装
- GitHub Releasesでの配布

**成果物**: 配布可能なアプリケーション

### [演習問題集](./hands-on/EXERCISES.md)
**所要時間**: 10時間+

各パートの演習問題と詳細な解答例が含まれています:
- 基礎編: 各機能の基本的な拡張
- 応用編: 実践的な機能追加
- 上級編: 高度な機能実装
- 総合課題: マークダウンプレビュー、タブ機能、Git連携など

## 🎯 学習の進め方

### 初心者向け
1. まず技術ドキュメント01-03を読んでElectronの基本を理解
2. ハンズオンPart 1-2を実施
3. 不明点があれば対応する技術ドキュメントを参照

### 中級者向け
1. 技術ドキュメントを通読
2. ハンズオンを順番に実施
3. 各章の演習問題にチャレンジ
4. 独自機能を追加してカスタマイズ

### 上級者向け
1. 技術ドキュメントで深掘りしたいトピックを選択
2. ハンズオンのコードをベースに独自アプリを開発
3. パフォーマンス最適化、セキュリティ強化を実施

## 📝 よくある質問

### Q: Electronとモバイルアプリ（Kotlin/Swift）の違いは?

A: Electronはデスクトップアプリケーションのフレームワークです。モバイルアプリとの主な違い:

- **プラットフォーム**: Electron (Windows/macOS/Linux) vs モバイル (iOS/Android)
- **ランタイム**: Chromium + Node.js vs ネイティブランタイム
- **配布**: 直接ダウンロード/Microsoft Store/Mac App Store vs App Store/Google Play Store必須
- **システムアクセス**: より広範なOS APIアクセス vs 厳格なサンドボックス

### Q: App Store / Google Play Storeで配布できますか?

A: Electronアプリは**デスクトップアプリ**なので:
- ✅ Mac App Store（macOS版）
- ✅ Microsoft Store（Windows版）
- ❌ Google Play Store（Androidではない）
- ❌ App Store（iOSではない）

モバイルアプリを開発したい場合は、React Native、Flutter、Kotlin/Swiftを検討してください。

### Q: Kotlin RuntimeはElectronで使えますか?

A: Electronは **Node.js + Chromium** ベースなので、Kotlinランタイムは直接使用しません。
- MainプロセスではNode.js（JavaScript/TypeScript）
- RendererプロセスではChromium（HTML/CSS/JavaScript）

Kotlinを使いたい場合は、Androidアプリ開発を検討してください。

### Q: プッシュ通知は実装できますか?

A: デスクトップ通知は可能ですが、モバイルのようなリモートプッシュ通知とは異なります:
- ✅ ローカル通知（Notification API）
- ✅ システムトレイ通知
- ⚠️ リモートプッシュ（WebSocketやポーリングで実装）

### Q: 課金機能は実装できますか?

A: Electronアプリでの課金方法:
- Stripe/PayPal等の決済APIを統合
- Mac App StoreではIn-App Purchase可能
- サブスクリプションはバックエンドAPIで管理

## 🔧 開発環境要件

- **Node.js**: 18.x以上
- **npm/yarn**: 最新版
- **OS**: Windows 10+, macOS 10.15+, Ubuntu 20.04+
- **エディタ**: VS Code推奨
- **メモリ**: 8GB以上推奨

## 📦 主要依存パッケージ

```json
{
  "electron": "^27.0.0",
  "electron-builder": "^24.0.0",
  "electron-updater": "^6.1.0",
  "typescript": "^5.0.0"
}
```

## 🚀 クイックスタート

```bash
# プロジェクトのクローン
git clone <your-repo-url>
cd electron-demo

# 依存パッケージのインストール
npm install

# 開発モードで起動
npm run dev

# ビルド
npm run build

# パッケージング
npm run dist
```

## 📚 参考リソース

- [Electron公式ドキュメント](https://www.electronjs.org/docs)
- [TypeScript公式](https://www.typescriptlang.org/)
- [Chromium開発者向けドキュメント](https://www.chromium.org/developers)
- [Node.js公式](https://nodejs.org/)

## 🤝 コントリビューション

ドキュメントの改善提案やバグ報告は、GitHubのIssueでお願いします。

## 📄 ライセンス

MIT License

---

**注意**: このドキュメントは教育目的で作成されています。本番環境での使用前にはセキュリティレビューを実施してください。
