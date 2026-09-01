# PostyLoop 更新手順書

この文書は、公式 LINE Harness を基礎にした PostyLoop fork を安全に更新するための運用手順です。

## 正本と対象

- 公式本家: `https://github.com/Shudesu/line-harness-oss`
- PostyLoop fork: `https://github.com/ktono1018/line-harness-oss`
- 公式 remote: `upstream`
- PostyLoop remote: `origin`
- Cloudflare Worker: `line-harness`
- Worker URL: `https://line-harness.posty-loop.workers.dev`
- Admin Pages project: `line-harness-admin-b2a92522`
- Admin URL: `https://line-harness-admin-b2a92522.pages.dev`
- D1: `line-harness`
- R2: `line-harness-images`

同名の別リポジトリ、検索結果、古い fork、記憶上の仕様は正本として使いません。

## 絶対に守ること

1. 作業開始時に公式 URL、`git remote -v`、現在 branch、`git status`を確認する。
2. 未commit変更がある場合は、stash・reset・上書きをせず停止する。
3. 公式変更は直接`main`へ入れず、更新用branchとPRを経由する。
4. 本番デプロイ前にローカルbuild/testとrollback先を確認する。
5. D1 migrationを伴う場合は、先にremote D1をexportする。
6. push、PR作成、merge、本番デプロイはそれぞれ実行直前に承認を取る。
7. API key、LINE token、Channel Secret、Cloudflare tokenをログ・PR・文書へ書かない。

## 1. 作業開始チェック

```bash
cd /Users/ktono1018/Documents/codex/line-harness-postyloop
git remote -v
git branch --show-current
git status --short --branch
git ls-remote https://github.com/Shudesu/line-harness-oss.git refs/heads/main
```

期待するremote:

```text
origin   https://github.com/ktono1018/line-harness-oss.git
upstream https://github.com/Shudesu/line-harness-oss.git
```

`git status`に変更が表示された場合は、以降へ進みません。

## 2. 公式更新をforkへ取り込む

### 推奨: GitHub Actionsが作成するPRを使う

forkの`Update from upstream` Workflowは、毎日および手動実行で公式`upstream/main`との差を確認し、更新用PRを作ります。

1. GitHubのActionsで`Update from upstream`を開く。
2. 必要なら`Run workflow`を実行する。
3. 作成された`chore: update from upstream` PRの差分を確認する。
4. PostyLoop表示箇所との競合を確認する。
5. ローカル検証が終わるまでmergeしない。

### 手動で更新branchを作る場合

```bash
git fetch upstream main
git fetch origin main
git switch -c codex/upstream-update-YYYYMMDD origin/main
git merge upstream/main
```

競合が出た場合は、その場で機械的に解消しません。特に次を確認します。

- `apps/web/src/app/layout.tsx`
- `apps/web/src/app/login/page.tsx`
- `apps/web/src/components/layout/sidebar.tsx`
- `packages/create-line-harness/src/commands/setup.ts`
- `packages/create-line-harness/src/commands/update.ts`
- `packages/create-line-harness/src/steps/clone-repo.ts`
- `packages/mcp-server/src/index.ts`

内部識別子の`create-line-harness`、`@line-harness/*`、API route、D1/R2 bindingは変更しません。

## 3. 変更内容を分類する

| 変更範囲 | 必要な公開作業 | D1 backup |
|---|---|---|
| `apps/web/**`だけ | Admin Pagesだけ | 不要 |
| `apps/worker/src/client/**` | Worker Assetsを含むWorker | 通常不要 |
| Worker API・Webhook・Cron | Worker | 推奨 |
| `packages/db/migrations/**` | D1 migration後にWorker | 必須 |
| CLI・MCPだけ | npm公開または利用環境更新 | Cloudflare変更なし |
| WorkerとAdminの両方 | Worker→Adminの順 | 変更内容による |

PostyLoopの表示名だけを変更する場合、通常はAdmin Pagesだけを更新します。Workerをソース版へ差し替えると、公式自動更新のfork判定に影響するため避けます。

## 4. ローカル検証

公式`package.json`は`pnpm@9.15.4`を指定しています。別majorのpnpmを使う場合は、依存build承認などの挙動差に注意します。

```bash
pnpm install --frozen-lockfile
pnpm build
pnpm --filter worker typecheck
pnpm -r test
git diff --check
git status --short --branch
```

Adminだけを検証する場合:

```bash
NEXT_PUBLIC_API_URL=https://line-harness.posty-loop.workers.dev pnpm --filter web test
NEXT_PUBLIC_API_URL=https://line-harness.posty-loop.workers.dev pnpm --filter web build
rg -l 'PostyLoop' apps/web/out
rg 'L Harness' apps/web/out
```

最後の検索は、意図しない旧表示名が公開成果物へ残っていないか確認するためです。

## 5. D1 backup

D1 migrationまたはWorkerのDB書き込み処理を変更する場合だけ実行します。

```bash
mkdir -p backups
npx wrangler d1 export line-harness \
  --remote \
  --output backups/line-harness-before-update-YYYYMMDD-HHMM.sql
```

backupファイルはGitへ追加しません。容量とSQL先頭を確認し、空ファイルでないことを確認します。

## 6. Admin Pagesだけを公開する

```bash
NEXT_PUBLIC_API_URL=https://line-harness.posty-loop.workers.dev \
  pnpm --filter web build

CLOUDFLARE_ACCOUNT_ID=<snsアカウントID> \
  npx wrangler pages deploy apps/web/out \
  --project-name line-harness-admin-b2a92522 \
  --branch main \
  --commit-dirty=true
```

公開後の確認:

```bash
curl -fsSL https://line-harness-admin-b2a92522.pages.dev/login \
  | rg -o '<title>[^<]+|PostyLoop|L Harness'
```

管理画面で次を目視確認します。

- ログイン画面が`PostyLoop`
- ログイン後のサイドバーが`PostyLoop`
- 既存LINEアカウント、友だち、タグ、シナリオが残っている
- Worker URLが変わっていない

## 7. Workerを公開する場合

Worker公開はAdmin表示変更より影響が大きいため、次を満たす場合だけ行います。

- Worker変更が目的に必要
- D1 migrationの有無を確認済み
- 必要ならD1 backup済み
- `wrangler.toml`のD1/R2/Assets/Cron/DO bindingを確認済み
- 全Workerテストとtypecheckが成功
- 直前Worker deployment IDを記録済み

forkのGitHub Actionsを使う場合、以下が必要です。

- Repository Variable: `LINE_HARNESS_CLOUDFLARE_DEPLOY=true`
- Secrets: Cloudflare token、account ID、D1名・ID、Admin API URLなど
- Variables: Worker名、LIFF ID、Bot Basic ID、Admin originなど

Workflowはforkの`main`へWorker関連変更が入った場合だけ実行されます。公式本家ではCloudflare deploy jobは実行されません。

## 8. 公式CLI updateを使う判断

```bash
npx create-line-harness@latest update
```

このコマンドは公開Workerの`/admin/version`と公式manifestを照合します。

- 公式version・hashと一致: 自動更新可能
- 既知versionでhash不一致: カスタム版として自動更新停止
- `0.0.0-dev`: 旧CLI環境として公式版への移行確認が出る

PostyLoopのAdmin Pagesだけをカスタマイズしている場合でも、公式updateはAdminを公式bundleで上書きする可能性があります。公式update後は、PostyLoop branchを最新公式版へ追従させ、Admin test/build後にPagesだけ再デプロイします。

## 9. Rollback

### Admin Pages

Cloudflare Pagesの直前Production deploymentを確認し、そのdeploymentへrollbackします。新規デプロイ前に必ず直前IDを記録します。

```bash
CLOUDFLARE_ACCOUNT_ID=<snsアカウントID> \
  npx wrangler pages deployment list \
  --project-name line-harness-admin-b2a92522
```

### Worker

```bash
npx wrangler deployments list --name line-harness --json
```

直前の安定versionを確認してからCloudflareのrollback機能を使います。D1 migrationはWorker rollbackだけでは戻りません。DBを戻す必要がある場合は、事前backupとmigration内容を確認して別途復旧します。

### Git

- merge前: branchを使わず終了
- merge後: 対象commitを`git revert`するPRを作る
- `git reset --hard`や履歴改変pushは使わない

## 10. 完了報告

更新後は以下を一画面で記録します。

- 公式main commit
- PostyLoop main commit
- 変更した範囲
- 実行したtest/build
- Cloudflare deployment ID
- 変更しなかった対象（Worker、D1、R2、Secretsなど）
- rollback先
- 残っている実機確認
