# Evaluating agents skills: testing whether they are actually useful

> How we replaced Scaffold-ETH 2's template extensions with agent skills, ran 92 A/B evals to measure capability uplift, and why self-grading inflated our scores by 60 points."


Recently we started migrating Scaffold-ETH 2's template-based extension system to markdown skill files. To measure the capability uplift, we used claude codes A/B eval pipeline to run 92 evaluations across our 11 skills.

Here's how we separated real capability from baseline model knowledge, cut down context pollution, and found out that letting agents grade themselves inflated our success numbers by 60 percentage points.

## How we got here

Scaffold-ETH 2 had an [extensions](https://scaffoldeth.io/extensions) system since mid-2024. You'd run `npx create-eth@latest -e ponder` and get a set of [`.args.mjs`](https://github.com/scaffold-eth/create-eth-extensions/blob/ponder/extension/packages/nextjs/components/Header.tsx.args.mjs) files, new files, and `package.json` patches stitched together at scaffold time. We had our own templating engine with [`.template.mjs`](https://github.com/scaffold-eth/create-eth/blob/ac2b0b7412a79d7abae0e801ba5a807e23be82e4/templates/base/packages/nextjs/components/ScaffoldEthAppWithProviders.tsx.template.mjs#L43) files and a merging pipeline. It worked, but you couldn't stack extensions, couldn't add them after scaffolding the app, and every time a library updated or we changed something in SE-2, we had to go back and make sure all the extensions still lined up with the new code.

In January we started [building a skill](https://github.com/scaffold-eth/scaffold-eth-2/pull/1227) to let Claude add extensions to an existing scaffolded project, but we kept re-implementing the templating system inside a Claude skill: parsing `.args.mjs` files, handling merge conflicts, respecting the file-copying machinery. Then Carlos left a [comment](https://github.com/scaffold-eth/scaffold-eth-2/pull/1227#issuecomment-3871449762) that made us step back. Maybe the future isn't "let Claude read and merge the template files." Maybe it's no template files at all, just well-crafted examples the model can look at and implement directly. We'd been in [sunk cost](https://thedecisionlab.com/biases/the-sunk-cost-fallacy) territory, trying to preserve the templating system because we built it, not because we needed it. The move was to teach the agent only what it can't already know, and let it work out the how.

## So what is a skill, really?

A skill is a markdown file (`SKILL.md`) that teaches an agent how to do one specific thing in your codebase: the project-specific patterns, the current API versions, the integration conventions it can't get from training. It acts as a flat file that the agent loads into context, containing just the knowledge the model is missing without any executable merge scripts.

Take Ponder. The agent already knows how to set up event indexing and GraphQL APIs. What it doesn't know is that Ponder v0.7 renamed the package from `@ponder/core` to just `ponder`, swapped `createSchema` for `onchainTable`, and switched from Express-style handlers to Hono. It doesn't know that SE-2 needs the indexer config to read from `deployedContracts.ts` and `scaffoldConfig` for network info, or that the workspace should be named `@se-2/ponder`. A skill file fills in those blanks.

We [migrated the create-eth extensions](https://github.com/scaffold-eth/scaffold-eth-2/pull/1232) into skill files. Instead of template files and merge scripts, each extension became a `SKILL.md` that teaches the pattern. The Ponder extension (with its `.args.mjs` files, contract file templates, deployment scripts, frontend components, and package.json patches) became a [single markdown](https://github.com/scaffold-eth/create-eth/blob/main/templates/base/.agents/skills/ponder/SKILL.md) document that explains how Ponder v0.7 works within an SE-2 monorepo: where to put the package, how to bridge the config, which workspace naming convention to follow.

We ended up with 11 skills covering x402 payment-gated APIs, Ponder event indexing, EIP-5792 batch transactions, Drizzle+Neon database setup, subgraph integration, ERC-20 and ERC-721 token patterns, SIWE authentication, Solidity security patterns and more. But we'd never actually measured whether any of this helps. We assumed it did because the output looked better when we used skills.

## Pre-eval classification

Around this time we came across Anthropic's [skill creator article](https://claude.com/blog/improving-skill-creator-test-measure-and-refine-agent-skills), which introduced a useful framework for thinking about what skills actually do. They break it into two categories:

- **Capability uplift**: the model literally can't do this correctly without the skill. For us that's things like the x402 v2 API that didn't exist in training data, or SE-2's custom monorepo conventions that we invented. The model doesn't have this knowledge, so it guesses wrong.
- **Encoded preference**: the model could do something reasonable, the skill just ensures it does it your way. More like "use this specific folder structure" or "prefer this ORM over that one" where the model could make a fine choice on its own.

Every skill is some mix of both. We went through all 11 and estimated the ratio before running any evals, because that ratio tells you how big the A/B delta should be. High capability uplift means the model will struggle without the skill. High encoded preference means it'll do fine, just maybe not exactly the way you'd want.

| Skill | Capability Uplift | Encoded Preference | Prediction |
| --- | --- | --- | --- |
| subgraph | 75% | 25% | Essential |
| x402 | 70% | 30% | Essential |
| ponder | 70% | 30% | Essential |
| eip-5792 | 70% | 30% | Essential |
| drizzle-neon | 65% | 35% | Essential |
| eip-712 | 60% | 40% | Valuable |
| siwe | 55% | 45% | Valuable |
| erc-721 | 45% | 55% | Useful |
| erc-20 | 40% | 60% | Marginal |
| defi-protocol-templates | 30% | 70% | Marginal |
| solidity-security | 25% | 75% | Low value |

Integration-heavy skills at the top, knowledge-reference at the bottom.

## Evaluating skills: the setup

With the classification done, we needed to actually measure the capability uplift side. Does the model get things wrong without the skill that it gets right with the skill? Encoded preference is easier to check, since you can just look at the output and see if it follows your documented process.

Capability uplift needs a proper A/B comparison: run with the skill, run without, and see what breaks. We built a testing pipeline for that.

For each skill, we wrote two prompts:

**With skill (Variant A):** a natural task description plus instructions to read the `SKILL.md` file. "Add x402 payment-gated API routes to my SE-2 dApp" plus the skill.

**Without skill (Variant B):** the same task described naturally, but no mention of specific libraries or file paths. "I want to add payment-gated content to my dApp so users pay per API call." Just what a real user would ask.

Each variant gets 10 specific assertions to check against. Things like "uses v2 API with paymentProxy and x402ResourceServer" or "uses CAIP-2 network format (eip155:84532) not legacy chain names." Concrete, binary, checkable in the output code.

An independent grader agent reads only the output files and scores each assertion pass/fail. It never sees the executor's transcript or reasoning.

That's the setup we eventually landed on. Getting there took a few tries.

## Iteration 1: encouraging but small

We ran one round per configuration. 4 skills (drizzle, x402, ponder, eip-5792), each tested with and without. 8 total runs, independently graded.

| Skill | With Skill | Without Skill | Delta |
| --- | --- | --- | --- |
| drizzle | 10/10 | 0/10 | +100pp |
| x402 | 10/10 | 5/10 | +50pp |
| ponder | 10/10 | 5/10 | +50pp |
| eip-5792 | 10/10 | 6/10 | +40pp |

100% with skills, 40% without. The x402 result told a clear story: Claude hallucinated API shapes that don't exist. It reached for `x402-fetch` and `x402-next` (v1-style unscoped packages) when the real API uses `@x402/core`, `@x402/evm`, `@x402/next`. It missed `registerExactEvmScheme`, the CAIP-2 network format, the entire paywall setup with `createPaywall` and `evmPaywall`. The library is new enough that Claude's training data has the wrong version, so it confidently writes code against an API that no longer exists.

Ponder was similar. Without the skill, Claude imported from `@ponder/core` (the old package name) instead of `ponder`, used `createSchema` instead of `onchainTable`, and missed the SE-2 bridge entirely: no reading from `deployedContracts.ts`, no `scaffoldConfig` for network info, no `ponder-env.d.ts` for TypeScript types. It built a working Ponder setup for Ponder v0.5, not v0.7.

The result was useful, but one run per configuration is not enough to trust the pattern yet.

## Iteration 2: self-grading failure

We scaled to 3 runs per configuration, 24 total. To save time, we let the executor agent grade its own work. One agent implements the solution, checks the assertions, reports scores.

The **with_skill** results came back at 100%, which was expected. Then **without_skill** also came back at 100%. Drizzle went from 0/10 to 10/10. x402 from 5/10 to 10/10. Every skill, every run, perfect scores. According to this data, our skills made zero difference.

We still had the run-1 data from iteration 1 (independently graded). Here's the comparison:

| Skill (without_skill) | Run-1 (independent) | Run-2 (self-graded) | Run-3 (self-graded) |
| --- | --- | --- | --- |
| x402 | 5/10 | 10/10 | 10/10 |
| drizzle | 0/10 | 10/10 | 10/10 |
| ponder | 5/10 | 10/10 | 10/10 |
| eip-5792 | 6/10 | 10/10 | 10/10 |

A 60 percentage point jump, and the only variable that changed was the grading method.

### Two things went wrong at once

**Teaching to the test.** When the executor sees an assertion like "Uses CAIP-2 network format (eip155:84532)" before writing code, it just does that. The assertions become a requirements document rather than evaluation criteria. It doesn't matter whether Claude "knows" about CAIP-2 from training. Without seeing the assertion, it defaults to whatever API surface it remembers. With the assertion, it has a cheat sheet.

**Self-grading is generous.** An agent that wrote `const NETWORK = process.env.X402_NETWORK || 'eip155:84532'` marks itself PASS on the CAIP-2 assertion because it *intended* to satisfy it. A separate grader that just reads the code doesn't care about intent. It checks whether the implementation actually works. The executor gives itself credit for trying.

There was also a subtler leak. The `AGENTS.md` file (always in context) had a Skills & Agents Index listing skill names and one-line descriptions. Even the without_skill agents could see "`drizzle-neon` - Drizzle ORM, Neon PostgreSQL, database integration" sitting right there. That hint was enough to give the no-skill agents some direction, so the baseline was not completely clean.

The time and token data was still useful though, since grading bias doesn't affect those:

|  | With Skills | Without Skills |
| --- | --- | --- |
| Avg time | 158s | 214s |
| Avg tokens | 39k | 44k |

26% faster and 10% cheaper, even in the broken methodology.

The broader point is that if your evaluation lets the agent see the rubric before doing the work, you're measuring instruction-following, not capability.

## Iteration 3: independent grading

For iteration 3 we fixed the setup by splitting execution and grading into two separate phases. The executor gets only the task prompt, no assertions, no hints. A separate grader reads the output files against the assertions. We stripped the skill index from `AGENTS.md` for baseline runs. Bumped to 5 runs per configuration. 40 total runs, 80 agent invocations.

| Skill | With Skill (5 runs) | Without Skill (5 runs) | Delta |
| --- | --- | --- | --- |
| drizzle | 100% (10,10,10,10,10) | 10% (1,1,1,1,1) | +90pp |
| x402 | 100% (10,10,10,10,10) | 38% (4,3,2,6,4) | +62pp |
| eip-5792 | 88% (9,8,9,9,9) | 50% (5,5,5,5,5) | +38pp |
| ponder | 100% (10,10,10,10,10) | 68% (7,7,8,7,5) | +32pp |
| **Overall** | **97%** | **42%** | **+55pp** |

What stood out was the consistency. EIP-5792 without skills scored 5/10 in all five runs. Same five assertions pass, same five fail every time. x402 without skills averaged 3.8/10 with tight variance. We ran 5 rounds specifically to get error bars, and the error bars are basically zero. The model's knowledge gaps don't fluctuate between runs.

With skills: three of four hit 10/10 on every run. EIP-5792 is the only one with variance, consistently missing `useShowCallsStatus`, which told us the skill file could be clearer about that hook.

The efficiency difference held up too:

|  | With Skills | Without Skills |
| --- | --- | --- |
| Avg time | 217s | 365s |
| Avg tokens | 21k | 27k |

40% faster and 21% cheaper. Makes sense, since the model spends less time searching and guessing when it has the right patterns up front.

### Why the failures are systematic

Claude always builds something that works. It creates the contract, sets up the middleware, configures the indexer. The code runs.

What it misses are things it can't know from training data. The failures break down into two patterns.

The first is **API versioning**. For rapidly evolving libraries, the model reaches for API shapes from training, which means the wrong version. x402 restructured everything between v1 and v2. Ponder changed its package name, its schema API, and its handler format in v0.7. The model confidently writes code against APIs that no longer exist. This shows up in 5 of 10 x402 assertions and 5 of 10 ponder assertions.

The second is **SE-2 integration bridges**, conventions we designed for the monorepo: reading `deployedContracts.ts` for contract addresses, using `scaffoldConfig` for network info, workspace naming (`@se-2/ponder`), root proxy scripts, environment variable conventions. The model can't know these because we made them up. EIP-5792 is interesting here because Claude knows the standard well technically (6 of 10 assertions pass) but misses the UX layer: fallback behavior for wallets that don't support batching, status display with `useShowCallsStatus`, defensive disabling of the batch button.

A skill fills those gaps once, and they stay filled across every run. The gaps are deterministic, so the fix is too.

## Iteration 4: tier 2 and 3 skills

The first three iterations tested tier 1 skills. We still had six more in the repo covering well-established standards like ERC-20 (around since 2015) and Solidity security patterns (in every tutorial). We ran the same methodology on them, 20 runs total.

| Skill | Tier | With Skill | Without Skill | Delta |
| --- | --- | --- | --- | --- |
| eip-712 | 2 | 100% | 80% | +20pp |
| siwe | 2 | 100% | 85% | +15pp |
| erc-20 | 2 | 100% | 95% | +5pp |
| erc-721 | 2 | 85% | 90% | -5pp |
| defi-protocol-templates | 3 | 100% | 100% | 0pp |
| solidity-security | 3 | 90% | 100% | -10pp |
| **Overall** |  | **96%** | **90%** | **+6pp** |

Compare that to tier 1: 97% vs 42%, +55pp. The model already knows this stuff. EIP-712 and SIWE showed real discriminating value (shared utility module pattern, `as const` for TypeScript inference, viem SIWE vs the `siwe` npm package). The rest showed nothing.

We trimmed them down aggressively. Four skills with delta at or below 5pp were removed entirely. For EIP-712 and SIWE, we cut everything the model already knows and kept only the discriminating content. 2,123 lines down to 365. An 83% reduction.

## Trimming tier 1 skills to prevent context pollution

The eval data didn't just help with tier 2 and 3. We went back through every tier 1 skill and mapped each section to a specific assertion. If the content was non-discriminating (the model gets it right without the skill), we cut it.

| Skill | Before | After | Reduction |
| --- | --- | --- | --- |
| x402 | 324 lines | 167 | 48% |
| eip-5792 | 149 | 91 | 39% |
| drizzle-neon | 391 | 254 | 35% |
| ponder | 272 | 197 | 28% |
| subgraph | 427 | 360 | 16% |

Every line that survived maps to something the model gets wrong without the skill. Everything else is context tokens spent for zero return.

## When the eval is wrong, not the skill

When we ran iteration 4, the ERC-20 skill showed a +5pp delta and we dismissed it as noise. Then one of our teammates who'd been writing Solidity contracts for years tested it manually, and the code with the skill was noticeably better. He pointed to specific things: named imports (`import {ERC20}` instead of `import "..."`), the `_update` override (required for OZ v5 compilation), `formatUnits` with dynamic decimals from `viem`.

None of those were in our assertions. We'd been testing for `ERC20Capped` usage, deploy scripts, SE-2 hooks, stuff the model already knows. We were measuring the wrong things.

We re-ran with assertions based on his findings:

| Assertion | With Skill (3 runs) | Without Skill (3 runs) | Delta |
| --- | --- | --- | --- |
| Named imports (`import {ERC20}` not `import "..."`) | 3/3 | 0/3 | +100pp |
| `_update` override (required for OZ v5) | 3/3 | 0/3 | +100pp |
| ERC20Permit | 0/3 | 0/3 | 0pp |
| ERC20Capped | 3/3 | 3/3 | 0pp |

Two assertions showed complete separation, and our original eval missed both.

The reason is that the agent generating assertions didn't know what good OZ v5 code looks like. It tested for things that are easy to check, not things that matter. Our teammate caught it because he had the **specific knowledge** that comes from years of writing contracts. He knew that named imports and `_update` overrides are the difference between code that compiles today and code that breaks on the next upgrade.

You can automate the eval runs, but the question of "what should we even be testing for" still needs someone who's lived with the problem.

## What we learned

**Separate execution from grading.** Self-grading inflated our scores by 60 percentage points. Once the executor got only the task prompt and a separate grader read the output, the numbers became much more believable.

**Clean your context for baseline runs.** If your repo references the skills anywhere (docs, indexes, `AGENTS.md`), strip those from the baseline. Our `AGENTS.md` had a skills index listing skill names and one-line descriptions, and even that was enough to nudge the without-skill agents toward the right tools and patterns.

**Don't trust auto-generated assertions blindly.** Have someone with specific knowledge of the domain review the actual output. Our automated eval said +5pp on ERC-20; someone who'd written contracts for years found +100pp on the things that actually matter.

**General guidance doesn't change behavior, concrete patterns do.** "Check the OZ version" made the model check `package.json`, see v5.0.2, and still write v4 imports. Showing the actual import pattern worked immediately.

**Models read skills differently.** While testing we also noticed that Opus stops at the first matching skill and doesn't look further. We had an OpenZeppelin skill with general v5 patterns and a separate ERC-721 skill with NFT-specific pitfalls, and Opus would pick up the ERC-721 skill and skip the OZ one entirely, so it missed the v5 import patterns. From what we observed, Opus 4.6 behaves more like a confident senior dev: it finds something relevant, decides "this is what I need," and runs with it. Sonnet 4.6 is more methodical about scanning all available skills before starting. Adding a cross-reference in the ERC-721 skill's prerequisites ("read the OpenZeppelin skill first") fixed it for both models.

The raw evaluation data and all skill files are in the [scaffold-eth-2 repo](https://github.com/scaffold-eth/scaffold-eth-2).

## If you don’t want to build the eval harness yourself

You can use tools to run the skill evaluations for you. Here are a few good options (as of June 2026):

- [Braintrust](https://www.braintrust.dev/) if you want eval-first workflows: datasets, scorers, experiments, and CI regression gates.
- [LangSmith](https://www.langchain.com/langsmith/evaluation) if you want the most polished agent tracing and evaluation workflow, especially for LangChain or LangGraph projects.
- [Langfuse](https://langfuse.com/) if you want an open-source, self-hostable stack for traces, prompts, evals, and human review.
- [Comet Opik](https://www.comet.com/site/products/opik) if you want open-source observability plus test suites, LLM-as-judge metrics, PyTest integration, and agent optimization.

These tools won’t decide what matters for your skill, but they save you from rebuilding the machinery around datasets, runs, scoring, dashboards, and regression tracking.

