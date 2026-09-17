# This repository is published, not authored

`.github/workflows/release.yml` and `README.md` are copied from
`github-actions/release-workflow/` in the churner monorepo, which is where
they are edited, reviewed and tested. Do not patch them here: the next
release overwrites the file, and the change would never have run against the
workflow's test suite (`shared/tests/release-workflow.test.ts`, which
EXECUTES this workflow's own SSM step and both host scripts against shimmed
`docker` / `aws` / `systemctl` / `curl`).

The two scripts this workflow fetches at run time are **not** in this
repository. They come from `churner-ai/environment-stack@v1` — the
public repository an environment host also fetches `bootstrap.sh` from, and
the tag this workflow's own `scripts-base-url` default names — and are
verified against the SHA-256 pins in this workflow's `env:` block before
either one runs.

Released from churner monorepo commit `f75e189`.
