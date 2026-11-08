# scaluq への改善提案（SKIPPER の実装・測定から）

**対象**: scaluq 0.2.0（PyPI wheel、CPU/OpenMP 版、`scaluq.default.f64`）
**作成日**: 2026-11-06
**環境**: macOS、32 コア、8 スレッド（`OMP_NUM_THREADS=8`）。時間は特記のない限り n=20（状態 16 MiB）、各 10〜20 回の平均。共有マシンのため数十 % のばらつきを含む。

SKIPPER は scaluq の `StateVector` だけを使って、補助量子ビットを持たない状態ベクトル演算（Pauli 和の行列ベクトル積、U のべき、射影など）を組み立てている。その中で見つかった不具合、性能上のボトルネック、あると助かる API を、影響の大きい順にまとめる。各項目に、SKIPPER 側で取った回避策を添える。

---

## CI 障害への対応（2026-11-08）

ユーザー依頼により、`@forest1040` の未マージ PR 全件を確認し、#456・#457・#458・#464・#467 の失敗／キャンセルを対応対象にした。PR は既存の upstream 向け PR を更新し、マージしない。

| PR | 原因 | 対応・検証状況 |
|---|---|---|
| [#456](https://github.com/qulacs/scaluq/pull/456) | macOS arm64 wheel ジョブが実行前にキャンセル（実行ステップなし、理由は公開されていない） | 再実行で同じ JSON HTTP 503 を確認。公式取得先フォールバック `4d017986` を push、CI 再実行中 |
| [#457](https://github.com/qulacs/scaluq/pull/457) | mypy 取得時に `files.pythonhosted.org` の DNS 解決が失敗。別の macOS ジョブは実行前キャンセル | 再実行後、共通の公式 JSON 取得先フォールバック `9a070663` を push、CI 再実行中 |
| [#458](https://github.com/qulacs/scaluq/pull/458) | macOS ジョブが実行前キャンセル。再実行した Clang インストールジョブは nlohmann/json のダウンロードで HTTP 503 | 再試行でも JSON HTTP 503 が継続。公式取得先フォールバック `e9f10fce` を push、CI 再実行中 |
| [#464](https://github.com/qulacs/scaluq/pull/464) | 静的解析スタブに `Kokkos::log` がなく `device_random.hpp` の解析でエラー（書式チェック自体は成功） | スタブを補う `f02d4bdc` を push。対象ヘッダの構文確認成功。再実行では共通の JSON 配布障害を確認し、フォールバック `8be24ace` も push、CI 再実行中 |
| [#467](https://github.com/qulacs/scaluq/pull/467) | CUDA `__half` と整数ゼロの比較が曖昧でコンパイル失敗。後続の HIP・SYCL・macOS ジョブはキャンセル | 通常・バッチの dispatch 2 か所を明示変換する `96bd335f` を push。F16 の関連ソース 2 件を GCC16 でコンパイル成功。再実行では共通の JSON 配布障害を確認し、フォールバック `a3fad2a1` も push。CUDA 等は CI 再実行中 |

修正は各 PR の fork ブランチへ push 済み。本書は差分に含めていない。統合検証ブランチにも取り込み、コード先端は `31a52403`。ローカル再ビルド後、C++ 1052 件・CPU 構成 CTest・PauliRotation の Python 回帰 36 件が成功した。CI の最終結果を確認してこの表を更新する。初回はネットワーク障害・キャンセルの該当ジョブだけを再実行したが、複数 PR・複数ランナーで nlohmann/json の GitHub release 配布が HTTP 503 または HTTP/2 stream refused で継続失敗した。このため共通 CMake 修正 `56ccb0b8` を各 PR に適用した。元の release URL に加え、公式 `codeload.github.com/nlohmann/json/tar.gz/refs/tags/v3.11.3` を代替取得先とし、バージョンは変えていない。CMake の実動作で主 URL 失敗→代替取得・展開・構成成功を検証し、`single_include/nlohmann/json.hpp` の SHA-256 が既存 release と一致することも確認した。統合側の CMake 再構成も成功。

追補（2026-11-08 19:47 JST）: #456・#457・#458・#464・#467 の最新コミットでは失敗チェック 0 件、CI は実行中または runner 待ち（各 22〜23 件未完了）。#464 の CI は Configure 成功後に clang-tidy 実行まで進んだ。ローカルでもキャッシュ済み依存を使い、全 `src/*.cpp` への clang-tidy がエラーなしで終了した。CI 全体の完了を確認済みとは扱わない。

追加課題（優先度 **未設定**）: `CMakeLists.txt` の `clang-tidy` ターゲットにある `sed -i` は BSD sed の macOS では失敗する。Linux CI とは別の既存の移植性課題であり、今回の PR に修正を混ぜない。ローカル検証時は同じ modmap 除去を Perl で行ってから clang-tidy を直接実行し、成功した。

今回の一時 worktree 4 個と `fix/ci-*` ローカルブランチ 6 個は、各 PR のリモートブランチへの push と作業ツリーのクリーンを確認した後に削除した。修正コミットは PR ブランチ・統合ブランチに残り、復元用 bundle も `ref/reboot-artifacts/ci-fixes-2026-11-08.bundle` へ保存・検証した（ignore 対象、未コミット）。


## 作業領域の整理（2026-11-08）

### 追加整理（同日 11:35 JST）

- PR #460〜#462・#465〜#470 のローカルブランチ 9 個を削除。削除前に `origin/<branch>` とコミットが一致することを確認した。PR は未マージのため、fork 上の PR 用ブランチは維持する。必要なら `git worktree add -b <branch> <path> origin/<branch>` で復元できる。
- 旧 D2 草稿の `scaluq-rotation` worktree と `perf/single-pauli-rotation` ブランチも削除。独自コミットと上記 PR 用ローカルブランチは `ref/reboot-artifacts/cleanup-2026-11-08/removed-local-branches.bundle` へ保存し、`git bundle verify` 成功を確認。未コミット差分は同ディレクトリの `rotation-uncommitted.patch` に保存した。バックアップは ignore 対象・ローカルのみで、PR やコミットには含めない。
- 現在の worktree は 5 個: 主作業ディレクトリ、ビルド依存ソースを提供する `scaluq-pauli`、別 AI の F2 作業用 `scaluq-metal`・`scaluq-metal-exp`・`scaluq-metal-py`。後者 2 個には未コミット変更があり、削除・変更していない。ローカルブランチも 5 個（`main`、現行統合検証、依存ソース用、F2 用 2 個）。

### 初回の整理記録

- 不要な worktree 9 個を削除: `scaluq-state`、`scaluq-c1`、`scaluq-grouped`、`scaluq-elementwise`、`scaluq-small-state`、`scaluq-pauli-rotation`、`scaluq-batched-pauli`、`scaluq-batched-algebra`、`scaluq-pauli-full-matrix`。削除前に追跡ファイルの変更・未追跡ファイルがないことと、維持するビルドの依存参照がないことを確認した。各 PR 用ブランチと fork 上のブランチは維持しているため、必要なら `git worktree add <path> <branch>` で再作成できる。
- 旧統合ブランチ `test/high-priority-improvements` と `test/medium-priority-improvements` は、全履歴が現行 `test/batched-operator-improvements` に含まれることを確認して削除。後者は fork 上からも削除した。現行の統合ブランチとその履歴・提案書は維持している。
- C4 完成版で置き換えられた旧 WIP ブランチ `feat/batched-linear-combination`（`05923b64`）も削除。復元用パッチは `ref/reboot-artifacts/superseded-batched-linear-combination.patch` に保存（ignore 対象・ローカルのみ）。
- 維持する worktree: 主作業ディレクトリ、依存ソースを提供する `scaluq-pauli`、未コミット草稿がある `scaluq-rotation`、別 AI の `scaluq-metal` と `scaluq-metal-exp`。PR はすべて未マージで、PR 用ブランチは削除していない。

## OS 再起動後の引き継ぎメモ（2026-11-07 13:28 JST）

- 作業場所: `/Users/tmori/work3/scaluq`。現在のブランチは `test/batched-operator-improvements`。本メモ追記前の HEAD は `91997988`、実装コードの先端は `f22587e5`。それまでの変更は commit・fork への push 済み。本メモも同じ統合検証ブランチへ commit・push する。
- C3 → C4 の依頼は実装・検証・PR 作成まで完了。upstream `qulacs/scaluq` の `main` 宛てに #468・#469 を作成し、マージしていない。他の実装 PR とブランチは下の追跡表を参照する。レビュー・CI 状態は記録時点のため、再開時に GitHub で確認する。
- 検証結果: C++ 1044 件 + CPU 構成 CTest、Python 75 件成功・12 件の環境依存スキップ、文書例 412 件、SKIPPER 全 440 件成功。f16・bf16 も実 Kokkos/OpenMP の個別チェックで重み付き和・内積の 4096 項集計を確認済み。ローカルのビルド・テスト・ベンチマークは終了しており、再実行が必要な未完了プロセスはない。GitHub Actions は OS の再起動に依存せず継続する。
- 次に確認すること: #467 の CUDA Install Library 失敗の原因確認・修正、および #462・#466・#468・#469 の CI・レビュー。修正が必要なら該当する独立 PR 用 worktree で直し、検証後に push し、統合検証ブランチにも取り込む。統合検証ブランチから upstream へ PR を作らない。
- 未対応の追加課題: **A4** `PauliOperator.get_full_matrix()` が係数を無視する件。再現と原因は本書 A4 に記載済み、優先度は未設定。C3・C4 の PR には修正を混ぜていない。他の残タスクは B3、D3・D4、E1〜E3、G1 等。F2 Metal は別 AI が調査・実装し、2026-11-07 にコード（PR #464）と調査結果（`ref/f2_metal_investigation.md`、`ref/f2_metal/`）を本ブランチへ統合した（下の「F2 調査結果」を参照）。
- **本書は統合検証用ブランチだけにコミットする。PR 用ブランチには追加せず、文書更新コミットを cherry-pick しない。** `/ref/` の ignore は維持し、本書を追加するときは対象を明示して `git add -f ref/scaluq_improvements.md`。他の参考資料やバックアップを追加しない。例外として、ユーザーの指示（2026-11-07）により F2 の調査結果 `ref/f2_metal_investigation.md` と `ref/f2_metal/` も本ブランチで追跡する。原本は `research/f2-metal`（PR #463）で、更新は原本から取り込む。PR 作成後・新規課題発見時には本書を更新する。

### 一時環境・ログの復元

`/tmp` は再起動で消える可能性がある。`/tmp/scaluq-test-venv`、`/tmp/scaluq-batched-verified` 等の Python 環境・配置済みモジュールに依存して再開しない。必要なベンチマーク台本と最終検証ログは **`ref/reboot-artifacts/` に退避済み（ignore 対象、未コミット、ローカルディスクのみ）**。PR 本文にも測定条件・結果を残してある。

- 本体の `build/` と worktree は永続ディスク上にある。現行 CMake は依存ソースとして `/Users/tmori/work3/scaluq-pauli/build-make/_deps/{kokkos,eigen,json,googletest}-src` を参照しているため、再開前にこれらの worktree・キャッシュを削除しない。
- Python 環境が消えた場合は、例えば `uv venv build/restart-venv --python 3.12` と `uv pip install --python build/restart-venv/bin/python nanobind numpy scipy pytest scikit-build-core setuptools-scm ninja clang-format` で永続側へ再作成する。CMake の `Python_EXECUTABLE` と必要なら `nanobind_DIR` を新しい環境に更新して再構成する。古い `/tmp` のパスを含む CMake cache をそのまま使わない。
- C++ の確認は `OMP_NUM_THREADS=2 OMP_PROC_BIND=false ./build/tests/scaluq_test`、構成テストは `ctest --test-dir build -R '^ScaluqCPUConfig$' --output-on-failure`。Python は再構成したビルドの拡張と、同じブランチの `python/scaluq` を組み合わせて使う。古い PyPI wheel を誤って検証しないよう、import 先と新 API の有無を確認する。
- CPU 台帳 `/tmp/cpu-plan` も再起動で消え得る。重いビルド・テスト前に利用状況を確認し、予約・スレッド数上限を設定し直す。今回のセッションの予約はすべて終了済み。

### 保留 worktree の注意

`/Users/tmori/work3/scaluq-rotation` にあった D2 草稿は、2026-11-08 の追加整理で差分・独自コミットをバックアップした後に worktree とローカルブランチを削除した（上記参照）。削除前の `src/gate/gate_pauli.cpp` には未コミット差分があった。最終 D2 は別 worktree の `perf/single-qubit-pauli-rotation`（PR #467）で完了しているため、この草稿を無条件で取り込まない。念のため差分を `ref/reboot-artifacts/paused-rotation-worktree.patch` に保存した。別 AI の `/Users/tmori/work3/scaluq-metal-exp` にある未追跡 `cfg.log`・`cfg-cpu.log` には触れていない。

## 実装・PR の追跡

PR の作成・状態変更のたびに、この表と該当項目のリンクを更新する。PR はすべて upstream `qulacs/scaluq` の `main` 向け。各ブランチは独立しており、他の PR のマージを待たずにレビューできる。`ref/` は原則コミット対象外。ただし本書 `ref/scaluq_improvements.md` と F2 の調査結果（`ref/f2_metal_investigation.md`、`ref/f2_metal/`）は統合検証用ブランチで追跡し、PR 対象から除外する（F2 の調査結果の原本は PR #463）。

| 項目 | PR | ブランチ（fork: forest1040/scaluq） | 状態 |
|---|---|---|---|
| B1 NumPy 相互変換・B2 部分状態アクセス | [#460](https://github.com/qulacs/scaluq/pull/460) | `feat/state-data-access` | レビュー待ち・未マージ（ローカル検証・CI 成功） |
| C1 fused Pauli 加算・メモリ制限付き Operator 適用 | [#461](https://github.com/qulacs/scaluq/pull/461) | `feat/pauli-apply-add` | レビュー待ち・未マージ（ローカル検証・CI 成功） |
| C5 XOR グループ型 Pauli 演算子 | [#462](https://github.com/qulacs/scaluq/pull/462) | `feat/grouped-pauli-operator` | レビュー待ち・未マージ（ローカル検証・CI 成功） |
| C2 対角演算の要素積 API | [#465](https://github.com/qulacs/scaluq/pull/465) | `feat/state-elementwise-product` | レビュー待ち・未マージ（ローカル検証・CI 成功） |
| D1 小さいホスト状態の Serial 実行しきい値 | [#466](https://github.com/qulacs/scaluq/pull/466) | `perf/small-state-dispatch` | レビュー待ち・未マージ（ローカル検証・CI 成功） |
| D2 1 量子ビット PauliRotation の専用カーネル振り分け | [#467](https://github.com/qulacs/scaluq/pull/467) | `perf/single-qubit-pauli-rotation` | レビュー待ち・未マージ（半精度比較の CI 修正 `96bd335f` push 済み。ローカル検証成功、CI 再実行中） |
| C3 バッチ状態への Pauli 適用 | [#468](https://github.com/qulacs/scaluq/pull/468) | `feat/batched-pauli-application` | レビュー待ち・未マージ（ローカル検証・CI 成功） |
| A4 Pauli 全行列での係数欠落修正 | [#470](https://github.com/qulacs/scaluq/pull/470) | `fix/pauli-full-matrix-coefficient` | レビュー待ち・未マージ（ローカル C++ 1052 件・CPU 構成 CTest・Python 回帰 6 件成功。CI 全成功） |
| C4 重み付きバッチ演算・内積 | [#469](https://github.com/qulacs/scaluq/pull/469) | `feat/batched-weighted-algebra` | レビュー待ち・未マージ（ローカル検証・CI 成功） |
| F2 Metal 調査報告（文書のみ） | [#463](https://github.com/qulacs/scaluq/pull/463) | `research/f2-metal` | レビュー待ち・未マージ。別 AI が担当 |
| F2 Metal 対応のコード（AdaptiveCpp で SYCL バックエンドをビルド） | [#464](https://github.com/qulacs/scaluq/pull/464)（draft） | `exp/f2-metal-scaluq` | draft。パッチを当てた Kokkos が必要。Metal f32 229/229、OpenMP 968/968。別 AI が担当 |

### 統合検証用ブランチ

レビュー待ちの PR を組み合わせた場合の互換性を確認するため、PR 用とは別に統合検証ブランチを置く。各 PR 用ブランチは `upstream/main` から独立して作成し、統合検証ブランチへ各変更を cherry-pick する。統合時に必要な説明文などの競合解消は、このブランチ内で行う。

| ブランチ | 起点・取り込む項目 | 現在の状態 | 公開状態 |
|---|---|---|---|
| `test/high-priority-improvements` | `upstream/main` (`01934038`) + B1・B2・C1・C5（PR #460・#461・#462） | `b16fd8f8`。C++・Python・文書例・SKIPPER の統合検証済み。最終の CI 互換修正も別途検証済み | 現行統合ブランチに包含。2026-11-08 にローカルブランチ削除 |
| `test/medium-priority-improvements` | `test/high-priority-improvements` (`b16fd8f8`) + C2・D1・D2 + C5 CUDA 互換修正 | C2・D1・D2（PR #465・#466・#467）取り込み済み。コード先端 `c03728cd`。中優先度の全体検証完了、後続の C5 CUDA 互換修正も個別検証済み | 現行統合ブランチに包含。2026-11-08 にローカル・fork のブランチ削除 |
| `test/batched-operator-improvements` | `test/medium-priority-improvements` (`02c26082`) + C3・C4・A4 + F2（PR #464 の 5 コミット） | C3・C4（PR #468・#469）統合検証済み。CI 修正 #464・#467 と共通 JSON 取得先フォールバックも取り込み（コード先端 `31a52403`）。A4（PR #470）取り込み済み。F2 を 2026-11-07 に取り込み、OpenMP 1052/1052・`ScaluqCPUConfig`・Metal f32 250/250 を確認。2026-11-08 に CPU で再ビルドし、C++ 1052 件・CPU 構成 CTest・A4 Python 回帰 6 件成功 | fork へ push（upstream の PR・マージ対象外） |

現在の主作業ディレクトリ `~/work3/scaluq` は `test/batched-operator-improvements` を checkout している。各 PR 用ブランチは fork 上で維持し、ローカルブランチ・worktree は必要になったときに再作成する（2026-11-08 の整理は上記参照）。 表中のコミット番号はコードを取り込んだ先端を指し、その後の本書だけの更新コミットは除く。

- 統合検証ブランチはテスト・ベンチマーク用とし、このブランチから upstream へ PR を出したり、そのままマージしたりしない。
- **本書は統合検証用ブランチにだけコミットする。PR 用ブランチには追加せず、本書の更新コミットも cherry-pick しない。** `/ref/` の ignore 設定は維持し、本書を最初に追跡するときだけ `git add -f ref/scaluq_improvements.md` で対象ファイルを明示する。他の参考資料は追加しない。
- PR 作成・更新前に、各 PR 用ブランチと `upstream/main` の差分に `ref/scaluq_improvements.md` が含まれないことを確認する。
- 統合検証で必要になった実装修正は、該当する独立 PR 用ブランチへ反映し、そのブランチを検証して push する。統合検証ブランチにも同じ修正を取り込む。
- PR のマージ待ちの間も、統合検証ブランチでは複数の改善を組み合わせて検証できる。SKIPPER の `main` の依存設定は変更しない。
- 統合検証ブランチを作成・更新したら、この表の起点・構成項目・先端コミット・検証状況・push 状況を更新する。PR 番号やレビュー状態も併せて追跡する。
- **作業中に追加の課題が見つかった場合は、本書に内容・根拠・優先度・関連 PR・対応状況を追記する。** 優先度が決まっていない場合は「未設定」と明記し、追加課題の記録と着手の判断は分ける。

API 方針の相談: [Issue #459](https://github.com/qulacs/scaluq/issues/459)。優先度「高」の B1・B2・C1・C5 は実装済みでレビュー待ち。優先度「中」の C2・D1・D2 も実装・検証済みでレビュー待ち（F2・G1 は対象外）。A1〜A3 は実装済み。C3 → C4 の順で実装・検証し、PR #468・#469 はレビュー待ち（C3 は優先度未設定、C4 は低）。追加課題 A4 は PR #470 で修正済み。2026-11-08 に統合ブランチで C++ 1052 件・CPU 構成 CTest・Python 回帰 6 件の成功を確認、CI も全成功。F2 は別 AI の PR #463・#464 を統合済み。その他の未対応項目は保留。

2026-11-07 のローカル検証（4 項目を統合した CPU/OpenMP、f32/f64）: C++ 988 件 + CPU 構成 CTest、Python 回帰 5 件、文書例 406 件、SKIPPER 全 440 件が成功。0〜3 量子ビットの Pauli 文字列 340 通りを密行列と照合し、キャッシュ・ストリーミング・エイリアス・期待値・遷移振幅を確認。SKIPPER のコード・依存設定は変更していない。

測定（macOS arm64、f64、8 スレッド、専有予約）: n=22 の NumPy 出力の追加ピーク RSS は約 64 MiB（従来のリスト経由は約 289 MiB）。128 項 / 8 XOR グループ、n=20 の人工演算子では、コピー→適用→加算 100.78 ms、C1 31.16 ms、C5 キャッシュ 2.750 ms（warmup 後 10 回平均、キャッシュ構築を除外）。分子ハミルトニアンや GPU の実測ではない。詳細は各 PR 本文を参照。

C5 の初回 CI では静的解析用 Kokkos スタブに `IndexType` が無く失敗したため、不要なポリシー指定を除去して追加コミット（`575afd26`）を push。実 Kokkos と解析スタブの構文確認、および f32/f64・OpenMP/HostSerial・キャッシュ有無の数値確認は成功。2026-11-07 12:46 の後続確認で CUDA Install Library のコンストラクタ内拡張ラムダ制約による失敗を確認。キャッシュ作成を名前付き Kokkos functor に変更し、`5747e6c8` を PR #462 へ追加 push した。f32/f64・OpenMP/HostSerial のキャッシュ有無の数値確認と解析スタブの構文確認は成功。CUDA 側の再検証は CI 実行中。この修正は中優先度の統合検証用ブランチにも取り込んだ。


### 中優先度の実装・検証（2026-11-07）

- C2: `StateVector.multiply_elementwise(other)` を追加。通常の別状態と同一領域のエイリアスは追加バッファなし。C++ の部分重複ビューだけは重みをコピーしてから適用する。任意の複素対角成分、非正規化状態、0 量子ビットを扱う。
- D1: `set_small_state_serial_threshold(max_amplitudes)` / `get_small_state_serial_threshold()` を追加。対象となるホストゲートの実行をしきい値で Serial に切り替える。既定値 0 は無効、バッチは総振幅数で判定。OpenMP スレッドプールの再構成は行わない。
- D2: 実係数の 1 量子ビット X/Y/Z PauliRotation と ParamPauliRotation を RX/RY/RZ の専用カーネルへ振り分ける。バッチの異なるパラメータにも対応。複素係数・複数量子ビット・密度行列は従来の一般経路を使用する。

高優先度も含む統合検証は、C++ 1020 件 + CPU 構成 CTest、Python 回帰 47 件、文書例 412 件、SKIPPER 全 440 件が成功。静的解析用スタブの構文確認も実施した。各 PR は `upstream/main` 起点で独立しており、本書はどの PR の差分にも含めていない。

CPU 測定（macOS arm64、f64、8 スレッド）: C2 は n=20 の CSR 対角適用 2.047 ms に対して要素積 0.262 ms。D1 は n=12 の RX で、無効時 46.38 µs、しきい値 4096 で 3.66 µs。D2 は n=20 の X が 384.42→342.21 µs、Y が 421.61→361.72 µs、Z は 389.84→396.50 µs でほぼ同等。別 AI の F2 ビルド（12 スレッド）が並行稼働していたため、参考の相対測定として扱う。設定・測定方法は各 PR 本文に記録した。


### C3・C4 の実装・検証（2026-11-07）

C3 は同じ Pauli と複素係数をバッチの全状態へ 1 回のカーネルで適用する。量子ビット範囲を先に検証し、パディングした保存領域、空バッチ、0 量子ビットにも対応。C4 は `linear_combination(coefs)`、状態別係数の `multiply_coef` / `add_state_vector_with_coef`、対応する行同士の `inner_product` を追加した。部分重複ビューには入力を保存して対処し、重み付き和と内積は最低でも float、f64 は double で積・集計を行う。

統合検証: C++ 1044 件 + CPU 構成 CTest、Python 回帰 75 件成功・12 件スキップ、文書例 412 件、SKIPPER 全 440 件が成功。Python のスキップは CPU ビルドに追加ホストモジュールが存在しないケース。C3 → C4 の組み合わせも密行列と照合した。f16・bf16 は実 Kokkos/OpenMP の個別実行で、4096 項の重み付き和と内積がどちらも正しく 4096 になることを確認した。静的解析用スタブの構文確認と生成された公開型スタブの引数・戻り値も確認済み。

測定（macOS arm64、f64、8 スレッド、専有予約、他の台帳上のビルドは終了済み）: 64 状態・12 量子ビット、warmup 後 100 回平均で、Pauli の行ごとの適用 3.202 ms → バッチ適用 0.115 ms、行ごとの重み付き和 3.354 ms → 線形結合 0.127 ms、行ごとの内積 3.515 ms → バッチ内積 0.162 ms。行ビューは事前作成し、線形結合の出力確保は測定に含めた。詳細は各 PR 本文を参照。

追加課題 A4（行列取得での係数欠落）は記録のみとし、この 2 件の PR では修正していない。本書は統合検証用ブランチにだけコミットし、PR #468・#469 の差分には含めていない。


## A. 不具合

### A1. `Gate.get_inverse()` が壊れたゲートを返す
- 現象: 個々のゲートに `get_inverse()` を呼ぶと、返ったオブジェクトの `gate_type()` が `ValueError` になり、状態に作用させられない。
- 期待: `Circuit.get_inverse()` と同じく、逆ゲートを返す。
- 回避策: 回路単位の `Circuit.get_inverse()` だけを使う。回帰テストで固定（`tests/test_foundation.py`）。

### A2. `Circuit.get_inverse()` が `DenseMatrix` ゲートを含むと失敗する
- 現象: `DenseMatrix` を含む回路で `RuntimeError`。
- 期待: 共役転置の `DenseMatrix` を生成して逆にする（`SparseMatrix` も同様）。
- 回避策: 密行列を別に保持して、その共役転置で逆回路を作る（`CircuitUnitary.from_matrix`）。

### A3. `Circuit.gate_list()` のゲートが回路の寿命に依存する
- 現象: 一時的な回路から `gate_list()` で取り出したゲートを、回路を破棄した後に使うとセグメンテーション違反になる。
- 期待: 返すゲートが所有権（参照カウント）を持つ。少なくとも文書に明記する。
- 回避策: 元の回路への参照をゲートと一緒に保持する。

### A4. `PauliOperator.get_full_matrix(n_qubits)` が係数を無視する（追加課題）

- 発見: 2026-11-07、C3 の密行列参照テストの作成中に確認。
- 優先度: **未設定**。対応状況: PR [#470](https://github.com/qulacs/scaluq/pull/470) で修正済み・未マージ。C3・C4 とは独立した修正ブランチ。
- 再現: `PauliOperator("X 0", 2).get_full_matrix(1)` は `[[0, 1], [1, 0]]` を返す一方、`get_matrix()` は `[[0, 2], [2, 0]]` を返す。ビルド済みの f64 Python モジュールでも再現を確認した。
- 原因: `src/operator/pauli_operator.cpp` の `get_full_matrix` が係数なしの triplet をそのまま格納しており、`_coef` を掛けていない。
- 期待: `get_full_matrix` は係数を含め、`get_full_matrix_ignoring_coef` だけが係数を無視する。0 量子ビット、非自明な複素係数、全精度で回帰テストを追加する。
- C3 の回避策: テスト参照値には `get_full_matrix_ignoring_coef(n) * coef()` を使い、この既存不具合を修正済みと仮定しない。
- 関連 PR: C3 の実装 PR [#468](https://github.com/qulacs/scaluq/pull/468)。A4 修正 PR [#470](https://github.com/qulacs/scaluq/pull/470)。
- 回帰検証: C++ typed tests と Python f32/f64 テストを追加。修正前のモジュールで Python 6 件すべての失敗を確認済み。F2 専有測定の終了後、2026-11-08 に永続 Python 環境と拡張を再ビルドし、統合ブランチで C++ 1052 件・CPU 構成 CTest・Python 回帰 6 件が成功。CI も全成功。本書は統合検証ブランチのみで管理し、PR #470 の差分には含めない。

## B. ホストとのデータ転送（影響大）

### B1. 振幅を NumPy 配列として受け渡す手段がない
- 実装 PR: [#460](https://github.com/qulacs/scaluq/pull/460)（upstream 向け、未マージ）。
- 現象:
  - `get_amplitudes()` は Python の `list[complex]` を返す。C++ の vector（16 B/振幅）→ Python の complex オブジェクトのリスト（約 40 B/振幅）→ NumPy（16 B/振幅）と経由するため、状態の約 4.5 本分の一時メモリを使う。n=26 では状態 1 GiB に対して一時 4.5 GiB になる。
  - `load(seq)` も Python のシーケンスとして変換するため、状態の約 4.6 本分の一時メモリを使う（n=22 で 64 MiB の状態に 296 MiB）。
- 提案: バッファプロトコル／DLPack による NumPy との相互変換を用意する。少なくとも `get_amplitudes_numpy()` と `load_numpy(ndarray)`（コピー 1 回、16 B/振幅）を追加する。
- 回避策:
  - 読み込みは、ゼロ状態に疎行列ゲート `|ψ⟩⟨0|` を作用させて作る（`kernels.sv.embed_low`。デバイス上の 2 本 + O(2^n) で済む）。
  - 確率分布は `get_marginal_probability` でデバイス上から読む。
  - 射影状態の読み出しは全体コピーが避けられないため、その分をメモリ見積もりに含めている。

### B2. 部分読み出し（連続範囲・部分空間）の API がない
- 実装 PR: [#460](https://github.com/qulacs/scaluq/pull/460)（upstream 向け、未マージ）。
- 現象: 補助量子ビットが 0 の部分（先頭 2^n 要素）だけが欲しくても、全 2^(n+a) 要素を読むしかない。`get_amplitude_at(i)` は 1 回 165〜240 µs かかり、ループでは使えない（n=22 で 65,536 回に 15.9 秒）。
- 提案:
  - `get_amplitudes(start, count)`（連続範囲）を追加する。
  - より一般には、指定した量子ビットを固定値に射影した部分状態を、小さい `StateVector` として取り出す `extract_substate(fixed_qubits, values)` を追加する。逆の `embed_substate`（小さい状態を、他の量子ビット = 0 として大きい状態に埋め込む）もあると、上記の回避策が不要になる。

### B3. `get_amplitude_at` / `set_amplitude_at` が 1 回ごとに重い
- 現象: 1 回 165〜240 µs で、デバイス同期のためと思われる。ホスト空間にコピーした状態（`copy_to_host_space()`）でも 231 µs だった。
- 提案: ホスト空間の状態ではフェンスを省く。複数インデックスをまとめて読み書きする API（`get_amplitudes_at(indices)`）を追加する。

## C. Pauli 演算子の適用

### C1. 「out += c·P·x」を 1 回の走査で行うカーネルがない
- 実装 PR: [#461](https://github.com/qulacs/scaluq/pull/461)（upstream 向け、未マージ）。
- 現象: `PauliOperator.apply_to_state` はインプレースだけなので、Pauli 和の行列ベクトル積は 1 項ごとに「コピー → 適用 → `add_state_vector_with_coef`」の 3 回の走査が必要になる。`Operator.apply_to_state` は内部で項数ぶんの一時状態を確保するので、項数の多い演算子では使えない。
- 提案: `PauliOperator.apply_add(x, out, coef)`（out += coef·P·x、x は読み取り専用）と、それを使った `Operator.apply_to_state(x, out)`（一時状態 0〜1 本）を追加する。Pauli の作用はインデックスの XOR と位相だけなので、1 回の走査で書ける。
- 回避策（SKIPPER 2026-11-06）: 隣り合う項の積 Q_j = P_j P_{j−1} を前もって計算し、コピーを最初の 1 回だけにして、1 項あたり 2 回の走査（適用 + 加算）にした（`PauliSum._chain`）。C1 があれば 1 回の走査になる。

### C2. 対角演算子（Z 文字列の和）を 1 回で作用させる手段がない
- 実装 PR: [#465](https://github.com/qulacs/scaluq/pull/465)（upstream 向け、未マージ）。
- 現象: Z だけからなる項（イジング型・分子ハミルトニアンの大部分）も 1 項ずつ処理するしかない。対角行列を `SparseMatrix` ゲートで作用させると、n=20 で 4.5 ms と Pauli 適用（0.54 ms）の約 8 倍かかり、CSR 配列のメモリ（32 B/振幅）も必要になる。
- 提案: 対角ゲート（`DiagonalMatrix(targets, diag)`）または「Z 文字列の和を対角として作用させる」専用カーネルを追加する。2 つの状態の要素ごとの積（`multiply_elementwise`）でもよい。

### C3. `StateVectorBatched` に `PauliOperator.apply_to_state` が使えない
- 実装 PR: [#468](https://github.com/qulacs/scaluq/pull/468)（upstream 向け、未マージ）。
- 現象: 回路やゲートはバッチ状態に作用するが、`PauliOperator` は `TypeError` になる。
- 提案: バッチ版の Pauli 適用を追加する。QPE の複数結果の条件付き状態などで、複数本の状態をまとめて処理できる。

### C4. バッチ状態を重み付きで 1 本にまとめるカーネルがない
- 実装 PR: [#469](https://github.com/qulacs/scaluq/pull/469)（upstream 向け、未マージ）。
- 2026-11-07 の着手前レビューで、既存草稿 `05923b64` は未検証であり、低精度の積・縮約の精度、部分重複ビュー、空バッチ、Python 公開 factory の内積転送、パディングされた小さい状態のテストを補う必要があると確認。これらを C4 の実装・検証タスクとして扱う（優先度は C4 と同じ低、PR #469）。C3 の実装後にこれらを補完し、検証済み。
- 現象: `StateVectorBatched.add_state_vector_with_coef` は全状態に共通のスカラー係数しか取れず、`multiply_coef` も共通。バッチの各状態を違う係数で足し合わせて 1 本の状態にする（out = Σ_b c_b ψ_b）には、`view_state_vector_at` 経由で状態ごとに B 回の加算が必要になる。
- 提案: `StateVectorBatched.linear_combination(coefs) -> StateVector`（out = Σ_b c_b ψ_b）と、状態ごとの係数を取る `multiply_coef(coefs: list)`・`add_state_vector_with_coef(coefs: list, states)` を追加する。あわせて状態ごとの内積（`inner_product` のバッチ版）もあるとよい。
- 効果: LCU の SELECT（Σ_j c_j P_j ψ）や角度だけ違う U_j の線形結合（Σ_j c_j U_j ψ）を、項をバッチにして 1 回の適用 + 1 回の縮約で計算できる。項の多いハミルトニアン（分子で L ≈ n^4）と小さい状態で効果が大きい（SKIPPER バックログ P-13）。

### C5. XOR グループ型の Pauli 演算子（期待値と行列ベクトル積）
- 実装 PR: [#462](https://github.com/qulacs/scaluq/pull/462)（upstream 向け、未マージ）。
- 背景: Pauli 文字列は X/Y の位置を表すビットマスク x と Y/Z の位置を表す z で P_{x,z} = i^{|x∧z|} X^x Z^z と書け、基底への作用は P_{x,z}|b⟩ = i^{|x∧z|} (−1)^{popcount(b∧z)} |b⊕x⟩ になる。**同じ x を持つ項（XOR グループ）は、基底の並べ替え |b⟩→|b⊕x⟩ が共通で、違うのは対角の符号だけ**である。したがって
  - 行列ベクトル積: (Hψ)_{b'} = c_0 ψ_{b'} + Σ_x h_x(b'⊕x) ψ_{b'⊕x}、ここで h_x(b) = Σ_{z∈Z_x} c_{x,z} i^{|x∧z|} (−1)^{popcount(b∧z)} は状態に依存しない対角の重み。
  - 期待値: ⟨ψ|H|ψ⟩ = c_0‖ψ‖² + Σ_x Σ_b h_x(b) ψ*_{b⊕x} ψ_b。
  - どちらも計算量は項数 L ではなく**グループ数 G** に比例する（O(G·2^n)）。
- 現状: scaluq の `Operator.apply_to_state` と期待値計算は、項ごとに状態全体を走査する（O(L·2^n)）。`apply_to_state` は項数ぶんの一時状態も確保する（SKIPPER の測定、本書 C1）。
- 効果の目安（分子ハミルトニアン）: LiH（STO-3G、Jordan–Wigner、n=12）は L=630 項に対して G=84 グループ。そのうち対角（x=0）のグループに 78 項が集まる。行列ベクトル積・期待値の走査回数が約 7.5 分の 1 になる見込み（SKIPPER では未測定）。参考として、同じ手法を NumPy／C 拡張で実装した qva（Qulacs ベース、単一スレッド）の実測では、LiH の期待値 1 回が Qulacs の `Observable` 2.74 ms に対して事前集約版 0.120 ms（22.8 倍）、粒子数セクタに限った疎評価で 0.023 ms（121 倍）、行列ベクトル積（n=8、L=256）でゲート回路比 27 倍（h_x キャッシュ時 184 倍）と報告されている（`ref/qva/docs/ch05_directlcu.tex`、`ch06_pauli_expectation.tex`）。一様ランダムな Pauli 和（G≈L）では効果がない（qva で 0.94 倍）。
- 提案:
  1. **XOR グループ化した演算子の表現**（`GroupedPauliOperator` など）: 項を x でまとめ、各グループの h_x(b) を状態サイズの対角配列としてデバイス上に事前集約して保持する（メモリ G·2^n·16 B。大きい n ではグループ内の z の列だけを持ち、走査時に符号を計算するストリーミング版に切り替える）。
  2. **行列ベクトル積** `apply_to_state(x, out, coef)`: グループごとに out[b⊕x] += coef·h_x(b)·x[b] を 1 回の走査で行う（一時状態なし）。対角グループ（x=0）は要素ごとの積 1 回。
  3. **期待値** `get_expectation_value(state)` と遷移振幅 `get_transition_amplitude(bra, ket)`: グループごとに Σ_b h_x(b) ψ*_{b⊕x} ψ_b を 1 回のリダクションで計算する。H が Hermite なら h_x(b⊕x) = conj(h_x(b)) を使って b < b⊕x の半分だけを走査し、実部の 2 倍を取る。h_x(b)=0 となる組（ホッピング項の XX+YY が打ち消す基底）を事前に除くとさらに減る（qva の LiH で 14% に圧縮）。
  4. **グループ内の項が多い場合の FWHT**（任意）: 状態側の積 q_x(b) = ψ*_{b⊕x}ψ_b を Walsh–Hadamard 変換すれば、グループ内の全 z の寄与を O(n·2^n) でまとめて得られる（qva では L_x ≥ 8〜16 で有利）。
- 注意（意味の保存）: 対称性による項の除去（状態がある Z₂ 対称性の固有状態なら、それと反可換な項の期待値は 0）は**期待値専用**である。同じ演算子を行列ベクトル積に使うと、直交するセクタへの成分が欠落して結果が誤る。API で期待値用と作用用を分け、フィルタ済みの演算子を `apply_to_state` に渡せないようにする。
- SKIPPER 側の対応: scaluq に C5 が入れば、`PauliSum.matvec_add` の連鎖評価（現在 1 項あたり 2 回の走査）をグループ単位の 1 回に置き換えられる（QSVT、Qubitization、LCU、Lanczos が対象）。期待値の計算部品（現在は無い）もこれで作れる。

## D. ゲート実行の性能

### D1. カーネル 1 回あたりの固定費が小さい状態で支配的
- 実装 PR: [#466](https://github.com/qulacs/scaluq/pull/466)（upstream 向け、未マージ）。
- 現象: n=16（1 MiB）ではゲートの種類によらず 1 回約 0.1 ms かかる（RX 0.106、CX 0.095、PauliRotation 0.12、Pauli 適用 0.125 ms）。状態の更新そのものは 0.02 ms 程度で、残りは並列領域の起動などの固定費と見られる。SKIPPER の最適化経路では、n≤16 の実行時間の大半がこの固定費になる。
- 追加の測定（2026-11-06）: n=12 ではゲート 1 個が 1 スレッドで約 4 µs、2 スレッド 16 µs、4 スレッド 25 µs、8 スレッド 47 µs と、スレッドを増やすほど遅くなる。n=20 では 1 スレッド 0.84 ms、8 スレッド 0.20 ms。LiH（12 量子ビット）の QPE は 1 スレッドで 8 スレッドの 7.5 倍速かった。
- 提案: 小さい状態では並列化しない（またはスレッド数を絞る）しきい値を設ける。実行時にスレッド数を変更する API を用意する。

### D2. 1 量子ビットの `PauliRotation` が `RX`/`RZ` より遅い
- CI 追補（2026-11-07 13:35 JST）: PR #467 の [CUDA Install Library](https://github.com/qulacs/scaluq/actions/runs/42335509107/job/128291646936) が失敗。追加の残タスクとして原因確認・必要な修正・CUDA CI の再確認を行う。ローカル CPU の検証成功と区別し、原因はログ確認前のため未確定とする。
- 実装 PR: [#467](https://github.com/qulacs/scaluq/pull/467)（upstream 向け、未マージ）。
- 現象: n=20 で `PauliRotation(X q)` 0.525 ms に対し `RX` 0.374 ms、`PauliRotation(Z q)` 0.462 ms に対し `RZ` 0.304 ms。1 量子ビットの `DenseMatrix` は 0.366 ms。
- 提案: `PauliRotation` の生成時、または `Circuit.optimize` で、1 量子ビットの場合を専用カーネルに振り分ける。
- 回避策（SKIPPER 2026-11-06）: 最適化経路では、1 量子ビットゲートを量子ビットごとに 2×2 行列へまとめて 1 個の `DenseMatrix` にする独自の融合を使う（`CircuitUnitary.fused_circuit`）。n=20 の Trotter ステップでは 59 ゲートが 37 ゲートに減り、QPE（m=6）が約 2 倍速くなった。

### D3. `Circuit.optimize` が大きい状態で遅くなることがある
- 現象: n=20 の Trotter 回路（59 ゲート）で、適用時間は元のまま 35 ms、`optimize(1)` 44 ms、`optimize(2)` 44 ms、`optimize(3)` 340 ms、`optimize(4)` 408 ms と、ゲート数が減っても遅くなる。n=16 では `optimize(2)` で 5.4 → 3.2 ms と速くなる。多量子ビットの `DenseMatrix` が大きい状態で遅いためと見られる（2 量子ビット `DenseMatrix` 0.61 ms に対し 1 量子ビット 0.37 ms）。
- 提案: 融合の判断に状態サイズを含むコストモデルを使う。少なくとも「1 量子ビットゲートだけを融合する」モードを用意する。

### D4. 制御付きゲートが制御なしより速い場合がある
- 現象: 同じ回転ゲートで、制御付きのほうが速いことがあった（F0 の実測）。作用する振幅が半分になるためと考えられるが、制御なしのカーネルにも改善の余地があるかもしれない。
- 提案: 制御なしの回転カーネルの実装を見直す（調査依頼）。

## E. API・文書

### E1. `get_marginal_probability` の文書
- 現象: 文書には「State must be normalized」とあるが、実際には正規化されていない状態でも振幅の 2 乗の和をそのまま返す。SKIPPER はこの動作に依存している（非正規化の分岐の確率を読む）。
- 提案: 正規化されていない状態での動作（2 乗和を返す）を仕様として明記する。

### E2. レジスタ全体の分布を 1 回で返す API
- 現象: k 量子ビットのレジスタの分布を得るには `get_marginal_probability` を 2^k 回呼ぶ必要があり、毎回状態全体を走査する（n=22、k=6 で 64 回 0.02 秒）。
- 提案: `get_register_distribution(qubits) -> ndarray[2^k]` を 1 回の走査で計算する API を追加する。

### E3. `uninitialized_state` の内容
- 現象: 文書上は未初期化（NaN を含み得る）だが、手元の実測では 0 だった。NaN の場合に `multiply_coef(0)` でゼロにすると NaN が残るので、SKIPPER は未初期化バッファを必ず `set_zero_norm_state` などで初期化している。
- 提案: 未初期化の内容が不定であることと、安全なゼロ化の方法を文書に明記する。

## F. Apple Silicon の GPU（Metal）対応

### F1. 現状
- PyPI の wheel は CPU（OpenMP）版だけで、GPU 版は CUDA 向けにソースからビルドする必要がある。scaluq は Kokkos の上に作られているが、Kokkos 本体に公式の Metal バックエンドは無い（2026-11 時点）。したがって Apple Silicon の GPU は使えない。
- scaluq の精度はテンプレート化されている（`precision_available` は `f16`・`f32`・`f64`・`bf16` を受け付ける）。手元の wheel は `f32` と `f64` だけを有効にしてビルドされている。

### F2. 方針: FP32 以下に制限し、Kokkos + AdaptiveCpp + Metal で検証する
**前提（FP32 以下に制限する理由）**: Apple Silicon の GPU には倍精度（FP64）のハードウェアが無く、AdaptiveCpp の Metal バックエンドも `double` を扱えない（将来ソフトウェアエミュレーションの予定があるが、性能は出ない）。そこで Metal 版は **`f32`（complex64）とそれ以下の精度だけを有効にしたビルド**とし、`f64` は CPU（OpenMP）版に残す。scaluq は精度ごとにモジュールが分かれている（`scaluq.default.f32` など）ので、この制限は既存の構造に自然に収まる。

**構成**:

```
scaluq（C++、Kokkos::View / parallel_for / parallel_reduce）
  └ Kokkos SYCL バックエンド（Kokkos_ENABLE_SYCL=ON、コンパイラ acpp）
      └ AdaptiveCpp（generic SSCP: コンパイル時に LLVM IR、実行時に Metal Shading Language へ変換）
          └ Apple Silicon GPU（Metal）
```

**AdaptiveCpp の Metal バックエンドの現状**（公式文書より）:
- 実験的で、開発中。SYCL の全機能が使えるわけではない。
- macOS 26 と Apple Silicon でのみ試験されている（他の版では動かない見込みと明記）。
- `double` 非対応。64 ビット整数のアトミックはエミュレーション（競合時に遅い）。ホストとデバイスの間のアトミックは保証されない。
- デバイス USM は使える。`sycl::stream`／printf は使えない。ポインタと整数の相互変換は制限がある。
- SYCL イベントごとに Metal のコマンドバッファが必要で、通常のイベントを返す操作は個別にコミットされる（粗粒度イベントの拡張で軽減できる）。
- ビルド: LLVM 15 以上（AArch64 ターゲット）、Apple の metal-cpp ヘッダ、`-DWITH_METAL_BACKEND=ON -DMETAL_INCLUDE_DIR=...`。`acpp-info` で Metal デバイスが見えれば導入できている。

**Kokkos の SYCL バックエンドと AdaptiveCpp の相性**（Kokkos 開発者の報告より）:
- Kokkos の SYCL 実装は Intel oneAPI（DPC++）を主な対象とし、DPC++ の拡張機能に依存している。AdaptiveCpp は Kokkos が前提とする SYCL 2020 への準拠度が低いとされる。
- AMD GPU（Frontier）で AdaptiveCpp を使った例では、SYCL 関数の不足などの問題があり、拡張機能を無効にすると「任意サイズのアトミック、TeamPolicy、Graph、Kokkos::Random が使えない」と報告されている。`acpp` と `Kokkos_ENABLE_SYCL=ON` で Kokkos を構成すること自体は行われている。
- したがって、この構成がそのまま動く保証は無い。Kokkos 側の修正が必要になる可能性が高い。

**scaluq 側で問題になりそうな箇所**（要確認）:
- `Kokkos::Random` を使う機能（Haar ランダム状態、サンプリング）。
- `TeamPolicy` や任意サイズのアトミックを使うカーネル（周辺確率、サンプリングの集計など）。
- これらは状態ベクトルの基本演算（ゲート、Pauli 適用、内積・ノルム）ではないので、Metal 版では無効にするか、ホスト側で代替する方針がとれる。SKIPPER の最適化経路は乱数・サンプリングを使わない。

### F2 調査の分担（2026-11-07）

ユーザーの方針により、**F2 の Metal 対応は別 AI が並行して調査する**。この作業セッションは C2・D1・D2 の実装、統合検証、upstream 向け PR 作成を担当し、F2 の調査・実装とは独立して進める。G1 も本セッションの対象外。

別 AI から F2 の調査結果が届いたら、本書に実現可能性・対応方針・前提環境・互換性の制約・実際に検証できた範囲を反映する。Metal の実装・検証に進む場合は、作業場所の方針どおり別ディレクトリ・別ブランチを使い、PR を作成したら番号と状態を本書の追跡表へ追加する。CPU 側の統合検証結果を Metal 対応の検証結果として扱わない。

### F2 調査結果（2026-11-07、別 AI）

詳細は `ref/f2_metal_investigation.md`（第 1〜9 版）。生データ・パッチ・再現プログラムは `ref/f2_metal/`。

- **実現可能性**: 可能。f32 の既存テストがすべて Apple M3 Ultra の GPU で成功した。単独のブランチでは 229/229。本ブランチに統合した状態では 250/250 で、C3・C4・C5 などの新機能を含む。
- **前提環境**: macOS 26.7.1、AdaptiveCpp develop（`6c923f68`）に Metal 用の小さなパッチ、Kokkos 5.2.0 にパッチ（`ref/f2_metal/patches/`、24 ファイル）。実行時は `KOKKOS_DEVICE_ID=1` が必要。f64 は使えない（Metal 版は f32 以下に限る）。
- **scaluq 側の変更**（PR #464、draft、6 コミット）:
  - CMake オプション `SCALUQ_SYCL_ACPP` を追加した。
  - デバイスコードから double を取り除いた。
  - スクラッチレベルをコンパイル時定数にした（Metal で変換できない IR を避けるため）。
  - f32 乱数の NaN を直した。
  - Metal ビルドではカウンタ方式（Philox）の乱数を使う。ほかのビルドは従来の乱数のまま。
  - `internal::Sum` の結果 View を HostSpace にした（本ブランチの統合検証で見つかった）。
  - `pyproject.toml` で `SCALUQ_SYCL_ACPP` を受け渡す（Python wheel 用）。
- **性能**（GPU が空いた状態、f32）:
  - n ≥ 24 のゲートはネイティブ Metal と同じ約 700 GB/s で、CPU（OpenMP 28 スレッド）の 2.6〜3 倍。
  - 小さい状態の 1 ゲートは約 19 µs。Kokkos パッチで間接カーネル用バッファを 4 個から 256 個に増やし、約 57 µs から縮めた。
  - reduce には約 0.26 ms の固定コストが残る。
- **互換性の制約**:
  - 上流の Kokkos・AdaptiveCpp にはまだ入っていない。報告済み: Kokkos #9750、AdaptiveCpp #2310〜#2313。
  - GPU 上で Kokkos 標準の乱数プールを大きな並列度で使うと、カーネルが暴走して OS を再起動するまで GPU に残る。scaluq の乱数は PR #464 で対策済み。
- **上流 scaluq への報告**: #471（`Operator::get_expectation_value` が項ごとの計算の 5〜18 倍遅い）、#472（f32 の `get_squared_norm`/`normalize` の精度劣化）。どちらも CPU でも起きる。
- **本ブランチで確認した範囲**: OpenMP の C++ 1052/1052、`ScaluqCPUConfig`、Metal の f32 C++ 250/250。CPU の結果を Metal の検証結果として扱わない、という方針は変わらない。
- **Python（2026-11-08）**: `db94d63b` から Metal 版の wheel（f32 だけ）を作り、GPU で動くことを確認した。f32 に置き換えた `python_test` は 93/93、doctest は 407/412（失敗は f64 の桁数で書かれた期待値だけ）。SKIPPER の f32 の精度テストも、この wheel で 12/12 成功した（SKIPPER 側のセッションが確認）。ビルドには `SCALUQ_USE_OMP=OFF` が要る。詳細は `ref/f2_metal_investigation.md` の冒頭と第 8 版。
- **残件: wheel の配布**: wheel は AdaptiveCpp のライブラリ（`libacpp-rt` などと、実行時に読み込む Metal バックエンド・JIT・bitcode、さらに Homebrew の LLVM 21）を絶対パスで参照するので、ビルドしたマシンでしか動かない。ほかのマシンで使うときは、これらの同梱か利用者側での導入が必要。未対応。詳細は `ref/f2_metal_investigation.md` 第 8 版の「残件: wheel の配布」。

### F3. 検証の手順
1. **AdaptiveCpp 単体**: Metal 対応で AdaptiveCpp をビルドし、`acpp-info` で GPU が見えること、簡単な SYCL カーネル（axpy、リダクション）が動くことを確認する。
2. **Kokkos SYCL バックエンドを acpp でビルド**: Kokkos の単体テストを走らせ、失敗するテストを洗い出す。`parallel_for`、`parallel_reduce`、`View`、`deep_copy` を優先する。
3. **scaluq の f32 ビルド**: `f64` を無効にして scaluq をビルドし、状態ベクトルの基本演算（1・2 量子ビットゲート、制御ゲート、`PauliOperator.apply_to_state`、`add_state_vector_with_coef`、内積・ノルム、`get_marginal_probability`）から動かす。乱数・サンプリングは後回しにする。
4. **精度と性能の比較**: 同じ回路を CPU f64、CPU f32、Metal f32 で実行し、誤差（f64 との差）と時間を比べる。状態サイズで CPU と GPU のどちらが速いかの境目を測る。
5. **必要なら**: 互換性の問題が大きい場合は、F4 の代替案に切り替える。Kokkos にネイティブの Metal バックエンドを追加するのは長期の課題とする。

### F4. 代替案
| 案 | 実現性 | 長所 | 短所 |
|---|---|---|---|
| A. Kokkos + AdaptiveCpp + Metal（F2、第一候補） | 実験的 | scaluq のソースをほぼそのまま使える。CUDA・HIP と同じ C++ コード | Kokkos SYCL と AdaptiveCpp の相性問題。AdaptiveCpp の Metal は macOS 26 限定・実験的 |
| B. 基本カーネルだけを Metal で直接実装 | 現実的 | 依存が少なく、性能を詰めやすい。共有メモリでホストとのコピーを省ける | scaluq と別のコードを保守する必要がある |
| C. Kokkos にネイティブ Metal バックエンドを追加 | 大規模 | 最も素直。他の Kokkos アプリにも効く | 開発量が大きい |

### F5. SKIPPER から見た効果と制約
- **効果**: 状態ベクトルの更新はメモリ帯域で律速される（D 節の測定では、n=20 の FAITHFUL で振幅・ゲートあたり約 0.27 ns、CPU 8 スレッド）。FP32 では 1 振幅 8 B（f64 の半分）なので、帯域とメモリの両面で有利になる。ユニファイドメモリなので、大きな状態（例: n=32 で 32 GiB）もコピーなしで GPU から扱える見込み。ただし GPU から使えるメモリの上限は OS の設定に依存する（要確認）。
  - **実測（2026-11-08）**: 統合ブランチ（`db94d63b`）の Metal 版 wheel（f32）で、SKIPPER の最適化経路（QPE、ウォーク QPE、QSVT、LCU。n = 16〜26）を測った。Metal は CPU の最速の設定（PyPI 版 scaluq 0.2.0、f64/f32、8 または 24 スレッド）より 2〜7 倍速い。例外は LCU の n ≤ 20 で、CPU と同程度。拡張 API を切って CPU と同じ経路にそろえても、n ≥ 24 で約 2 倍速い。詳細は `ref/f2_metal_investigation.md` の第 9 版。
- **固定費**: n≤16 ではカーネル起動の固定費（D1）が支配的で、GPU の起動費とコマンドバッファのコミット費が加わるとかえって遅くなるおそれがある。状態サイズで CPU と GPU を切り替える。
  - **実測（2026-11-08）**: ゲートの多い Trotter の QPE でも、n = 16 で Metal が CPU の 2.2 倍速かった。Kokkos パッチで起動遅延を約 19 µs に縮めた後の値。切り替えが必要なのは、LCU のように 1 回だけの短い計算（n ≤ 20）に限られる。
  - **#472 の実例**: CPU f32 の 8 スレッドで、n = 26 の入力状態のノルム² が 1.0000447 になり、SKIPPER の検査で拒否された。Metal では起きていない。
- **精度（FP32 の制約）**:
  - 単精度の丸め誤差は相対 1e-7 程度で、SKIPPER の最適化経路と FAITHFUL の照合基準（1e-10）は満たせない。FP32 用の許容誤差（目安 1e-5 程度、反復回数に応じて）を契約ごとに決め、誤差台帳（`validation/ledger.py`）に精度を記録する必要がある。
  - 長い反復（QPE の U 2^m−1 回、Horner、Clenshaw、KPM のモーメント）では誤差が回数とともに増える。内積・ノルムのリダクションは補償付き加算（Kahan など）を使うか、部分和をホストの f64 で集めるのが望ましい。
  - QPE の分布を Fejér 折畳み + FFT で求める部分はホスト（NumPy、f64）で行うので、GPU では相関 c_k = ⟨ψ|U^kψ⟩ の計算だけが FP32 になる。
  - 三角検証（密行列・FAITHFUL・最適化）の基準は f64 の CPU で保ち、Metal f32 は「f64 との差が許容内か」で検証する。
- **SKIPPER 側の対応**: Metal 版 scaluq ができた段階で、(1) `Workspace` と `CircuitUnitary` / `PauliSum` が精度（`f32`/`f64`）と実行空間を選べるようにする、(2) 誤差台帳に精度と FP32 の許容誤差を加える、(3) `plan()` のメモリ見積もりの 1 振幅あたりのバイト数を精度に合わせる。現時点では変更不要。

## G. GPU 以外のアクセラレータ

状態ベクトル・シミュレーションの計算は、ほぼすべて「状態全体を 1 回なめる」演算で、倍精度複素数（complex128）を使い、メモリ帯域で律速される。演算密度（バイトあたりの演算数）は低い。行列積を高速化するアクセラレータの多くは低精度・高演算密度向けなので、そのままでは効果が出にくい。以下は種類ごとの見立てで、いずれも SKIPPER では未測定である。

| アクセラレータ | 精度・特徴（一般的な公開情報の範囲） | 状態ベクトルへの適性 | 使いどころ |
|---|---|---|---|
| Apple AMX／SME（CPU 内の行列演算ユニット。M4 以降は Arm SME） | 倍精度を含む行列の外積・積を高速に行う。Accelerate（BLAS）経由で使うのが標準 | △ 1〜2 量子ビットゲートは演算密度が低く効果は小さい | 多量子ビットの融合ゲート（`DenseMatrix` k≥3）。D3 で遅かった部分を、状態を (2^k × 2^(n−k)) 行列とみなした行列積にすれば効く可能性がある |
| Apple Neural Engine（ANE） | 低精度（FP16／INT8）の推論向け。直接プログラムする公開 API は限られる（Core ML 経由） | × complex128 が無く、精度が足りない | 対象外（低精度の近似シミュレーションを研究する場合のみ） |
| Google TPU | 主に bfloat16／FP32 の行列演算。倍精度はエミュレーションになる | × 倍精度複素数では効率が出ない | 単精度（complex64）で許される用途に限る |
| Intel／AMD／Qualcomm の NPU | 低精度推論向け | × | 対象外 |
| Intel AMX（Xeon の行列ユニット） | BF16／INT8 中心 | × | 対象外 |
| CPU の SIMD（AVX-512、Arm SVE／NEON） | 倍精度を扱え、帯域を使い切りやすい | ○ 既に Kokkos／コンパイラ経由で使っている | カーネルのベクトル化の確認（D1・D2 の固定費と別問題） |
| FPGA | 任意精度の専用回路を組める。HBM 搭載品なら帯域も大きい | △ 研究段階。開発費が大きい | 固定回路の大量反復（同じ U を何度も掛ける QPE など）に限れば候補 |
| 複数ノード（MPI）・複数 GPU | 帯域とメモリを足し合わせられる | ○ n≥32 の大規模で唯一の現実的な道 | 量子ビットをノードに分配し、上位量子ビットへのゲートだけ通信する方式（既存の分散シミュレータと同じ） |

低精度での精度と速度を実測する計画は `docs/lowprec_benchmark_plan.md` にまとめた（このマシンは M3 Ultra で SME は無く、行列演算ユニットは Accelerate 経由の AMX）。

### G1. 提案
1. **精度の選択肢（complex64）**: 単精度の状態ベクトルを公式にサポートし、精度とメモリ・帯域を引き換えにできるようにする。scaluq には `f32` のモジュールがあるが、SKIPPER は誤差台帳の都合で `f64` を使っている。Metal 版（F2）は FP32 以下に限るので、この項目はその前提でもある。単精度にすれば帯域律速のカーネルは約 2 倍速くなり、状態のメモリも半分になる見込み。低精度向けアクセラレータを使う前提にもなる。
2. **融合ゲートを行列積として実行する経路**: k 量子ビットの融合ゲートを、状態の並べ替え + BLAS の行列積（AMX／SME／GPU のテンソル演算ユニット）として実行する。D3 の `Circuit.optimize` が遅くなる問題を、融合を深くする方向で解決できる。
3. **実行空間の抽象化**: Metal（F1）・SME・分散（MPI）など複数のバックエンドを、状態サイズや精度に応じて選べるようにする。小さい状態は CPU 1 スレッド、大きい状態は GPU や分散、という切り替え（D1 の固定費の問題も含む）。
4. **分散実行**: n≥32（状態 64 GiB 以上）では単一マシンのメモリに収まらない。量子ビット分配と通信の最小化を scaluq 本体で提供する。

### G2. SKIPPER 側の対応
現時点では不要。精度（complex64）を選べるようになった場合は、誤差台帳（`validation/ledger.py`）に丸め誤差の項目として記録し、結果契約ごとに許容誤差を確認する必要がある。融合を深くする経路ができた場合は、`CircuitUnitary.fused_circuit` の融合規則を差し替える。

---

## 修正の進め方（方針、2026-11-06 決定。修正は未着手）

### 作業場所
- qulacs/scaluq を GitHub でフォークし、SKIPPER の外の **別ディレクトリ**（例: `~/work3/scaluq`）にクローンして作業する。リモートはフォークを `origin`、本家を `upstream` とする。
- SKIPPER の `ref/` の下には置かない。理由:
  - リポジトリが入れ子になり、誤ってコミットに入るおそれがある（過去にコミット `de51707` で作業用ディレクトリへの参照が入った）。
  - C++（Kokkos + nanobind + CMake）のビルド生成物が SKIPPER のツリーに溜まり、pytest・ruff・uv が拾うおそれがある。
  - `ref/` は読むだけの参考資料（コミット対象外）の置き場である。
  - PR のレビュー待ちが長いので、SKIPPER と独立に維持できる作業場所がよい。
- Metal／AdaptiveCpp の実験（F3）も同じく別ディレクトリ・別ブランチで行う。

### ブランチと PR
- **必ずブランチを切り、PR を出す**。本家の `main` へは、他の人のレビューを受けてからマージされる（すぐにはマージされない前提で進める）。
- **PR を作成したら、その都度この文書の「実装・PR の追跡」に対象項目・upstream の PR 番号・リンク・状態を反映する**。レビューやマージで状態が変わった場合も更新する。本書だけは統合検証用ブランチにコミットし、PR 用ブランチには追加しない。他の `ref/` 配下はコミット対象外のままとする。
- ブランチは `upstream/main` から作り、提案ごとに小さく分ける（例: `fix/gate-get-inverse`、`fix/dense-matrix-inverse`、`feat/pauli-apply-add`、`feat/grouped-pauli-operator`）。1 PR = 1 テーマ。
- 未マージの複数 PR を組み合わせる検証には、上記「統合検証用ブランチ」を使用する。PR 用ブランチの独立性を維持し、統合ブランチの構成・検証・push 状況を本書へ記録する。
- 順序: まず不具合の修正（A1 `Gate.get_inverse`、A2 `DenseMatrix` を含む回路の逆、A3 `gate_list` の寿命）から。小さく、受け入れられやすい。
- API を追加するもの（B1 NumPy との受け渡し、B2 部分読み出し、C1 out += c·P·x、C4 バッチの重み付き縮約、C5 XOR グループ型 Pauli 演算子など）は、実装の前に issue で方針を相談する。
- 着手前に、本家の貢献手順（CONTRIBUTING、CLA の有無、コード書式、CI）を確認して従う。

### テストと効果の確認
- scaluq 自体のテスト（C++ と Python）を通す。
- 手元でビルドした scaluq で SKIPPER の全テストとベンチマークを走らせ、効果（時間・メモリ）を PR に記載する。
- scaluq のビルドとベンチマークは重いので、CPU 台帳（`/tmp/cpu-plan`）に予約してから行う。

### SKIPPER 側の扱い
- SKIPPER の `main` は PyPI 版の scaluq のまま保つ。
- 手元ビルドを試すときは SKIPPER 側も作業ブランチを切り、そのブランチでだけ `pyproject.toml` の `[tool.uv.sources]` に `scaluq = { path = "../scaluq", editable = true }` を指定する。
- SKIPPER のコードは「新しい API があれば使い、無ければ現在の方法に戻る」形（機能の有無の検査とフォールバック）にしておく。scaluq の新版が出たときに、そのまま速くなる。

### 未決事項
- フォークを作る GitHub アカウント（forest1040 の想定）。
- 作業ディレクトリの場所（`~/work3/scaluq` の想定）。
- 最初の PR を A1〜A3 の不具合修正から始めること（推奨）。

---

## 優先度の目安（SKIPPER から見た効果）

| 優先 | 項目 | 効果 |
|---|---|---|
| 高 | B1 NumPy 相互変換、B2 部分読み出し | FAITHFUL のホストメモリを状態の約 5 本分から 1 本分に減らせる |
| 高 | C1 out += c·P·x | Pauli 和の matvec の走査を 2 回から 1 回へ（QSVT、Qubitization、LCU が最大約 2 倍） |
| 低 | C4 バッチの重み付き縮約・状態ごとの係数 | LCU の SELECT や角度だけ違う U_j の線形結合のバッチ化（小さい状態で有効） |
| 高 | C5 XOR グループ型の Pauli 演算子（期待値・行列ベクトル積） | 分子ハミルトニアンで走査回数を L から G へ（LiH で 630 → 84）。C1・C2 の一般化。qva の実測で期待値 22.8〜121 倍 |
| 中 | C2 対角演算子 | Z 項の多いハミルトニアンで matvec の走査回数を大幅に削減 |
| 中 | D1 固定費、D2 1 量子ビット PauliRotation | n≤16 の最適化経路、Trotter 回路の高速化 |
| 中 | F2 Metal 対応（FP32 以下、Kokkos + AdaptiveCpp + Metal） | Apple Silicon の GPU とユニファイドメモリで大きな状態を高速化。実測で n≥24 のゲートは CPU の 2.6〜3 倍、ネイティブ Metal と同等（PR #463・#464、上流の Kokkos・AdaptiveCpp への取り込みが前提） |
| 中 | G1-1 complex64、G1-2 融合ゲートの行列積実行 | 帯域律速カーネルの約 2 倍速、深い融合による U 適用の短縮（見込み、未測定） |
| 低 | G1-4 分散実行 | n≥32 の大規模（SKIPPER の当面の範囲外） |
| 低 | A1〜A3、D3、E1〜E3 | 正しさ・使い勝手（回避策あり） |

測定スクリプトは SKIPPER リポジトリの `benchmarks/` と、本書作成時の一時スクリプト（未保存）による。再現が必要な項目は `tests/` の該当テストと `docs/design.md` §11, §14 を参照。

## 参考資料（F 節）
- AdaptiveCpp Metal バックエンド: https://adaptivecpp.github.io/AdaptiveCpp/install-metal/
- AdaptiveCpp のコンパイル方式（generic SSCP）: https://github.com/AdaptiveCpp/AdaptiveCpp/blob/develop/doc/compilation.md
- D. Arndt（Kokkos）, IWOCL 2025 講演資料（Kokkos SYCL バックエンドと AdaptiveCpp）: https://www.iwocl.org/wp-content/uploads/iwocl-2025-daniel-arndt-kokkos.pdf
- Experiences with SYCL on AMD GPUs with Kokkos（ORNL）: https://www.osti.gov/biblio/3016989


