# Contribution 2: Adapter: Zhipu GLM Model Provider

**Contribution Number:** 2 
**Student:** Jonathan Ballona Sanchez 
**Issue:** https://github.com/orthogonalhq/nous-core/issues/320 
**Status:** Phase 2 

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

**Implement:** Branch `feat/zhipu-glm-provider`. *(Link commits here as work lands.)*

**Review — self-review checklist (per `CONTRIBUTING.md`):**
- [ ] Leaf reuses the shared protocol; no `implementation.ts`, no edits to shared protocol code.
- [ ] `definition.ts` is metadata-only — no env reads, no network calls.
- [ ] Generated catalogs updated via the script, not by hand; `check:generated` clean.
- [ ] Factory fails closed on missing `ZHIPU_API_KEY`; no OpenAI-key fallback reachable.
- [ ] `completionsPath` override prevents doubled `/v1`; `modelListEndpoint` relative to the `/paas/v4` base.
- [ ] `nativeToolUse` not advertised unless the shared native tool-use loop is verified for Zhipu.
- [ ] `parseResponse(...)` returns a text fallback rather than throwing (inherited from shared adapter — asserted in test).
- [ ] Conventional-commit messages; no unrelated changes in the diff.

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

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
