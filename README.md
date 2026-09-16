# `churner-ai/release-workflow`

The reusable GitHub Actions workflow that drives a release's whole lifecycle
— cut a candidate, promote it to production, or roll production back — across
the two environments the
[Churner environment stack](../../infrastructure/customer/environment-stack/README.md)
created in your account, under your identity.

Apply the environment stack **twice**: once for your candidate environment
(`rc`) and once for `production`. Then call this workflow with both sets of
coordinates; each action uses the one it belongs to.

## Calling it

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      action:
        type: choice
        options: [cut-rc, promote, rollback]
        required: true
      target-tag:
        description: rollback only — leave empty to list available points
        required: false
        default: ''
      confirm-short-sha:
        description: promote only
        required: false
        default: ''

jobs:
  release:
    permissions:
      id-token: write     # mint the OIDC assertion for the deployer roles
      contents: read
    uses: churner-ai/release-workflow/.github/workflows/release.yml@v1
    with:
      project: MC
      action: ${{ inputs.action }}
      target-tag: ${{ inputs.target-tag }}
      confirm-short-sha: ${{ inputs.confirm-short-sha }}
      aws-region: us-east-2

      environment: production
      role-arn: arn:aws:iam::111122223333:role/churner-deployer-mc-production
      db-secret-arn: arn:aws:secretsmanager:us-east-2:111122223333:secret:rds!db-1111

      rc-environment: rc
      rc-role-arn: arn:aws:iam::111122223333:role/churner-deployer-mc-rc
      rc-codebuild-project: churner-mc-rc
      rc-db-secret-arn: arn:aws:secretsmanager:us-east-2:111122223333:secret:rds!db-2222

      # One list serves both: the candidate resolves the same NAMES under its
      # own prefix. `rc-secret-keys` overrides it, for the asymmetric case.
      secret-keys: STRIPE_SECRET_KEY SESSION_SECRET
      health-path: /api/health
    secrets:
      tracker-token: ${{ secrets.CHURNER_TRACKER_TOKEN }}
```

`permissions` goes on **your** job, not inside the reusable workflow — a
called workflow can only narrow what the caller was granted. Two things have
to be true or a role assume fails, both inherited from the environment
stack's trust policy:

- `id-token: write`, or there is no OIDC token to present;
- **the workflow runs from a ref this repository controls.** GitHub does not
  issue a writable id-token to a fork's `pull_request`, and the trust policy
  admits only this repository's own subjects — both the ref shape
  (`repo:<owner>/<repo>:ref:refs/*`) and the one a job bound to a GitHub
  environment presents (`repo:<owner>/<repo>:environment:*`).

`promote` and `rollback` run their job under `environment: <your production
environment's name>`, so a GitHub environment protection rule (required
reviewers, a wait timer) holds them for approval — the Actions run shows
"Waiting for approval…" rather than failing. **Create that environment in
your repository's settings and put the reviewers on it**: without it the gate
is a name with no rule behind it, and the job runs straight through.
`cut-rc` carries no gate — it deploys to the candidate host, where nothing it
does reaches a real user.

## Two environments, one artefact

`rc` and `production` are separate created environments — separate hosts,
databases, registries and deployer roles (design spec §7.2/§7.4).

- **`cut-rc`** builds the given `sha` on the CANDIDATE environment's
  CodeBuild project, tags it `rc-<sha>` (or `image-tag`, if set), and deploys
  it on the CANDIDATE host. Production is untouched, and this job's
  credentials cannot reach it.
- **`promote`** reads what is CURRENTLY RUNNING on the candidate host,
  refuses unless its short sha matches `confirm-short-sha`, **copies that
  exact image** into production's repository as `prod-<sha>`, and deploys it
  on the PRODUCTION host. The copy is a copy — the manifest, then every blob
  production does not already hold — because `ecr:PutImage` validates that a
  manifest's layers exist in the repository it is written to, so a
  manifest-only retag cannot cross two repositories. Rebuilding from the same
  commit would have been simpler and would promote a *different* artefact.
- **`rollback`**, given a `target-tag`, redeploys it on the production host.
  Left empty, it runs the `list` job instead: no deploy, just the available
  rollback points printed to the run summary.

### What a rollback point is

`deploy-release.sh` tags the OUTGOING image `<environment>-prev-<unix-ts>` on
the host, right before removing the container a new release replaces. That
tag is **host-local**: it is never pushed, so it survives a bad deploy but
not a replaced host, and `deploy-release.sh` falls back to the local image
when a pull of it fails. What survives a host is the `prod-<sha>` tags in
production's own repository — every image ever promoted — and any of those is
equally valid as a `target-tag`.

The `list` job reads the host's own image list, scoped to this environment's
own repository, over the same `ssm:SendCommand` every other action uses. No
new AWS grant.

## Inputs

| Input | Required | Default | Meaning |
|---|---|---|---|
| `project` | yes | — | Churner project key, e.g. `MC`. |
| `action` | yes | — | `cut-rc`, `promote`, or `rollback`. |
| `aws-region` | yes | — | Region both environment stacks live in. |
| `environment` | no | `production` | The PRODUCTION environment's name, as you named it in the wizard. Also the GitHub environment whose protection rules gate `promote`/`rollback`. |
| `role-arn` | yes | — | Production's `churner-deployer-<projectKey>-<environment>` role. Assumed by `promote`, `rollback`, `list`. |
| `db-secret-arn` | no | `''` | Production's RDS-managed master secret ARN (`ChurnerEnv<Name>DbSecretArn` in the stack outputs). The HOST reads it; this workflow only names it. Empty for an environment with no database. |
| `host-instance-id` | no | `''` | Override for production's host. Leave it empty — the host is found by its tag. |
| `rc-environment` | no | `rc` | The CANDIDATE environment's name. |
| `rc-role-arn` | yes | — | The candidate's deployer role. Assumed by `cut-rc`, and by the one `promote` step that reads what rc is running. |
| `rc-codebuild-project` | yes | — | The candidate's CodeBuild project. Read only by `cut-rc`. |
| `rc-db-secret-arn` | no | `''` | The candidate's own database secret ARN. |
| `rc-host-instance-id` | no | `''` | Override for the candidate host. |
| `secret-keys` | no | `''` | Space-separated NAMES of the application secrets the PRODUCTION container needs. Names only — see below. |
| `rc-secret-keys` | no | `''` | The same, for the CANDIDATE container. **Leave it empty and the candidate gets `secret-keys`** — the same names, under rc's own prefix. Set it only for the asymmetric case. |
| `image-tag` | no | `''` | Override for the tag `cut-rc` builds under. Leave empty — the default is `rc-<sha>`. |
| `sha` | no | `''` | Commit to build (`cut-rc`). Leave empty to use this run's own ref head. |
| `confirm-short-sha` | no | `''` | `promote` only, and required for it. The 7-character prefix of the commit you mean to promote, checked against what is ACTUALLY running on the candidate. |
| `target-tag` | no | `''` | `rollback` only. The tag to redeploy. Leave EMPTY to run the `list` job instead. |
| `health-path` | no | `/` | Rooted path the readiness gate polls on the candidate release's own port before it is swapped live. |
| `tracker-url` | no | `https://churner.ai` | Base URL of your Churner instance. https only. |
| `scripts-base-url` | no | `churner-ai/environment-stack` @ `refs/tags/v1` | Where `host/deploy-release.sh` and `scripts/copy-image.sh` are fetched from — the SAME base URL the environment host's own `bootstrap.sh` uses. Tags are immutable once published. |

| Secret | Required | Meaning |
|---|---|---|
| `tracker-token` | no | Bearer token for `POST /api/internal/deployment-event`. Leave it unset and every report step warns and skips — a missing token never fails a deploy. |

## Application secrets

`secret-keys` and `rc-secret-keys` carry **names, never values**. Each name is
read on the HOST it belongs to, under that host's own instance profile, from
`<that environment's secrets prefix>/<NAME>` in Secrets Manager, and passed to
the container as `-e NAME` — so no value reaches this workflow's logs or
docker's argv. The names are usually the same on both sides; the VALUES are
not, because each environment resolves them under its own prefix.

The convention the scaffolded caller renders them from is a file in your
repository:

```
# .churner/release/secrets
# One environment-variable name per line. Comments and blank lines ignored.
STRIPE_SECRET_KEY
SESSION_SECRET
```

**Leave it empty and the candidate gets `secret-keys`** — the same names,
resolved on rc's own host under rc's own prefix. A
`.churner/release/secrets.rc` beside the file above, when the repository
carries one, is what the scaffolded caller renders `rc-secret-keys` from
instead; that is the asymmetric case, and it is the only reason the two are
separate inputs.

**Store each name under BOTH environments' prefixes before the deploy that
names it** — `<production prefix>/STRIPE_SECRET_KEY` *and*
`<rc prefix>/STRIPE_SECRET_KEY`. A name the host cannot read is a hard
failure, deliberately, because a container silently missing its configuration
fails somewhere far less obvious. That is also why the two lists are separate
inputs rather than one: a production-only secret added to a single shared list
would break the next `cut-rc`, which is a deploy to the candidate failing over
a name that has nothing to do with it.

`DATABASE_URL` is not one of these — the host composes it from the
RDS-managed secret named by `db-secret-arn` (or `rc-db-secret-arn`).

## Deployer role grants

Every AWS action this workflow calls is one the relevant environment's
deployer role grants, and nothing more
(`infrastructure/customer/environment-stack/README.md` → "The boundary"):

```
codebuild:StartBuild / codebuild:BatchGetBuilds     cut-rc's build           (rc role)
ssm:DescribeInstanceInformation                     find a host by tag       (both)
ssm:SendCommand / ssm:GetCommandInvocation          run the host script      (both)
ecr:BatchGetImage / GetDownloadUrlForLayer          read rc's image          (prod role, on rc's repo)
ecr:BatchCheckLayerAvailability                     skip what is already there
ecr:InitiateLayerUpload / UploadLayerPart / CompleteLayerUpload / PutImage
                                                    write it into production (prod role, own repo)
```

`ecr:GetAuthorizationToken` is also granted (resourceless) but this workflow
never calls it: docker authentication happens ON THE HOST, under its own
instance profile, inside `deploy-release.sh`. A test asserts that set both
ways — every call granted, on the repository it names, and no grant unused
except that one.

## The two scripts

`host/deploy-release.sh` (run on the environment host, by SSM) and
`scripts/copy-image.sh` (run on the runner, by `promote`) are fetched fresh
from `scripts-base-url` on every run and checksum-verified before they
execute — never heredocs embedded in this workflow, so each can be tested on
its own. `shared/tests/release-workflow.test.ts` EXECUTES both, against
shimmed `docker` / `aws` / `systemctl` / `curl`.

`deploy-release.sh` reads the database secret and every name in `secret-keys`
entirely on the host, starts the new release on a second port, and swaps the
proxy's route only after that release answers its own health check — so a
release that fails its health gate is a no-op: the previous one keeps serving
and nothing is swapped.

## Reporting

Every action reports to `POST /api/internal/deployment-event` under the env
keys Churner's Build tab renders: **`rc`** for `cut-rc`, **`prod`** for
`promote` and `rollback`. Those are Churner's env keys, not the AWS
environment names above — a project whose production environment is called
`production` still reports `prod`.

A report step runs only when the steps before it SUCCEEDED. The event schema
carries no status field, so reporting after a failed build or deploy would
record something that did not happen; a failed run is visible as a failed run
— the Build tab reads the workflow run itself. The POST can never fail a
deploy either way: each step is `continue-on-error: true` and short-circuits
with a `::warning::` when no token is configured.

If you want the Build tab's live-status column (not just the recorded
history), have your deployed application serve:

```
GET /api/config  ->  { "buildSha": "<sha>" }
```

The recorded column works without it.

## Container labels

Every release container carries three labels — the record `promote` reads to
find out what the candidate is actually running:

```
docker run -d \
  --label churner.release=true \
  --label churner.release.tag=<image tag, e.g. rc-abc1234> \
  --label churner.release.sha=<full commit sha> \
  ...
```
