# 仕様詳細（OCaml版ソースコード準拠）

このドキュメントは、OCaml版の実装から抽出した「実際の挙動」をGo版に落とし込むための仕様メモです。
README等の説明にない細部はここを正とします。

## DSL（構文・リテラル）
### トークン
- 識別子: `[A-Za-z_][A-Za-z0-9_]*`
- 文字列: `"..."`（`\\n`, `\\t`, `\\r`, `\\` のエスケープ対応）
- 正規表現リテラル: `/.../`（`\/` をエスケープ）

### 数値
- 整数: 10進に加えて `0x`(16進) / `0o`(8進) / `0b`(2進)
- 浮動小数: `123.45` 形式（指数表記なし）

### 単位
- サイズ: `B`, `KB`, `MB`, `GB`, `KiB`, `MiB`, `GiB`
- 時間: `s`, `m`, `h`, `d`

### ファイル種別
- `file` / `f`, `dir` / `d`, `symlink` / `l`

### 演算子
- 比較: `==`, `!=`, `<`, `<=`, `>`, `>=`, `=~`
- 論理: `!`, `&&`, `||`

## DSL（型と比較の許可）
### 文字列系
- `name`, `path`, `content`: `==`, `!=`, `=~`

### 数値系
- `size`, `mtime`, `entropy`, `perm`: `==`, `!=`, `<`, `<=`, `>`, `>=`
- `=~` は不可

### 種別
- `type`: `==`, `!=` のみ

### 正規表現
- OCaml版はPCRE (`re`/`Re.Pcre`) を使用
- Go版はRE2のため、先読み等の非互換は「仕様差分」として明示

## 正規化ルール
- `size`:
  - 単位指定はバイトに正規化
  - 単位なしの整数は「バイト」として扱う
- `mtime`:
  - `age = now - mtime`（更新からの経過秒）を比較対象とする
  - 単位なしの整数は「秒」として扱う

## 走査・評価の挙動
### hidden / ignore
- `--hidden` が無い場合、先頭が`.`の名前は探索から除外（剪定）
- `ignore`（config + `-E/--exclude`）は Glob として扱い、正規表現にコンパイルして判定

### max depth
- `max_depth` は「現在深さ >= max_depth」で停止（深さはルート=0）

### symlink
- `--follow` 時のみディレクトリシンボリックリンクを再帰

### Lazy stat
- 解析で `size/mtime/perm` が不要なら `lstat` を省略
- `type` は `readdir` の種類情報で評価（不明時のみ `stat`）

## 走査の最適化（仕様として扱う部分）
- `path == "..."`
  - 先頭一致ではなく「開始パス」として解釈し、探索起点に適用
- `name` 条件
  - `name` のみで判定できる場合は **stat前** にフィルタし、明らかに一致しないエントリは `stat` を回避

## content / entropy
- ファイル内容読み取りは失敗時 `false`
- `--stealth` の場合は読み取り後に `atime/mtime` を復元

## 実行オプション（-x / -X）
- `{}` があれば置換、無ければ末尾にパスを追加
- パスはシェル安全にクオート
- 実行は `sh -c` で行う（バッチは全パスを空白結合して渡す）

## 設定ファイル
- パス: `XDG_CONFIG_HOME/quasifind/config.json`（無ければ `~/.config/quasifind/config.json`）
- 初回起動時にデフォルトを自動生成
- フィールド:
  - `fuzzy_finder`: `auto` / `fzf` / `builtin`
  - `ignore`: glob配列
  - `email`, `webhook_url`, `slack_url`
  - `rule_sources`: `{name,url,kind}`（`extensions` / `filenames`）
- `--reset-config` でデフォルトに戻す

## ルール（suspicious / update-rules）
### rules.json
- パス: `XDG_CONFIG_HOME/quasifind/rules.json`（無ければ `~/.config/quasifind/rules.json`）
- 初回起動時にデフォルトを自動生成
- 形式: `{version: "1.0", rules: [{name, expr}...] }`

### デフォルト suspicious ルール
- 隠し実行ファイル: `^\..*\.(sh|py|exe)$`
- 危険権限: `perm == 0o777`
- SUID: `perm >= 0o4000`
- `/tmp` かつ `size > 100MB`
- 拡張子: `.(payload|backdoor|exploit)`
- base64風の名前: `^[A-Za-z0-9+/=]{30,}\.[a-z]+$`

### update-rules
- `config.rule_sources` を `curl` で取得
- 取得内容を `extensions` / `filenames` へ変換し `name =~ /.../` のルールに
- 既存の `Generated:` ルールは置き換え

## Ghostファイル検出
- `lsof +L1` を実行し、末尾列（ファイル名）を抽出
- `root` 配下にあるパスのみ表示
- `lsof` が失敗した場合は警告してスキップ

## 履歴
- パス: `XDG_DATA_HOME/quasifind/history.jsonl`（無ければ `~/.local/share/quasifind/history.jsonl`）
- 1行1JSON:
  - `timestamp` (float)
  - `command` (string list)
  - `results_count` (int)
  - `results_sample` (最大5件)
  - `full_results_path` (results/ 配下のテキスト; 0件はnull)
- `history preview` は保存済み results を表示

## インタラクティブ選択（history --exec）
- TTY必須（非TTYなら選択不可）
- `fuzzy_finder=auto` は fzf があれば使用、無ければ内蔵TUI
- `fuzzy_finder=fzf` で fzf 強制、未インストール時は内蔵TUIへフォールバック

## プロファイル
- パス: `XDG_CONFIG_HOME/quasifind/profiles/*.json`
- 保存項目: `root_dir`, `expr`, `max_depth`, `follow_symlinks`, `include_hidden`, `exclude`

## Watchモード
- ポーリング方式
- 差分検出: 新規 / 変更 / 削除
- `--log` でファイル出力
- 通知: webhook (curl), email (mail), slack (webhook)

## Stealth
- プロセス名の偽装（macOS: `pthread_setname_np`, Linux: `prctl`）
- `--stealth` 時は `content/entropy` で `atime/mtime` を復元
