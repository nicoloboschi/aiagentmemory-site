# AI Memory Pricing 2026: Mem0, Zep, Supermemory, Hindsight

> What agent memory products cost as of 11 November 2026: free tiers, paid plans, usage prices and which ones you can self-host. Read from each vendor's pricing page.

Source: https://aiagentmemory.org/articles/ai-memory-pricing-compared/ · Updated 2026-11-11 · aiagentmemory.org


## Which memory products cost what?

Agent memory pricing comes in three shapes: a free tier with monthly limits, a fixed monthly plan with credits, and pay-per-use billing. As of 11 November 2026, Hindsight is the only one of the four with no plan fee. You pay for what you use on its Cloud, or nothing beyond your own servers if you self-host.

## Price table

Prices and limits as shown on each vendor's pricing page, read on 11 November 2026.

| Product | Free tier | Paid plans | Usage prices | Self-hosting |
|---|---|---|---|---|
| Hindsight (Vectorize) | Self-hosting is free. Cloud gives new accounts free credits to start. | None. Pay as you go. | Retain $10.00 per 1M tokens. Recall $0.75 per 1M tokens. Reflect $0.05 per call. Storage $0.25 per 1M tokens per month, first 30 days free. | Yes, free. MIT license, full server. |
| Mem0 | Free: 10,000 adds and 1,000 retrievals a month for the first 30 days, model calls included. With your own LLM key: 50,000 adds and 10,000 retrievals a month. | Custom, priced by Mem0. Includes graph memory, user profiles, Dream consolidation, SSO and an SLA. | Usage past the free limits, with a monthly cap you set. | Only as on-prem deployment, on the Custom plan. |
| Zep | Free: $0, 10,000 credits a month, 2 projects. | Flex: $125 a month, 50,000 credits, 5 projects. Flex Plus: $375 a month, 200,000 credits, 10 projects. Enterprise: custom. | Overage $25 per 10,000 credits on Flex. $75 per 40,000 credits on Flex Plus. | Flex is cloud only. Enterprise can run inside your own VPC. |
| Supermemory | Free: $0, $5 of credits, 1M memory tokens, 1M queries. | Pro: $19 a month, $20 credits. Max: $100 a month, $130 credits. Scale: $399 a month, $600 credits. Enterprise: custom. | Credits are the meter. The page lists no per-unit price. | Scale has a self-hosted option. Enterprise has air-gapped or dedicated managed instances. |

Units differ. A Mem0 add, a Zep credit and a Supermemory credit are not the same thing, so the table does not give a cost per call. Compare the free tier, then the plan that matches your volume.

Mem0's page shows a date of 10 November 2026. Our fact sheet notes that since 26 October 2026 Mem0 lists only Free and Custom plans. Prices change, so check the vendor page before you budget.

## Where Hindsight is strongest on cost

- **Self-hosting with the full feature set.** The open-source Hindsight server is MIT licensed, with no usage limits and no telemetry, according to its [pricing page](https://vectorize.io/pricing). Its graph, temporal recall, consolidation, reflect and mental models are in that free version. Mem0 keeps graph memory and Dream on its paid Custom plan. Zep's self-hosting is Enterprise only. Supermemory's is on its $399 Scale plan.
- **Pay per token, start free.** New Cloud accounts get free credits to start, and stored memories are free for their first 30 days. There is no monthly seat fee.
- **Model calls are cheap.** Hindsight Models are billed per million input tokens: hs-embed-1 at $0.005, hs-decide-1-flash at $0.02, and hs-rerank-1 at $0.015, or $0.02 when used with hs-decide-1-flash. Output tokens for these models are free, per the same page.

## Honest limits

- **Writes cost the most.** Retain costs $10.00 per 1M tokens, far more than recall at $0.75, because retain and consolidation call an LLM. Heavy write workloads will cost more than read-heavy ones.
- **Each recall returns more context.** On Mem0's own comparison page, Hindsight used about 27K tokens per retrieval on BEAM 10M, and Mem0 under 7K. Those are Mem0's measurements, not Vectorize's. Lower `max_tokens` to cut the cost of each recall.
- **No compliance certifications published.** Mem0's page lists SOC 2 Type 1 and HIPAA. Hindsight publishes none.

## What this table leaves out

- **Letta.** Its pricing link redirects to a page for Letta Code, a coding agent, with plans from $0 to $20 a month. That page does not list the memory server price, so Letta is not in the table. See [Letta: memory, setup and pricing](/articles/letta-ai-guide/).
- **Honcho.** Its pricing page returned a 404 on 11 November 2026, so there is no current price to report. See [Honcho memory](/articles/honcho-llm-memory/).
- **Azure AI Foundry and Vertex AI Agent Engine.** Both bill per use through the cloud account. See [Azure AI Foundry memory](/articles/azure-ai-foundry-long-term-memory/) and [Vertex AI Agent Engine Memory Bank](/articles/vertex-ai-agent-engine-memory-bank/).

For how these products compare on accuracy and not price, see [open-source memory systems compared](/articles/open-source-memory-systems-compared/) and [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/). Hindsight's code and docs are at [github.com/vectorize-io/hindsight](https://github.com/vectorize-io/hindsight).


## FAQ

### How much does AI agent memory cost?

Each product has a free tier. The lowest paid plans are Supermemory Pro at $19 a month and Zep Flex at $125 a month, as of 11 November 2026. Hindsight has no plan fee: its Cloud bills per token, and its open-source server is free to self-host under the MIT license.

### Which agent memory tool can I self-host for free?

Hindsight. Its open-source server is MIT licensed and includes its graph, temporal recall, consolidation, reflect and mental models. Mem0 keeps graph memory on its Custom plan, Zep runs in your own cloud only on Enterprise, and Supermemory's self-hosted option is on its $399 Scale plan.

### Can I compare these memory products by cost per call?

Not directly. Mem0 bills adds and retrievals, Zep and Supermemory bill credits, and Hindsight bills tokens and calls. Pick the free tier or plan that fits your volume, then test your own workload.

