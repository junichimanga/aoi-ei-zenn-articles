---
title: "Instagram自動投稿、30日試した運用ログ"
emoji: "📝"
type: "tech"
topics: ["instagram", "javascript", "typescript", "claude", "ai"]
published: true
---
# Instagram自動投稿、30日試した運用ログ

## 結論から言う

毎日自動で投稿しても、見られなければ意味がない。

これが30日でわかった、最初に知っておくべきことだ。

パイプラインは完成した。HeyGenでAIアバター動画を生成し、Whisperで字幕を起こし、Remotionで合成して、Graph APIで自動投稿する。コマンド1本で全部動く。技術的には、正しく動いている。

なのに、フォロワーはほぼ動かなかった。

成功談を書くつもりはない。30日分の観測データをそのまま素材にして、「なぜ伸びなかったのか」を仮説として整理する。同じパイプラインを組もうとしている人に、先に知っておいてほしいことがある。

---

## パイプラインの構成

使ったスタックは以下のとおり。

| 役割 | ツール |
|---|---|
| AIアバター動画生成 | HeyGen（Web画面で手動生成） |
| 字幕タイムスタンプ | OpenAI Whisper API |
| 字幕合成・動画レンダリング | Remotion |
| 動画ホスティング | Cloudinary |
| 投稿 | Instagram Graph API |

CLIから1コマンドで実行できる。

```bash
npm run workflow -- "https://app.heygen.com/videos/<ID>" "タイトル" automation "トピック"
```

内部では以下の順で処理が走る。

1. HeyGen APIで動画MP4直リンクを取得
2. Whisperで音声 → 字幕タイムスタンプ変換
3. RemotionでカラオケReel字幕を合成
4. Cloudinaryに動画をアップロード
5. Instagram Graph APIでReel投稿
6. WordPressに下書き保存

---

## 主要スクリプトの実装

### HeyGen 動画URL取得

```js
// scripts/heygen.js（抜粋）
import fetch from "node-fetch";

export async function getHeyGenVideoUrl(videoId) {
  const res = await fetch(
    `https://api.heygen.com/v1/video_status.get?video_id=${videoId}`,
    {
      headers: {
        "X-Api-Key": process.env.HEYGEN_API_KEY,
        Accept: "application/json",
      },
    }
  );
  const json = await res.json();
  if (json.data?.status !== "completed") {
    throw new Error(`動画がまだ完成していません: ${json.data?.status}`);
  }
  return json.data.video_url;
}
```

### Whisper 字幕タイムスタンプ取得

```js
// scripts/add_subtitles.js（抜粋）
import OpenAI from "openai";
import fs from "fs";

export async function getSubtitleTimestamps(localVideoPath) {
  const client = new OpenAI();
  const transcription = await client.audio.transcriptions.create({
    file: fs.createReadStream(localVideoPath),
    model: "whisper-1",
    response_format: "verbose_json",
    timestamp_granularities: ["word"],
    language: "ja",
  });
  return transcription.words; // [{ word, start, end }]
}
```

### Instagram Reel 投稿

```js
// scripts/post_video.js（抜粋）
import fetch from "node-fetch";

export async function postInstagramReel({ videoUrl, caption }) {
  const accountId = process.env.INSTAGRAM_BUSINESS_ACCOUNT_ID;
  const token = process.env.INSTAGRAM_ACCESS_TOKEN;

  // Step 1: コンテナ作成
  const containerRes = await fetch(
    `https://graph.facebook.com/v19.0/${accountId}/media`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        media_type: "REELS",
        video_url: videoUrl,
        caption,
        access_token: token,
      }),
    }
  );
  const { id: containerId } = await containerRes.json();

  // Step 2: 公開
  const publishRes = await fetch(
    `https://graph.facebook.com/v19.0/${accountId}/media_publish`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        creation_id: containerId,
        access_token: token,
      }),
    }
  );
  return publishRes.json();
}
```

---

## 30日の観測データ

数字はそのまま出す。ただし、これは**筆者のアカウント環境（規模・投稿頻度・テーマ）での観測値**だ。再現性を保証するものではない。

### 再生数の推移

| 期間 | 傾向 |
|---|---|
| 1〜10日目 | 既存フォロワー中心に再生。新規流入はほぼなし |
| 11〜20日目 | 2本、発見タブに乗ったと思われる急増あり |
| 21〜30日目 | 急増後は元の水準に戻る。連続性なし |

### 保存率

保存数÷再生数の比率を追った。自動化コンテンツよりも、テキスト主体の「ノウハウ整理系」投稿で保存が発生しやすい傾向を観測した（条件：AIツール解説ジャンル、筆者のアカウント）。

### フォロワー推移

30日で微増。ただし、増加のタイミングは投稿本数と連動していなかった。投稿数ではなく、特定の1本が引き起こしていた。

---

## ハマりどころ：伸びなかった本当の理由

パイプラインを組んで気づいたのは、技術の問題ではなかった。

### 理由1：コンテンツが「届いた後」を設計していなかった

自動化は「投稿まで」を解決する。しかしInstagramのアルゴリズムが評価するのは、投稿後の行動だ。保存、コメント、シェア。これらが発生するコンテンツ設計がなければ、自動で投稿し続けても数字は動かない。

最初の2週間、キャプションは自動生成したままだった。ハッシュタグは追加していた。でも「何を保存させるか」を考えていなかった。これが最大の敗因だと思っている。

### 理由2：発見タブへの初速が足りなかった

Reelが既存フォロワー以外に届くには初速が必要だという仮説を持っている（根拠：複数のアカウント運用事例の観察）。投稿後30分〜1時間の保存・シェアが少ないと、アルゴリズムに広げてもらえない状態になるようだ。

自動投稿の弱点は、この「初速を作る動き」がゼロな点だ。投稿してそのまま。ストーリーでの告知もない。アルゴリズムに判断材料を渡していなかった。

### 理由3：動画の冒頭1秒を甘く見ていた

HeyGen × 字幕合成の動画は品質が高い。しかし、最初の1秒でスクロールを止める「引き」がなかった。

サムネイルとして機能する冒頭フレームを意識的に設計していなかった。技術的な完成度と、視聴者が止まる理由は別物だった。

### 理由4：Whisperの字幕に無校正で投稿していた

コスト自体は問題ない。90秒動画1本あたりのWhisperコストは筆者の環境で約1円。月30本で約30円。

ただし、音声品質によって字幕精度が落ちるケースがあることに気づくのに時間がかかった。HeyGenの出力音声は概ね良好だが、一部の語尾や固有名詞でズレが生じる。自動チェックフローを入れていなかった。

```js
// 字幕ズレを検出する簡易チェック（追加した）
function validateSubtitles(words) {
  const suspiciousPatterns = [/[ｦ-ﾟ]/, /\s{3,}/]; // 半角カナ・連続空白を警告
  return words.filter(({ word }) =>
    suspiciousPatterns.some((re) => re.test(word))
  );
}
```

---

## 今後の仮説

パイプラインは維持する。ここを捨てる理由はない。変えるのは以下だ。

- **コンテンツ設計**：「何を保存させるか」を最初に決める
- **投稿後の動き**：自動投稿後30分、ストーリーで告知する運用を追加
- **冒頭1秒**：テキストアニメーションより静止画テロップを先行させる
- **字幕チェック**：Whisper出力に簡易校正フローを組み込む

これらは仮説段階だ。31日目以降の観測で検証する。

---

## まとめ

自動化パイプラインは、動いた。コマンド1本でHeyGenからInstagramまでつながる仕組みは完成している。

伸びなかったのは、パイプラインではなく、パイプラインの出口にあるコンテンツの問題だった。

自動化は時間を節約する。でも、その節約した時間を「コンテンツ設計」に使わなければ意味がない。

システムより先に、設計を変えること。順番、間違えてる人多すぎ。

非効率は罪。あなたもやってみて。

---

> 蒼井詠の「AIエージェント はじめの30歩」（無料PDF）
> → https://aoi-ei.com/ai-cheatsheet/