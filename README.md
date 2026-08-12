# Contribution 2: Adapter: Zhipu GLM Model Provider

**Contribution Number:** 2 
**Student:** Jonathan Ballona Sanchez 
**Issue:** https://github.com/orthogonalhq/nous-core/issues/320 
**Status:** Phase 4 In Progress — PR [#429](https://github.com/orthogonalhq/nous-core/pull/429) merged Aug 11, 2026

---

## Why I Chose This Issue

I chose issue #320 because I’m interested in integrating and evaluating different LLM providers. Implementing the Zhipu GLM provider matches my experience with TypeScript, LLM APIs, RAG, and model evaluation while giving me an opportunity to learn more about Zhipu’s models and API behavior.

I previously contributed the Moonshot Kimi provider leaf to Nous. Working on another provider will help me deepen my understanding of the repository’s provider architecture, testing conventions, credential handling, and generated catalogs. My goal is to become familiar enough with Nous to make broader and more meaningful contributions in the future.


Status Notes so far: I want to work on this issue but I want to make sure that the AI301 course instructor approves my issue selection before I comment on the issue. I meet approval guidelines, but official approval is what I'm hoping for.

---

## Understanding the Issue

### Problem Description
Currently users can't select or route to the Zhipu's models. Zhipu exposes an OpenAI Chat Completions compatible API, so adding it should follow the standard OpenAI compatible leaf path documented in the provider-adapter guides: create a vendor leaf under `provider/zhipu/` that reuses the shared `protocols/openai-api/` protocol, then regenerate the catalogs so the generator discovers it. Existing leaves xai, groq, and mooshot are the templates.

### Expected Behavior

* A zhipu leaf exists at `self/subcortex/providers/src/providers/zhipu/` without the four required files:
    * `definition.ts` -> providerDefinition( as const staisfied ProviderDefinitionLeaf), metadata only: `vendorKey`: `'zhipu'`, `protocol: 'chat-completions'`, `adapterKey: 'chat-completions'`, default endpoint + model, `auth` (env var, vault namespace, bearer header, `required: true`, `purpose: 'api_key'`), `modelListEndpoint`/`modelListFormat: 'openai-models'`, capabilities
    * `adapter.ts` -> re-exports `providerAdapter` fromt eh shared openai-api adapter
    * `prvider.ts` -> `providerFactory` (`as const satisfies ProviderFactoryModule`) that builds a `ChatCompletionsProvider` from `ZHIPU_API_KEY` / explicit `apiKey`, failing closed when absent
    * `index.ts` -> re-exports the public surface
* After running `generate:providers`, `zhipu` appears in `PROVIDER_DEFINITIONS` and resolves via `resolveProviderDefinition('zhipu')`
* Requests hit the correct endpoint (no doubled version segment), and `parseResponse(...)` returns a text fallback instead of throwing on malformed output
* `check:generated`, `typecheck`, and the provider teest suite (including a new `zhipu.test.ts`) pass

### Current Behavior

There is no Zhipu provider anywhere in the codebase, no `providers/zhipu/` leaf, no entries in the generated catalogs (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`), and no tests. Zhipu/GLM models are unreachable.

### Affected Components

* New: `self/subcortex/providers/src/providers/zhipu/{definition,adapter,provider,index}.ts`
* New: `self/subcortex/providers/src/__tests__/providers/zhipu.test.ts` (modeled on `xai.test.ts`)
* Reused: `self/subcortex/providers/src/protocols/openai-api/` (shared adapter + `ChatCompletionsProvider`)
* Regenerated (do not hand-edit): `self/subcortex/providers/src/{provider-definitions,provider-adapters,provider-factories}.ts` via `pnpm --filter @nous/subcortex-providers run generate:providers`
* Contracts to satisfy: `self/subcortex/providers/src/schemas/{provider-definition,provider-adapter,provider-factory}.ts`

---

## Reproduction Process

### Environment Setup

#### Prerequesities (environment)
1. Install and wire nvm (macos users may use homebrew)
```bash
brew install nvm
mkdir -p ~/.nvm
cat << 'EOF' >> ~/.zshrc
export NVM_DIR="$HOME/.nvm"
[ -s "/opt/homebrew/opt/nvm/nvm.sh" ] && \. "/opt/homebrew/opt/nvm/nvm.sh"
[ -s "/opt/homebrew/opt/nvm/etc/bash_completion.d/nvm" ] && \. "/opt/homebrew/opt/nvm/etc/bash_completion.d/nvm"
EOF
source ~/.zshrc
```
2. Install pnpm and open new terminal subsequently
```bash
brew install pnpm
```
3. Install Node 22
```bash
nvm install 22
nvm use 22
```
#### Repo Setup
1. Forked `orthonalhq/nous-core` 
2. Clone forked repo locally and add upstream
```bash
git clone git@github.com:jballo/nous-core.git
cd nous-core
git remote add upstream https://github.com/orthogonalhq/nous-core.git
```
3. Fetch upstream and create feature branch from integration branch (not main):
```bash
git fetch upstream
git checkout -b feat/zhipu-glm-provider upstream/feat/contributor-friendly-inference-provider-surface
```
4. Install and verify
```bash
pnpm install
pnpm build
pnpm test
```
### Reproduction Evidence

* **Commit showing reproduction:** https://github.com/jballo/nous-core/commit/9604e2138f8c42bfa4ce05b2783c7c92bd84b48b
* **Screenshots/logs:**
   ![pnpm build result](https://ix7l8rtzlf.ufs.sh/f/fsfgZfmadWBrCacJjLsYyWGqkzhK04c6wj2ZnPfgl18MEvFX)
   ![pnpm test result](https://ix7l8rtzlf.ufs.sh/f/fsfgZfmadWBrUzwKtW9BTnKGmPoR5OC1wZXEJVdYkHpyftxA)
* **My findings:** Synced to `feat/contributor-friendly-inference-provider-surface` on branch `feat/zhipu-glm-provider`. After `pnpm install`, build passed. Code gen tests failed due to the order of llms in provider-codegen.test.ts. Second failure was due to missing Mistral API key required. These issue are not relevant to the issue, which in turn will not touch those issues to keep scope of commits for the sole purpose of adding zhipu provider.

---
## Analysis

**Root cause:** There is no Zhipu leaf, so the generator never discovers or catalogs the provider — Zhipu/GLM models are simply unreachable. This isn't a bug to fix but a missing driver package to add.

The one real technical trap is **endpoint path construction**. The shared `ChatCompletionsProvider` builds request URLs as `endpoint.replace(/\/$/, '') + completionsPath`, where `completionsPath` defaults to `/v1/chat/completions` (`protocols/openai-api/provider.ts:73,92`). Zhipu's OpenAI-compatible base already carries a version segment (`https://open.bigmodel.cn/api/paas/v4`), so the default would yield the wrong `…/v4/v1/chat/completions`. This is the same class of doubled-`/v1` defect that was just fixed for xAI (commits `a4dc1950`, `3e7a9749`). It must be handled by overriding `completionsPath` in the factory — exactly as Perplexity does (`providers/perplexity/provider.ts`).

A secondary trap is the **credential fallback boundary**: `ChatCompletionsProvider` falls back to `process.env.OPENAI_API_KEY` if no key is passed, so the Zhipu factory must resolve `ZHIPU_API_KEY` explicitly and fail closed, or a stray OpenAI key could be sent to Zhipu's endpoint (see the `#413` note in `perplexity/provider.ts`).

## Proposed Solution

Add a certified `zhipu` provider leaf that **reuses the shared `chat-completions` protocol** (no `implementation.ts`), declaring only Zhipu-specific identity/endpoint/credential metadata. The factory constructs `ChatCompletionsProvider` with a `completionsPath: '/chat/completions'` override and a fail-closed `ZHIPU_API_KEY` resolution. Regenerate the provider catalogs so the generator wires Zhipu into `PROVIDER_DEFINITIONS` / adapters / factories, and add a leaf integration test modeled on `xai.test.ts`.

## Implementation Plan

*Using UMPIRE framework (adapted):*

**Understand:** NueOS has no Zhipu (GLM) provider. Zhipu speaks the OpenAI Chat Completions wire format, so it should be added as a standard OpenAI-compatible leaf that reuses `protocols/openai-api/`, without touching shared protocol code.

**Match:** Direct precedents already in the repo:
- `providers/xai/`, `providers/groq/`, `providers/moonshot/` — remote, bearer-auth, `chat-completions` leaves that re-export the shared adapter/provider. Cleanest overall template.
- `providers/perplexity/` — the precedent for a **non-`/v1` base**: overrides `completionsPath: '/chat/completions'` and resolves its key fail-closed. This is the pattern Zhipu needs because of its `…/paas/v4` base.
- `provider-definitions.ts` / `provider-adapters.ts` / `provider-factories.ts` — `@generated` catalogs produced by `scripts/generate-provider-aggregates.mjs`.

**Plan:**
1. **Create `providers/zhipu/definition.ts`** → export `providerDefinition` (`as const satisfies ProviderDefinitionLeaf`): `vendorKey: 'zhipu'`, `displayName: 'Zhipu GLM'`, `providerType: 'text'`, `providerClass: 'remote_text'`, `protocol: 'chat-completions'`, `adapterKey: 'chat-completions'`, `defaultEndpoint: 'https://open.bigmodel.cn/api/paas/v4'`, `defaultModelId` (e.g. `glm-4.6` — confirm), `auth` (`envVar: 'ZHIPU_API_KEY'`, `vaultKeyNamespace: 'zhipu'`, bearer header, `required: true`, `purpose: 'api_key'`), `modelListEndpoint: '/models'` + `modelListFormat: 'openai-models'`, `capabilities` (`streaming`, `modelListing`; omit `nativeToolUse` per the groq/`#390` note unless verified), `isLocal: false`.
2. **Create `providers/zhipu/adapter.ts`** → re-export `chatCompletionsAdapter as providerAdapter` (+ `createChatCompletionsAdapter`) from `../../protocols/openai-api/adapter.js`.
3. **Create `providers/zhipu/provider.ts`** → export `providerFactory` (`as const satisfies ProviderFactoryModule`): resolve `options?.apiKey ?? process.env.ZHIPU_API_KEY`, throw `NousError('Zhipu API key required …', 'PROVIDER_AUTH_FAILED', { failoverReasonCode: 'PRV-AUTH-FAILURE' })` if missing, then `new ChatCompletionsProvider(config, { apiKey, completionsPath: '/chat/completions' })`.
4. **Create `providers/zhipu/index.ts`** → re-export `providerAdapter`, `providerDefinition`, `providerFactory`, and `ChatCompletionsProvider`.
5. **Regenerate catalogs (do not hand-edit):** `pnpm --filter @nous/subcortex-providers run generate:providers` — wires `zhipu` into the three generated files.
6. **Add tests:** `__tests__/providers/zhipu.test.ts`, modeled on `xai.test.ts` (see Evaluate).

**Implement:** Branch `feat/zhipu-glm-provider`. Commits `291d6f25`, `ebaf56ca`, `237e84b8`, and the review fix `bb241c1f` — see [Code Changes](#code-changes) for the annotated list.

**Review — self-review checklist (per `CONTRIBUTING.md`):**
- [x] Leaf reuses the shared protocol; no `implementation.ts`, no edits to shared protocol code.
- [x] `definition.ts` is metadata-only — no env reads, no network calls.
- [x] Generated catalogs updated via the script, not by hand; `check:generated` clean.
- [x] Factory fails closed on missing `ZHIPU_API_KEY`; no OpenAI-key fallback reachable.
- [x] `completionsPath` override prevents doubled `/v1`; `modelListEndpoint` relative to the `/paas/v4` base.
- [x] `nativeToolUse` not advertised unless the shared native tool-use loop is verified for Zhipu — *missed on the first pass, caught in review and fixed in `bb241c1f`.*
- [x] `parseResponse(...)` returns a text fallback rather than throwing (inherited from shared adapter — asserted in test).
- [x] Conventional-commit messages; no unrelated changes in the diff.

**Evaluate — verification:**

Run:

```bash
pnpm --filter @nous/subcortex-providers run check:generated
pnpm --filter @nous/subcortex-providers run typecheck
pnpm --filter @nous/subcortex-providers exec vitest run \
  src/__tests__/provider-codegen.test.ts \
  src/__tests__/public-exports.test.ts \
  src/__tests__/providers/zhipu.test.ts
```

`zhipu.test.ts` asserts:
- Registered in `PROVIDER_DEFINITIONS`; `resolveProviderDefinition` / `resolveProviderFactory` resolve.
- `ProviderDefinitionSchema.safeParse` passes; `wellKnownProviderId === deriveBuiltInProviderId('zhipu')`.
- Auth contract (`ZHIPU_API_KEY`, `vaultKeyNamespace: 'zhipu'`, bearer, `required: true`).
- Factory instantiates via explicit key and via `ZHIPU_API_KEY`, and **fails closed** without leaking `OPENAI_API_KEY`.
- Adapter parses a normal GLM response and returns a text fallback on malformed input without throwing.
- Built URL is `https://open.bigmodel.cn/api/paas/v4/chat/completions` (no doubled `/v1`).
---

## Implementation Notes

### Week 1 Progress (Jul 24–25, 2026)

Added **Zhipu GLM** as a certified inference provider in `subcortex-providers`. Zhipu's `paas/v4` API is OpenAI Chat Completions–compatible, so the leaf reuses the shared `chat-completions` protocol adapter rather than introducing a new one — the actual code surface is a thin definition + factory, with the bulk of the effort going into getting the endpoint and auth semantics exactly right.

**Challenges / decisions:**

- **Doubled version segment.** Zhipu's base URL (`https://api.z.ai/api/paas/v4`) already carries the API version, so the OpenAI default of appending `/v1/chat/completions` would produce a broken doubled path. Resolved by overriding `completionsPath` to `/chat/completions` in the factory and documenting *why* the default doesn't apply (same class of bug fixed earlier for the xAI leaf in `3e7a9749` / `a4dc1950`).
- **Credential leak / fail-closed.** `ChatCompletionsProvider` falls back to `OPENAI_API_KEY` when no key is passed, which would silently send an OpenAI credential to Zhipu's servers. The factory now resolves `ZHIPU_API_KEY` explicitly and throws `PROVIDER_AUTH_FAILED` if it's absent, so the provider refuses to construct rather than leaking.
- **No model discovery.** Zhipu exposes no public model-list endpoint, so discovery is omitted and the runtime falls back to `defaultModelId` (`glm-4.6`). Documented inline so it doesn't read as an oversight.
- **Live test gating.** The live smoke test is gated behind `NOUS_ZHIPU_LIVE_BT=1` so it never runs in normal CI and requires no credential to be present.

### Week 2 Progress (Jul 30 – Aug 11, 2026)

Opened [PR #429](https://github.com/orthogonalhq/nous-core/pull/429) against the integration branch `feat/contributor-friendly-inference-provider-surface` (not `main`), per the provider-leaf work. Review came back on Aug 4 with **changes requested**, plus a longer writeup separating maintainer-owned shared-surface problems from the one narrow change asked of this leaf.

**The requested change:** drop `nativeToolUse: true` from the definition. Z.AI supports native function calling upstream, but the capability flag is meant to describe what works end to end through Nous, and the shared Chat Completions path cannot complete that round trip yet (#390). Fixed in `bb241c1f`, along with a reworked comment that separates Z.AI's upstream support from what Nous can currently do.

Before pushing the fix I traced the path to confirm the reasoning rather than just complying: tool definitions never reach the request body, and `tool_calls` are dropped before `parseResponse` ever sees them. So the flag was advertising behavior the runtime could not deliver. Tracing it also surfaced something worth reporting back: the gateway decides whether to send tools from the *adapter* capability (`agent-gateway.ts:321`, `prompt-composer.ts:53`), not the leaf flag, and the shared `chat-completions` adapter still declares `nativeToolUse: true`. That means the leaf change makes the metadata honest but is not a functional fix, and the same gap exists on the `vllm`, `xai`, and `azure-openai` leaves. Raised it as context, explicitly not as scope for this PR.

**Maintainer-owned items this contribution surfaced (recorded, not worked around):**

- **Key validation blocks Settings save.** The shared key-validation helper needs a model-list or simple GET health endpoint. Z.AI has neither, so it reports the key as invalid and "Save & Test" refuses to save. Tracked under #413. Maintainer was explicit: no Zhipu-specific server branch, no invented endpoint. Environment-based registration works with the configured default model until that lands.
- **Streaming path not selected.** The live test proves `ChatCompletionsProvider.stream()` works against Z.AI, but the gateway does not pick that path because the shared adapter advertises `streaming: false`. Also #413.
- **Centralized test churn.** The five global provider-test edits are the merge churn tracked by #414, with #431 as the first cleanup slice. Handled at merge time, no rebase required.

Approved and merged Aug 11 as the initial early-access Zhipu GLM provider leaf.

### Code Changes

- **Files modified:**
  - Added (the leaf):
    - `self/subcortex/providers/src/providers/zhipu/definition.ts` — provider definition (endpoint, auth, `chat-completions` protocol, capabilities)
    - `self/subcortex/providers/src/providers/zhipu/provider.ts` — factory with fail-closed key resolution and `completionsPath` override
    - `self/subcortex/providers/src/providers/zhipu/adapter.ts` — re-exports the shared `chatCompletionsAdapter`
    - `self/subcortex/providers/src/providers/zhipu/index.ts` — leaf barrel
  - Registration:
    - `self/subcortex/providers/src/provider-definitions.ts`
    - `self/subcortex/providers/src/provider-adapters.ts`
    - `self/subcortex/providers/src/provider-factories.ts`
  - Tests:
    - `self/subcortex/providers/src/__tests__/providers/zhipu.test.ts` (178 lines) — definition/adapter/factory/registry coverage
    - `self/subcortex/providers/src/__tests__/providers/zhipu.live.test.ts` (97 lines) — gated live blackbox smoke test
    - Updated expectation counts in `adapter-resolver`, `provider-codegen`, `provider-definition-types`, `provider-definitions`, and `provider-pipeline-integration` tests

- **Key commits:**
  - [`291d6f25`](https://github.com/orthogonalhq/nous-core/commit/291d6f25f5bb37caba85539614f7cf5ef82f86bb) `feat(subcortex-providers): add Zhipu GLM provider leaf` (Jul 24, +70) — the leaf itself: `definition.ts`, `provider.ts`, `adapter.ts`, `index.ts`. Contains both non-obvious decisions, the `completionsPath` override and the fail-closed key resolution.
  - [`ebaf56ca`](https://github.com/orthogonalhq/nous-core/commit/ebaf56ca6fa10b09a6e4ce2470a1f75ddb7f5997) `feat(subcortex-providers): register Zhipu GLM provider leaf and tests` (Jul 25, +204 / -6) — catalog registration in the three generated files, the 178-line unit test, and the five centralized expectation-count updates.
  - [`237e84b8`](https://github.com/orthogonalhq/nous-core/commit/237e84b851f1a6c20b3826875e034c4f46a23c23) `test(subcortex-providers): add gated live smoke test for Zhipu GLM leaf` (Jul 25, +97) — live invoke + stream smoke test behind `NOUS_ZHIPU_LIVE_BT=1`, following the `codex-cli.live.test.ts` pattern.
  - [`bb241c1f`](https://github.com/orthogonalhq/nous-core/commit/bb241c1ffa0e22c36103c46aeaa864d45980d17e) `fix(subcortex-providers): drop unsupported nativeToolUse claim from Zhipu leaf` (Aug 6, +4 / -3) — the single review fix. Small diff, but it is the one that made advertised capability match actual behavior.

  Totals across the PR: 14 files changed, +372 / -6.

---

## Pull Request

**PR Link:** https://github.com/orthogonalhq/nous-core/pull/429

**Closes:** [#320 Adapter: Zhipu GLM Model Provider](https://github.com/orthogonalhq/nous-core/issues/320)

**Base branch:** `feat/contributor-friendly-inference-provider-surface`

**PR Description:** Summarized the leaf structure (metadata-only `ProviderDefinitionLeaf`, built-in id derived from `vendorKey`, shared `openai-api` chat-completions protocol so no `implementation.ts`), then called out each decision that a reviewer would otherwise have to reverse-engineer: the `paas/v4` base already carrying its version segment so the factory drops `/v1`, the fail-closed `ZHIPU_API_KEY` resolution instead of the `OPENAI_API_KEY` fallback, omitted discovery with fallback to `defaultModelId`, and `glm-4.6` as the default model. Verification section covered `pnpm test` / `lint` / `typecheck` / `build` plus screenshots, and a manual run of the gated live test against the real API (2 passed).

**Maintainer Feedback:**

- **Aug 4, 2026 (@atlamors, CHANGES_REQUESTED):** Endpoint, path, bearer auth, fail-closed factory, and generated catalogs all confirmed correct, and the gated live tests were called out as useful. One requested change: remove `nativeToolUse: true` and rework the nearby comment, because the flag describes what works end to end through Nous and the shared Chat Completions path cannot complete the tool-call round trip yet. The parser test was allowed to stay as coverage that the shared parser understands Z.AI's tool-call shape. Everything else the PR exposed (key validation without a health endpoint, streaming capability mismatch, centralized test churn) was pulled out as maintainer-owned work under #390, #413, #414, and #431, with an explicit note that the Moonshot leaf and shared adapter had given a misleading example and that the inconsistency was theirs. Review closed with an invitation to push back if the boundary seemed wrong.
- **Aug 6, 2026 (my response, `bb241c1f`):** Removed the flag, reworked the comment to separate Z.AI's upstream function calling from what Nous can complete, and pointed at #390.
- **Aug 7, 2026 (my follow-up):** Posted the trace confirming why the removal was correct, agreed with the two-layer split (advertised capability on the leaf, behavior on the shared adapter), and reported the adapter-level flag plus the three other affected chat-completions leaves as context for #390 rather than as scope here.
- **Aug 11, 2026 (@atlamors, APPROVED):** Confirmed the requested change was addressed and the shared-path reading was correct. Verified the mechanically merged state against the integration branch: generated-provider checks, typecheck, build, and all 56 focused tests passed, with the two credentialed live tests skipping as designed. Only conflicts were two global provider-roster expectations from providers merged in the meantime, handled at merge under #414 and #431.

**Status:** Merged (Aug 11, 2026) as the initial early-access Zhipu GLM provider leaf.

---

## Learnings & Reflections

### Technical Skills Gained

- **The provider-leaf contract in `subcortex-providers`.** How a definition, factory, adapter, and the three generated catalog files compose into a registered provider, and why a leaf built on an existing protocol should be metadata plus a thin factory with no `implementation.ts`.
- **Reusing a shared protocol adapter correctly.** An OpenAI-compatible API is only compatible up to its URL and auth conventions. The interesting work was finding the two places where Zhipu diverges from the OpenAI defaults the shared code assumes.
- **Fail-closed credential handling.** A convenience fallback in shared code (`OPENAI_API_KEY`) becomes a credential leak the moment a second vendor reuses the path. Refusing to construct is the correct behavior, and `NousError` with `PROVIDER_AUTH_FAILED` plus a failover reason code is how this codebase expresses it.
- **Gated live tests.** Structuring a credentialed test so it skips silently in CI, requires no key to be present, and still gives real end-to-end evidence when run deliberately.
- **Reading a capability flag through its consumer.** Following `nativeToolUse` from the definition into `agent-gateway.ts` and `prompt-composer.ts` to find out which layer the runtime actually reads. This is what turned a one-line review fix into an understanding of the shared surface.

### Challenges Overcome

- **Doubled version segment.** `https://api.z.ai/api/paas/v4` already carries the version, so the shared default of appending `/v1/chat/completions` produced a broken path. Overrode `completionsPath` in the factory and documented why the default does not apply, which is the same class of bug fixed earlier for the xAI leaf in `3e7a9749` / `a4dc1950`.
- **Silent credential fallback.** Found by reading `ChatCompletionsProvider` rather than by a failing test, since the failure mode is a successful-looking request carrying the wrong vendor's key. Resolved by explicit key resolution and a hard throw.
- **No model-list endpoint.** Omitting discovery was straightforward. What was not obvious is that the same missing endpoint breaks the shared key-validation helper and therefore the Settings "Save & Test" flow. The right move was to report it, not to invent an endpoint or add a vendor-specific branch, and it is now recorded under #413.
- **A capability flag copied from a working example that was itself wrong.** `nativeToolUse: true` came from the Moonshot leaf. It looked like the established pattern for a chat-completions provider. The fix was small, but arriving at it required tracing the request path to see that tools never reach the body and `tool_calls` are dropped before the parser runs.

### What I'd Do Differently Next Time

- **Verify capability flags against the runtime, not against a sibling leaf.** Existing providers are reliable examples of *shape* and unreliable examples of *truth*. For anything a leaf asserts about behavior, find the code that consumes the assertion before setting it. Doing that trace before review instead of after would have caught this one myself.
- **Exercise the whole configuration surface, not just construct and invoke.** The leaf worked against the real API well before I knew that a key could not be saved through Settings. Walking the full path a user takes (add key, save and test, select a model, run a turn) would have surfaced the #413 limitations during Week 1 rather than during review.
- **Open the PR closer to when the code is done.** The commits were authored Jul 24 and 25 and the PR went up Jul 30. Opening earlier would have started the review conversation, and the shared-surface discussion that came out of it, several days sooner.
- **Keep the pushback-with-evidence habit.** Tracing the path before responding turned a compliance reply into information the maintainer could use for #390. That was the most valuable thing I contributed beyond the leaf itself, and it cost one focused read of the gateway.

---

## Resources Used

- [Z.AI API docs](https://docs.z.ai/) for the `paas/v4` Chat Completions surface, bearer auth, and confirmation that no public model-list endpoint exists
- Existing leaves in `self/subcortex/providers/src/providers/` as structural references: `moonshot` and `xai` for chat-completions leaf shape, `codex-cli.live.test.ts` for the gated live-test pattern
- The xAI path-fix commits `3e7a9749` and `a4dc1950`, which established the `completionsPath` override precedent for a base URL that already carries its version
- `ChatCompletionsProvider` in `src/protocols/openai-api/provider.js`, read directly to find the `OPENAI_API_KEY` fallback
- `agent-gateway.ts` and `prompt-composer.ts` for how tool-use capability is actually consumed at runtime
- Issues and PRs: [#320](https://github.com/orthogonalhq/nous-core/issues/320) (the task), [#390](https://github.com/orthogonalhq/nous-core/issues/390) (shared native tool-use bridge), [#413](https://github.com/orthogonalhq/nous-core/issues/413) (protocol/adapter runtime boundaries, key validation and streaming), [#414](https://github.com/orthogonalhq/nous-core/issues/414) and [#431](https://github.com/orthogonalhq/nous-core/pull/431) (provider test refactor)
- [Conventional Commits](https://www.conventionalcommits.org/) for the commit format required by the contributor checklist
