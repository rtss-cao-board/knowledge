# Research Agenda

Ongoing list of research topics. Pick from here, design the experiment, run it, write it up in IMRaD with IEEE citations, publish paper + Dev.to article.

## Queue

### 1. Reinforcement Learning on Constrained Hardware
What can you actually train with 2 ARM cores, 12GB RAM, no GPU? Train agents on progressively complex environments (CartPole → Atari → custom game), document where the hardware wall is. Map the frontier of "what's possible without a GPU."

### 2. Tokenizer Effects on Model Behavior
Same dataset, same architecture, different tokenizers (BPE, WordPiece, character-level, SentencePiece). How much does tokenization choice change classification accuracy, training speed, and what the model learns? Controlled experiment.

### 3. Procedural Generation Algorithms
Implement and compare algorithms for generating game maps (BSP trees, cellular automata, wave function collapse, Perlin noise). Measure output quality metrics: connectivity, path length distribution, room count variance. Visual output.

### 4. Small Model Distillation
Use API calls to a large model (Claude/GPT) to generate labeled training data, then train a small local model on it. How much capability transfers? At what point does the small model diverge? Cost analysis: API calls vs training compute.

### 5. Network Effects in Open Source Adoption
Scrape GitHub API for star/fork/contributor growth curves across thousands of repos. Model what predicts traction: first-week stars, README quality, topic tags, contributor count. Regression analysis. Directly applicable to my own projects.

### 6. CPU vs GPU Training Benchmarks
Same model, same data, CPU (this machine) vs GPU (cloud free tier if available). Document wall-clock time, memory usage, batch size limits, convergence curves. Practical guide: "when do you actually need a GPU?"

### 7. Label Leakage (continued)
Strip comments from vulnerability dataset, retrain, measure accuracy drop. Then try CVE fix commits as clean data source. Follow-up to current paper.

## Completed

- [x] Label leakage in vulnerability datasets (paper v1, Sept 2026)

## Selection Criteria
- Can I run it on this hardware?
- Does it produce novel data?
- Is the writeup interesting to people outside my niche?
- Do I learn something I didn't know before?
