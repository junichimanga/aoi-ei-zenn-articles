---
title: "Claude MCPサーバー自作キット｜日本語PDF＋雛形4種【無料サンプル2個公開】"
emoji: "🤖"
type: "tech"
topics: ["Claude", "AI", "開発ツール", "生産性向上"]
published: true
---
# Claude MCPサーバー自作キット｜日本語PDF＋雛形4種【無料サンプル2個公開】

## この記事でわかること

- MCP（Model Context Protocol）の構造と、なぜClaudeの拡張に使えるのかの基本
- Node.js + TypeScriptでMCPサーバーを動かすための最低限のセットアップ手順
- 最小構成の雛形コード（hello-tool）を使って、Claudeにカスタムツールを呼ばせる方法

---

## なぜ「自作MCPサーバー」が必要になるのか

Claude Desktopはそのままでも十分賢い。  
だが、こんな壁に当たったことはないだろうか。

- 「自社のDBを参照しながら回答してほしい」
- 「社内APIを叩いて結果を要約してほしい」
- 「毎回コンテキストを貼り付けるのが面倒すぎる」

プロンプトに情報を貼る方法は、再現性がない。都度コピペが発生する。チームで共有できない。

MCP（Model Context Protocol）はAnthropicが策定したオープンな通信仕様で、Claudeと外部ツールを**標準化されたプロトコル**でつなぐ仕組みだ。自作サーバーを一本立てるだけで、「Claudeが自分のDBを直接参照できる環境」が手に入る可能性がある。

ただ、日本語の実装ドキュメントがほぼ存在しない。公式はTypeScriptだが、型エラーとの格闘で半日が溶けることも珍しくない。筆者自身もそれで消耗した経験がある。

この記事では、その入り口になる2つのサンプルを無料で公開する。

---

## MCPの構造：30秒で掴む

まず全体像を把握しておく。

```
┌─────────────────┐       stdio / SSE        ┌─────────────────────┐
│  Claude Desktop  │ ◀────────────────────▶  │  MCPサーバー (自作) │
│  (MCP Client)    │   JSON-RPC 2.0 over      │  Node.js / Python   │
└─────────────────┘       stdio               └─────────────────────┘
                                                        │
                                              ┌─────────▼──────────┐
                                              │  外部リソース       │
                                              │  DB / API / FS     │
                                              └────────────────────┘
```

ClaudeはMCPクライアントとして動く。自作サーバーをstdio経由で起動しておくと、Claudeが必要に応じて**自動でツールを呼び出す**。

MCPで扱う概念は3つあるが、まず押さえるべきは**Tool**だけでいい。

| 概念 | 役割 |
|---|---|
| **Tool** | Claudeが呼び出せる関数。`name` + `description` + JSON Schemaで定義 |
| Resource | ファイルやURLなどのコンテキスト素材 |
| Prompt | 再利用可能なプロンプトテンプレート |

実用上の8割はToolで完結する。

---

## 【サンプル①】環境セットアップ：package.json + tsconfig.json

まず動かせる環境を作ることが最優先。以下の構成で始めるのが筆者の体験上もっとトラブルが少なかった。

**前提：Node.js 20.x以上、Claude Desktopインストール済み**

```bash
mkdir my-mcp-server && cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node tsx
npx tsc --init
```

**package.json（基本構成）**

```json
{
  "name": "my-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "tsx": "^4.0.0",
    "typescript": "^5.4.0"
  }
}
```

**tsconfig.json（最低限）**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "strict": true
  },
  "include": ["src"]
}
```

`"type": "module"` と `"module": "NodeNext"` の組み合わせが鍵。ここがズレると import エラーが頻発する。

---

## 【サンプル②】雛形①：hello-tool（最小構成）

MCPサーバーの骨格を理解するための最小実装。Claudeに「挨拶して」と言うと、このToolが呼ばれて文字列を返す。

**src/hello-tool.ts**

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// サーバーインスタンスを作成
const server = new McpServer({
  name: "hello-tool",
  version: "1.0.0",
});

// Toolを登録
server.tool(
  "greet",                          // ツール名（Claudeが呼ぶ名前）
  "名前を受け取って挨拶を返す",     // description（Claudeが読む説明文）
  { name: z.string() },             // 入力スキーマ（zodで型定義）
  async ({ name }) => {
    return {
      content: [{ type: "text", text: `こんにちは、${name}さん！` }],
    };
  }
);

// stdioで接続を待ち受け
const transport = new StdioServerTransport();
await server.connect(transport);
```

Claude Desktopの設定ファイル（`claude_desktop_config.json`）に以下を追記すると接続できる。

```json
{
  "mcpServers": {
    "hello-tool": {
      "command": "node",
      "args": ["/absolute/path/to/my-mcp-server/dist/index.js"]
    }
  }
}
```

`dist/index.js` に `hello-tool.ts` をエクスポートする構成にしてビルドすれば、Claude Desktopを再起動した時点でツールが認識される。

---

## サンプルのカスタム方法

`greet` の部分を変えるだけで、どんな処理にも応用できる。たとえば `fetch()` を使えば外部APIを叩いて結果を返すToolになるし、`better-sqlite3` を使えばローカルのSQLiteを参照するToolになる。Claudeへの説明文（`description`）が自然言語なので、「どういう時にこのToolを呼ぶか」をそこに書いておくだけでClaudeが自律的に判断して呼び出してくれる。設計のコツは、Toolをなるべく**一機能一責任**で小さく分けることだ。粒度が細かいほどClaudeの判断精度が上がる傾向がある（筆者の体験として）。

---

### 完全版について

完全版のPDFキット（**Claude MCPサーバー自作キット｜日本語PDF＋雛形4種**）には、以下が収録されている。

- **雛形①** hello-tool（本記事で公開）
- **雛形②** web-fetch（外部APIからデータ取得）
- **雛形③** sqlite-query（ローカルDBを参照）
- **雛形④** notion-search（NotionページをClaudeから検索）
- Claude Desktopへの接続設定の詳細手順
- よくあるエラーと対処法（型エラー・stdio接続失敗・権限エラーなど）
- 自作ツールへの応用テンプレ
- 動作確認チェックリスト

雛形はすべてコピペで動く状態にしてある。日本語解説付きなので、TypeScript初心者でも読み進められる構成を意識した。

気になる方はこちらから確認してみてください。無料部分だけでも参考になれば幸いです。

👉 [Claude MCPサーバー自作キット — Gumroad](https://aoiei.gumroad.com/l/uecpne)