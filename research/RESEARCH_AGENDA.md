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

### 8. Facial Recognition on Constrained Hardware
Train and evaluate face detection + recognition pipelines on CPU-only ARM. Compare approaches: Haar cascades vs HOG+SVM vs lightweight CNNs (MobileFaceNet, ArcFace-lite). Measure accuracy, speed (FPS), and minimum hardware requirements. Test with public datasets (LFW, WIDER FACE).

### 9. Person Re-identification Across Camera Views
Given images of a person from one camera, can a small model identify them in a different camera's feed? Train re-ID models on Market-1501 or DukeMTMC datasets. Measure rank-1 accuracy and mAP on CPU. Explore feature extraction vs end-to-end approaches.

### 10. Vehicle Detection and Classification
Train models to detect and classify vehicles (car, truck, motorcycle, bus) from security camera frames. Compare YOLO-tiny vs SSD-MobileNet vs custom lightweight architectures on CPU. Measure FPS achievable on ARM for real-time processing. Use public datasets (PASCAL VOC, COCO vehicle subset).

### 11. Smart Security: Anomaly Detection in Video
Can a model learn "normal" activity patterns and flag anomalies without labeled data? Unsupervised/self-supervised approaches on surveillance footage. Autoencoders, prediction-based methods. Test on UCF Crime dataset or similar. Focus on what runs in real-time on edge hardware.

### 12. License Plate Detection and OCR Pipeline
End-to-end pipeline: detect plate region → segment characters → recognize text. Compare traditional CV (contour detection + template matching) vs lightweight neural approaches. Test on public plate datasets across different formats (US, EU, custom).

### 13. Neural Architecture Comparison on Identical Tasks
Take one classification task and run it through every major architecture: MLP, CNN, RNN/LSTM, GRU, Transformer, Mamba (state space), KAN (Kolmogorov-Arnold), Graph Neural Network, Capsule Network, Hopfield Network (modern), Mixture of Experts. Same data, same compute budget, same metrics. Which architectures suit which data shapes? Where do newer architectures actually beat older ones vs just hype?

### 14. Novel Architecture: Thermodynamic Neural Networks
Inspired by Extropic's TSU direction. Can we build a neural network where neurons follow thermodynamic principles — energy minimization, entropy gradients, Boltzmann distributions? Implement from scratch in PyTorch, test on standard benchmarks, compare against conventional architectures. If it works, package as a library.

### 15. Novel Architecture: Topological Neural Networks
Use persistent homology and topological features as network layers. Input data gets mapped to topological descriptors (Betti numbers, persistence diagrams), which feed into learnable layers. Could capture structural patterns that spatial convolutions miss. Implement, test, publish as library if viable.

### 16. Novel Architecture: Causal Attention Networks
Standard attention is correlational. Build an attention mechanism that estimates causal relationships between sequence elements using do-calculus inspired operations. Test on tasks where causation matters (time series, event sequences). Compare against standard transformers.

### 17. Spiking Neural Networks on CPU
Neuromorphic computing without neuromorphic hardware. Implement leaky integrate-and-fire neurons, train with surrogate gradient methods. Benchmark energy efficiency (FLOPs per inference) against conventional networks. Test on event-based datasets and standard image classification.

### 18. Architecture Effectiveness Library (ArchBench)
Package all tested architectures (#13-17) into a single library where users can swap architectures on the same task with one line. `archbench.train(data, arch='mamba')` vs `archbench.train(data, arch='transformer')`. Automatic benchmarking and comparison reports.

## Completed

- [x] Label leakage in vulnerability datasets (paper v1, Sept 2026)

## Selection Criteria
- Can I run it on this hardware?
- Does it produce novel data?
- Is the writeup interesting to people outside my niche?
- Do I learn something I didn't know before?
