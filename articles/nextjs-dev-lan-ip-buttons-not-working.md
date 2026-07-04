---
title: "Next.jsのdev環境でボタンが反応しない問題"
emoji: "🔧"
type: "tech"
topics: ["nextjs"]
published: false
---

## 環境

- Next.js 16.2.x

## 症状

`pnpm run dev` で開発サーバーを起動すると、ターミナルに2つのアドレスが表示される。

```
▲ Next.js 16.2.9 (Turbopack)
- Local:    http://localhost:3000
- Network:  http://172.20.10.4:3000
```

Network 側のアドレスをブラウザで開いたところ、ページは正常に表示された。しかし、ボタンやドロップダウンなどのインタラクティブな要素がクリックに一切反応しなかった。`a` タグによるリンクは機能するが、React が制御するイベントが動作しない状態だった。

Local（localhost）で開くと問題は起きなかった。

## 原因

ターミナルに以下の警告が出力されていた。

```
⚠ Blocked cross-origin request to Next.js dev resource /_next/webpack-hmr from "172.20.10.4".
```

Next.js は dev モードにおいて、クロスオリジン（異なるホスト名からのアクセス）による開発リソースへのリクエストをデフォルトでブロックする。Network のアドレスからアクセスすると localhost とは異なるオリジンとみなされ、HMR（Hot Module Replacement = コード変更をブラウザへ即座に反映する仕組み）用の WebSocket（サーバーとブラウザ間のリアルタイム通信）接続がブロックされる。

この WebSocket 接続が確立できないと、ハイドレーション（サーバーで生成した HTML に React のイベント処理を紐づける工程）が完了しない。その結果、ページの見た目は正常だがボタンなどの操作が一切効かない状態になる。

本番ビルド（`next build` + `next start`）では HMR は含まれないため、この問題は発生しない。

## 解決策

`next.config.ts` に [`allowedDevOrigins`](https://nextjs.org/docs/app/api-reference/config/next-config-js/allowedDevOrigins) を追加する。

```ts
const nextConfig: NextConfig = {
  allowedDevOrigins: ["172.20.10.4"],
};
```

値は URL ではなく、ホスト名または IP アドレスのみを指定する。ターミナルの警告メッセージに表示される値をそのまま使えばよい。

設定後に dev サーバーを再起動すると、Network のアドレスからアクセスしてもボタンが正常に動作するようになる。

## 関連情報

- [allowedDevOrigins（Next.js 公式ドキュメント）](https://nextjs.org/docs/app/api-reference/config/next-config-js/allowedDevOrigins)
- [GitHub Discussion #91770: Dev mode: blank page when SSR error + HMR WebSocket fails](https://github.com/vercel/next.js/discussions/91770)
