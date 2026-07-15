# CLAUDE.md — 筋トレログ (Strength Log)

このリポジトリで作業するClaude Code向けの引き継ぎ書です。**作業を始める前に必ず読んでください。**

## これは何か

スマホで使う、筋トレ記録用の単一HTMLアプリ。`index.html` 一枚で完結しており、ビルド工程・依存パッケージ・フレームワークは一切ありません。バニラのHTML/CSS/JS。フォントだけGoogle Fonts (Oswald / Noto Sans JP) を読み込みます。

- `index.html` — 本体（これを編集する）
- `strength-log.html` — `index.html` と中身は完全に同一のコピー。**両方を常に一致させること**（片方だけ直すと不整合になる）。

## 実行・確認方法

ビルド不要。ブラウザで `index.html` を開くだけで動く。編集後は最低限これを確認する:

```bash
# 構文チェック（メインの<script>を取り出して parse できるか）
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=[...h.matchAll(/<script[^>]*>([\s\S]*?)<\/script>/g)].pop()[1];new Function('\"use strict\";'+m);console.log('OK')"
```

## 最重要: データ保存の落とし穴（何度も事故った）

- 保存は `window.storage`（Anthropic製プレビュー環境のみ）→ なければ `localStorage`（通常のブラウザ／ホスティング時）→ どちらもダメなら**メモリのみ**、の順でフォールバックする。
- **`window.storage` は「存在するかどうか」で判定してはいけない。** プレビュー環境では存在するのに実際には保存できないことがある。必ず「書いて読み返せるか」を実測する `probeBackend()` の結果 (`backend` 変数) で分岐すること。
- **書き込んだ内容は必ず `mem` にも保持する** (`sSet` 内)。保存が効かない端末でも、画面上のデータが初期化されないようにするため。これを外すと「記録が全部消えた」事故が再発する。
- 取り込み直後などに「保存 → すぐ読み戻し」をすると、保存が効かない環境で読み戻しが `null` になり、`normalize(null)` が初期データを返してメモリ上のデータを上書きしてしまう。**読み戻しに失敗したら初期化せず、メモリ上の値を使う**こと（`loadUserData` 参照）。
- ユーザー一覧 (`meta`) が読めないときに「初回起動」とみなして上書きしない。`reconcileUsers()` が保存領域を走査して迷子のプロフィールを復元する。この設計を壊さないこと。

## データ構造（localStorage / window.storage のキー）

- `strengthlog:meta:v1` … `{ users:[{id,name}], currentUserId }`
- `strengthlog:user:<id>:v1` … `{ exercises:[{id,name,body}], splits:[{id,name,exIds:[]}], sets:[{id,exId,weight,reps,ts}], __name }`
- `strengthlog:import:<importId>` … 取り込み済みフラグ

`__name` は一覧が壊れても名前ごと復元できるよう、各ユーザーblobに自己記述として持たせている。消さないこと。

### 埋め込みデータ (SUGANO_BUNDLE)
`<script id="sugano-bundle">` に、あるユーザー(菅野涼太)のExcelから変換した約4,290セットが埋め込まれている。初回に一度だけ取り込まれる。**この4,290セットを破壊しないこと。** 破壊的変更をする関数（削除・マージ）は、必ず件数を確認してから。

## コードの地図（すべて index.html 内の1つのIIFE）

- 保存層: `probeBackend / sGet / sSet / sDel / sList / save / saveMeta`
- 読み込み: `loadMeta / loadUserData / reconcileUsers / recoverUsers / maybeImport / ensureData`
- 記録タブ: `renderSplitChips / renderSelects / renderInsight（前回比較テーブル+目安）/ renderToday`
- 履歴タブ: `renderHistory（30日ずつページング）/ setChip / splitsForDay / moveDay`
- 分割タブ: `renderSplits / exRowHtml / openSplitModal / openAddExSheet / moveExercise`
- ユーザー: `renderUserPill / renderUserList / switchUser / commitUser / runRecovery`
- 目安ロジック: `planFor / targetFor / recentSessions`（前回の同じ順のセットを基準に、10回以上で+2.5kg、5〜9回で+1回）

## 変更時のルール

1. **関数を消す/切り出すときは範囲に細心の注意を。** 過去、隣の関数 (`renderSplitChips`) を巻き込んで削除し、`renderAll` が起動時に落ちてアプリ全体が無反応になった事故がある。編集後、下記の「呼ばれているのに未定義の関数」チェックを必ず走らせる。
2. **HTMLの閉じタグ。** `<section>` の閉じ忘れでタブが入れ子になり、履歴タブが表示されない事故があった。`<main>` 内の section が全てトップレベル(入れ子でない)か確認する。
3. **重い描画は可視時のみ。** 履歴は全部描くと88万文字になりスマホが固まる。`renderAll` は履歴タブが active のときだけ `renderHistory` を呼ぶ。この最適化を戻さないこと。
4. 編集したら `index.html` → `strength-log.html` へコピーして同期する。

```bash
# 呼ばれているのに未定義の関数がないか
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=[...h.matchAll(/<script[^>]*>([\s\S]*?)<\/script>/g)].pop()[1];const d=new Set();for(const x of m.matchAll(/function\s+([A-Za-z_\$][\w\$]*)\s*\(/g))d.add(x[1]);for(const x of m.matchAll(/(?:const|let|var)\s+([A-Za-z_\$][\w\$]*)\s*=/g))d.add(x[1]);const need=['renderAll','renderSplitChips','renderSplits','renderHistory','renderInsight','renderToday','renderSelects'];console.log(need.filter(n=>!d.has(n)).length?'MISSING: '+need.filter(n=>!d.has(n)):'OK')"
# 同期
cp index.html strength-log.html
```

## デザインの約束

- 配色: グラファイト地 (`--bg:#14161a`) + アンバー (`--accent:#f6a609`)。CSS変数で管理。生の色をベタ書きしない。
- フォント: 数字・見出しは Oswald、日本語は Noto Sans JP。
- 「今日」の行は赤 (`--today:#ff6f63`) + 白枠で強調。前回超えのセットは緑 (`--up:#6fca5a`) の下地。

## やらないこと / 注意

- localStorage を `window.storage` が使える環境で使わない（プレビューで壊れる）。フォールバック時のみ。
- 破壊的なデータ変更を「確認・テストせずに」行わない。特に4,290セットの埋め込みデータ。
- 目標体重・ログイン・複数端末同期は未実装。やるならサーバー(Supabase/Firebase)が必要で、別途相談。
