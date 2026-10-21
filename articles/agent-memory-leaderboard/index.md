# Agent Memory Leaderboard: Benchmarks and Scores

> Agent memory leaderboard guide: where to find LongMemEval, LoCoMo and BEAM results, who runs each leaderboard, a score table with sources, and why scores differ.

Source: https://aiagentmemory.org/articles/agent-memory-leaderboard/ · Updated 2026-10-20 · aiagentmemory.org


There is no official **agent memory leaderboard**. The benchmark authors (LongMemEval, LoCoMo, BEAM) publish datasets, not rankings. The leaderboards you can find are run by memory vendors: **agentmemorybenchmark.ai** by Vectorize, the company that makes Hindsight; **OmniMemEval** by MemTensor, the company that makes MemOS; and score pages from Mem0, Zep, Honcho and past.dev.

So every ranking has an owner, and every score depends on the answer model, judge and token budget that owner chose. This page lists where the leaderboards are, what each benchmark measures, the published scores with who reported each one, and how to read them. Everything was checked on the source pages on 12 October 2026. For GitHub stars and PyPI and npm download counts for 23 memory projects, see [AI agent memory statistics](/articles/agent-memory-statistics/).

## What is an agent memory leaderboard?

**An agent memory leaderboard is a table that ranks memory systems (Mem0, Zep, Hindsight, MemOS and others) by their scores on long-conversation benchmarks such as LongMemEval, LoCoMo and BEAM. It either runs every system through one test harness with fixed settings, or collects scores that each vendor or paper published.**

The difference matters. A **same-harness leaderboard** runs every system with the same answer model, judge and prompts, so rows can be compared, but the harness owner chose how to configure each competitor. A **collected leaderboard** copies published numbers, so each row was run under different settings and rows can't be compared directly. Most pages mix both. For how the benchmarks work and how to run one yourself, see the [LLM memory evaluation guide](/articles/llm-memory-evaluation/).

## Where to find agent memory leaderboards

| Leaderboard | Run by | How the numbers are made | What it covers |
|---|---|---|---|
| [agentmemorybenchmark.ai](https://agentmemorybenchmark.ai/) (AMB) | Vectorize, maker of Hindsight | Own runs with its open harness ([repo](https://github.com/vectorize-io/agent-memory-benchmark)), plus an "external results" section copied from papers and vendor blogs, each with a note on its settings | BEAM, LoCoMo, LongMemEval, PersonaMem, LifeBench, PrecisionMemBench, a coding-agent test |
| [OmniMemEval results](https://github.com/MemTensor/OmniMemEval/blob/main/docs/user_memory/results.md) | MemTensor, maker of MemOS | Own runs of 14 backends with one setup: GPT-4.1-mini answers, GPT-4o-mini judge; plus published reference scores | LoCoMo, LongMemEval, BEAM 100K and 10M, PersonaMem v2, HaluMem |
| [Mem0 research](https://mem0.ai/research) and [compare pages](https://mem0.ai/compare/mem0-vs-hindsight) | Mem0 | Mem0's own runs on its managed Platform, plus competitor numbers on the compare pages | LoCoMo, LongMemEval, BEAM, tokens per retrieval |
| [Zep research](https://www.getzep.com/research/) | Zep | Zep's own runs, GPT-5.4 as reader and judge | LoCoMo, LongMemEval |
| [Honcho evals](https://honcho.dev/evals/) | Plastic Labs, maker of Honcho | Honcho's own runs, Claude Haiku 4.5 as the main model | LongMemEval S, LoCoMo, BEAM |
| [past.dev BEAM leaderboard](https://past.dev/benchmarks/beam) | past.dev | Collected vendor results plus its own run (GPT-5.6 Luna answers and judges) | BEAM, all four tiers |
| [Hindsight benchmarks](https://benchmarks.hindsight.vectorize.io/) | Vectorize | Hindsight's scores only; raw run files hosted on AMB | 8 datasets |

Some facts about these leaderboards that change how you read them:

- **AMB is not independent.** Vectorize built it, and its launch post says the team built it "because we wanted to be honest about how Hindsight performs." Its harness supports other providers, including Mem0, Mastra and Cognee, but the results it publishes from its own runs cover Hindsight, a plain hybrid-search baseline and Cognee on parts of LoCoMo and PersonaMem. Competitor scores on AMB come from the external section.
- **OmniMemEval is not independent either.** It's run by the maker of MemOS, and MemOS leads its LoCoMo and LongMemEval tables. Its results page says competitor adapters "are not a claim that every adapter has reached a globally optimal product-specific configuration."
- **MemoryBench** ([repo](https://github.com/supermemoryai/memorybench)), from Supermemory, is a harness for running Supermemory, Mem0 and Zep on LoCoMo, LongMemEval and ConvoMem yourself. It doesn't publish a leaderboard.
- **The LongMemEval repo** has no leaderboard. Its latest news (May 2026) is LongMemEval-V2 for agentic contexts.

## What the main memory benchmarks measure

| Benchmark | What it tests | Size | Scoring |
|---|---|---|---|
| **[LongMemEval](https://arxiv.org/abs/2410.10813)** (ICLR 2025) | Extraction, multi-session reasoning, temporal reasoning, knowledge updates, abstention in chat history | 500 questions; S variant about 115K tokens per question | LLM-judged accuracy |
| **[LoCoMo](https://arxiv.org/abs/2402.17753)** (ACL 2024) | Single-hop, multi-hop, temporal and open-domain questions over long conversations | Released set: 10 conversations (LoCoMo10), about 1,540 scored questions once adversarial ones are dropped | F1 in the paper; most vendors use an LLM judge instead |
| **[BEAM](https://arxiv.org/abs/2510.27246)** (ICLR 2026) | Ten abilities, incl. contradiction resolution, event ordering, preference following, at 100K, 500K, 1M and 10M tokens | 100 conversations, 2,000 questions | LLM-judged "nugget" score per answer |
| **[PersonaMem](https://arxiv.org/abs/2504.14225)** (COLM 2025) | Tracking a user's traits and changing preferences, then picking the most fitting reply | 180+ simulated histories, up to 60 sessions each | Multiple-choice accuracy |
| **[LifeBench](https://arxiv.org/abs/2603.03781)** (2026) | Long-horizon memory from many sources, including habits and procedures, not only stated facts | 10 simulated users (per AMB) | LLM-judged accuracy |

LongMemEval and LoCoMo are the most reported, and top self-reported scores on both now sit between 89% and 95%. That leaves little room to tell systems apart. **BEAM spreads systems out more**, because at 1M and 10M tokens no context window holds the whole history. PersonaMem and LifeBench are harder still: the PersonaMem paper found frontier models near 50% accuracy, and the LifeBench abstract says top memory systems "reach just 55.2% accuracy." More detail on each is in the [memory evaluation guide](/articles/llm-memory-evaluation/), and on the time questions in [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/).

## Agent memory benchmark results: who reported what

These are the scores each vendor publishes for itself, with the source and the models where the source names them.

| System | LongMemEval S | LoCoMo | BEAM 1M | BEAM 10M | Reported by | Models used |
|---|---|---|---|---|---|---|
| Hindsight | **94.6** | 92 | 73.9 | 64.1 | Vectorize ([benchmarks site](https://benchmarks.hindsight.vectorize.io/), AMB); Mem0's compare page lists the same four | Not stated on the site |
| Mem0 Platform | 94.4 | 92.5 | 64.1 | 48.6 | Mem0 ([research](https://mem0.ai/research)) | Not stated |
| Zep | 90.2 | **94.7** | not listed | not listed | Zep ([research](https://www.getzep.com/research/)) | GPT-5.4 reader and judge |
| Honcho | 90.4 | 89.9 | 61.8 | 40.9 | Plastic Labs ([evals](https://honcho.dev/evals/)) | Claude Haiku 4.5; gemini-2.5-flash-lite for extraction |
| MemOS | 89.2 | 88.8 | not listed | 56.8 | MemTensor (README, via OmniMemEval) | GPT-4.1-mini answers, GPT-4o-mini judge |
| Supermemory | 85.2 | not listed | not listed | not listed | Supermemory ([report](https://supermemory.ai/research/longmembench)) | Gemini 3 Pro answers, GPT-4o judge; 84.6 with GPT-5 |
| past.dev | not listed | not listed | **90.65** | **85.03** | past.dev ([BEAM page](https://past.dev/benchmarks/beam)) | GPT-5.6 Luna answers and judges |

Every row is vendor-run. Hindsight's README adds that research collaborators at Virginia Tech and The Washington Post reproduced its results; that is a reproduction by collaborators, not an independent audit. Hindsight's site also reports PersonaMem 32K at 86.6 and LifeBench EN at 71.5. BEAM scores above are nugget scores shown out of 100.

Two things stand out. On LongMemEval and LoCoMo, the top four systems are within about five points of each other, which is less than the effect of changing the answer model. On BEAM 10M, the gap is larger: Hindsight reports 64.1 against 48.6 for Mem0 and 40.9 for Honcho, and Mem0's [comparison page](https://mem0.ai/compare/mem0-vs-hindsight) shows the same Hindsight and Mem0 figures. past.dev reports far higher BEAM scores than anyone else, from a run where one model both answers and judges; no other source has reproduced them.

## Same-harness results: OmniMemEval

OmniMemEval is the largest published run where every system used the same answer model (GPT-4.1-mini) and judge (GPT-4o-mini). Remember it's run by the maker of MemOS.

| Backend | LoCoMo | LongMemEval S | BEAM 100K | BEAM 10M | LoCoMo context tokens |
|---|---|---|---|---|---|
| MemOS | **88.83** | **89.20** | 66.87 | 56.75 | 5,400 |
| Cognee | 83.48 | 51.80 | 59.30 | 56.02 | 32,532 |
| EverOS | 82.75 | 80.40 | 58.64 | 47.73 | 8,559 |
| Hindsight | 81.99 | 72.20 | 70.22 | **59.75** | 24,683 |
| Mem0 (cloud) | 77.68 | 56.00 | 70.41 | 43.33 | 17,395 |
| Letta | 77.12 | 77.67 | 69.22 | 52.30 | 14,188 |
| Supermemory | 73.53 | 66.07 | 65.98 | 52.49 | 15,238 |
| Viking | 69.33 | 61.07 | **70.76** | 58.14 | 5,964 |
| Zep / Graphiti | 63.83 | 79.80 | 68.70 | 56.11 | 1,862 |

Source: [OmniMemEval results snapshot](https://github.com/MemTensor/OmniMemEval/blob/main/docs/user_memory/results.md), checked 12 October 2026. Selected rows; the page lists 14 backends plus PersonaMem v2 and HaluMem.

What this table shows:

- **Scores drop a lot under someone else's harness.** Mem0 reports 94.4 on LongMemEval; here Mem0 cloud scores 56.0. Zep reports 94.7 on LoCoMo; here it scores 63.8. Hindsight reports 92 on LoCoMo; here it scores 82.0. Only MemOS matches its own README, because its README numbers come from this harness.
- **Hindsight is highest on BEAM 10M** (59.75) and within a point of the top at 100K, under a competitor's settings. That matches the pattern in the vendor numbers: its lead shows on the longest histories.
- **Hindsight is middle of the pack on LongMemEval here** (72.20). Most of the drop is one category: 14.3% on single-session-assistant questions (things the assistant said). It was run self-hosted, not on Hindsight Cloud.

## Why leaderboard numbers aren't directly comparable

Two scores for the same benchmark can differ by 30 points without either being false. Before you compare, check:

1. **Answer model.** A stronger model reading the same retrieved memory scores higher. Supermemory's own report shows 85.2 with Gemini 3 Pro and 84.6 with GPT-5 for the same memory.
2. **Judge model and prompt.** LongMemEval's script uses GPT-4o. Zep uses GPT-5.4, OmniMemEval uses GPT-4o-mini, and AMB uses Gemini models.
3. **Metric.** LoCoMo's paper uses F1; vendors use LLM-judged accuracy. Supermemory's 95% Recall@15 on LongMemEval measures retrieval, not answers.
4. **Dataset variant.** LoCoMo10 with or without adversarial questions; LongMemEval S, M or oracle, original or the September 2025 cleaned release; BEAM tier.
5. **Retrieval mode.** One search call per question, or an agent that searches several times. AMB publishes Hindsight results in more than one mode, and the scores differ.
6. **Token budget.** A system that hands the answer model 40K tokens of context has an easier job than one that hands it 2K.

### Tokens per retrieval: the cost axis

Accuracy without token counts hides the cost. Every retrieved token is paid again on every LLM call.

| System | Tokens per retrieval | Source |
|---|---|---|
| Mem0 Platform | about 6.7K-7K (6,956 mean on LoCoMo) | Mem0 research and compare page |
| Zep | 5,760 median on LoCoMo, 4,408 on LongMemEval | Zep research page |
| Hindsight | 23.9K-43.6K depending on benchmark, about 27K on BEAM 10M | Mem0's compare page |
| Hindsight (OmniMemEval run) | 24,683 on LoCoMo, 23,815 on BEAM 10M | OmniMemEval, includes the answer prompt |

Hindsight returns about 4x more context per retrieval than Mem0 by default. The budget is a setting (`max_tokens` on recall), so you can trade some accuracy for fewer tokens, but the published scores were run at the larger budget. Compare accuracy and tokens together, not one alone.

### What no leaderboard here measures

Every score on this page is a judged **answer**. None of them reads the memory store to see whether the
right facts are in it. [Beyond Downstream QA](https://arxiv.org/abs/2610.22209) (Toma et al., Amazon Web
Services and the Technical University of Cluj-Napoca, October 2026) measured both on one system and found
they **diverge by an average of 16 percentage points**, and that the system labeled its own memory
operations (create, update, reinforce, supersede) correctly only **20%** of the time, which no QA score
revealed. Its FEB-30 benchmark scores the store directly, with claim coverage, update detection recall and
a false update rate.

The paper doesn't name the system it tested, and no leaderboard publishes these metrics yet, so there's
nothing to rank. The practical reading: a one-point gap on LongMemEval tells you little about whether a
system handles a fact that changed. Test that yourself. The same applies to deletion, which the
[Bemula benchmark](https://arxiv.org/abs/2610.18052) found no system passes more than 1.5% of the time.
Both are covered in [LLM memory evaluation](/articles/llm-memory-evaluation/).

## Where Hindsight fits on these leaderboards

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed agent memory server from Vectorize. On the published data it's a strong choice when:

- **Histories are long.** It reports 73.9 on BEAM 1M and 64.1 on BEAM 10M, 9.8 and 15.5 points over Mem0 by Mem0's own page, and it has the top BEAM 10M score in MemTensor's OmniMemEval run, where a competitor set up every system.
- **You want every feature open source.** Graph and temporal recall, consolidation into observations, mental models and `reflect` are all in the MIT release. Mem0 keeps graph memory and temporal reasoning on its paid Pro plan.
- **The agent should reason over memory.** `reflect` answers questions that need thinking across many memories, not only lookup.

Its limits on the same data: it returns more tokens per retrieval than Mem0 or Zep, writes cost more because retain and consolidation both call an LLM, and its README says it "may be overkill" for simple workflows. It also has no published compliance certifications, where Mem0 lists SOC 2 Type 1 and HIPAA. On LongMemEval and LoCoMo its self-reported lead over Mem0 and Zep is within a point or two, or behind. More in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/) and the [Hindsight explainer](/articles/ai-memory-hindsight/); for every system's license and storage model, see the [LLM memory comparison](/articles/llm-memory-comparison/).

## How to use a memory leaderboard to choose

1. **Pick the benchmark that looks like your data.** Chat assistants: LongMemEval. Very long histories: BEAM 1M or 10M. Personalization: PersonaMem.
2. **Find out who ran each score.** Vendor-run, competitor-run and paper-run numbers carry different weight.
3. **Match the settings.** Compare only rows with the same answer model, judge and dataset variant, or use a same-harness table.
4. **Read tokens per retrieval next to accuracy.** A small accuracy gain at 5x the context may cost more than it's worth.
5. **Shortlist two or three systems**, then run them on your own data with one fixed answer model and judge. AMB, OmniMemEval and MemoryBench are all open harnesses you can fork for this.
6. **Dump the store when you do.** Feed each system a fact that changes, then read what it kept. That check
   is the one the leaderboards skip, and it is where systems differ most.


## FAQ

### Is there an official leaderboard for agent memory?

No. The LongMemEval, LoCoMo and BEAM authors don't run official leaderboards. The leaderboards that exist are run by memory vendors: agentmemorybenchmark.ai by Vectorize (the company behind Hindsight), OmniMemEval by MemTensor (MemOS), and pages by Mem0, Zep, Honcho and past.dev. Each one runs or collects scores under its own settings.

### Which memory system tops the agent memory leaderboards?

It depends on who ran the test. On self-reported scores, Hindsight, Mem0, Zep and others all sit near 90-95% on LongMemEval and LoCoMo. On the longer BEAM test the spread is wider: Hindsight reports 73.9 at 1M tokens and 64.1 at 10M, against 64.1 and 48.6 for Mem0, and Mem0's own comparison page lists the same figures.

### Why do memory benchmark scores differ so much between sources?

Each run picks its own answer model, judge model, prompts, dataset version and retrieval budget. A stronger answer model or a larger context budget raises the score. Mem0 reports 94.4 on LongMemEval for its platform, while MemTensor's OmniMemEval measured Mem0 cloud at 56.0 under fixed GPT-4.1-mini settings.

