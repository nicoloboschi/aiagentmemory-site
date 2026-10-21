# LLM Parametric Memory: What Model Weights Remember

> What parametric memory is in LLMs: how facts live in model weights, how much a model holds, how to update it (fine-tuning, ROME, MEMIT, Memory Tuning).

Source: https://aiagentmemory.org/articles/llm-parametric-memory/ · Updated 2026-10-12 · aiagentmemory.org


**LLM parametric memory** is the knowledge a language model stores in its weights during training. It's why a model knows Paris is in France without being told. It's free to use at query time, but it's frozen after training, can't be updated per user, and is hard to inspect or correct. That's why agents put user and task memory somewhere else.

This page covers how facts end up in weights, how much a model can hold, how to change it, and the "latent memory" research that sits between weights and text. For how parametric memory fits next to the memory types agents actually manage, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What is parametric memory in an LLM?

**Parametric memory is knowledge encoded in a neural network's trained parameters. In an LLM, it includes facts, language patterns and skills learned from the training data. The model uses it implicitly on every call, with no lookup, but it changes only through training or direct weight edits.**

The term was popularized by the RAG paper ([Lewis et al., 2020](https://arxiv.org/abs/2005.11401)). It described models that "combine pre-trained parametric and non-parametric memory": a seq2seq model as the parametric part and a dense vector index of Wikipedia as the non-parametric part. The paper's motivation still applies. Models store facts in their parameters, but "providing provenance for their decisions and updating their world knowledge remain open research problems."

In the CoALA agent framework ([Sumers et al., 2023](https://arxiv.org/abs/2309.02427)), LLM weights are the agent's **implicit procedural memory**: the stochastic "production system" that does the reasoning. Everything else an agent remembers lives outside.

## How LLMs store knowledge in weights

Several lines of research show that facts aren't spread evenly through a model. They have a location.

- **Models as knowledge bases.** [Language Models as Knowledge Bases?](https://arxiv.org/abs/1909.01066) (Petroni et al., 2019) showed that BERT, with no fine-tuning, could answer fill-in-the-blank factual queries at a level competitive with traditional NLP methods that had some access to oracle knowledge. Some types of facts were learned much more readily than others.
- **Feed-forward layers as key-value stores.** [Geva et al. (2020)](https://arxiv.org/abs/2012.14913) found that feed-forward layers, about two-thirds of a transformer's parameters, act as key-value memories: keys match patterns in the input, values push the output toward likely next tokens.
- **Facts are localized.** [ROME](https://arxiv.org/abs/2202.05262) (Meng et al., 2022) traced factual recall in GPT models to "a distinct set of steps in middle-layer feed-forward modules" while processing the subject's tokens, and showed it could edit a single fact by changing those weights.

The practical takeaway: parametric memory is real and structured, but it's addressed by patterns in the input, not by IDs. You can't list what's in it or delete one user's data from it.

## How much can parametric memory hold?

[Physics of Language Models: Part 3.3](https://arxiv.org/abs/2404.05405) (Allen-Zhu and Li, 2024) measured factual capacity on controlled datasets and found language models "can and only can store 2 bits of knowledge per parameter," even when quantized to int8. By their estimate, a 7B model could store 14B bits, more than English Wikipedia and textbooks combined.

Capacity isn't the only limit. Exposure matters. [Kandpal et al. (2022)](https://arxiv.org/abs/2211.08411) showed that a model's accuracy on a factual question tracks how many training documents mention the relevant entities. Rare, **long-tail** facts are learned poorly, and the authors estimate models would need to grow by many orders of magnitude to answer them well. Retrieval reduced that dependence.

For agents, the long tail is exactly where user memory lives. Your customer's account history appears in zero training documents.

## Updating parametric memory

| Method | What it changes | Scale | Strength | Weakness |
|---|---|---|---|---|
| Full or LoRA fine-tuning | Many weights, via gradient training | Datasets of examples | Teaches style, format and domain skills | Costly; risk of forgetting older knowledge; needs many examples |
| ROME | One fact via a rank-one weight edit | Single facts | Targeted, inspectable | Doesn't scale to many facts at once |
| MEMIT | Many facts via edits across layers | "Thousands of associations" (GPT-J, GPT-NeoX) | Batch editing | Edits can interact; side effects need testing |
| Lamini Memory Tuning | Many small adapter "memory experts" | Large fact sets | Targets near-zero loss on key facts | Vendor method; results mostly self-reported |
| Retrieval (non-parametric) | Nothing in the model | Unlimited | Instant updates, provenance, per-user scope | Retrieval quality limits recall |

CoALA notes that fine-tuning the agent's LLM "is a costly form of learning," so studies use fixed learning schedules. A 2023 survey of model editing ([Yao et al.](https://arxiv.org/abs/2305.13172)) frames the goal as changing behavior in one domain "without negatively impacting performance across other inputs," which is the hard part.

### What is memory tuning in an LLM?

**Memory Tuning** is Lamini's name for training facts directly into a model with a **Mixture of Memory Experts** (MoME). The 2024 Lamini paper, [Banishing LLM Hallucinations Requires Rethinking Generalization](https://arxiv.org/abs/2406.17642) (Li et al.), argues that models hallucinate facts when training loss on those facts stays above a threshold, as it does in normal pretraining. Memory Tuning instead "targets near zero training loss for key facts."

The model keeps a frozen backbone and adds a large index of adapters. Facts are stored in what the paper calls "millions of memory experts that are retrieved dynamically," and a cross-attention step picks the relevant experts at inference. Treat accuracy claims for it as vendor-reported unless you test it on your own data.

## Latent memory: inside the model, outside the weights

Between frozen weights and plain text there's a third option: memory kept as hidden vectors the model reads directly.

- **The KV cache.** The keys and values computed for the current context are a short-lived latent memory. The [context window](/articles/context-window-of-an-llm/) is its limit.
- **MemoryLLM.** [MemoryLLM](https://arxiv.org/abs/2402.04624) (Wang et al., 2024) adds "a fixed-size memory pool within the latent space" of a transformer that the model updates with new text. The authors report no sign of performance degradation after nearly a million memory updates.
- **Memory³.** [Memory³](https://arxiv.org/abs/2407.01178) (Yang et al., 2024) adds "explicit memory," which it describes as the third form of memory after model parameters and context key-values. A 2.4B model trained this way beat larger models and RAG baselines in the authors' tests.
- **Test-time learning.** Architectures like Google's Titans update a neural memory module while the model runs; see [Google Titans and human-like memory](/articles/google-titans-give-ai-human-like-memory/).

These are research systems. None of them gives you per-user isolation, deletion or audit trails out of the box, which is why production agents still store memory as text or structured records.

## Parametric vs external memory for AI agents

| | Parametric memory | External memory |
|---|---|---|
| Where | Model weights | Database, files, graph |
| Updated | Training or weight editing | Any time, per request |
| Per-user scope | No | Yes, by namespace or ID |
| Provenance | None | Can link to source messages |
| Delete one fact | Hard, often impossible | One call |
| Cost per query | None beyond inference | A retrieval step and prompt tokens |
| Best for | General knowledge, language, reasoning skill | User facts, events, changing data, private data |

The two work together. Parametric memory supplies the general knowledge and reasoning; external memory supplies what's specific, recent or private. Some external systems shrink the per-query cost: Hindsight's [mental models](https://github.com/vectorize-io/hindsight) are standing answers kept current in the background, so reading one is a database read with no retrieval step. The trade-offs on the external side are covered in [RAG vs agent memory](/articles/rag-vs-agent-memory/), and the full set of memory categories is in [types of AI agent memory](/articles/ai-agents-memory-types/).


## FAQ

### What is parametric memory in an LLM?

Parametric memory is the knowledge stored in a language model's weights during training: facts, language patterns and skills. It is available on every call with no retrieval, but it is fixed at training time, hard to inspect, and can't be updated per user. The term comes from the 2020 RAG paper, which contrasted it with non-parametric memory in an external index.

### What is memory tuning in LLMs?

Memory Tuning is a method from the company Lamini that trains a large set of adapter 'memory experts' on specific facts until the training loss for those facts is near zero, aiming to stop the model from hallucinating them. At inference, only the relevant experts are selected. It was described in a 2024 Lamini paper and is a vendor method, not a general standard.

### What is latent memory in an LLM?

Latent memory is information kept inside the model's hidden representations rather than in its trained weights or in text. Examples include the KV cache of the current context and research models like MemoryLLM, which adds a fixed-size pool of updatable memory vectors to a transformer.

