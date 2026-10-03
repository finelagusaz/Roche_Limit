<!-- devkit:ghost-template -->
<!-- 上の行は「まだ書き終えていない」しるしです。書き終えたら、上の行とこの行を消してください。 -->

# GHOST.md

このゴーストに固有の情報です。AI エージェントは、作業の前に `AGENTS.md` とあわせて読みます。
作者が自由に書き換えるファイルで、開発キットの更新（`tools/update-devkit.ps1`）で上書きされることはありません。

## このゴーストについて

- 名前（`ghost/master/descript.txt` の `name`）: Roche_Limit（ロッシュの限界）
- キャラクター（`sakura.name` / `kero.name`）: `\0` = オフィーリア / `\1` = エプシロン
- 作者: Fine Lagusaz
- 配布先・ネットワーク更新の URL:
- 元にしたテンプレートやゴースト:

## ライセンス

- 辞書:
- シェル: （作者と利用条件。改変した画像の配布や商用利用ができるか）

## 文字コードと改行（最重要・編集前に必読）

ファイル種別ごとに**文字コードが異なる**。誤ると即座に文字化け・破損する。

- **`.dic`（YAYA 辞書）= UTF-8 / BOM無し**。`yaya.txt` で `charset.dic, UTF-8` 指定。
  - 改行は**既存ファイルの改行を保つ**こと。2026-10-04 の実測では、作業ツリー上で CRLF のもの（`rs_menu.dic` など大半）と LF のもの（`aitalks/*.dic`、`rs_aitalk.dic`、`rs_aitalk_old.dic`、`rs_communicate.dic`）が混在している。git のインデックス上はほぼ LF（`core.autocrlf=input` が add 時に LF 化する。例外は `rs_menu.dic` で CRLF のまま）。
  - スクリプトで `.dic` を生成・整形する場合、Python なら `io.open(..., newline="")` でバイト保存し、元の改行を維持すること。
  - `.gitattributes` / `.editorconfig` は、この混在を書き換えないように手直ししてある（キットの seed の `* text=auto eol=lf` / `end_of_line = lf` は使っていない）。
- **SSP 向けメタ `.txt`（`descript.txt` / `changelog.txt` / `readme.txt` / `install.txt` / `surfacetable.txt` / `system_config.txt` など）= Shift_JIS (cp932)**。
  これらを UTF-8 として読むと文字化けする。読み書きの際は cp932 を明示すること。`yaya.txt` と `surfaces.txt` は UTF-8（ASCII のみの `updates.txt` 等はどちらでも同じ）。

## 辞書の構成

- 読み込む辞書（`ghost/master/yaya.txt` の `dic` / `dicdir`）: 読み込み順に意味がある。
  1. `system_config.txt`（YAYA システム設定。**必ず最初**。`dicdir, system` でサブモジュール辞書を読む）
  2. `rs_word.dic`（単語）, `rs_string.dic`（文字列リソース）
  3. `rs_aitalk.dic`（ランダムトークの**ロジック**専用）
  4. `dicdir, aitalks`（`aitalks/` 配下の `.dic` を**再帰的に**全ロード＝トークデータ群）
  5. `rs_aitalk_old.dic`（旧トーク）, `rs_bootend.dic`（起動/終了/切替）, `rs_communicate.dic`（他ゴースト会話）
  6. マウス系（`rs_mouse*.dic`）, `rs_menu.dic`（メニュー）, `rs_etc.dic`, `rs_dic.dic`（用語解説）, `rs_property.dic` ほか
- 緊急モード（`yaya_emerg.txt`）:
- システム辞書: `ghost/master/system/`（YAYA システム辞書 [yaya-dic] の git submodule。`git submodule update --init --recursive` で取得。無いと起動しない）

## イベントと辞書ファイルの対応

| ファイル | 主な中身 |
|---|---|
| `rs_aitalk.dic` | `OnAiTalk` とランダムトークの合成ロジック |
| `aitalks/**/*.dic` | ランダムトークのデータ（下の表） |
| `rs_bootend.dic` | 起動・終了・切替 |
| `rs_communicate.dic` | 他ゴーストとの会話 |
| `rs_mouse*.dic` | マウス反応（竜姫シェル専用は `rs_mouse_df.dic`） |
| `rs_menu.dic` | メニュー |
| `rs_dic.dic` | 用語解説 |

新しいイベントに反応させるときに書く場所:

### ランダムトークの合成機構（このゴーストの心臓部）

トークは「**ロジック（`rs_aitalk.dic`）**」と「**データ（`aitalks/*.dic`）**」に分離されている。

- `OnAiTalk`（`rs_aitalk.dic`）が起点。`communicateratio` % で他ゴーストへ話しかけ、見切れ中なら `MikireTalk`、チェイン中なら `ChainTalk`、通常は `RandomTalk` を呼ぶ。
- `RandomTalkEx : nonoverlap` が候補配列 `_talk_array` を `,=` で動的合成して `parallel` で1本選ぶ:
  - 常時: `aryNormalTalk` `aryWorldTalk` `aryKariuTalk` `aryExtremeWorldTalk` `aryTopicTalk` `aryMicroTalk`
  - **シェル依存**: `shellname` が「竜姫 / Drachen Fürstin」なら `aryDrachenFurstinTalk`、それ以外は `aryExiceOnlyTalk`
  - **旧トーク**: `flgTalkOld == 1` のとき `aryNormalTalkOld` を追加
  - **季節**: `GetSeasonSlot` で 春/夏/秋/冬 → 対応する `arySpringTalk` 等
  - **時間帯**: `GetTimeSlot` が「深夜」なら `aryMidnightTalk`

### トークデータと配列名の対応（`aitalks/`）

| ファイル | 配列名 | 用途 |
|---|---|---|
| `aitalks/normal.dic` | `aryNormalTalk` | 常時トーク（最大） |
| `aitalks/world.dic` | `aryWorldTalk` | 世界設定トーク |
| `aitalks/kariu.dic` | `aryKariuTalk` | — |
| `aitalks/extreme_world.dic` | `aryExtremeWorldTalk` | — |
| `aitalks/topic.dic` | `aryTopicTalk` | — |
| `aitalks/shell/exice.dic` | `aryExiceOnlyTalk` | イクサイス シェル限定 |
| `aitalks/shell/drachen.dic` | `aryDrachenFurstinTalk` | 竜姫 シェル限定 |
| `aitalks/shell/micro.dic` | `aryMicroTalk` | — |
| `aitalks/season/{spring,summer,autumn,winter}.dic` | `arySeasonTalk` 各種 | 四季 |
| `aitalks/time/midnight.dic` | `aryMidnightTalk` | 深夜 |
| `aitalks/chain.dic` | （チェイントーク） | 連鎖会話 |
| `rs_aitalk_old.dic` | `aryNormalTalkOld` | 旧トーク（`flgTalkOld` で切替） |

### 旧トーク機能（`flgTalkOld`）

`rs_menu.dic` の TALKOLD メニューから `flgTalkOld` を切り替えると、`aryNormalTalkOld`（`rs_aitalk_old.dic`）が `RandomTalkEx` に合成される。既定は `flgTalkOld=0`（未初期化・未保存）。

## キャラクターとサーフェス

| スコープ | キャラクター | 人物像（一人称、口調、性格） | 使えるサーフェス |
|---|---|---|---|
| `\0` | オフィーリア | | |
| `\1` | エプシロン | | |

- シェル: `shell/` に `master`, `Drachen_Furstin`, `Fantastic_Voyage`。`.gitignore` で一部の派生シェル（`exice_w`, `custom`, `master+` 等）は除外。シェル名（`shellname`）でトーク・マウス反応が分岐する。
- バルーン: リポジトリ直下の `Roche Limit/` に同梱。
- 当たり判定（`surfaces.txt` の `collision`）:
- トークで使わないサーフェス（アニメーション用の部品など）:

## トークの書き方

（間の取り方、改行の入れ方、話し手の交代、ユーザーの呼び方など、このゴーストの決まり。既存のトークから短い例を 1 つ載せるとよい）

## 独自のルール

- **トーク分割の鉄則**: 同名の `: array` を複数ファイルに定義しても連結されない（YAYA は同名定義のうち1つだけを選択する）。カテゴリを分割・追加する際は**必ず固有の配列名**を付け、`rs_aitalk.dic` の `RandomTalkEx` に `_talk_array ,= aryXxxTalk` を1行追加して組み込むこと。配列名の付け忘れ・合成漏れはトークが一切出ない無言バグになる。
- **賞味期限切れトーク**: 旬を過ぎたトークは `aitalks/normal.dic` から削除し、`rs_aitalk_old.dic` へ移送する（内容は無改変で移す）。
- **動作確認**: SSP にゴーストを読ませ、変更後はゴースト右クリック →「再読み込み」（または `F5`）で辞書を再コンパイルして反映する。`OnAiTalk` を繰り返し発火させて該当トークの出現を目視確認する。開発キットの `tools/check.ps1`・`tools/sstp.ps1` も使える（`docs/agents/commands.md`）。
- **エラーログ**: 既定では実行ログは無効。構文エラーや実行時エラーを追う際は `ghost/master/yaya.txt`（および `system_config.txt`）の `// log, ayame.log` のコメントを外すと `ayame.log` に記録される。
- **仕様調査**: `ukagaka-doc` MCP の検索は**単一キーワード**が有効（句で検索すると not_found になりやすい）。
