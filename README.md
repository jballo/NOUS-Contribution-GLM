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

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

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
