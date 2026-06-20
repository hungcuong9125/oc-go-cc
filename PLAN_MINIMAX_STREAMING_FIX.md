# PLAN: Fix streaming bug for `minimax-m3` (and other Go Anthropic-native models)

**Status:** Pending review - NOT yet applied
**Author:** Codebase investigation
**Date:** 2026-06-18
**Proposed branch:** `fix/minimax-streaming-anthropic-raw`

---

## 1. Problem summary

Streaming requests to model `minimax-m3` (and other Go models in `IsAnthropicModel` such as `qwen3.5-plus`, `qwen3.6-plus`, `qwen3.7-plus`) are consistently rejected by upstream with the error:

```
API error 400: {"type":"error","error":{"type":"invalid_request_error",
"message":"Error from provider (MiniMax): invalid params,
function name or parameters is empty (2013)"}}
```

After falling back to `deepseek-v4-pro`, streaming works normally. Non-streaming also works (because it goes through a different path).

## 2. Root cause (verified twice)

**Streaming path is missing a branch for Go Anthropic-native models.**

Comparison of the two paths in `internal/handlers/messages.go`:

| Path | Non-streaming (line ~601) | Streaming (line ~341-399) |
|------|---------------------------|---------------------------|
| Zen model + Anthropic endpoint | Sends raw body (line 593) | Sends raw body (line 343-358) |
| Zen model + Responses/Gemini | Separate transform | Separate transform |
| **Go + `IsAnthropicModel`** | **Sends raw body (line 601-603)** | **MISSING - falls through to OpenAI transform (line 397)** |
| OpenAI-compatible | Transform (line 605) | Transform (line 397) |

Consequence: with `minimax-m3` in streaming:
1. `handleStreaming` does not recognize the Anthropic-native model on the Go provider
2. Falls through to `h.requestTransformer.TransformRequest(anthropicReq, model)` (line 397)
3. `transformTools` (line 573) converts tools from Anthropic `{name, input_schema}` to OpenAI `{type:"function", function:{name, parameters}}`
4. Sends to `https://opencode.ai/zen/go/v1/messages` with OpenAI-format body
5. MiniMax upstream receives OpenAI format, does not find `name`/`input_schema` at the top level as it expects → returns 400

**Secondary cause**: `transformTools` currently with `input_schema: {}` produces `parameters: {}` — an empty object, which also triggers validation failure upstream. This fix helps other OpenAI models (kimi/glm/mimo) reduce schema errors but does NOT replace the streaming fix.

## 3. Implementation plan (3 phases, in order)

### Phase 1: Fix streaming branch (REQUIRED)

**File:** `internal/handlers/messages.go`

**Location:** In `handleStreaming`, right after the existing `if client.IsZen(model) { ... }` block and before the OpenAI-compatible section (`TransformRequest`).

**Structure to fix:** change the current Zen block into an `if / else if` chain. Do NOT insert a block starting with `} else if` before `if client.IsZen(model)` as that would be syntactically incorrect.

```go
if client.IsZen(model) {
    endpointType := client.ClassifyEndpoint(model.ModelID)
    switch endpointType {
    case client.EndpointAnthropic:
        // existing Zen Anthropic path
    case client.EndpointResponses:
        // existing Responses path
    case client.EndpointGemini:
        // existing Gemini path
    default:
        // Fall through to OpenAI-compatible handling
    }
} else if client.IsAnthropicModel(model.ModelID) {
    // NEW: Go Anthropic-native path
}
```

**Code for the new branch** (placed right after the closing `}` of the `if client.IsZen(model) { ... }` block, copying structure from the Zen Anthropic block lines 343-358):

```go
} else if client.IsAnthropicModel(model.ModelID) {
    // Go provider Anthropic-native models (MiniMax, Qwen) in streaming.
    // This branch is only reached after client.IsZen(model) is false.
    // Mirror the non-streaming path at executeAnthropicRequest (line 601)
    // which sends the raw Anthropic body. TransformRequest would otherwise
    // convert tools to OpenAI format and trigger upstream 400
    // "function name or parameters is empty (2013)" on the
    // /v1/messages Anthropic endpoint.
    modelBody := replaceModelInRawBody(rawBody, model.ModelID)
    if err := h.handleAnthropicStreaming(ctx, rw, modelBody, model.ModelID, model); err != nil {
        cancel()
        if clientCtx.Err() == context.Canceled {
            h.logger.Info("client disconnected during anthropic stream")
            return
        }
        h.logger.Warn("anthropic streaming failed", "model", model.ModelID, "error", err)
        continue
    }
    cancel()
    latency := time.Since(streamStart)
    h.metrics.RecordSuccess(model.ModelID, latency)
    h.logger.Info("streaming completed", "model", model.ModelID, "latency", latency)
    return
}
```

**Do not use a merged condition** like `if client.IsZen(model) || ...`: Zen must still go through `ClassifyEndpoint`, while Go Anthropic-native only needs the raw Anthropic branch. Using `else if client.IsAnthropicModel(model.ModelID)` is sufficient because it only runs when `client.IsZen(model)` is already false.

**Recommended addition alongside Phase 1:** harden `replaceModelInRawBody`.

The current helper searches for the exact string `"model":"`. If the request body has valid whitespace like `"model": "claude-opus-4-8"`, the helper won't replace the model and upstream will receive the wrong model. Since the new branch depends on this helper, DEV should change it to JSON-based replacement:

1. `json.Unmarshal(rawBody, &map[string]json.RawMessage{})`
2. Marshal `modelID` into a JSON string and assign to key `"model"`
3. `json.Marshal` the object again
4. If unmarshal/marshal fails or `model` is missing, log a warning and fall back to `rawBody`

This also makes the Zen Anthropic path more robust and requires separate tests for compact JSON, pretty/whitespace JSON, and missing `model`.

### Phase 2: Regression tests (REQUIRED)

**New file or add to:** `internal/handlers/messages_test.go`

**Objective:** Lock behavior — when streaming to `minimax-m3` via the Go provider, the body sent upstream MUST have `tools[].input_schema` (Anthropic), and MUST NOT have `tools[].function.parameters` (OpenAI).

**Proposed P0 test structure:** call `h.handleStreaming(...)` directly in the `handlers` package to test the correct branch, without depending on the token counter or routing.

1. Spin up `httptest.NewServer` as a fake upstream.
2. Server records the received request body into a channel/buffer.
3. Server returns a fake SSE response (1 `message_start` event + 1 `message_stop` + `data: [DONE]`).
4. Build `Config` with:
   - `OpenCodeGo.AnthropicBaseURL` = URL of the httptest server
   - `OpenCodeGo.BaseURL` = URL of another httptest server that should fail the test if called incorrectly
   - `APIKey` = "test-key"
5. Build `MessagesHandler` with:
   - `client` = `client.NewOpenCodeClient(atomicCfg)`
   - `streamHandler` = `transformer.NewStreamHandler()`
   - `metrics` = `metrics.New()`
   - `logger` = `slog.Default()`
6. Build `rawBody` Anthropic with `model: "claude-opus-4-8"`, `stream: true`, and `tools: [{name: "Bash", input_schema: {...}}]`
7. Parse `rawBody` into `types.MessageRequest`
8. Call `h.handleStreaming(recorder, req, &anthropicReq, []config.ModelConfig{{Provider:"opencode-go", ModelID:"minimax-m3"}}, rawBody)`
9. Assert: captured upstream body has `model == "minimax-m3"`, `tools[0].name == "Bash"`, `tools[0].input_schema` exists, and NO `function` field.

**Full `HandleMessages` test to cover routing end-to-end:** enable `RespectRequestedModel: true`, add `Models["minimax-m3"] = {Provider:"opencode-go", ModelID:"minimax-m3"}`, and send a request body with `model:"minimax-m3"`. Without forcing this, streaming might be routed by `RouteForStreaming` to the `fast` scenario instead of `default`.

**Test cases to cover:**
- P0: `TestHandleStreaming_GoAnthropicModel_SendsRawAnthropicBody` — core branch test
- P0: `TestHandleStreaming_GoAnthropicModel_ReplacesModelInBody` — verify model changed from requested model to routed model
- P0: `TestReplaceModelInRawBody_HandlesWhitespace` — guard for JSON-based helper
- P0: `TestReplaceModelInRawBody_ReturnsOriginalWhenModelMissing` — fallback behavior
- P1: `TestHandleMessages_StreamingRequestedMinimax_UsesAnthropicEndpoint` — full handler/routing test if desired
- P1: `TestHandleStreaming_GoAnthropicModel_FallsThroughOnError` — verify fallback to next model

**Reference pattern:** `internal/handlers/health_test.go` already has a pattern using `httptest.NewRecorder`.

### Phase 3: Harden `transformTools` (RECOMMENDED but not required)

**File:** `internal/transformer/request.go`

**Objective:** Increase robustness for OpenAI-compatible models (kimi, glm, mimo, qwen) — prevent upstream rejection due to empty schemas or missing fields.

**Proposed code** (based on prior team report, with refinements):

```go
func (t *RequestTransformer) transformTools(tools []types.Tool) []types.ToolDef {
    var result []types.ToolDef

    for _, tool := range tools {
        name := strings.TrimSpace(tool.Name)
        if name == "" {
            continue
        }

        schema := tool.InputSchema
        switch {
        case len(schema) == 0, string(schema) == "null", string(schema) == "{}":
            schema = []byte(`{"type":"object","properties":{},"additionalProperties":false}`)
        default:
            var schemaObj map[string]interface{}
            if err := json.Unmarshal(schema, &schemaObj); err != nil {
                schema = []byte(`{"type":"object","properties":{},"additionalProperties":false}`)
            } else {
                if _, ok := schemaObj["type"]; !ok {
                    schemaObj["type"] = "object"
                }
                if _, ok := schemaObj["properties"]; !ok {
                    schemaObj["properties"] = map[string]interface{}{}
                }
                if fixed, err := json.Marshal(schemaObj); err == nil {
                    schema = fixed
                }
            }
        }

        result = append(result, types.ToolDef{
            Type: "function",
            Function: types.FunctionDef{
                Name:        name,
                Description: tool.Description,
                Parameters:  json.RawMessage(schema),
            },
        })
    }

    return result
}
```

**Imports needed:** none for `request.go`; `encoding/json` and `strings` are already present.

**Tests to add** in `internal/transformer/request_test.go`:
- `TestTransformTools_SkipsEmptyName`
- `TestTransformTools_FillsEmptySchema`
- `TestTransformTools_FillsMissingType`
- `TestTransformTools_FillsMissingProperties`
- `TestTransformTools_RecoversFromInvalidJSON`
- `TestTransformTools_PreservesValidSchema` (regression for normal case)

**Scope recommendation:** harden at minimum `transformTools` in `request.go` first, since this is the OpenAI Chat Completions path related to the reported errors. Do not simultaneously apply to `transformToolsForResponses` and `transformToolsForGemini` without adding endpoint-specific tests; if expanding, separate into a dedicated commit/refactor with a `sanitizeToolSchema(schema json.RawMessage) json.RawMessage` helper.

## 4. Test strategy

**After Phase 1, before Phase 2:** run `go test ./...` to ensure the fix does not break existing tests.

**After Phase 2:** run `go test ./... -count=1` + new tests must pass. If time permits, also run `make test` to cover `-race` per the Makefile.

**After Phase 3:** run `go test ./... -count=1` again + verify the new `transformTools` tests pass.

**Manual smoke test (after building the binary):**
```bash
# Set OC_GO_CC_API_KEY pointing to a real upstream
make build
./bin/oc-go-cc &
# Trigger an Anthropic request with model minimax-m3, streaming=true, tools with both full and empty schemas
# Verify: streaming succeeds, no more "function name or parameters is empty" logs
```

**Regression check:** Routing for other OpenAI models (kimi, glm, mimo, deepseek) must have unchanged behavior. `make lint` may fail if DEV machine does not have `golangci-lint` installed; if so, note the missing tool but do not skip `gofmt`/`go test`.

## 5. Risks and mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Phase 1 accidentally routes Zen models to the Go branch | Medium | Use `if client.IsZen(model) { ... } else if client.IsAnthropicModel(model.ModelID) { ... }`; add test for Zen Anthropic model if time permits |
| `replaceModelInRawBody` cannot find `"model":"` in unusual format | Medium | Switch to JSON-based replacement in Phase 1 and test compact/pretty/missing-model |
| JSON-based `replaceModelInRawBody` changes field order/whitespace format | Low | Anthropic endpoint receives JSON semantically, not dependent on order; assert using parsed JSON rather than string |
| Phase 3 changes parameters key order format | Low | `json.Marshal` on a map does not preserve order, but upstream validators don't care about key order — confirm with smoke test |
| Test missing fallback chain case | Medium | Add a separate test for the case where a Go model fails → falls back to the next OpenAI model |
| `transformTools` hardening breaks existing tests | Medium | Run `go test ./internal/transformer/...` immediately after the change |

## 6. Rollback plan

Phase 1 and Phase 2 should be a single atomic commit (fix + regression tests), rollback via `git revert`. Phase 3 is a separate commit, rollback independently.

If streaming encounters new issues after Phase 1:
1. `git revert` the Phase 1 commit
2. Debug via upstream logs
3. Consider: should `transformTools` hardening (Phase 3) be kept as a temporary measure?

## 7. Acceptance criteria

- [ ] `go test ./... -count=1` passes entirely
- [ ] `make lint` passes
- [ ] `make build` succeeds
- [ ] Manual smoke test: streaming `minimax-m3` with 29 tools (as in user logs) works, no more error 2013
- [ ] Manual smoke test: streaming other OpenAI models (kimi, glm, mimo) does not regress
- [ ] Manual smoke test: non-streaming `minimax-m3` does not regress
- [ ] New Phase 2 tests cover all P0 cases: raw Anthropic body, model replacement, whitespace replacement, missing-model fallback
- [ ] New Phase 3 tests cover the 6 cases listed in Phase 3

## 8. Out of scope

- No changes to routing logic (`internal/router/`)
- No changes to config schema
- No refactoring of `transformToolsForResponses`/`transformToolsForGemini` in Phase 1/2. If Phase 3 is expanded to those endpoints, separate into a dedicated commit/refactor with endpoint-specific tests.
- No changes to thinking/reasoning_content logic in assistant messages (already has guard for DeepSeek)
- No changes to config example

## 9. Timeline estimate

- Phase 1: 25-35 minutes (fix streaming branch + harden `replaceModelInRawBody`)
- Phase 2: 60-90 minutes (write tests, debug httptest setup)
- Phase 3: 30 minutes (fix + tests)
- **Total: ~1.5-2 hours**, excluding review

## 10. References

- User log: `16:54:36` -> `16:54:38` shows ~1.5s fail latency (fast upstream rejection)
- Team report 1: proposes `transformTools` hardening
- Team report 2: identifies root cause as missing streaming branch
- `internal/handlers/messages.go:601` — non-streaming reference
- `internal/handlers/messages.go:343-358` — Zen Anthropic streaming reference
- `internal/handlers/health_test.go` — httptest pattern reference
