---
title: "即戦力CLAUDE.md & .cursorrules テンプレ【無料サンプル2個公開】"
emoji: "🤖"
type: "tech"
topics: ["Claude", "AI", "開発ツール", "生産性向上"]
published: true
---
# 即戦力CLAUDE.md & .cursorrules テンプレ20選【無料サンプル2個公開】

## この記事でわかること

- `CLAUDE.md` / `.cursorrules` を設定すると、AIの出力がどう変わるか
- Rails（Hotwire）向けテンプレートの実例とBefore/After比較
- Flutter（Riverpod）向けテンプレートの設計方針と実装例

---

## 「毎回ゼロから書く」をやめる

AI（Claude・Cursor）にコードを頼むとき、何も設定しないまま使っていないだろうか。

設定なしのAIは、プロジェクトのコーディング規約を知らない。Fat Controllerを平気で出す。RuboCopに怒られるコードを書く。Riverpodのベストプラクティスを無視した `StatefulWidget` を量産する。

その都度「〜で書いて」「〜は使わないで」とプロンプトに書き足していく——この繰り返しが積み重なると、想像以上に時間を消費する。

`CLAUDE.md`（ClaudeのProject Knowledge）や `.cursorrules` は、**「毎回言わなくていいルール」をファイルに閉じ込める**仕組みだ。一度書けば、以降は指示なしでAIがプロジェクト規約に沿ったコードを出してくれるようになる。

ただし「何を・どう書くか」で効果の出方が変わる。本記事では、筆者が実運用で使っているテンプレートから2種類を抜粋して公開する。

---

## サンプル1：Ruby on Rails（Hotwire）テンプレート

Rails + Hotwire 構成向けの `CLAUDE.md` 抜粋。

````markdown
# Rails プロジェクト — CLAUDE.md

## 技術スタック
- Ruby 3.3 / Rails 7.2
- Hotwire（Turbo + Stimulus）
- PostgreSQL / Sidekiq / Redis

## コーディング規約
- Fat Controller 禁止 → ビジネスロジックは Service Object に切り出す
- N+1 禁止 → 関連は必ず `includes` で eager load する
- STI（Single Table Inheritance）禁止 → ポリモーフィック関連 or 別テーブル

## よく使うコマンド
```bash
bin/dev                   # Foreman でサーバー起動（JS/CSS ビルド込み）
bundle exec rspec         # テスト全実行
bundle exec rubocop -A    # Lint 自動修正
rails db:migrate          # マイグレーション
rails routes | grep post  # ルート確認
```

## 禁止事項
| NG | 代替 |
|---|---|
| Controller にビジネスロジック | Service Object に切り出す |
| `raw_sql` 直書き | `where` / `Arel` を使う |
| `includes` なし N+1 | 必ず `includes` を書く |
| `params[:post]` 直渡し | Strong Parameters を使う |
````

### Before / After で比較する

同じ「投稿作成のコントローラーを書いて」というプロンプトに対して、設定の有無でAIの出力がどう変わるか。

**設定なし（AI出力）:**

```ruby
def create
  @post = Post.new(params[:post])
  @post.user = current_user
  if @post.save
    redirect_to @post
  else
    render :new
  end
end
```

Strong Parameters なし。Service Object なし。Turbo 非対応。典型的なFat Controllerが出てくる。

**CLAUDE.md あり（AI出力）:**

```ruby
def create
  result = Posts::CreateService.new(user: current_user, params: post_params).call
  if result.success?
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to result.post, notice: "作成しました" }
    end
  else
    @post = result.post
    render :new, status: :unprocessable_entity
  end
end

private

def post_params
  params.require(:post).permit(:title, :content)
end
```

Service Object パターン・Strong Parameters・Turbo Stream 対応が、指示なしで自動的に入った。筆者の体験として、このテンプレートを導入してからレビューコメントの数がかなり減った。

---

## サンプル2：Flutter（Riverpod）テンプレート

````markdown
# Flutter プロジェクト — CLAUDE.md（Riverpod）

## 技術スタック
- Flutter 3.22（Dart 3.4）
- flutter_riverpod 2.5（状態管理）
- go_router 14 / freezed 2 / dio 5

## Widget の分類ルール
| 種類 | 継承元 | 責務 |
|---|---|---|
| Page | `ConsumerWidget` | 画面全体。Provider からデータを受け取り Widget に渡す |
| Widget | `StatelessWidget` | UIのみ。状態を持たない |
| Form | `ConsumerStatefulWidget` | フォームのみ。TextEditingController を持つ |

## NG パターン
- Page 内にビジネスロジックを書かない → Repository / Service に切り出す
- `StatefulWidget` を安易に使わない → Provider で管理できないか先に検討する
- `ref.read` を `build` 内で使わない → `ref.watch` を使う
````

対応する Provider 設計の実例：

```dart
// ✅ Page の例（データ取得はProviderに委譲）
class PostsPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final posts = ref.watch(postsProvider);
    return posts.when(
      data: (data) => PostList(posts: data),
      loading: () => const CircularProgressIndicator(),
      error: (e, st) => ErrorWidget(message: e.toString()),
    );
  }
}
```

```dart
// providers/posts_provider.dart
@riverpod
Future<List<Post>> posts(PostsRef ref) async {
  final repo = ref.watch(postRepositoryProvider);
  return repo.findAll();
}
```

Widget の責務分類をAIに事前に伝えておくことで、`StatefulWidget` の乱用や Page 内へのロジック混入をある程度防げる。特に「NG パターン」のセクションは効果が出やすいと感じている。

---

## 使い方・カスタム方法

テンプレートをそのまま使うよりも、**自分のプロジェクト固有の禁止事項・ディレクトリ構成・よく使うコマンドを書き足す**ことで、より精度が上がる。最低限「技術スタック（バージョン付き）」と「禁止事項」の2セクションがあるだけでも、AIが出す初手のコードの一貫性が変わってくる。

配置場所は、Cursorなら `.cursorrules`、Claudeなら Project Knowledge の `CLAUDE.md`。どちらも同じ内容で流用できることが多い。

---

### 完全版について

本記事で紹介した2種類に加え、**20種類のテンプレートを収録したドキュメント**を販売している。

収録テーマの一部：

- Next.js App Router / TypeScript
- Python（FastAPI）
- Go（Echo）
- Supabase + RLS 設計
- セキュリティ共通ルール（全テンプレート共通）

各テンプレートにはコーディング規約・禁止事項・よく使うコマンド・Before/After が含まれており、プロジェクト開始時にコピーしてすぐ使える形になっている。

👉 [即戦力CLAUDE.md & .cursorrules テンプレ20選 — Gumroad](https://9157725409008.gumroad.com/l/owkiwk)

無料部分だけでも、日々のAI開発の参考になれば幸いです。