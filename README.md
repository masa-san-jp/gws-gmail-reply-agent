# gws-gmail-reply-agent

Gmail 返信支援を行う Google Workspace 向けエージェントです。  
An agent for assisting Gmail reply workflows in Google Workspace environments.

## 概要

この README は、このリポジトリの役割を示すための最小 README です。
詳細な使い方、設計、運用ルールが必要な場合は今後追加します。

## 現在の収録内容

現時点の既定ブランチには、このREADMEと[実装ガイド](docs/spec.md)を収録しています。`Code.gs`と`Index.html`はガイド内のコード例であり、独立したソースファイルやデプロイ済みサービスを同梱しているわけではありません。

## 用例と設計思想

想定する利用例は、「日程確認メールを選ぶ → 返信案と返信前のタスクを確認する → 『もう少し短く』と修正を指示する → 宛先・内容を確認して送信する」という流れです。「返信支援」は案の作成と修正を助けることであり、相手への約束や送信判断は利用者が確認する必要があります。

実装ガイドは、Gmail・Apps Script・Geminiを組み合わせ、Googleのサービス内で処理を構成する方針を示しています。ただし「Google内」は端末内処理という意味ではなく、メールの差出人・件名・本文などがGemini APIに送られる設計です。組織のデータ取扱方針とAPI利用条件を確認してから導入してください。

## 技術的背景と注意点

- Apps Script Web AppがUIとバックエンドを担い、GmailAppでメールを取得・返信し、UrlFetchAppでGemini APIを呼び出す構成例です。
- GmailアクセスはApps ScriptのOAuth権限、Geminiのキーはスクリプトプロパティで管理する想定です。
- ガイドの「未返信」一覧は実際には `in:inbox is:unread -from:me` の検索結果です。未読と未返信は一致しないため、返信済み判定として保証されません。
- 例示コードは自然言語の送信意図をAIが判定し、その結果で送信関数を呼びます。独立した最終確認ダイアログがあるわけではありません。本番利用前には誤送信対策、二重送信防止、例外処理、アクセス範囲を検証してください。

## 成立と展開

[2026年4月19日に実装ガイドが追加](https://github.com/masa-san-jp/gws-gmail-reply-agent/commit/321f981a9ceb4fec552c106a5174933090511fab)され、[5月12日に最小READMEが追加](https://github.com/masa-san-jp/gws-gmail-reply-agent/commit/7345fbc8876c0dde171a15b28a5bb268c0be6526)されています。これは文書の成立履歴であり、本番稼働の実績ではありません。

ガイドには検索条件・返信トーン・署名の変更、Google Tasks連携などの拡張案があります。Tasks連携等は追加実装を要します。掲載モデル名、利用枠、認証・デプロイ条件はガイド作成時の記述として扱い、導入時には提供元の現行仕様を確認してください。
