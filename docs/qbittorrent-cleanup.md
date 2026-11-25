# qbittorrent-cleanup

`cmd/qbittorrent-cleanup` is a single-shot CLI that reconciles a qBittorrent
instance over its WebUI API: it deletes torrents whose files are gone and
resumes errored torrents that can recover. It runs to completion and exits;
scheduling is the operator's concern (cron, a systemd timer, a Kubernetes
CronJob).

## Jobs

The binary runs named jobs. Two exist, declared as `jobDeleteMissingFiles` and
`jobResumeErrored` in `cmd/qbittorrent-cleanup/main.go`:

| Job                  | Behaviour |
| -------------------- | --------- |
| `delete-missing-files` | Lists every torrent and keeps those in the `missingFiles` state, then deletes them with their files. |
| `resume-errored`     | Lists the `errored` filter and resumes every entry except those in the `missingFiles` state, which the other job owns. |

`delete-missing-files` deliberately fetches the unfiltered torrent list and
filters client-side instead of trusting `?filter=errored`: some qBittorrent
versions omit `missingFiles` torrents from that filter, so the filtered call
silently under-reports.

With no `--job` flag both jobs run, in that order. An unrecognised job name
prints `Unknown job: <name>` and is skipped.

## Configuration

Flags are bound through viper with the `QBT_` environment prefix, so every flag
has an env equivalent (`--url` maps to `QBT_URL`, and so on).

| Flag        | Env            | Default           | Meaning |
| ----------- | -------------- | ----------------- | ------- |
| `--url`     | `QBT_URL`      | —                 | qBittorrent WebUI base URL (required) |
| `--user`    | `QBT_USER`     | —                 | WebUI username (required) |
| `--password`| `QBT_PASSWORD` | —                 | WebUI password (required) |
| `--timeout` | `QBT_TIMEOUT`  | `30s`             | Deadline for the whole run, applied as a `context.WithTimeout` |
| `--job`     | `QBT_JOB`      | both jobs         | Repeatable job selector: `--job=delete-missing-files --job=resume-errored` |

A missing `--url`, `--user` or `--password` is fatal before any request is made.

```bash
go run ./cmd/qbittorrent-cleanup \
    --url https://qbittorrent.example \
    --user alice \
    --password '...' \
    --job=delete-missing-files
```

## WebUI API surface

The client is a thin wrapper over `net/http` with a cookie jar, because the
WebUI authenticates by session cookie rather than a bearer token.

| Call | Endpoint | Notes |
| ---- | -------- | ----- |
| `Login` | `POST /api/v2/auth/login` | Form-encoded `username`/`password`. Accepts `200` and `204` as success — qBittorrent replies `204` on some versions. |
| `GetTorrents` | `GET /api/v2/torrents/info` | Optional `?filter=`; decodes `hash`, `state`, `name`. |
| `DeleteTorrents` | `POST /api/v2/torrents/delete` | Form-encoded `hashes` joined with `\|` and `deleteFiles=true`. |
| `Resume` | `POST /api/v2/torrents/start` | Form-encoded `hashes` joined with `\|`. |

## Exit behaviour

Only an authentication failure fails the run. `Login` returns an error,
`runCleanup` propagates it, `main` exits `1` — so a scheduler sees a non-zero
status instead of a silent success when credentials rot.

Everything after login is best-effort: a fetch, delete or resume error is
printed and the job returns without aborting the process, and the remaining job
still runs. A run that found nothing to do exits `0`.

## Build and test

```bash
go test ./...      # unit checks against the API client
skaffold build     # ko build of ghcr.io/shikanime-labs/xqbit/qbittorrent-cleanup
```

`skaffold.yaml` targets `linux/amd64` and `linux/arm64`. Dependencies are
vendored under `vendor/`, so builds need no network fetch. The Nix dev shell
(`nix develop` or `direnv allow`) provides the Go toolchain.
