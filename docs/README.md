<!-- owner: Automata | zone: technical | purpose: Canonical technical doc index for shikanime-labs/xqbit -->
# xqbit — technical index

Extra Go tooling for qBittorrent. The repository ships small CLIs that manage torrents through the qBittorrent WebUI API, with a Skaffold build pipeline.

## Canonical sources

- [Repository README](https://github.com/shikanime-labs/xqbit#readme)
- [Technical wiki](https://github.com/shikanime-labs/xqbit/wiki)

## Tools

### qbittorrent-cleanup

`cmd/qbittorrent-cleanup/` (with `main.go` and `main_test.go`) removes `missingFiles` torrents and resumes recoverable errored torrents by talking to the qBittorrent WebUI API.

Configuration is via flags or `QBT_`-prefixed environment variables:

| Flag | Env |
| --- | --- |
| `--url` | `QBT_URL` |
| `--user` | `QBT_USER` |
| `--password` | `QBT_PASSWORD` |
| `--timeout` | `QBT_TIMEOUT` |

```bash
go run ./cmd/qbittorrent-cleanup \
    --url https://qbittorrent.example \
    --user alice \
    --password '...'
```

## Build and checks

`skaffold build` builds the container image; `go test ./...` runs the checks. Enter the toolchain with `direnv allow` or `nix develop`.

## Repository layout

A Go module with vendored dependencies: `cmd/qbittorrent-cleanup/`, `skaffold.yaml`, `flake.nix` (devenv shell), `.github/workflows/` (integration, nix, release, skaffold), and `vendor/` (vendored Go dependencies such as fsnotify).

## Documentation rule

In-repo `docs/` is canonical for technical detail. The [wiki](https://github.com/shikanime-labs/xqbit/wiki) is a rendered human index that links here.
