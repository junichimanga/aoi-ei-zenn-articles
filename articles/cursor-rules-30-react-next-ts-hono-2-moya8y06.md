---
title: "即効Cursor Rules 30選｜React/Next/TS/Hono別【無料サンプル2個公開】"
emoji: "🤖"
type: "tech"
topics: ["Claude", "AI", "開発ツール", "生産性向上"]
published: true
---
# 即効 Cursor Rules 30選｜React/Next/TS/Hono別【無料サンプル2個公開】

## この記事でわかること

- `.cursorrules` を置くだけで AI が出力するコードの品質がどう変わるか
- スタック（React / Next.js / Hono / Python）ごとのルール設計の考え方
- 今日からコピペで使える Cursor Rules サンプル 2 個

---

## 「惜しいコード」をAIが出し続ける理由

Cursor を使い始めてから、コーディングのスピードが体感として上がる。  
ただ、生成されたコードをよく見ると気になる点が出てくる。

- 全例外を `500` で返している
- 型定義が `any` だらけになっている
- セッション管理がグローバル変数で実装されている

AIが悪いわけではない。**指示が足りていないだけだ。**

`.cursorrules` はその指示書にあたる。プロジェクトのルートに置くだけで、Cursor がコーディング規約を理解した状態でコードを生成するようになる。

「毎回同じことをレビューで指摘している」と感じているなら、それはルール化のサインかもしれない。一度ルールとして書いてしまえば、以後は AI が自動的に避けてくれる可能性がある。

---

## 無料サンプル①：Python エラーハンドリング & ログ設計ルール

📦 **想定プロジェクト**: 本番運用を前提とした FastAPI / Python バックエンド  
🎯 **ねらい**: 例外ハンドラの統一・ログの構造化・エラー情報の適切な秘匿を標準化する

```
# .cursorrules — Python エラーハンドリング & ログ規約

## カスタム例外クラス
  ```python
  class AppError(Exception):
      def __init__(self, message: str, status_code: int = 400, code: str = "APP_ERROR"):
          self.message     = message
          self.status_code = status_code
          self.code        = code
          super().__init__(message)

  class NotFoundError(AppError):
      def __init__(self, resource: str):
          super().__init__(
              f"{resource}が見つかりません", status_code=404, code="NOT_FOUND"
          )

  class UnauthorizedError(AppError):
      def __init__(self):
          super().__init__("認証が必要です", status_code=401, code="UNAUTHORIZED")
  ```

## グローバル例外ハンドラ（main.py に登録）
  ```python
  @app.exception_handler(AppError)
  async def app_error_handler(request: Request, exc: AppError):
      return JSONResponse(
          status_code=exc.status_code,
          content={"error": exc.message, "code": exc.code},
      )

  @app.exception_handler(Exception)
  async def unhandled_error_handler(request: Request, exc: Exception):
      logger.exception("未処理の例外が発生しました")   # スタックトレースはログへ
      return JSONResponse(
          status_code=500,
          content={"error": "サーバーエラーが発生しました"},  # 詳細はクライアントに返さない
      )
  ```

## 構造化ログ（structlog 推奨）
  - logger.info / logger.error にコンテキスト情報を必ず含める
  - ログにパスワード・トークン・APIキーを含めない
  - 本番環境のログレベルは WARNING 以上にする
```

❌ **アンチパターン（ルールなしの AI 出力例）**

```python
# NG: 全例外を 500 で返す・例外詳細をクライアントに露出
@app.get("/users/{id}")
async def get_user(id: str):
    try:
        return await user_service.get(id)
    except Exception as e:
        raise HTTPException(500, detail=str(e))
        # ← DB接続情報・スタックトレースが漏洩する可能性がある
        # ← NotFoundException も 500 になる（正しくは 404）
```

`detail=str(e)` の一行が、本番環境でのセキュリティインシデントの引き金になるケースがある。ルールとして AI に渡しておくと、このパターンを自動的に回避してくれる可能性が高い。

💡 **カスタムポイント**: `structlog` の代わりに `loguru` を使う場合、該当行を `from loguru import logger` に差し替えるだけでそのまま機能する。

---

## 無料サンプル②：プロジェクト構成別 ルール組み合わせ早見表

`.cursorrules` は「全部入り」が正解ではない。スタックと無関係なルールを詰め込むと、AI がコンテキストを無駄に消費して出力の焦点がぼやける場合がある。「必要なルールを、正しい順で積む」という設計が重要だ。

完全版では以下の早見表を収録している。

| # | プロジェクト構成 | 適用ルール番号 | 推奨連結順 |
|---|---|---|---|
| 1 | Next.js + Tailwind + Zod | 01, 03, 06, 07, 08, 11, 13 | 11 → 06 → 07 → 08 → 01 → 03 → 13 |
| 2 | Hono + Cloudflare Workers | 16, 17, 18, 19, 20 | 16 → 17 → 18 → 20 → 19 |
| 3 | React SPA + Zustand | 01, 04, 09, 10, 12 | 09 → 10 → 01 → 04 → 12 |
| 4 | FastAPI + SQLAlchemy | 25, 26, 27, 28, 29, 30 | 25 → 26 → 28 → 29 → 27 → 30 |

「推奨連結順」は後ろのルールが前のルールを上書きするかたちで設計されており、競合が起きにくい順番になっている。新しいルールを追加するときは末尾に足すのが基本的に安全だ。

---

## 使い方とカスタム方法

基本的な使い方は、ルールテキストをコピーしてプロジェクトルートの `.cursorrules` ファイルに貼るだけだ。複数ルールを使う場合は、上の早見表の「推奨連結順」で上から並べて連結する。

カスタムするときは各ルール末尾の **💡 カスタムポイント** を起点にするとよい。ライブラリの差し替えや禁止パターンの追加など、チームのコーディング規約に合わせて書き換えてほしい。

また、ルールはプロジェクトに追加するだけでなく、Cursor の「User Rules」（グローバル設定）に入れておくことで、すべてのプロジェクトに共通で効かせることもできる。個人の書き方の癖に関するルール（変数名・コメントスタイルなど）はグローバル、プロジェクト固有の規約は `.cursorrules` に分けると管理しやすい。

---

### 完全版について

完全版「**即効 Cursor Rules 30選｜React/Next/TS/Hono別**」には、以下のスタックを網羅した 30 ルールをすべて収録している。

- **React / Next.js**（App Router 対応）
- **TypeScript** 型安全設計・Zod バリデーション
- **Hono + Cloudflare Workers**
- **FastAPI / Python** バックエンド
- **テスト設計**（Vitest / Pytest）
- **エラーハンドリング & 構造化ログ**

各ルールにはアンチパターンの例とカスタムポイントを記載しており、コピペしてすぐ試せる形式でまとめている。

無料で紹介した 2 つのサンプルだけでも、今日のコードレビューの手間を少し減らす参考になれば幸いだ。  
「全スタック分まとめて使いたい」と思ったら、下記から確認してみてほしい。

👉 **[Gumroad で完全版を確認する](https://aoiei.gumroad.com/l/cdbhg)**