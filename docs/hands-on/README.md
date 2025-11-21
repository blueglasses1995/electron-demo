# Electron アプリ開発 ハンズオンカリキュラム

TypeScriptを使用したElectronアプリケーション開発を、実際に動くメモ帳アプリを作りながら学ぶハンズオンカリキュラムです。

## 📚 カリキュラム構成

### [Part 1: 環境構築](./01-environment-setup.md)
**所要時間**: 30分 | **難易度**: ⭐☆☆☆☆

- Node.js/npmのインストール
- プロジェクトの初期化
- TypeScript環境の構築
- 最小限のElectronアプリの作成

**成果物**: Hello World Electronアプリ

### [Part 2: 基本アプリケーション開発](./02-basic-app-development.md)
**所要時間**: 2時間 | **難易度**: ⭐⭐☆☆☆

- アプリケーションメニューの作成
- ファイル操作（開く/保存）
- IPC通信の基礎
- テキストエディタUIの構築

**成果物**: ファイルの読み書きができるメモ帳アプリ

### [Part 3: IPC通信とローカルストレージ](./03-ipc-and-storage.md)
**所要時間**: 2時間 | **難易度**: ⭐⭐⭐☆☆

- electron-storeによる設定管理
- IPCパターン（invoke/handle, イベント通知）
- 設定画面の実装
- テーマシステムの構築
- 自動保存機能

**成果物**: 設定機能付きメモ帳アプリ

### [Part 4: ネットワーク通信と認証](./04-network-and-auth.md)
**所要時間**: 3時間 | **難易度**: ⭐⭐⭐⭐☆

- REST API通信の実装
- 認証フロー（ログイン/ログアウト）
- トークン管理とリフレッシュ
- クラウド同期機能
- オフライン対応

**成果物**: クラウド同期機能付きメモ帳アプリ

### [Part 5: パッケージングとデプロイ](./05-packaging-and-deployment.md)
**所要時間**: 2時間 | **難易度**: ⭐⭐⭐☆☆

- electron-builderの設定
- プラットフォーム別ビルド
- 自動更新機能の実装
- GitHub Releasesでの配布
- CI/CDパイプライン

**成果物**: 配布可能なアプリケーション

### [演習問題集](./EXERCISES.md)
**所要時間**: 10時間+ | **難易度**: ⭐⭐⭐⭐⭐

各パートの演習問題と解答例、総合課題が含まれています。

## 🎯 学習の進め方

### 初心者の方

1. **Part 1**から順番に進める
2. 各ステップでコードを実際に書いて動作確認
3. エラーが出たらトラブルシューティングを参照
4. 演習問題の基礎編にチャレンジ

### 中級者の方

1. Part 1-2は軽く流し読み
2. Part 3から本格的に実装
3. 演習問題の応用編にチャレンジ
4. 独自機能を追加してカスタマイズ

### 上級者の方

1. 気になるパートだけピックアップ
2. 演習問題の上級編・エキスパート編にチャレンジ
3. 技術ドキュメントと照らし合わせて深掘り
4. 独自アプリケーションの開発

## 🛠️ 必要な環境

### ソフトウェア

- **Node.js**: 18.x以上
- **npm**: 9.x以上
- **Git**: 2.x以上
- **テキストエディタ**: VS Code推奨

### ハードウェア

- **CPU**: 2GHz以上、2コア以上推奨
- **RAM**: 4GB以上（8GB推奨）
- **ストレージ**: 5GB以上の空き容量
- **OS**: Windows 10+, macOS 10.15+, Ubuntu 20.04+

## 📦 セットアップ

### 1. Node.jsのインストール

```bash
# バージョン確認
node --version
npm --version

# nvmを使用する場合
nvm install 18
nvm use 18
```

### 2. プロジェクトの作成

```bash
# プロジェクトディレクトリ作成
mkdir my-notepad-app
cd my-notepad-app

# 初期化
npm init -y

# ElectronとTypeScriptのインストール
npm install --save-dev electron typescript
npm install --save-dev @types/node
```

### 3. カリキュラムの開始

Part 1のドキュメントを開いて、ステップバイステップで進めていきます。

## 📖 関連ドキュメント

本カリキュラムは、以下の技術ドキュメントと連携しています:

1. [Electronアーキテクチャの基礎](../01-electron-architecture.md)
2. [ハードウェアとOS基盤](../02-hardware-and-os.md)
3. [プロセス・スレッド管理](../03-process-thread-management.md)
4. [ネットワーク通信](../04-network-communication.md)
5. [認証認可とアカウント管理](../05-authentication-authorization.md)
6. [UIレンダリング](../06-ui-rendering.md)
7. [オフライン動作とデータ同期](../07-offline-operation.md)
8. [配布・インストール・自動更新](../08-distribution-and-updates.md)

## 🎓 学習のヒント

### デバッグ方法

```typescript
// Mainプロセス
console.log('Main process:', data);

// Rendererプロセス（DevTools）
console.log('Renderer process:', data);

// ファイルに出力（本番環境）
import log from 'electron-log';
log.info('Info message');
log.error('Error message');
```

### パフォーマンス測定

```typescript
// 処理時間の測定
console.time('operation');
// ... 処理
console.timeEnd('operation');

// メモリ使用量
console.log('Memory:', process.memoryUsage());
```

### よくあるエラーと解決方法

#### エラー: "Cannot find module 'electron'"

```bash
npm install --save-dev electron
```

#### エラー: "contextBridge is not defined"

```typescript
// webPreferences で contextIsolation: true を確認
new BrowserWindow({
  webPreferences: {
    contextIsolation: true,
    preload: path.join(__dirname, 'preload.js')
  }
});
```

#### エラー: "require is not defined"

```typescript
// Rendererプロセスでは require は使えません
// preload.ts で必要な機能を公開してください
```

## 💡 おすすめリソース

### 公式ドキュメント

- [Electron公式ドキュメント](https://www.electronjs.org/docs)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [Node.js公式ドキュメント](https://nodejs.org/docs/)

### コミュニティ

- [Electron Discord](https://discord.gg/electron)
- [Stack Overflow - Electron Tag](https://stackoverflow.com/questions/tagged/electron)
- [GitHub Discussions](https://github.com/electron/electron/discussions)

### ツール・ライブラリ

- [electron-builder](https://www.electron.build/) - パッケージング
- [electron-store](https://github.com/sindresorhus/electron-store) - 設定管理
- [electron-log](https://github.com/megahertz/electron-log) - ロギング
- [electron-updater](https://www.electron.build/auto-update) - 自動更新

## 🚀 完成後のNext Steps

カリキュラムを完了したら、以下のステップに進みましょう:

### 1. 機能拡張

- マークダウンプレビュー
- タブ機能
- 検索・置換機能
- プラグインシステム

### 2. UI/UX改善

- アニメーション追加
- レスポンシブデザイン
- アクセシビリティ対応
- キーボードショートカット拡充

### 3. パフォーマンス最適化

- 大きなファイルの処理
- メモリ使用量の削減
- 起動時間の短縮
- レンダリング最適化

### 4. セキュリティ強化

- Content Security Policy強化
- コード署名の実装
- セキュリティ監査
- 脆弱性対策

### 5. 公開・配布

- GitHub Releasesで公開
- Microsoft Store / Mac App Storeへ申請
- ユーザードキュメント作成
- フィードバック収集

## 📝 フィードバック

カリキュラムの改善提案やバグ報告は、GitHubのIssueでお願いします。

## 📄 ライセンス

このカリキュラムはMITライセンスの下で公開されています。

---

**Happy Coding! 🎉**

Electronアプリ開発を楽しみながら学びましょう！
