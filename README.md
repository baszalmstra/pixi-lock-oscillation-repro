# pixi lock oscillation repro

Minimal reproducer: every `pixi lock` reports the lock as updated, even when no inputs changed.

## Layout

```
.
├── pyproject.toml      # repro -> c (path = "sub/c")
└── sub/
    ├── c/pyproject.toml  # c -> b (path = "../b")
    └── b/pyproject.toml
```

## Reproduce

```sh
pixi lock
pixi lock
pixi lock
```

Each invocation prints:

```
✔ Updated lock-file
Environment: default
  ~ (pypi) b          <unknown>  ->  <unknown>
```

With `pixi lock -vv` the cause is logged:

```
the pypi dependencies of environment 'default' for platform linux-64 are out of date
because metadata for local package 'c' has changed:
dependencies changed - added: [b], removed: [b]
```

## Cause

Look at `pixi.lock`:

```yaml
- pypi: ../b              # wrong — from repo root, ../b is OUTSIDE the repo
  name: b
- pypi: sub/c
  name: c
  requires_dist:
  - b @ ../b              # c's relative path to b — correct from sub/c
```

`c`'s `requires_dist` records `b @ ../b`, which is correct relative to `sub/c/`
(resolves to `sub/b`). Pixi copies that string verbatim as `b`'s top-level
`pypi:` path entry, but the top-level entry is interpreted relative to the
repo root, where `../b` points outside the repo.

On the next `pixi lock`, comparing freshly-resolved metadata for `c` against
the lockfile-recorded metadata flags `b` as both added and removed, so the
environment is treated as out-of-date forever.
