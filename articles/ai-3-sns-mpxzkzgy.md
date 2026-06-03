---
title: "AI自動化で後悔した3つ——SNS凍結・読者離れの記録"
emoji: "📝"
type: "tech"
topics: ["sns", "javascript", "typescript", "claude", "ai"]
published: true
---
# AI自動化で後悔した3つ——SNS凍結・読者離れの記録

## 先に結論を書く

自動化は「やるか・やらないか」ではなく、「どこで止めるか」の問題だった。

Instagram、メルマガ、AI動画。3つで同じ失敗をした。どれも「便利なはずだった」。どれも途中で切り戻すことになった。成功体験は各所で読める。この記事はその逆だ。何を自動化し、どこで止め、今どうしているかを時系列で書く。

---

## 失敗① Instagram DM自動返信——シャドウバン気味になった話

キャプションに「DMください」と書いたとき、届いたDMに自動返信するシステムを実装した。その話。

### 実装内容

Instagram Graph APIのMessaging Webhookを使い、受信→即返信のループを組んだ。

```javascript
// scripts/auto_dm.js
const axios = require('axios');
const express = require('express');
const app = express();
app.use(express.json());

async function sendAutoReply(recipientId, replyText) {
  const url = 'https://graph.facebook.com/v18.0/me/messages';
  const res = await axios.post(
    url,
    {
      recipient: { id: recipientId },
      message: { text: replyText },
      messaging_type: 'RESPONSE',
    },
    { params: { access_token: process.env.INSTAGRAM_ACCESS_TOKEN } }
  );
  return res.data;
}

// Metaのwebhookエンドポイント
app.post('/webhook/instagram', async (req, res) => {
  const entries = req.body.entry || [];
  for (const entry of entries) {
    for (const msg of entry.messaging || []) {
      if (msg.message && !msg.message.is_echo) {
        await sendAutoReply(
          msg.sender.id,
          'ありがとう。詳しくはこちら → https://aoi-ei.com'
        );
      }
    }
  }
  res.sendStatus(200);
});

app.listen(3001);
```

### ハマりどころ

1週間は問題なかった。2週目から、フォロー外アカウントへのDMが届かないケースが増えた。インサイトを確認するとリーチが落ちていた。

Meta公式ポリシーの該当箇所がこれ。

> 「同一テキストを複数のユーザーに送信する行為は、スパムとして扱われる場合があります」

毎回まったく同じ文字列を送り続けていた。これが原因だったと判断している。凍結はしていない。ただ、シャドウバン相当の状態になっていた可能性は高い。

### 止めた判断と今の方法

「届かないDMは存在しないのと同じ」。それに気づいた時点で止めた。

今は投稿とキャプション生成だけ自動化している。DM返信は手動だ。手間は増えた。ただ、届いている。

---

## 失敗② メルマガをAI全任せにしたら解除率が跳ねた

読者600人に向けて毎日送るメルマガを、全文AIで生成して自動配信していた。3週目で止めた。

### 実装内容

OpenAI APIで本文を生成し、WordPress REST API経由でFumi Mail Studioに下書き作成→即送信するバッチを組んだ。

```javascript
// scripts/auto_newsletter.js
const OpenAI = require('openai');
const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function generateNewsletterBody(topic) {
  const completion = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      {
        role: 'system',
        content:
          'AIツール活用を発信するクリエイターとして、読者向けに300字程度のメルマガ本文を書いてください。体言止め・短文を使ってください。',
      },
      { role: 'user', content: `今日のテーマ: ${topic}` },
    ],
  });
  return completion.choices[0].message.content;
}

async function runDailyNewsletter() {
  const topic = '今日から使えるAIツール1つ';
  const body = await generateNewsletterBody(topic);

  const credential = Buffer.from(
    `${process.env.WP_USER}:${process.env.WP_APP_PASSWORD}`
  ).toString('base64');

  await fetch('https://aoi-ei.com/wp-json/fms/v1/newsletters', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${credential}`,
    },
    body: JSON.stringify({
      subject: `AI日報 ${new Date().toLocaleDateString('ja-JP')}`,
      body,
      status: 'publish', // ← ここが問題だった
    }),
  });
}

// 毎朝9時に自動実行
const cron = require('node-cron');
cron.schedule('0 9 * * *', runDailyNewsletter, { timezone: 'Asia/Tokyo' });
```

### ハマりどころ

1週目の解除はほぼなかった。2週目で少し増えた。3週目で明らかに増えた。

配信後の本文を読み返すと、情報として正確だし論理も通っている。ただ、「どこかで読んだことがある文章」に見えた。自分が書いた回と書いていない回を並べたとき、解除のタイミングが一致していた。

自動化が「壊れている」ことに、数値が動くまで気づけなかった。これが一番まずかった。人が書いていれば「今日のは変だな」と気づける。機械が書くと気づかない。

### 止めた判断と今の方法

解除が増えた時点で止めた。仮説で動いた判断だったが、結果は改善した。

今はメルマガの構成・要点出しだけAIに頼む。文章は自分で書く。頻度を毎日から週2〜3本に落とした。筆者の体験として、解除のペースは落ち着いた。

---

## 失敗③ AI動画を毎日量産したら保存数がゼロに張り付いた

HeyGen・Remotion・Whisperのパイプラインで、毎日1本のリールを全自動投稿していた。再生はされた。保存されなかった。

### 実装内容

プロジェクトのメインワークフローをそのまま毎日自動実行していた。

```javascript
// scripts/workflow.js（抜粋）
const { getHeyGenVideoUrl } = require('./heygen');
const { addSubtitles } = require('./add_subtitles');
const { uploadToCloudinary } = require('./upload_cloudinary');
const { postReel } = require('./post_video');
const { generateCaption } = require('./generate_caption');

async function runDailyWorkflow(heygenVideoId, title, theme = 'automation') {
  console.log('[1/5] HeyGen動画URL取得中...');
  const videoUrl = await getHeyGenVideoUrl(heygenVideoId);

  console.log('[2/5] Whisper字幕生成中...');
  const subtitledUrl = await addSubtitles(videoUrl);

  console.log('[3/5] Cloudinaryアップロード中...');
  const cloudinaryUrl = await uploadToCloudinary(subtitledUrl);

  console.log('[4/5] キャプション生成中...');
  const caption = await generateCaption({ title, theme });

  console.log('[5/5] Instagram投稿中...');
  const result = await postReel({ videoUrl: cloudinaryUrl, caption });

  console.log('完了:', result.id);
  return result;
}

module.exports = { runDailyWorkflow };
```

コードは動く。毎日投稿できていた。問題はそこではなかった。

### ハマりどころ

再生数は一定を維持していた。ただ、保存数がほぼゼロになった。

後から量産した動画の内容を並べると共通点があった。原稿がすべてAI生成で、「今日だけ使える情報」ばかりだった。

保存されるのは「後で見返す価値がある動画」だ。「今すぐ見て忘れていい動画」を量産していた。その区別を、量産を始める前に考えていなかった。

### 止めた判断と今の方法

保存数が1週間連続でほぼゼロになった時点で止めた。

週3本に減らした。スクリプトは手書きに戻し、字幕合成・アップロード・投稿の3工程だけ自動化している。本数は減ったが、保存数は回復した。

---

## 3つの失敗に共通していたパターン

時系列で見ると、構造がまったく同じだった。

1. 「ここまで自動化できる」と気づく
2. 一気に全部自動化する
3. 数値が落ちる
4. **どこが原因か分からない**
5. 全部切り戻す

自動化は段階的に入れるべきだった。全部を一度に任せると、何が壊れたか切り分けられなくなる。

もう一つ。「自動化が静かに壊れている状態」が最大のリスクだった。プログラムはエラーを吐かない。ただ、質の悪いアウトプットを出し続ける。気づくのはいつも、数値が動いてからだ。

---

## まとめ

| 自動化した内容 | 何が起きたか | 今の方法 |
|---|---|---|
| Instagram DM自動返信 | 同一テキスト繰り返しでリーチ低下 | DM返信は手動。投稿とキャプションのみ自動 |
| メルマガ全文AI生成+自動配信 | 3週目前後で解除が増加（筆者の体験） | 構成・要点はAI。本文は手書き。週2〜3本に削減 |
| AI動画の毎日自動投稿 | 再生はされるが保存数がゼロに | 週3本。スクリプトは手書き。後工程のみ自動 |

自動化そのものは間違っていない。「全部を任せる」と「どこが壊れたか気づけない状態」がセットでついてくる。それに気づくのが遅かった。

半自動が今のところ最適解だ。「どこを任せ、どこは人が触るか」の設計を先に決めること。あとから切り戻すのは、はじめから分けておくより時間がかかる。

---

蒼井詠の「AIエージェント はじめの30歩」（無料PDF）  
→ https://aoi-ei.com/ai-cheatsheet/

何を自動化すべきで、何を手放してはいけないか。30のチェックポイントとして整理した。

---

非効率は罪よ。あなたもやってみて。