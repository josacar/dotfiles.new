---
name: opencode-go-model-sync
description: Synchronizes OpenCode Go model lists and per-model USD/token pricing with the local opencode client config, crush config, and the local bifrost proxy config on rock-3a. Use when the user asks to update opencode models, add new OpenCode Go models, refresh model lists, keep model costs in sync across configs, or re-sync after upstream price changes.
---

# OpenCode Go Model Sync

This skill synchronizes the model lists **and** per-model pricing from the OpenCode Go API with three local configs:

1. `dot_config/opencode/opencode.jsonc` in the chezmoi dotfiles repo (`~/.local/share/chezmoi`)
2. `dot_config/crush/crush.json` in the chezmoi dotfiles repo (`~/.local/share/chezmoi`)
3. `files/bifrost-config.json` in the rock-3a repo (`~/code/rock-3a`)

## When to Use This Skill

Use this skill when the user:

- Asks to "update opencode models"
- Asks to "add new models" from OpenCode Go
- Wants to sync the local bifrost proxy model list
- Wants to sync model costs/pricing across opencode, crush, and bifrost
- Mentions new models appeared on `https://opencode.ai/zen/go/v1/models`
- Mentions an upstream price change to a known model
- Wants crush config kept in sync
- Wants all three configs kept in sync

## Sources of Truth

- **Model list API**: `https://opencode.ai/zen/go/v1/models`
- **Pricing/docs**: `https://opencode.ai/docs/go`
- **Bifrost config schema**: `https://www.getbifrost.ai/schema` — see the `$defs.provider_pricing_override` definition for the `governance.pricing_overrides` schema. The supported `pricing_patch` fields are documented at `https://docs.getbifrost.ai/providers/custom-pricing`.

## Files to Update

| File | Repo | Purpose |
|------|------|---------|
| `dot_config/opencode/opencode.jsonc` | `~/.local/share/chezmoi` | OpenCode client provider/model definitions |
| `dot_config/crush/crush.json` | `~/.local/share/chezmoi` | Crush provider/model definitions |
| `files/bifrost-config.json` | `~/code/rock-3a` | Bifrost proxy allowed-model lists (`providers.<provider>.keys[].models`) AND per-model USD/token pricing (`governance.pricing_overrides`) |

## Step-by-Step Workflow

### 1. Fetch Upstream Model List

Use the `fetch` tool on `https://opencode.ai/zen/go/v1/models` to get the current model IDs.

### 2. Identify Providers

Map each model ID to the correct provider based on the OpenCode Go docs endpoint table:

- **OpenAI-compatible endpoint** (`/zen/go/v1/chat/completions`) → `opencode-go` provider
  - Examples: `glm-*`, `kimi-*`, `deepseek-*`, `mimo-*`, `hy3-preview`
- **Anthropic endpoint** (`/zen/go/v1/messages`) → `opencode-go-anthropic` provider
  - Examples: `minimax-*`, `qwen3.*`
- **Free models** (`/zen` endpoint) → `opencode-go-free` provider
  - Examples: `*-free`

### 3. Update the OpenCode Client Config

Edit `dot_config/opencode/opencode.jsonc`:

- Add missing models under the correct provider's `models` object.
- Use the existing entry for the same model family as a template for:
  - `family`
  - `cost` (from docs pricing table)
  - `limit` (`context` and `output` token limits)
  - `modalities`
  - `reasoning` and `interleaved` when the model supports reasoning
- Free models have zero cost, no reasoning, and typically `output: 16384`.
- Preserve existing formatting and alphabetical/version ordering within families.

### 4. Update the Crush Config

Edit `dot_config/crush/crush.json`:

- Add missing models to `providers.bifrost.models`.
- Use the existing entry for the same model family as a template for:
  - `cost_per_1m_in`, `cost_per_1m_out`, `cost_per_1m_in_cached`, `cost_per_1m_out_cached`
  - `context_window`
  - `default_max_tokens`
  - `can_reason`
  - `supports_attachments`
- Free models have all costs set to `0` and typically `can_reason: false`.

### 5. Update the Bifrost Proxy Config

Edit `files/bifrost-config.json` in `~/code/rock-3a`:

- Add the new model ID to the `models` array of the matching provider under `providers.<provider>.keys[0].models`.
- Keep the array in the same order as the OpenCode client config when possible.

### 6. Add the Model's Cost to the Bifrost Pricing Overrides

Bifrost tracks per-model cost in `governance.pricing_overrides` (top-level key in the same `files/bifrost-config.json`) — **not** in the `providers.<provider>.keys[].models` array, which only carries allowed-model IDs.

The override schema is `provider_pricing_override` from `https://www.getbifrost.ai/schema`. Required fields:

- `id` — stable, unique string. Use `<provider>-<model-slug>-pricing` (e.g. `opencode-go-glm-5.2-pricing`).
- `name` — human label, mirroring the model name from the opencode.jsonc entry.
- `scope_kind` — `provider` for these models (cost is per-provider, not per-virtual-key).
- `provider_id` — matches the bifrost provider key (`opencode-go`, `opencode-go-anthropic`, `opencode-go-free`).
- `match_type` — `exact` (we pin each model individually).
- `pattern` — the bare model ID **without** the provider prefix (e.g. `glm-5.2`, not `opencode-go/glm-5.2`). This matches the value sent on the wire to bifrost.
- `request_types` — `["chat_completion"]` (covers both stream and non-stream; the per-stream request type is not in the enum).
- `pricing_patch` — a **JSON-encoded string** (not a nested object) of cost fields expressed as **USD per token**, not USD per million tokens. The crush config stores costs per 1M tokens, so divide by `1_000_000` when writing the patch.

  Field mapping from the crush entry:

  | Crush field (per 1M)        | `pricing_patch` field (per token) |
  |-----------------------------|-----------------------------------|
  | `cost_per_1m_in`             | `input_cost_per_token`            |
  | `cost_per_1m_out`            | `output_cost_per_token`           |
  | `cost_per_1m_in_cached`     | `cache_read_input_token_cost`     |
  | `cost_per_1m_out_cached`    | *(no equivalent in the public patch schema — omit)* |

  Only include `cache_read_input_token_cost` when the value is non-zero; per the schema *"only fields with non-zero values are applied."*

Example entry added for GLM-5.2 in `governance.pricing_overrides`:

```json
{
  "id": "opencode-go-glm-5.2-pricing",
  "name": "GLM-5.2 pricing",
  "scope_kind": "provider",
  "provider_id": "opencode-go",
  "match_type": "exact",
  "pattern": "glm-5.2",
  "request_types": ["chat_completion"],
  "pricing_patch": "{\"input_cost_per_token\":0.0000014,\"output_cost_per_token\":0.0000044,\"cache_read_input_token_cost\":0.00000026}"
}
```

#### Handling free models

Free models have all costs set to `0`. Because the Bifrost pricing patch *"only applies non-zero values,"* a zero-cost override is a no-op — **skip these models entirely**; do not add an entry to `governance.pricing_overrides`. They still need to be added to `providers.opencode-go-free.keys[0].models` (step 5).

#### Keeping pricing in sync when costs change

If the cost of an existing model changes upstream (e.g. GLM-5.2 input drops from $1.4 to $1.2 per 1M):

1. Update `dot_config/opencode/opencode.jsonc` — edit the existing entry's `cost` block.
2. Update `dot_config/crush/crush.json` — edit the existing entry's `cost_per_1m_*` fields.
3. Update `files/bifrost-config.json` — find the matching entry in `governance.pricing_overrides` by `pattern` (the bare model ID) and rewrite its `pricing_patch` string.

The pattern+provider pair is the lookup key; do **not** create a second override for the same model.

### 7. Validate JSON/JSONC

Verify all files remain valid (opencode.jsonc is JSON with comments; crush.json and bifrost-config.json are strict JSON).

For `bifrost-config.json`, validate `governance.pricing_overrides` entries against the `provider_pricing_override` definition in `https://www.getbifrost.ai/schema`:

- Every required field is present: `id`, `name`, `scope_kind`, `match_type`, `pattern`, `request_types`.
- `scope_kind: "provider"` entries include `provider_id`.
- `pricing_patch` is a *string* that parses as JSON; values are USD per token (not per 1M).
- IDs are unique; `pattern` matches exactly one model in the crush config.

### 8. Commit and Push the Repos

For each repo, in order:

1. `git status` and `git diff` to review changes.
2. Stage only the relevant files.
3. Commit with a clear, concise message (e.g., "Add GLM-5.2 to opencode, crush, and bifrost configs").
4. `git pull --rebase` if the remote has moved.
5. `git push origin main`.

## Example: Adding GLM-5.2

1. Fetch API and see `glm-5.2` is missing locally.
2. Add to `dot_config/opencode/opencode.jsonc` under `provider.bifrost.models`:

```json
"opencode-go/glm-5.2": {
  "name": "GLM-5.2",
  "family": "glm",
  "status": "active",
  "reasoning": true,
  "interleaved": { "field": "reasoning_content" },
  "cost": { "input": 1.4, "output": 4.4, "cache_read": 0.26 },
  "limit": { "context": 202752, "output": 32768 },
  "modalities": { "input": ["text"], "output": ["text"] }
}
```

3. Add an equivalent entry to `dot_config/crush/crush.json` under `providers.bifrost.models`:

```json
{
  "id": "opencode-go/glm-5.2",
  "name": "GLM-5.2",
  "cost_per_1m_in": 1.4,
  "cost_per_1m_out": 4.4,
  "cost_per_1m_in_cached": 0.26,
  "cost_per_1m_out_cached": 0,
  "context_window": 203000,
  "default_max_tokens": 16384,
  "can_reason": false,
  "supports_attachments": true
}
```

4. Add `"glm-5.2"` to `providers.opencode-go.keys[0].models` in `files/bifrost-config.json`.
5. Add a pricing override to `governance.pricing_overrides` in the same file:

```json
{
  "id": "opencode-go-glm-5.2-pricing",
  "name": "GLM-5.2 pricing",
  "scope_kind": "provider",
  "provider_id": "opencode-go",
  "match_type": "exact",
  "pattern": "glm-5.2",
  "request_types": ["chat_completion"],
  "pricing_patch": "{\"input_cost_per_token\":0.0000014,\"output_cost_per_token\":0.0000044,\"cache_read_input_token_cost\":0.00000026}"
}
```

6. Validate all three files parse, then commit and push both repos.

## Common Pitfalls

- **Do not** assume all models use the same endpoint. Always check the docs endpoint table.
- **Do not** forget the crush config. It uses a different schema than opencode.jsonc.
- **Do not** forget the bifrost proxy config — both the `models` allow-list under `providers.<provider>.keys[0].models` AND the per-model cost under `governance.pricing_overrides`. The two live in the same `files/bifrost-config.json` but serve different purposes.
- **Do not** store USD per million tokens in `pricing_patch` — the patch uses USD **per token**. Divide the crush `cost_per_1m_*` value by `1_000_000`.
- **Do not** add `pricing_patch` overrides for free models — zero-value fields are skipped by bifrost and have no effect.
- **Do not** create two overrides for the same model (one keyed by id, one by pattern). The pattern+provider pair is the lookup key.
- **Do not** commit unrelated changes in either repo.
- Free models are served from `https://opencode.ai/zen` (not `/zen/go`), so they belong to the `opencode-go-free` provider in bifrost.

## Notes

- The default model in opencode.jsonc is currently `bifrost/opencode-go/deepseek-v4-flash`. Changing the default requires an explicit user request.
- If the remote repo has moved ahead, rebase before pushing.
