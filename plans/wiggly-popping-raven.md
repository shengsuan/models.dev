# Add ShengSuanYun provider

## Context

The user asked to add a new AI model provider, **ShengSuanYun** (`https://router.shengsuanyun.com/api/v1/models`), following the conventions documented in `README.md` (static provider/model TOML schema) and `sync.md` (automated sync provider architecture). ShengSuanYun is an OpenRouter-shaped multi-model relay (aggregator) exposing 194 models across many upstream labs (Alibaba/Qwen, Zhipu/GLM, ByteDance Seed, Anthropic, OpenAI, Google, DeepSeek, Moonshot/Kimi, MiniMax, Tencent, xAI/Grok, Xiaomi, Baidu, StepFun, Meta, Meituan/LongCat, StreamLake, InternLM).

Two decisions the user made explicitly (overriding my more conservative default recommendations):
1. **Auto-create all 194 models**, not just the subset that cleanly resolves to an existing canonical `base_model`. Models without a clean canonical match get heuristic capability guesses and stand-alone (non-factored) TOMLs.
2. **Register the provider in the existing `aggregators` sync group** so it runs on the hourly CI sync cron, like OpenRouter/Kilo/Requesty.

Empirically confirmed (via curl) that `GET /v1/models` needs no auth (200) while `POST /v1/chat/completions` needs an `Authorization` header (401 without one) — same public-catalog/authenticated-inference shape as OpenRouter/Kilo. This means `fetchModels()` needs no secret, and the AI SDK type should be `@ai-sdk/openai-compatible` with `api = "https://router.shengsuanyun.com/api/v1"`.

Because this is a first-time sync of 194 new models, it will vastly exceed `MAX_CREATED_MODELS`/`MAX_MODEL_CHURN` in `packages/core/src/sync/auto-merge.ts:4-6` (10/15), so the first sync PR will not auto-merge in CI and needs manual review — this is expected and not a blocker.

## Source API shape (from empirical fetch, `/tmp/ssy.json`)

Each entry: `{ company, name, api_name, description (zh), max_tokens, context_window, supports_prompt_cache, architecture: { input, output, tokenizer }, pricing: { prompt, completion, cache, image, request }, id, support_apis }`.

- `id` === `api_name`, shaped `<org>/<model>[:thinking]`, e.g. `anthropic/claude-sonnet-4.5:thinking`, `ali/qwen3.7-max`, `bigmodel/glm-4.6`.
- `pricing.prompt/completion/cache` are raw units; **divide by 10,000** to get USD per 1M tokens (empirically verified against known GPT-5/Gemini/Grok/DeepSeek prices). `pricing.image`/`pricing.request` are always 0 across all 194 models — ignore them.
- `architecture.input`/`architecture.output` are `+`-joined modality strings (`"text"`, `"image+text"`, `"text+image+audio+video"`, or `""` meaning text-only) — split on `+`, lowercase, map to the `Modality` enum.
- `supports_prompt_cache` is unreliable (85/194 models report `false` despite nonzero `pricing.cache`) — use `pricing.cache > 0` directly as the signal for emitting `cost.cache_read`, ignore the boolean field.
- No `created`/timestamp field anywhere — `release_date`/`last_updated` cannot be derived from source data for standalone (non-factored) models.
- Reasoning signal: only the literal `:thinking` ID suffix (e.g. `bigmodel/glm-4.6:thinking`, `anthropic/claude-sonnet-4.5:thinking`, `ali/qwen-plus-latest:thinking`). No other capability flags (`tool_call`, `structured_output`) are exposed anywhere in the payload — these must be best-effort guesses.

## Org prefix → canonical provider mapping

`resolveCanonicalBaseModel`/`resolveModelMetadataBaseModel` in `packages/core/src/sync/providers/openrouter.ts` already handle `anthropic`, `openai`, `google`, `deepseek`, `x-ai`/`xai`, `tencent`, `xiaomi`, `moonshot`/`moonshotai`, `minimax`, `meta`/`meta-llama` (→`llama`), `stepfun`/`stepfun-ai` prefixes directly, including anthropic dot→dash and llama-4-scout/maverick-17b suffix candidate logic. ShengSuanYun IDs using these prefixes can route straight through `resolveModelMetadataBaseModel(id)` after prefix remap (see below) with **no new code needed** for those.

ShengSuanYun-specific prefixes needing a local remap table before delegating to the shared resolver:

| SSY prefix | canonical metadata dir | notes |
|---|---|---|
| `ali` | `alibaba` | e.g. `ali/qwen3-max` → try `alibaba/qwen3-max` |
| `bigmodel` | `zhipuai` | e.g. `bigmodel/glm-4.6` → `zhipuai/glm-4.6`; strip `:thinking` before lookup |
| `bytedance` | `bytedance-seed` | ID format differs: `doubao-seed-2-0-pro` → try `seed-2.0-pro` (translate `doubao-seed-` prefix → `seed-`, and last two `-N-M` numeric segments → `N.M`); `doubao-pro-256k` has no canonical match |
| `longcat` | `meituan` | e.g. `longcat/longcat-2.0` → `meituan/longcat-2.0` |
| `baidu`, `streamlake`, `intern` | *(none)* | no existing canonical `models/` entries — these always fall through to standalone models |

Implementation approach: write a small `SHENGSUANYUN_PREFIX_REMAP` table (`ali→alibaba`, `bigmodel→zhipuai`, `bytedance→bytedance-seed`, `longcat→meituan`) plus a `translateBytedanceID()` helper for the numeric-dash-to-dot rewrite, then try, in order: (1) exact-remap candidate via `resolveModelMetadataBaseModel`, (2) fall back to `resolveModelMetadataBaseModel(originalID)` unchanged (covers prefixes already known to `CANONICAL_PROVIDER_PREFIXES`, e.g. `anthropic/`, `openai/`, `x-ai/`, `tencent/`, `xiaomi/`, `moonshot/`, `minimax/`, `meta/`, `stepfun/`, `google/`, `deepseek/`). Always strip a trailing `:thinking` before resolution and try both the stripped and un-stripped ID (mirroring `nano-gpt.ts`'s `NANO_GPT_VARIANT_SUFFIX` / `stripNanoGptVariantSuffixes` pattern) since some upstream files exist per-variant (e.g. `qwen3-next-80b-a3b-thinking.toml`) while others share one file for both modes (e.g. `glm-4.6.toml` has no separate thinking file).

Models with no canonical match at all (`baidu/*`, `streamlake/*`, `intern/*`, `bytedance/doubao-pro-256k`, and any other unresolved ID) become **standalone full models**, per the user's explicit "auto-create everything" decision — not skipped.

## Capability heuristics for standalone (non-base_model) models

Per the user's explicit choice to auto-create all 194 models with best-effort guesses:

- `reasoning`: `true` if the ID ends with `:thinking`, else `false`. When `true`, set `reasoning_options = []` (empty array, matching the `openrouter.ts` pattern of leaving effort/budget unspecified when the API gives no signal — see `providers/openrouter/models/z-ai/glm-4.6.toml`'s `reasoning_options = []`).
- `tool_call`: default `true` (the API gives no signal, but essentially all modern chat-completions-shaped relay models support tool calling; this matches the "best-effort guess" instruction and errs toward the common case).
- `structured_output`: leave `undefined` (omit) rather than guessing `true`/`false` — matches `AuthoredModel`'s optional field and avoids overclaiming a specific JSON-schema-mode guarantee with zero evidence either way.
- `attachment`: `true` if `architecture.input` contains any non-text modality.
- `family`: reuse the existing duplicated `inferFamily(id, name)` pattern (regex match against `ModelFamilyValues` from `packages/core/src/family.ts`, plus `inferKimiFamily()` first) — copy the same ~15-line helper used in `openrouter.ts`/`chutes.ts`/`kilo.ts`/`requesty.ts` into the new module (established repo convention of small per-provider duplication over shared abstraction).
- `release_date`/`last_updated`: no timestamp in source data. Use `existing?.release_date ?? today` / `existing?.last_updated ?? today` (matches the Tinfoil/Venice/Chutes precedent for endpoints lacking creation timestamps).
- `description`: `existing?.description ?? describeModel({...})` (from `packages/core/src/describe.ts`), not the API's raw Chinese-language `description` field (schema requires English/latin catalog conventions elsewhere in the repo; other providers do not use the raw source description).
- `name`: `existing?.name ?? humanizeModelName(model.name-or-id)` — the API's own `name` field (e.g. "DeepSeek-V4-Pro-0813", "Claude Sonnet 4.5 Thinking") is generally already presentable and can be used directly (unlike Chutes which had to humanize a raw slug).
- `limit`: `context = model.context_window`, `output = model.max_tokens`.
- `cost`: `input = pricing.prompt/10000`, `output = pricing.completion/10000`, `cache_read = pricing.cache > 0 ? pricing.cache/10000 : undefined`.
- `temperature`: default `true` (no signal either way; matches the repo-wide default assumption used in Chutes for the same absent-signal scenario).
- `open_weights`: `existing?.open_weights ?? false` (ShengSuanYun is a hosted relay; default to closed/unknown rather than guessing open-source per-model without a Hugging Face signal).

For models that DO resolve to a `base_model`, use `factorBaseModel(baseModel, values, limit, existing?.base_model_omit)` (from `openrouter.ts`) exactly like `chutes.ts`/`tinfoil.ts` do — this only writes real overrides (cost, limit, reasoning_options if thinking) and inherits the rest from canonical metadata, so the heuristic guesses above mostly only matter for the ~40-60 unresolved standalone models.

## Files to create

1. **`providers/shengsuanyun/provider.toml`**
   ```toml
   name = "ShengSuanYun"
   env = ["SHENGSUANYUN_API_KEY"]
   npm = "@ai-sdk/openai-compatible"
   api = "https://router.shengsuanyun.com/api/v1"
   doc = "https://router.shengsuanyun.com/api/v1/models"
   ```
   (No further documentation could be retrieved externally — WebFetch on the doc/marketing pages returned empty JS-rendered content, and WebSearch is unavailable in this environment. Point `doc` at the models endpoint itself, matching the Chutes precedent (`doc = "https://llm.chutes.ai/v1/models"`) for providers with no separate docs page found.)

2. **`providers/shengsuanyun/logo.svg`** — simple `currentColor`, no fixed size/colors, per README's logo convention. Use a generic placeholder glyph (e.g. a simple geometric mark) since no official brand asset was retrievable.

3. **`packages/core/src/sync/providers/shengsuanyun.ts`** — new sync provider module, structurally modeled on `chutes.ts` (prefix-remap table + `resolveModelMetadataBaseModel`/`factorBaseModel` reuse) and `nano-gpt.ts` (`:thinking`-suffix stripping pattern). Exports:
   - `ShengSuanYunModel`/`ShengSuanYunResponse` Zod schemas matching the shape above.
   - `shengsuanyun: SyncProvider<ShengSuanYunModel>` object (`id: "shengsuanyun"`, `name: "ShengSuanYun"`, `modelsDir: "providers/shengsuanyun/models"`, `fetchModels`/`parseModels`/`translateModel`).
   - `resolveShengSuanYunBaseModel(id)` exported for testing (mirrors `resolveNanoGptBaseModel`/`resolveBaseModel` exports in sibling modules).
   - `buildShengSuanYunModel(model, existing, today?)` exported, implementing the heuristics above.

4. **`packages/core/test/shengsuanyun.test.ts`** — small test file following `packages/core/test/empiriolabs.test.ts`'s pattern: assert `resolveShengSuanYunBaseModel` correctly resolves known-good IDs (`anthropic/claude-sonnet-4.5:thinking` → `anthropic/claude-sonnet-4-5-...` or whatever the real canonical file is, `ali/qwen3-max` → `alibaba/qwen3-max`, `bigmodel/glm-4.6` → `zhipuai/glm-4.6`, `bytedance/doubao-seed-2-0-pro` → `bytedance-seed/seed-2.0-pro`, `longcat/longcat-2.0` → `meituan/longcat-2.0`) and correctly returns `undefined` for unresolvable ones (`baidu/ernie-4.0-turbo-128k`, `streamlake/kat-coder-pro-v1`, `intern/intern-s1`).

## Files to modify

5. **`packages/core/src/sync/index.ts`**:
   - Add `import { shengsuanyun } from "./providers/shengsuanyun.js";` (alphabetically ordered with existing imports, between `requesty` and `tinfoil`).
   - Add `shengsuanyun: SyncProvider<any>;` to the `providers` type block and `shengsuanyun,` to the `providers` object (alphabetical, same position).
   - Add `"shengsuanyun"` to the `aggregators` array in `groups` (alphabetical among existing entries, per the user's explicit decision).

6. **`sync.md`**: add a new `## ShengSuanYun Notes` section after `## Requesty Notes`, documenting: source endpoint, no-auth catalog / authed inference shape, the org-prefix remap table, the `:thinking`-suffix reasoning heuristic, the lack of a `created` timestamp, and that `tool_call`/`structured_output` are unauthenticated best-effort guesses (not API-verified) — matching the candor of the existing per-provider notes (e.g. Chutes' `chat_template_kwargs` caveat).

## Verification

1. `bun models:sync shengsuanyun --dry-run` — inspect the diff; confirm ~194 creates, spot-check a handful of both base_model-factored entries and standalone entries for sane values (cost divided correctly, modalities right, `:thinking` models show `reasoning = true`).
2. `bun models:sync shengsuanyun` — write files.
3. `bun models:sync shengsuanyun --dry-run` again — expect a clean (no-diff) result.
4. `bun validate` — full schema validation across the repo.
5. `bun test packages/core/test/shengsuanyun.test.ts` — run the new resolver unit tests.
6. Manually skim a sample of generated `providers/shengsuanyun/models/**/*.toml` files (a base_model-factored one and a standalone one) to sanity-check output shape against hand-authored siblings (e.g. `providers/openrouter/models/z-ai/glm-4.6.toml`, `providers/chutes/models/...`).


GET https://router.shengsuanyun.com/docs   404