# TTSと感情制御の解析

## 1. 感情制御の入力層

Irodori-TTSへ渡る発話は、次の要素を合成したものです。

| 層 | データ | 時間スケール |
| --- | --- | --- |
| 話者性 | 参照音声 `referencePath` | キャラクター単位 |
| 演技・声質 | `ttsCaption` | キャラクター単位 |
| 瞬間的スタイル | `emojiStyle` | 発話ターン単位 |
| 発話本文 | LLM返答の各チャンク | 文単位 |
| 話速 | `durationScale` | ターン単位 |
| 品質・計算量 | `steps` | ターン単位 |

キャラクターの恒常的な声と、その場の感情を分離している点が重要です。

## 2. TTS本文の加工

LLM返答にはIrodori絵文字を残しません。`strip_irodori_style_marks()` で既知の絵文字を除去した後、TTS直前に `apply_emoji_style()` が選択された絵文字を先頭へ1つだけ付けます。

```text
表示本文: 今日は少し眠いかも。
emojiStyle: 😪
TTS入力: 😪今日は少し眠いかも。
```

この処理により、画面表示用テキストと音声制御用テキストを分離しつつ、Irodori-TTSには期待する記法で渡せます。

## 3. `ttsCaption` の役割

`ttsCaption` はVoiceDesignの自然言語条件です。例では、年齢感、声の高さ、演技、距離感、息遣い、発音、録音品質などを英語で記述しています。

研究上は、次の要素を分けて実験できます。

- 声質: bright, smoky, low resonance
- キャラクター性: teasing, friendly, confident
- 演技: intimate conversational acting, restrained confidence
- 音響品質: clear pronunciation, clean studio sound
- 状態: sleepy softness, energetic, relaxed

参照音声だけでなく、captionにもキャラクター性を持たせているため、両者が競合した場合の寄与率は実験対象になります。

## 4. 参照音声のパス変換

ユーザーがUIから音声を追加すると、ブラウザはData URL形式のbase64を `/api/reference` へ送ります。

サーバは次を行います。

1. 拡張子を許可リストで検査する。
2. base64を厳密モードでデコードする。
3. 80 MiBを上限にする。
4. `Character/<id>/reference/` へ一意な名前で保存する。
5. 実行時には絶対パスを返す。
6. `profile.txt/json` へ保存するときはキャラフォルダ相対パスへ戻す。

この「実行時は絶対パス、配布用定義は相対パス」という変換により、インストール場所に依存しにくくしています。

無効な拡張子、存在しないパスの場合は既定参照音声へフォールバックします。

## 5. 話速制御

UIの話速は2段階です。

| UI値 | `durationScale` |
| --- | ---: |
| normal | 1.0 |
| fast | 0.86 |

文字列を速読向けに書き換えるのではなく、Irodori-TTSのduration scaleへ直接渡します。返却データにもscaleを含めるため、ログから条件を再確認できます。

## 6. Irodori-TTS呼び出し

ローカル生成では、Irodori-TTSのGradio UIをHTTP経由で操作せず、`gradio_app_voicedesign._run_generation()` をPythonモジュールとして直接呼びます。

主な固定・可変値:

| 引数 | 値 |
| --- | --- |
| checkpoint | `Aratako/Irodori-TTS-600M-v3-VoiceDesign` |
| text | 絵文字を先頭付与したチャンク |
| caption | キャラクターのTTS Caption |
| ref wav | キャラクター参照音声 |
| steps | UI値 |
| duration scale | 1.0または0.86 |
| schedule | `linear` |
| CFG mode | `independent` |
| CFG text | 3.0 |
| CFG caption | 4.0 |
| CFG speaker | 5.0 |

CFG scaleが `speaker > caption > text` の順で固定されている点は、話者再現を強めつつ、演技captionと本文を制御する意図と推測できます。これはコードから読める設定値に基づく推測です。

## 7. 実行デバイスと精度

デバイスが `auto` の場合、Irodori-TTS側の `default_runtime_device()` を使います。

精度が `auto` の場合:

- CUDA/XPUでbf16が利用可能ならbf16
- それ以外は利用可能な先頭候補
- 候補がなければfp32

モデルとcodecは別々のdevice/precision項目を持ちますが、既定では同じ自動選択結果を使います。

## 8. 生成の直列化

`Irodori_lock` により、同一プロセス内の生成は直列化されています。HTTPサーバ自体はマルチスレッドですが、モデル推論を同時実行させません。

利点:

- GPUメモリ競合を避けやすい
- Irodori側のグローバル状態や作業ディレクトリ変更を保護できる

制約:

- 複数ユーザーや複数チャンクでも並列生成されない
- 長い返答ほど待ち時間が累積する

## 9. Irodori出力の回収

`_run_generation()` の戻り値から、`saved[1]: ...` という文字列を正規表現で探し、生成wavの場所を取得しています。

そのwavを `static/generated/` へコピーし、ブラウザから読める `/generated/...wav` に変換します。

これはIrodori内部APIの正式な構造化戻り値ではなく、表示用detail文字列の形式に依存しています。Irodori側の出力形式変更に弱い接続点です。

## 10. リモートTTS

2P音声は次の2経路を持ちます。

### HTTPサーバ経路

別PCの `/synthesize` へJSONを送り、wavをbase64で受け取ります。通信しやすい一方、音声全体をJSON内のbase64として運ぶため、バイナリよりサイズが増えます。

### SSH/CLIフォールバック

HTTPが失敗し、SSH設定がある場合:

1. 生成要求をJSONファイルにする。
2. SCPで別PCへ送る。
3. PowerShell EncodedCommandでIrodoriの `infer.py` を実行する。
4. wavをSCPで取得する。

PowerShellスクリプトをUTF-16LEからbase64化して渡すのは、SSH越しの引用符や日本語の問題を避ける実務的テクニックです。

## 11. TTS結果に付与されるメタデータ

各音声チャンクには次が返ります。

- 表示本文 `text`
- 実際のTTS入力 `ttsText`
- caption
- reference
- emojiStyle
- speechRate / durationScale
- model/codec deviceとprecision
- expression
- 公開URL
- 生成時間
- 元wavの場所

音声だけでなく生成条件を残すため、再現性や実験比較に使いやすい構造です。

## 12. 研究上の実験候補

- 同じ参照音声でcaptionだけを変える。
- 同じcaptionで参照音声だけを変える。
- 同じ本文でemojiStyleだけを変える。
- CFG scale固定値を変え、話者性・演技・明瞭度を比較する。
- 文分割単位を句読点、文字数、意味節で比較する。
- 同一絵文字が短文・長文でどの程度安定するか測る。

