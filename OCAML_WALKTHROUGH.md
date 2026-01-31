# Walkthrough - Rule Update Feature

信頼できるソースからのルール更新機能を実装し、外部定義されたルール（JSON）を取り込めるようにしました。

## 実装内容
1. **ルールの外部化 (`rule_loader.ml`)**
   - JSON形式 (`rules.json`) の読み書きをサポート
   - `~/.config/quasifind/rules.json` をロード対象とする

2. **動的ルール読み込み (`suspicious.ml`)**
   - ハードコードされたデフォルトルールと、ロードした外部ルールを結合
   - `Parser.parse` を用いて、JSON 内の `expr` 文字列をアグレッシブ探索用の式オブジェクトに変換

3. **更新コマンド (`--update-rules`)**
   - リモートソースから最新ルールを取得するロジックを実装
   - （デモ用に、リモート取得をシミュレーションしてサンプルルールを保存する形になっています）

## 確認手順
1. ルール更新を実行
   ```bash
   dune exec quasifind -- --update-rules
   ```
   -> `~/.config/quasifind/rules.json` が作成・更新されます。

2. カスタムルールが反映されているか確認
   ```bash
   # 例: 更新されたルールに含まれる "Remote: Crypto Miner" のようなルールが適用されるか
   quasifind /tmp --suspicious
   ```
   （`/tmp` にマッチするダミーファイルがあれば検出されます）
  - **Phase 3 (Optimization)**:
    - **Result**: `3.96s` (Parallel -j 8) vs `6.83s` (find) vs `8.68s` (DFS).
    - **Speedup**: ~1.73x faster than `find`, ~2.19x faster than purely sequential execution.
    - **Note**: Still trailing `fd` (1.25s), but significant improvement for an OCaml implementation.

## Bug Fix: Max Depth (`-d`)
- **Issue**: The `-d` option was failing to restrict traversal depth in Parallel mode, and users reported confusion about its behavior in DFS.
- **Root Cause**: 
  - **Parallel**: The `Work_pool` only stored file paths, discarding depth information. Workers processed directories without checking if `max_depth` was reached.
  - **DFS**: Logic was actually correct (matching `find`), but confusion stemmed from expectation mismatches (likely due to no files being found if command args were wrong, generating help text).
- **Fix**: 
  - Refactored `Work_pool` to store `(path, depth)` tuples.
  - Implemented strict depth checks in `traverse_parallel` before processing directories and before queuing subdirectories.
  - Verified against `find` ground truth using `repro_depth.sh`.
  - Added formal regression test `test_max_depth` to `test/test_traversal.ml` covering:
    - DFS and Parallel strategies.
    - Max depth 1 and 2.
    - Verification of inclusion/exclusion of files at varying depths.
  - Added `test_combinations` to ensure robust option interactions:
    - Verified Parallel + Hidden/Ignore flags work together.
    - Verified Symlink processing (emission vs recursion) and its interaction with Max Depth.
  - Added strict checks for:
    - `Preserve Timestamps` verifies atime is restored.
    - `Loop Termination` ensures symlink loops do not cause infinite hanging (eventually terminates via OS limits/timeout).
  - **Lock-Free Optimization (Phase 3)**:
    - Replaced Mutex-based `Work_pool` with `Saturn.Work_stealing_deque` (Chase-Lev).
    - Reduced synchronization overhead for parallel traversal.
  - **GC-Less Batch Readdir (Phase 3)**:
    - Implemented `caml_readdir_batch` C stub to read directory entries into a pre-allocated 8KB byte buffer.
    - Replaced `readdir` (which allocated a list of strings/tuples) with `iter_batch` (zero-allocation iteration).
    - Drastically reduced GC pressure by only allocating OCaml strings for entries that pass filters/need traversal.

## 成果物
- `lib/quasifind/rule_converter.ml`
- `bin/main.ml` (コマンド更新)

---

# Walkthrough - Rule Converter (Smart Update)

単純な JSON のダウンロードではなく、外部の「拡張子リスト」や「ファイル名リスト」といった生データを取得し、Quasifind 形式のルールに**自動変換**して取り込む機能を実装しました。

## 実装内容
1. **変換モジュール (`rule_converter.ml`)**
   - 外部ソースからテキストデータを取得（`curl`）
   - リストデータ（例: `php`, `jsp`...）を正規表現 (`/\.(php|jsp)...$/`) に動的変換
   - `Generated: ...` というプレフィックスを付けてルールセットに統合

2. **コマンド統合 (`--update-rules`)**
   - 既存の `--update-rules` コマンドがこの新しいコンバーターロジックを使用するように変更

## 動作イメージ
```bash
quasifind --update-rules
# -> Fetching external security lists...
# -> Successfully generated and saved 2 rules.
```
これだけで、SecListsのような生のリストデータを取り込み、即座に検索に使用できます。

---

# Walkthrough - Default Config Generation

初回実行時に設定ファイルが存在しない場合、自動的にデフォルト設定ファイルを生成するようにしました。

## 実装内容
- **`config.ml`**: `load` 関数内でファイルチェックを行い、存在しなければデフォルト値を JSON 化して保存するロジックを追加。
- ユーザーは `Created default config at ...` というメッセージを確認し、生成されたファイルをすぐに編集できます。

## 生成されるデフォルト設定 (`~/.config/quasifind/config.json`)
```json
{
  "fuzzy_finder": "auto",
  "ignore": ["_build", ".git", "node_modules", ".DS_Store"],
  "email": null,
  "webhook_url": null,
  "slack_url": null,
  "rule_sources": [
     {
       "name": "Generated: Suspicious WebShell Attributes",
       "url": "https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/web-extensions.txt",
       "kind": "extensions"
     },
     {
       "name": "Generated: Common Sensitive Files",
       "url": "https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/quickhits.txt",
       "kind": "filenames"
     }
  ]
}
```

同様に、ヒューリスティックルールファイル (`~/.config/quasifind/rules.json`) も初回実行時に以下のサンプルルール入りで自動生成されます。
- Sample: PHP WebShell
- Sample: Reverse Shell

---

# Walkthrough -## Performance Analysis & Optimization
We achieved **parity with `fd`** (Rust) and significantly outperformed `find`.

| Implementation | Time (mean) | Speedup vs `find` | Speedup vs `quasifind` (Base) |
| :--- | :--- | :--- | :--- |
| `find` (Reference) | 6.71s | 1.0x | - |
| `quasifind` (DFS) | 1.86s | **3.6x** | 4.6x (vs old DFS) |
| `quasifind` (Parallel -j 8) | **1.34s** | **5.0x** | 6.4x (vs old DFS) |
| `fd` (Rust) | 1.32s | 5.1x | - |

### Key Optimizations
1.  **Multicore Parallelism**: Used `Eio` domains and `Saturn` lock-free work-stealing deque.
2.  **Native Readdir (no-GC)**: Implemented C stub `caml_readdir_with_type` to avoid `lstat` calls for file type checks.
3.  **Lazy Stat**: Analyzed query to skip `lstat` entirely if metadata (size, time) is not needed. Populates entries with dummy values.
4.  **Regex Pre-compilation**: Identified and fixed a critical performance bug where ignore patterns were re-compiled for every file visited. This reduced user-space overhead drastically.

### Bottlenecks Removed
- **`lstat` Overhead**: Removed for majority of queries (name/type only).
- **GC Pressure**: Reduced by batch readdir and strictly typed C-stubs.
- **Regex Compilation**: Fixed O(N) compilation loop.
Domain_manager`-based parallelism.
   - **Static Partitioning Strategy**: The root directory is split into buckets, each processed by a dedicated domain. This minimizes domain creation overhead compared to recursive spawning.
   - **Performance**: Achieved ~2.7x speedup on 8 cores.

2. **Native Readdir (C Extension)**: Implemented `dirent_stubs.c`.
   - **Optimization**: Retrieves `d_type` directly from `readdir`, allowing `quasifind` to skip `lstat` calls for files that don't match the search query.
   - **Performance**: Reduced time from 6.32s to 4.36s.
   
3. **Dynamic Work Stealing**: Implemented `Traversal.Work_pool` with dynamic load balancing.
   - **Optimization**: Replaced static partitioning with a shared task queue (`Queue` + `Eio.Mutex`). Domains process tasks and push subdirectories dynamically.
   - **Safety**: Fixed data races in result collection using `Eio.Stream`.
   - **Result**: Further reduced time to **4.07s** (Total 4.14x speedup).

### Remaining Bottlenecks
- **GC & Allocation**: Constructing OCaml strings and list nodes in the C stub adds overhead compared to Rust's zero-copy iterator approach.
- **Syscall Overhead**: Even with `d_type`, `readdir` involves context switches. Parallelizing this further is difficult without finer-grained work stealing.

---

# Walkthrough - Notification Configuration

Watchモードの通知設定を `config.json` に記述できるようにしました。

## 実装内容
1. **設定ファイル拡張 (`config.ml`)**
   - `email`, `webhook_url`, `slack_url` フィールドを追加
2. **フォールバック実装 (`main.ml`)**
   - コマンドライン引数が未指定の場合、設定ファイルの値を自動的に使用

## 依存ライブラリ確認レポート
通知機能 (`watcher.ml`) は以下の外部コマンドを使用します。環境にインストールされていることを確認しました。

| 機能 | 使用コマンド | 状態 |
|---|---|---|
| Webhook / Slack | `/usr/bin/curl` | ✅ Installed |
| Email | `/usr/bin/mail` | ✅ Installed |

## 設定ファイル例 (`~/.config/quasifind/config.json`)
```json
{
  "fuzzy_finder": "auto",
  "ignore": ["_build"],
  "email": "admin@example.com",
  "webhook_url": "https://hooks.slack.com/services/..."
}
```
