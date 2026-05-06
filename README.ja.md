# clauroboros

[English](./README.md) | [한국어](./README.ko.md) | [日本語](./README.ja.md)

[Ouroboros](https://github.com/Q00/ouroboros) ワークフローを Claude Code
プラグインとしてネイティブ移植。

`ouroboros` Python CLI のラッパーでは**ありません**。spec-first ループ
(interview → seed → evaluate → drift → unstuck → ralph) を Claude Code の
スラッシュコマンドと skill として直接実装しています。状態はプロジェクト
ルートの `.ouroboros/` に保存され、[Karpathy コーディングハーネス](https://github.com/forrestchang/andrej-karpathy-skills/blob/main/CLAUDE.md)
が skill にバンドルされ、コーディングタスクごとにトリガーされます。

## インストール

git リモートから:
```sh
claude plugin marketplace add shseooo/clauroboros
claude plugin install clauroboros@clauroboros-cc
```

ローカルチェックアウトから (開発用):
```sh
claude plugin marketplace add ./clauroboros
claude plugin install clauroboros@clauroboros-cc
```

## 移植した概念

| 概念        | Claude Code の仕組み                                                                                |
| ---------- | --------------------------------------------------------------------------------------------------- |
| Seed       | `.ouroboros/seed.json` (正本) + `seed.yaml` (ミラー)。finalize 後にロック。                          |
| Interview  | `/ooo-interview <ゴール>` がエージェントに Socratic な質問を一度に一つずつ行うよう指示                    |
| Evaluate   | `/ooo-evaluate` がテストランナーを自動検出して実行し、コードベースを検査して各 AC を採点                  |
| Drift      | `/ooo-drift` が自己評価をリクエスト → `0.5*goal + 0.3*constraint + 0.2*ontology`                        |
| Unstuck    | `/ooo-unstuck [persona]` で 5 種類の lateral persona の一つを有効化 (state.json に記録)                  |
| Ralph      | `/ooo-ralph [N]` 単一コマンド内部で evaluate-and-fix の自己ループ (ハード cap 付き)                      |
| ハーネス    | Karpathy ガイドラインを `skills/clauroboros/SKILL.md` にバンドル — コーディングトリガーで auto-load   |

## 用語集

- **Seed** — 不変の仕様: goal + acceptance criteria + constraints + ontology
  + exposed assumptions。一度ロックされると全コマンドが権威データとして扱う。
  `seed.json` (正本) + `seed.yaml` (ミラー) として保存。
- **Acceptance Criterion (AC)** — 「完了」 の一部を定義する検証可能な単一
  ステートメント。各 AC は mechanical テストまたはコードベース検査で
  `pass` / `fail` / `n/a` に採点される。seed をロックするには AC ≥ 5 が必要。
- **Ambiguity score** — インタビュー中の仕様の曖昧度を表す 0–1 のスコア。
  ambiguity ≤ 0.2 のときだけ seed がロックされる。
- **Constraint** — 解が守るべき非機能要件 (性能、セキュリティ、互換性、
  依存性ポリシー等)。
- **Ontology** — プロジェクト固有の語彙を固定する 用語 → 定義 マップ。
  全コマンドと AC が同一概念を参照することを保証。
- **Persona** — 詰まり状態を打開する lateral-thinking レンズ 5 種:
  `inverter`, `first-principles`, `naive-newcomer`, `adversary`, `architect`。
  `state.json` に記録され、次セッションが引き継げる。
- **Drift** — 現在の作業とロック済み seed の乖離。重み付け
  `0.5*goal + 0.3*constraint + 0.2*ontology`。≤ 0.30 で OK、超えると DRIFTED。
- **Ralph** — ハードターン cap 付きの自己 evaluate-and-fix ループ。全 AC
  通過 (CONVERGED) または cap 到達まで繰り返す。
- **Hard cap (ハード cap)** — `/ooo-ralph` が強制停止されるまでの最大反復回数
  (デフォルト 8)。暴走ループを防ぐためのもの。cap に到達しても AC が収束
  していなければ、AC を弱めて偽の成功を作るのではなく、正直に停止して報告する。
- **Scope creep (スコープクリープ)** — ロック済み seed からの漸進的乖離:
  機能の追加、ゴールの拡大、用語の再定義、制約の緩和など。`/ooo-drift` で検出。
  seed は推奨ではなく境界線 — 本当にゴールが変わったなら、黙って広げずに
  止めて `/ooo-interview` をやり直す。
- **Karpathy ハーネス** — コーディング行動ガイドライン (まず考える、外科的
  変更、成功基準を定義、仮定を明示)。skill にバンドルされ、全コーディング
  タスクに適用される。

## スラッシュコマンド

| コマンド                  | 動作                                                              |
| ------------------------- | ----------------------------------------------------------------- |
| `/ooo-interview <ゴール>`      | ゴールを seed に結晶化する Socratic インタビューを開始              |
| `/ooo-seed`                   | ロック済み seed (または現在の draft) を出力                         |
| `/ooo-evaluate`               | mechanical テスト + 各 AC をファイルレベルの証拠で採点              |
| `/ooo-drift`                  | ロック済み seed に対する drift を自己評価                            |
| `/ooo-unstuck [id]`           | lateral persona 5 種類のいずれかに切り替え                          |
| `/ooo-ralph [on\|off\|N]`     | ハード cap 付きインライン evaluate-and-fix ループ                   |
| `/ooo-status`                 | interview / seed / persona / ralph / drift の状態                  |
| `/ooo-reset`                  | セッション状態をクリア (ロック済み seed は保持)                     |

## いつどのコマンドを使うか

| 状況                                                       | コマンド                 |
| ---------------------------------------------------------- | ------------------------ |
| 曖昧なゴールで新機能/タスクを開始                          | `/ooo-interview <ゴール>`     |
| 現在の仕様 (ロック済み seed または draft) を確認            | `/ooo-seed`                  |
| 意味のある作業ステップ完了 — AC に対して検証               | `/ooo-evaluate`              |
| 何度か編集後 scope creep を疑う                            | `/ooo-drift`                 |
| 詰まる、ループする、低品質な解しか出ない                    | `/ooo-unstuck [persona]`     |
| ハード cap 内で自律的に全 AC 通過させたい                   | `/ooo-ralph [N]`             |
| ループ上どこにいるか忘れた                                  | `/ooo-status`                |
| 進行中状態をクリアし seed は保持                            | `/ooo-reset`                 |

### `/ooo-unstuck` の persona 選択ガイド

| 詰まっている理由                                     | Persona            |
| ---------------------------------------------------- | ------------------ |
| 設計が過剰に複雑に感じる                             | `inverter`         |
| 悪い土台の上にパッチを重ねている                     | `first-principles` |
| コードベースが見知らぬもので仮定が隠れている          | `naive-newcomer`   |
| 解が正しいか判断できない                             | `adversary`        |
| モジュール境界が間違っているように感じる             | `architect`        |

### ループ運用ガイド

- 些末でない作業なら **コードを書く前に** `/ooo-interview` を実行。seed を先に
  結晶化するコストは後で書き直すコストよりはるかに小さい。
- 「完了」 を宣言する前のゲートとして `/ooo-evaluate` を使う — AC 採点なしの
  自己宣言は禁止。
- `/ooo-drift` は定期的に (数回の編集後、リファクタ後) 実行 — 安価で scope creep
  の早期検出に効く。
- `/ooo-ralph` は残作業が mechanical (失敗 AC + 明確な修正経路) のときに使用。
  **AC が間違っているときは使わない**。AC が間違いなら止めて `/ooo-interview`
  で seed を改訂。収束させるために AC を弱めるのは絶対に禁止。

## Claude Code ハーネスの制約

Claude Code が公開していない機能:
- ターンごとの `before_agent_start` system prompt 注入
- 次ターンへの自動 follow-up メッセージ

そのため:
- **Karpathy ハーネス** は SKILL に配置 — description のコーディングキーワード
  で auto-load。さらに全コマンド本文から明示的に参照し、常に適用されることを保証。
- **Persona の持続性** はファイル (state.json) ベースのみ。SKILL がセッション
  開始時に state.json を再読込するよう指示。ターン自動デクリメントフックは無し。
- **Ralph** は複数ターンではなく単一コマンド内部で cap まで自己ループ。

## 状態ファイル

```
.ouroboros/
├── seed.json        # 正本 seed (全コマンドが権威データとして読込)
├── seed.yaml        # 人間 / Ouroboros 互換ミラー
├── state.json       # interview / persona / ralph / acGrades / drift
└── interview.jsonl  # append-only イベントログ
```

## プラグインのファイル構成

```
clauroboros/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── commands/
│   ├── ooo-interview.md
│   ├── ooo-seed.md
│   ├── ooo-evaluate.md
│   ├── ooo-drift.md
│   ├── ooo-unstuck.md
│   ├── ooo-ralph.md
│   ├── ooo-status.md
│   └── ooo-reset.md
└── skills/
    └── clauroboros/
        └── SKILL.md
```

## ワークフロー例

```
1. /ooo-interview "TODO 管理 CLI を作る"
   → エージェントが一度に一つずつ Socratic インタビューを実施
   → 回答ごとに .ouroboros/state.json の seedDraft を更新
   → ambiguity ≤ 0.2 + AC ≥ 5 に到達したら seed.json + seed.yaml をロック

2. (コードを書く)

3. /ooo-evaluate
   → npm test / pytest / make test / cargo test / go test を自動検出 + 実行
   → 各 AC ごとにコードベースを検査し verdict (pass/fail/n-a) と証拠を記録
   → pass / fail の集計を出力

4. /ooo-drift
   → goal / constraint / ontology の divergence を自己評価
   → weighted ≤ 0.30 なら OK、超えると DRIFTED

5. (詰まったとき) /ooo-unstuck adversary
   → 現在の解を壊そうとする persona を有効化

6. /ooo-ralph 8
   → cap=8 内で evaluate → 失敗 AC 修正を自己反復
   → 全 pass で CONVERGED を出力し自動停止
```

## メモ

- ランタイム依存ゼロ。エージェントが Claude Code 組込みの `Read` / `Write` /
  `Edit` / `Bash` / `Grep` / `Glob` ツールで `.ouroboros/` 状態ファイルを直接管理。
- seed YAML は SKILL.md のスキーマに従いエージェントが手書き。常に
  double-quote スカラ、固定キー順序。読込は JSON サイドカーから行うので
  quoting バグでループが壊れない。
- Ralph はユーザ指定の N (デフォルト 8) でハード cap。AC を弱めて「収束」
  させることは絶対に無し — 正直に通せなければ止めて報告する。
