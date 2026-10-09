# CLAUDE.md

このリポジトリは [contabo/terraform-provider-contabo](https://github.com/contabo/terraform-provider-contabo) の fork。
upstream へ PR を提出しつつ、マージされるまでの間 Terraform Registry に `takumin/contabo` として独自リリースを公開する。

## ブランチ

| ブランチ | 役割 | 起点 |
|---|---|---|
| `main` | `upstream/main` の追従専用。独自コミットは入れない | `upstream/main` |
| `feat/*`, `fix/*` | upstream への PR 用。upstream にそのまま出せる差分のみ | `upstream/main` |
| `release` | 独自リリース用 (デフォルトブランチ)。`upstream/main` + PR ブランチの merge + fork 専用の変更 | `upstream/main` |

- PR ブランチに fork 専用の変更 (`CLAUDE.md`, `.goreleaser.yml` の owner 等) を混入させない
- fork 専用の変更は `release` にのみコミットする
- PR は `contabo/terraform-provider-contabo:main` を base にして作成する

## リモートとタグ

- `origin`: `takumin/terraform-provider-contabo`
- `upstream`: `contabo/terraform-provider-contabo` (`remote.upstream.tagOpt --no-tags` を設定済み)
- upstream のタグを origin に push しない (`v*` タグの push でリリース workflow が走るため)
- `git push --tags` は使わず、タグは個別に push する

## バージョニング

- 日付ベースの `vYYYY.M.D` (例: `v2026.10.10`)
- SemVer の制約上ゼロ埋めしない (`v2026.10.01` は不可、`v2026.10.1` とする)
- 同日に複数回リリースしない。必要になった場合は翌日の日付を使う
- タグは `release` ブランチ上でのみ打つ

## 手順

### upstream への追従

```zsh
git fetch upstream
git switch main && git merge --ff-only upstream/main && git push origin main
git switch feat/xxx && git rebase upstream/main && git push --force-with-lease origin feat/xxx
git switch release && git merge upstream/main && git merge feat/xxx
```

### 独自リリース

```zsh
git switch release
git tag v$(date +%Y.%-m.%-d)
git push origin release v$(date +%Y.%-m.%-d)
```

タグの push で `.github/workflows/release.yaml` が goreleaser を実行し、GitHub Release 経由で Terraform Registry に取り込まれる。
GPG 署名用に fork 側の Secrets (`GPG_PRIVATE_KEY`, `PASSPHRASE`) と Registry への公開鍵登録が必要。

### 利用側

```hcl
terraform {
  required_providers {
    contabo = {
      source  = "takumin/contabo"
      version = "~> 2026.10"
    }
  }
}
```

```zsh
terraform state replace-provider registry.terraform.io/contabo/contabo registry.terraform.io/takumin/contabo
```

### upstream でマージされた後

1. `release` から該当 PR ブランチの差分が `upstream/main` に含まれたことを確認する
2. 利用側の `source` を `contabo/contabo` に戻し、`terraform state replace-provider` を逆向きに実行する
3. 独自で抱える差分がなくなったら Registry 上の `takumin/contabo` を deprecated にする

## 現在 upstream に提出中/予定の変更

- オブジェクトストレージバケットの import 対応
