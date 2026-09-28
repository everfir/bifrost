# Everfir fork

Upstream: https://github.com/maximhq/bifrost

Company fork: https://github.com/everfir/bifrost

## Scope

The Anthropic Responses converter preserves top-level `instructions` followed by
leading system/developer messages. Existing content blocks and cache-control
metadata are retained. The converter also serves Anthropic-compatible provider
paths; native raw-body passthrough is unchanged.

This patch does not establish full Codex/Opus compatibility. Freeform tools and
the Responses `parallel_tool_calls: false` mapping remain separate limitations.

## Go dependency

The reusable module is `core/`, requiring Go 1.27.0 or later. Keep upstream
imports (`github.com/maximhq/bifrost/core/...`) and replace the module with the
company fork at a fixed commit. Do not import both module paths in one program.

Run from the consuming Go module, substituting the reviewed fork commit:

```sh
go mod edit -replace=github.com/maximhq/bifrost/core=github.com/everfir/bifrost/core@<reviewed-commit>
go get github.com/maximhq/bifrost/core
go mod tidy
```

Go resolves the commit to a pseudo-version in `go.mod`. Commit `go.mod` and
`go.sum` in the consuming project. No gateway deployment is needed to import the
core library. Keep changes on the company's `dev` branch; fetch upstream and
review incoming changes before updating the pinned dependency.

## Verification

The regression in `core/providers/anthropic/requestbuilder_test.go` checks the
serialized upstream body for streaming and non-streaming requests, instruction
ordering, cache metadata, and repeat conversion without input mutation. Before
the fix, coexistence cases fail because the base instruction is missing.

```sh
cd core
go test ./providers/anthropic -skip '^TestAnthropic$' -count=1
```

Provider-harness folder `119. Anthropic instructions coexistence` checks the
actual upstream request returned by `x-bf-send-back-raw-request`. It requires a
configured Anthropic key and makes paid calls. The live run is separate from
the local unit and module-consumption checks.

From the repository root, ensure port 8080 is free, start the gateway, then run
the last command in another terminal after `/health` responds:

```sh
lsof -nP -iTCP:8080 -sTCP:LISTEN
make dev APP_DIR=$(pwd)/tests/integrations/python
make run-provider-harness-test PROVIDER=anthropic FEATURE="instructions coexistence"
```
