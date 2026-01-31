# タスク一覧（Go版再実装）

## 0. 事前準備
- [ ] ディレクトリ構成の確定（`cmd/`, `internal/`, `docs/`）
- [ ] 最低限のCLI実行（`quasifind --help`）

## 1. DSL（パーサ/型チェック/評価）
- [ ] AST定義（式・演算子・値・単位）
- [ ] パーサ実装（トークナイザ含む）
- [ ] 数値リテラル（10進/16進/8進/2進, 浮動小数）
- [ ] 文字列リテラル（エスケープ対応）
- [ ] 正規表現リテラル `/.../`（`\/` エスケープ）
- [ ] サイズ単位（B/KB/MB/GB/KiB/MiB/GiB）
- [ ] 時間単位（s/m/h/d）と `mtime` の「経過秒」評価
- [ ] ファイル種別エイリアス（file/f, dir/d, symlink/l）
- [ ] 型チェック（不正な比較の検出）
- [ ] 評価器（ファイル情報に対する判定）
- [ ] DSLの単体テスト

## 2. 走査（DFS）
- [ ] ディレクトリ走査の基盤
- [ ] `-d/--max-depth` 実装
- [ ] `-H/--hidden` 実装
- [ ] `-L/--follow` 実装
- [ ] `-E/--exclude` 実装（glob → regex 変換）
- [ ] `path == "..."` の起点最適化
- [ ] `name` 条件の先行フィルタ（stat前の剪定）
- [ ] Lazy stat（size/mtime/perm不要時はstat省略）
- [ ] 走査の単体/結合テスト

## 3. CLI・実行
- [ ] `quasifind [DIR] "EXPR"` の実装
- [ ] `-x/--exec` と `-X/--exec-batch`
- [ ] `{}` プレースホルダ置換（未指定時は末尾追加）
- [ ] すべてのパスをシェル安全にクオート
- [ ] 出力整形・終了コードの方針決定
- [ ] `history` サブコマンド（一覧）
- [ ] `history --exec` / `-e`（選択結果の出力）
- [ ] `history preview`（保存済み results の表示）
- [ ] `--save-profile` / `-p`（プロファイル保存・読込）

## 4. パフォーマンス
- [ ] ワーカープール実装（並列走査）
- [ ] Lazy stat（必要時のみ `lstat`）
- [ ] `name` 先行フィルタ
- [ ] Ignore/Exclude の事前コンパイル

## 5. 高度機能
- [ ] `content` / `entropy` の実装
- [ ] `--check-ghost` の実装
- [ ] `--suspicious` と `rules.json`
- [ ] `--update-rules`（外部リストの取り込みと自動変換: extensions/filenames）
- [ ] default suspicious ルール（隠し実行/777/SUID/巨大tmp等）
- [ ] `--stealth`（プロセス名偽装）
- [ ] `--stealth` 時の atime/mtime 復元（content/entropy）

## 6. 設定・履歴・監視
- [ ] `config.json` 読み書き（初回デフォルト生成）
- [ ] `fuzzy_finder` 設定（`auto`/`fzf`/`builtin`）
- [ ] `ignore`（Globパターンの静的除外）
- [ ] `rules.json` の初回デフォルト生成・reset
- [ ] `history.jsonl`（XDG_DATA_HOME）と results 保存
- [ ] `--watch` / `--interval`
- [ ] 通知（email/webhook/slack）
- [ ] 履歴 / プロファイル

## 7. テスト・ベンチ
- [ ] パーサ/評価/走査のユニットテスト
- [ ] 互換性テスト（OCaml版と同条件のケース）
- [ ] ベンチマーク（`find`/`fd` 比較）

## 8. ドキュメント
- [ ] README（Go版の仕様・差分）
- [ ] 互換性の注意点（正規表現差分など）
