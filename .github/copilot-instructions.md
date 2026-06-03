---
applyTo: "**"
---

# Copilot Instructions — Kube-Argus

Real-time Kubernetes dashboard delivered as a **single Go binary with the React frontend embedded** (`web/dist/` is served by the Go HTTP server). Targets self-managed, EKS, GKE, AKS, kind, k3s. No database, no CRDs, no agents on worker nodes.

<!-- These instructions are derived from .understand-anything/knowledge-graph.json and recent commit history. Keep them in sync when refactoring or introducing new conventions. -->

## Repository layout

```
cmd/server/         # Go backend, package main, one file per HTTP feature area
web/                # React 19 + Vite + TypeScript + Tailwind frontend
  src/
    components/
      views/        # Feature views (XxxView.tsx) — one per resource kind
      modals/       # Modal dialogs (XxxModal.tsx)
      ui/           # Reusable atoms, charts, timelines
    context/        # ThemeContext, AuthContext
    hooks/          # useFetch, useMetrics, useFavorites, useDrainBg, useAIStream
    layout/         # NamespacePicker, UserMenuDropdown
    App.tsx, main.tsx, routing.ts, types.ts, index.css
deploy/
  helm/kube-argus/  # Helm chart (templates, values.yaml, values.schema.json)
  k8s/              # Raw manifests (alternative to Helm)
docs/               # GitHub Pages landing page (auto-synced to gh-pages on change)
.github/workflows/  # ci.yaml, release.yaml, pages-sync.yaml
Dockerfile, docker-compose.yaml
```

**Historical pitfall:** `.gitignore` once shadowed `cmd/server/` because of a stray `/server` entry. The current `.gitignore` has `/server` (the bare binary), not `cmd/server`. If you add ignore rules, do not glob anything that would re-shadow the backend source tree.

## Backend conventions (`cmd/server/*.go`)

- **Single package `main`.** All `.go` files live directly under `cmd/server/`. Group by HTTP feature area (`pods.go`, `nodes.go`, `workloads.go`, `networking.go`, `storage.go`, `jit.go`, `audit.go`, `slack.go`, `webhook.go`, ...). Do not introduce sub-packages without a strong reason — the current layout is intentional after `refactor: move Go source files into cmd/server/ for idiomatic project layout`.
- **Routes are registered in `main.go`** on a `http.ServeMux`. Handler naming is `apiXxx` (e.g. `apiPods`, `apiNodeAction`, `apiJITAction`). Action endpoints use a trailing slash and parse the resource from the path (e.g. `/api/nodes/`, `/api/workloads/`, `/api/jit/`, `/api/cronjobs/`).
- **Middleware chain** (outer → inner): `gzipWrap(authMiddleware(corsWrap(mux)))`. New global behaviour belongs in a wrapper added in `main.go`, not inside individual handlers.
- **JSON helpers** are in `cmd/server/server.go` — use them, do not hand-roll `json.NewEncoder`:
  - `j(w, v)` — success JSON
  - `je(w, msg, code)` — error JSON with status code
  - `jk8s(w, err)` — translate a `k8serr.StatusError` into the right HTTP code + message
  - `jGz(w, r, v)` — gzip-aware JSON for large payloads
- **AuthZ helpers** are in `auth.go` / `jit.go`. They return `bool` and write the error response themselves; callers `return` on false:
  - `requireAdmin(w, r)` — admin role required
  - `requireAdminOrJIT(w, r, namespace, ownerKind, ownerName)` — admin OR an active JIT grant for that specific workload. Use this for write actions on workloads/pods (scale, restart, exec, CronJob trigger, delete).
- **Kubernetes cache** is in `cache.go`. Read it via `cache.mu.RLock()` / `defer cache.mu.RUnlock()`. Never call `client-go` List directly from handlers — it bypasses the shared cache and breaks the "one set of API calls per refresh, not per-user" guarantee.
- **Logging:** structured JSON via `log/slog`, level from `LOG_LEVEL` env (`debug`/`info`/`warn`/`error`, default `info`). Use `slog.Info("msg", "key", value, ...)` — not `log.Printf` and not `fmt.Println` (except the startup banner). Established by `feat: structured JSON logging with configurable LOG_LEVEL`.
- **TTY-aware ANSI:** only emit colour codes when `term.IsTerminal(int(os.Stdout.Fd()))` returns true. The startup banner in `main.go` is the canonical example. Established by `fix: only emit ANSI color codes when stdout is a real terminal`.
- **Don't swallow errors silently.** When an upstream (metrics-server, Prometheus) fails, log it via `slog` and surface the error on the response — established by `fix: log metrics-server errors instead of silently discarding them` and `fix: show Prometheus error message on node metrics panel`.
- **Persistence model:** the project has no database. State that must survive restarts lives in ConfigMaps:
  - JIT requests: `JIT_CONFIGMAP_NAME` (default `kube-argus-jit`), managed by `jit.go`
  - Audit log: `AUDIT_CONFIGMAP_NAME` (default `kube-argus-audit`), managed by `audit.go`
  Both files own their own `Init*` + `Restore` + `Persist` flow. Match the pattern if you add new persisted state.
- **Notification config precedence:** Settings page UI (stored via `apiSlackSettings` / `apiWebhookSettings`) takes precedence over env vars at runtime. Env vars only act as a bootstrap fallback for headless deploys.
- **Outbound webhook signing:** if `NOTIFY_WEBHOOK_SECRET` is set, send `X-KubeArgus-Signature: sha256=<hex(HMAC)>` over the JSON body. The Slack interactive callback verifies `SLACK_SIGNING_SECRET` on inbound. Don't add new outbound integrations without HMAC support.
- **Metrics sources:** prefer Prometheus when `PROMETHEUS_URL` is set, else fall back to raw cAdvisor via the kubelet (`c48efaa`). Vanilla clusters without Prometheus must still get usable node metrics.

## Frontend conventions (`web/src/`)

- **React 19 + Vite + TypeScript + Tailwind CSS.** No Redux/Zustand; state lives in components, contexts, and custom hooks.
- **Data fetching goes through `hooks/useFetch.tsx`.** It is the canonical pattern:
  - `const { data, err, loading, refetch } = useFetch<T>(url, 10_000)` — second arg is the poll interval in ms. Pass `null` for the URL to skip the fetch.
  - On HTTP 401 it redirects to `/auth/login` automatically.
  - It has a 30-second in-memory SWR cache keyed by URL.
  - Mutations use the sibling `post(url)` helper, which propagates 403 admin-required and 401 unauthorized as thrown errors.
  - Do not call `fetch()` directly from components.
- **10-second refresh** is the standard cadence (`useFetch(url, 10_000)`). Use 5s only for streaming/active panels; longer intervals for static metadata. Don't introduce per-component polling primitives.
- **Shared TypeScript types** for cluster resources live in `web/src/types.ts`. Add new shapes there rather than redeclaring inline.
- **View component layout:** each cluster resource has a `XxxView.tsx` (list) under `components/views/`. Resources with a detail page also have a companion `XxxDetailView.tsx` — established by `feat: service & HPA detail views, resource graph click-through, bugfixes`. Examples: `PodsView` + `PodDetailView`, `NodesView` + `NodeDescribeView`, `WorkloadsView` + `WorkloadDetailView`, `ServicesView` + `ServiceDetailView`, `HPAView` + `HPADetailView`. Add the detail companion when introducing a new resource view if it has a meaningful drill-down.
- **Modals** go in `components/modals/` and use the `XxxModal.tsx` naming. JIT-gated actions (restart, scale, CronJob trigger, exec) trigger `JITRequestModal` when the user is a viewer.
- **Shareable URL state:** view, namespace, filter, and selection should be reflected in the URL so a link can restore the same view. Don't store these only in component state. Established by `feat: webhook notifications, audit consistency, shareable URLs, perf`.
- **Theme:** both light and dark are supported via `ThemeContext`. New UI must work in both — established by `feat: light theme redesign, workload restart with JIT, auth crash fix`. Tailwind's `dark:` modifier is the mechanism; avoid hard-coded `bg-white`/`text-black` without a `dark:` counterpart.
- **Streaming UIs** (logs, AI diagnostics, drain progress) use:
  - `useAIStream` for OpenAI-compatible chat-completion SSE
  - `useDrainBg` for the drain wizard (survives page reload)
  - WebSocket directly for `/api/exec` and `/api/ws/presence`
- **Favorites** persist via `useFavorites` (localStorage per-user). Don't sprinkle ad-hoc localStorage usage.
- **Init containers** must be rendered alongside regular containers in any container-listing view. Established by `feat: namespace favorites, cronjob trigger, restart timeline, init containers`.

## Deployment & packaging

- **Single binary.** `Dockerfile` is a three-stage build: Node 20 stage builds `web/dist`, Go 1.25 stage cross-compiles using `BUILDPLATFORM` / `TARGETARCH` (Go native multi-arch, not QEMU — see `8b851f6`), final stage is Alpine 3.21 running as `nobody`.
- **Helm chart** lives in `deploy/helm/kube-argus/`. Three rules:
  1. Every new top-level value must also be declared in `values.schema.json` (established by `chore: add values.schema.json for ArtifactHub values table display`).
  2. Keep `Chart.yaml` `version` and `appVersion` aligned (established by `chore: align chart version with appVersion`).
  3. Install docs always include `--namespace kube-argus --create-namespace` (established by `docs: add --namespace and --create-namespace to all Helm install commands`).
- **Raw manifests** in `deploy/k8s/` are the alternative install path. Mirror any change made to the Helm `deployment.yaml` / `rbac.yaml` here as well, otherwise non-Helm users drift.
- **ArtifactHub:** verified publisher requires `artifacthub-repo.yml` at **both** the repo root AND inside the chart directory (`6b6c8ce`, `58d4611`). Don't move or delete either.
- **Docs landing page** in `docs/index.html` auto-syncs to the `gh-pages` branch via `.github/workflows/pages-sync.yaml`. Edit `docs/` directly; do not commit to `gh-pages`.

## CI/CD

- **Release on tag push.** `.github/workflows/release.yaml` builds multi-arch images, pushes to GHCR, packages the Helm chart, and auto-creates a GitHub Release with CHANGELOG content as the body (`8211e0f`, `e7c5175`). It also deletes-and-recreates an existing release to support retagging (`da578ab`).
- **CHANGELOG.md is the source of truth** for release notes. Update it before tagging.
- **Helm repo is served from GitHub Pages** at `https://manishchaudhary101.github.io/kube-argus` (`b808880`). OCI registry under `oci://ghcr.io/manishchaudhary101/charts/kube-argus` is the alternative.

## Security defaults

- Apache 2.0 licensed; report vulnerabilities per `SECURITY.md`.
- All Go and npm vulnerabilities should resolve cleanly — Dependabot alerts have been zeroed before (`5a5279f`, `f7fbf6d`). Keep `golang.org/x/crypto` and `go-jose/v4` current.
- **Secret YAML redaction:** the YAML viewer redacts Secret `data` by default; explicit reveal is gated to admin. Established by `v1.2.7: secret YAML redaction, ArtifactHub screenshots, dep bumps`. Do not bypass this in new views.
- AWS Secrets Manager is an optional bootstrap source for the OIDC client secret (`AWS_SECRET_NAME` + `AWS_REGION`). Do not commit any secret values.

## When in doubt

- The knowledge graph at `.understand-anything/knowledge-graph.json` is the structured map of the codebase. The guided tour (12 steps) is the recommended onboarding path.
- `README.md` is the canonical feature list; the `<!-- BEGIN AGENT-OWNED -->` block at the bottom is regenerated from the graph + recent commits.
- `CONTRIBUTING.md` covers local development setup.
