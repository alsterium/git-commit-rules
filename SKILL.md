---
name: git-commit-rules
description: Apply safe, atomic Conventional Commits to repository changes, including automatic commits after verified logical units while protecting pre-existing worktree changes and honoring repository-specific policies.
---

# Git Commit Rules

Git 作業を伴う実装・修正・文書化では、ユーザーがコミットを明示しなくても、検証済みの論理単位が完成するたびに安全にコミットする。コミットはこのスキルの目的であり、push や履歴改変まで自動化する許可ではない。

## 適用範囲と優先順位

- ファイルを変更しない説明・レビュー・調査・診断ではコミットしない。
- ユーザーの明示的な指示がある場合はそれを最優先する。「コミットしない」「push しない」などの指示を自動コミットより優先する。
- ユーザー指示の次に、リポジトリ固有の `AGENTS.md`、`CONTRIBUTING*`、commitlint 設定、フック、その他の開発規約を適用し、最後にこのスキルを適用する。競合を安全に解決できない場合は停止して報告する。
- 作業開始時にリポジトリルートを特定し、ルートから適用範囲にある `AGENTS.md`、`CONTRIBUTING*`、Git 設定、package manager のスクリプト、commitlint やフックの設定を読む。対象ディレクトリに近い規約を優先する。

## 作業開始時の安全確認と差分所有権

コミット前提の作業を始めるとき、次の状態を読み取り専用で記録する。

1. `git rev-parse --show-toplevel`、現在の HEAD、ブランチ名、`git status --short --branch` を確認する。
2. unstaged/staged の差分、既存の untracked path、対象ファイルの内容またはパッチを基準状態として記録する。
3. 基準時点で存在した差分・staged 変更・untracked ファイルは、ユーザー所有の保護対象として扱う。基準後に Codex が作成または変更したことを明確に示せる差分だけを候補にする。
4. コミット直前に HEAD、ブランチ、status、候補差分を再確認する。別プロセスやユーザーによる変更、ブランチ変更、所有権が不明な差分を検知したら停止する。

既存差分と Codex の差分が同じファイルに混在する場合は、パッチ単位で明確に分離できるときだけ対象ハンクを stage する。行の所有権を確信できない、同じハンクが混ざる、既存の staged 変更を壊さずに分離できない場合は、勝手に整形・stash・reset せず停止して報告する。

次の広い操作は禁止する。

- `git add .`、`git add -A`、`git commit -a`、無制限の glob でユーザー差分を取り込むこと
- `git reset --hard`、`git checkout`、`git restore`、stash などで既存差分を隠す・破棄すること
- 他の変更を取り込むための一括整形や無関係なリネーム

既存の index 変更がある場合は、実際の index を直接コミットに使わず、HEAD から作った一時 index に Codex 所有の明確なファイルまたはハンクだけを載せる方法を優先する。一時 index は作業終了後に削除し、実際の index にある保護対象を変更しない。対象ファイルに保護対象の差分が混在する場合はこの方法でも自動コミットせず停止する。

一時 index を使う場合は、HEAD を `git read-tree HEAD` で一時 index に読み込み、`GIT_INDEX_FILE=<temporary-index> git add -- <explicit-clean-paths>`、staged diff の確認、`GIT_INDEX_FILE=<temporary-index> git commit` の順に行う。コミット後は Codex が所有する clean path だけを実際の index に反映して index と HEAD を同期し、保護対象の status と diff が基準時点から変わっていないことを確認する。対象 path が clean でない、ハンクの分離が必要、またはフックが一時 index と互換であることを確認できない場合は停止する。

## 停止条件

次のいずれかに該当したら stage や commit を進めず、理由と安全に残っている状態だけを報告する。

- 対象差分を Codex の変更として特定できない、または保護対象と安全に分離できない
- 秘密情報・認証情報の疑いがある（例: `.env`、秘密鍵、token、password、credential、接続文字列）。値を出力・転載せず、path と疑いの種類だけを報告する
- `UU` などの merge conflict、未完了の merge/rebase/cherry-pick/revert、detached HEAD、想定外のブランチ変更など Git 状態が異常
- 破壊的変更の影響、`!`/`BREAKING CHANGE:`、移行手順が明確でない
- リポジトリ固有規約と共通規約の衝突を解決できない
- フック失敗に対して、原因を対象範囲内で修正して再実行しても解消しない
- コミットできる検証済みの論理単位がまだない

秘密情報や異常状態を理由に停止するとき、診断目的でも秘密値や機密ログを表示しない。

## 粒度とコミット頻度

- 1 コミット 1 目的の原子的な粒度にする。機能、バグ修正、リファクタリング、ドキュメント、テスト、設定変更など、独立して説明・検証できる目的は分ける。
- 実装と、その実装を直接検証するテストは同じ目的なら同じコミットに含めてよい。独立したテスト整理、文書化、フォーマットだけの変更は分ける。
- 複数目的の差分を見つけたら、目的ごとにファイルまたはハンクを分離し、各論理単位の検証が終わるたびにコミットする。
- WIP、作業途中、意味のない空コミットは原則作成しない。未完成の変更を残す必要がある場合は、ユーザーに状態を報告して次の指示を待つ。
- 関連するテスト・型チェック・Lint・ビルドを可能な範囲で実行する。安全性に問題がなく、失敗が既知または対象外であるためユーザーの目的に沿って進める場合は、失敗内容と実行できなかった検証を `検証:` に明記してコミットしてよい。危険な失敗、回帰、秘密情報、壊れたフックはこの例外に含めない。

## コミットメッセージ

Conventional Commits を使う。詳細な type、scope、本文、破壊的変更の例は [references/commit-message-policy.md](references/commit-message-policy.md) を、メッセージ作成時に必要な範囲だけ読む。

形式は次のとおり。

```text
<type>(<scope>): <日本語の短い命令形 summary>

変更:
- 何を変更したか

理由:
- なぜ変更したか

検証:
- 実行したコマンドと結果
```

- 基本 type は `feat`、`fix`、`refactor`、`docs`、`test`、`chore`、`style`、`perf`、`build`、`ci`、`revert` から選ぶ。リポジトリ固有の type が定義されていればそれを優先する。
- `scope` は原則必須とし、機能名、パッケージ名、サービス名、層、設定領域から最も具体的で安定したものを推定する。通常は小文字の kebab-case にする。意味のある scope がない場合は、規約が許せば省略し、許さなければ `repo` や `config` のような事実に即した中立 scope を使う。
- summary は日本語の短い命令形、72 文字以内、末尾に句点や `。` を付けない。可能なら Conventional Commit のヘッダー全体も 72 文字以内に収める。
- 本文は常に付け、空行の後に `変更:`、`理由:`、`検証:` の順で記載する。検証を実行できなかった場合も、その理由を `検証:` に書く。
- 破壊的変更は `<type>(<scope>)!:` を使い、本文に `BREAKING CHANGE:` と影響範囲・移行手順を記載する。影響や移行方法が不明なままコミットしない。

## 自動コミット手順

1. 変更目的を分類し、コミットする一つの論理単位を決める。作業開始時の保護対象を再確認する。
2. その単位に対する関連テスト、型チェック、Lint、ビルド、または最小限の検証を実行する。実行結果をメッセージ用に記録する。
3. `git diff --check` と対象差分の内容を確認し、秘密情報・意図しないファイル・保護対象がないことを確認する。stage 後にも `git diff --cached --check` と staged diff 全体を読む。
4. 明示的な path またはハンクだけを stage する。複数の目的が同じファイルにある場合はパッチ staging を使い、分離できなければ停止する。index に他の staged 変更がある場合は一時 index を使うか、保護できないなら停止する。
5. 上記の形式で type、scope、summary、本文を作成する。メッセージにテスト失敗を隠さない。
6. 通常の `git commit` を実行し、フックを必ず通す。`--no-verify` で回避しない。フックが失敗したら出力を読み、対象範囲内で修正して再実行する。解消しなければ停止条件に従う。
7. コミット後に `git show --stat --oneline HEAD`、`git status --short --branch`、必要なら `git diff HEAD` を確認し、コミット内容が対象単位だけで、基準時点の保護対象が維持されていることを確認する。誤った差分が入った場合は amend や reset で隠さず停止して報告する。
8. 次の論理単位があれば同じ手順を繰り返す。push は行わない。

amend、rebase、squash、履歴の書き換え、force push、push は、ユーザーが明示的に依頼した場合だけ扱う。通常の自動コミットでは実行しない。
