# rs_aitalk.dic カテゴリ分割 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 巨大な `ghost/master/rs_aitalk.dic` をロジック専用ファイルとカテゴリ別データファイル（`aitalks/` 配下）に無損失で分割し、竜姫専用トークを有効化する。

**Architecture:** トーク文字列の配列を「固有名の配列＋`RandomTalkEx` での `,=` 合成」という既存イディオムに沿って別ファイルへ切り出す。抽出は行範囲の逐語コピー（バイト保存）で行い、「抽出断片を連結すると元に一致する」ことで無損失を機械検証する。各コミットでゴーストが読込可能な状態を保つ（データファイルは yaya.txt に載るまで不活性）。

**Tech Stack:** YAYA SHIORI 辞書（`.dic`）、`yaya.txt`（辞書ロード設定）、Python 3（抽出・検証スクリプト）、git。YAYA文法の確認には `mcp__ukagaka-doc__search_docs` / `mcp__ukagaka-doc__get_doc`（要 Claude Code 再起動＋承認）。

## Global Constraints

- 文字コード **UTF-8 / BOM無し / LF改行** を厳守（元ファイルがこの形式。`git config core.autocrlf=input`、`.gitattributes` 無し）。
- 抽出は **逐語（バイト保存）**。配列内の条件節 `if`/`elseif` はブロックごと移動し、途中行で切らない。
- 配列名は外部参照のため厳密維持: `aryNormalTalk` `aryExiceOnlyTalk` `aryDrachenFurstinTalk` `arySpringTalk` `arySummerTalk` `aryAutumnTalk` `aryWinterTalk` `aryMidnightTalk`。
- 新設配列（固有名）: `aryWorldTalk` `aryKariuTalk` `aryAriumTalk` `aryTopicTalk`（常時A）、`aryMicroTalk`（ミクロ限定B・配列内部で自己ゲート）。
- 竜姫シェル判定文字列は既存と完全一致で流用: `'竜姫 / Drachen Fürstin'`（ü 含む）。ミクロ判定は `shellname == "ミクロの決死圏"`。
- 各タスク完了時、ゴーストは読込可能（重複定義／未定義参照を残さない）。
- 作業ブランチ: `feature/rs-aitalk-split`（既存）。
- 検証は (1) 無損失バイト一致 (2) ブレース均衡 (3) 参照解決（静的）。実行時ロード確認は SSP でユーザが行う（CHECKPOINT）。
- 設計の根拠: `docs/superpowers/specs/2026-06-29-rs-aitalk-split-design.md`。

---

### Task 1: 境界マップの確定とバックアップ参照

元ファイルの各カテゴリ境界（行番号）と各配列定義の開始/終了を確定し、後続スクリプトの定数にする。元ファイルを参照用にコピーしておく（無損失検証の golden source）。

**Files:**
- Read: `ghost/master/rs_aitalk.dic`
- Create（作業外）: `<scratchpad>/rs_aitalk.orig.dic`（参照コピー）

**Interfaces:**
- Produces: 確定した行番号定数（後続タスクの抽出スクリプトが consume）。以下の「期待値」を実行時に確認・更新する。

- [ ] **Step 1: 元ファイルを参照用にコピー**

```bash
cp ghost/master/rs_aitalk.dic "$TMP/rs_aitalk.orig.dic"
wc -l ghost/master/rs_aitalk.dic   # 期待: 13773
```
（`$TMP` はスクラッチパッドの絶対パスに置換）

- [ ] **Step 2: カテゴリ境界アンカーの行番号を出力**

```bash
grep -nE '^\s*aryNormalTalk : array|^//---- 世界観|^\s*//質問箱|^\s*//カリウ関連|^\s*//アリウム|ミクロの決死圏用|^\s*if SurfaceMode != .5. && shellname == "ミクロの決死圏"|^\s*elseif SurfaceMode == .5. && shellname == "ミクロの決死圏"' ghost/master/rs_aitalk.dic
```
期待（目安・実値で更新する）:
```
120: aryNormalTalk : array
11695: //---- 世界観 ----
12513: //質問箱
12613: //カリウ関連
12757: //アリウム、Extreme Worldに関連するトーク
12936: // ミクロの決死圏用
12937: if SurfaceMode != '5' && shellname == "ミクロの決死圏"
13017: elseif SurfaceMode == '5' && shellname == "ミクロの決死圏"
```

- [ ] **Step 3: 各標準配列定義と章末の行番号を出力**

```bash
grep -nE '^(aryExiceOnlyTalk|aryDrachenFurstinTalk|arySpringTalk|arySummerTalk|aryAutumnTalk|aryWinterTalk|aryMidnightTalk) : array|^\s*// イクサイス専用トーク|^//---- トークチェイン|^//\*\*\*\* 時報' ghost/master/rs_aitalk.dic
```
期待（目安・実値で更新する）:
```
13040: aryExiceOnlyTalk : array {
13131: aryDrachenFurstinTalk : array {
13167: arySpringTalk : array
13224: arySummerTalk : array
13353: aryAutumnTalk : array
13401: aryWinterTalk : array
13500: aryMidnightTalk : array {
13524: //---- トークチェイン ----
13695: //**** 時報/重なり ****  (チェイン章の直後＝チェイン終端の目印)
```

- [ ] **Step 4: aryNormalTalk 配列の閉じ括弧（2連の `}`）を確認**

```bash
sed -n '13029,13033p' ghost/master/rs_aitalk.dic | cat -A
```
期待: 13030 と 13031 が単独の `}`（13030=ミクロ elseif ブロック閉じ、13031=aryNormalTalk 閉じ）。13032 空行、13033 が `// ----------------------`。

- [ ] **Step 5: 確認結果を控える（コミット不要）**

確定した行番号を Task 3 のスクリプト定数へ反映する。リポジトリ変更は無し。

---

### Task 2: aitalks フォルダ作成とサブフォルダ dic ロードのスモークテスト

サブフォルダ指定の `dic` がYAYAで解決されるかを、本格展開の前にダミー1ファイルで確認する（設計の最大リスク項目）。

**Files:**
- Create: `ghost/master/aitalks/`, `ghost/master/aitalks/shell/`, `ghost/master/aitalks/season/`, `ghost/master/aitalks/time/`
- Create（一時）: `ghost/master/aitalks/_smoke.dic`
- Modify: `ghost/master/yaya.txt`（一時的に1行追加）

- [ ] **Step 1: フォルダ作成**

```bash
mkdir -p ghost/master/aitalks/shell ghost/master/aitalks/season ghost/master/aitalks/time
```

- [ ] **Step 2: ダミー辞書を作成（UTF-8/LF/BOM無し）**

`ghost/master/aitalks/_smoke.dic`:
```
SubfolderLoadSmoke
{
	"OK"
}
```

- [ ] **Step 3: yaya.txt に一時ロード行を追加**

`dic, rs_aitalk.dic` 行の直後に追加:
```
dic, aitalks/_smoke.dic
```

- [ ] **Step 4: CHECKPOINT（ユーザ実行時確認）**

ユーザに依頼: SSP でゴーストを再起動し、YAYA のエラーログにロード失敗が無いことを確認。可能なら `\![eval,SubfolderLoadSmoke]` 等で `OK` が返るか確認。
Expected: エラー無くロードされる。→ サブフォルダ構成で続行。
失敗時: パス区切りを `\` に変えて再確認。なお解決不可なら設計を平置き（`aitalks/` を作らず `rs_aitalk_<cat>.dic`）へフォールバック（spec 5.4 / リスク欄参照）し、本計画の以降のパスを読み替える。

- [ ] **Step 5: ダミーを撤去して整地**

```bash
rm ghost/master/aitalks/_smoke.dic
```
`yaya.txt` の `dic, aitalks/_smoke.dic` 行を削除。空フォルダは Task 3 まで残す（git は空フォルダを追跡しないため、この時点ではコミットしない）。

---

### Task 3: 抽出スクリプトでデータ14ファイルを生成（不活性）

決定論的なスクリプトで、元ファイルを 14 のデータファイルへ逐語分割する。この時点では `yaya.txt` に未登録＝ロードされないため、ゴーストは不変（既存 `rs_aitalk.dic` がそのまま動く）。生成物は無損失検証してからコミットする。

**Files:**
- Create: `aitalks/{normal,world,kariu,arium,topic,chain}.dic`, `aitalks/shell/{exice,drachen,micro}.dic`, `aitalks/season/{spring,summer,autumn,winter}.dic`, `aitalks/time/midnight.dic`
- Create（作業外）: `<scratchpad>/extract.py`, `<scratchpad>/rs_aitalk.logic.dic`（トリム後の rs_aitalk.dic 案・Task 4 で適用）
- Read: `ghost/master/rs_aitalk.dic`

**Interfaces:**
- Consumes: Task 1 で確定した行番号。
- Produces: 14 データファイル。各データファイルは対応する `aryX : array { ... }`（または chain 定義群）を含む。`aryMicroTalk` は内部に `if/elseif` 条件節を保持。`rs_aitalk.logic.dic`（Task 4 で `rs_aitalk.dic` に上書き適用する案）。

- [ ] **Step 1: 抽出スクリプトを書く**

`<scratchpad>/extract.py`（行番号は Task 1 の確定値で置換。1始まりの行番号、範囲は両端含む）:

```python
import io, os, sys

SRC = r"ghost/master/rs_aitalk.dic"
OUT = r"ghost/master/aitalks"
LOGIC_OUT = sys.argv[1]  # 例: <scratchpad>/rs_aitalk.logic.dic

# --- Task1 で確定した行番号（1始まり・両端含む） ---
NORMAL_HEADER = 120      # 'aryNormalTalk : array'
BRACE_OPEN    = 121      # '{'
CONTENT_START = 122      # 本体ブロック先頭
WORLD   = (11695, 12512) # 世界観＋FS注記
TOPIC   = (12513, 12612) # 質問箱・死生観・空調・日々
KARIU   = (12613, 12756)
ARIUM   = (12757, 12935)
MICRO   = (12936, 13030) # // ミクロの決死圏用 〜 elseif ブロック閉じ（if/elseif 込み）
ARRAY_CLOSE = 13031      # aryNormalTalk の '}'
EXICE   = (13040, 13122) # aryExiceOnlyTalk : array { ... }
DRACHEN = (13131, 13157)
SPRING  = (13167, 13218)
SUMMER  = (13224, 13347)
AUTUMN  = (13353, 13395)
WINTER  = (13401, 13494) # 内部に SurfaceMode 7/9 条件節あり（込みで移動）
MIDNIGHT= (13500, 13522)
CHAIN   = (13524, 13694) # チェイン12種（mikirego は含めない）

with io.open(SRC, "r", encoding="utf-8", newline="") as f:
    lines = f.readlines()   # 各要素は末尾の改行(\n)を保持

def sl(a, b):               # 1始まり両端含むスライス
    return lines[a-1:b]

def body(rng):
    return "".join(sl(rng[0], rng[1]))

def write(path, text):
    full = os.path.join(OUT, path)
    os.makedirs(os.path.dirname(full), exist_ok=True)
    with io.open(full, "w", encoding="utf-8", newline="") as f:
        f.write(text)

def arraywrap(name, inner):
    # 末尾が改行で終わることを保証
    if not inner.endswith("\n"):
        inner += "\n"
    return "%s : array\n{\n%s}\n" % (name, inner)

# --- 新設配列（aryNormalTalk から切り出す節） ---
write("normal.dic", arraywrap("aryNormalTalk", body((CONTENT_START, WORLD[0]-1))))
write("world.dic",  arraywrap("aryWorldTalk",  body(WORLD)))
write("topic.dic",  arraywrap("aryTopicTalk",  body(TOPIC)))
write("kariu.dic",  arraywrap("aryKariuTalk",  body(KARIU)))
write("arium.dic",  arraywrap("aryAriumTalk",  body(ARIUM)))
# ミクロ: if/elseif ブロックをそのまま配列内部に保持
write("shell/micro.dic", arraywrap("aryMicroTalk", body(MICRO)))

# --- 既存標準配列（定義ごと逐語移動） ---
write("shell/exice.dic",   body(EXICE) if body(EXICE).endswith("\n") else body(EXICE)+"\n")
write("shell/drachen.dic", body(DRACHEN) if body(DRACHEN).endswith("\n") else body(DRACHEN)+"\n")
write("season/spring.dic", body(SPRING) if body(SPRING).endswith("\n") else body(SPRING)+"\n")
write("season/summer.dic", body(SUMMER) if body(SUMMER).endswith("\n") else body(SUMMER)+"\n")
write("season/autumn.dic", body(AUTUMN) if body(AUTUMN).endswith("\n") else body(AUTUMN)+"\n")
write("season/winter.dic", body(WINTER) if body(WINTER).endswith("\n") else body(WINTER)+"\n")
write("time/midnight.dic", body(MIDNIGHT) if body(MIDNIGHT).endswith("\n") else body(MIDNIGHT)+"\n")
write("chain.dic",         body(CHAIN) if body(CHAIN).endswith("\n") else body(CHAIN)+"\n")

# --- トリム後 rs_aitalk.dic（ロジック専用）を生成 ---
# 削除する行範囲（元ファイル基準・両端含む）
remove = [
    (WORLD[0], ARRAY_CLOSE-1),   # 世界観〜ミクロ（aryNormalTalk 内のタ末尾節すべて）
    (EXICE[0], EXICE[1]),
    (DRACHEN[0], DRACHEN[1]),
    (SPRING[0], SPRING[1]),
    (SUMMER[0], SUMMER[1]),
    (AUTUMN[0], AUTUMN[1]),
    (WINTER[0], WINTER[1]),
    (MIDNIGHT[0], MIDNIGHT[1]),
    (CHAIN[0], CHAIN[1]),
]
removed = set()
for a, b in remove:
    removed.update(range(a, b+1))   # 1始まり

logic = []
for i, ln in enumerate(lines, start=1):
    if i in removed:
        continue
    logic.append(ln)
    if i == ARRAY_CLOSE - 1:        # 本体ブロック末尾の直後に aryNormalTalk を閉じる
        # （WORLD[0]..ARRAY_CLOSE-1 を削除済みなので、ここで '}' を補う）
        pass
# 上のループでは ARRAY_CLOSE-1 が削除対象なので別処理が必要。下記で再構築する。

logic = []
i = 1
n = len(lines)
while i <= n:
    if WORLD[0] <= i <= ARRAY_CLOSE-1:
        # aryNormalTalk のタ末尾節を丸ごとスキップし、本体直後で配列を閉じる
        logic.append("}\n")             # aryNormalTalk を閉じる（13031 相当）
        i = ARRAY_CLOSE + 1             # 元の '}'(13031) も飛ばす
        continue
    skip = False
    for a, b in remove[1:]:            # 標準配列・チェインの削除
        if a <= i <= b:
            i = b + 1
            skip = True
            break
    if skip:
        continue
    logic.append(lines[i-1])
    i += 1

with io.open(LOGIC_OUT, "w", encoding="utf-8", newline="") as f:
    f.write("".join(logic))

print("written data files + logic file")
```

- [ ] **Step 2: スクリプトを実行**

```bash
python "$TMP/extract.py" "$TMP/rs_aitalk.logic.dic"
ls -R ghost/master/aitalks
```
Expected: 14 個の `.dic` が生成される（直下6＋shell3＋season4＋time1）。

- [ ] **Step 3: 無損失検証（抽出断片の連結＝元の該当範囲）**

`<scratchpad>/verify_lossless.py`:
```python
import io
SRC=r"ghost/master/rs_aitalk.dic"; OUT=r"ghost/master/aitalks"
L=io.open(SRC,encoding="utf-8",newline="").readlines()
def sl(a,b): return "".join(L[a-1:b])
def inner(path):
    t=io.open(OUT+"/"+path,encoding="utf-8",newline="").read()
    # aryX : array\n{\n ... \n}\n の内側を取り出す
    s=t.index("{\n")+2; e=t.rindex("}\n")
    return t[s:e]
checks=[
 (inner("normal.dic"), sl(122,11694)),
 (inner("world.dic"),  sl(11695,12512)),
 (inner("topic.dic"),  sl(12513,12612)),
 (inner("kariu.dic"),  sl(12613,12756)),
 (inner("arium.dic"),  sl(12757,12935)),
 (inner("shell/micro.dic"), sl(12936,13030)),
]
ok=True
for got,exp in checks:
    if got!=exp:
        ok=False; print("MISMATCH len",len(got),len(exp))
# 定義丸ごと移動分はファイル内容＝元スライス（末尾改行差は許容）
def whole(path,a,b):
    t=io.open(OUT+"/"+path,encoding="utf-8",newline="").read()
    return t.rstrip("\n")==sl(a,b).rstrip("\n")
for p,a,b in [("shell/exice.dic",13040,13122),("shell/drachen.dic",13131,13157),
 ("season/spring.dic",13167,13218),("season/summer.dic",13224,13347),
 ("season/autumn.dic",13353,13395),("season/winter.dic",13401,13494),
 ("time/midnight.dic",13500,13522),("chain.dic",13524,13694)]:
    if not whole(p,a,b): ok=False; print("WHOLE MISMATCH",p)
print("LOSSLESS OK" if ok else "LOSSLESS FAILED")
```
```bash
python "$TMP/verify_lossless.py"
```
Expected: `LOSSLESS OK`。失敗時は Task 1 の行番号を見直す。

- [ ] **Step 4: ブレース均衡検証（各データファイル＋ロジック案）**

`<scratchpad>/verify_braces.py`（YAYA文字列 `'...'` `"..."` 内の括弧を無視するクォート対応カウンタ）:
```python
import io,glob
def balance(t):
    depth=0; q=None
    for ch in t:
        if q:
            if ch==q: q=None
            continue
        if ch in "'\"": q=ch; continue
        if ch=="{": depth+=1
        elif ch=="}": depth-=1
        if depth<0: return None
    return depth
files=glob.glob("ghost/master/aitalks/**/*.dic",recursive=True)
files.append(r"%s/rs_aitalk.logic.dic"%__import__("os").environ.get("TMP",""))
bad=False
for fp in files:
    t=io.open(fp,encoding="utf-8",newline="").read()
    d=balance(t)
    print(("OK " if d==0 else "NG "),d,fp)
    if d!=0: bad=True
print("BRACES FAILED" if bad else "BRACES OK")
```
```bash
python "$TMP/verify_braces.py"
```
Expected: 全ファイル `OK 0` と `BRACES OK`。（`$TMP` をスクリプト内で参照できるよう環境変数 `TMP` を設定して実行）

- [ ] **Step 5: データファイルをコミット（不活性なのでゴースト不変）**

```bash
git add ghost/master/aitalks
git commit -m "feat: ランダムトークをカテゴリ別データファイルへ分離 (aitalks/)"
```

---

### Task 4: 切り替え — rs_aitalk.dic をロジック化し yaya.txt と RandomTalkEx を結線

トリム後の `rs_aitalk.dic`（ロジック専用）を適用し、同時に `RandomTalkEx` の合成行・竜姫分岐・`yaya.txt` のロード行を加える。この3つを1コミットで行い、未定義参照が残らないようにする（アトミックな切り替え）。

**Files:**
- Modify: `ghost/master/rs_aitalk.dic`（Task 3 の `rs_aitalk.logic.dic` で上書き → さらに RandomTalkEx を手編集）
- Modify: `ghost/master/yaya.txt`

**Interfaces:**
- Consumes: Task 3 の `aitalks/*` 配列定義（`aryWorldTalk` 等）、`rs_aitalk.logic.dic`。
- Produces: `aitalks/*` を読み込みローテーションに合成した動作。

- [ ] **Step 1: トリム後ロジックを適用**

```bash
cp "$TMP/rs_aitalk.logic.dic" ghost/master/rs_aitalk.dic
grep -nE '^aryNormalTalk : array|^aryExiceOnlyTalk|^aryWinterTalk|^//---- トークチェイン' ghost/master/rs_aitalk.dic || true
```
Expected: `aryNormalTalk : array` は残る（中身は本体ブロックのみ）。`aryExiceOnlyTalk` 等の**配列定義は消えている**（ラッパー `ExiceOnlyTalk` は残る）。`//---- トークチェイン` は消えている。

- [ ] **Step 2: RandomTalkEx に常時A配列の合成行を追加**

`ghost/master/rs_aitalk.dic` の `RandomTalkEx` 内、`_talk_array ,= aryNormalTalk` の直後を次のように変更:
```
	_talk_array = IARRAY
	_talk_array ,= aryNormalTalk
	_talk_array ,= aryWorldTalk
	_talk_array ,= aryKariuTalk
	_talk_array ,= aryAriumTalk
	_talk_array ,= aryTopicTalk
	_talk_array ,= aryMicroTalk
```
（`aryMicroTalk` は配列内部で `shellname == "ミクロの決死圏"` を判定するので無条件合成でよい）

- [ ] **Step 3: 竜姫専用トークの接続（else 分岐を追加）**

同じく `RandomTalkEx` 内の既存ブロックを変更:
```
	if shellname != '竜姫 / Drachen Fürstin' {
		_talk_array ,= aryExiceOnlyTalk
	}
	else {
		_talk_array ,= aryDrachenFurstinTalk
	}
```

- [ ] **Step 4: yaya.txt にデータファイルのロード行を追加**

`dic, rs_aitalk.dic` 行の直後に追加（パス区切りは `/`）:
```
dic, aitalks/normal.dic
dic, aitalks/world.dic
dic, aitalks/kariu.dic
dic, aitalks/arium.dic
dic, aitalks/topic.dic
dic, aitalks/chain.dic
dic, aitalks/shell/exice.dic
dic, aitalks/shell/drachen.dic
dic, aitalks/shell/micro.dic
dic, aitalks/season/spring.dic
dic, aitalks/season/summer.dic
dic, aitalks/season/autumn.dic
dic, aitalks/season/winter.dic
dic, aitalks/time/midnight.dic
```

- [ ] **Step 5: 参照解決の静的検証**

`<scratchpad>/verify_refs.py`:
```python
import io,glob,re
texts={fp:io.open(fp,encoding="utf-8",newline="").read()
       for fp in glob.glob("ghost/master/**/*.dic",recursive=True)}
alltext="\n".join(texts.values())
need=["aryNormalTalk","aryWorldTalk","aryKariuTalk","aryAriumTalk","aryTopicTalk",
      "aryMicroTalk","aryExiceOnlyTalk","aryDrachenFurstinTalk","arySpringTalk",
      "arySummerTalk","aryAutumnTalk","aryWinterTalk","aryMidnightTalk"]
bad=False
for name in need:
    # 定義は『name : array』または『name\n{』形式
    if not re.search(r'(?m)^%s\s*:\s*array'%re.escape(name), alltext):
        print("MISSING DEF:",name); bad=True
# チェイン参照 :chain=xxx が定義(行頭 xxx\n{{CHAIN)を持つか
chains=set(re.findall(r':chain=([^\'"\s:]+)', alltext))
for c in chains:
    if c in ("end",): continue
    if not re.search(r'(?m)^%s\s*$'%re.escape(c), alltext):
        print("MISSING CHAIN:",c); bad=True
print("REFS FAILED" if bad else "REFS OK")
```
```bash
python "$TMP/verify_refs.py"
```
Expected: `REFS OK`（全配列定義が存在、全 `:chain=` 参照先が存在）。`MISSING` が出たら修正。

- [ ] **Step 6: CHECKPOINT（ユーザ実行時ロード確認）**

ユーザに依頼: SSP でゴースト再起動。YAYA エラーログが無いこと、ランダムトークが従来どおり出ること、`about:talk`（コミュニケート）で件数が表示されることを確認。
Expected: エラー無し・トーク出現。失敗時はログのファイル名/行で原因特定。

- [ ] **Step 7: コミット**

```bash
git add ghost/master/rs_aitalk.dic ghost/master/yaya.txt
git commit -m "refactor: rs_aitalk.dic をロジック専用化し aitalks を結線・竜姫トーク有効化"
```

---

### Task 5: rs_communicate.dic の統計更新とバグ修正

`aryNormalTalk` から節を抜き出したことに伴い `about:talk` 統計を整える。既存の合計式バグ（`arySpringTalk` 二重加算・`aryMidnightTalk` 漏れ）も直す。

**Files:**
- Modify: `ghost/master/rs_communicate.dic`（165–174行付近）

- [ ] **Step 1: 表示行に新設配列（常時A）を追加**

`通常ランダムトーク: %(ARRAYSIZE(aryNormalTalk))\n/` の並びに追加:
```
			世界観トーク: %(ARRAYSIZE(aryWorldTalk))\n/
			カリウトーク: %(ARRAYSIZE(aryKariuTalk))\n/
			アリウムトーク: %(ARRAYSIZE(aryAriumTalk))\n/
			小テーマトーク: %(ARRAYSIZE(aryTopicTalk))\n/
```
（`aryMicroTalk` はシェル依存で非ミクロシェルでは 0 になるため表示は任意。入れる場合も無害。）

- [ ] **Step 2: 合計式を修正（重複・漏れ解消＋新配列加算）**

既存の合計式を次へ置換:
```
			\n合計: %(ARRAYSIZE(aryNormalTalk) + ARRAYSIZE(aryWorldTalk) + ARRAYSIZE(aryKariuTalk) + ARRAYSIZE(aryAriumTalk) + ARRAYSIZE(aryTopicTalk) + ARRAYSIZE(arySpringTalk) + ARRAYSIZE(arySummerTalk) + ARRAYSIZE(aryAutumnTalk) + ARRAYSIZE(aryWinterTalk) + ARRAYSIZE(aryMidnightTalk) + ARRAYSIZE(aryNormalTalkOld))\e"
```

- [ ] **Step 3: 静的確認（参照名の存在）**

```bash
python "$TMP/verify_refs.py"
```
Expected: `REFS OK`（`aryNormalTalkOld` は `rs_aitalk_old.dic` に定義。未定義警告が出る場合は綴り `aryNormalTalkOld` の所在を確認）。

- [ ] **Step 4: CHECKPOINT（ユーザ）**

`about:talk` を実行し、各カテゴリ件数と合計が表示・整合することを確認。

- [ ] **Step 5: コミット**

```bash
git add ghost/master/rs_communicate.dic
git commit -m "fix: about:talk 統計に新カテゴリを追加し合計式の重複/漏れを修正"
```

---

### Task 6: 最終統合検証と受け入れ

全体の静的検証を再実行し、ユーザの実機受け入れで完了とする。

**Files:** （変更なし。検証のみ）

- [ ] **Step 1: 無損失・ブレース・参照の総点検**

```bash
python "$TMP/verify_lossless.py"   # LOSSLESS OK
python "$TMP/verify_braces.py"     # BRACES OK（aitalks 配下）
python "$TMP/verify_refs.py"       # REFS OK
git diff --stat HEAD~3             # 変更ファイル一覧の確認
```
Expected: 3つとも OK。差分は `aitalks/*`（新規14）＋ `rs_aitalk.dic` 縮小 ＋ `yaya.txt` ＋ `rs_communicate.dic`。

- [ ] **Step 2: チェイン発火の抜き取り確認（任意・実機）**

`:chain=star` 等を含むトークから連鎖が継続することをユーザが確認。

- [ ] **Step 3: 竜姫シェルでの受け入れ（実機）**

ユーザ依頼: シェルを `竜姫 / Drachen Fürstin` に切替。竜姫専用トークが出現し、イクサイス専用が出ないことを確認。master では従来どおり（イクサイス専用が出て竜姫専用は出ない）。
Expected: 竜姫専用トークがローテーションに登場（本設計の主目的の一つ）。

- [ ] **Step 4: ミクロシェルでの受け入れ（任意・実機）**

`ミクロの決死圏`（Fantastic_Voyage）シェルで、ミクロ専用トークが従来どおり出ることを確認。

- [ ] **Step 5: 完了報告**

`docs/superpowers/specs/2026-06-29-rs-aitalk-split-design.md` の成果物と差分が一致することを確認し、ブランチ `feature/rs-aitalk-split` の統合方針（PR/マージ）をユーザに確認する（superpowers:finishing-a-development-branch）。

---

## Self-Review（計画作成者によるチェック結果）

- **Spec coverage:** 分離（Task3/4）・分類体系のフォルダ化（Task2/3）・挙動保存（無損失検証 Task3, 実機 Task4/6）・外部参照維持（Task4/5 の参照検証）・竜姫接続（Task4 Step3, Task6 Step3）・統計更新とバグ修正（Task5）・サブフォルダdicリスク（Task2 スモーク）を網羅。
- **Placeholder scan:** 具体コード/コマンド/期待値を記載。行番号は Task1 で実値確認する前提の「目安」と明示。
- **Type/name consistency:** 配列名は Global Constraints と各 Task で一致（aryWorldTalk/aryKariuTalk/aryAriumTalk/aryTopicTalk/aryMicroTalk、据え置き群）。
