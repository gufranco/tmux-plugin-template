# tmux-plugin-template

A starting point for fast, non-blocking tmux status plugins.

Every status plugin built from this template shares one rule: the status render
never waits on slow work, and nothing is written to a temp file. A cached value
lives in a tmux server user-option and is read instantly. When it goes stale, a
detached background worker recomputes it and writes it back. The result is a
status bar that stays responsive even when the underlying probe is slow, such as a
CPU sample or a network call.

## What you get

- `src/lib/utils/cache.sh`: the async, temp-file-free cache.
- `src/lib/utils/platform.sh`: memoized OS detection.
- `src/lib/tmux/tmux-ops.sh`: tmux option get, set, and unset.
- `src/lib/utils/has-command.sh`: command availability probe.
- `src/lib/utils/error-logger.sh`: opt-in logging, off by default.
- `src/lib/utils/constants.sh`: shared defaults.
- `test/helpers.bash`: unit harness with an in-memory tmux mock.
- `test/tmux_helpers.bash`: integration harness with a real tmux socket.
- `Makefile`, `.github/workflows/tests.yml`: test, lint, and a kcov coverage gate.

## How the cache works

A plugin sets `CACHE_PREFIX` to namespace its options, then drives every
placeholder through the cache. The worker computes a raw value once; the
placeholder scripts are pure mappers that read it.

```bash
CACHE_PREFIX="cpu_revamped"
source "src/lib/utils/cache.sh"

# Worker: the only place slow work happens. Runs in the background.
refresh_cpu() {
  local pct
  pct=$(read_cpu_percentage)     # may sample for a moment; off the hot path
  cache_set percent "${pct}"
}

# Hot path: trigger a refresh when stale, then echo the cached value at once.
cache_render percent 5 refresh_cpu
```

State for key `percent` under prefix `cpu_revamped` lives in:

| Option | Meaning |
|--------|---------|
| `@cpu_revamped_percent` | the value |
| `@cpu_revamped_percent_ts` | unix epoch of the last write |
| `@cpu_revamped_percent_lock` | `1` while a worker runs |
| `@cpu_revamped_percent_lock_ts` | unix epoch the lock was taken |

The cache API:

| Function | Purpose |
|----------|---------|
| `cache_get KEY` | echo the cached value, empty when unset |
| `cache_set KEY VALUE` | store a value and stamp the write time |
| `cache_age KEY` | seconds since the last write |
| `cache_is_fresh KEY MAX_AGE` | true when within MAX_AGE seconds |
| `cache_should_refresh KEY MAX_AGE` | true when stale and unlocked |
| `cache_refresh_if_stale KEY MAX_AGE WORKER` | background-refresh when stale |
| `cache_render KEY MAX_AGE WORKER` | refresh when stale, then echo the value |

Set `CACHE_SYNC=1` to run the worker inline instead of detached. Tests use this.

## Create a plugin from this template

1. On GitHub, choose "Use this template" to generate a new repository, or
   `gh repo create <owner>/tmux-<metric>-revamped --template <owner>/tmux-plugin-template`.
2. Set `CACHE_PREFIX` and the logging namespace for your metric in the dispatcher.
3. Add `src/lib/<metric>/<metric>.sh` with pure, per-platform parsers.
4. Add `src/<plugin>.sh` as the dispatcher and `<plugin>-revamped.tmux` as the
   entry point that registers your status placeholders.
5. Add one bats file per module under `test/lib/`, keeping coverage at 95%.
6. To pull later improvements to the shared core, add this template as a remote:
   `git remote add upstream <template-url>` and merge the shared files when needed.

## Conventions

- `#!/usr/bin/env bash`, bash arrays and `[[ ]]`, not POSIX.
- Source guards named `_TMUX_PLUGIN_<MODULE>_LOADED`.
- `export -f` on every public function.
- Option names namespaced `@<plugin>_revamped_*`.
- shellcheck clean at `--severity=warning`.

## Testing

```bash
make test        # full bats suite, unit and integration
make test-unit   # unit tests only
make lint        # shellcheck
make coverage    # kcov line coverage, enforces 95% (Linux)
```

Coverage runs on the Linux CI job. On macOS, SIP prevents kcov from tracing bash
through bats, so measure coverage on Linux and rely on the bats suite locally.

## License

[MIT](LICENSE), copyright Gustavo Franco.
