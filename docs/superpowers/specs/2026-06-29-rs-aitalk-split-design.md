# rs_aitalk.dic カテゴリ分割 設計書

- 日付: 2026-06-29
- 対象: `ghost/master/rs_aitalk.dic`（ランダムトーク辞書、約13,773行 / 1.1MB）
- 目的: 巨大化した単一辞書を「ロジック」と「カテゴリ別データ」に分割し、編集性・見通しを改善する。

## 1. 概要

`rs_aitalk.dic` は OnAiTalk まわりのロジックと、約11,500行の巨大なトーク配列 `aryNormalTalk` を含む単一ファイルである。本設計では次の方針で複数ファイルへ分割する。

- **ロジックとデータを完全分離**する。`rs_aitalk.dic` はイベントハンドラ・選択関数・見切れ等の「ロジック専用」にし、トーク本体（配列・チェイン）は `ghost/master/aitalks/` フォルダ下のカテゴリ別ファイルへ移す。
- 分割の大分類は **「出現条件」**（常時／条件付き／構造）を軸とし、中・小分類を「内容ジャンル」とする。これは `RandomTalkEx` の `,=` 合成ロジックと一対一で対応するため、分類と実装が破綻しにくい。
- **既存マーカー（`//`・`//----`）の範囲のみ**を分割単位とする。約11,500行の未分類な本体ブロックは中身を再分類せず、1ファイル（`aryNormalTalk`）にまとめて残す。
- **挙動（トークの出現頻度・条件）は完全保存**する。配列名は外部参照のあるものを厳密に維持し、新設配列は無条件合成して頻度を変えない。

## 2. 現状の構造（調査結果）

| 区分 | 内容 | おおよその行範囲 |
|---|---|---|
| ロジック | OnTranslate / OnAiTalk / RandomTalk / RandomTalkEx / NormalTalk | 1–118 |
| データ（本体配列）| `aryNormalTalk : array`（下記サブ節を内包） | 120–13031 |
| └ 一般（未分類本体）| `//ランダムトーク` のみ。マーカー無し | 122–11694 |
| └ 世界観 | `//---- 世界観 ----` | 11695–12483 |
| └ FS用イクサイス（注記）| `// FS用イクサイスユニット…` | 12484–12512 |
| └ 質問箱 | `//質問箱` | 12513–12527 |
| └ 死生観 | `//死生観` | 12528–12544 |
| └ 空調絡みの話 | `//空調絡みの話` | 12545–12557 |
| └ 日々は誰かとともに | `//日々は誰かとともに` | 12558–12612 |
| └ カリウ関連 | `//カリウ関連` | 12613–12756 |
| └ アリウム/Extreme World | `//アリウム、Extreme Worldに関連するトーク` | 12757–12935 |
| └ ミクロの決死圏用 | `// ミクロの決死圏用` | 12936–13031 |
| ロジック＋データ | `ExiceOnlyTalk`（13036）/ `aryExiceOnlyTalk`（イクサイス専用）| 13033–13122 |
| ロジック＋データ | `DrachenFurstinTalk`（13127）/ `aryDrachenFurstinTalk`（竜姫専用）| 13124–13157 |
| ロジック＋データ | 季節 春/夏/秋/冬 各ラッパー＋配列 | 13159–13494 |
| ロジック＋データ | `MidnightTalk`（13496）/ `aryMidnightTalk`（深夜）| 13495–13522 |
| データ（構造）| トークチェイン12種 | 13524–13694 |
| ロジック | OnMinuteChange（時報/重なり）| 13695–13732 |
| ロジック | OnSecondChange / Mikire系 / `mikirego` チェイン | 13733–13765 |
| ロジック | OnSurfaceRestore | 13767–末尾 |

注: 行範囲は調査時点の目安。`aryNormalTalk` の終端付近（13030–13031 の二重 `}`）はミクロ節内に入れ子ブロックがある可能性があり、実装時に正確な括弧対応を確認する。

### 外部参照（重要）

- `rs_communicate.dic`（165–174行, `about:talk`）が `ARRAYSIZE(aryNormalTalk)`, `arySpringTalk`, `arySummerTalk`, `aryAutumnTalk`, `aryWinterTalk`, `aryMidnightTalk`, `aryNormalTalkOld` を参照（統計表示）。
- `RandomTalkEx`（66–110行）が `aryNormalTalk` / `aryExiceOnlyTalk`（非・竜姫時）/ `aryNomarlTalkOld`（旧, ※綴り `Nomarl`）/ 季節4種 / `aryMidnightTalk` を `,=` 合成。
- `aryDrachenFurstinTalk` と各ラッパー（`NormalTalk` / `DrachenFurstinTalk` / `SpringTalk` 等 / `MidnightTalk`）は **どこからも呼ばれていない**（竜姫専用トークは現状ローテーション未接続）。本設計では `RandomTalkEx` に接続し、竜姫シェル時に出現させる（5.3 参照）。
- シェル名の確認: `shell/Drachen_Furstin/descript.txt` の `name,竜姫 / Drachen Fürstin` と、`RandomTalkEx` 内の判定文字列 `'竜姫 / Drachen Fürstin'`（ü 含む）は完全一致。`shellname` による分岐はこの文字列で正しく働く。

## 3. 目標 / 非目標

**目標**
- ロジックとカテゴリ別データを別ファイルに分離する。
- 分類体系（大/中/小）を整理し直す。
- トークの出現挙動を完全に保つ（分割対象の常時/条件付きトーク）。
- **竜姫専用トーク（`aryDrachenFurstinTalk`）を `RandomTalkEx` に接続し、竜姫シェル時に出現させる。**
- 外部参照（統計・条件分岐）を壊さない。

**非目標（スコープ外）**
- 約11,500行の本体ブロックの意味的再分類。
- 未使用ラッパー関数（`NormalTalk` / `DrachenFurstinTalk` 等）の整理（別途検討）。
- トーク内容そのものの追加・修正・校正。
- 旧トーク辞書 `rs_aitalk_old.dic` の変更。

## 4. 分類体系とファイル構成

大分類（出現条件）を**フォルダ階層**で表現する。常時トーク（A）と構造トーク（C）は `aitalks/` 直下、条件付きトーク（B）は `aitalks/{shell,season,time}/` のサブフォルダへ。ファイル名は `aitalks/` 配下では `rs_aitalk_` プレフィックスを外す（フォルダで文脈が付くため）。

```
【ロジック】データを持たない（master 直下に据え置き）
  rs_aitalk.dic … OnTranslate / OnAiTalk / RandomTalk / RandomTalkEx /
                  各ラッパー関数 / OnMinuteChange / 見切れ(OnSecondChange/Mikire系/mikirego) /
                  OnSurfaceRestore

【データ】ghost/master/aitalks/ 配下に配置
aitalks/
  ├ normal.dic       … aryNormalTalk   〔A:一般・本体 約11,500行／中身は分類しない〕
  ├ world.dic        … aryWorldTalk    〔A:世界観＋FS用イクサイスの注記〕
  ├ kariu.dic        … aryKariuTalk    〔A:舞台・固有名詞／カリウ関連〕
  ├ arium.dic        … aryAriumTalk    〔A:アリウム/Extreme World〕
  ├ micro.dic        … aryMicroTalk    〔A:ミクロの決死圏〕
  ├ topic.dic        … aryTopicTalk    〔A:小テーマ＝質問箱・死生観・空調・日々は誰かとともに〕
  ├ chain.dic        … チェイン12種     〔C:構造／:chain=名前 で呼ばれるため合成不要〕
  │                      (ogre / school_a_swimming_suit / star / アンドロイド / フィルタリング /
  │                       辞書攻撃 / 八城その1 / 法と人 / お酒の変化 / 勝算はあるのか /
  │                       ことの顛末 / フォローにまわってみた)
  │                      ※ mikirego は見切れロジックに密結合のため rs_aitalk.dic 側に残す。
  ├ shell/           〔B:シェル限定〕
  │   ├ exice.dic    … aryExiceOnlyTalk      （非・竜姫シェル時のみ）
  │   └ drachen.dic  … aryDrachenFurstinTalk （竜姫シェル時のみ・本設計で新規接続）
  ├ season/          〔B:季節限定〕
  │   ├ spring.dic   … arySpringTalk
  │   ├ summer.dic   … arySummerTalk
  │   ├ autumn.dic   … aryAutumnTalk
  │   └ winter.dic   … aryWinterTalk
  └ time/            〔B:時間帯限定〕
      └ midnight.dic … aryMidnightTalk（深夜）
```

大分類A（常時）= `RandomTalkEx` へ無条件 `,=` 合成。大分類B（条件付き）= 既存の条件分岐を維持（シェルは本設計で竜姫分岐を追加）。大分類C（構造）= 合成不要。

### 配列名の方針

- **据え置き（外部参照あり）**: `aryNormalTalk`, `aryExiceOnlyTalk`, `aryDrachenFurstinTalk`, `arySpringTalk`, `arySummerTalk`, `aryAutumnTalk`, `aryWinterTalk`, `aryMidnightTalk`。
- **新設（5つ）**: `aryWorldTalk`, `aryKariuTalk`, `aryAriumTalk`, `aryMicroTalk`, `aryTopicTalk`。

## 5. 実装方針

### 5.1 ファイルの作成と移動
- `ghost/master/aitalks/` と、その下に `shell/` `season/` `time/` サブフォルダを作成する。
- 各カテゴリのトーク文字列を、対応する `aitalks/...dic` に「配列定義」として切り出す。
- 据え置き名の配列（イクサイス/竜姫/季節/深夜）は配列定義のみをデータファイルへ移し、対応ラッパー関数（`ExiceOnlyTalk` 等）は `rs_aitalk.dic`（ロジック）に残す。
- 文字コードは既存に合わせ **UTF-8**（`yaya.txt` の `charset.dic, UTF-8`）。BOM無しを維持する。

### 5.2 YAYA配列マージの仕組み（設計前提）
YAYA では同名 `: array` 関数を複数定義しても**連結されず**、呼び出し時に1定義のみが選択される。よって一つの配列を複数ファイルへ「同名で」分割することはできない。本ゴーストの既存実装も、カテゴリごとに**固有名の配列**を用意し `RandomTalkEx` で `,=` 合成する方式を採っている。本設計はこの既存イディオムに従う。
（検証: 分割後に `about:talk` の各カウント・合計が分割前の値と一致することで確認する。）

### 5.3 `RandomTalkEx`（rs_aitalk.dic）の編集
大分類A の新設配列を**無条件**で合成する行を追加する（本体ブロックと同じ常時ローテーション挙動を保つため）。

```
_talk_array ,= aryNormalTalk      // 既存
_talk_array ,= aryWorldTalk       // 追加
_talk_array ,= aryKariuTalk       // 追加
_talk_array ,= aryAriumTalk       // 追加
_talk_array ,= aryMicroTalk       // 追加
_talk_array ,= aryTopicTalk       // 追加
```
条件付き（旧/季節/深夜）の既存合成行は変更しない。

**シェル限定の接続（竜姫専用の有効化）**: 既存のイクサイス分岐に `else` を足し、竜姫シェル時に `aryDrachenFurstinTalk` を合成する。これによりイクサイス／竜姫が対称になる（共通の常時トーク=大分類A は両シェルとも受け取る）。

```
if shellname != '竜姫 / Drachen Fürstin' {
    _talk_array ,= aryExiceOnlyTalk      // 既存（非・竜姫）
}
else {
    _talk_array ,= aryDrachenFurstinTalk // 追加（竜姫シェル時）
}
```

### 5.4 `yaya.txt` の編集
`dic, rs_aitalk.dic` の行はロジックとして残し、その直後に新データファイル群の `dic,` 行を追加する。

```
dic, rs_aitalk.dic              // ランダムトーク（ロジック）
dic, aitalks/normal.dic         // 通常トーク本体
dic, aitalks/world.dic          // 世界観
dic, aitalks/kariu.dic          // カリウ関連
dic, aitalks/arium.dic          // アリウム/Extreme World
dic, aitalks/micro.dic          // ミクロの決死圏
dic, aitalks/topic.dic          // 小テーマ集約
dic, aitalks/chain.dic          // トークチェイン
dic, aitalks/shell/exice.dic    // イクサイス専用
dic, aitalks/shell/drachen.dic  // 竜姫専用
dic, aitalks/season/spring.dic  // 季節：春
dic, aitalks/season/summer.dic  // 季節：夏
dic, aitalks/season/autumn.dic  // 季節：秋
dic, aitalks/season/winter.dic  // 季節：冬
dic, aitalks/time/midnight.dic  // 深夜
```
パス区切りは `/`（Windows でも可・移植性が高い）。YAYA は配列/関数をグローバルに解決するため読み込み順は問わないが、可読性のため上記順とする。

### 5.5 `rs_communicate.dic`（about:talk 統計）の更新
`aryNormalTalk` から題材を抜き出すため `ARRAYSIZE(aryNormalTalk)` が縮む。統計を正しく保つよう、新設配列を表示行と合計式に加える。あわせて**既存バグ**（合計式での `arySpringTalk` の二重加算、および `aryMidnightTalk` の合計漏れ）を修正する。

- 表示に追加: 世界観 / カリウ / アリウム / ミクロ / 小テーマ の各 `ARRAYSIZE`。
- 合計式: `aryNormalTalk + aryWorldTalk + aryKariuTalk + aryAriumTalk + aryMicroTalk + aryTopicTalk + arySpringTalk + arySummerTalk + aryAutumnTalk + aryWinterTalk + aryMidnightTalk + aryNormalTalkOld`（重複・漏れを解消）。

## 6. 挙動の保存と検証

- **頻度保存**: 大分類A は全て無条件合成のため、本体に内包されていた頃と出現確率は不変。条件付き（B）は既存条件をそのまま移送。
- **件数検証（機械的）**: 分割前の `aryNormalTalk` の要素数 = 分割後の `aryNormalTalk + aryWorldTalk + aryKariuTalk + aryAriumTalk + aryMicroTalk + aryTopicTalk` の合計、を突合する。
- **ロード検証（実行時）**: SSP/YAYA でゴーストを起動し、エラーログ無し・`about:talk` の合計が分割前と一致することを確認（ユーザ環境での起動確認）。
- **チェイン検証**: `:chain=` で参照される12チェインが `aitalks/chain.dic` から解決されることを、代表トークのチェイン発火で確認。
- **竜姫接続検証**: 竜姫シェル（`竜姫 / Drachen Fürstin`）に切替時、`aryDrachenFurstinTalk` のトークが出現すること、かつ `aryExiceOnlyTalk` が出ないことを確認。master 等の非竜姫シェルでは従来どおりイクサイス専用が出て竜姫専用が出ないことを確認。

## 7. リスク・既知の論点

- **配列終端の二重括弧**（13030–13031）: ミクロ節に入れ子ブロックがある可能性。切り出し時に括弧対応を正確に確認する。
- **文字コード/改行/BOM**: UTF-8（BOM無し）・既存改行コードを維持。崩れると文字化け・ロード失敗の恐れ。
- **同名配列の非連結**（5.2）: 設計の根幹前提。既存イディオムに従い、検証で担保する。
- **竜姫専用の接続**: 本設計で `RandomTalkEx` に `else` 分岐を追加して有効化する。判定文字列はシェル名（`竜姫 / Drachen Fürstin`、ü 含む）と完全一致を確認済み。タイプミスや全角/半角・濁点違いがあると無効化されるため、既存文字列を流用する。
- **大容量ファイルの分割操作**: 1.1MB を扱うため、手作業ではなくスクリプト等で機械的に切り出し、差分の取り違えを防ぐ。
- **サブフォルダ指定の動作**: `dic, aitalks/season/spring.dic` のようなサブパス指定が YAYA で正しく解決されることを、実装初期にダミー1ファイルで確認してから本格展開する（解決不可なら平置きへフォールバック）。

## 8. 成果物

- 新規フォルダ: `ghost/master/aitalks/`（＋ `shell/` `season/` `time/` サブフォルダ）。
- 新規データファイル（14ファイル、すべて `aitalks/` 配下）:
  - 直下: `normal.dic` / `world.dic` / `kariu.dic` / `arium.dic` / `micro.dic` / `topic.dic` / `chain.dic`
  - `shell/`: `exice.dic` / `drachen.dic`
  - `season/`: `spring.dic` / `summer.dic` / `autumn.dic` / `winter.dic`
  - `time/`: `midnight.dic`
- 変更ファイル: `rs_aitalk.dic`（ロジックのみに縮小＋RandomTalkEx の大分類A合成行追加＋竜姫シェル分岐の接続）, `yaya.txt`（dic行追加）, `rs_communicate.dic`（統計更新＋バグ修正）。
