---
title: "Cursor Full Pack — 70本のRules完全装備【無料サンプル2個公開】"
emoji: "📦"
type: "tech"
topics: ["bundle", "AI", "Claude", "Cursor"]
published: true
---
# Cursor Full Pack — 70本のRules完全装備【無料サンプル2個公開】

## この記事でわかること

- `.cursorrules` / `CLAUDE.md` がなぜ補完精度に直結するのか
- Next.js（TypeScript）向けルールの実例と書き方のコツ
- FastAPI 向けルールの実例と、チーム・個人運用への応用方法

---

## 「とりあえずCursorを入れた」状態で止まっていないか

CursorはAIコーディングツールとして補完精度が高いと評価されることが多い。しかし、インストールして終わりにしているケースは思った以上に多い。

補完精度はルールファイルの有無で体感が変わる。プロジェクトルートに `.cursorrules`（または `.cursor/rules/*.mdc`）を置くだけで、Cursorは「このプロジェクトの文脈」を把握した状態でコードを生成し始める。

たとえばこんな宣言が効く。

- 「`any` を使わず、型を明示してほしい」
- 「コンポーネントは `export default` ではなく named export で」
- 「テストは Vitest で書く。Jest は使わない」

これらをルールとして宣言しておくと、毎回プロンプトで指示しなくてすむ。逆に言えば、ルールがない状態は「毎回口頭で仕様を説明しなければならないメンバーが常駐している」状態に近い。非効率以外の何物でもない。

問題は、スタックごとに最適なルールをゼロから書くのに時間がかかることだ。英語圏のリソース（cursor.directory など）を参考にしても、日本語コメントがなかったり、自分のプロジェクト構成に合わなかったりする。

---

## 無料サンプル① Next.js / TypeScript 用 `.cursorrules`

```
# Project: Next.js 15 App Router + TypeScript

## コンポーネント設計
- React コンポーネントは必ず named export にする（export default 禁止）
- コンポーネントファイル名は PascalCase（例: UserCard.tsx）
- ロジックを含む場合は hooks/ に切り出し、コンポーネントは UI のみにする

## TypeScript
- `any` 型の使用禁止。unknown を経由する
- 型定義は types/ 配下の .ts ファイルに集約する
- as を使ったキャストは最終手段。型ガードを優先する

## ファイル構成（App Router）
- ページは app/ 配下の page.tsx
- レイアウトは layout.tsx
- サーバーコンポーネントをデフォルトとし、
  インタラクションが必要な箇所のみ "use client" を付与する

## スタイリング
- Tailwind CSS を使用する
- CSS Modules は使わない
- クラス名が長くなる場合は cn()（clsx + tailwind-merge）を使う

## テスト
- テストフレームワークは Vitest + Testing Library
- __tests__/ 以下にテストファイルを置く
- ファイル名は *.test.tsx

## その他
- console.log を本番コードに残さない
- TODO コメントには担当者名と日付を付ける
  （例: // TODO(yamada 2025-06-01): ~~）
```

このルールは筆者がNext.jsプロジェクトで実際に使っているベースをもとにしている。`any` 禁止やnamed exportのような「チームで揉めがちな選択」をルール化しておくと、コードレビューの指摘が体感として減りやすい。

---

## 無料サンプル② FastAPI 用 `CLAUDE.md`

Claude CodeやCursorは、プロジェクトルートの `CLAUDE.md` も読み込む。`.cursorrules` と同様に、プロジェクトの規約を事前に伝えることができる。

```markdown
# Project: FastAPI + Python 3.12

## アーキテクチャ
- ルーター: routers/ 配下でリソース単位に分割（users.py, items.py など）
- ビジネスロジック: services/ に切り出す
- DB アクセス: repositories/ でまとめる（SQLAlchemy 2.0 使用）
- スキーマ: schemas/ に Pydantic モデルを配置

## コーディング規約
- 型アノテーションを必ず付ける（mypy strict モード準拠）
- 関数・クラスには docstring を書く（Google スタイル）
- `Optional[X]` ではなく `X | None` を使う（Python 3.10+ 構文）
- 例外は HTTPException を使い、詳細メッセージを含める

## 非同期処理
- DB アクセスを含む関数は `async def` を使う
- ブロッキング処理は `asyncio.to_thread()` でラップする

## テスト
- pytest + httpx.AsyncClient を使う
- フィクスチャは conftest.py に集約
- テスト DB はインメモリ SQLite を使う

## セキュリティ
- 認証は JWT（python-jose）
- パスワードハッシュは bcrypt
- 環境変数は pydantic-settings で管理。os.environ を直接使わない
```

FastAPIはプロジェクト構成の自由度が高い分、チームによって書き方が大きく異なりやすい。このCLAUDE.mdをプロジェクトルートに置くと、AIに「このプロジェクトの流儀」を事前に伝えられるため、提案されるコードのスタイルが安定しやすくなる。

---

## ルールファイルの使い方とカスタマイズ

`.cursorrules` はプロジェクトルートに置くだけで機能する。Cursor 0.43以降は `.cursor/rules/` ディレクトリ内に複数の `.mdc` ファイルとして分割管理することもできる。スコープ指定が可能になり、ファイルの種類ごとにルールを切り替えられるのが利点だ。

カスタマイズの基本的なアプローチは「禁止事項より推奨事項を先に書く」こと。ネガティブな指示だけが並ぶルールより、「こう書いてほしい」という正例を先に示す方が、筆者の体験では補完結果が意図に近くなりやすかった。

チームで運用する場合は、`.cursorrules` をGitで管理して全員が同じルールを共有する方法が運用しやすい。新しいメンバーがジョインしたとき、環境セットアップと同時にAIへの指示も揃う状態になる。

---

### 完全版について

今回紹介したのはサンプルの一部にすぎない。**Cursor Full Pack** では以下の3シリーズをまとめて収録している。

| シリーズ | 内容 |
|---|---|
| 即効Cursor Rules 30選 | React / Next.js / TypeScript / Hono スタック別 30本 |
| .rules 職人パック 20選 | Next.js / Rails / FastAPI / Flutter など 20フレームワーク |
| 即戦力テンプレ 20選 | モノレポ / Python / CLI / 業務委託案件など類型別 20本 |

合計70本以上のルールを一括で入手できる。スタックが複数あるほど、ゼロから書くコストを省けるセット構成になっている。

👉 [Cursor Full Pack をGumroadで見る](https://aoiei.gumroad.com/l/xeytm)

無料部分のサンプルだけでも、自分のプロジェクトに合わせてカスタマイズするヒントになれば幸いです。