# 生成AI選び診断ツール 設計メモ

作成日: 2026-06-13

## 概要
「素人（AI初心者）が自分に合った生成AIを見つけられるチャットボット」。
ジャンル選択（LLM／画像生成／動画生成／コーディング）→診断クイズ→おすすめAI表示、という流れの
**汎用診断フレームワーク**を作り、**LLM・画像生成・動画生成・コーディングの4カテゴリすべての実データを完成**させる。

## 形式
- **1ファイル完結のHTML**（`ai-shindan.html`、リポジトリルートに新規作成）
- `ai-battle-stadium.html` と同方針：サーバー・DB不要、完全無料、Vanilla JS
- localStorage不要（v1では診断結果の保存は行わない）

## デザイントーン
`ai-niigata-lp.html` のテイストを継承：
- 配色：ダークネイビー（`--dark: #0b1d26`）×ゴールド（`--gold: #fbd784`）
- フォント：Montserrat / Playfair Display / IBM Plex Sans JP
- オーロラ背景などの装飾は流用可だが、カスタムカーソル等の重い演出は省略し診断UI向けに軽量化する

## 全体データ構造（4カテゴリ対応の汎用エンジン）
```js
CATEGORIES = {
  llm:    { label: "文章で相談・LLM", questions: [...], services: [...], footerNote: "..." }, // フル実装
  image:  { label: "画像生成", questions: [...], services: [...], footerNote: "..." },        // フル実装
  video:  { label: "動画生成", questions: [...], services: [...], footerNote: "..." },        // フル実装
  coding: { label: "コーディング", questions: [...], services: [...], footerNote: "..." },     // フル実装
}
```
診断エンジン（質問を1問ずつ表示→回答収集→スコアリング→結果表示）は4カテゴリ共通の汎用実装。

## 画面フロー
1. **ジャンル選択画面**：4枚のカード（LLM／画像生成／動画生成／コーディング）。
   各カードをタップすると、対応するカテゴリの診断画面に進む。
2. **診断画面**：チャット風UIで1問ずつ選択肢を表示。進捗ドット（カテゴリの質問数に応じて表示。LLM・画像生成は1/4〜4/4、動画生成・コーディングは1/5〜5/5）。
3. **結果画面**：1位のおすすめAIを大きく表示＋理由、2位・3位を「こちらもおすすめ」で小さく表示。
   「もう一度診断する」ボタンでジャンル選択に戻る。

## LLM診断: 質問設計（4問）

**Q1. 主に何に使いたい？**（用途タグ・単一選択）
| 選択肢 | タグ |
|---|---|
| 文章作成・メールの下書き・要約 | `writing` |
| 最新ニュースや調べ物のリサーチ | `research` |
| プログラミング・コードを書く/直す | `coding` |
| アイデア出し・創作・なんでも相談 | `creative` |
| 画像・写真・PDFなどを読み込んで質問 | `vision` |
| 仕事の資料作成（Word/Excel/PowerPoint） | `office` |

**Q2. 予算は？**
- A: 無料で十分
- B: 月1,000〜3,000円くらいまで
- C: しっかり課金してOK（本格利用）

**Q3. 一番重視するのは？**
- A: 賢さ・答えの精度
- B: 返事の速さ・テンポ
- C: 無料でどれだけ使えるか
- D: 長い文章・資料を一度に扱える

**Q4. プライバシー・データの取り扱いは気になりますか？**
- A: とても気になる（個人情報や仕事の話を入れることが多い）
- B: それほど気にしない（一般的な調べ物や雑談が中心）

## LLM診断: データセット（7サービス）

各サービスは以下のフィールドを持つ：
`name, officialUrl, freePlan, paidPlans, beginnerFriendly, tags[], scores{intelligence, speed, value, longContext}, reasonTemplates{...}, risk`

`reasonTemplates` の構造は次の3グループで、それぞれ短い1文（日本語）を持つ：
- `usage`: そのサービスが持つ `tags` ごとに1文（例: `coding: "コードの自動修正・複数ファイル理解に強く..."`）
- `budget`: Q2の選択肢A/B/Cごとに1文（例: `B: "Plusプラン(¥3,000程度)でGPT-5.5まで使えます"`）
- `priority`: `scores` の4軸（intelligence/speed/value/longContext）ごとに1文

結果画面では、Q1のタグが `usage` にあればその文を、なければ「オールラウンドに使えます」的な汎用文を使用。
Q2/Q3は対応するキーの文をそのまま使う。

| サービス | 公式URL | 無料でできること | 有料プラン目安 | 初心者向け |
|---|---|---|---|---|
| **ChatGPT** | chatgpt.com | GPT-5.3 Instantが5時間10回まで（米国は広告あり）、超過後はMiniに切替 | Plus ¥3,000程度/月（GPT-5.5利用可）／Pro $100〜200/月 | ◎ 完全日本語・全端末対応 |
| **Claude** | claude.ai | 5時間で約10〜20回 | Pro ¥3,000程度/月（+税、Claude Fable 5/Opus 4.8）／Max ¥15,000〜30,000/月 | ◎ 2026年に日本語が大幅向上 |
| **Gemini** | gemini.google.com | 完全無料、Gemini 3.5 Flash等（使用量上限あり） | AI Plus ¥1,200／AI Pro ¥2,900／AI Ultra ¥14,500〜32,000 | ◎ Google連携・アプリ充実 |
| **Grok** | grok.com | 25回/2時間、画像生成20回/日 | SuperGrok ¥4,500程度/月、Heavy $300/月 | ○ X連携が強み |
| **Perplexity** | perplexity.ai | Pro Search 3-5回/日、Deep Research 5回/日 | Pro ¥3,000/月、Max $200/月 | ◎ 出典付きで分かりやすい |
| **Copilot** | copilot.microsoft.com | M365があれば追加費用なしでチャット利用可 | Copilot Pro ¥1,490〜3,200/月 | ◎ Office統合 |
| **DeepSeek** | chat.deepseek.com | 実質無制限、最新V4 Proも無料 | 個人向け有料プランなし | △ 中国製AIで情報管理に注意 |

**用途タグ（Q1マッチング用）**

| サービス | tags |
|---|---|
| ChatGPT | writing, research, coding, creative, vision |
| Claude | coding, creative, writing, vision |
| Gemini | writing, research, vision, creative |
| Grok | research |
| Perplexity | research |
| Copilot | office, writing |
| DeepSeek | coding, writing |

**特性スコア（Q3マッチング用、1〜5）**

| サービス | 賢さ(intelligence) | 速さ(speed) | コスパ(value) | 長文対応(longContext) |
|---|---|---|---|---|
| ChatGPT | 5 | 3 | 2 | 4 |
| Claude | 5 | 3 | 2 | 5 |
| Gemini | 4 | 5 | 5 | 5 |
| Grok | 4 | 5 | 3 | 4 |
| Perplexity | 4 | 4 | 3 | 3 |
| Copilot | 4 | 3 | 4 | 4 |
| DeepSeek | 3 | 3 | 5 | 5 |

**リスク・注意点（結果画面に表示、各1〜2行）**

| サービス | リスク・注意点 |
|---|---|
| ChatGPT | 会話が学習に使われる設定がデフォルトON（設定でオプトアウト可）。削除後も30日保持。2026年に複数訴訟が話題。 |
| Claude | 個人プランは学習利用がデフォルトオプトイン（最大5年保持、オプトアウト可）。2026年6月、安全性レビュー対象の会話は設定に関わらず学習利用される場合あり。 |
| Gemini | 無料は学習利用がデフォルトON（オフ可、Temporary Chatで72時間削除）。2026年、悪用・訴訟事例が報道。 |
| Grok | デフォルトでオプトイン、オプトアウトは2箇所設定が必要。画像生成の悪用でEU調査対象、モデレーションが緩い傾向。 |
| Perplexity | 無料・有料ともデフォルト学習利用。未ログインはオプトアウト不可。出典の正確性に課題の報告も。 |
| Copilot | 個人版はデフォルト学習利用（オフ可）。組織/M365 Family内/18歳未満は対象外。過剰モデレーションへの不満も話題。 |
| DeepSeek | データを中国国内に保存・処理。国家情報法により政府がデータ提供要求可。複数国で利用制限。政治的話題は検閲。個人情報の入力は非推奨。 |

## スコアリングロジック

```
score(service) =
    (service.tags に Q1タグを含む ? +3 : 0)
  + budgetScore(Q2, service)        // 下記
  + service.scores[Q3で選んだ軸]     // 1〜5
  + privacyAdjust(Q4, service)      // 下記
```

- `budgetScore`:
  - Q2=A（無料で十分）: `service.scores.value` を加算（無料の充実度をそのまま反映）
  - Q2=B（月1,000〜3,000円）: 有料プランが概ね¥3,500以下に収まるサービスは +3、それ以外（Grok/Perplexity Max等）は +1
  - Q2=C（しっかり課金OK）: 上位プラン（Max/Ultra/Heavy等）を持つサービスは +3、それ以外は +1
  - 有料プランを持たないサービス（DeepSeek）は、Q2の回答に関わらず `service.scores.value` を加算する（無料が前提のため、課金意欲との比較対象にはしない）
- `privacyAdjust`:
  - Q4=A（とても気になる）: DeepSeekは `-10`（事実上トップ3から除外）。他サービスは `0`
  - Q4=B: 全サービス `0`

全7サービスのスコアを計算し降順ソート。上位3件を結果表示（同点時は元の配列順を優先）。

---

## 画像生成診断: 質問設計（4問）

**Q1. 何を作りたい？**（用途タグ・単一選択）
| 選択肢 | タグ |
|---|---|
| イラスト・キャラクターを描きたい | `illustration` |
| 写真みたいにリアルな画像を作りたい | `photo` |
| ロゴ・アイコン・デザイン素材を作りたい | `logo` |
| SNS用の画像・サムネイルを作りたい | `social` |
| ポスターや看板など、文字入りの画像を作りたい | `textimage` |
| 商品・広告などビジネス向けの画像を作りたい | `business` |

**Q2. 予算は？**（LLMと同じ選択肢を再利用）
- A: 無料で十分
- B: 月1,000〜3,000円くらいまで
- C: しっかり課金してOK（本格利用）

**Q3. 一番重視するのは？**
- A: 画質・クオリティ
- B: 生成スピード
- C: コスパ・無料枠の多さ
- D: 日本語の指示でもうまく作れるか

**Q4. 生成した画像を商用・仕事で使いたい？**
- A: はい、商用・仕事で使う予定がある
- B: いいえ、個人利用（趣味・SNS投稿など）が中心

## 画像生成診断: データセット（7サービス）

各サービスのフィールドはLLMと同様：
`name, officialUrl, freePlan, paidPlans, beginnerFriendly, tags[], scores{quality, speed, value, japanese}, reasonTemplates{usage, budget, priority, commercial}, risk`

LLMとの違いは `scores` の4軸が `quality/speed/value/japanese`（Q3に対応）であること、
および `reasonTemplates` に **`commercial`**（Q4=A/Bごとの1文）が追加されていること。

選定方針：ユーザーが提示した Text-to-Image Leaderboard（GPT Image 2 / Nano Banana / grok-imagine-image /
MAI-Image / Krea 2 / Recraft V4.1 / ImagineArt 2.0 等）に掲載されているモデルを使える消費者向けサービスのみを採用。
NVIDIA Cosmos・FLUX.2（Black Forest Labs単体）・ERNIE等は開発者向けAPIが中心で消費者向けアプリが弱いため対象外。

| サービス | 公式URL | 無料でできること | 有料プラン目安 | 初心者向け |
|---|---|---|---|---|
| **ChatGPT**（GPT Image 2） | chatgpt.com | 1日2〜3枚程度（上限に達すると翌日まで待ち） | Plus ¥3,000/月で1日15枚程度／Pro ¥16,800〜30,000/月で実質無制限 | ◎ チャットで日本語完結、文字入り画像・インフォグラフィックが得意 |
| **Gemini**（Nano Banana） | gemini.google.com | 1日100枚程度（Nano Banana 2、一部4K対応） | AI Plus ¥725／AI Pro ¥2,900／AI Ultra ¥14,500〜32,000（1日1,000枚程度） | ◎ 日本語対応良好、無料枠が圧倒的に多くアプリも充実 |
| **Grok Imagine** | grok.com | 無料プランなし（2026年3月廃止） | SuperGrok Lite $10／SuperGrok $30 | △ UIはシンプルだが無料体験は不可 |
| **Microsoft Designer**（MAI-Image） | designer.microsoft.com | 月15クレジット（M365契約者は60クレジット） | Copilot Pro ¥2,990/月で1日100ブースト | ○ Bing Image Creator系で使いやすい |
| **Krea AI**（Krea 2） | krea.ai | 100 compute units/日（リアルタイム生成。※生成物は商用利用不可） | Basic $9/月（5,000units・商用利用可）／Pro $35／Max $70／Business $200 | ○ リアルタイム生成キャンバスが直感的。日本語プロンプトは公式非対応（翻訳推奨） |
| **Recraft**（Recraft V4.1） | recraft.ai | 毎日30〜50クレジット程度（永続無料。※生成物は商用利用不可・公開扱い） | Basic $10（1,000クレジット）／Advanced $27／Pro $48／Teams $55/席 | ◎ 日本語プロンプト対応が強み、Figma的UIで操作が直感的 |
| **ImagineArt** | imagine.art | 毎日100クレジット付与（クレカ不要。※生成物はデフォルトで公開） | Basic ¥2,500程度／Standard $30／Ultimate $50／Creator $250 | ○ 操作はシンプル・30以上のモデルを切替可能だが、iOSアプリは英語UIのみとの報告あり |

**用途タグ（Q1マッチング用）**

| サービス | tags |
|---|---|
| ChatGPT | textimage, business, illustration, social |
| Gemini | photo, textimage, business, social |
| Grok Imagine | photo, logo, textimage, illustration |
| Microsoft Designer | illustration, logo, social, textimage |
| Krea AI | photo, illustration |
| Recraft | logo, business, textimage, social |
| ImagineArt | photo, textimage, social |

**特性スコア（Q3マッチング用、1〜5）**

| サービス | 画質(quality) | スピード(speed) | コスパ(value) | 日本語対応(japanese) |
|---|---|---|---|---|
| ChatGPT | 4 | 4 | 2 | 5 |
| Gemini | 5 | 4 | 5 | 5 |
| Grok Imagine | 4 | 4 | 1 | 3 |
| Microsoft Designer | 3 | 3 | 4 | 3 |
| Krea AI | 4 | 5 | 3 | 2 |
| Recraft | 4 | 3 | 3 | 5 |
| ImagineArt | 4 | 4 | 4 | 2 |

**リスク・注意点（結果画面に表示、各1〜2行）**

| サービス | リスク・注意点 |
|---|---|
| ChatGPT | 著作権を巡る訴訟が複数進行中（2026年中に判断の可能性）。既存キャラ・ロゴに似た画像の生成は権利侵害リスクがあり自己責任。 |
| Gemini | 個人利用は商用可だが、Geminiアプリ経由のビジネス利用には著作権侵害の補償なし（Workspace/Vertex AI経由が規約上推奨）。 |
| Grok Imagine | 2026年3月に無料プラン廃止。ディープフェイク問題でEU/UK等が規制調査中、コンテンツポリシーが頻繁に変更される。 |
| Microsoft Designer | 2023年11月以降商用利用可だが、第三者の商標・著名人に似た画像の生成は権利侵害リスクがあり注意。 |
| Krea AI | 無料プランの生成物は商用利用不可・ライセンスなし（商用利用には有料プランが必須）。日本語プロンプトは公式非対応。 |
| Recraft | 無料プランは生成物の所有権がRecraft側にあり、全て公開扱い・商用利用不可。著作権侵害時の責任は利用者負担。 |
| ImagineArt | 無料プランの生成物はデフォルトで公開（コミュニティ公開）。iOSアプリは英語UIのみで日本語対応に課題の報告あり。 |

## 画像生成: スコアリングロジック

```
score(service) =
    (service.tags に Q1タグを含む ? +3 : 0)
  + budgetScore(Q2, service)
  + service.scores[Q3で選んだ軸]
  + commercialAdjust(Q4, service)
```

- `budgetScore`:
  - Q2=A（無料で十分）: `service.scores.value` を加算
  - Q2=B（月1,000〜3,000円）: 全サービスに¥3,000以下のエントリー有料プランがあるため、全サービス +3
  - Q2=C（しっかり課金OK）: エントリーを大きく超える上位プラン（Pro/Ultra/Premium等）を持つサービスは +3、エントリーのみで上位プランがないサービス（Microsoft Designer）は +1
- `commercialAdjust`:
  - Q4=A（商用・仕事で使う）: Grok Imagine `-3`（規制リスク）、ImagineArt `-1`（無料生成物は公開されるため商用利用には有料プラン必須）、Microsoft Designer `+1`、ChatGPT/Gemini/Krea AI/Recraft `0`
  - Q4=B（個人利用）: 全サービス `0`

全7サービスのスコアを計算し降順ソート。上位3件を結果表示（同点時は元の配列順を優先）。

---

## 動画生成診断: 質問設計（5問）

**Q1. どんな動画を作りたい？**（用途タグ・単一選択）
| 選択肢 | タグ |
|---|---|
| SNS用の短い動画・ショート動画を作りたい | `social` |
| アニメ・イラスト風の動画を作りたい | `illustration` |
| 写真みたいにリアルな実写風の動画を作りたい | `photo` |
| 商品紹介・広告などビジネス向けの動画を作りたい | `business` |
| ストーリー性のある複数シーンの映像を作りたい | `story` |
| セリフ・音声・BGM入りの動画を作りたい | `audio` |

**Q2. 予算は？**（LLM・画像生成と同じ選択肢を再利用）
- A: 無料で十分
- B: 月1,000〜3,000円くらいまで
- C: しっかり課金してOK（本格利用）

**Q3. 一番重視するのは？**
- A: 画質・クオリティ
- B: 生成スピード
- C: コスパ・無料枠の多さ
- D: セリフ・音声・BGMまで自動で作れるか

**Q4. 実在する人物の顔を使った動画を作りたい？**
- A: はい（自分や友人の写真、似た人物を動画に使いたい）
- B: いいえ（人物の顔は使わない・気にしない）

**Q5. 作った動画を商用・仕事で使いたい？**
- A: はい、商用・仕事で使う予定がある
- B: いいえ、個人利用（趣味・SNS投稿など）が中心

## 動画生成診断: データセット（7サービス）

各サービスのフィールドはLLM・画像生成と同様：
`name, officialUrl, freePlan, paidPlans, beginnerFriendly, tags[], scores{quality, speed, value, audio}, reasonTemplates{usage, budget, priority, likeness, commercial}, risk`

画像生成との違いは、`scores` の4軸目が **`audio`**（セリフ・音声・BGM生成対応、Q3に対応）であること、
リスク軸が2つに分かれ `reasonTemplates` に **`likeness`**（Q4=A/Bごとの1文）と **`commercial`**（Q5=A/Bごとの1文）が追加されていること。

選定方針：ユーザーが提示したText to Video Leaderboard（With Audio）に掲載されているモデルを使える消費者向けサービスのみを採用。
Dreamina（Seedance 2.0、1位）、Kling AI（Kling 3.0）、Google（Veo 3.1／Flow）、Vidu（Q3 Pro）、PixVerse（V6）、Grok Imagine（grok-imagine-video）、
LTX Studio（LTX-2.3）の7サービスをランキング上位から選定。
HappyHorse-1.0（Alibaba-ATH）・SkyReels V4（Skywork AI）・Wan 2.6（Alibaba）・Agnes-Video-V2.0（Sapiens AI）等は、
初心者向けの消費者アプリが確認できないため対象外。

| サービス | 公式URL | 無料でできること | 有料プラン目安 | 初心者向け |
|---|---|---|---|---|
| **Google**（Veo 3.1／Flow） | gemini.google.com | Geminiアプリで月数回程度の動画生成が無料（透かしあり） | AI Pro ¥2,900/月（生成数アップ）／AI Ultra ¥14,500〜32,000/月（Veo 3.1フル機能・Flowで本格制作） | ◎ チャット感覚で使え日本語完全対応 |
| **Grok Imagine**（grok-imagine-video） | grok.com | 無料プランなし（2026年に廃止） | SuperGrok Lite $10/月／SuperGrok $30/月 | △ 生成は高速だが無料体験不可、ポリシー変更が多い |
| **Kling AI**（Kling 3.0） | klingai.com | 毎日クレジット付与（数本の短尺動画を生成可能） | Standard $6.99/月／Pro・Premier上位プラン | ◎ Web UIが直感的で画質・表現力ともにトップクラス |
| **PixVerse**（V6） | pixverse.ai | 毎日クレジット付与（数本生成可能、無料分は商用利用不可） | Standard $10/月／Pro $30/月／Premium上位 | ◎ テンプレートが豊富でSNS向け動画を作りやすい |
| **Vidu**（Q3 Pro） | vidu.com | 毎日クレジット付与（数本生成可能） | Standard $10/月／Pro上位プラン | ○ キャラクターの一貫性維持機能が使いやすい |
| **Dreamina**（Seedance 2.0） | dreamina.capcut.com | 毎日クレジット付与（比較的多め） | Basic $15/月／上位プランあり | ◎ CapCut連携でSNS投稿までスムーズ |
| **LTX Studio**（LTX-2.3） | ltx.studio | 少量のクレジットで試用可（透かしあり） | Lite $15/月（個人利用）／Standard $35/月（商用利用可）／Pro上位プラン | ○ ストーリーボード形式で複数シーンの映像制作に強い |

**用途タグ（Q1マッチング用）**

| サービス | tags |
|---|---|
| Google | photo, business, story, audio |
| Grok Imagine | social, photo, story |
| Kling AI | photo, illustration, story, business |
| PixVerse | social, illustration, photo |
| Vidu | illustration, story, social |
| Dreamina | illustration, social, business, audio |
| LTX Studio | story, business, audio |

**特性スコア（Q3マッチング用、1〜5）**

| サービス | 画質(quality) | スピード(speed) | コスパ(value) | 音声対応(audio) |
|---|---|---|---|---|
| Google | 4 | 3 | 2 | 5 |
| Grok Imagine | 4 | 5 | 2 | 2 |
| Kling AI | 5 | 3 | 4 | 3 |
| PixVerse | 4 | 4 | 3 | 2 |
| Vidu | 4 | 4 | 3 | 2 |
| Dreamina | 5 | 4 | 4 | 4 |
| LTX Studio | 3 | 2 | 2 | 3 |

**リスク・注意点（結果画面に表示、各1〜2行）**

| サービス | リスク・注意点 |
|---|---|
| Google | AI Ultraプランは高額（¥14,500〜32,000/月）。実在の人物・著名人の肖像を使う動画生成には厳格な制限があり、SynthIDによる透かしが付与される。 |
| Grok Imagine | 2026年にディープフェイク関連の問題が表面化し、EU/UK等で規制調査が進行中。実写人物の顔を使う動画生成は特に注意が必要。 |
| Kling AI | 顔写真アップロード機能は悪用を禁止する規約があるが運用は緩やかとの指摘も。中国企業のためデータの取り扱いに留意。 |
| PixVerse | フェイススワップ・リップシンク機能あり。無料プランの生成物は公開され商用利用不可。 |
| Vidu | キャラクター参照機能で実写人物の顔も利用可能。中国企業のためデータの取り扱いに留意。 |
| Dreamina | 実写人物の顔写真アップロードは規約で禁止（なりすまし対策）。有料プランでも規約上「個人利用が主」とされ商用利用には注意。ByteDance（中国企業）のためデータの取り扱いに留意。 |
| LTX Studio | 無料/Liteプランの生成物は個人利用限定（商用利用にはStandard以上が必要）。過去にFaceSwitch機能で実写人物の顔の取り扱いに懸念が指摘されたことがある。 |

## 動画生成: スコアリングロジック

```
score(service) =
    (service.tags に Q1タグを含む ? +3 : 0)
  + budgetScore(Q2, service)
  + service.scores[Q3で選んだ軸]
  + likenessAdjust(Q4, service)     // 下記
  + commercialAdjust(Q5, service)   // 下記
```

- `budgetScore`:
  - Q2=A（無料で十分）: `service.scores.value` を加算
  - Q2=B（月1,000〜3,000円）: 全サービスに¥3,000以下のエントリー有料プラン（Lite/Standard/Basic等）があるため、全サービス +3
  - Q2=C（しっかり課金OK）: 全サービスがエントリーを大きく超える上位プラン（Pro/Premier/Ultra等）を持つため、全サービス +3
- `likenessAdjust`（Q4=実在する人物の顔を使うか）:
  - Q4=A（はい）: Dreamina `-3`（実写人物の顔アップロードを規約で禁止しており目的を果たせない）、Grok Imagine `-2`（2026年のディープフェイク問題で規制リスクが大きい）、LTX Studio `-1`（FaceSwitch機能で過去に懸念が指摘）、Google/Kling AI/PixVerse/Vidu `0`
  - Q4=B（いいえ）: 全サービス `0`
- `commercialAdjust`（Q5=商用利用するか）:
  - Q5=A（商用利用する）: LTX Studio `-2`（無料/Liteは個人利用限定、商用には$35/月のStandard以上が必要）、Dreamina `-2`（規約上「個人利用が主」とされ商用利用には注意）、Grok Imagine `-1`（規制面の不確実性）、Google/Kling AI/PixVerse/Vidu `0`
  - Q5=B（個人利用）: 全サービス `0`

全7サービスのスコアを計算し降順ソート。上位3件を結果表示（同点時は元の配列順を優先）。

---

## コーディング診断: 質問設計（5問）

**Q1. 何をしたい？**（用途タグ・単一選択）
| 選択肢 | タグ |
|---|---|
| プログラミングの勉強・わからないことを質問したい | `learn` |
| ふだん使っているエディタでコード補完・提案を受けたい | `autocomplete` |
| 「こんなアプリ・ツールが欲しい」と伝えて作ってもらいたい（バイブコーディング） | `vibe` |
| 既存コードのバグ修正・リファクタリングを手伝ってほしい | `debug` |
| プログラミングなしでWebサービス・アプリを作りたい | `nocode` |
| ちょっとした自動化スクリプトを作りたい | `automation` |

**Q2. 予算は？**（LLM・画像生成・動画生成と同じ選択肢を再利用）
- A: 無料で十分
- B: 月1,000〜3,000円くらいまで
- C: しっかり課金してOK（本格利用）

**Q3. 一番重視するのは？**
- A: コードの正確さ・品質
- B: 返答スピード・テンポ
- C: 無料でどれだけ使えるか
- D: どこまでAIにお任せできるか（自動化・エージェント度）

**Q4. ターミナルやエディタへのインストールが必要でも平気？**
- A: できればブラウザだけで完結させたい
- B: ターミナル・エディタへのインストールは平気

**Q5. 自分のコードを外部のAIサービスに渡すことについて**
- A: とても気になる
- B: それほど気にしない

## コーディング診断: データセット（7サービス）

各サービスのフィールドは動画生成と同様：
`name, officialUrl, freePlan, paidPlans, beginnerFriendly, tags[], scores{quality, speed, value, autonomy}, reasonTemplates{usage, budget, priority, environment, privacy}, risk`

動画生成との違いは、`scores` の4軸目が **`autonomy`**（自動化・エージェント度、Q3に対応）であること、
`reasonTemplates` のリスク軸が **`environment`**（Q4=A/Bごとの1文、ターミナル操作への抵抗）と
**`privacy`**（Q5=A/Bごとの1文、コードを外部AIに渡す抵抗）であること。

選定方針：ユーザー指定の Claude Code・Codex を軸に、初心者が遭遇しやすい主要なAIコーディング支援ツールを
「エディタ拡張型」（GitHub Copilot）・「AI専用IDE型」（Cursor・Antigravity）・「CLI/クラウドエージェント型」
（Claude Code・Codex）・「ブラウザ完結型」（Replit・bolt.new）の4タイプの利用形態がバランスするように7サービスを選定。
なお、当初候補だった「Gemini Code Assist for individuals」は2026年6月18日にサービス終了し、
後継の「Antigravity」（Gemini 3 Pro採用のエージェント型開発環境、現在パブリックプレビュー）へ統合されるため、
データセットには移行後のAntigravityを採用し、移行期特有の不安定さはリスク欄に明記する。

| サービス | 公式URL | 無料でできること | 有料プラン目安 | 初心者向け |
|---|---|---|---|---|
| **GitHub Copilot** | github.com/features/copilot | 月2,000回のコード補完＋月50回のチャット・編集（クレカ不要） | Pro $10/月（約1,500円）／Pro+ $39/月／Max $100/月 | ◎ VS Code拡張機能を入れてGitHubでログインするだけ |
| **Cursor** | cursor.com | Hobbyプラン：月2,000回のコード補完＋月50回のプレミアムモデル利用 | Pro $20/月（約3,000円）／Pro+ $60/月／Ultra $200/月 | ○ VS Codeベースの専用エディタをインストール、日本語UI対応 |
| **Claude Code** | claude.com/product/claude-code | なし（Claude Pro以上のプランが必須） | Pro $20/月（約3,000円）／Max 5x $100/月／Max 20x $200/月 | ○ 2026年からネイティブインストーラー・デスクトップアプリ・Web版も登場 |
| **Codex**（ChatGPT） | chatgpt.com/codex | ChatGPT Freeに含まれる（利用回数の上限は低め） | ChatGPT Plus $20/月（約3,000円）／Pro $100〜/月 | ◎ ChatGPTのサイドバーからすぐ使える、ゼロセットアップ |
| **Replit** | replit.com | ブラウザのみで利用可。毎日少量のAgentクレジット＋月1,200分の実行時間 | Core $25/月（約3,800円）／Pro $100/月 | ◎ ブラウザだけで完結、日本語の指示でアプリ作成からデプロイまでAgentが自動化 |
| **Antigravity**（Google） | antigravity.google | パブリックプレビュー中は無料（Gemini 3 Pro、レート制限あり） | Google AI Pro ¥2,900/月／AI Ultra ¥14,500〜32,000/月 | △ VS Codeフォークの専用デスクトップアプリをダウンロードする必要がある |
| **bolt.new** | bolt.new | 1日30万トークン・月100万トークンまで（Boltブランディング表示あり） | Pro $25/月（約3,800円、月1,000万トークン） | ◎ ブラウザのみで登録後すぐ使え、日本語プロンプトでアプリ生成 |

**用途タグ（Q1マッチング用）**

| サービス | tags |
|---|---|
| GitHub Copilot | autocomplete, debug, learn |
| Cursor | autocomplete, vibe, debug, automation |
| Claude Code | vibe, debug, automation, learn |
| Codex | vibe, debug, automation, learn |
| Replit | nocode, vibe, learn, automation |
| Antigravity | vibe, debug, automation |
| bolt.new | nocode, vibe |

**特性スコア（Q3マッチング用、1〜5）**

| サービス | 正確さ(quality) | スピード(speed) | コスパ(value) | 自動化度(autonomy) |
|---|---|---|---|---|
| GitHub Copilot | 4 | 5 | 4 | 3 |
| Cursor | 5 | 4 | 3 | 4 |
| Claude Code | 5 | 3 | 1 | 5 |
| Codex | 5 | 4 | 4 | 5 |
| Replit | 3 | 4 | 3 | 5 |
| Antigravity | 4 | 3 | 4 | 5 |
| bolt.new | 3 | 4 | 3 | 4 |

**リスク・注意点（結果画面に表示、各1〜2行）**

| サービス | リスク・注意点 |
|---|---|
| GitHub Copilot | 2026年4月以降、無料/Pro/Pro+の利用データはデフォルトで学習に使われる設定に変更（オプトアウト可）。価格・プラン体系の変更が頻繁。 |
| Cursor | 2025年の価格体系変更で「有料プランでも使える量が予想より少ない」と大きな反発があった経緯あり。無料/Proはデフォルトで最大30日データ保存。 |
| Claude Code | 無料プランなし（Pro以上が必須）。自律的にファイルの作成・変更・削除を行うため、誤操作でファイルが削除された事例が複数報告されている。 |
| Codex | デスクトップ版で予期しないファイル削除事故やトークン窃取事件が報告されている。個人プラン(Plus)はデフォルトで学習に使われる（オプトアウト可）。 |
| Replit | 2025年、AIエージェントが本番データベースを削除し報告を改ざんする事故が発生し話題に。重要なデータを扱う際は権限設定の確認が必要。 |
| Antigravity | 2026年6月時点でパブリックプレビュー中（前身のGemini Code Assistから移行直後）。今後機能・価格が大きく変わる可能性が高い。 |
| bolt.new | エラー修正の繰り返しでトークンを大量消費しやすく、無料枠を使い切りやすい。プロジェクトを「公開」設定にすると誰でもコードを閲覧可能になる。 |

## コーディング: スコアリングロジック

```
score(service) =
    (service.tags に Q1タグを含む ? +3 : 0)
  + budgetScore(Q2, service)
  + service.scores[Q3で選んだ軸]
  + environmentAdjust(Q4, service)  // 下記
  + privacyAdjust(Q5, service)      // 下記
```

- `budgetScore`:
  - Q2=A（無料で十分）: `service.scores.value` を加算
  - Q2=B（月1,000〜3,000円）: エントリー有料プランが¥3,000以下のサービス（GitHub Copilot/Cursor/Claude Code/Codex/Antigravity）は +3、それを超えるサービス（Replit/bolt.new）は +1
  - Q2=C（しっかり課金OK）: エントリーを大きく超える上位プラン（Pro+/Max/Ultra/Pro等）を持つサービスは +3、エントリーのみで上位プランがないサービス（bolt.new）は +1
- `environmentAdjust`（Q4=ターミナル・インストールへの抵抗）:
  - Q4=A（ブラウザだけで完結させたい）: Replit `+2`・bolt.new `+2`（完全ブラウザ完結でインストール不要）、Codex `+1`（ブラウザ版が主軸でゼロセットアップ可）、GitHub Copilot `-1`（エディタへの拡張機能インストールが必要）、Cursor `-2`・Claude Code `-2`・Antigravity `-2`（専用エディタ・CLIのインストールが必須）
  - Q4=B（インストールは平気）: 全サービス `0`
- `privacyAdjust`（Q5=コードを外部AIに渡す抵抗）:
  - Q5=A（とても気になる）: Replit `+1`（非公開プロジェクトはデフォルトで学習対象外）、Cursor `+1`（プライバシーモードでゼロデータ保持に設定可能）、Antigravity `-1`（個人データが収集され人間レビューに使われる可能性も明記）、GitHub Copilot/Claude Code/Codex/bolt.new `0`
  - Q5=B（気にしない）: 全サービス `0`

全7サービスのスコアを計算し降順ソート。上位3件を結果表示（同点時は元の配列順を優先）。

---

## 結果画面

- **1位**（大きいカード）
  - サービス名／おすすめプランと価格
  - 「あなたにおすすめな理由」：`reasonTemplates` に定義されている軸の分だけ理由文を組み立てる
    - LLM: Q1（用途）・Q2（予算）・Q3（重視点）の3文
    - 画像生成: Q1（用途）・Q2（予算）・Q3（重視点）・Q4（商用利用）の4文
    - 動画生成: Q1（用途）・Q2（予算）・Q3（重視点）・Q4（実写人物の顔）・Q5（商用利用）の5文
    - コーディング: Q1（用途）・Q2（予算）・Q3（重視点）・Q4（ターミナル・インストールへの抵抗）・Q5（コードを外部AIに渡す抵抗）の5文
  - リスク・注意点（上表の内容を1〜2行で表示）
  - 「公式サイトを見る」ボタン（`officialUrl` を新規タブで開く）
- **2位・3位**（小さいカード）：サービス名／一言理由／公式リンク
- **共通フッター注記**：カテゴリごとの `footerNote` を表示
  - LLM: 「個人情報・機密情報を入力する際は、各サービスのプライバシー設定をご確認ください」
  - 画像生成: 「生成画像を商用利用する際は、各サービスの利用規約・ライセンス条件を必ずご確認ください」
  - 動画生成: 「実写人物の顔を使う場合は本人の同意やなりすまし防止に関する各サービスの規約を、商用利用する場合は利用規約・ライセンス条件をあわせてご確認ください」
  - コーディング: 「AIエージェントはファイルの作成・変更・削除を自律的に行うことがあります。重要なファイルは必ずバックアップを取り、最初は『実行前に確認』の設定で使うことをおすすめします。コードの学習利用に関するプライバシー設定も各サービスでご確認ください」
- 「もう一度診断する」ボタン → ジャンル選択画面に戻る

## テスト

- `jinkenhi-timer` と同様に、スコアリング関数を純粋関数として実装し、ブラウザconsoleで実行されるセルフテストを用意する
  - LLM例: 「Q1=coding, Q2=A(無料), Q3=value, Q4=B」→ DeepSeekが1位になることをassert
  - LLM例: 「Q4=A」→ DeepSeekがトップ3から除外されることをassert
  - 画像生成例: 「Q1=business, Q2=A(無料), Q3=value, Q4=A(商用利用)」→ Geminiが1位になることをassert
  - 画像生成例: 「Q1=photo, Q2=C(しっかり課金), Q3=quality, Q4=A(商用利用)」→ Geminiが1位、Grok Imagineは1位にならないことをassert
  - 動画生成例: 「Q1=audio, Q2=A(無料), Q3=audio(重視点=音声), Q4=B(実写人物の顔は使わない), Q5=B(個人利用)」→ Dreaminaが1位になることをassert
  - 動画生成例: 「Q1=photo, Q2=C(しっかり課金), Q3=quality, Q4=A(実写人物の顔を使う), Q5=A(商用利用)」→ Kling AIが1位、Grok Imagineは1位にならないことをassert
  - コーディング例: 「Q1=nocode, Q2=A(無料), Q3=value, Q4=B(インストール平気), Q5=B(気にしない)」→ Replitが1位、Claude Codeが最下位になることをassert
  - コーディング例: 「Q1=vibe, Q2=A(無料), Q3=value, Q4=A(ブラウザのみ), Q5=A(プライバシー重視)」→ Codexが1位になり、Claude Codeは1位にならないことをassert
- UI動作は手動でブラウザ確認（ジャンル選択→診断→結果→もう一度診断、全4カテゴリ）
