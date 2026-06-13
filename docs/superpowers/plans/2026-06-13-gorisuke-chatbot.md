# ゴリスケチャットボット Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `ai-shindan.html`(生成AI選び診断)を「ゴリスケチャットボット」としてリブランドし、画面切り替え型UIを連続チャット形式UIに置き換える。マスコット「ゴリすけ」がアバター+吹き出しで質問・診断結果を語りかける。

**Architecture:** CATEGORIES データとスコアリング関数(budgetScore/extraAdjust/scoreService/buildReasons/rankServices)はそのまま再利用。画面切り替えエンジン(showScreen/renderGenreScreen/renderQuestion/renderResults/resetToGenre)を、チャットメッセージを動的に追加するエンジン(addBotMessage/addUserMessage/addQuickReplies/showTyping/startChat/askNextQuestion/showResultMessage)に置き換える。ゴリすけ画像はリサイズしてbase64で埋め込み、単一HTMLファイルの完結性を維持する。

**Tech Stack:** Vanilla HTML/CSS/JS(単一ファイル)、macOS `sips`(画像リサイズ)、Node.js(base64埋め込み・セルフテスト実行)、Playwright(UI動作確認)

---

## 参考: 設計書

`docs/superpowers/specs/2026-06-13-gorisuke-chatbot-design.md`

## 前提

- `ai-shindan.html` は既に完成済み(全4カテゴリ×7サービス、セルフテスト12 pass/0 fail)
- ゴリすけ画像: `/Users/yamadatoshi/yamada-ai-claude/gorisuke-character.png`(1024×1536、RGBA、透過背景あり)

---

### Task 1: ゴリすけ画像をリサイズしてbase64埋め込み

**Files:**
- Modify: `/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html`(`<script>`タグ直後)

- [ ] **Step 1: 画像をリサイズ**

```bash
sips -Z 360 /Users/yamadatoshi/yamada-ai-claude/gorisuke-character.png --out /tmp/gorisuke-360.png
```

Expected: `/private/tmp/gorisuke-360.png` (240×360前後) が作成される

- [ ] **Step 2: base64化して `<script>` 直後に `GORISUKE_IMG` 定数を挿入**

```bash
node -e "
const fs = require('fs');
const b64 = fs.readFileSync('/tmp/gorisuke-360.png').toString('base64');
const htmlPath = '/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html';
const html = fs.readFileSync(htmlPath, 'utf8');
const decl = '<script>\n' + 'const GORISUKE_IMG = \"data:image/png;base64,' + b64 + '\";\n';
const updated = html.replace('<script>\n', decl);
fs.writeFileSync(htmlPath, updated);
console.log('inserted, base64 length:', b64.length);
"
```

Expected: `inserted, base64 length: 87172` 前後の数値が出力される

- [ ] **Step 3: 挿入を確認**

```bash
grep -c "const GORISUKE_IMG" /Users/yamadatoshi/yamada-ai-claude/ai-shindan.html
grep -c "const CATEGORIES" /Users/yamadatoshi/yamada-ai-claude/ai-shindan.html
```

Expected: 両方とも `1`

- [ ] **Step 4: Commit**

```bash
cd /Users/yamadatoshi/yamada-ai-claude
git add ai-shindan.html
git commit -m "feat(gorisuke-chatbot): ゴリすけ画像をbase64で埋め込み"
```

---

### Task 2: タイトル・見出しを「ゴリスケチャットボット」に変更

**Files:**
- Modify: `/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html` (`<title>` 行、`<h1>`/`<p>` 行)

- [ ] **Step 1: `<title>` を変更**

`<title>生成AI選び診断 | あなたにぴったりのAIが見つかる</title>` を以下に置き換える:

```html
<title>ゴリスケチャットボット | あなたにぴったりの生成AIが見つかる</title>
```

- [ ] **Step 2: `<h1>`/`<p>` を変更**

```html
    <h1>生成AI選び診断</h1>
    <p>かんたんな質問に答えるだけで、あなたにぴったりの生成AIが見つかります</p>
```

を以下に置き換える:

```html
    <h1>ゴリスケチャットボット</h1>
    <p>ゴリすけと話して、あなたにぴったりの生成AIを見つけよう</p>
```

- [ ] **Step 3: Commit**

```bash
cd /Users/yamadatoshi/yamada-ai-claude
git add ai-shindan.html
git commit -m "feat(gorisuke-chatbot): タイトル・見出しをゴリスケチャットボットに変更"
```

---

### Task 3: CSSをチャットUI用に書き換え

**Files:**
- Modify: `/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html`(`<style>...</style>` ブロック全体)

既存の `<style>` ブロック(`:root` から `.btn-secondary:hover { ... }` まで)を、以下の内容に**全体置き換え**する。

- [ ] **Step 1: `<style>` ブロックを以下に置き換え**

```html
<style>
:root {
  --gold: #fbd784;
  --dark: #0b1d26;
  --dark-card: rgba(255,255,255,0.04);
  --border: rgba(251,215,132,0.25);
  --text-muted: rgba(255,255,255,0.65);
}

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  background: var(--dark);
  color: #fff;
  font-family: 'IBM Plex Sans JP', 'Montserrat', sans-serif;
  min-height: 100vh;
  line-height: 1.7;
  position: relative;
  overflow-x: hidden;
}

body::before {
  content: '';
  position: fixed;
  top: -20%;
  left: -10%;
  width: 60vw;
  height: 60vw;
  background: radial-gradient(circle, rgba(251,215,132,0.15), transparent 70%);
  filter: blur(60px);
  z-index: -1;
  animation: drift 30s ease-in-out infinite alternate;
}
body::after {
  content: '';
  position: fixed;
  bottom: -20%;
  right: -10%;
  width: 50vw;
  height: 50vw;
  background: radial-gradient(circle, rgba(11,80,120,0.4), transparent 70%);
  filter: blur(80px);
  z-index: -1;
  animation: drift 40s ease-in-out infinite alternate-reverse;
}
@keyframes drift {
  from { transform: translate(0,0); }
  to { transform: translate(5%, 5%); }
}

.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 40px 20px 80px;
}

header.app-header {
  text-align: center;
  margin-bottom: 40px;
}
header.app-header h1 {
  font-family: 'Playfair Display', serif;
  font-size: clamp(28px, 5vw, 42px);
  color: var(--gold);
  letter-spacing: 0.05em;
}
header.app-header p {
  color: var(--text-muted);
  margin-top: 8px;
  font-size: 14px;
}

@keyframes fadeIn { from { opacity:0; transform: translateY(10px);} to {opacity:1; transform:translateY(0);} }

.chat {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.msg {
  display: flex;
  gap: 10px;
  align-items: flex-start;
  animation: fadeIn 0.4s ease;
}
.msg.user {
  justify-content: flex-end;
}
.msg.hero {
  align-items: center;
  flex-wrap: wrap;
}

.avatar {
  width: 44px;
  height: 44px;
  border-radius: 50%;
  flex-shrink: 0;
  background-image: var(--gorisuke-avatar);
  background-size: 250% auto;
  background-position: center 5%;
  background-repeat: no-repeat;
  background-color: var(--dark-card);
  border: 2px solid var(--gold);
}

.hero-img {
  width: 120px;
  border-radius: 12px;
  flex-shrink: 0;
}

.bubble {
  background: var(--dark-card);
  border: 1px solid var(--border);
  border-radius: 16px 16px 16px 4px;
  padding: 14px 18px;
  max-width: 80%;
  min-width: 0;
  font-size: 15px;
}
.bubble.user {
  background: linear-gradient(135deg, var(--gold), #f5b942);
  color: var(--dark);
  border: none;
  border-radius: 16px 16px 4px 16px;
  font-weight: 600;
}
.bubble.typing {
  display: flex;
  gap: 4px;
  align-items: center;
  padding: 18px;
}
.bubble.typing span {
  width: 8px; height: 8px;
  border-radius: 50%;
  background: var(--gold);
  opacity: 0.5;
  animation: bounce 1.2s infinite ease-in-out;
}
.bubble.typing span:nth-child(2) { animation-delay: 0.15s; }
.bubble.typing span:nth-child(3) { animation-delay: 0.3s; }
@keyframes bounce {
  0%, 80%, 100% { transform: translateY(0); opacity: 0.4; }
  40% { transform: translateY(-6px); opacity: 1; }
}

.quiz-question-text {
  font-size: 16px;
  font-weight: 700;
  margin-top: 6px;
}

.progress-dots {
  display: flex;
  gap: 8px;
  margin-bottom: 8px;
}
.dot {
  width: 8px; height: 8px;
  border-radius: 50%;
  background: rgba(255,255,255,0.15);
  border: 1px solid var(--border);
}
.dot.active { background: var(--gold); box-shadow: 0 0 6px var(--gold); }
.dot.done { background: rgba(251,215,132,0.5); }

.quick-replies {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-left: 54px;
}

.quick-reply-btn {
  background: var(--dark-card);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 14px 18px;
  color: #fff;
  font-size: 15px;
  text-align: left;
  cursor: pointer;
  transition: border-color 0.25s ease, transform 0.25s ease, background 0.25s ease;
  font-family: inherit;
}
.quick-reply-btn:hover {
  border-color: var(--gold);
  background: rgba(251,215,132,0.08);
  transform: translateX(4px);
}

.msg.bot.result .bubble {
  background: transparent;
  border: none;
  padding: 0;
  max-width: 100%;
}

.result-first {
  background: var(--dark-card);
  border: 1px solid var(--gold);
  border-radius: 20px;
  padding: 32px;
  margin-bottom: 24px;
  box-shadow: 0 0 40px rgba(251,215,132,0.1);
}
.result-rank {
  display: inline-block;
  background: var(--gold);
  color: var(--dark);
  font-weight: 800;
  font-size: 13px;
  padding: 4px 12px;
  border-radius: 999px;
  margin-bottom: 12px;
}
.result-first h2 {
  font-family: 'Playfair Display', serif;
  font-size: clamp(26px, 5vw, 36px);
  color: var(--gold);
  margin-bottom: 8px;
}
.result-plan { color: var(--text-muted); font-size: 14px; margin-bottom: 16px; }
.result-reasons { list-style: none; margin-bottom: 16px; }
.result-reasons li {
  padding-left: 1.4em;
  position: relative;
  margin-bottom: 8px;
  font-size: 15px;
}
.result-reasons li::before {
  content: '✓';
  position: absolute; left: 0;
  color: var(--gold);
  font-weight: 700;
}
.result-risk {
  background: rgba(255,255,255,0.04);
  border-left: 3px solid var(--gold);
  padding: 12px 16px;
  font-size: 13px;
  color: var(--text-muted);
  border-radius: 8px;
  margin-bottom: 20px;
}
.result-others {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
  margin-bottom: 24px;
}
@media (max-width: 600px) { .result-others { grid-template-columns: 1fr; } }
.result-small {
  background: var(--dark-card);
  border: 1px solid var(--border);
  border-radius: 14px;
  padding: 18px;
}
.result-small .result-rank { font-size: 11px; }
.result-small h3 { font-size: 16px; margin: 6px 0 6px; }
.result-small p { font-size: 13px; color: var(--text-muted); margin-bottom: 10px; }

.footer-note {
  font-size: 12px;
  color: var(--text-muted);
  text-align: center;
  margin-bottom: 24px;
  padding: 0 10px;
}

.btn-primary, .btn-link {
  display: inline-block;
  background: linear-gradient(135deg, var(--gold), #f5b942);
  color: var(--dark);
  font-weight: 700;
  text-decoration: none;
  padding: 12px 28px;
  border-radius: 999px;
  border: none;
  cursor: pointer;
  font-size: 14px;
  transition: transform 0.25s ease, box-shadow 0.25s ease;
}
.btn-primary:hover, .btn-link:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(251,215,132,0.3);
}
.btn-link.small {
  padding: 8px 18px;
  font-size: 12px;
  background: transparent;
  border: 1px solid var(--gold);
  color: var(--gold);
}
.actions-center { text-align: center; margin-top: 12px; }
</style>
```

- [ ] **Step 2: Commit**

```bash
cd /Users/yamadatoshi/yamada-ai-claude
git add ai-shindan.html
git commit -m "feat(gorisuke-chatbot): CSSをチャットUI用に書き換え"
```

---

### Task 4: HTML bodyをチャットコンテナに置き換え

**Files:**
- Modify: `/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html`(`<div class="container">` 内、`<header>` 以降)

- [ ] **Step 1: `<header>` 以降〜`</div>`(container閉じ)までを以下に置き換え**

以下のブロック:

```html
  <header class="app-header">
    <h1>ゴリスケチャットボット</h1>
    <p>ゴリすけと話して、あなたにぴったりの生成AIを見つけよう</p>
  </header>

  <section id="screen-genre" class="screen active">
    <div class="genre-grid" id="genre-grid"></div>
  </section>

  <section id="screen-quiz" class="screen">
    <div class="progress-dots" id="progress-dots"></div>
    <h2 class="quiz-question" id="quiz-question"></h2>
    <div class="quiz-options" id="quiz-options"></div>
  </section>

  <section id="screen-result" class="screen">
    <div id="result-first"></div>
    <div class="result-others" id="result-others"></div>
    <p class="footer-note" id="footer-note"></p>
    <div class="restart-wrap">
      <button class="btn-secondary" id="restart-btn">もう一度診断する</button>
    </div>
  </section>
</div>
```

を、以下に置き換える:

```html
  <header class="app-header">
    <h1>ゴリスケチャットボット</h1>
    <p>ゴリすけと話して、あなたにぴったりの生成AIを見つけよう</p>
  </header>

  <div class="chat" id="chat"></div>
</div>
```

(Task 2で `<h1>`/`<p>` は既に変更済みのため、ここでは `<section>` 3つを `<div class="chat" id="chat"></div>` に置き換えるのが本質的な変更点)

- [ ] **Step 2: Commit**

```bash
cd /Users/yamadatoshi/yamada-ai-claude
git add ai-shindan.html
git commit -m "feat(gorisuke-chatbot): HTML bodyをチャットコンテナに置き換え"
```

---

### Task 5: エンジンJSをチャット形式に書き換え

**Files:**
- Modify: `/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html`(`function showResults() {` から `showScreen('genre');` までのブロック)

以下のブロック(`function showResults() {` から、ファイル末尾 `showScreen('genre');` の手前にある `runSelfTests` 定義・`if (new URLSearchParams...)` 行を**除いた**範囲)を全体置き換える:

**置き換え対象(現状のコード、削除する):**

```js
function showResults() {
  const category = CATEGORIES[state.categoryKey];
  const ranked = rankServices(state.categoryKey, state.answers).slice(0, 3);
  renderResults(category, ranked);
  showScreen('result');
}

function renderResults(category, ranked) {
  const first = ranked[0];
  const rest = ranked.slice(1);

  const firstEl = document.getElementById('result-first');
  firstEl.innerHTML =
    '<div class="result-rank">1位 おすすめ</div>' +
    '<h2>' + first.service.name + '</h2>' +
    '<div class="result-plan">無料: ' + first.service.freePlan + '<br>有料: ' + first.service.paidPlans + '</div>' +
    '<ul class="result-reasons">' +
      buildReasons(state.categoryKey, first.service, state.answers).map(function (r) { return '<li>' + r + '</li>'; }).join('') +
    '</ul>' +
    '<div class="result-risk">⚠️ ' + first.service.risk + '</div>' +
    '<div class="actions-center"><a class="btn-primary" href="https://' + first.service.officialUrl + '" target="_blank" rel="noopener">公式サイトを見る</a></div>';

  const othersEl = document.getElementById('result-others');
  othersEl.innerHTML = '';
  rest.forEach(function (entry, i) {
    const div = document.createElement('div');
    div.className = 'result-small';
    const reasons = buildReasons(state.categoryKey, entry.service, state.answers);
    div.innerHTML =
      '<div class="result-rank">' + (i + 2) + '位</div>' +
      '<h3>' + entry.service.name + '</h3>' +
      '<p>' + reasons[0] + '</p>' +
      '<a class="btn-link small" href="https://' + entry.service.officialUrl + '" target="_blank" rel="noopener">公式サイトを見る</a>';
    othersEl.appendChild(div);
  });

  document.getElementById('footer-note').textContent = category.footerNote;
}

const state = {
  categoryKey: null,
  questionIndex: 0,
  answers: {}
};

const GENRE_ICONS = { llm: "💬", image: "🎨", video: "🎬", coding: "💻" };

function showScreen(name) {
  document.querySelectorAll('.screen').forEach(function (s) {
    s.classList.remove('active');
  });
  document.getElementById('screen-' + name).classList.add('active');
}

function renderGenreScreen() {
  const grid = document.getElementById('genre-grid');
  grid.innerHTML = '';
  Object.keys(CATEGORIES).forEach(function (key) {
    const card = document.createElement('div');
    card.className = 'genre-card';
    card.innerHTML = '<div class="icon">' + GENRE_ICONS[key] + '</div><h2>' + CATEGORIES[key].label + '</h2>';
    card.addEventListener('click', function () { startQuiz(key); });
    grid.appendChild(card);
  });
}

function startQuiz(categoryKey) {
  state.categoryKey = categoryKey;
  state.questionIndex = 0;
  state.answers = {};
  showScreen('quiz');
  renderQuestion();
}

function renderQuestion() {
  const category = CATEGORIES[state.categoryKey];
  const question = category.questions[state.questionIndex];
  const total = category.questions.length;

  const dots = document.getElementById('progress-dots');
  dots.innerHTML = '';
  for (let i = 0; i < total; i++) {
    const dot = document.createElement('div');
    dot.className = 'dot';
    if (i < state.questionIndex) dot.classList.add('done');
    if (i === state.questionIndex) dot.classList.add('active');
    dots.appendChild(dot);
  }

  document.getElementById('quiz-question').textContent = question.text;

  const optionsEl = document.getElementById('quiz-options');
  optionsEl.innerHTML = '';
  question.options.forEach(function (opt) {
    const btn = document.createElement('button');
    btn.className = 'option-btn';
    btn.textContent = opt.label;
    btn.addEventListener('click', function () { selectAnswer(opt.value); });
    optionsEl.appendChild(btn);
  });
}

function selectAnswer(value) {
  const qKey = 'q' + (state.questionIndex + 1);
  state.answers[qKey] = value;
  state.questionIndex++;
  const category = CATEGORIES[state.categoryKey];
  if (state.questionIndex < category.questions.length) {
    renderQuestion();
  } else {
    showResults();
  }
}

function resetToGenre() {
  state.categoryKey = null;
  state.questionIndex = 0;
  state.answers = {};
  showScreen('genre');
}
```

**新しいコード(上記の代わりに挿入する):**

```js
function renderResultsHtml(category, ranked) {
  const first = ranked[0];
  const rest = ranked.slice(1);

  const firstHtml =
    '<div class="result-first">' +
    '<div class="result-rank">1位 おすすめ</div>' +
    '<h2>' + first.service.name + '</h2>' +
    '<div class="result-plan">無料: ' + first.service.freePlan + '<br>有料: ' + first.service.paidPlans + '</div>' +
    '<ul class="result-reasons">' +
      buildReasons(state.categoryKey, first.service, state.answers).map(function (r) { return '<li>' + r + '</li>'; }).join('') +
    '</ul>' +
    '<div class="result-risk">⚠️ ' + first.service.risk + '</div>' +
    '<div class="actions-center"><a class="btn-primary" href="https://' + first.service.officialUrl + '" target="_blank" rel="noopener">公式サイトを見る</a></div>' +
    '</div>';

  const othersHtml = '<div class="result-others">' + rest.map(function (entry, i) {
    const reasons = buildReasons(state.categoryKey, entry.service, state.answers);
    return '<div class="result-small">' +
      '<div class="result-rank">' + (i + 2) + '位</div>' +
      '<h3>' + entry.service.name + '</h3>' +
      '<p>' + reasons[0] + '</p>' +
      '<a class="btn-link small" href="https://' + entry.service.officialUrl + '" target="_blank" rel="noopener">公式サイトを見る</a>' +
      '</div>';
  }).join('') + '</div>';

  return firstHtml + othersHtml + '<p class="footer-note">' + category.footerNote + '</p>';
}

const state = {
  categoryKey: null,
  questionIndex: 0,
  answers: {}
};

const GENRE_ICONS = { llm: "💬", image: "🎨", video: "🎬", coding: "💻" };

function scrollToLatest(el) {
  el.scrollIntoView({ behavior: 'smooth', block: 'end' });
}

function addBotMessage(html, extraClass) {
  const chat = document.getElementById('chat');
  const msg = document.createElement('div');
  msg.className = 'msg bot' + (extraClass ? ' ' + extraClass : '');
  msg.innerHTML = '<div class="avatar"></div><div class="bubble">' + html + '</div>';
  chat.appendChild(msg);
  scrollToLatest(msg);
  return msg;
}

function addUserMessage(text) {
  const chat = document.getElementById('chat');
  const msg = document.createElement('div');
  msg.className = 'msg user';
  msg.innerHTML = '<div class="bubble user"></div>';
  msg.querySelector('.bubble').textContent = text;
  chat.appendChild(msg);
  scrollToLatest(msg);
}

function addQuickReplies(options, onSelect) {
  const chat = document.getElementById('chat');
  const wrap = document.createElement('div');
  wrap.className = 'quick-replies';
  options.forEach(function (opt) {
    const btn = document.createElement('button');
    btn.className = 'quick-reply-btn';
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

function showTyping(callback) {
  const chat = document.getElementById('chat');
  const msg = document.createElement('div');
  msg.className = 'msg bot';
  msg.id = 'typing-indicator';
  msg.innerHTML = '<div class="avatar"></div><div class="bubble typing"><span></span><span></span><span></span></div>';
  chat.appendChild(msg);
  scrollToLatest(msg);
  setTimeout(function () {
    const el = document.getElementById('typing-indicator');
    if (el) el.remove();
    callback();
  }, 500);
}

function buildProgressDots(currentIndex, total) {
  let html = '<div class="progress-dots">';
  for (let i = 0; i < total; i++) {
    let cls = 'dot';
    if (i < currentIndex) cls += ' done';
    if (i === currentIndex) cls += ' active';
    html += '<div class="' + cls + '"></div>';
  }
  html += '</div>';
  return html;
}

function startChat() {
  const chat = document.getElementById('chat');
  chat.innerHTML = '';
  state.categoryKey = null;
  state.questionIndex = 0;
  state.answers = {};

  const hero = document.createElement('div');
  hero.className = 'msg bot hero';
  hero.innerHTML =
    '<img class="hero-img" src="' + GORISUKE_IMG + '" alt="ゴリすけ">' +
    '<div class="bubble">やっほー!ゴリすけだよ🦍<br>AIツール選びを手伝うよ!</div>';
  chat.appendChild(hero);
  scrollToLatest(hero);

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

function selectGenre(categoryKey) {
  state.categoryKey = categoryKey;
  state.questionIndex = 0;
  state.answers = {};
  askNextQuestion();
}

function askNextQuestion() {
  const category = CATEGORIES[state.categoryKey];
  const question = category.questions[state.questionIndex];
  const total = category.questions.length;

  showTyping(function () {
    addBotMessage(buildProgressDots(state.questionIndex, total) + '<div class="quiz-question-text">' + question.text + '</div>');
    addQuickReplies(question.options, function (opt) {
      addUserMessage(opt.label);
      selectAnswer(opt.value);
    });
  });
}

function selectAnswer(value) {
  const qKey = 'q' + (state.questionIndex + 1);
  state.answers[qKey] = value;
  state.questionIndex++;
  const category = CATEGORIES[state.categoryKey];
  if (state.questionIndex < category.questions.length) {
    askNextQuestion();
  } else {
    showResultMessage();
  }
}

function showResultMessage() {
  const category = CATEGORIES[state.categoryKey];
  const ranked = rankServices(state.categoryKey, state.answers).slice(0, 3);
  showTyping(function () {
    addBotMessage('よし、診断結果が出たよ!');
    addBotMessage(renderResultsHtml(category, ranked), 'result');
    addQuickReplies([{ value: 'restart', label: 'もう一度診断する 🔄' }], function () {
      startChat();
    });
  });
}
```

- [ ] **Step 2: ファイル末尾の初期化コードを置き換え**

以下の3行(`runSelfTests` 定義の直後、ファイル末尾):

```js
document.getElementById('restart-btn').addEventListener('click', resetToGenre);
renderGenreScreen();
showScreen('genre');
```

を以下に置き換える:

```js
document.documentElement.style.setProperty('--gorisuke-avatar', 'url("' + GORISUKE_IMG + '")');
startChat();
```

- [ ] **Step 3: Commit**

```bash
cd /Users/yamadatoshi/yamada-ai-claude
git add ai-shindan.html
git commit -m "feat(gorisuke-chatbot): エンジンJSをチャット形式に書き換え"
```

---

### Task 6: セルフテストを実行して回帰がないことを確認

**Files:**
- (確認のみ、編集なし) `/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html`

スコアリングロジック(`budgetScore`/`extraAdjust`/`scoreService`/`buildReasons`/`rankServices`)とCATEGORIESデータは変更していないため、既存の12アサーションが全て通ることを確認する。

- [ ] **Step 1: スクリプト本体を抽出してNodeで実行**

```bash
cd /Users/yamadatoshi/yamada-ai-claude
start_line=$(grep -n "^const CATEGORIES" ai-shindan.html | head -1 | cut -d: -f1)
end_line=$(grep -n "^if (new URLSearchParams" ai-shindan.html | head -1 | cut -d: -f1)
sed -n "${start_line},$((end_line-1))p" ai-shindan.html > /tmp/gorisuke-test.js
echo "runSelfTests();" >> /tmp/gorisuke-test.js
node /tmp/gorisuke-test.js
```

Expected:

```
PASS: LLM: coding/無料/コスパ重視/プライバシー気にしない → DeepSeekが1位
PASS: LLM: プライバシー重視(Q4=A) → DeepSeekがトップ3から除外
PASS: 画像生成: business/無料/コスパ重視/商用利用 → Geminiが1位
PASS: 画像生成: photo/しっかり課金/画質重視/商用利用 → Geminiが1位
PASS: 画像生成: 同条件でGrok Imagineは1位にならない
PASS: 動画生成: audio/無料/音声重視/実写顔なし/個人利用 → Dreaminaが1位
PASS: 動画生成: photo/しっかり課金/画質重視/実写顔あり/商用利用 → Kling AIが1位
PASS: 動画生成: 同条件でGrok Imagineは1位にならない
PASS: コーディング: nocode/無料/コスパ重視/インストール平気/プライバシー気にしない → Replitが1位
PASS: コーディング: 同条件でClaude Codeが最下位
PASS: コーディング: vibe/無料/コスパ重視/ブラウザのみ/プライバシー重視 → Codexが1位
PASS: コーディング: 同条件でClaude Codeは1位にならない
セルフテスト: 12 pass / 0 fail
```

全て `PASS` で `12 pass / 0 fail` であればOK。`FAIL` がある場合はCATEGORIESデータかスコアリング関数を誤って編集していないか確認する。

- [ ] **Step 2: 後始終**

```bash
rm /tmp/gorisuke-test.js /tmp/gorisuke-360.png
```

---

### Task 7: Playwrightでチャットフローを手動確認

**Files:**
- (確認のみ、編集なし) `/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html`

- [ ] **Step 1: Playwright環境を準備**

```bash
mkdir -p /tmp/pw-gorisuke && cd /tmp/pw-gorisuke && npm init -y >/dev/null 2>&1 && npm install playwright >/dev/null 2>&1 && npx playwright install chromium 2>&1 | tail -2
```

Expected: Chromiumのダウンロード完了メッセージ、またはすでにインストール済みならエラーなく終了

- [ ] **Step 2: 全4カテゴリのチャットフローを検証するスクリプトを作成**

`/tmp/pw-gorisuke/verify.js` を以下の内容で作成:

```js
const { chromium } = require('playwright');
const path = require('path');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage({ viewport: { width: 480, height: 900 } });
  const fileUrl = 'file://' + path.resolve('/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html');

  page.on('pageerror', err => console.log('PAGE ERROR:', err.message));

  await page.goto(fileUrl);

  // ヒーローメッセージ確認
  await page.waitForSelector('.msg.hero .hero-img');
  console.log('hero image visible: OK');

  for (const categoryKey of ['llm', 'image', 'video', 'coding']) {
    // タイピング解除後、ジャンル選択のクイックリプライが出るまで待つ
    await page.waitForSelector('.quick-replies', { timeout: 3000 });
    const genreButtons = await page.$$('.quick-replies .quick-reply-btn');
    console.log(`[${categoryKey}] genre options: ${genreButtons.length}`);

    const idx = ['llm', 'image', 'video', 'coding'].indexOf(categoryKey);
    await genreButtons[idx].click();

    // 質問ループ: クイックリプライが出る → 最初の選択肢をクリック
    let qNum = 0;
    while (true) {
      await page.waitForSelector('.quick-replies', { timeout: 3000 });
      const options = await page.$$('.quick-replies .quick-reply-btn');
      qNum++;
      console.log(`[${categoryKey}] Q${qNum}: options=${options.length}`);
      await options[0].click();

      // 結果カードが出たら終了
      const resultCard = await page.$('.msg.bot.result');
      if (resultCard) break;
      if (qNum > 10) { console.log('TOO MANY QUESTIONS'); break; }
    }

    await page.waitForSelector('.msg.bot.result .result-first', { timeout: 3000 });
    const firstName = await page.textContent('.msg.bot.result .result-first h2');
    const link = await page.getAttribute('.msg.bot.result .result-first a.btn-primary', 'href');
    const othersCount = (await page.$$('.msg.bot.result .result-small')).length;
    const footerNote = await page.textContent('.msg.bot.result .footer-note');
    console.log(`[${categoryKey}] 1st place: ${firstName}`);
    console.log(`[${categoryKey}] official link: ${link}`);
    console.log(`[${categoryKey}] others count: ${othersCount}`);
    console.log(`[${categoryKey}] footer note: ${footerNote.slice(0, 30)}...`);

    // 「もう一度診断する」でリスタート
    await page.waitForSelector('.quick-replies', { timeout: 3000 });
    const restartBtns = await page.$$('.quick-replies .quick-reply-btn');
    await restartBtns[0].click();
    await page.waitForSelector('.msg.hero .hero-img', { timeout: 3000 });
    console.log(`[${categoryKey}] restart -> hero message OK`);
    console.log('---');
  }

  await browser.close();
})().catch(e => { console.error('FATAL', e); process.exit(1); });
```

- [ ] **Step 3: スクリプトを実行**

```bash
cd /tmp/pw-gorisuke && node verify.js
```

Expected:
- `hero image visible: OK`
- 各カテゴリで `genre options: 4`
- LLM/画像生成は `Q1`〜`Q4`(4問)、動画生成/コーディングは `Q1`〜`Q5`(5問)が出力される
- 各カテゴリで `1st place:`、`official link: https://...`、`others count: 2`、`footer note:` が出力される
- 各カテゴリで `restart -> hero message OK`
- `PAGE ERROR` が出力されないこと

エラーが出た場合は、該当するセレクタ・関数名がTask 5のコードと一致しているか確認する。

- [ ] **Step 4: アバター画像のクロップ位置を目視確認**

```js
// /tmp/pw-gorisuke/screenshot.js
const { chromium } = require('playwright');
const path = require('path');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage({ viewport: { width: 480, height: 700 } });
  const fileUrl = 'file://' + path.resolve('/Users/yamadatoshi/yamada-ai-claude/ai-shindan.html');
  await page.goto(fileUrl);
  await page.waitForTimeout(1200);
  await page.screenshot({ path: '/tmp/pw-gorisuke/chat.png' });
  await browser.close();
})();
```

```bash
cd /tmp/pw-gorisuke && node screenshot.js
```

`/tmp/pw-gorisuke/chat.png` を確認し、円形アバター内にゴリすけの顔がきれいに収まっているか確認する。ズレている場合は `ai-shindan.html` の `.avatar` ルールの `background-size`(現在 `250% auto`)と `background-position`(現在 `center 5%`)を調整し、再度スクリーンショットで確認する。

- [ ] **Step 5: 後始終**

```bash
rm -rf /tmp/pw-gorisuke
```

---

## Self-Review

**1. Spec coverage:**
- 会話フロー(ヒーロー→ジャンル選択→質問→結果→リスタート): Task 5の`startChat`/`selectGenre`/`askNextQuestion`/`selectAnswer`/`showResultMessage`で実装 ✓
- メッセージタイプ(ボットテキスト・クイックリプライ・結果カード・ヒーロー・ユーザー・タイピング): Task 3のCSS(`.bubble`, `.bubble.user`, `.bubble.typing`, `.hero-img`, `.quick-reply-btn`)とTask 5のJS(`addBotMessage`/`addUserMessage`/`addQuickReplies`/`showTyping`)で実装 ✓
- ビジュアル(gold/darkテーマ継続、アバター円形クロップ): Task 3の`:root`継続・`.avatar`ルールで実装、Task 7 Step 4で目視確認 ✓
- タイトル変更: Task 2で実装 ✓
- 画像埋め込み(リサイズ+base64): Task 1で実装 ✓
- セルフテスト12件の継続: Task 6で確認 ✓
- 手動確認項目(4カテゴリ全フロー、リンク、リスタート等): Task 7で実装 ✓

**2. Placeholder scan:** "TBD"/"後で"/"適切なエラー処理"等のプレースホルダーなし。全ステップに具体的なコード・コマンドを記載済み。

**3. Type consistency:**
- `state.categoryKey`/`questionIndex`/`answers` はTask 1完成時の構造を継続使用
- `CATEGORIES[key].label`/`.questions`/`.services`/`.footerNote` は既存データ構造のまま参照
- `buildReasons(categoryKey, service, answers)`/`rankServices(categoryKey, answers)` は既存シグネチャのまま呼び出し
- 新規関数名(`addBotMessage`/`addUserMessage`/`addQuickReplies`/`showTyping`/`buildProgressDots`/`startChat`/`selectGenre`/`askNextQuestion`/`selectAnswer`/`showResultMessage`/`renderResultsHtml`/`scrollToLatest`)はTask 5内で定義と使用が一致

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-06-13-gorisuke-chatbot.md`. ユーザーは事前に「お願いします」と承認済みのため、Inline Executionでこのまま実装を進める。
