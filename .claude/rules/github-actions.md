---
description: .github/workflows/ を編集する際の方針。GitHub Actions のキャッシュ方針（容量 10 GB・効果の計測・保存は main の push のみ・キー設計・サイズ・計測コマンド・PR のチェックリスト）を定める。
paths:
  - ".github/workflows/**"
---

# GitHub Actions ルール

`.github/workflows/` を編集するときの方針。現在はキャッシュ方針のみを定める。

## キャッシュ方針

GitHub Actions のキャッシュ（`actions/cache`・`actions/setup-node` の `cache`・`Swatinem/rust-cache` 等）を
追加・変更するときの方針。効かないキャッシュは復元時間と容量を浪費し、他のキャッシュを追い出すため、
「入れれば速くなる」とは考えない。

### 現状（2026-10-11 時点）

- 本リポジトリは GitHub Actions のキャッシュを使っていない（キャッシュ 0 件・0 バイト）。
  `ci.yml` の `structure`・`skill-tests` は依存の導入が無く（Node.js は `ubuntu-latest` 同梱のものを使う）、
  `lint-docs` は Fandhe-AI/actions の reusable workflow に任せている
- スキルのテストに依存パッケージを足すなどでキャッシュを入れる場合は、本節に従う

### 前提: 容量と課金

- リポジトリあたりの上限は 10 GB（Org 設定も 10 GB）。超えると最終アクセスの古い順に追い出され、
  7 日間アクセスの無いキャッシュは削除される
- Org の Actions 予算は $0 で、超過時は停止する。そのため 10 GB を超える上限は使えない
  （上限を上げると、10 GB を超えた時点でキャッシュが読み取り専用になる）。上限・予算の変更はユーザーが判断する

### 入れる前に効果を測る

- self-hosted runner ではツールチェーン・pnpm store・`CARGO_TARGET_DIR` 等がジョブをまたいで残ることがあり、
  GitHub のキャッシュが無意味（復元時間だけかかる）な場合がある。本リポジトリは GitHub ホステッドだが、
  private の派生リポジトリで self-hosted を使う場合は特に確かめる
- 組織内の実例
  - mirror-ui: pnpm キャッシュの復元に 1 ジョブ平均 75 秒かけていたが、install は `downloaded 0` で効果が無かったため撤去した
  - team-hub: `Swatinem/rust-cache` が `./target` を保存対象にしていたが、runner の `CARGO_TARGET_DIR` が別の場所で
    registry しか保存されておらず、さらに後処理が共有 `CARGO_HOME` の crate を消していたため撤去した
- キャッシュを足す・残す前に、キャッシュ有無のジョブログで効果を確かめる。見るのはパッケージの `downloaded` 件数、
  依存の再ビルド数（`Compiling` の行数）、復元にかかった秒数

### 保存は main への push だけ

PR から保存したキャッシュは `refs/pull/<N>/merge` スコープに入り、その PR からしか読めないのに容量を共有し、
`main` のキャッシュを追い出す。保存は `main` への push に限る。

- `actions/cache` は restore（全イベント）と save（`main` の push のみ）に分ける

    ```yaml
    - name: Restore cache
      id: cache
      uses: actions/cache/restore@<sha> # vX.Y.Z
      with:
        path: <path>
        key: <key>
        restore-keys: <prefix>
    # ...ビルド・テスト...
    - name: Save cache
      if: github.event_name == 'push' && github.ref == 'refs/heads/main' && steps.cache.outputs.cache-hit != 'true'
      uses: actions/cache/save@<sha> # vX.Y.Z
      with:
        path: <path>  # restore と同じ
        key: ${{ steps.cache.outputs.cache-primary-key }}
    ```

- `Swatinem/rust-cache` は `save-if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}` を付ける
  （`github.ref` だけだと `main` 上の `schedule`・`workflow_dispatch` でも保存する。下の `cache-save` も同じ）
- Fandhe-AI/actions の `rust-base-ci.yml` を `cache: true` で呼ぶ場合は `cache-save: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}`
  を渡す（`cache` の既定は false、`cache-save` の既定は true のため、渡さないと PR でも保存する）
- `actions/setup-node` の `cache: pnpm` のように restore と save が一体の action は、PR でも保存する点に注意する

### キーの設計

- キーには中身のハッシュ（lockfile 等）を入れる。`github.sha`・`github.run_id` など毎回変わる値は入れない
  （入れる場合は restore-keys の前方一致で直近を拾う設計にし、保存は `main` に限る）
- `hashFiles('**/Cargo.lock')`・`hashFiles('**/pnpm-lock.yaml')` のような再帰 glob は使わず、ビルド対象に本当に入る
  ルートの lockfile を指定する。再帰 glob では `skills/*/` や `.agents/skills/*/samples/**` などスキルのサンプルの lockfile が混入し、
  依存が変わらないのにキーが変わって全キャッシュを作り直す（fandhe-browser・fandhe-frontend・fandhe-db・
  fandhe-container・template-frontend-react_router で発生）
- ビルド成果物（Rust の `target/` 等）をキャッシュするなら、キーにツールチェーンの版を入れる（`rustc -vV` の
  release・commit-hash、`dtolnay/rust-toolchain` の `cachekey` 出力等）。入れないと runner イメージの更新後も古い成果物に
  exact hit し続け、exact hit では再保存されないため依存を毎回全再ビルドする（fandhe-db で依存 135 crate を毎回再ビルドしていた）
- restore-keys で前方一致の部分復元を許し、キーが変わっても cold にならないようにする

### サイズを抑える

- 1 世代（キャッシュ種別ごとの最新 1 件）の合計が 10 GB に余裕をもって収まるようにする。
  キャッシュ種別を増やすときは「1 件のサイズ × 種別（ジョブ）× OS」を見積もる
- Rust なら保存前に workspace メンバーの成果物を prune する、`CARGO_INCREMENTAL=0`、CI 限定で
  `CARGO_PROFILE_DEV_DEBUG=line-tables-only` 等で縮める

### 追い出されても hit する設計

- 上限に達しても、毎回読まれる `main` の最新キャッシュは最後まで残る（最終アクセスの古い順に追い出されるため）。
  キーの切り替えで古い世代が追い出されても、新しい世代が保存済みならミスにならない
- 保存を `main` に限り、キーが不要に変わらない状態を保てていれば上限を上げる必要はない

### 計測コマンド

```bash
# 使用量（件数・バイト数）
gh api repos/{owner}/{repo}/actions/cache/usage
# 一覧（key・サイズ・ref・最終アクセス）
gh cache list -L 100
# 不要なキャッシュの削除（共有状態のため、key と ref を確認してから消す）
gh cache delete <id>
```

ジョブログでは次の行を見る。

- `Cache restored from key: <key>`: primary key と一致すれば exact hit、それ以外は restore-keys による部分復元
- `Cache not found for input keys: ...`: ミス
- `Cache saved with key: <key>`: 保存された（PR で出ていれば保存条件の誤り）

### キャッシュを追加・変更する PR のチェックリスト

PR 本文に次を確認したことを書く。

1. 効果を測った（キャッシュ有無の `downloaded` 件数・再ビルド数・復元秒数）
2. 保存が `main` への push に限られている
3. キーが中身のハッシュ（ルートの lockfile のみ）と、成果物を含むならツールチェーンの版でできており、restore-keys がある
4. 1 世代の合計サイズを見積もり、10 GB に余裕をもって収まる
