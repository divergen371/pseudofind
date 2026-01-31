# 実装方針（OCaml版からGo版への再実装）

## 目的
- OCaml版 `quasifind` の主要機能・挙動をGoで再実装し、同等のCLI体験と性能指向を維持する。
- 既存のOCaml資料を正として、互換性・差分を明確にしながら段階的に実装する。

## 参照ソース（OCaml版）
- `OCAML_README.md`: 仕様・CLI・機能一覧
- `OCAML_IMPLEMENTATION_PLAN.md`: 最適化方針
- `OCAML_WALKTHROUGH.md`: 実装詳細と履歴
- `OCAML_TASK.md`: 過去のタスク履歴

## 仕様ソース（Go版）
- `docs/SPEC.md`: OCaml版ソースに基づく詳細仕様（実装時の正）

## 主要機能（OCaml版準拠の一覧）
- CLI: `quasifind [DIR] "EXPR" [OPTIONS]`
- サブコマンド: `history`（履歴一覧/選択）, `history --exec`（選択コマンド出力）
- 検索DSL: 型付き式、単位（サイズ/時間）、正規表現
- 走査オプション: `--hidden/-H`, `--max-depth/-d`, `--follow/-L`, `--jobs/-j`
- 実行: `-x`（各ファイル）, `-X`（バッチ）
- 高度機能: `content`, `entropy`, `suid`, `sgid`, `--check-ghost`, `--suspicious`
- 設定: `config.json`（ignore, fuzzy_finder, 通知設定など）
- ルール更新: `--update-rules` / `rules.json`
- 監視: `--watch`, `--interval`, 通知（email/webhook/slack）

## 互換性方針
- **優先順位**: 1) CLI互換性 2) DSL互換性 3) 挙動（隠し/除外/深さ/シンボリックリンク） 4) 性能
- **差分の許容**: Goの標準ライブラリ制約により、正規表現やファイル種別判定の挙動差が出る可能性を明文化する。
- **.gitignore**: OCaml版同様に「静的なignoreリストのみ」採用（階層的な.gitignore読み取りは非対応）。

## スコープ（段階的実装）
### Phase 1: MVP（機能核）
- CLI骨格（`quasifind [DIR] "EXPR" [OPTIONS]`）
- DSL: パーサ・型チェック・評価
  - フィールド: `name`, `path`, `type`, `size`, `mtime`, `perm`
  - 演算子: `==`, `!=`, `<`, `<=`, `>`, `>=`, `=~`
  - 論理: `&&`, `||`, `!`
  - 単位: `B/KB/MB/GB/KiB/MiB/GiB`, `s/m/h/d`
- 走査: DFS（単一スレッド）＋基本オプション
  - `-d/--max-depth`, `-H/--hidden`, `-L/--follow`, `-E/--exclude`
- 出力: パスの列挙＋`-x`/`-X` 実行
- `history` サブコマンド（最低限の履歴保存と一覧）

### Phase 2: パフォーマンス向上
- 並列走査（ワーカープール）
- Lazy stat（メタデータ不要なら `lstat` を省略）
- 事前フィルタ（`name` 条件を先に適用して `stat` を削減）
- Ignore/Exclude の正規表現を事前コンパイル

### Phase 3: 高度機能
- `content` / `entropy` / `suid` / `sgid` の評価
- `--check-ghost`（削除済みファイル検出）
- `--suspicious`（ルールベース探索）
- `--update-rules` / `rules.json` / `config.json` 読み書き

### Phase 4: 監視・通知
- `--watch` / `--interval`
- `--email` / `--webhook` / `--slack`
- 履歴 / プロファイル / ファジー検索

## 実装上の重点（OCaml版からの移植観点）
### 1. 走査・並列化
- OCaml版の「ワークスティーリング」思想はGoのワーカープールに置換。
- まずは固定ワーカー + キューで安定化、その後に分割戦略を最適化。

### 2. ファイル種別と `stat` 最小化
- Goの `os.ReadDir` / `DirEntry.Type()` を活用し、可能な限り `stat` 回数を削減。
- `DirEntry.Type()` が不明な場合は `Info()` で補完（最小限に抑える）。
- `name` 条件がある場合は **先にフィルタ** を適用。

### 3. 正規表現の互換性
- OCaml版はPCRE互換（`re`）だが、GoはRE2。
- 先読み等の一部構文が使えないため、**DSLの正規表現互換性は制限**とする。
- READMEに「使用可能/不可の構文」ガイドを記載する。

### 4. DSL・評価設計
- `parser` / `typecheck` / `eval` を明確に分離し、テストを容易にする。
- ルールJSONの `expr` は同じDSLパーサで解釈する。
- `content`/`entropy` などコスト高い評価は「必要なときだけ計算」する。

### 5. 設定・履歴
- `config.json` は初回起動時にデフォルト生成（OCaml版に合わせる）。
- `XDG_CONFIG_HOME/quasifind/config.json`（デフォルト: `~/.config/quasifind/config.json`）
- `fuzzy_finder`: `auto` / `fzf` / `builtin`（選択ロジックの優先順位を明文化）
- `ignore`: Globパターンの静的除外リスト
- 履歴はJSONLなど単純な追記形式で実装し、`history` で一覧・`--exec` で出力。

## 検証方針
- パーサ/評価/走査のユニットテストを先行して整備。
- OCaml版と同じ入力ケースで比較する回帰テストを用意。

## 成果物
- `cmd/quasifind`（CLI）
- `internal/ast`, `internal/parser`, `internal/eval`, `internal/traversal`
- `internal/history`, `internal/config`, `internal/rules`
- `docs/`（本方針 + タスク管理）
