# Holographic Memory for AI Agents: HRR Explained

> What holographic memory means for AI agents: holographic reduced representations (HRR), how bind and unbind work, and the Hermes Agent holographic provider.

Source: https://aiagentmemory.org/articles/holographic-memory-ai-agent/ · Updated 2026-10-15 · aiagentmemory.org


**Holographic memory for AI agents** means storing facts as **holographic reduced representations (HRR)**: vectors bound together with circular convolution and added into one fixed-size vector, then pulled back out by unbinding with a cue. The idea dates from Tony Plate's 1995 paper. The main working example in agents is the Holographic memory provider in Hermes Agent.

The name sounds futuristic, and many pages online treat it as a vague vision of "multi-dimensional recall". It isn't. HRR is a specific, small piece of math with known strengths and a known capacity limit. This page explains both, and what it looks like inside a real agent.

## What is holographic memory for an AI agent?

**Holographic memory is a way of storing many associations in one fixed-width vector. Each item gets a random high-dimensional vector; pairs are joined with a binding operation (circular convolution), and the results are summed. Querying with one half of a pair returns a noisy copy of the other half, which is then matched against known items.**

It's called "holographic" because, as in an optical hologram, each item is spread across the whole vector instead of sitting in one slot. Damage or noise blurs every memory a little rather than wiping one out.

HRR belongs to a family called **vector symbolic architectures** (VSA), also known as hyperdimensional computing. They all share three operations:

| Operation | What it does | HRR version |
|---|---|---|
| **Bind** | Joins two vectors into a new one that looks unlike either | Circular convolution |
| **Unbind** | Recovers one part given the other | Circular correlation |
| **Bundle** | Stores several vectors in one | Addition (superposition) |

This is a different idea from the semantic memory most agents use. A vector database stores one embedding per fact and finds neighbors by similarity. An HRR store can put many facts in one vector and answer "what is bound to X?" with algebra. For the usual approach, see [semantic memory in AI agents](/articles/semantic-memory-ai-agents/).

## Holographic reduced representations: the research behind it

The core paper is Tony Plate's ["Holographic Reduced Representations"](https://redwood.berkeley.edu/wp-content/uploads/2020/08/Plate-HRR-IEEE-TransNN.pdf), *IEEE Transactions on Neural Networks* 6(3), 1995. It proposed circular convolution as the binding step so that structured information fits in a vector of fixed size. It also showed that unbinding returns a noisy result that needs a **clean-up memory**: a lookup that snaps the noisy vector to the closest known item.

HRR has shown up in neural network research several times since:

- **Associative LSTM** (Danihelka et al., ICML 2016, [arXiv:1602.03032](https://arxiv.org/abs/1602.03032)) added an HRR-like associative memory to LSTMs without adding parameters. It noted that HRR retrieval gets noisier as more is stored, and reduced the noise by keeping redundant copies.
- **HolE** (Nickel, Rosasco and Poggio, AAAI 2016, [arXiv:1510.04935](https://arxiv.org/abs/1510.04935)) used circular correlation to build compositional embeddings of whole knowledge graphs for link prediction.
- **Hrrformer** (Alam et al., ICML 2023, [arXiv:2305.19534](https://arxiv.org/abs/2305.19534)) rebuilt self-attention with HRR to handle sequences over 100,000 tokens in malware detection. The authors report near state-of-the-art results on the Long Range Arena benchmark and training up to 280 times faster.

None of these is an LLM agent memory system. There's no established line of peer-reviewed work on HRR memory for LLM agents as of October 2026. What exists is engineering: open-source plugins that use HRR as one retrieval signal, with Hermes Agent's being the most visible.

## How bind, unbind and bundle work (Python example)

The easiest way to see HRR is to run it. This sketch uses the **phase form** of HRR, where each vector is a list of angles. Binding becomes adding angles, which is the same as circular convolution in the frequency domain. It's the form Hermes Agent's plugin uses.

```python
import numpy as np

DIM = 1024
rng = np.random.default_rng(0)

def atom():
    """A random phase vector: each item gets one."""
    return rng.uniform(0, 2 * np.pi, DIM)

def bind(a, b):    # circular convolution, in phase form
    return (a + b) % (2 * np.pi)

def unbind(m, a):  # circular correlation, the inverse
    return (m - a) % (2 * np.pi)

def bundle(*vs):   # superposition: many items in one vector
    return np.angle(np.sum([np.exp(1j * v) for v in vs], axis=0)) % (2 * np.pi)

def sim(a, b):     # about 0 for unrelated vectors, 1 for identical
    return float(np.mean(np.cos(a - b)))

names = ["alice", "bob", "carol", "paris", "tokyo", "lima"]
v = {n: atom() for n in names}
lives_in = atom()

# One vector holds three facts: alice->paris, bob->tokyo, carol->lima
memory = bundle(
    bind(bind(v["alice"], lives_in), v["paris"]),
    bind(bind(v["bob"], lives_in), v["tokyo"]),
    bind(bind(v["carol"], lives_in), v["lima"]),
)

# Ask: where does bob live?
noisy = unbind(memory, bind(v["bob"], lives_in))
for city in ["paris", "tokyo", "lima"]:
    print(city, round(sim(noisy, v[city]), 2))
# paris -0.01
# tokyo 0.51
# lima 0.02
```

The answer comes back clearly: Tokyo scores 0.51, the other cities sit near zero. Note the final step. The unbound vector isn't Tokyo; it's a noisy copy that you compare against every known city. That comparison is Plate's clean-up memory.

### The capacity limit

Bundling isn't free. Each extra fact adds noise to every other one. Running the same test with random key-value pairs in one 1,024-dimension vector gives these average match scores (same code, our run):

| Facts in one vector | Match score for the right answer |
|---|---|
| 3 | 0.52 |
| 10 | 0.28 |
| 50 | 0.13 |
| 250 | 0.05 |
| 1,000 | 0.03 |

Unrelated vectors score about 0 with a spread of a few hundredths, so by a few hundred facts the right answer is hard to tell from noise. Hermes Agent's code estimates signal-to-noise as the square root of dimension over item count and logs a warning when it drops below 2, which is about 256 items at 1,024 dimensions. That's why real systems keep one vector per fact and use bundles only for small groups.

## The Hermes Agent holographic memory provider

[Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research, MIT license) has built-in memory files plus one optional external **memory provider**. Its [memory providers page](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) lists Holographic as "Local-only memory with advanced retrieval, no external dependencies." The plugin came from a community contribution (PR #2351) and lives in `plugins/memory/holographic/`.

### What it stores and how

From the [plugin source](https://github.com/NousResearch/hermes-agent/tree/main/plugins/memory/holographic):

- **Storage** is one local SQLite file (`$HERMES_HOME/memory_store.db` by default) with an FTS5 full-text index.
- **Facts** are short text rows with a category, tags and a **trust score** starting at 0.5.
- **Entities** are pulled out with regular expressions (capitalized multi-word names, quoted terms, "aka" patterns), not with an LLM.
- **Each fact gets an HRR vector**: its words bundled together and bound to a "content" role, plus each entity bound to an "entity" role. Atom vectors come from SHA-256 hashes, so they're identical on every machine.
- **Each category gets a bank**: a bundle of all its fact vectors, rebuilt on every write.
- **NumPy is optional.** Without it, the HRR features switch off and search falls back to keywords.

### How retrieval works

Plain search is hybrid. FTS5 finds candidates, then each is scored as 0.4 × keyword rank + 0.3 × word overlap (Jaccard) + 0.3 × HRR similarity, multiplied by the fact's trust score. Optional time decay can down-weight old facts.

The HRR-specific actions of the `fact_store` tool use unbinding:

1. **`probe`**: facts where a named entity plays the entity role.
2. **`related`**: facts structurally connected to an entity in any role.
3. **`reason`**: facts where *all* of several entities appear, an AND query done in vector space.
4. **`contradict`**: pairs of facts that share entities but have dissimilar content, flagged as possible conflicts.

The other actions are `add`, `search`, `update`, `remove` and `list`. A second tool, `fact_feedback`, moves trust up 0.05 when a fact helped and down 0.10 when it didn't. Facts below a trust of 0.3 are filtered out of search by default.

### Setup and status

Install it with `hermes plugins install holographic`, then enable it with `hermes memory setup` and pick "holographic", or `hermes config set memory.provider holographic`. Settings sit under `plugins.hermes-memory-store` in `config.yaml`: `db_path`, `auto_extract` (off by default), `default_trust` and `hrr_dim` (1,024 by default).

Holographic no longer ships inside Hermes. As of October 15, 2026 the Hermes docs say it is maintained by itpartypattaya ([hermes-plugin-holographic](https://github.com/itpartypattaya/hermes-plugin-holographic)) and installed from the plugin catalog, with the same provider name, settings and database. Nous Research's interim copy (`NousResearch/hermes-plugin-holographic`) is archived. RetainDB and ByteRover, which left core on the same date, are still unmaintained.

## Holographic memory vs vector and graph memory

| | HRR memory (Hermes Holographic) | Embedding vector store | Knowledge graph memory |
|---|---|---|---|
| Needs an embedding model or LLM | No | Embedding model | Usually an LLM for extraction |
| Finds paraphrases ("car" vs "automobile") | No, word atoms only | Yes | Depends on extraction |
| Entity and multi-entity queries | Yes, by unbinding | Only indirectly | Yes, by traversal |
| Grows without degrading | One vector per fact does; bundles don't | Yes | Yes |
| Runs fully offline | Yes, SQLite plus optional NumPy | With a local model | With a local LLM and DB |

The trade-off is clear. HRR gives cheap, deterministic, offline structure, but it doesn't understand meaning. Trained embeddings do, which is why most systems use them; see [vector databases for LLM memory](/articles/vector-database-for-llm-memory/). If entity relationships are what you need, a graph is the more common answer, covered in [knowledge graphs for AI memory](/articles/ai-memory-knowledge-graph/).

Within Hermes, the other providers take those routes. Mem0 runs server-side LLM fact extraction, Honcho models users across sessions, and [Hindsight](https://github.com/vectorize-io/hindsight), which the Hermes docs list for knowledge graph and entity resolution, extracts structured facts with an LLM. The [Hermes Agent memory guide](/articles/hermes-agent-memory/) covers the built-in layers and how providers plug in.

## When holographic memory makes sense

HRR memory fits a narrow but real set of cases:

- **Local, single-user agents** where you want no API calls and no model downloads for memory.
- **Entity-centric facts** ("what do I know about Project Atlas and Dana together?") where keyword search alone returns too much.
- **Experiments with symbolic structure**, since bind and unbind give you exact, inspectable operations.

It's a poor fit when users phrase things in many ways, when memory must be shared across users or machines, or when you need long-term support. If you're moving off Holographic after it leaves core and still want everything local, Hindsight can run with an embedded Postgres and a local model through Ollama, LM Studio or llama.cpp. It keeps entity queries (as graph search) and adds paraphrase matching, at the cost of an LLM call on every write, which HRR never needs. For the bigger picture of how agents store and recall facts, start with [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is holographic memory in AI agents?

It's agent memory built on holographic reduced representations (HRR), a 1995 technique by Tony Plate that stores associations by binding vectors with circular convolution and adding them into one fixed-size vector. You get facts back by unbinding with a cue. The best-known agent example is the Holographic memory provider in Hermes Agent.

### Is holographic memory better than a vector database?

Not in general. HRR lets you ask structural questions such as which facts involve a given entity, with no LLM or embedding model. But a superposed vector gets noisy as you add items, and plain HRR word vectors don't capture meaning the way trained embeddings do. In practice HRR is used as one signal next to keyword search.

### Is the Hermes Agent holographic memory provider still supported?

Yes, as a plugin. It left the Hermes core tree and, per the Hermes docs in October 2026, is maintained by itpartypattaya and installed with hermes plugins install holographic, keeping the same provider name, settings and database. RetainDB and ByteRover left core on October 15, 2026 and are unmaintained.

