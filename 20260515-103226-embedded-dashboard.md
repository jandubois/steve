# Deep Review: 20260515-103226-embedded-dashboard

| | |
|---|---|
| **Date** | 2026-05-15 10:32 |
| **Repo** | [rancher/steve](https://github.com/rancher/steve) |
| **Round** | 1 |
| **Author** | Mark Yen <mark.yen@suse.com> |
| **Branch** | `embedded-dashboard` (top 3 commits, equivalent to [rancher-sandbox/rancher-desktop-steve#11](https://github.com/rancher-sandbox/rancher-desktop-steve/pull/11)) |
| **Commits** | `039e08a7` Embed dashboard assets into steve<br>`78be894d` CI: Release workflow: Be explicit about permissions<br>`ab052c21` Address AI review comments |
| **Reviewers** | Claude Opus 4.7 (effort: xhigh), Codex GPT 5.5 (effort: xhigh), Gemini 3.1 Pro (effort: default) |
| **Verdict** | **Merge with fixes** — embedded-dashboard approach is sound; runtime correctness gaps in caching, SPA asset handling, build-input freshness, CLI flag honoring, and graceful shutdown should land first. |
| **Wall-clock time** | `29 min 3 s` |

## Executive Summary

This branch turns `steve` into a single-binary Rancher Desktop launcher: it
embeds the dashboard SPA via `go:embed`, listens on a random localhost port,
and rewrites the dashboard's hard-coded `127.0.0.1:6120` references on the
fly. Build path is overhauled (Docker/Drone → Makefile + GitHub Actions with
SHA-pinned actions, two-job least-privilege split, SHA-512 verification of
the downloaded dashboard tarball).

Five Important findings: the ETag/304 caching path is dead code (`Main.Sum`
is empty for `go build` binaries); the inherited `--http(s)-listen-port`
flags are silent no-ops; missing assets under enumerated dashboard
directories return HTML with `text/html` instead of 404; bumping
`DASHBOARD_VERSION` in the Makefile does not trigger re-download of the
tarball; the Shutdown goroutine can be killed mid-drain when `run()` returns,
so in-flight requests terminate abruptly. Ten Suggestions cover micro-perf,
defensive coding, build hygiene, and Strunk's long-form-flag convention.

Structure: Critical 0, Important 5, Suggestions 10, Design Observations 5.

---

## Critical Issues

None.

## Important Issues

I1. **ETag/304 cache path is dead code — `Main.Sum` is always empty for `go build` binaries** — `main.go:152-158`, `main.go:177-183`, `main.go:202-205` [Claude Opus 4.7, Gemini 3.1 Pro]

```go
etag := ""
if buildInfo, ok := runtimedebug.ReadBuildInfo(); ok {
    etag = buildInfo.Main.Sum
}
```

`runtime/debug.BuildInfo.Main.Sum` is the module checksum that the Go proxy
computes when it ingests a module version. For binaries produced by `go
build` from a working tree — i.e. exactly the Makefile path at
`Makefile:30-34` — `Main.Path`, `Main.Version`, and `Main.Sum` are all
empty. Verified locally: a one-file `go build` reports `Main.Sum=""`. The
release binaries therefore ship with `etag == ""`, the `If-None-Match`
short-circuit at line 177 never fires, and the `ETag` response header at
line 204 is never set. Every HTML/JS asset is re-read and re-rewritten on
every browser refresh, with no client-side caching at all.

A secondary defect: even if `Main.Sum` were populated, RFC 7232 requires
ETag values to be quoted, and the body is transformed on the wire (port
substitution), which makes the ETag at best a weak validator (`W/"..."`).

Fix: switch to the `vcs.revision` setting (populated automatically when
`go build` runs against a git checkout, present in CI). Quote and weak-mark
it per RFC 7232:

```go
etag := ""
if buildInfo, ok := runtimedebug.ReadBuildInfo(); ok {
    for _, s := range buildInfo.Settings {
        if s.Key == "vcs.revision" {
            etag = `W/"` + s.Value + `"`
            break
        }
    }
}
```

If `vcs.revision` is not reliable across all distribution channels, fall
back to hashing `dashboardFiles` at startup — the embedded bytes are the
real cache key.

I2. **`--https-listen-port` and `--http-listen-port` are silent no-ops** — `main.go:45-47`, `pkg/server/cli/clicontext.go:80-89` [Claude Opus 4.7]

```go
app.Flags = append(
    stevecli.Flags(&config),
    debug.Flags(&debugconfig)...)
```

`stevecli.Flags` still registers `--https-listen-port` (default 9443) and
`--http-listen-port` (default 9080) into `config.HTTPSListenPort` and
`config.HTTPListenPort` respectively. The pre-PR `run` body honored them
via `s.ListenAndServe(ctx, config.HTTPSListenPort, config.HTTPListenPort,
nil)`. The new body always listens on `127.0.0.1:0` (line 65-71) and never
reads either field. The help text exposes both flags, and the user's
input is accepted without complaint — but discarded.

The same coupling drags in `--ui-path`, `--offline`, `--kubeconfig`, and
other `stevecli` flags that may or may not apply to an embedded dashboard
launcher; future stevecli additions land here automatically.

Fix: filter the upstream flag list to the flags this binary actually
honors, or declare a purpose-built flag set for the launcher. A minimal
filter:

```go
upstream := stevecli.Flags(&config)
filtered := upstream[:0]
for _, f := range upstream {
    name := strings.Split(f.GetName(), ",")[0]
    if name == "https-listen-port" || name == "http-listen-port" {
        continue
    }
    filtered = append(filtered, f)
}
app.Flags = append(filtered, debug.Flags(&debugconfig)...)
```

I3. **Missing assets under enumerated dashboard directories return HTML, not 404** — `main.go:160-175` [Gemini 3.1 Pro]

```go
actualPath := strings.TrimPrefix(r.URL.Path, "/")
if _, err := subFS.Open(actualPath); errors.Is(err, fs.ErrNotExist) {
    // If the file does not exist, serve the index file because this is
    // a single-page application.
    actualPath = "index.html"
}
switch path.Ext(actualPath) {
case ".html", ".js":
default:
    http.FileServerFS(subFS).ServeHTTP(w, r)
    return
}
```

The SPA fallback unconditionally rewrites `actualPath` to `index.html` for
any missing path under an enumerated top-level dashboard directory (e.g.
`/_nuxt/`, `/assets/`, `/c/`). After the rewrite, `path.Ext(actualPath)`
becomes `.html`, so a request for `/_nuxt/old-chunk.js` (a stale asset
referenced by a cached `index.html`) returns the SPA shell with
`Content-Type: text/html` and HTTP 200. Browsers fail opaquely when they
try to parse an HTML document as a script or stylesheet, and the user
sees a broken page rather than a clean 404 that would trigger a reload.

Fix: only fall back to the index for paths with no extension (typical of
SPA routes); let extensioned 404s propagate as 404s.

```go
if _, err := subFS.Open(actualPath); err != nil {
    if path.Ext(strings.TrimRight(actualPath, "/")) == "" {
        actualPath = "index.html"
    }
    // else: leave actualPath as-is so the FileServerFS path returns 404.
}
```

I4. **Dashboard extraction does not depend on Makefile, so version bumps silently embed the old dashboard** — `Makefile:1-2`, `Makefile:37-44` [Codex GPT 5.5]

```make
DASHBOARD_VERSION := v2.11.1.rd4
DASHBOARD_CHECKSUM := 22839af7ae78f1c9dbf2559a0be8a6cd355426b6dc31795ce8fd86a4257d4d509778b07b4cee36088c0c5ec072e6c09e2d8e7bbb4d070d0b70cd2c158abfe668
...
dashboard/index.html: dashboard.tgz
    mkdir -p "$(@D)"
    tar -xzf "$<" -C "$(@D)"

dashboard.tgz:
    wget -O "$@" "https://github.com/.../desktop-$(DASHBOARD_VERSION)/rancher-dashboard-desktop-embed.tar.gz"
    echo "$(DASHBOARD_CHECKSUM)  dashboard.tgz" | sha512sum -c -
```

`dashboard.tgz` has no prerequisites. If a developer bumps `DASHBOARD_VERSION`
or `DASHBOARD_CHECKSUM` in a workspace with an older `dashboard.tgz`, make
treats the existing file as up-to-date and skips both download and
checksum verification. The executable target depends on `$(MAKEFILE_LIST)`
at line 30, so the binary rebuilds — but embeds the old dashboard. Local
extraction at line 39 also overlays the new archive on top of any
leftover files from the prior version (no `rm -rf` first).

Fix: declare the Makefile as a prerequisite for both targets, verify the
checksum on every extraction, and replace the extracted tree atomically:

```diff
-dashboard/index.html: dashboard.tgz
-    mkdir -p "$(@D)"
-    tar -xzf "$<" -C "$(@D)"
+dashboard/index.html: dashboard.tgz $(MAKEFILE_LIST)
+    echo "$(DASHBOARD_CHECKSUM)  dashboard.tgz" | sha512sum -c -
+    rm -rf "$(@D)"
+    mkdir -p "$(@D)"
+    tar -xzf "$<" -C "$(@D)"

-dashboard.tgz:
+dashboard.tgz: $(MAKEFILE_LIST)
     wget -O "$@" "https://github.com/.../desktop-$(DASHBOARD_VERSION)/rancher-dashboard-desktop-embed.tar.gz"
     echo "$(DASHBOARD_CHECKSUM)  dashboard.tgz" | sha512sum -c -
```

I5. **Shutdown goroutine can be killed mid-drain when `run()` returns** — `main.go:122-139` [Claude Opus 4.7]

```go
server := &http.Server{Handler: mux}
go func() {
    <-ctx.Done()
    <-time.After(10 * time.Millisecond)
    timeoutCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := server.Shutdown(timeoutCtx); err != nil {
        logrus.Errorf("error shutting down server: %v", err)
    }
}()
if err := server.Serve(listener); err != nil && !errors.Is(err, http.ErrServerClosed) {
    return err
}
return nil
```

`http.Server.Shutdown` causes `Serve` to return `http.ErrServerClosed`
immediately, but Shutdown itself blocks until idle connections close (up
to the 5s timeout). The current code lets `run` return as soon as `Serve`
unblocks; the goroutine is still inside `Shutdown`, draining in-flight
requests. When `main` returns (or `logrus.Fatal` fires), Go terminates
the process and kills the shutdown goroutine, severing live connections.

Fix: synchronize on the shutdown goroutine before returning.

```diff
+shutdownDone := make(chan struct{})
 go func() {
+    defer close(shutdownDone)
     <-ctx.Done()
-    <-time.After(10 * time.Millisecond)
     timeoutCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
     defer cancel()
     if err := server.Shutdown(timeoutCtx); err != nil {
         logrus.Errorf("error shutting down server: %v", err)
     }
 }()
-if err := server.Serve(listener); err != nil && !errors.Is(err, http.ErrServerClosed) {
-    return err
-}
-return nil
+err = server.Serve(listener)
+<-shutdownDone
+if err != nil && !errors.Is(err, http.ErrServerClosed) {
+    return err
+}
+return nil
```

While here, drop the `10 * time.Millisecond` wait — see S3.

## Suggestions

S1. **Per-request `http.FileServerFS` construction** — `main.go:173` [Claude Opus 4.7]

```go
default:
    // For non-text files, just serve them directly without rewriting.
    http.FileServerFS(subFS).ServeHTTP(w, r)
    return
```

The handler is invoked on every non-HTML/JS request and has no per-request
state. Hoist it out of the closure so it is built once.

```diff
+    fileServer := http.FileServerFS(subFS)
     return func(w http.ResponseWriter, r *http.Request) {
         ...
-            http.FileServerFS(subFS).ServeHTTP(w, r)
+            fileServer.ServeHTTP(w, r)
             return
```

S2. **Dead `fs.ReadFileFS` startup check and redundant type assertion** — `main.go:149-151`, `main.go:188` [Claude Opus 4.7]

```go
if _, ok := subFS.(fs.ReadFileFS); !ok {
    return nil, fmt.Errorf("sub filesystem does not support ReadFile")
}
...
contents, err := subFS.(fs.ReadFileFS).ReadFile(actualPath)
```

`fs.Sub` on an `embed.FS` always returns a value that implements
`fs.ReadFileFS`. The error branch cannot fire, and the second assertion on
every request is unnecessary. Type the local variable as `fs.ReadFileFS`
once:

```diff
-    subFS, err := fs.Sub(dashboardFiles, "dashboard")
+    raw, err := fs.Sub(dashboardFiles, "dashboard")
     if err != nil {
         return nil, fmt.Errorf("could not create sub filesystem: %w", err)
     }
-    if _, ok := subFS.(fs.ReadFileFS); !ok {
-        return nil, fmt.Errorf("sub filesystem does not support ReadFile")
-    }
+    subFS := raw.(fs.ReadFileFS)
```

…then drop the second assertion at line 188.

S3. **Unexplained 10 ms shutdown sleep** — `main.go:125-128` [Claude Opus 4.7]

```go
go func() {
    <-ctx.Done()
    // Wait some time for steve stuff to shut down
    <-time.After(10 * time.Millisecond)
```

The comment names neither the resource being waited on nor any invariant
the 10 ms is meant to establish. Steve's shutdown is driven by `ctx`
cancellation elsewhere; if a specific teardown order is required, encode
it (e.g. a `steveDone` channel). Otherwise remove the line — `http.Server.Shutdown`
already drains in-flight handlers under its own 5 s timeout (line 130).

S4. **`mux.Handle("/c/", rewriter)` panics if the dashboard ever adds a top-level `c/`** — `main.go:101`, `main.go:110-116` [Claude Opus 4.7]

```go
mux.Handle("/c/", rewriter)
...
for _, f := range files {
    if f.IsDir() {
        mux.Handle("/"+f.Name()+"/", rewriter)
```

`http.ServeMux.Handle` panics on duplicate pattern registration. Today no
top-level dashboard directory is named `c/`, but a future dashboard
release can crash the launcher at startup.

```diff
+for _, f := range files {
+    pattern := "/" + f.Name()
+    if f.IsDir() {
+        pattern += "/"
+    }
+    if pattern == "/c/" {
+        continue // already registered above
+    }
+    mux.Handle(pattern, rewriter)
+}
```

S5. **`/api/steve-port` handler is documented as obsolete** — `main.go:117-120` [Claude Opus 4.7]

```go
// Handle the steve-port API, which is no longer used.
mux.Handle("/api/steve-port", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "%d", port)
}))
```

If the comment is accurate, the handler is dead code and should be
removed. If it is kept for older Rancher Desktop builds, the comment
should say which builds and until when. The current wording invites a
future reader to delete it.

S6. **Hard-coded `127.0.0.1:6120/` replacement misses the trailing-slash-less form** — `main.go:156-158` [Gemini 3.1 Pro]

```go
replacer := strings.NewReplacer(
    "http://127.0.0.1:6120/",
    fmt.Sprintf("http://127.0.0.1:%d/", port))
```

The replacer only matches the form with a trailing slash. A JS literal
like `const api = "http://127.0.0.1:6120";` or a CSS `url(http://127.0.0.1:6120)`
slips through unchanged and points the client at the wrong port. Cheapest
mitigation: add the slash-less variant to the replacer pairs.

S7. **`.gitignore` no longer ignores the locally built `steve` binary** — `.gitignore:7` [Gemini 3.1 Pro]

```diff
 .idea
-steve
+/dashboard/
```

The Makefile writes to `release/`, but a developer who runs `go build` in
the repository root produces `./steve` (or `./steve.exe`), which would
now show up under `git status` and could be committed by accident.
Restoring the entry costs nothing.

S8. **Stray double space in `go build` invocation** — `Makefile:32` [Claude Opus 4.7]

```make
env GOOS=$(call GOOS,$(1)) GOARCH=$(call GOARCH,$(1)) go build  -ldflags '-s -w' -trimpath -o "$$@"
```

Cosmetic. Remove the duplicate space.

S9. **`wget` invocation is noisy and uses short-form `-O`** — `Makefile:43` [Claude Opus 4.7]

```make
wget -O "$@" "https://github.com/.../desktop-$(DASHBOARD_VERSION)/rancher-dashboard-desktop-embed.tar.gz"
```

Default `wget` output writes a per-100 KB progress meter that clutters CI
logs. The project's long-form-flag convention also asks for the spelled-out
option.

```diff
-    wget -O "$@" "https://github.com/.../desktop-$(DASHBOARD_VERSION)/rancher-dashboard-desktop-embed.tar.gz"
+    wget --no-verbose --output-document="$@" "https://github.com/.../desktop-$(DASHBOARD_VERSION)/rancher-dashboard-desktop-embed.tar.gz"
```

S10. **`os.Stdout = os.Stderr` reassignment is non-obvious** — `main.go:85-88` [Claude Opus 4.7]

```go
if err := os.Stdout.Close(); err != nil {
    return err
}
os.Stdout = os.Stderr
```

The intent — redirect any later `fmt.Print*` calls to stderr now that fd 1
is closed — is reasonable but invisible. A one-line comment saves the
next reader a detour through git history.

## Design Observations

### Concerns

**The rewriter is a band-aid for a hard-coded port in the dashboard build** *(future)* [Claude Opus 4.7]

The dashboard ships JS/HTML with literal `http://127.0.0.1:6120/` strings,
and the launcher stream-rewrites them on every text-asset response. This
is the right band-aid but cements an awkward boundary: the SPA contains
URLs it cannot use, and steve is on the hook to stream-rewrite static
assets forever. The handler at `main.go:117-120` already exposes the
runtime port at `/api/steve-port`; making the dashboard discover its own
origin at startup would eliminate the rewriter, the ETag question (I1)
entirely, and the per-asset `strings.NewReplacer` cost. Worth raising
upstream in `rancher-sandbox/rancher-desktop-dashboard`.

**`stevecli.Flags` couples the launcher to flags it does not honor** *(in-scope)* [Claude Opus 4.7]

Pulling the entire `stevecli` flag set wholesale (I2) means every
upstream change to that flag set lands here, regardless of whether this
binary can act on it. A purpose-built `dashboardcli` package — or a thin
wrapper that re-exports only the flags this binary actually consumes —
makes the contract explicit and keeps future steve flag additions
opt-in.

**Path hijacking via top-level dashboard directories** *(future)* [Gemini 3.1 Pro]

Top-level directories in the embedded dashboard are routed via
`mux.Handle("/"+f.Name()+"/", rewriter)`. If the dashboard repository
ever introduces a top-level directory named `v1`, `api`, or `k8s`, it
will hijack the corresponding steve API routes. A startup check against
a reserved-prefix list would catch this at boot rather than at first
request.

### Strengths

**Makefile shape** [Claude Opus 4.7] — Per-platform explicit rules,
`.DELETE_ON_ERROR` so checksum failures self-clean, and
`$(MAKEFILE_LIST)` as an executable-target prerequisite so Makefile
edits trigger rebuilds. (I4 addresses one remaining gap.) *(in-scope)*

**Two-job release workflow** [Claude Opus 4.7] — Splitting into a
`build` job with `contents: read` and a separate `release` job with
`contents: write` is the right least-privilege shape. The top-level
empty `permissions: {}` adds a good belt-and-braces default. *(in-scope)*

**Dashboard checksum lives next to its version** [Claude Opus 4.7] —
`Makefile:1-2` puts the security-relevant pair in one place, making it
hard to bump one without the other. *(in-scope)*

**Outer mux preserves steve routing** [Codex GPT 5.5] — Explicitly
mounting only the embedded dashboard's top-level static paths (rather
than catching every unknown URL as an SPA fallback) keeps the existing
steve API surface intact. *(in-scope)*

**`go:embed` + dynamic localhost port** [Gemini 3.1 Pro] — Embedding
the dashboard via `go:embed` collapses the deployment artifact into a
single self-contained binary, and the random localhost port keeps the
dashboard off the network. *(in-scope)*

## Testing Assessment

The fork has zero `*_test.go` files (pre-existing — see Repo Overview).
This branch adds non-trivial routing logic with no tests. Prioritized
gaps, in order of value:

1. **Rewrite path** — assert that `http://127.0.0.1:6120/foo` in `index.html`
   is rewritten to `http://127.0.0.1:<port>/foo`, and that `Content-Type`
   is set correctly for `.html` and `.js`.
2. **SPA fallback** — request a deep URL like `/c/local/explorer/anything`
   and assert the response is the rewritten `index.html`. Add a second
   case that asserts a missing `.js` returns 404 (covering I3).
3. **Non-text asset path** — request an embedded `.png` and assert verbatim
   bytes with the right `Content-Type`.
4. **ETag** — once I1 is fixed, assert that a matching `If-None-Match`
   returns 304 with no body.
5. **Mux composition** — `/` returns 302 to `/c/local/explorer`;
   `/api/steve-port` returns the listening port; an unknown `/foo`
   reaches the steve handler.

These are pure handler tests; `httptest.NewRecorder` plus the embedded
FS is enough — no Kubernetes apiserver needed.

Codex verified that `go test ./...` passes and that `make
release/steve.sha512sum` builds all five platform archives.

## Documentation Assessment

- `main.go:1-5` adds a serviceable package-level godoc.
- `rewriteHandler` has a one-sentence doc.
- The Makefile is well-commented for a Makefile.
- `.github/workflows/release.yml` has no comment explaining why the
  two-job split exists; a one-line top comment ("`build` runs
  unprivileged; `release` is the only job with `contents: write`")
  would help future readers.
- No README change. Per `app.Description = "This is part of Rancher
  Desktop 2.x and should not be run manually."`, end users should not
  invoke this binary directly — but the help text inherited from
  `stevecli` still exposes the now-no-op `--http(s)-listen-port` flags
  (tied to I2).

## Commit Structure

Three commits, clean for review:
- `039e08a7` — bulk feature (embed dashboard, rewrite handler, Makefile).
- `78be894d` — focused permissions tightening on the release workflow.
- `ab052c21` — conventional post-AI-review fixup commit.

No squash necessary.

## Acknowledged Limitations

- `main.go:117` — `// Handle the steve-port API, which is no longer
  used.` The author has acknowledged the endpoint is obsolete; this
  branch keeps it for compatibility. S5 promotes it not because the
  limitation is wrong, but because the comment does not say *why* it
  stays.
- `main.go:186-187` — `// At the moment, embed.FS just stores the whole
  thing in memory anyway, so this should not be a problem.` The author
  acknowledges the read-then-replace cost is tolerable given embed.FS's
  implementation. Reasonable; nothing to do.

## Unresolved Feedback

None. The "Address AI review comments" commit (`ab052c21`) on this
branch already incorporates earlier feedback; the rancher-sandbox PR
has no inline review comments yet.

## Agent Performance Retro

### [Claude]

Claude carried this review: it raised three of the five Important findings
(I1, I2, I5) plus all but two Suggestions, and was the only agent to spot
the silent no-op CLI flags (I2) and the abrupt shutdown teardown (I5).
Calibration was right on — every finding survived verification against the
current code with no downgrades. Claude's "Strengths" claim that the SPA
fallback works for trailing slashes was correct and resolved a direct
contradiction with Gemini.

### [Codex]

Codex produced exactly one finding (I4) but it was load-bearing: the
Makefile dependency gap that lets a `DASHBOARD_VERSION` bump silently
embed the old dashboard. Neither other agent noticed. Codex also did
real verification work — `go test ./...` and `make
release/steve.sha512sum` — which gave the report a concrete pass/fail
data point. Its finding density is low, but its hit rate was 100 % and
the unique insight justified the slot.

### [Gemini]

Gemini found two genuine issues (I1 ETag and I3 SPA-fallback content-type
bug); the I3 catch was unique and important. It also produced one false
positive (a claim that `fs.Sub(...).Open("foo/")` returns `fs.ErrInvalid`
and breaks the SPA fallback) which a quick stdlib test disproved — the
result is actually `fs.ErrNotExist` and the code works. Gemini did not
run `git blame` (a known limitation due to daily quota); Claude and Codex
covered regression labeling.

### Summary

| | Claude Opus 4.7 | Codex GPT 5.5 | Gemini 3.1 Pro |
|---|---|---|---|
| Duration | 12m 48s | 9m 03s | — |
| Findings | 3I 8S | 1I | 2I 2S |
| Tool calls | 29 (Bash 21, Read 8) | 55 (shell 48, stdin 7) | — |
| Design observations | 4 | 1 | 2 |
| False positives | 0 | 0 | 1 |
| Unique insights | 9 | 1 | 2 |
| Files reviewed | 10 | 10 | 10 |
| Coverage misses | 0 | 0 | 0 |
| **Totals** | **3I 8S** | **1I** | **2I 2S** |
| Downgraded | 0 | 0 | 0 |
| Dropped | 0 | 0 | 1 |


Claude provided the most value by volume and breadth. Codex's single
Makefile finding was the highest-leverage individual insight of the
round. Gemini's I3 would have been missed without it. All three were
needed.

## Review Process Notes

### Skill improvements

- [ ] Add a review dimension covering `runtime/debug.BuildInfo` field
  semantics: distinguish fields populated only for proxy-installed
  modules (`Main.Path`, `Main.Version`, `Main.Sum`) from build-time
  settings populated for any VCS build (`vcs.revision`, `vcs.modified`,
  `vcs.time`). Flag code that uses the proxy-only fields as cache keys
  or build identifiers — the path is always empty for `go build`
  binaries.

- [ ] Add a review dimension covering `http.ServeMux` pattern collisions:
  any code that registers patterns derived from variable input (filesystem
  enumeration, configuration, plugin discovery) against the same mux as
  hand-registered patterns. ServeMux panics on duplicate registration;
  flag when no reserved-prefix check or `try`/`recover` guards the
  registration loop.

- [ ] Add a review dimension covering SPA fallback handlers: any handler
  that unconditionally rewrites a missing path to `index.html` (or any
  HTML shell) should preserve the original extension's content-type
  semantics — extensioned 404s must propagate as 404s, not become HTML
  with status 200. The pattern recognizes both `embed.FS`-backed and
  on-disk static servers.

### Repo context updates

None — the rancher-desktop-steve fork has not yet accumulated enough
convention-specific guidance to warrant a `deep-review-context.md`.

## Appendix: Original Reviews

### [Claude] Opus 4.7

See `review-claude-pass-1.md`. Findings: I1, I2, I3, S1, S2, S3, S4,
S5, S6, S7, S8 (renumbered I1, I2, I5, S1, S2, S3, S4, S5, S8, S9, S10
in this consolidated report).

### [Codex] GPT 5.5

See `review-codex-pass-1.md`. Findings: I1 (renumbered I4 in this
consolidated report). Verification: `go test ./...` passed; `make
release/steve.sha512sum` completed all five platform archives.

### [Gemini] 3.1 Pro

See `review-gemini-pass-1.md`. Findings: I1, I2, I3, S1, S2.
Consolidation: I1 → I1 (combined with Claude's I1); I2 → dropped as
false positive after verification (`fs.Sub(...).Open("foo/")` returns
`fs.ErrNotExist`, not `fs.ErrInvalid`); I3 → I3; S1 → S6; S2 → S7.
