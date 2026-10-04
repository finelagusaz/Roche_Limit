# GHOST.md

このゴーストに固有の情報です。AI エージェントは、作業の前に `AGENTS.md` とあわせて読みます。
作者が自由に書き換えるファイルで、開発キットの更新（`tools/update-devkit.ps1`）で上書きされることはありません。

## このゴーストについて

- 名前（`ghost/master/descript.txt` の `name`）: Roche_Limit（ロッシュの限界）
- キャラクター（`sakura.name` / `kero.name`）: `\0` = オフィーリア / `\1` = エプシロン
- 作者: Fine Lagusaz
- 配布先・ネットワーク更新の URL: 配布先 https://blankrune.sakura.ne.jp/ 。ネットワーク更新は `rs_string.dic` の `On_homeurl` が返す https://blankrune.sakura.ne.jp/named/roche_limit_plus/update/ （ゴーストの `descript.txt` には `homeurl` を書いていない）
- 元にしたテンプレートやゴースト: `changelog.txt` によると、2004/7 の製作開始時に綾河くもさんの「少女Y」をベースにし、SHIORI は YuhnaSE を使った。のちに aya5.dll を使い、2006/11/01（Ver.0.425）に YAYA へ差し替えた。以後もシステム辞書の更新・差し替えを重ねている（直近の大きなものは 2023/03/07 Ver.1.025「システム辞書差し替え・いろいろ刷新」）。作者の記憶では、konnoayame を経て konnoyayame をもとにシステム辞書を直したが、更新履歴には書かれていない（未確認）。`ghost/master/updates.txt` に残る `aya5.dll` などは AYA5 時代の跡
- 設定資料: リポジトリ直下の `memo.txt`（cp932）。キャラクターの性格・経歴・周辺人物・用語の作者メモ。人物像を書くときはここを参照する

## ライセンス

- 辞書: 自由に利用してよい（作者いわく「煮るなり焼くなり好きにして」。改変・再配布も可）。ただし `ghost/master/system/`（yaya-dic）と `yaya.dll` はそれぞれの元のライセンスに従う
- 設定（世界観・キャラクター）: `readme.txt` の「FSW宣言」で、フリーシェアワールドとして全設定を開放している
- シェル: 改変は可。改変したものの再配布は、それぞれの元のシェルの条件に従う。
  - `master`: オフィーリアは くーげるしゅれいばー さんのフリーシェル「量産型イクサイス」、エプシロン/アズリエルは 一哉那順 さんのフリーシェル「なつめがね」（`shell/master/readme.txt`）
  - `Drachen_Furstin`（竜姫）: Kamizaiduki and Fine
  - `Fantastic_Voyage`（ミクロの決死圏）: Wiz★ さん作。readme で転載・再配布・改変とも**不可**と明記

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
- 緊急モード（`yaya_emerg.txt`）: なし
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
| `\0` | オフィーリア（本名アルギズ。イクサイスユニット装着時の呼び名） | 一人称「私」。です・ます調。相方を `%keroname`、ユーザーを `%username` と呼ぶ。戦闘用アンドロイド。基本は温厚だが割り切った冷たさが同居する。他者を守ることに存在理由を見出す。人の話は聞いてから返し、考えを押しつけない。割とルーズ。本と猫に目が無い | 0 素 / 1 赤面 / 2 驚き / 3 困惑 / 4 溜息 / 5 笑顔 / 6 目閉じ / 7 怒り / 8 冷笑 / 9 苦笑 |
| `\1` | エプシロン（中身はアズリエル・ムーンリット。ラボからリモートで操作するサポートユニット＝アバター） | 一人称「僕」、相手を「君」と呼ぶ。「〜だよ」「〜さ」のくだけた口調。理論重視で、感情と理屈を分ける。ルールや規約を重視し、差別には反発する。批判的に物事を見がちだが周りからは達観して見える。真顔でボケる | 人型のとき 10〜29（surfacetable では「アズ」系。10 が素）。AFK や簡易的な操作のとき 40 通常 / 41 驚愕 / 42 混乱 |

- アズとエプシロンはキャラクターとしては同一人物。サーフェスの「アズ」「エプシロン」の違いは、アバターの状態（人型か、AFK・簡易操作か）の違い。
- 竜姫シェル（`Drachen_Furstin`）ではイクサイスではないので、互いに本名（アルギズ / アズリエル）で呼び合い、イクサイス装着時のトークはしない。
- 二人は互いを大切な友人と思っている（表に出さない設定は `memo.txt`）。周辺人物（マリ、八城騏一郎、ラボの開発チームなど）も `memo.txt` にある。

- シェル: `shell/` に `master`, `Drachen_Furstin`, `Fantastic_Voyage`。`.gitignore` で一部の派生シェル（`exice_w`, `custom`, `master+` 等）は除外。シェル名（`shellname`）でトーク・マウス反応が分岐する。
- バルーン: リポジトリ直下の `Roche Limit/` に同梱。
- 当たり判定（`surfaces.txt` の `collision`、`master` の `\0`）: Head, Face, Bust, Skirt, Leg, Lip, Hand, thigh, Ribbon
- トークで使わないサーフェス（アニメーション用の部品など）: 100・101（`\0` の目パーツ）、201〜207・300・301・2001・3001〜3005（`\1` のパーツ）。`\s[-1]` は姿を消すときに使う。
  - 辞書に `\s[1001]` が 2 か所あるが、surfacetable に 1001 は無い（未確認）。

## トークの書き方

- 読点のあとに `\w4`、文末に `\w9`。
- 話し手が替わる前に `\n\n[half]` を入れる。
- 長い台詞は行末の `/` で次の行へ続ける。
- チェイントークの起点は `\e:chain=名前`。
- 相方は `%keroname`、ユーザーは `%username` で呼ぶ。

例（`aitalks/normal.dic`）:

```
'\0\s[0]そういえば標準装備の武器が無いんですよね。\w9\n\n[half]/
\1\s[10]あ、\w4そっか。\w9\n普段使っているのは\w4昔から君が持っているものなんだよね。\w9\n\n[half]/
\0\s[6]ユニットで強化された索敵能力を活かさないのもあれですし\w4…。\w9\n\n[half]/
\1\s[10]何処かと戦争でもするつもりかい?\w4\n\n[half]/
\0\s[5]そんなことはしませんよ。\w9\n備えあれば憂いなしってことわざがあるように備えたいだけです。\w9\e'
```

## 独自のルール

- **トーク分割の鉄則**: 同名の `: array` を複数ファイルに定義しても連結されない（YAYA は同名定義のうち1つだけを選択する）。カテゴリを分割・追加する際は**必ず固有の配列名**を付け、`rs_aitalk.dic` の `RandomTalkEx` に `_talk_array ,= aryXxxTalk` を1行追加して組み込むこと。配列名の付け忘れ・合成漏れはトークが一切出ない無言バグになる。
- **賞味期限切れトーク**: 旬を過ぎたトークは `aitalks/normal.dic` から削除し、`rs_aitalk_old.dic` へ移送する（内容は無改変で移す）。
- **動作確認**: SSP にゴーストを読ませ、変更後はゴースト右クリック →「再読み込み」（または `F5`）で辞書を再コンパイルして反映する。`OnAiTalk` を繰り返し発火させて該当トークの出現を目視確認する。開発キットの `tools/check.ps1`・`tools/sstp.ps1` も使える（`docs/agents/commands.md`）。
- **エラーログ**: 既定では実行ログは無効。構文エラーや実行時エラーを追う際は `ghost/master/yaya.txt`（および `system_config.txt`）の `// log, ayame.log` のコメントを外すと `ayame.log` に記録される。
- **仕様調査**: `ukagaka-doc` MCP の検索は**単一キーワード**が有効（句で検索すると not_found になりやすい）。
