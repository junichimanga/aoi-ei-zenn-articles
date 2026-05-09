---
title: "Claude MCPサーバー自作ハンズオン｜TS版・公開まで【無料サンプル2個公開】"
emoji: "🤖"
type: "tech"
topics: ["Claude", "AI", "開発ツール", "生産性向上"]
published: true
---
# Claude MCPサーバー自作ハンズオン｜TS版・公開まで【無料サンプル2個公開】

## この記事でわかること

- MCP（Model Context Protocol）の仕組みと、TypeScriptで自作する意義
- プロジェクト初期化からpackage.json設定まで、動かせる土台の作り方
- Claude Desktopへの接続・npmへの公開という「一気通貫の流れ」の全体像

---

## なぜ今、MCPサーバーを自作するのか

**MCP（Model Context Protocol）** は、Anthropicが2024年11月に公開したオープン仕様だ。ClaudeとローカルファイルやAPIを「標準プロトコル」でつなぐための橋渡し役を担う。

従来、ClaudeにローカルのデータやAPIを触らせるには、プロンプトに直接貼り付けるか、独自のラッパーAPIを用意するかしかなかった。MCPはその橋渡しを標準化し、誰でも同じ設計で実装できる状態にした。

```
┌─────────────────┐    stdio / SSE    ┌────────────────────┐
│  Claude Desktop  │ ◄───────────────► │  Your MCP Server   │
│  (MCP Client)   │   JSON-RPC 2.0    │  (Node.js / Python)│
└─────────────────┘                   └────────────────────┘
                                               │
                                          任意のリソース
                                    (ファイル / DB / API)
```

2025年現在、日本語の実装解説はまだ少ない状況だ。社内APIやCMSとClaudeを直接つなぐことができれば、業務フローを変えられる可能性がある。また、作ったサーバーはnpmで配布・販売できる資産にもなりえる。エコシステムが成熟しきる前に手を動かしておく価値は、筆者の体験としても感じるところだ。

---

## サンプル①：プロジェクト初期化スクリプト

まず土台を作る。以下のコマンドをそのまま実行すれば、MCP対応のTypeScriptプロジェクトが立ち上がる。

```bash
# プロジェクトディレクトリ作成
mkdir my-mcp-server && cd my-mcp-server

# npm初期化（-yで全てデフォルト）
npm init -y

# 必要なパッケージをインストール
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node tsx

# tsconfig.json生成
npx tsc --init --target ES2022 --module NodeNext --moduleResolution NodeNext \
  --outDir dist --rootDir src --strict true --esModuleInterop true
```

ここで重要なのは `--module NodeNext` の指定だ。MCP SDKはESM前提で設計されており、CommonJS設定のままにしておくとimportエラーが頻出する。筆者も最初に嵌ったポイントのひとつで、ここだけ先に知っておくとかなりスムーズに進む。

---

## サンプル②：MCPサーバー用 package.json テンプレート

初期化後、`package.json` をMCPサーバー向けに整える必要がある。特に `"type": "module"` と `"bin"` の設定が後の工程で効いてくる。

```json
{
  "name": "my-mcp-server",
  "version": "1.0.0",
  "description": "My first MCP server",
  "type": "module",
  "main": "dist/index.js",
  "bin": {
    "my-mcp-server": "dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsx src/index.ts",
    "start": "node dist/index.js",
    "prepare": "npm run build"
  },
  "keywords": ["mcp", "claude", "model-context-protocol"],
  "license": "MIT",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "zod": "^3.22.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "tsx": "^4.0.0",
    "typescript": "^5.0.0"
  }
}
```

`"bin"` フィールドを設定しておくと、npm公開後に `npx my-mcp-server` で即起動できる。Claude Desktopの設定ファイル（`claude_desktop_config.json`）からコマンドで呼び出す際にも、この設定が効いてくる。

---

## カスタマイズの考え方

上記2つのサンプルは「どんなMCPサーバーを作る場合にも共通する土台」だ。ここに `src/index.ts` を追加し、実際のツール定義（`tools/list` および `tools/call` のハンドラ）を実装していく流れになる。

用途別のサーバー構成の例としては、次のようなものが考えられる。

- **Echoサーバー**：最初の動作確認用。入力をそのまま返すだけの最小構成から始めると、MCPの通信構造が理解しやすい
- **ファイル読み込みサーバー**：Claudeにローカルファイルを参照させたいケース。パスを受け取って`fs.readFile`で返すだけでも実用的に使える
- **外部API連携サーバー**：天気APIやSlack、Notionなど外部サービスをClaudeから操作したいとき。`zod`でリクエストのバリデーションを入れるのが安全だ

土台さえ固まれば、あとはツールの数を増やしていくだけ。最終的には `npm publish` でパッケージとして公開し、他の人にも使ってもらえる状態を目指せる。

---

### 完全版について

完全版のハンズオン資料では、この記事の土台に加えて以下の内容を一冊にまとめている。

- **サーバー①Echo・②ファイル読み込み・③外部API呼び出し**の完全実装コード
- **Claude Desktopへの接続手順**（設定ファイルの書き方・パスの通し方）
- **npmパッケージとして公開する**ためのステップバイステップ解説
- **トラブルシュート集10項目**（初回実装でよく出るエラーと対処法）
- **付録**：設定ファイルテンプレート一覧

→ **[Claude MCPサーバー自作ハンズオン｜TS版・公開まで（Gumroad）](https://aoiei.gumroad.com/l/tkrmth)**

この記事の無料部分だけでも、環境構築のヒントとして参考になれば幸いです。手を動かしながら読むのが一番身につく、と筆者は感じています。