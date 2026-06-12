# 生成AI選び診断ツール 設計メモ（LLM編）

作成日: 2026-06-13

## 概要
「素人（AI初心者）が自分に合った生成AIを見つけられるチャットボット」の第一弾。
ジャンル選択（LLM／画像生成／動画生成／コーディング）→診断クイズ→おすすめAI表示、という流れの
**汎用診断フレームワーク**を作り、今回は **LLMカテゴリのみ実データを完成**させる。
他3カテゴリは枠（UI・診断エンジン）のみ用意し「近日公開」として表示する。

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
  llm:    { label: "文章で相談・LLM", questions: [...], services: [...] }, // 今回フル実装
  image:  { label: "画像生成", comingSoon: true },  // 近日公開
  video:  { label: "動画生成", comingSoon: true },
  coding: { label: "コーディング", comingSoon: true },
}
```
診断エンジン（質問を1問ずつ表示→回答収集→スコアリング→結果表示）はカテゴリに依存しない汎用実装とし、
将来 `image` / `video` / `coding` に `questions` と `services` を追加するだけで動作するようにする。

## 画面フロー
1. **ジャンル選択画面**：4枚のカード（LLM／画像生成／動画生成／コーディング）。
   後者3つは「近日公開」バッジ付き。タップすると「準備中：今後データを追加していきます」という
   メッセージカードを表示し、ジャンル選択に戻れる。
2. **診断画面**：チャット風UIで1問ずつ選択肢を表示。進捗ドット（1/4, 2/4, 3/4, 4/4）を表示。
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

## 結果画面

- **1位**（大きいカード）
  - サービス名／おすすめプランと価格
  - 「あなたにおすすめな理由」：Q1（用途）・Q2（予算）・Q3（重視点）にそれぞれ対応する1文（`reasonTemplates` から組み立て）
  - リスク・注意点（上表の内容を1〜2行で表示）
  - 「公式サイトを見る」ボタン（`officialUrl` を新規タブで開く）
- **2位・3位**（小さいカード）：サービス名／一言理由／公式リンク
- **共通フッター注記**：「個人情報・機密情報を入力する際は、各サービスのプライバシー設定をご確認ください」
- 「もう一度診断する」ボタン → ジャンル選択画面に戻る

## 他カテゴリ（画像生成・動画生成・コーディング）の扱い

- ジャンル選択画面に4枚のカードを表示し、LLM以外の3枚には「近日公開」バッジ
- タップすると「準備中：今後データを追加していきます」というメッセージカードを表示し、ジャンル選択に戻るボタンのみ表示
- データ追加時は `CATEGORIES.image` 等に `questions` と `services` を追加するだけで、診断エンジン・結果画面はそのまま再利用できる

## テスト

- `jinkenhi-timer` と同様に、スコアリング関数を純粋関数として実装し、ブラウザconsoleで実行されるセルフテストを用意する
  - 例: 「Q1=coding, Q2=A(無料), Q3=value, Q4=B」→ DeepSeekが1位になることをassert
  - 例: 「Q4=A」→ DeepSeekがトップ3から除外されることをassert
- UI動作は手動でブラウザ確認（ジャンル選択→診断→結果→もう一度診断、近日公開タップ）
