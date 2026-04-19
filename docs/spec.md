# Gmail Reply Agent — 実装ガイド（Google Workspace 完結版）

すべて Google エコシステム内で完結する。外部サービスへの依存なし。

```
優先順位: Google Workspace 完結 > Google Antigravity > Claude Code
```

-----

## アーキテクチャ

```
Gmail
 └─ Apps Script Web App (UI + バックエンド)
      ├─ GmailApp              … メール取得・送信（Apps Script 標準）
      └─ Gemini API            … 返信案生成 / タスク抽出 / 文面修正
           └─ Google AI Studio キー（または Vertex AI）
```

- **AI モデル**: `gemini-2.0-flash`（高速・低コスト、Google AI Studio 無料枠あり）
- **認証**: Apps Script OAuth（Gmail） + Google AI Studio API キー（スクリプトプロパティ）
- **外部通信**: Google のサーバー間のみ（`generativelanguage.googleapis.com`）

-----

## ファイル構成

```
プロジェクト/
├── Code.gs     ← バックエンド（Gmail + Gemini API）
└── Index.html  ← チャット UI
```

-----

## Code.gs

```javascript
// ============================================================
// 定数
// ============================================================
const GEMINI_MODEL = "gemini-2.0-flash";
const GEMINI_ENDPOINT = `https://generativelanguage.googleapis.com/v1beta/models/${GEMINI_MODEL}:generateContent`;
const MAX_EMAILS = 20;

function getGeminiKey() {
  return PropertiesService.getScriptProperties().getProperty("GEMINI_API_KEY");
}

// ============================================================
// Web App エントリポイント
// ============================================================
function doGet() {
  return HtmlService.createHtmlOutputFromFile("Index")
    .setTitle("Gmail Reply Agent")
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

// ============================================================
// 未返信メール一覧取得
// ============================================================
function getUnrepliedEmails() {
  const threads = GmailApp.search("in:inbox is:unread -from:me", 0, MAX_EMAILS);

  return threads.map(thread => {
    const messages = thread.getMessages();
    const last = messages[messages.length - 1];
    return {
      threadId:     thread.getId(),
      messageId:    last.getId(),
      subject:      thread.getFirstMessageSubject(),
      from:         last.getFrom(),
      date:         last.getDate().toLocaleString("ja-JP"),
      body:         last.getPlainBody().slice(0, 3000),
      messageCount: messages.length,
    };
  });
}

// ============================================================
// Gemini API 共通呼び出し
// ============================================================
function callGemini(systemInstruction, userMessage) {
  const payload = {
    system_instruction: {
      parts: [{ text: systemInstruction }]
    },
    contents: [
      { role: "user", parts: [{ text: userMessage }] }
    ],
    generationConfig: {
      temperature: 0.4,
      maxOutputTokens: 1024,
    }
  };

  const options = {
    method: "post",
    contentType: "application/json",
    payload: JSON.stringify(payload),
    muteHttpExceptions: true,
  };

  const url = `${GEMINI_ENDPOINT}?key=${getGeminiKey()}`;
  const res = UrlFetchApp.fetch(url, options);
  const json = JSON.parse(res.getContentText());

  if (json.error) throw new Error(json.error.message);
  return json.candidates[0].content.parts[0].text;
}

// ============================================================
// 返信案 + タスク生成（初回）
// ============================================================
function generateReplyAndTasks(emailData) {
  const system = `あなたは優秀なビジネスメールアシスタントです。
受け取ったメールに対して以下を JSON 形式のみで返してください。前置きや説明は不要です。

{
  "reply": "そのまま送れる返信本文",
  "tasks": ["返信前に必要なタスク（なければ空配列）"]
}`;

  const user = `差出人: ${emailData.from}
件名: ${emailData.subject}
日時: ${emailData.date}

本文:
${emailData.body}`;

  const raw = callGemini(system, user);
  const clean = raw.replace(/```json|```/g, "").trim();
  return JSON.parse(clean);
}

// ============================================================
// 返信案の修正
// ============================================================
function modifyReply(currentReply, instruction, emailBody) {
  const system = `あなたは優秀なビジネスメールアシスタントです。
現在の返信案をユーザーの指示に従って修正してください。
修正後の返信本文のみを返してください。前置きや説明は不要です。`;

  const user = `【元のメール本文（参考）】
${emailBody.slice(0, 1500)}

【現在の返信案】
${currentReply}

【修正指示】
${instruction}`;

  return callGemini(system, user);
}

// ============================================================
// 送信意図の判定
// ============================================================
function interpretSendIntent(instruction) {
  const system = `ユーザーの発言が「メールを今すぐ送信してよい」という意図かを判定してください。
"true" か "false" のみ返してください。それ以外は返さないでください。`;

  const result = callGemini(system, instruction);
  return result.trim().toLowerCase() === "true";
}

// ============================================================
// メール送信
// ============================================================
function sendReply(messageId, replyBody) {
  const message = GmailApp.getMessageById(messageId);
  const thread = message.getThread();
  thread.reply(replyBody);
  thread.markRead();
  return { success: true, subject: thread.getFirstMessageSubject() };
}
```

-----

## Index.html

```html
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Gmail Reply Agent</title>
<style>
  :root {
    --bg: #0f0f13;
    --surface: #1a1a22;
    --border: #2e2e3e;
    --accent: #4285f4;
    --accent2: #34a853;
    --text: #e8e8f0;
    --muted: #888898;
    --user-bubble: #4285f422;
    --ai-bubble: #1e1e2e;
  }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body {
    font-family: 'Hiragino Sans', 'Yu Gothic UI', sans-serif;
    background: var(--bg); color: var(--text);
    height: 100vh; display: flex; flex-direction: column;
  }
  header {
    padding: 14px 20px; border-bottom: 1px solid var(--border);
    display: flex; align-items: center; gap: 10px;
    background: var(--surface);
  }
  header .logo { font-size: 18px; font-weight: 700; color: var(--accent2); }
  header .sub  { font-size: 12px; color: var(--muted); }
  header .badge {
    margin-left: 8px; font-size: 10px; background: #1a3a2a;
    color: var(--accent2); border: 1px solid var(--accent2);
    border-radius: 4px; padding: 2px 6px;
  }
  #refreshBtn {
    margin-left: auto; background: var(--accent); color: #fff;
    border: none; border-radius: 6px; padding: 6px 14px;
    font-size: 13px; cursor: pointer;
  }
  .layout { display: flex; flex: 1; overflow: hidden; }
  #emailList {
    width: 300px; border-right: 1px solid var(--border);
    overflow-y: auto; background: var(--surface); flex-shrink: 0;
  }
  .email-item {
    padding: 12px 14px; border-bottom: 1px solid var(--border);
    cursor: pointer; transition: background 0.15s;
  }
  .email-item:hover { background: #252530; }
  .email-item.active { background: #1a2a40; border-left: 3px solid var(--accent); }
  .email-item .sender { font-size: 12px; color: var(--accent2); font-weight: 600; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .email-item .subj   { font-size: 13px; font-weight: 500; margin-top: 3px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .email-item .meta   { font-size: 11px; color: var(--muted); margin-top: 2px; }
  #chatArea { flex: 1; display: flex; flex-direction: column; overflow: hidden; }
  #emailPreview {
    padding: 12px 18px; border-bottom: 1px solid var(--border);
    background: var(--surface); font-size: 12px; color: var(--muted);
    max-height: 90px; overflow-y: auto; line-height: 1.5; display: none;
  }
  #messages {
    flex: 1; overflow-y: auto; padding: 16px 18px;
    display: flex; flex-direction: column; gap: 12px;
  }
  .msg { max-width: 85%; padding: 12px 16px; border-radius: 12px; font-size: 14px; line-height: 1.65; white-space: pre-wrap; word-break: break-word; }
  .msg.user   { align-self: flex-end; background: var(--user-bubble); border: 1px solid var(--accent); }
  .msg.ai     { align-self: flex-start; background: var(--ai-bubble); border: 1px solid var(--border); }
  .msg.system { align-self: center; font-size: 12px; color: var(--muted); background: transparent; border: none; padding: 4px; }
  .tasks-box { background: #1a2a1a; border: 1px solid #2a4a2a; border-radius: 8px; padding: 10px 14px; margin-top: 8px; font-size: 13px; }
  .tasks-box .tasks-title { color: var(--accent2); font-weight: 600; margin-bottom: 6px; font-size: 12px; }
  .tasks-box li { margin-left: 16px; margin-top: 3px; }
  #inputArea { padding: 12px 16px; border-top: 1px solid var(--border); background: var(--surface); display: flex; gap: 8px; align-items: flex-end; }
  #userInput {
    flex: 1; background: var(--bg); border: 1px solid var(--border);
    border-radius: 8px; color: var(--text); padding: 10px 12px;
    font-size: 14px; resize: none; min-height: 42px; max-height: 120px; font-family: inherit; line-height: 1.5;
  }
  #userInput:focus { outline: none; border-color: var(--accent); }
  #sendBtn {
    background: var(--accent); color: #fff; border: none; border-radius: 8px;
    padding: 10px 18px; font-size: 14px; cursor: pointer; flex-shrink: 0; font-family: inherit;
  }
  #sendBtn:disabled { opacity: 0.4; cursor: default; }
  .placeholder { color: var(--muted); font-size: 14px; text-align: center; margin: auto; }
  .spinner { display: inline-block; width: 14px; height: 14px; border: 2px solid var(--muted); border-top-color: var(--accent); border-radius: 50%; animation: spin 0.7s linear infinite; vertical-align: middle; margin-right: 6px; }
  @keyframes spin { to { transform: rotate(360deg); } }
</style>
</head>
<body>
<header>
  <div>
    <div class="logo">✉ Reply Agent <span class="badge">Gemini</span></div>
    <div class="sub">Google Workspace 完結 · 未返信メール AI 処理</div>
  </div>
  <button id="refreshBtn" onclick="loadEmails()">🔄 更新</button>
</header>

<div class="layout">
  <div id="emailList"><div class="placeholder" style="padding:20px;font-size:13px;">読み込み中...</div></div>
  <div id="chatArea">
    <div id="emailPreview"></div>
    <div id="messages"><div class="placeholder">← メールを選択してください</div></div>
    <div id="inputArea">
      <textarea id="userInput" placeholder="修正指示や「送って」など自然言語で入力..." rows="1"
        onkeydown="handleKey(event)"></textarea>
      <button id="sendBtn" onclick="handleSend()" disabled>→</button>
    </div>
  </div>
</div>

<script>
let emails = [], currentEmail = null, currentDraft = "", state = "idle";

function loadEmails() {
  document.getElementById("emailList").innerHTML =
    '<div class="placeholder" style="padding:20px;font-size:13px;"><span class="spinner"></span>取得中...</div>';
  google.script.run
    .withSuccessHandler(renderEmailList)
    .withFailureHandler(e => alert("エラー: " + e.message))
    .getUnrepliedEmails();
}

function renderEmailList(data) {
  emails = data;
  const el = document.getElementById("emailList");
  if (!data.length) { el.innerHTML = '<div class="placeholder" style="padding:20px;font-size:13px;">未返信メールなし 🎉</div>'; return; }
  el.innerHTML = data.map((m, i) => `
    <div class="email-item" id="ei${i}" onclick="selectEmail(${i})">
      <div class="sender">${esc(m.from.split("<")[0].trim())}</div>
      <div class="subj">${esc(m.subject)}</div>
      <div class="meta">${m.date} · ${m.messageCount}通</div>
    </div>`).join("");
}

function selectEmail(i) {
  document.querySelectorAll(".email-item").forEach(e => e.classList.remove("active"));
  document.getElementById("ei" + i).classList.add("active");
  currentEmail = emails[i]; currentDraft = ""; state = "drafting";
  const preview = document.getElementById("emailPreview");
  preview.style.display = "block";
  preview.textContent = `件名: ${currentEmail.subject}\nFrom: ${currentEmail.from}\n\n${currentEmail.body.slice(0, 300)}...`;
  const msgs = document.getElementById("messages");
  msgs.innerHTML = "";
  addMsg("system", `「${currentEmail.subject}」を選択`);
  addMsg("ai", '<span class="spinner"></span>Gemini が返信案を生成中...');
  document.getElementById("sendBtn").disabled = true;
  google.script.run
    .withSuccessHandler(onDraftReady)
    .withFailureHandler(e => addMsg("ai", "エラー: " + e.message))
    .generateReplyAndTasks(currentEmail);
}

function onDraftReady(result) {
  currentDraft = result.reply;
  document.getElementById("messages").lastChild.remove();
  if (result.tasks && result.tasks.length) {
    addMsg("ai", `<div class="tasks-box"><div class="tasks-title">📋 返信前に必要なタスク</div><ul>${result.tasks.map(t => `<li>${esc(t)}</li>`).join("")}</ul></div>`, true);
  }
  addMsg("ai", `【返信案】\n\n${currentDraft}\n\n---\n修正指示を入力するか、「このまま送って」と伝えてください。`);
  state = "ready";
  document.getElementById("sendBtn").disabled = false;
}

function handleKey(e) { if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); handleSend(); } }

function handleSend() {
  const input = document.getElementById("userInput");
  const text = input.value.trim();
  if (!text || state === "idle" || !currentEmail) return;
  input.value = "";
  addMsg("user", text);
  document.getElementById("sendBtn").disabled = true;
  google.script.run
    .withSuccessHandler(isSend => isSend ? doSend() : doModify(text))
    .withFailureHandler(e => addMsg("ai", "エラー: " + e.message))
    .interpretSendIntent(text);
}

function doModify(instruction) {
  addMsg("ai", '<span class="spinner"></span>修正中...');
  google.script.run
    .withSuccessHandler(newReply => {
      document.getElementById("messages").lastChild.remove();
      currentDraft = newReply;
      addMsg("ai", `【修正後の返信案】\n\n${currentDraft}\n\n---\nさらに修正するか「送って」と伝えてください。`);
      document.getElementById("sendBtn").disabled = false;
    })
    .withFailureHandler(e => addMsg("ai", "エラー: " + e.message))
    .modifyReply(currentDraft, instruction, currentEmail.body);
}

function doSend() {
  addMsg("ai", '<span class="spinner"></span>送信中...');
  google.script.run
    .withSuccessHandler(res => {
      document.getElementById("messages").lastChild.remove();
      addMsg("ai", `✅ 送信完了\n「${res.subject}」への返信を送りました。`);
      currentEmail = null; currentDraft = ""; state = "idle";
      loadEmails();
    })
    .withFailureHandler(e => addMsg("ai", "エラー: " + e.message))
    .sendReply(currentEmail.messageId, currentDraft);
}

function addMsg(role, html, isHtml = false) {
  const div = document.createElement("div");
  div.className = "msg " + role;
  if (isHtml || html.includes("<span")) div.innerHTML = html;
  else div.textContent = html;
  document.getElementById("messages").appendChild(div);
  div.scrollIntoView({ behavior: "smooth" });
}
function esc(s) { return String(s).replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;"); }
loadEmails();
</script>
</body>
</html>
```

-----

## セットアップ手順

### 1. Google AI Studio で API キーを取得

1. [aistudio.google.com](https://aistudio.google.com) を開く
1. 「Get API key」→「Create API key」
1. キーをコピー（無料枠: 15 RPM / 100万トークン/日）

### 2. Apps Script プロジェクトを作成

1. [script.google.com](https://script.google.com) →「新しいプロジェクト」
1. `コード.gs` を `Code.gs` にリネームして上記コードを貼り付け
1. ファイル追加（+）→「HTML」→ 名前 `Index` にして上記 HTML を貼り付け

### 3. API キーを登録

「プロジェクトの設定（⚙）」→「スクリプトプロパティ」→ プロパティを追加：

|プロパティ名          |値        |
|----------------|---------|
|`GEMINI_API_KEY`|`AIza...`|

### 4. 権限承認 & デプロイ

```
「実行」→「doGet を実行」→ Gmail 権限を承認
「デプロイ」→「新しいデプロイ」→ 種類：ウェブアプリ
  実行ユーザー: 自分
  アクセス権: 自分のみ（または組織内ユーザー）
```

表示された URL をブックマーク → 完成。

-----

## 通信範囲（Google Workspace 完結の根拠）

|通信先       |用途      |ドメイン                               |
|----------|--------|-----------------------------------|
|Gmail サーバー|メール取得・送信|`*.google.com`                     |
|Gemini API|AI 処理   |`generativelanguage.googleapis.com`|

Anthropic・OpenAI 等への通信は一切なし。

-----

## フォールバック戦略

```
① Google Workspace 完結（本実装）← 現在ここ
   └─ もし詰まったら:
      ↓
② Google Antigravity（VS Code + Gemini）
   └─ clasp でローカル開発 → Apps Script にデプロイ
      添付ファイル解析・複雑なロジック追加が容易
      ↓
③ Claude Code
   └─ callGemini → callAnthropic に差し替えるだけ
      エンドポイントと認証ヘッダーの変更のみ
```

-----

## カスタマイズ早見表

|やりたいこと               |変更箇所                                                         |
|---------------------|-------------------------------------------------------------|
|検索条件を変えたい            |`Code.gs` の `GmailApp.search()` クエリ文字列                       |
|返信のトーン固定             |`generateReplyAndTasks()` の system 文字列に追記                    |
|署名を自動付与              |`sendReply()` で `replyBody += "\n\n" + 署名文字列`                |
|Google Tasks に自動登録   |Tasks 高度なサービスを有効化 → `Tasks.newTask()`                        |
|Gemini → Claude に切り替え|`callGemini` を `callAnthropic` に差し替え（エンドポイントと認証ヘッダーのみ）       |
|モデルを変えたい             |`GEMINI_MODEL` 定数を変更（`gemini-2.0-flash` / `gemini-1.5-pro` 等）|
