# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

**Roche_Limit（ロッシュの限界）** は YAYA SHIORI で動作する伺か（Ukagaka）ゴースト。
キャラクターは本体 `\0`＝**オフィーリア**（sakura.name）、相方 `\1`＝**エプシロン**（kero.name）。
ビルドシステムは存在せず、成果物は SSP が読み込む辞書（`.dic`）・設定・シェル画像の集合体。
作者: Fine Lagusaz。

## 文字コードと改行（最重要・編集前に必読）

ファイル種別ごとに**文字コードが異なる**。誤ると即座に文字化け・破損する。実測で確認済みの規約:

- **`.dic`（YAYA 辞書）= UTF-8 / BOM無し / CRLF**。`yaya.txt` で `charset.dic, UTF-8` 指定。
  改行は LF ではなく CRLF（`grep -c $'\r' file` が `wc -l` と一致すれば CRLF）。
  スクリプトで `.dic` を生成・整形する場合、Python なら `io.open(..., newline="")` でバイト保存し CRLF を維持すること（LF で書き出すと原状を壊す）。`git config core.autocrlf` は `input` だが blob 自体が CRLF。
- **SSP 向けメタ `.txt`（`descript.txt` / `changelog.txt` / `readme.txt` / `install.txt` / `updates.txt`）= Shift_JIS (cp932)**。
  これらを UTF-8 として読むと文字化けする。読み書きの際は cp932 を明示すること。

## 開発ワークフロー

ビルド・lint・自動テストの仕組みは無い。YAYA はゴースト読み込み時に `.dic` をその場でコンパイルする。

- **初回セットアップ**: `git submodule update --init --recursive`
  （`ghost/master/system` は YAYA システム辞書 [yaya-dic] のサブモジュール。これが無いと起動しない）
- **動作確認 = SSP で再読み込み**: SSP にゴーストを読ませ、変更後はゴースト右クリック →「再読み込み」（または `F5`）で辞書を再コンパイルして反映する。「単体テスト」という概念は無く、`OnAiTalk` を繰り返し発火させて該当トークの出現を目視確認するのが基本。
- **エラーログ**: 既定では実行ログは無効。構文エラーや実行時エラーを追う際は `ghost/master/yaya.txt`（および `system_config.txt`）の `// log, ayame.log` のコメントを外すと `ayame.log` に記録される。開発中は有効化推奨。
- **YAYA / さくらスクリプト / SHIORI イベントの仕様調査**: `.mcp.json` の `ukagaka-doc` MCP（`mcp__ukagaka-doc__search_docs` / `get_doc`）で引ける。検索は**単一キーワード**が有効（句で検索すると not_found になりやすい）。

## アーキテクチャ

### 辞書のロード順（`ghost/master/yaya.txt`）

`yaya.txt` が全辞書のエントリポイント。読み込み順に意味がある:

1. `system_config.txt`（YAYA システム設定。**必ず最初**。`dicdir, system` でサブモジュール辞書を読む）
2. `rs_word.dic`（単語）, `rs_string.dic`（文字列リソース）
3. `rs_aitalk.dic`（ランダムトークの**ロジック**専用）
4. `dicdir, aitalks`（`aitalks/` 配下の `.dic` を**再帰的に**全ロード＝トークデータ群）
5. `rs_aitalk_old.dic`（旧トーク）, `rs_bootend.dic`（起動/終了/切替）, `rs_communicate.dic`（他ゴースト会話）
6. マウス系（`rs_mouse*.dic`）, `rs_menu.dic`（メニュー）, `rs_etc.dic`, `rs_dic.dic`（用語解説）, `rs_property.dic` ほか

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

各カテゴリは**固有名の `aryXxxTalk : array`** として定義され、上記 `RandomTalkEx` で合成される:

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

### トーク分割の鉄則

**同名の `: array` を複数ファイルに定義しても連結されない**（YAYA は同名定義のうち1つだけを選択する）。
カテゴリを分割・追加する際は**必ず固有の配列名**を付け、`rs_aitalk.dic` の `RandomTalkEx` に `_talk_array ,= aryXxxTalk` を1行追加して組み込むこと。配列名の付け忘れ・合成漏れはトークが一切出ない無言バグになる。

### 旧トーク機能（`flgTalkOld`）

`rs_menu.dic` の TALKOLD メニューから `flgTalkOld` を切り替えると、`aryNormalTalkOld`（`rs_aitalk_old.dic`）が `RandomTalkEx` に合成される。既定は `flgTalkOld=0`（未初期化・未保存）。
旬を過ぎた「賞味期限切れトーク」は `aitalks/normal.dic` から削除し、この `rs_aitalk_old.dic` へ移送する運用（内容は無改変で移す）。

## シェルとバルーン

- `shell/`（リポジトリ直下）: `master`, `Drachen_Furstin`, `Fantastic_Voyage` の各シェル。`.gitignore` で一部の派生シェル（`exice_w`, `custom`, `master+` 等）は除外。
- `Roche Limit/`（リポジトリ直下）: 同梱バルーン。
- シェル名（`shellname`）でトーク・マウス反応が分岐する（例: 竜姫シェル専用の `rs_mouse_df.dic`）。
