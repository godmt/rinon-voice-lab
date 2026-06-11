# キャラクター定義と追加方法の解析

## 1. 編集元と自動生成物

利用者が手作業で編集する中心は `Character/<id>/profile.txt` です。

| ファイル | 役割 |
| --- | --- |
| `Character/<id>/profile.txt` | 人間向けの編集元 |
| `Character/<id>/profile.json` | キャラクター単位の構造化コピー |
| `profiles/characters.json` | 全キャラクターと選択状態をまとめた実行時スナップショット |

JSON類はUI・サーバ処理の利便性のために自動生成されます。`profiles/` は `.gitignore` 対象です。

## 2. 最小構成

正常動作が確認されている `usako` の追加例から、最小構成は次の通りです。

```text
Character/
  usako/
    profile.txt
    reference/
      usako_ref.wav
    expressions/
      neutral/
        neutral.png
```

`profile.json` は読み込み・保存時に生成できます。

## 3. `profile.txt` の形式

```text
# Rinon Voice Lab character profile
id: example
name: キャラクター名
referencePath: reference\example.wav
portrait: /Character/example/expressions/neutral/neutral.png

[systemPrompt]
人格と会話方針

[ttsCaption]
VoiceDesign用の演技説明

[expressions]
neutral=/Character/example/expressions/neutral/neutral.png
happy=/Character/example/expressions/happy/happy.png
shy=/Character/example/expressions/shy/shy.png|/Character/example/expressions/shy/shy_02.png
```

### パーサの規則

- 空行と `#` で始まる行は無視
- セクション外の `key: value` を基本項目として読む
- `[systemPrompt]` と `[ttsCaption]` は複数行を保持
- `[expressions]` は `key=url|url` 形式
- 未知セクションは作られるが、キャラクターデータには採用されない

## 4. 読み込み優先順位

キャラクターフォルダ内では更新日時を比較します。

```text
profile.txtが存在し、
profile.jsonがない、またはTXTの更新日時がJSON以上
  -> profile.txtを読む

それ以外
  -> profile.jsonを読む
```

その後、`profiles/characters.json` の全体データへ、キャラクターフォルダ側の定義をID単位で上書きします。

最後に正規化・素材コピーを行い、`profile.txt`, `profile.json`, `profiles/characters.json` を再生成します。

したがって、手動編集後は「キャラ読込」を実行し、TXTの内容をシステム表現へ反映させる運用です。

## 5. IDの正規化

キャラクターIDは小文字化され、使用できない文字を `_` に変換します。

使用可能:

- `a-z`
- `0-9`
- `_`
- `-`

最大48文字です。重複IDには `_2`, `_3` が付与されます。

IDはフォルダ名とURLに使われるため、表示名とは分離されています。

## 6. `systemPrompt` の設計

この項目は毎ターンLLMのsystem messageへ直接入ります。

有効な構成例:

1. 成人性・外見・基本属性
2. 性格
3. 会話の長さと距離感
4. 得意な話題
5. 疲労時や興奮時の変化
6. 口調・語尾・口癖
7. 相談時の方針
8. 禁止事項
9. 数件の会話例

`usako` は平常時、疲労時、趣味で早口になる時を文章で条件分岐させています。単一の性格形容詞より、状況と反応を対で書く方がLLMの挙動を制御しやすい実例です。

## 7. `ttsCaption` の設計

人格プロンプトとTTS Captionは別目的です。

| 項目 | 対象モデル | 内容 |
| --- | --- | --- |
| `systemPrompt` | LLM | 何をどう話すか |
| `ttsCaption` | Irodori-TTS | どんな声・演技で話すか |

LLM人格に「眠そう」と書いても、TTSへ自動的には伝わりません。恒常的な声の傾向は `ttsCaption` にも記述する必要があります。

一方、そのターンだけの感情は `emojiStyle` が担当します。

## 8. 参照音声

`referencePath` はキャラクターフォルダ相対で書くと移植しやすくなります。

```text
referencePath: reference\example.wav
```

読み込み後の実行時データでは絶対パスへ変換されますが、ファイルへ保存し直す際は相対パスへ戻されます。

## 9. 表情差分

表情キーは `expression_for_emoji()` が返すキーに合わせると自動感情選択で使われます。

基本候補:

- `neutral`
- `happy`
- `surprised`
- `soft`
- `angry`
- `worried`
- `sad`
- `shy`
- `teasing`
- `tender`
- `question`
- `sleepy`

独自キーも登録できますが、現在の絵文字対応表が返さないキーは、自動感情選択からは到達しません。手動または将来のコード拡張向けです。

## 10. 素材の自己完結化

UIで既存の共通画像や外部位置の参照音声を指定して保存すると、サーバは可能なものをキャラクターフォルダへコピーします。

- 画像: `expressions/<key>/`
- 音声: `reference/`

同名で内容が違う場合は、短いUUIDを付けて衝突を避けます。

この処理により、キャラクターフォルダを単位として持ち運びやすくしています。

## 11. UI編集時の流れ

UIで保存すると:

1. ブラウザ内のキャラクター辞書をJSON化
2. `/api/characters` へPOST
3. ID、文字列、表情配列をサニタイズ
4. 素材をキャラフォルダへ集約
5. キャラごとのTXT/JSONを生成
6. 全体の `profiles/characters.json` を生成
7. 正規化後のデータをUIへ返す

UI編集とTXT編集は同じ内部キャラクターデータへ収束します。

## 12. 注意点

- `profile.txt` と `profile.json` を両方手編集しない。
- TXT編集後は、JSONより更新日時が新しい状態で読み込む。
- 同名キャラクターは話者表示判定を曖昧にする可能性がある。
- `neutral` 画像は必ず用意するのが安全。
- 独自表情キーは自動絵文字対応に追加しない限り自動選択されない。
- 参照音声や画像の著作権・利用条件はキャラクター単位で確認する。

