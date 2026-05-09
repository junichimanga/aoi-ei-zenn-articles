---
title: "Claude Code 完全攻略パック【無料サンプル2個公開】"
emoji: "📦"
type: "tech"
topics: ["bundle", "AI", "Claude", "Cursor"]
published: true
---
# Claude Code 完全攻略パック【無料サンプル2個公開】

## この記事でわかること

- `CLAUDE.md` を置くだけで Claude Code の回答がプロジェクトに最適化される理由
- すぐコピペで使える Next.js App Router 向け `CLAUDE.md` テンプレのサンプル
- MCP サーバー自作の最小構成と Claude Desktop への接続手順の概要

---

## Claude Code、ちゃんと使えてる？

Claude Code を起動してコードを貼る。AI が返答する。でも何かズレてる。

そのズレの正体は、**コンテキストの不足**。

Claude Code はポテンシャルが高い。ただし、あなたのプロジェクトのことは何も知らない状態で起動している。フレームワークは何か、命名規則はどうか、テストはどう書くか——これを毎回説明するのは非効率。

解決策は `CLAUDE.md`。

プロジェクトルートに置くだけで、Claude Code が自動で読み込む設定ファイル。「このプロジェクトの憲法」と思ってもらえれば近い。

もう一つの課題は **MCP（Model Context Protocol）**。

Claude Code が外部ツールや DB と連携するための仕組みだが、公式ドキュメントは英語のみ。日本語の実用的な情報はまだ少なく、自作しようとして詰まる人が多い印象がある。

この記事では、そのどちらも「手元で動かせるレベル」まで具体化して共有する。

---

## 無料サンプル① CLAUDE.md テンプレ（Next.js App Router 版）

以下をプロジェクトルートの `CLAUDE.md` にコピーして使う。

```markdown
# CLAUDE.md — Next.js App Router プロジェクト

## プロジェクト概要
- フレームワーク: Next.js 14（App Router）
- 言語: TypeScript（strict モード）
- スタイリング: Tailwind CSS
- 状態管理: Zustand
- テスト: Vitest + Testing Library

## ディレクトリ構成の方針
- `app/`        — ルートとページ（サーバーコンポーネント優先）
- `components/` — 再利用 UI コンポーネント
- `lib/`        — ユーティリティ・API クライアント
- `hooks/`      — カスタムフック（クライアント専用）

## コーディングルール
- コンポーネントは関数宣言（arrow function 禁止）
- `"use client"` は最小範囲に限定する
- Props は inline 型より `interface` を使う
- `any` は禁止。`unknown` + 型ガードで対応

## 命名規則
- コンポーネントファイル: PascalCase（例: `UserCard.tsx`）
- フック: `use` プレフィックス（例: `useUserData.ts`）
- 定数: UPPER_SNAKE_CASE

## テストの方針
- ユーティリティ関数はユニットテストを書くこと
- コンポーネントテストは `userEvent` を使う（`fireEvent` は非推奨）
- `describe` ブロックの粒度: 1 関数 or 1 コンポーネントにつき 1 ブロック

## やってはいけないこと
- `pages/` ディレクトリへの新規ファイル追加
- `getServerSideProps` の新規使用
- `console.log` を本番コードに残す
```

これを置くだけで、Claude Code がプロジェクトの文脈を把握した状態で答えてくれる。「App Router なのに Pages Router の書き方を提案してくる」というズレが減る体験がある。

---

## 無料サンプル② MCP サーバー最小構成（Hello World）

自作 MCP サーバーの最小構成。Node.js + TypeScript で動く。

```typescript
// src/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "my-first-mcp",
  version: "1.0.0",
});

// ツールの定義
server.tool(
  "say_hello",
  "名前を受け取って挨拶を返すツール",
  {
    name: z.string().describe("挨拶する相手の名前"),
  },
  async ({ name }) => {
    return {
      content: [
        {
          type: "text",
          text: `こんにちは、${name}！自作 MCP サーバーから返信しています。`,
        },
      ],
    };
  }
);

// サーバー起動
const transport = new StdioServerTransport();
await server.connect(transport);
```

**Claude Desktop の `claude_desktop_config.json` に追記する設定：**

```json
{
  "mcpServers": {
    "my-first-mcp": {
      "command": "node",
      "args": ["/absolute/path/to/dist/index.js"]
    }
  }
}
```

ビルドして設定ファイルを保存、Claude Desktop を再起動するだけ。`say_hello` ツールが Claude から呼べる状態になる。ここまで動けば、あとは `server.tool()` を追加していくだけで機能を拡張できる。

---

## カスタム方法・活用のポイント

**CLAUDE.md** は「今のプロジェクトの実態」に合わせて更新し続けるのが前提。最初から全項目を埋めようとせず、「Claude に誤った提案をしてほしくないこと」が出てきたら都度追記するスタイルが実用的。チームで使う場合は Git で管理し全員が共有できる状態にしておくと、属人化が消えて新メンバーのオンボーディングコストも下がる傾向がある。

**MCP サーバー**は、上記の Hello World をベースに `server.tool()` を増やしていくだけで機能を拡張できる。社内 DB へのクエリ、Notion API の呼び出し、ファイル操作——Claude から直接呼べるツールとして実装できる。`zod` でスキーマを定義しておくことで、型安全なまま Claude とやり取りできる点が体験として良い。「Claude が外部と繋がる窓口を自分で作れる」という感覚が、使い始めると変わってくる。

---

### 完全版について

この記事は「**Claude Code 完全攻略パック**」の一部を抜粋・再構成したもの。

完全版には以下の 4 コンテンツが収録されている。

| 収録コンテンツ | 内容 |
|---|---|
| MCP サーバー自作キット（日本語 PDF＋雛形 4 種） | Hello / Fetch / SQLite / Notion テンプレを即起動 |
| MCP 自作ハンズオン TypeScript 版 | ゼロ → npm 公開まで完走するキット＋つまずき集 |
| .rules 職人パック 20 選 | Next.js・Rails・FastAPI 等 20 フレームワーク対応 |
| CLAUDE.md テンプレ 20 選 | モノレポ・Python・CLI・業務委託案件など類型別 20 本 |

単品で揃えると ¥6,920 のところ、バンドルで **30% 引き**。

無料部分だけでも参考になれば幸いです。「フレームワーク別の .rules をもっと見たい」「MCP のハンズオンを手元で完走したい」と感じた方は、完全版をご確認ください。

👉 [Claude Code 完全攻略パック（Gumroad）](https://aoiei.gumroad.com/l/qyylad)