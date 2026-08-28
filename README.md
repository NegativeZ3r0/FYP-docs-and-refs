# Final Year Project

### Hybrid GNN-LLM Pipeline for C/C++ Vulnerability Detection and Patch suggetion
---
### Objectives
- #### Convert the cached C/C++ parent codebase and new code into a graph representation.

- #### Isolate the subgraphs affected by the new code changes.

- #### Construct a GNN classifier using available datasets (e.g., MegaVul).

- #### Perform classification (vulnerable or safe) on the subgraphs using the GNN.

- #### If classified as vulnerable, pass the subgraph to an LLM to explain the vulnerability and suggest a patch.
---
### Youtube links
- #### 3b1b: [Neural networks](https://youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi&si=YywqY28auQHWvN2x)
- #### Welch labs: [Neural Networks Demystified](https://youtube.com/playlist?list=PLiaHhY2iBX9hdHaRr6b7XevZtgZRa1PoU&si=drADguL2u0tl96Bp)
- #### Stanford: [Stanford CS224W Machine Learning with Graphs | Jure Leskovec](https://youtube.com/playlist?list=PLoROMvodv4rOP-ImU-O1rYRg2RFxomvFp&si=aWfrEHd7_dotZrvL)
- #### [Graph-based Vulnerability Discovery - Featuring Dr. Tom Ganz](https://www.youtube.com/watch?v=KWG5v1oHwMM)
- #### Vizuara: [Graph Neural Networks - Theory, Applications and Research](https://youtube.com/playlist?list=PLPTV0NXA_ZSg4Pimkso0nHxwYMB6IGX7l&si=LLfTAqWyhhvchRFj)
- #### [ICLR 2021 Keynote - "Geometric Deep Learning: The Erlangen Programme of ML" - M Bronstein](https://www.youtube.com/watch?v=w6Pw4MOzMuo&pp=0gcJCRMMAYcqIYzv)
---
### Articles
- #### [Codebadger github](https://github.com/qcri/codebadger)
- #### [Codebadger website](https://www.codebadgertech.com/)
- #### [NiteAgent: Code Review Agent](https://niteagent.com/blog/building-ai-code-review-agent/?hl=enIN#:~:text=Multi%2DAgent%20Code%20Review%20Architecture%20The%20production%20architecture,review%2C%20deduplicates%2C%20%E2%94%82%20%E2%94%82%20prioritizes%2C%20formats%20output)
- #### [Dzone: Modern Vulnerability Detection: Using GNNs to Find Subtle Bugs](https://dzone.com/articles/modern-vulnerability-detection-gnns-subtle-bugs)
- #### [keygraph, The Startup](https://keygraph.io/agentic-sast)

---
### Datasets

| **Dataset**                                                                               | **Type**                      | **Languages / CWEs**       | **Scale / Key Stats**                      | **CPG & GNN Training Relevance**                                                                 |
| ----------------------------------------------------------------------------------------- | ----------------------------- | -------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| [**MegaVul**](https://huggingface.co/datasets/hitoshura25/megavul)                        | Real-World                    | C/C++, Multiple CWEs       | 952 vuln, 15,871 non-vuln (1,904 balanced) | Includes pre-computed Joern CPGs with 33 node and 21 edge types; ideal for `FastVulnGNN`.        |
| [**DiverseVul**](https://huggingface.co/datasets/bstee615/diversevul)                     | Real-World                    | C/C++, 150 CWEs            | 330k–523k functions (41k+ vuln)            | Serves as the primary large-scale benchmark for training `VulGNN` and graph-prompt models.       |
| [**PrimeVul**](https://huggingface.co/datasets/starsofchance/PrimeVul)                    | Real-World                    | C/C++, 140 CWEs            | 6,968 vuln, 228,800 safe functions         | Chronologically split to eliminate data leakage; high label quality for CPG slice generation.    |
| [**FormAI-v2**](https://huggingface.co/datasets/Joshfcooper/formai-v2-full)               | Synthetic / Formally Verified | C programs, 4+ mapped CWEs | 331,000 compilable C programs              | Labeled via ESBMC bounded model checking; eliminates false negatives for graph feature learning. |
| [**SARD / Juliet Test Suite**](https://samate.nist.gov/SARD/test-suites/112)              | Synthetic                     | C/C++, Java, PHP, C#       | 64k+ test cases (33.3k used in VulGNN)     | Provides precise sink/statement annotations for node-level and graph-level classification.       |
| [**Big-Vul (BigVul)**](https://huggingface.co/datasets/bstee615/bigvul)                   | Real-World                    | C/C++, 348 CVE entries     | 11,834 vuln, 253k non-vuln functions       | Widely used for extracting PDG/CPG code slices (e.g., in GGNN training pipelines).               |
| [**ReposVul**](https://github.com/Eshe0922/ReposVul)                                      | Real-World                    | 4 Languages, 236 CWEs      | 6,134 CVE entries from 1,491 projects      | Multi-file and inter-procedural repository-level context for evaluating graph generalizability.  |
| [**SVEN**](https://huggingface.co/datasets/bstee615/sven)                                 | Real-World                    | C/C++, Python              | ~1,600 verified program pairs              | High-quality paired vulnerable-patched samples for graph-diff/slice extraction.                  |
| [**D2A**](https://huggingface.co/datasets/claudios/D2A)                                   | Real-World                    | C/C++                      | 1,295 vuln, 5,393 non-vuln functions       | Differential static analysis benchmark used for cross-domain evaluation.                         |
| [**FFmpeg & QEMU**](https://github.com/VulDetProject/ReVeal)                              | Real-World                    | C/C++                      | Thousands of patch commits                 | Source for inter-procedural taint graphs and data-flow dependency subgraphs.                     |
| [**CVEfixes / CrossVul / ReVeal / Devign**](https://github.com/secureIT-project/CVEfixes) | Real-World                    | C/C++, Multi-language      | Aggregated into DiverseVul                 | Foundational function-level CPG benchmarks for graph neural network baselines.                   |
| **CVE-Details**                                                                           | Security Metadata             | Multi-domain CVE records   | 716,771 vulnerability records (1999–2017)  | Used for token/entity graph embeddings (e.g., `Vulnerability2Vec`).                              |

### Key Dataset Details for GNN Training
- **DiverseVul & Its Aggregations**: DiverseVul incorporates deduplicated subsets from older benchmarks including **Devign**, **ReVeal**, **Big-Vul**, **CrossVul**, and **CVEfixes**. It is used to train `VulGNN` via `GeneralConv` graph attention layers.
- **MegaVul**: Designed specifically with pre-generated Joern CPGs. It aligns directly with multi-relational GNNs (e.g., `FastVulnGNN`), supporting 33 distinct node types and 21 relation edge types (such as `AST`, `CFG`, `REACHING_DEF`, and `POST_DOMINATE`).
- **FormAI-v2**: Contains LLM-generated code verified using ESBMC (Efficient SMT-based Bounded Model Checker). It provides noise-free ground truth for training CPG query generators and edge-conditioned classifiers without commit-scraping label noise. 
- **SARD / NIST Juliet**: Provides ground-truth statement-level triggers (e.g., CWE-121 through CWE-126 buffer overflows). It is used for both node classification (sink point detection) and hybrid training when mixed with real-world code to stabilize GNN convergence.
---
## [Agent workflow architecture](complete-architecture-specification.md)

