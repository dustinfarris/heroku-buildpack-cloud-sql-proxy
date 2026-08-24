# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Heroku **classic buildpack** (not CNB) that vendors the Google Cloud SQL Auth Proxy binary into the slug and generates a startup script the app invokes from its Procfile. The whole buildpack is three POSIX `sh` scripts in `bin/`. There is no build step, no dependency manifest, no linter, and no test suite.

## Commands

There is nothing to build or test locally — the scripts only run inside a Heroku build. Verify changes by deploying:

```sh
heroku buildpacks:add https://github.com/dustinfarris/heroku-buildpack-cloud-sql-proxy -a <app>
git push heroku master           # watch the "-----> " lines from bin/compile
heroku run 'cat bin/run_cloud_sql_proxy' -a <app>   # inspect the generated script
heroku run 'bin/run_cloud_sql_proxy' -a <app>       # smoke-test the proxy start
```

To exercise `bin/compile` without Heroku, call it with three directories directly: `bin/compile <build_dir> <cache_dir> <env_dir>` (it downloads a linux/amd64 binary, so `--version` will fail on macOS — the download and slug layout are still checkable).

## Architecture

**Buildpack API contract.** `bin/detect` always succeeds (this buildpack applies to every app). `bin/compile $BUILD_DIR $CACHE_DIR $ENV_DIR` does all the work. `bin/release` is a required no-op.

**Two-phase design — this is the key thing to understand.** `bin/compile` runs at *build* time but its main output is a shell script that runs at *dyno boot* time:

1. Build time: download `cloud-sql-proxy` (version pinned by `VERSION` in `bin/compile`) into `$CACHE_DIR/cloud_sql_proxy_$VERSION`, then copy it to `$BUILD_DIR/google/bin/cloud_sql_proxy`.
2. Build time: emit `$BUILD_DIR/bin/run_cloud_sql_proxy` via a heredoc.
3. Runtime: the app's Procfile runs `bin/run_cloud_sql_proxy`, which base64-decodes `$GCLOUD_CREDENTIALS` to `/app/google/credentials.json` and starts the proxy in the background.

**Heredoc escaping is the main hazard.** The generated script is written with an *unquoted* `<<EOF`, so the outer shell expands variables as it writes. Anything meant for runtime must be escaped (`\$GCLOUD_CREDENTIALS`, `\$GCLOUD_INSTANCE`); anything meant to be baked in at build time is left unescaped. Getting this backwards silently produces an empty variable in the slug.

**`$BUILD_DIR` becomes `/app` at runtime.** The generated script hardcodes `/app/google/bin/cloud_sql_proxy` and `/app/google/credentials.json`. If you change where the binary is vendored under `$BUILD_DIR`, change the hardcoded `/app/...` paths in the heredoc to match.

**`$ENV_DIR` is deliberately unused.** Config vars are never read at build time; `GCLOUD_CREDENTIALS` and `GCLOUD_INSTANCE` are consumed only by the runtime script from the dyno environment. Keep it that way unless a change genuinely needs a value during the build.

**Cache key.** `$CACHED_BINARY` embeds `$VERSION`, so bumping `VERSION` naturally invalidates the cache and forces a fresh download. Never drop the version from that filename.

## `GCLOUD_INSTANCE` syntax

v2 takes the instance connection name as a **positional argument**, not `-instances=NAME=tcp:PORT`. The port defaults to the database's default (5432 for Postgres) and is overridden per-instance with `?port=N` or globally with `--port`. `README.md` documented the v1 form until it was corrected; don't reintroduce it.

Other v1 to v2 flag renames already applied in `bin/compile`: `-credential_file` -> `--credentials-file`, `-instances=` -> positional args.

Note that `$GCLOUD_INSTANCE` is interpolated unquoted in the generated script, so a value containing `?port=` is a glob pattern as far as `sh` is concerned. It is harmless today (nothing in `/app` matches), but quote the expansion if that heredoc line is ever touched.
