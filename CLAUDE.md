# CLAUDE.md — 筋トレログ (Strength Log)

このリポジトリで作業するClaude Code向けの引き継ぎ書です。**作業を始める前に必ず読んでください。**

## これは何か

スマホで使う、筋トレ記録用の単一HTMLアプリ。`index.html` 一枚で完結しており、ビルド工程・依存パッケージ・フレームワークは一切ありません。バニラのHTML/CSS/JS。フォントだけGoogle Fonts (Oswald / Noto Sans JP) を読み込みます。

- `index.html` — 本体（これを編集する）
- `strength-log.html` — `index.html` と中身は完全に同一のコピー。**両方を常に一致させること**（片方だけ直すと不整合になる）。
- `index.personal.html` — 個人データ埋め込み済みのローカル専用コピー（後述、`.gitignore`済み）。
- `manifest.json` / `icon-192.png` / `icon-512.png` / `icon-512-maskable.png` / `apple-touch-icon.png` — PWA化（iOSの「ホーム画面に追加」）用。アイコンは「グリッチバーベル」（バーベルを緑・赤にずらして三重に重ねたマーク、Pillowで生成）。ヘッダー左上の`.brand-mark`もこの`icon-192.png`を表示している。アイコンを差し替える場合はこの5ファイルとヘッダーの表示を両方確認すること。

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

### 初回登録 (onboarding) とプロフィール
本当に何も無い初回起動時だけ（`loadMeta`、迷子データ復元も旧データ移行も無い場合）、`showOnboarding()`が名前・性別・トレーニング歴・目的を聞く全画面を出す。結果は各ユーザーblobの`data.profile`（`{gender, experience, purpose}`）に保存され、ユーザー切り替えシートの「プロフィールを編集」（`openProfileModal`/`saveProfile`）から後から変更できる。
- `profile.purpose`（`strength`/`hypertrophy`/`cut`）は`targetFor`のしきい値（`PURPOSE_THRESHOLDS`）を切り替える: 筋力アップ=5回、筋量アップ=10回（デフォルト）、減量=15回以上で重量アップ。`profile`が無い既存ユーザーはhypertrophy扱いにフォールバックする。
- `gender`は現状プロフィール保存のみで、計算には使っていない。`experience`（`beginner`/`intermediate`/`advanced`）は`planFor`の目安ロジックの分岐に使う（下記）。

### 埋め込みデータ (SUGANO_BUNDLE)
`<script id="sugano-bundle" src="sugano-bundle.local.js">` が、あるユーザー(菅野涼太)のExcelから変換したセット（4,000件超、随時Excel取り込みで増える）を読み込む。初回に一度だけ取り込まれる。**このデータを破壊しないこと。** 破壊的変更をする関数（削除・マージ）は、必ず件数を確認してから。

**このファイルは `.gitignore` されており、Gitリポジトリには含まれない。** 実データは `sugano-bundle.local.js`（リポジトリ直下、ローカルのみ）に `window.SUGANO_BUNDLE={...}` の形で置かれている。理由: 本リポジトリはGitHub Pagesで公開しているため、実在する個人の名前とトレーニング記録をコミットすると誰でも閲覧できてしまう（過去に一度、公開リポジトリへ丸ごとコミットしてしまい、リポジトリを削除して作り直す事故があった）。
- `index.html` / `strength-log.html` は空の参照タグだけを持つ。`sugano-bundle.local.js` が存在しない環境（GitHub Pages上の公開版や他人の端末）では単にスクリプトが404し、`window.SUGANO_BUNDLE` は `undefined` のまま → 通常の「データなし」の起動になる（既存のフォールバック処理で問題なく動く）。
- **個人データを含むファイルを絶対にコミットしない。** 新しく個人データ的なものを埋め込む場合も、必ずこの「`.local.js`を`.gitignore`する」パターンに従うこと。

### 個人用ファイル (index.personal.html)
`index.html` と `sugano-bundle.local.js` を1回だけ合体させた自己完結型のスナップショット。OneDrive経由でスマホなど他端末からも開けるようにするためのもの（GitHub Pagesは個人アカウントでは非公開にできないため、データ入りのURLは絶対に公開しない設計）。`.gitignore`済み。
- **自動更新されない。** `index.html` のコードや `sugano-bundle.local.js` のデータを更新したら、その都度作り直す必要がある（下記コマンド参照）。

```bash
# index.personal.html の再生成（<script id="sugano-bundle" src=...> の行番号を都度確認すること）
grep -n 'id="sugano-bundle"' index.html
head -n <その行-1> index.html > index.personal.html
printf '<script id="sugano-bundle">' >> index.personal.html
cat sugano-bundle.local.js >> index.personal.html
printf '</script>\n' >> index.personal.html
tail -n +<その行+1> index.html >> index.personal.html
```

### Excelデータの更新方法
アプリ内にExcel取り込みUIは無い（一時期SheetJSで実装したが、UIが煩雑なため削除済み）。菅野涼太の`sugano-bundle.local.js`を最新のExcelに合わせて更新したい場合は、Claude Codeにその都度Excelファイルの場所を伝えて変換してもらう運用（Python + openpyxlで差分抽出 → `mergeIncoming`と同じ重複排除ロジックで安全にマージ）。

## コードの地図（すべて index.html 内の1つのIIFE）

- 保存層: `probeBackend / sGet / sSet / sDel / sList / save / saveMeta`
- 読み込み: `loadMeta / loadUserData / reconcileUsers / recoverUsers / maybeImport / mergeIncoming / ensureData`
- 記録タブ: `renderSplitChips / renderSelects / renderInsight（前回比較テーブル+目安）/ renderToday`
- 履歴タブ: `renderHistory（30日ずつページング）/ setChip / splitsForDay / moveDay`
- 分割タブ: `renderSplits / exRowHtml / openSplitModal / openAddExSheet / moveExercise`
- ユーザー: `renderUserPill / renderUserList / switchUser / commitUser / runRecovery`
- 目安ロジック: `planFor` が `profile.experience==="advanced"` かどうかで2方式に分岐する。
  - **中級者・初心者（既定）**: `doubleTargets`。規定回数（`repsThresholds().up`）に全セットが達したら+1段階、達成までは同じ重量で回数を狙う複合プログレッション式。始めたて（過去セッション数<3）は繰り上げが2セットずつ。
  - **上級者**: `advTarget`（`targetFor`にフォールバックあり）。直近3セッション（`recentSessions(exId,3)`）の重量が一定幅で伸びていればそのトレンドで次の重量を予測するトレンド学習式。
  - 前回が「ボリューム未達の追加セット」で通常よりちょうど1セット多かった場合の吸収ロジック（catchup、`planFor`内）は上級者モードのみに残っている（中級・初心者は`doubleTargets`が完結して処理するため通らない）。
  - 増量幅(`step`)は種目ごとの`exercise.step`（kg、種目編集モーダルで設定）→ `inferStep(exId)`（履歴から自動推定）→ `2.5`（既定）の優先順で決まる。
  - 重量は入力・計算・刻み推定・表示すべて**小数第2位**まで対応（`roundW`/`fmtW`/`snapStep`）。ボリューム・差分表示は従来通り`fmtNum`（小数1桁）のまま区別すること。
- セッションのグルーピング: `sessionKey`/`currentSessionKey`/`sessionMap`が、前のセットから1時間以内なら日付をまたいでも同じセッション（同じ日）として扱う（深夜練習対応）。`dateKey`/`todayKey`はラベル表示・インポート時の重複判定にのみ残っており、集計系（`exSessions`/`todaySetsOf`/`pastSessionsList`/`renderToday`/`renderHistory`/`moveDay`）は`sessionKey`基準。`_sessionMap`は`save()`内で毎回無効化される。

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

## ホスティング

- GitHubリポジトリ: https://github.com/TimeToBulkUp/strength-log （Public）
- GitHub Pages公開URL: https://timetobulkup.github.io/strength-log/
- pushすると数分でPagesに反映される。`sugano-bundle.local.js` はpushされないので、公開版は常にデータなしの汎用アプリとして動く。

## やらないこと / 注意

- localStorage を `window.storage` が使える環境で使わない（プレビューで壊れる）。フォールバック時のみ。
- 破壊的なデータ変更を「確認・テストせずに」行わない。特に4,290セットの埋め込みデータ。
- 目標体重・ログイン・複数端末同期は未実装。やるならサーバー(Supabase/Firebase)が必要で、別途相談。
- **`sugano-bundle.local.js`（個人データ）を絶対にGitにコミット/pushしない。** `.gitignore` を消さないこと。
