# 解析ロードマップ

この文書は、Rinon Voice Lab から「キャラクターがしゃべる仕組み」「表情差分」「音声感情コントロール」「記憶」を順に読み取るための作業索引です。初回は全体像を整理し、次回以降に各章を深掘りしていきます。

## 優先して知るべき要素

| 優先 | テーマ | 読む主な場所 | 研究上の問い |
| --- | --- | --- | --- |
| 1 | 会話生成パイプライン | `static/app.js`, `app.py` の `/api/chat`, `request_lmstudio()` | キャラ人格、履歴、ユーザー入力はどのようにLLMへ渡るか |
| 2 | Irodori-TTS連携 | `synthesize_sentence()`, `tools/remote_luvia_tts_server.py` | 声質・感情・話速・参照音声はどの引数で制御されるか |
| 3 | 表情差分同期 | `expression_for_emoji()`, `randomExpressionImage()`, `Character/*/expressions/` | 音声感情と立ち絵差分はどう同期されるか |
| 4 | キャラクター定義 | `Character/*/profile.txt`, `profile.json`, `/api/characters` | キャラ追加や人格編集に必要な最小データは何か |
| 5 | 記憶と履歴 | `profiles/latest_session.json`, `logs/chat.jsonl`, `compact_messages_for_context()` | 短期履歴、永続保存、要約はどう分かれているか |
| 6 | 2P/自動会話 | `startAutoConversation()`, `continueAutoConversation()`, `activeStage()` | キャラ同士の会話はどのようにターン制御されるか |
| 7 | 外部Speak | `/api/speak`, `/api/speak-events` | Codex等の外部ツールからキャラを喋らせる入口は何か |
| 8 | Web検索メモ | `web_search()`, `format_web_results()` | 外部情報は記憶ではなく一時コンテキストとしてどう注入されるか |

## 作成済みの解析文書

1. [会話生成パイプライン](chat-pipeline.ja.md)
   - `/api/chat` の入力JSON、処理順、返却JSONを具体例で整理する。

2. [TTSと感情制御](tts-and-emotion.ja.md)
   - Irodori-TTSに渡す引数、`ttsCaption`、絵文字、参照音声、話速の関係をまとめる。

3. [表情差分システム](expression-system.ja.md)
   - 絵文字から表情キーへの対応、キャラクター別差分フォルダ、ランダム差分選択を図示する。

4. [記憶・履歴システム](memory-system.ja.md)
   - `history`, `latest_session.json`, `chat.jsonl`, コンテキスト圧縮の役割分担を整理する。

5. [キャラクター定義と追加方法](character-authoring.ja.md)
   - ユーザーが編集する `profile.txt` と、`reference`, `expressions` の実用構成を整理する。JSON類は自動生成物として扱う。

## 初回解析で見えた要点

- 会話本文はLM Studioが生成し、音声はIrodori-TTSが生成する。
- Irodoriのスタイル絵文字は、TTSの感情制御とUI表情切替の共通キーとして使われている。
- 表情画像はリアルタイム生成ではなく、キャラクターごとに用意された差分を選ぶ方式。
- 記憶はベクトルDB型ではなく、ブラウザ履歴、セッションJSON、ログJSONL、コンテキスト要約で構成される。
- 2P会話も特別なLLMではなく、話者とプロンプトを切り替えて同じ `/api/chat` を使う。
- 2P音声だけ別PCで生成できる設計があり、GPU負荷分散の実験に向いている。
- 日本語ファイルはUTF-8で正常に読める。こちらの端末出力ではUTF-8を明示して読む必要がある。

## 調査時に守る方針

- MIT License由来の研究・解析であることを明記し、ライセンス本文は改変しない。
- 未追跡ファイル、特に `Character/usako/` はユーザー追加物として扱い、必要があるまで編集しない。
- 解析メモは `docs/` に追記し、コード変更とは分ける。
- 実装を変更する前に、まず現在の設計意図を読み取る。
