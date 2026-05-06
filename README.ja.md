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
| Interview  | `/interview <ゴール>` がエージェントに Socratic な質問を一度に一つずつ行うよう指示                    |
| Evaluate   | `/evaluate` がテストランナーを自動検出して実行し、コードベースを検査して各 AC を採点                  |
| Drift      | `/drift` が自己評価をリクエスト → `0.5*goal + 0.3*constraint + 0.2*ontology`                        |
| Unstuck    | `/unstuck [persona]` で 5 種類の lateral persona の一つを有効化 (state.json に記録)                  |
| Ralph      | `/ralph [N]` 単一コマンド内部で evaluate-and-fix の自己ループ (ハード cap 付き)                      |
| ハーネス    | Karpathy ガイドラインを `skills/clauroboros/SKILL.md` にバンドル — コーディングトリガーで auto-load   |

## スラッシュコマンド

| コマンド                  | 動作                                                              |
| ------------------------- | ----------------------------------------------------------------- |
| `/interview <ゴール>`      | ゴールを seed に結晶化する Socratic インタビューを開始              |
| `/seed`                   | ロック済み seed (または現在の draft) を出力                         |
| `/evaluate`               | mechanical テスト + 各 AC をファイルレベルの証拠で採点              |
| `/drift`                  | ロック済み seed に対する drift を自己評価                            |
| `/unstuck [id]`           | lateral persona 5 種類のいずれかに切り替え                          |
| `/ralph [on\|off\|N]`     | ハード cap 付きインライン evaluate-and-fix ループ                   |
| `/status`                 | interview / seed / persona / ralph / drift の状態                  |
| `/reset`                  | セッション状態をクリア (ロック済み seed は保持)                     |

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
│   ├── interview.md
│   ├── seed.md
│   ├── evaluate.md
│   ├── drift.md
│   ├── unstuck.md
│   ├── ralph.md
│   ├── status.md
│   └── reset.md
└── skills/
    └── clauroboros/
        └── SKILL.md
```

## ワークフロー例

```
1. /interview "TODO 管理 CLI を作る"
   → エージェントが一度に一つずつ Socratic インタビューを実施
   → 回答ごとに .ouroboros/state.json の seedDraft を更新
   → ambiguity ≤ 0.2 + AC ≥ 5 に到達したら seed.json + seed.yaml をロック

2. (コードを書く)

3. /evaluate
   → npm test / pytest / make test / cargo test / go test を自動検出 + 実行
   → 各 AC ごとにコードベースを検査し verdict (pass/fail/n-a) と証拠を記録
   → pass / fail の集計を出力

4. /drift
   → goal / constraint / ontology の divergence を自己評価
   → weighted ≤ 0.30 なら OK、超えると DRIFTED

5. (詰まったとき) /unstuck adversary
   → 現在の解を壊そうとする persona を有効化

6. /ralph 8
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
