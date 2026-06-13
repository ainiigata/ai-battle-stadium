# ゴリスケチャットボット「一つ前に戻る」機能 設計ドキュメント

作成日: 2026-06-14

## 概要

`ai-shindan.html`(ゴリスケチャットボット)の診断チャットに、「一つ前に戻る」機能を追加する。
診断中・診断結果表示後に、ユーザーが直前の回答を選び直せるようにする。

対象ファイルは `ai-shindan.html` のみ。

## 1. 「戻る」ボタンの表示場所と動作

| 画面 | 「戻る」表示 | 押したときの動作 |
|---|---|---|
| ジャンル選択画面(最初の画面) | 表示しない | - |
| Q1(各カテゴリの最初の質問) | 表示する | ジャンル選択画面に戻る |
| Q2以降の質問 | 表示する | 1つ前の質問に戻り、選び直せる |
| 診断結果画面 | 「もう一度診断する 🔄」の横に表示する | 最後の質問に戻り、選び直せる |

「戻る」を押すと、現在表示中のチャット吹き出し(質問・回答)を該当数だけチャット欄から削除し、対応する質問を改めて表示し直す(巻き戻し方式)。削除された質問に記録されていた回答(`state.answers`)も削除する。

## 2. 実装方針

### 2-1. ヘルパー関数

```js
function removeLastMessage() {
  const chat = document.getElementById('chat');
  if (chat.lastElementChild) chat.lastElementChild.remove();
}
```

### 2-2. 共通の「1つ前の質問に戻る」処理

```js
function goBackToQuestion(removeCount) {
  for (let i = 0; i < removeCount; i++) removeLastMessage();
  delete state.answers['q' + state.questionIndex];
  state.questionIndex--;
  askNextQuestion();
}
```

呼び出し時点の `state.questionIndex` は「今まさに表示している質問のインデックス」(質問中の「戻る」)または「カテゴリの質問総数」(結果画面の「戻る」)。いずれの場合も、直前に記録された回答のキーは `'q' + state.questionIndex` であるため、この共通関数で削除・巻き戻しができる。

### 2-3. 質問中の「戻る」

```js
function goBackFromQuestion() {
  if (state.questionIndex === 0) {
    removeLastMessage(); // 現在の質問(Q1)の吹き出し
    removeLastMessage(); // ジャンル選択で選んだ回答の吹き出し
    removeLastMessage(); // 「今日は何について知りたい?」の吹き出し
    state.categoryKey = null;
    state.answers = {};
    showGenreSelection();
  } else {
    goBackToQuestion(3); // 現在の質問 / 直前の回答 / 直前の質問 の3件を削除
  }
}
```

### 2-4. 結果画面の「戻る」

```js
function goBackFromResult() {
  goBackToQuestion(4); // 結果カード / 「結果が出たよ」/ 最後の回答 / 最後の質問 の4件を削除
}
```

### 2-5. `showGenreSelection()` の切り出し

`startChat()` の後半(ジャンル選択の表示処理)を関数として切り出し、`startChat()` と `goBackFromQuestion()` の両方から呼べるようにする。

```js
function showGenreSelection() {
  showTyping(function () {
    addBotMessage('今日は何について知りたい?');
    addQuickReplies(
      Object.keys(CATEGORIES).map(function (key) {
        return { value: key, label: GENRE_ICONS[key] + ' ' + CATEGORIES[key].label };
      }),
      function (opt) {
        addUserMessage(opt.label);
        selectGenre(opt.value);
      }
    );
  });
}
```

`startChat()` はヒーローメッセージ追加後にこの関数を呼ぶように変更する。

### 2-6. `askNextQuestion()` への「戻る」選択肢追加

`addQuickReplies` に渡す選択肢配列の末尾に `{ value: '__back__', label: '← 一つ前に戻る' }` を追加し、選択時に `__back__` なら `addUserMessage`/`selectAnswer` を呼ばず `goBackFromQuestion()` を呼ぶ。

```js
function askNextQuestion() {
  const category = CATEGORIES[state.categoryKey];
  const question = category.questions[state.questionIndex];
  const total = category.questions.length;

  showTyping(function () {
    addBotMessage(buildProgressDots(state.questionIndex, total) + '<div class="quiz-question-text">' + question.text + '</div>');
    addQuickReplies(
      question.options.concat([{ value: '__back__', label: '← 一つ前に戻る' }]),
      function (opt) {
        if (opt.value === '__back__') {
          goBackFromQuestion();
          return;
        }
        addUserMessage(opt.label);
        selectAnswer(opt.value);
      }
    );
  });
}
```

### 2-7. `showResultMessage()` への「戻る」選択肢追加

```js
function showResultMessage() {
  const category = CATEGORIES[state.categoryKey];
  const ranked = rankServices(state.categoryKey, state.answers).slice(0, 3);
  showTyping(function () {
    addBotMessage('よし、診断結果が出たよ!');
    addBotMessage(renderResultsHtml(category, ranked), 'result');
    addQuickReplies(
      [
        { value: 'restart', label: 'もう一度診断する 🔄' },
        { value: '__back__', label: '← 一つ前に戻る' }
      ],
      function (opt) {
        if (opt.value === '__back__') {
          goBackFromResult();
          return;
        }
        startChat();
      }
    );
  });
}
```

## 3. CSS

`.quick-reply-btn` の定義の近くに、控えめな見た目の「戻る」専用クラスを追加する。

```css
.quick-reply-back {
  background: transparent;
  border: 1px solid var(--border);
  color: var(--text-muted);
}
```

`addQuickReplies` 内で、`opt.value === '__back__'` の場合に `btn.className += ' quick-reply-back'` を追加する。

```js
function addQuickReplies(options, onSelect) {
  const chat = document.getElementById('chat');
  const wrap = document.createElement('div');
  wrap.className = 'quick-replies';
  options.forEach(function (opt) {
    const btn = document.createElement('button');
    btn.className = 'quick-reply-btn' + (opt.value === '__back__' ? ' quick-reply-back' : '');
    btn.textContent = opt.label;
    btn.addEventListener('click', function () {
      wrap.remove();
      onSelect(opt);
    });
    wrap.appendChild(btn);
  });
  chat.appendChild(wrap);
  scrollToLatest(wrap);
}
```

## 4. テスト方針

- 既存のNodeベースセルフテスト(`?test=1`)はスコアリングロジック(`rankServices`等)のみが対象でDOM操作を含まないため、新規の自動テストは追加しない。既存14件は変更しないため14/0 passを維持する。
- ブラウザでの手動確認を行う:
  - 各カテゴリでQ1の「戻る」を押すと、ジャンル選択画面に戻ること
  - Q2以降で「戻る」を押すと、1つ前の質問が再表示され、選び直せること(進行ドットが正しく1つ戻ること)
  - 診断結果画面で「戻る」を押すと、最後の質問に戻って選び直せ、再診断すると新しい結果が表示されること
  - 「戻る」ボタンが他の選択肢と視覚的に区別できること(枠線のみの控えめな見た目)

## 5. 影響範囲

- `CATEGORIES` データ、スコアリング関数(`scoreService`等)、セルフテストへの変更はなし
- 変更対象は `askNextQuestion`、`showResultMessage`、`addQuickReplies`、`startChat`(`showGenreSelection`切り出し)、新規関数(`removeLastMessage`、`goBackToQuestion`、`goBackFromQuestion`、`goBackFromResult`、`showGenreSelection`)、CSS
