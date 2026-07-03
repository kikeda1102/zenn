---
title: "Next.js 16のdev環境でボタンが反応しない？LAN IPアクセス時のHMR WebSocket問題と解決策"
emoji: "🔧"
type: "tech"
topics: ["nextjs", "react", "pnpm", "shadcn"]
published: false
---

## はじめに

Next.js 16 の開発環境で、ページは表示されるのにボタンやリンクが一切反応しないという現象に遭遇しました。本番環境では問題なく動作しており、原因の特定に苦労しましたが、最終的に Next.js 16.2.0 以降の仕様変更に起因する問題であることがわかりました。

この記事では、問題の症状・原因・解決策を整理して共有します。

## 環境

- Next.js 16.2.x（React 19.2.4）
- pnpm
- shadcn/ui v4（@base-ui/react）
- macOS

## 症状

- `pnpm run dev` で開発サーバーを起動
- `http://192.168.0.3:3000/`（LAN IP）でブラウザからアクセス
- ページは正常に表示される
- しかし、ボタン、リンク、ドロップダウンなど、すべてのインタラクティブ要素がクリックに反応しない
- `http://localhost:3000/` でアクセスすると正常に動作する
- 本番環境（`next build` + `next start`、Vercelデプロイ）では問題なし

## 原因

### Next.js 16.2.0 以降の仕様変更

Next.js 16.2.0 で、dev モードのハイドレーションプロセスに debugChannel という仕組みが導入されました。これは HMR（Hot Module Replacement）の WebSocket を経由して動作します。

この debugChannel のストリームが正常に閉じないと、ハイドレーションが完了しません。

### LAN IP アクセス時に何が起きるか

問題の因果関係は以下の通りです。

1. `next dev` はデフォルトで `localhost`（127.0.0.1）にバインドされる
2. ブラウザが `192.168.0.3`（LAN IP）からアクセスすると、クライアントサイドのJSに埋め込まれた HMR WebSocket の接続先が一致せず、WebSocket 接続に失敗する
3. debugChannel が閉じないため、`createRoot().render()` が呼ばれず、ハイドレーションがハングする
4. ハイドレーションが完了しない = React のイベントリスナーが DOM に付与されない
5. ページはサーバーサイドレンダリングされた HTML がそのまま表示されるため見た目は正常だが、一切のインタラクションが効かない

### なぜ本番では問題ないのか

本番ビルド（`next build` + `next start`）では、HMR や debugChannel は一切含まれません。ハイドレーションは WebSocket に依存せず即座に完了するため、LAN IP からのアクセスでも正常に動作します。

## 解決策

### devスクリプトに `--hostname 0.0.0.0` を追加する

`package.json` の `dev` スクリプトを修正します。

```diff
{
  "scripts": {
-   "dev": "next dev",
+   "dev": "next dev --hostname 0.0.0.0",
  }
}
```

`--hostname 0.0.0.0` を指定することで、開発サーバーがすべてのネットワークインターフェースでリッスンするようになり、LAN IP からの HMR WebSocket 接続も正常に確立されます。

### 代替策: localhost でアクセスする

修正が不要な場合は、単に `http://localhost:3000/` でアクセスすれば問題は発生しません。ただし、スマートフォンなど別デバイスからの動作確認が必要な場合は、上記の `--hostname` 指定が必要です。

## 補足: 紛らわしいポイント

この問題が厄介なのは、以下の点で原因の特定が難しいところです。

- ページの見た目は完全に正常 — SSRされたHTMLが表示されるため、一見問題がないように見える
- コンソールにエラーが出ない場合がある — WebSocket 接続失敗が静かに処理されることがある
- 特定のライブラリの問題に見える — shadcn/Radix UI のハイドレーション問題と誤認しやすい
- `localhost` では再現しない — 開発者のメインマシンで `localhost` を使っていると気づけない

## 関連情報

- [GitHub Discussion #91770: Dev mode: blank page when SSR error + HMR WebSocket fails](https://github.com/vercel/next.js/discussions/91770)

## まとめ

Next.js 16.2.0 以降の dev モードで LAN IP からアクセスした際にボタンが反応しない場合は、`next dev --hostname 0.0.0.0` を試してみてください。原因は HMR WebSocket の接続失敗によるハイドレーションの未完了です。
