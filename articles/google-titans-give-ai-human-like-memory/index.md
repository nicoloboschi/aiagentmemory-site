# Google Titans: How AI Learns to Memorize at Test Time

> Google Titans explained from the paper: a neural long-term memory that learns at test time using surprise, momentum and forgetting, its MAC/MAG/MAL variants and results.

Source: https://aiagentmemory.org/articles/google-titans-give-ai-human-like-memory/ · Updated 2026-10-12 · aiagentmemory.org


**Google Titans** is a research architecture that gives a language model a second kind of memory: a small neural network that keeps learning while the model reads. Attention handles the recent context as short-term memory. The neural memory compresses the distant past into its weights, using a "surprise" signal to decide what's worth storing. The paper reports scaling past 2 million tokens.

It's research, not a product. You can't call a Titans model through an API, and Google hasn't released official code. What it offers is a clear idea about how future models might handle very long inputs without paying full attention cost on every token.

## What is Google Titans?

**Titans is a family of model architectures, introduced in "Titans: Learning to Memorize at Test Time" by Ali Behrouz, Peilin Zhong and Vahab Mirrokni of Google Research, that combines attention as short-term memory with a neural long-term memory module whose parameters are updated during inference.** The paper was posted to arXiv on 31 December 2024 ([arXiv 2501.00663](https://arxiv.org/abs/2501.00663)) and presented as a poster at NeurIPS 2025.

The starting point is a trade-off the abstract states directly. Recurrent models compress history into a "fixed-size memory," so detail gets lost. Attention sees the whole context window exactly, but its cost grows quadratically with length, which caps how long the window can be. Titans tries to get both: exact attention over a recent slice, plus a learned memory for everything older.

The paper frames this with human memory terms. Attention is short-term memory: precise but limited. The neural module is long-term memory: it holds more, less precisely. A third piece, **persistent memory**, holds task knowledge that doesn't depend on the input.

## How the Titans neural long-term memory works

The core of Titans is the **neural long-term memory module (LMM)**. It's a small multi-layer perceptron. At each step, the model takes a gradient step on that MLP's weights, so the memory "learns" the input as it goes. This happens at test time, not only in training.

### Memory as key-value regression

Each input token is projected into a key and a value. The memory is trained online to map keys to values, with a squared-error loss: how far the memory's output for key k is from value v. That is **associative memory**: given a key, recall the value.

Reading the memory is just a forward pass with a query, with no weight update. So the model writes by learning and reads by inference.

### Surprise decides what gets stored

The paper defines surprise in one line: "a simple definition of surprise for a model can be its gradient with respect to the input." If the memory already predicts a token's value well, the gradient is small and little changes. If the token is unexpected, the gradient is large and the memory updates more.

The idea comes from how people remember. In the paper's words, the module is designed "so an event that violates the expectations (being surprising) is more memorable." The authors note a problem with using raw gradients, though. After a very surprising moment, the gradient can shrink fast, and the memory would miss what comes right after.

### Momentum carries surprise forward

To fix that, Titans splits surprise into two parts:

- **Momentary surprise**: the gradient for the current token.
- **Past surprise**: a decaying record of recent surprise, which works like momentum in gradient descent.

The update term at step t is the decayed past surprise minus the scaled current gradient. Both the decay and the step size are data-dependent, so the model learns how long a surprising event keeps influencing the memory.

### Forgetting with a learned gate

Memory capacity is finite, so Titans also forgets. Before each update, an adaptive gate α between 0 and 1 shrinks the old memory. Near 0, the memory keeps the past; near 1, it clears it. The paper describes this as a form of weight decay and says it generalizes the forgetting gates in modern recurrent models such as Mamba, Mamba2 and Gated DeltaNet.

Putting it together, each step does: shrink the old memory by the forget gate, then add the surprise term. The paper also shows how to compute this in parallel over chunks with matrix multiplications and a parallel scan, so training on accelerators is practical.

### Deeper memory holds more

The memory MLP can have one layer (which behaves like a linear memory, similar to linear attention) or more. The paper argues deeper memories are more expressive. In its ablation, deeper memory gave better perplexity at longer sequence lengths but trained more slowly.

## Titans variants: MAC, MAG and MAL

The paper offers three ways to wire the long-term memory into a model. They differ in where the memory sits relative to attention.

| Variant | How memory is used | What attention sees |
|---|---|---|
| **MAC** (Memory as a Context) | Sequence is split into segments. The current segment queries the memory, and the retrieved content is prepended, with persistent memory, before attention. The attention output then updates the memory. | Persistent tokens + retrieved memory + current segment |
| **MAG** (Memory as a Gate) | A sliding-window attention branch and the memory branch run in parallel. A gate mixes their outputs. | A local sliding window |
| **MAL** (Memory as a Layer) | The memory layer processes the input first; sliding-window attention runs on its output. | A local sliding window over memory-processed input |

MAC is the variant the paper leans on for long-context tasks. MAL is the closest to existing hybrid designs that stack recurrent and attention layers. The paper also tests the memory module alone (labeled LMM) as a pure recurrent model.

## Titans results: what the paper reports

All numbers below come from the first (and, as of October 2026, only) arXiv version. The paper itself notes that results for larger models were still being finalized.

### Language modeling

Models were trained on FineWeb-Edu at three sizes. Average accuracy on commonsense reasoning tasks (Table 1):

| Size / tokens | Transformer++ | Best Titans |
|---|---|---|
| 340M / 15B | 42.92 | 46.17 (LMM) |
| 400M / 15B | 45.64 | 48.65 (MAC) |
| 760M / 30B | 48.69 | 52.51 (MAC) |

At 340M, Titans (LMM) reached a WikiText perplexity of 26.18 versus 31.52 for Transformer++.

### Needle in a haystack

On the single-needle tasks from the RULER suite (S-NIAH) at 16K tokens, Titans (MAC) scored 98.4, 97.4 and 95.2 across the three task types. Some recurrent baselines collapsed on the hardest one: TTT and Mamba2 scored 0.0.

### BABILong and the 2M-token claim

**BABILong** asks models to reason over facts scattered through very long documents. The paper tests the MAC variant and reports that it beats GPT-4 and GPT-4o-mini few-shot; after fine-tuning, "Titans outperform all models even extremely large models like GPT4." In the fine-tuned setting it says Llama 3.1 8B with retrieval does worse than Titans with about 70 times fewer parameters. The exact scores are in figures, so they aren't repeated here.

The headline claim, from the abstract: Titans "can effectively scale to larger than 2M context window size with higher accuracy in needle-in-haystack tasks compared to baselines."

### Ablations, time series and DNA

Removing each piece hurt perplexity (Table 5). The full memory scored 27.01; without weight decay (forgetting) it was 29.04, without momentum 28.98, and with a linear instead of deep memory 28.49. The memory module also performed well on time-series forecasting and was "competitive with state-of-the-art architectures" on genomics benchmarks.

## Titans and MIRAS

In December 2025, Google Research published a blog post, [Titans + MIRAS: Helping AI have long-term memory](https://research.google/blog/titans-miras-helping-ai-have-long-term-memory/), pairing Titans with a follow-up paper. The post calls Titans "the specific architecture (the tool)" and MIRAS "the theoretical framework (the blueprint)."

**MIRAS** comes from [It's All Connected: A Journey Through Test-Time Memorization, Attentional Bias, Retention, and Online Optimization](https://arxiv.org/abs/2504.13173) (Behrouz, Razaviyayn, Zhong and Mirrokni, April 2025). It describes transformers, Titans and linear RNNs as associative memories defined by four choices:

1. **Memory architecture**: a vector, a matrix or a deep MLP.
2. **Attentional bias**: the objective the memory optimizes, which decides what it prioritizes.
3. **Retention gate**: the regularizer that balances new learning against keeping old knowledge.
4. **Memory algorithm**: the optimizer used to update the memory.

The paper derives three new models from this. **YAAD** uses a Huber loss so a single outlier, like a typo, doesn't swamp the memory. **MONETA** uses stricter generalized-norm penalties. **MEMORA** constrains the memory to behave like a probability map for stable updates.

## Is Titans memory the same as agent memory?

No, and the "human-like memory" headlines blur this. Titans memory is **in-model, in-run memory**. Its weights reset when a new sequence starts. It helps one model read a very long input; it doesn't remember a user from last week.

That puts Titans next to other ways of [extending the LLM context window](/articles/extending-llm-context-window/), not next to memory databases. It's closer to the context window than to [parametric memory](/articles/llm-parametric-memory/) in the usual sense: the weights it updates are a scratch memory, not the base model's knowledge.

Agents that need memory across sessions still store facts outside the model, in a memory server such as Mem0 or [Hindsight](https://github.com/vectorize-io/hindsight), and load the relevant ones into the window, as the [AI agent memory guide](/articles/ai-agent-memory-explained/) describes. Even with a very long effective context, you'd still want an external store for anything that must survive a restart, be edited, or be audited.

| | Titans neural memory | Context window | External agent memory |
|---|---|---|---|
| Where it lives | Weights of a small MLP inside the model | The current request | A database or service |
| Lifetime | One sequence | One request | Until deleted |
| How it's written | Gradient steps at inference | Your app builds the prompt | Your app or agent writes records |
| Can you inspect it? | Not directly | Yes, it's text | Yes |

For the basics of what the window is and why it's finite, see the [context window of an LLM](/articles/context-window-of-an-llm/).

## Trying Titans in Python

There's no official release. The most used community version is [lucidrains/titans-pytorch](https://github.com/lucidrains/titans-pytorch), which its README calls an "Unofficial implementation of Titans." It's useful for experiments, not for reproducing the paper's exact numbers. From its README:

```python
# pip install titans-pytorch
import torch
from titans_pytorch import NeuralMemory, MemoryAsContextTransformer

# The neural memory module on its own
mem = NeuralMemory(dim=384, chunk_size=64)
seq = torch.randn(2, 1024, 384)
retrieved, mem_state = mem(seq)
assert seq.shape == retrieved.shape

# A small MAC-style transformer
transformer = MemoryAsContextTransformer(
    num_tokens=256,
    dim=256,
    depth=2,
    segment_len=128,             # local attention window
    num_persist_mem_tokens=4,    # persistent memory
    num_longterm_mem_tokens=16,  # retrieved long-term memory
)
token_ids = torch.randint(0, 256, (1, 1023))
loss = transformer(token_ids, return_loss=True)
loss.backward()
```

The README examples call `.cuda()`; the CPU version above is for a quick shape check.

## Key takeaways

- **Titans** adds a neural long-term memory that learns at test time, alongside attention as short-term memory.
- **Surprise** (the gradient), **momentum** and a **forget gate** decide what the memory stores and drops.
- Three variants: **MAC**, **MAG**, **MAL**. MAC is the long-context workhorse.
- The paper reports beating Transformer++ at 340M to 760M parameters, strong BABILong results, and scaling past **2M tokens**. These are research-scale results from one arXiv version.
- **MIRAS** generalizes the idea into four design choices.
- It's in-model memory for long inputs, not persistent memory across sessions.


## FAQ

### What is Google Titans?

Titans is a family of sequence-model architectures from Google Research (Behrouz, Zhong and Mirrokni, arXiv 2501.00663). It pairs attention, used as short-term memory, with a neural long-term memory module, a small MLP whose weights are updated by gradient steps while the model reads its input.

### Does Titans give LLMs human-like memory?

Only in a loose sense. The paper borrows ideas from human memory, such as storing surprising events more strongly and forgetting gradually, but the memory still lives inside one model run. It isn't persistent memory across sessions like a database-backed agent memory system.

### Can I use Titans today?

Not as a product. Google hasn't released official Titans code or weights as of October 2026. The paper's models are research-scale (up to 760M parameters in the first arXiv version), and lucidrains/titans-pytorch is an unofficial PyTorch implementation for experiments.

