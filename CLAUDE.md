# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project mission

SongLore reads a user's Spotify Liked Songs, picks one (latest-added or random), and uses the latest Claude model with web search to write an interesting introduction to it — angled on whichever dimension is most compelling for that track (composer, performer, arranger, lyricist, production history, cultural context, etc.). The generated piece can optionally be turned into speech and pushed to delivery channels (Telegram bot first; YouTube / Bilibili later).

This is the **core engine** for that idea. It is intentionally a backend-only service.

## Hard constraints

These are non-negotiable and shape every design decision:

- **Public repository.** No secrets, credentials, tokens, OAuth client secrets, refresh tokens, database dumps, or user data may ever be committed. All secrets load from env vars or mounted files (K8s Secrets in production, `.env` locally — `.env` is already gitignored).
- **No frontend, no admin UI.** Configuration is files / env / CLI flags. If a piece of state needs human editing, it lives in a config file or a small CLI subcommand — not a web page.
- **Minimum surface area for HTTP.** The only HTTP endpoints that should exist are the ones genuinely required: the Spotify OAuth callback, and any webhook receivers third-party platforms force on us (e.g. Telegram webhook mode). No "nice-to-have" REST APIs.
- **Go + K8s.** Single statically-linked binary, configured via env, deployed as a K8s Deployment (or CronJob for the periodic generation tick). Don't introduce a sidecar, message broker, or external dependency without a concrete forcing reason.
- **Don't over-engineer.** Prefer one struct over an interface-with-one-impl. Prefer a function over a package. Add abstraction only when a second concrete use case actually appears. The whole service should stay readable in an afternoon.
- **Tests are required, not optional.** Every package ships with table-driven unit tests. External integrations (Spotify, Claude, Telegram, TTS) are tested against fakes/mocks at the boundary; the boundary types themselves get a thin contract test. Aim for the tests to actually catch regressions, not to chase coverage numbers.

## Intended scope (what belongs in this repo)

Roughly in priority order — build the earlier ones first, and don't scaffold the later ones until they're actually next:

1. **Spotify auth** — OAuth 2.0 Authorization Code flow with PKCE. Persist the refresh token to a small local store (file or embedded KV; do *not* introduce a Postgres dependency for this). Refresh access tokens on demand.
2. **Liked Songs reader** — paginated fetch of the user's saved tracks, with selection modes: `latest` (most-recently-added) and `random`.
3. **Claude agent** — call the latest Claude model with web search enabled to research the chosen track and produce the introduction. The agent picks the angle (composer / performer / arranger / lyricist / cultural context / production story / etc.) — it is not hard-coded by us.
4. **Scheduler / frequency config** — how often a new piece is generated. Prefer a K8s CronJob over an in-process scheduler if the cadence is coarse (≥ hourly).
5. **TTS output (optional, toggleable)** — convert the written intro to audio. Pluggable provider; start with one.
6. **Delivery: Telegram bot** — push text (and audio if TTS is on) to a configured chat.
7. **Delivery: YouTube / Bilibili** — later. Will likely need video composition; treat as a separate concern when its turn comes.

Anything not on this list (web dashboard, multi-tenant accounts, analytics, recommendation engine, social features) is **out of scope** unless explicitly added here.

## Architectural guidance

- **Boundaries first.** Each external system (Spotify, Claude, TTS provider, Telegram) gets one small package that owns *all* the I/O for that system and exposes a narrow Go interface to the rest of the code. The rest of the code never imports the SDK directly. This is what makes the service testable.
- **Composition root in `cmd/`.** Wiring (config loading, client construction, dependency injection) happens in `main`. Packages under `internal/` should not reach for env vars or global state.
- **One binary, multiple subcommands** is fine (`songlore serve`, `songlore generate-once`, `songlore auth`) — but only add subcommands when there's a real use for them.
- **Persistence is minimal.** The only state we *must* keep is the Spotify refresh token and (probably) a small log of recently-picked tracks to avoid repeats. A flat file or BoltDB/Bbolt-style embedded store is sufficient. Don't reach for a network database.
- **Config:** env vars for secrets, a YAML/TOML file for non-secret settings (frequency, mode, enabled outputs). Document every config key in one place.

## Common commands

The codebase is at the scaffolding stage; once `go.mod` exists, the standard Go workflow applies:

- `go build ./...` — compile everything.
- `go test ./...` — run all tests.
- `go test -run TestName ./path/to/pkg` — run a single test.
- `go test -race ./...` — race detector; use before merging anything touching goroutines.
- `go vet ./...` and `gofmt -s -w .` — keep tree clean.

K8s manifests, Dockerfile, and CI config don't exist yet. When adding them, keep the Dockerfile a multi-stage build producing a `scratch` or `distroless` image, and keep manifests plain YAML (no Helm chart unless we have a real reason).

## When picking up work here

- Re-read this file's **Hard constraints** section before adding a dependency, an HTTP endpoint, or a config surface — most "obvious" additions violate one of them.
- If a task feels like it needs a new package, check first whether it's really one new function in an existing package.
- Surface uncertainty about scope back to the user rather than guessing — the explicit minimalism above means "should we build X?" is a real question, not a rhetorical one.
