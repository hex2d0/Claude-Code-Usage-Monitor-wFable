# Phase 0 調査結果：3行ブロックゲージ

調査日：2026-10-13
対象：フォーク`hex2d0/Claude-Code-Usage-Monitor-wFable`の`78b3d12`。上流`CodeZeno/Claude-Code-Usage-Monitor`の`main`（`623e891`、v2.18.1、2026-10-05）に`PLAN.md`を1ファイル足しただけの状態。2026-10-13時点で上流`main`に新しいコミットはない。

コードは変更していない。このコミットで追加するのはこの文書だけ。

> **2026-10-13追記：** Phase 1・2は完了し、[hex2d0/Claude-Code-Usage-Monitor-wFable#1](https://github.com/hex2d0/Claude-Code-Usage-Monitor-wFable/pull/1)でマージされた。手順2の結果と実機確認の結果は§8にまとめた。§0〜§7はPhase 0時点の記録のまま残している。

---

## 0. 要約

1. **PLAN.md §4の「既知事項」は古い。** 上流はv2.13.43（2026-09-21）から`/api/oauth/usage`の`limits`配列を読んでいる。Fable週次枠は`claude.limits.weekly_scoped_fable.*`としてテーマから参照できる。このため、PLAN Phase 1の手順1と2（`UsageResponse`へのフィールド追加、`UsageData`へのOptionセクション追加）は不要になる見込み。Phase 1で残る作業は、`--diagnose`へのログ追加とフィクスチャテストだけ。
2. **3行ブロックゲージは、ほぼテーマJSONだけで作れる。** 3行レイアウト、10×10px＋1px空白のゲージ、色、見出し、Fableの割り当て、`--`表示、`4d/23h/59m/59s`のカウントダウンは、いずれも既存のテーマ機能で表現できる。ブロックの途中までの塗り（1px＝1%）も、100% DPIでは式による補正で厳密に一致する。
3. **コード変更が必要なのは、組み込みテーマとしての登録だけ。** 対象は`build.rs`と`src/theme_engine.rs`のリスト追加、それに合わせた既存テスト2か所の修正。描画コード（`theme_rendering.rs`）は触らずに済む見込み。
4. **仕様どおりにできない点は2つ。** DPI 125%と150%では、上流の描画がブロックを整数pxにそろえるため「1論理px＝1%」が厳密には成り立たない。定数をまとめる場所もない（テーマJSONに変数がない）。どちらも判断をお願いしたい（§6）。
5. **手順2（生レスポンスの取得）は未実施。** このコンテナにはユーザーの認証情報がないため。取得の手順と、送り返してほしい項目を§3にまとめた。
6. **ビルドは通った。** Linux上でWindows向け（GNUターゲット）のリリースEXEをクロスビルドできた。既存テストはwine上で515件中366件が通る。ただし`theme_engine`系のテストはwineで動かないため、Phase 2のテストはWindows実機での実行が必要。

---

## 1. 作業ブランチ

PLAN §8は`feature/block-gauge`を指定しているが、このセッションの実行環境は`claude/gallant-ptolemy-xn9prw`への作業とpushを求めている。今回は後者を使った。`feature/block-gauge`に揃える必要があれば指示してほしい。

---

## 2. 手順1：ビルドと動作確認

### 2.1 ビルド

この環境はLinuxのコンテナで、Windowsも、上流CIが使うMSVCツールチェーンもない。代わりにmingw-w64で`x86_64-pc-windows-gnu`向けにクロスビルドした。

| 確認 | 結果 |
|---|---|
| `cargo build --release --locked --target x86_64-pc-windows-gnu` | 成功。`claude-code-usage-monitor.exe`、7.7MB、PE32+ GUI |
| `cargo test --no-run`（同ターゲット） | 成功。テスト515件のバイナリを生成 |
| MSVCターゲット（上流CIの`windows-latest`と同じ構成） | この環境では未実施 |

ビルドで1点だけ回避策が要った。`src/poller/windows_credentials.rs:23`の`#[link(name = "Advapi32")]`は大文字を含むため、大文字小文字を区別するLinuxのmingwでは`libadvapi32.a`が見つからない。作業用ディレクトリに`libAdvapi32.a`という名前のシンボリックリンクを作り、`RUSTFLAGS=-L`で渡して解決した。リポジトリは変更していない。Windows上のビルドには影響しない。

### 2.2 既存テスト（変更前のベースライン）

テストバイナリはwine 9.0で動かした。Windows 10以降に標準で入っている`winsqlite3.dll`がwineにはないので、crates.io経由で入手したSQLiteソースから代わりのDLLをビルドした。テストは1件ずつ別プロセスで実行した。

| 結果 | 件数 | 内訳 |
|---|---|---|
| 成功 | 366 | `poller::*`（Claude取得層を含む）、`updater`、`accounts`、`models`など |
| 中断 | 137 | wine 9.0に`SystemTimeToTzSpecificLocalTimeEx`が実装されていない（呼び出し元は`src/theme_engine/theme_datetime.rs:159`）。`theme_engine::tests`の67件、`studio_app`の31件、`window`配置系の26件など |
| 失敗 | 8 | ウィンドウ合成やプロセス起動など、Win32の実挙動に依存するもの（`window::layered_window_tests`の4件など） |
| 無視 | 4 | 上流で`#[ignore]`指定されたもの |

これがPhase 1と2に与える影響は次のとおり。

- **Phase 1**（`poller::claude`と`limits`のテスト）は、この環境でも大半を検証できる。
- **Phase 2**（テーマの描画テスト）は、この環境では実行できない。Windows上で`cargo test`を流してもらう必要がある。

### 2.3 現状の表示

Windowsのデスクトップがないため、この環境では確認できない。手元のWindows 11で上流のEXEが表示されることを確認してほしい。

---

## 3. 手順2：`/api/oauth/usage`の生レスポンス（未実施）

### 3.1 現状

このコンテナにはユーザーの認証情報（`%USERPROFILE%\.claude\.credentials.json`）がない。ほかの手段で代用することもしない。そのため、レスポンスの構造はまだ確定していない。

上流のコードが前提にしている形は次のとおり。出典は`src/poller/claude/limits.rs:8-17`と、テストのフィクスチャ（`limits.rs:170`、`src/poller/claude.rs:1146`、`claude.rs:1165`）。

```json
{
  "five_hour": {"utilization": 17.0, "resets_at": "2026-10-13T08:00:00.123456Z"},
  "seven_day": {"utilization": 42.0, "resets_at": "..."},
  "limits": [
    {"kind": "weekly_scoped", "percent": 10, "resets_at": "...",
     "scope": {"model": {"id": null, "display_name": "Fable"}}, "is_active": true}
  ]
}
```

ただし、上流のフィクスチャのうち「実レスポンスから取った」と明記されているのは`spend`の1件（`claude.rs:1303`）だけ。`limits`まわりのフィクスチャには出典が書かれていない。`USER_GUIDE.md:130`でも、バインディングの例は「説明用」とされている。

### 3.2 実レスポンスで確かめたいこと

| # | 項目 | 上流の前提 | 違っていた場合の影響 |
|---|---|---|---|
| 1 | `limits`があり、JSON配列であること | 配列 | 配列以外だとレスポンス全体の読み込みに失敗し、3行とも前回値のまま（stale）になる（`claude.rs:31`、`236-245`） |
| 2 | Fable枠の`kind` | `weekly_scoped` | テーマが参照するキー名が変わる |
| 3 | `scope.model.display_name`と`scope.model.id` | `"Fable"`と`null` | キーは`slug(kind)_slug(display_name、なければid)`で作られる（`limits.rs:39-46`）。たとえば`"Claude Fable 5.1"`なら`weekly_scoped_claude_fable_5_1`になり、`weekly_scoped_fable`を参照するテーマでは**エラーにならないまま「枠なし」の表示になる** |
| 4 | 使用率のフィールド名・型・尺度 | `percent`（`utilization`も可）、数値、0〜100 | `null`や文字列ならその要素は捨てられる。`percent`と`utilization`が両方あるとserdeが重複エラーを出し、やはり捨てられる。0〜1の小数なら、ほぼ0%と表示される |
| 5 | `resets_at`の型と形式 | 文字列。`Z`か`±HH:MM`、小数秒は可 | 数値ならその要素は捨てられる。`+0000`（コロンなし）はリセット時刻不明として扱われる（`src/poller.rs:420-514`） |
| 6 | `is_active`の型 | 真偽値。省略時はfalse | `null`ならその要素は捨てられる |
| 7 | `five_hour`と`seven_day`が引き続きあるか | ある。なければ`limits`の`session`と`weekly_all`で代用する | 両方ないと、1行目と2行目も「枠なし」の表示になる |
| 8 | Fableの`resets_at`が未来の時刻であること | 未来 | 過去の時刻が返ると、上流の仕様で5秒ごとの再取得が続く（`src/window.rs:2798-2802`）。上流の既存の挙動で、今回の変更とは関係ない |

### 3.3 取得の方法（どちらか、または両方）

**A. APIを追加で叩かない方法（先にこちらを推奨）**

上流のEXEを普段どおり1回動かすと、`%APPDATA%\ClaudeCodeUsageMonitor\usage-cache.json`にパース後の`limits`が保存される（`src/app_settings.rs:308-309`、`375-384`）。各要素の`key`、`kind`、`model`、`model_id`、`scope`、`is_active`、`percentage`、`resets_at`を確認できる。Theme Studioの「Variables」パネルでも、`claude.limits.*`の正確なキー名が見られる。

この方法では、元のフィールド名（`percent`か`utilization`か）、`resets_at`の元の書式、パースに失敗して捨てられた要素はわからない。`scope`をそのまま貼る前に、アカウントを特定できる値がないか確認してほしい。

**B. 生レスポンスを1回だけ取得する方法（PLANの手順2）**

Windows PowerShell 5.1（`powershell.exe`）で1回だけ実行する。PowerShell 7は日時の文字列を自動で`DateTime`に変換してしまい、`resets_at`の元の書式がわからなくなるので使わない。

出力されるのは、キー名、型、使用率、リセット時刻、モデル名だけ。トークンやその他の値は出力しない。トークンの期限が切れていて401が返る場合は、先に`claude`を1回起動してから再実行する。

```powershell
$ErrorActionPreference = 'Stop'
$credPath = Join-Path $env:USERPROFILE '.claude\.credentials.json'
try {
  $token = (Get-Content -Raw -LiteralPath $credPath | ConvertFrom-Json).claudeAiOauth.accessToken
  $res = Invoke-WebRequest -UseBasicParsing -Uri 'https://api.anthropic.com/api/oauth/usage' `
    -Headers @{ Authorization = "Bearer $token"; 'anthropic-beta' = 'oauth-2025-04-20' }
} finally {
  Remove-Variable token -ErrorAction SilentlyContinue
}
$raw = $res.Content | ConvertFrom-Json
function TypeOf($v) { if ($null -eq $v) { 'null' } else { $v.GetType().Name } }
$out = [ordered]@{
  top_level = @(foreach ($p in $raw.PSObject.Properties) { [ordered]@{ key = $p.Name; type = (TypeOf $p.Value) } })
  buckets = @(foreach ($p in $raw.PSObject.Properties) {
    if ($null -ne $p.Value -and $p.Value.PSObject.Properties['utilization']) {
      [ordered]@{ key = $p.Name; fields = @($p.Value.PSObject.Properties.Name)
                  utilization = $p.Value.utilization; resets_at = $p.Value.resets_at }
    } })
  limits_type = (TypeOf $raw.limits)
  limits = @(foreach ($l in $raw.limits) {
    [ordered]@{
      fields = @($l.PSObject.Properties.Name)
      kind = $l.kind; percent = $l.percent; utilization = $l.utilization
      percent_type = (TypeOf $l.percent); utilization_type = (TypeOf $l.utilization)
      resets_at = $l.resets_at; resets_at_type = (TypeOf $l.resets_at)
      is_active = $l.is_active; is_active_type = (TypeOf $l.is_active)
      scope_fields = @(if ($l.scope) { $l.scope.PSObject.Properties.Name })
      model_fields = @(if ($l.scope.model) { $l.scope.model.PSObject.Properties.Name })
      model_id = $l.scope.model.id; model_display_name = $l.scope.model.display_name
    } })
}
$path = Join-Path $env:TEMP 'oauth-usage-redacted.json'
$out | ConvertTo-Json -Depth 6 | Set-Content -Encoding UTF8 $path
Write-Output "wrote $path"
```

出力された`oauth-usage-redacted.json`の中身を確認したうえで、内容を送ってほしい。これがPhase 1のフィクスチャの元になる。

---

## 4. PLAN §4「上流コードの既知事項」と現在のコードの照合

| PLANの記述 | 現在のコード | 根拠 |
|---|---|---|
| `/api/oauth/usage`を先に試す（Bearer、`anthropic-beta: oauth-2025-04-20`） | 正しい | `src/poller/claude.rs:19`、`205-209` |
| `UsageResponse`は`five_hour`・`seven_day`・`spend`のみを読み、モデル別の枠は読まない | **古い**。`limits: Option<Vec<Value>>`と、残りのキーを受け取る`#[serde(flatten)] extra`もある。パース処理は`src/poller/claude/limits.rs`にある | `claude.rs:26-34`、`256-263`、`CHANGELOG.md:417` |
| 429と5xxは一時障害として扱い、フォールバックしない | 正しい（401と403は認証エラー、通信エラーも一時障害） | `claude.rs:312-320`、テスト`1266-1282` |
| 404等のときだけMessages APIのヘッダから読む | おおむね正しい（認証エラー・429・5xx以外のHTTPエラーはすべてフォールバックする） | `claude.rs:317`、`224-229` |
| ヘッダ経路にFable枠があるかは未確認 | **コード上は「ない」で確定**。読むのは5h、7d、status、resetのヘッダだけで、`limits`は空のまま返る。この経路を通るアカウントでは、FBL1W行が常に`--`になる | `claude.rs:419-473` |
| 認証情報は`.credentials.json`、デスクトップアプリ、WSLの順に探す | 正しい（`CLAUDE_CONFIG_DIR`を指定するとデスクトップアプリとWSLは探さない） | `claude.rs:613-615`、`697-737` |
| `src/poller.rs`：ポーリング、Retry-After、`time_until_display_change` | 正しい | `src/poller.rs:93-108`、`363-382`、`539-561` |
| `src/models.rs`：`UsageData`・`UsageSection`（`session`・`weekly`・`credits`） | **不完全**。ほかに`monthly`、`limits: Vec<UsageLimit>`、`stale`もある | `src/models.rs:58-89` |
| 設定ファイルと診断ログの場所 | 正しい | `src/app_settings.rs:253-258`、`src/diagnose.rs:59-61` |
| Theme Studioは未調査 | 調査済み（§5）。Claudeの追加枠のバインディングは`USER_GUIDE.md:110-160`に説明があり、Fableのゲージを作る例も載っている | `src/theme_engine/theme_limits.rs:5-98` |

---

## 5. 手順3：テーマ機能で何ができるか

### 5.1 判定一覧

判定の意味：
- **テーマのみ**：テーマJSONだけで実現できる
- **上流実装済み**：上流にすでにあり、追加作業は不要
- **要コード**：Rustコードの変更が必要

| PLANの項目 | 判定 | 方法と根拠 |
|---|---|---|
| 3行レイアウト | テーマのみ | 子要素は親に対する論理pxのx/y/幅/高さ（式）で置く。上流のClassicテーマが同じ形（見出し、109pxのゲージ、値）を2行で描いているので、3行目は子要素を足すだけ（`src/theme_engine.rs:105-148`、`src/theme_engine/theme_rendering.rs:783-865`、`src/themes/classic-usage-widget.json`） |
| ブロック分割ゲージ（10px＋1px空白、全幅109px） | テーマのみ | 既存の`Progress`要素に`segments: 10`、`segment_gap: "1"`、`width: "109"`、`corner_radius: "0"`を指定する。上流テーマの幅は`ceil(10 / max(1, providers.count)) * 11 - 1`（Claudeだけ有効なら109）だが、ほかのプロバイダを有効にするとブロック数が変わるので、**10と109を直接書く**（`src/theme_engine.rs:290-307`、`theme_rendering.rs:1016-1042`） |
| 端数の部分塗り（1px＝1%） | テーマのみ（100% DPI）。125%と150%は§5.3 | 上流の塗りは、空白を含むゲージ全体の長さに比例する（`theme_rendering.rs:1584-1585`）。使用率をそのまま渡すと、0〜100の整数101通りのうち45通りで1px足りない（例：11%は2ブロック目が0pxになる）。`value`の式で補正すれば全通り一致する（§5.2） |
| 使用率を0〜100にクランプして描画 | 上流実装済み | 描画時に`(value/100).clamp(0,1)`している（`theme_rendering.rs:1031`）。補正式の中でも`clamp`を使う |
| 色の固定指定 | テーマのみ | `#RRGGBB`と`#RRGGBBAA`の書式に対応（`src/theme_engine/theme_expression.rs:470-488`）。エラーメッセージ（`validate_paint`）は`#AARRGGBB`と書いているが、実際のパーサは`#RRGGBBAA`なので注意 |
| 未使用色＝`#DE7356`の不透明度25%相当 | テーマのみ | トラックの色を`#DE735640`にする。ウィジェットはピクセル単位のアルファ付きでタスクバーに重なる（`UpdateLayeredWindow`の`ULW_ALPHA`、`src/window/positioning.rs:160-234`）。このため**実際のタスクバー背景と自動で合成され、ライト用とダーク用の色を分ける必要がない**。条件は、ルートの`background`を`none`にし、`placement.nest`に`"taskbar"`を明示すること。タスクバーから外れて浮動表示になったときは、上流のカード背景の上に描かれる |
| 見出し文字列 | テーマのみ | テンプレート中の`{}`以外の文字はそのまま表示される（`theme_rendering.rs:20-35`） |
| 3つ目のデータ系列（Fable）の割り当て | テーマのみ（取得とバインディングは上流実装済み） | `claude.limits.weekly_scoped_fable.{available, percentage, reset.unix, reset.seconds}`。キー名は§3の確認待ち。代わりのキーとして、`claude.model.fable.*`（同じモデルの枠が1つだけのとき）と`claude.scoped.*`（有効な枠が1つだけのとき）もある（`theme_limits.rs:5-40`）。取得からテーマまで通しで確かめるテストもある（`claude.rs:1160-1203`） |
| 使用済%の表示（`17%`） | テーマのみ | `{claude.session:usage_badge}`のように書く（weeklyとFableも同様）。値があれば`NN%`、Fable枠がなければ`--`、エラー時は`!`になる（`theme_expression.rs:490-579`）。`.display`を付けると「残量表示」の設定に従ってしまうので付けない |
| Fable枠がないとき`--`と未使用色のゲージ | テーマのみ | 存在しない枠のフィールドは既定値（`available=0`、`percentage=0`、`reset.*=0`）を返し、検証エラーにもならない（`theme_limits.rs:137-157`、`src/theme_engine.rs:1759-1764`）。ゲージは空になり、`usage_badge`は`--`を返す |
| カウントダウンの単位切り替え（`4d`→`23h`→`59m`→`59s`） | テーマのみ（式を組み合わせる） | 上流の`duration_short`も最大単位で表示するが、接尾辞が翻訳されるため、日本語環境では`4日`、`23時間`、`今`になる（`theme_expression.rs:589-605`、`src/localization/locales/ja.toml:45-49`）。そこで`floor`と`if`で数値と接尾辞を組み立てる（§5.4） |
| リセット時刻不明のとき`--` | テーマのみ | 時刻が不明だと`reset.unix`は0になる（`src/theme_engine.rs:1688-1698`）。`reset.unix <= 0`で分岐する |
| 毎秒の再描画は残り1分未満のときだけ | 上流実装済み | `schedule_countdown_timer`が、全セクション（`limits`を含む）の`time_until_display_change`の最小値で次の再描画を予約している（`src/window.rs:2782-2820`、`src/models.rs:92-97`）。テーマ側の対応は不要。ただし、テーマJSONに文字列`time.now`、`time.local`、`time.utc`を入れると時計用のタイマーが別に動くので入れない（`src/theme_engine.rs:2430-2443`） |
| 論理pxで設計し、DPIに合わせて拡大 | 上流実装済み | 要素ごとに`round(論理px × 倍率)`で描画する（`theme_rendering.rs:252-268`、`308-309`）。式からDPI倍率は読めない |
| Windows 11の48pxタスクバーに収める | テーマのみ | 高さを`"46"`にする（上流の組み込みテーマと同じ）。48pxのタスクバーで倍率1.0、1.25、2.0のときに退避しないことが上流のテストで確かめられている（`src/window/placement_tests.rs:17-77`）。14px×3行＝42pxなので46に収まる。タスクバーから1pxでもはみ出すと浮動表示に退避する（`positioning.rs:1014-1031`） |
| 文字10〜11px | テーマのみ | `font_size`は論理pxで、下限はない。1行で描かれ、はみ出すと`…`で省略される（`theme_rendering.rs:1263-1268`）。そのため、各テキスト枠は最も長い文字列（`100%`、`ALL5H`）に合わせて幅を取る |
| stale（取得失敗で前回値を保持） | 上流どおり | 上流の組み込みテーマには**staleを示す見た目の印がなく**、前回値を通常どおり表示する。必要なら`claude.stale`を参照できる（`src/theme_engine.rs:1645-1650`）。Claudeだけを使う構成では、stale中もカウントダウンは止まらない（`src/poller/accounts.rs:242-245`、`src/window.rs:2604-2605`） |
| 認証エラー・未ログイン | 上流どおり | %欄は`!`（上流の表記）、ゲージは空、カウントダウンは`--`になる。テーマの式では`claude.has_error`を使わないこと。この値はデータがないときに定義されず、起動時のテーマ検証で失敗するため（`src/theme_engine.rs:1512`、`1563`、`2373`、`src/theme_engine/theme_storage.rs:287-312`） |
| 組み込みテーマとして登録 | **要コード（小）** | §7.2 |
| Claude以外のプロバイダを設定で無効にする | 上流実装済み | 既定で有効なのはClaudeだけ（`src/providers.rs`の`default_enabled`）。有効なプロバイダだけが取得される（`src/poller/accounts.rs:90`） |

### 5.2 ゲージの塗り（補正式）

上流は、ゲージの左端から`along`番目のピクセル（空白を含めて数える）を`(along + 1) / 109 <= value / 100`のとき塗る。空白は塗らない。

使用率p（整数）に対して、塗るべき幅は「ブロック内のp px＋その間にある空白`floor(p/10)`個」なので、`value`を次のように変換すれば、どのpでもブロック内のちょうどp pxが塗られる（101通りすべてで一致することを確認済み）。

```
p = round(clamp(<binding>.percentage, 0, 100))
value = (p + floor(p / 10)) / 109 * 100 + 0.001
```

式の中で変数は使えないので、pの部分は毎回そのまま書き下す。API側の使用率には小数があり得る。端数を四捨五入にするか切り捨てにするかは§6で判断してほしい。

### 5.3 DPI 125%と150%のゲージ

上流の`SegmentLayout`は、すべてのブロックと空白の幅を物理pxでそろえる。空白は四捨五入、ブロックは切り捨てで決まる（`theme_rendering.rs:1611-1636`、テスト`src/theme_engine/tests.rs:1145-1180`）。

| 倍率 | ブロック | 空白 | 描画幅／枠の幅 |
|---|---|---|---|
| 100% | 10px | 1px | 109／109（厳密） |
| 125% | 12px | 1px | 129／136（右端が7px短い） |
| 150% | 14px | 2px | 158／164（右端が6px短い） |

125%では1ブロックが12.5pxであるべきだが、整数pxでは描けない。上流は「すべてのブロックを同じ幅にする」ほうを選んでいる。これより厳密にするには描画コードの変更が必要で、ほかの組み込みテーマの見た目も変わる（変えないようにするには、新しい設定項目で切り替える必要がある）。

### 5.4 カウントダウンの式（案、Phase 2で検証する）

```
{if(R.unix <= 0, "--", if(R.seconds >= 86400, floor(R.seconds / 86400), if(R.seconds >= 3600, floor(R.seconds / 3600), if(R.seconds >= 60, floor(R.seconds / 60), floor(R.seconds / 1))))):0}{if(R.unix <= 0, "", if(R.seconds >= 86400, "d", if(R.seconds >= 3600, "h", if(R.seconds >= 60, "m", "s"))))}
```

`R`の部分には`claude.session.reset`、`claude.weekly.reset`、`claude.limits.weekly_scoped_fable.reset`を書く。

- `{…}`は最初の`}`で閉じる。書式（`:0`）は、引用符の外にある最後の`:`で分けられる（`theme_rendering.rs:36-41`、`theme_expression.rs:43-65`）。
- 切り捨てで単位を決めるので、`time_until_display_change`（`as_secs()`で切り捨て）と切り替わりの時刻がそろう。
- 式の評価に失敗した場合は、上流の仕様で`--`が表示される（`theme_rendering.rs:59`）。

---

## 6. 判断してほしい点

### 6.1 仕様どおりにできない点、または仕様に書かれていない点

| # | 論点 | 選択肢 | 推奨 |
|---|---|---|---|
| D1 | DPI 125%と150%で「1論理px＝1%」が厳密に成り立たない（§5.3） | (a) 上流どおり整数pxにそろえるのを受け入れる（テーマのみで済む）　(b) 描画コードを拡張する（新しい設定項目で切り替え） | (a) |
| D2 | PLAN Phase 2の手順3「塗り幅計算を純粋関数に切り出して単体テスト」。塗りはテーマの式で書けるので、本体には切り出す関数がない | (a) 新しいテーマを指定の使用率（0、1、10、37、99、100、100超、負値）で実際に描画し、塗られたピクセルを確かめるテストを書く（本体コードは変えない）　(b) 補正計算をRustの関数として追加する | (a)。ただしこのテストはwineで動かないため、Windowsで`cargo test`を実行してもらう必要がある |
| D3 | PLAN §7「既定値は定数として1か所にまとめる」。テーマJSONには変数がなく（`src/theme_engine.rs:83-98`）、`#DE7356`と`40`（25%）は各要素に繰り返し書くことになる | (a) 新しいテーマファイル1つに収まっていることをもって「1か所」とみなす　(b) コードを変更する | (a) |
| D4 | 組み込みテーマにする場合、上流のテスト`extra_limits_do_not_change_builtin_rendering_or_headlines`（`src/theme_engine/theme_limits.rs:329-367`）と衝突する。このテストは「追加の枠があっても組み込みテーマの描画は変わらない」ことを確かめているが、Fableを表示する新しいテーマはこれに反する（`USER_GUIDE.md:112-114`にも同じことが書かれている） | (a) 組み込みテーマにし、このテストの対象から新しいテーマだけを外す　(b) 組み込みにせず、Theme Studioで取り込むJSONとして配布する（コード変更なし） | (a)（PLANの「組み込みテーマとして」に従う） |
| D5 | 見出しと%の文字色が仕様にない（指定はカウントダウンの`#DE7356`だけ）。`#DE7356`はライトのタスクバー上だとコントラストが約2.8:1と低い | (a) 見出しは上流のラベル色（ダーク`#888888`、ライト`#404040`）、%は`#DE7356`　(b) すべて`#DE7356`　(c) 別途指定する | (a) |
| D6 | カウントダウンの接尾辞 | (a) Windowsの言語に関係なくASCIIの`d/h/m/s`（§5.4の式）　(b) 上流の`duration_short`（日本語環境では`日/時間/分/秒`） | (a)（仕様の表記どおり） |
| D7 | リセット時刻を過ぎてから再取得（5秒ごと）までの間の表示 | `0s`／`--` | `0s` |
| D8 | 使用率が小数で返った場合 | ゲージと%表示の端数をどちらも四捨五入にする（%表示は上流の書式`{:.0}`のまま） | 四捨五入（ゲージと%表示がそろう） |

### 6.2 推奨案のまま進めてよいか確認したい点

- **E1**：PLAN Phase 1の手順1と2（`UsageResponse`へのフィールド追加と`UsageData`へのOptionセクション追加）はやらない。上流の`limits`をそのまま使う。
- **E2**：テーマの高さは`"46"`にする（上流と同じ。48pxのタスクバーで退避しないことはテスト済み）。Windows 10（40px）では浮動表示に退避するが、PLANの「動作すればよい」に当てはまるとみなす。
- **E3**：stale時は上流どおり見た目の印を付けない。認証エラー時は%欄が`!`（上流の表記）、カウントダウンは`--`。
- **E4**：%とゲージは使用済みの量だけを示し、アプリの「残量表示」設定には従わない（`.percentage`を参照する）。
- **E5**：1行目と2行目で、片方の枠だけがレスポンスにない場合は`0%`と表示される（上流はプロバイダ単位で有無を判定している）。仕様は`--`をFable行についてしか定めていないので、このままにする。
- **E6**：新しいテーマは既定のテーマにはせず、テーマギャラリーから選んでもらう。既定にするなら`theme_storage.rs:323`の1行と、WindowsのVMテストの修正が要る。
- **E7**：トレイアイコンは、上流Classicテーマの`tray_icon`定義を新しいテーマにコピーして引き継ぐ。コピーしなければ、アプリの標準アイコンが表示される（`src/window.rs:1050`）。
- **E8**：`--diagnose`には、取得に成功した直後の`limits`の値だけを出す（`src/poller/accounts.rs:204-218`）。staleで持ち越した値まで出すには、`src/window.rs`にもログの追加が要る。
- **E9**：ブランチは`claude/gallant-ptolemy-xn9prw`を使い続ける（§1）。

---

## 7. Phase 1とPhase 2の変更対象ファイル

### 7.1 Phase 1（取得層）

| ファイル | 変更内容 |
|---|---|
| `src/poller/accounts.rs` | 取得成功時の`--diagnose`ログ（204-218行）に、`limits`の`key`、`percentage`、リセット時刻を加える。`scope`とトークンは出さない |
| `src/poller/claude/limits.rs` | テストを追加する：Fable枠あり、Fable枠なし、`resets_at: null`、`percent`と`utilization`の両方がある要素。§3の結果が上流の前提と違う場合だけ、パース処理を最小限直す |
| `src/poller/claude.rs` | 実レスポンス（アカウントを特定できる値は除く）を元にしたフィクスチャで、`usage_from_response`を通すテストを加える。`UsageResponse`は変えない見込み |
| `src/models.rs` | 変更しない（`UsageData.limits`と`UsageLimit`がすでにある） |

### 7.2 Phase 2（描画）

| ファイル | 変更内容 |
|---|---|
| `src/themes/<新テーマ>.json`（新規） | 3行ブロックゲージのテーマ本体。`nest: "taskbar"`、ルートの背景は`none`、高さ`"46"`、右クリックで`show_context_menu("classic-v1")`、`claude.has_error`と`time.*`は使わない |
| `build.rs` | `build_themes`のファイル名リスト（43-48行）に追加する |
| `src/theme_engine.rs` | テーマIDの定数を追加し、`BUILTIN_THEME_SOURCES`の末尾に加える（先頭はClassicのままにする。`starter()`が先頭を使うため） |
| `src/theme_engine/tests.rs` | `BUILTIN_THEME_SOURCES.len() == 3`と並び順の確認（1792-1795行）を直す。D2が(a)なら、ゲージの描画テストをここに追加する |
| `src/theme_engine/theme_limits.rs` | D4が(a)なら、`extra_limits_do_not_change_builtin_rendering_or_headlines`の対象から新しいテーマを外す |
| `src/window/placement_tests.rs` | 任意：48pxのタスクバーで退避しないことを確かめる対象に新しいテーマを加える |
| `src/theme_engine/theme_rendering.rs` | D1が(b)のときだけ変更する |

組み込みテーマはアプリ起動時に検証され（`ensure_starter_theme`、`src/theme_engine/theme_storage.rs:287-312`）、テーマJSONに誤りがあると**アプリが起動しなくなる**。Phase 2では、Windows上で`cargo test`を流してから実機で確認する手順にしたい。

---

## 8. Phase 0以降に確認できたこと（2026-10-13更新）

Phase 1・2のコミットは`2eee8ff`、`22f778b`、`4c58a48`、`745eb89`。テーマの表示名は「Claude + Fable Block Gauge」（IDは`claude-block-gauge`）。

### 8.1 確認済み

| 項目 | 結果 |
|---|---|
| 手順2：Fable枠の構造 | §3.3の方法A（`usage-cache.json`）で確認した。`limits`は配列で、Fable枠は`kind: "weekly_scoped"`、`scope.model.display_name: "Fable"`、`scope.model.id: null`。キーは想定どおり`weekly_scoped_fable`になる。`scope`には`surface: null`もあった。`is_active`はfalseなので、`claude.scoped.*`では参照できない。使用率には小数があり得る（例：13.6170508）。リセット時刻は秒単位で、パースに成功している。ほかに`session`、`weekly_all`と、未知の種類の枠が1つあった |
| Windows 11の実機での表示 | ユーザーが実機で確認し、問題なしと報告した。確認項目はPLAN Phase 2の4（DPI 100%・125%・150%、ライトとダークのタスクバー、カウントダウンの単位切り替え、Fable枠なし・stale・認証エラーの表示、Explorer再起動後の復帰）と、`--diagnose`のログ |
| 実機のSegoe UIでの文字幅 | 上の実機確認で、見出し・%・カウントダウンの表示に問題は報告されていない |
| 上流issueの実例（CodeZeno/Claude-Code-Usage-Monitor#56、CodeZeno/Claude-Code-Usage-Monitor#61） | 実データでキーを確認できたため、参照の必要がなくなった |

### 8.2 まだ確認していないこと

- 生レスポンスの元のフィールド名（`percent`か`utilization`か）と、`resets_at`の元の書式。キャッシュにはパース後の値しか残らないため。ただし、現在のパーサで読めていることは確認できている
- MSVCターゲットでのビルドと`cargo test`（上流CIの構成）。こちらではGNUターゲットでのみビルドとテストを実行した
- タスクバーのロックが無効な状態で、アプリボタンがウィジェットの位置まで並んだときの挙動。上流のテーマと同じ仕組みで浮動表示に退避するはずだが、このテーマの幅（222論理px）では個別に確かめていない

---

## 付録：この環境での検証手順（Linuxコンテナ、次のセッション向け）

コンテナは使い捨てなので、次のセッションでは以下をやり直す必要がある。`$SP`は作業用ディレクトリ（リポジトリの外）を指す。

```bash
# Windows向けのRustターゲットとmingw-w64
rustup target add x86_64-pc-windows-gnu
apt-get install -y mingw-w64 wine64

# #[link(name = "Advapi32")] を大文字小文字を区別するmingwで解決させる
mkdir -p $SP/mingw-case-libs
ln -sf /usr/x86_64-w64-mingw32/lib/libadvapi32.a $SP/mingw-case-libs/libAdvapi32.a
export RUSTFLAGS="-L $SP/mingw-case-libs"

cargo build --release --locked --target x86_64-pc-windows-gnu
cargo test --no-run --locked --target x86_64-pc-windows-gnu

# wineにないwinsqlite3.dllの代わりを、libsqlite3-sysに同梱のSQLiteソースからビルドする
#（libsqlite3-sys 0.30を features=["bundled"] で使う使い捨てのcargoプロジェクトで cargo fetch しておく）
mkdir -p $SP/winsqlite
x86_64-w64-mingw32-gcc -O1 -shared '-DSQLITE_API=__declspec(dllexport)' \
  -o $SP/winsqlite/winsqlite3.dll ~/.cargo/registry/src/*/libsqlite3-sys-0.30.1/sqlite3/sqlite3.c

# テストは1件ずつ別プロセスで実行する（1件が未実装APIで中断すると、プロセス全体が止まるため）
export WINEPREFIX=$SP/wineprefix WINEDEBUG=-all WINEPATH="Z:$SP/winsqlite"
EXE=$(ls -t target/x86_64-pc-windows-gnu/debug/deps/claude_code_usage_monitor-*.exe | head -1)
/usr/lib/wine/wine64 "$EXE" --exact 'poller::claude::limits::tests::malformed_entries_do_not_hide_valid_quotas'
```
