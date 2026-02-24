# Add Vertex AI Support to cagent's Anthropic Provider

## Context

The `google` provider in cagent only supports Gemini models on Vertex AI (routes to `publishers/google/models/`). Claude models on Vertex AI live under `publishers/anthropic/models/` and need the Anthropic SDK's `vertex` package. This patch adds Vertex AI as an alternative backend for the `anthropic` provider, following the same `provider_opts` pattern the Gemini provider already uses.

The Anthropic Go SDK (`v1.22.1`, already in go.mod) has a `vertex` sub-package with `vertex.WithGoogleAuth(ctx, region, projectID)` that returns an `option.RequestOption`. This means the standard `anthropic.NewClient()` is used with the vertex option — all existing streaming, thinking, beta API, and message conversion code works unchanged.

## Target Usage

```yaml
models:
  claude_vertex:
    provider: anthropic
    model: claude-opus-4-6@20250514
    max_tokens: 150000
    provider_opts:
      project: my-gcp-project
      location: us-central1
```

Auth: Google Cloud ADC (`gcloud auth application-default login`), no `ANTHROPIC_API_KEY` needed.

## Changes

### 1. `pkg/model/provider/anthropic/client.go`

- **Add import**: `"github.com/anthropics/anthropic-sdk-go/vertex"`
- **Add `providerOption` helper** (same pattern as Gemini provider)
- **Add `isVertexAI` helper**: returns true when `provider_opts` has `project` or `location`
- **Modify `NewClient`**: In the `gateway == ""` branch, check `isVertexAI(cfg)` first. If true:
  - Extract and validate `project` and `location` via `environment.Expand`
  - Call `vertex.WithGoogleAuth(ctx, location, project)` wrapped in `recover()` (the SDK panics on auth failure instead of returning an error)
  - Create client with `anthropic.NewClient(vertexOpt)` and assign to `clientFn`
  - Skip `ANTHROPIC_API_KEY` requirement
- Existing direct-API path becomes the `else` branch — no changes to it

### 2. `pkg/config/gather.go`

- In the `"anthropic"` case (~line 106), only require `ANTHROPIC_API_KEY` when `provider_opts` does NOT have `project`/`location` set. Mirrors the `"google"` case pattern.

### 3. Tests

- **`pkg/model/provider/anthropic/client_test.go`**: Add tests for `isVertexAI`, `providerOption`, and Vertex AI validation (missing project/location errors)
- **`pkg/config/testdata/env/anthropic_vertex_model.yaml`**: New test fixture for gather.go env var detection

### 4. Update user's `daily_tasks_agent.yaml`

After the patch, update the config to use `provider: anthropic` with Vertex AI `provider_opts`:

```yaml
models:
  vertex_opus:
    provider: anthropic
    model: claude-opus-4-6@20250514
    max_tokens: 150000
    provider_opts:
      project: ai-cobypear-000000
      location: global
  vertex_haiku:
    provider: anthropic
    model: claude-haiku-4-5@20251001
    max_tokens: 60000
    provider_opts:
      project: ai-cobypear-000000
      location: global
  vertex_sonnet:
    provider: anthropic
    model: claude-sonnet-4-5
    max_tokens: 30000
    provider_opts:
      project: ai-cobypear-000000
      location: global
```

## Verification

1. `go build ./...` — compiles
2. `go test ./pkg/model/provider/anthropic/... ./pkg/config/...` — tests pass
3. Manual test with a Vertex AI-enabled GCP project and ADC credentials
4. Verify existing `provider: anthropic` configs (without `provider_opts`) still work unchanged

## Notes

- The Files API (`/v1/files`) likely won't work through Vertex AI (the vertex middleware only rewrites `/v1/messages` paths). File upload will fail with a clear API error — acceptable for initial implementation.
- `vertex.WithGoogleAuth` panics on failure, so we wrap in `recover()` to surface a proper error.
