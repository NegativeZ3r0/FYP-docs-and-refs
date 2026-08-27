# C/C++ Hybrid Vulnerability Detection Architecture: A GNN-LLM Joint Specification

This document provides a complete, mathematically grounded architectural specification for a state-of-the-art hybrid vulnerability detection and patching pipeline tailored specifically for C/C++ target codebases. The design integrates **Memgraph** for incremental Code Property Graph (CPG) caching, a deterministic **Gatekeeper** for fast CI/CD filtering, a hybrid **GNN classifier (NNConv + GINEConv)** for structure-aware relational learning, and **CGP-Tuning** for cross-modal soft-prompt alignment with Code Large Language Models (LLMs).

---

## Architecture Overview

```
                      +------------------------------------------+
                      |        Incoming Git PR / Commit          |
                      +------------------------------------------+
                                           |
                                           v
+------------------+         +----------------------------+
|  Memgraph Cache  | <-----> |   Phase 1: Gatekeeper      | ---> [ SAFE ] (Exit Early)
| (Full Parent CPG)|         | (Incremental Taint Slicing)|
+------------------+         +----------------------------+
                                           | (Intersection Found)
                                           v
                             +----------------------------+
                             |  Backward Program Slicing  |
                             | (AST + CFG + PDG Subgraph) |
                             +----------------------------+
                                           |
                                           v
                             +----------------------------+
                             | Phase 2: Hybrid GNN        | ---> [ SAFE ] (Exit Early)
                             | (NNConv + GINEConv + ECC)  |
                             +----------------------------+
                                           | (Classified Vulnerable)
                                           v
                             +----------------------------+
                             | Phase 3: CGP-Tuning        |
                             | (Cross-Modal MHA Alignment)|
                             +----------------------------+
                                           |
                                           v
                             +----------------------------+
                             | Code LLM (Qwen2.5-Coder)   |
                             | - Natural Lang Explanation |
                             | - Automated Corrective PR  |
                             +----------------------------+
```

---

## Phase 1: Incremental CPG Caching & Gatekeeper Traversal

Processing a complete codebase's Code Property Graph (CPG) for every pull request is computationally prohibitive. This architecture utilizes **Memgraph** [71, 174] as a high-performance, in-memory graph database to cache the parent codebase CPG and perform ultra-fast path traversals.

### 1. Incremental CPG Caching
* **Parent CPG Caching:** The compilation of the base branch's CPG is stored permanently in **Memgraph** [172].
* **Git Delta Tracking:** For any incoming pull request, file-level SHA-256 cryptographic hashes are evaluated. Only modified files ($V_{edit}$) are parsed into micro-CPGs using Joern's fuzzy parser [103].
* **Dynamic Merging:** These micro-CPGs are dynamically merged with the cached parent graph in Memgraph by stitching call graph edges ($E_{CG}$) [161] and argument-to-parameter data-flows ($E_{IDFG}$) [103].

### 2. The Gatekeeper (Mathematical Taint Slicing)
Based on patch-based vulnerability discovery principles (PAVUDI), a patch is modeled as a 4-tuple [104]:
$$\mathcal{M} = (V_{source}, V_{sink}, V_{san}, V_{edit})$$

* $V_{source}$: Attacker-controlled inputs (e.g., `read`, `recv`, `getenv`, `scanf`) [104].
* $V_{sink}$: Security-critical execution sinks (e.g., `memcpy`, `strcpy`, `free`, pointer dereferences) [104].
* $V_{san}$: Identified sanitisation variables or boundary-checking routines.
* $V_{edit}$: Abstract Syntax Tree (AST) nodes directly introduced or modified by the Git diff [104].

The gatekeeper runs a deterministic path-traversal query in Memgraph to determine whether an execution or data-flow path $p$ exists such that:
$$p \text{ connects } V_{source} \rightarrow V_{sink} \quad \text{and} \quad p \cap V_{edit} \neq \emptyset$$

If no such path intersects the edit set, the PR is immediately declared **"Safe"** and bypasses the downstream neural classifiers [104].

### 3. Program Slicing
If intersection is confirmed, a **Backward Program Slicing** traversal is executed in Memgraph starting from the intersection nodes [105]. This isolates:
1. **Data Dependencies** via the Program Dependence Graph (PDG) to capture variables affecting the edit [105, 140].
2. **Control Dependencies** via the Control Flow Graph (CFG) to capture conditionals guarding execution [105, 140].

This isolates a focused subgraph, reducing total code representation volume by up to $90\%$ while retaining $100\%$ semantic fidelity [114].

---

## Phase 2: Hybrid Graph Neural Network (GNN) Classifier

Standard GNNs treat program graphs as homogeneous structures, ignoring the semantic differences between syntax trees (AST), execution control (CFG), and variable propagation (PDG). Our Phase 2 model preserves heterogeneity by combining **NNConv** and **GINEConv** to learn propagation rules across Joern's **21 discrete edge types** [76, 106].

```
               [ Joern Sliced Code Property Graph ]
               (33 Node Types, 21 Edge Types) [91]
                               |
               [Edge-Conditioned GNN Layers] [76]
        +----------------------+----------------------+
        |                                             |
        v                                             v
  [NNConv Block]                               [GINEConv Block] [108]
  Calculates edge-conditioned messages:        Preserves topological isomorphism:
  m_ij = MLP([h_i || h_j || e_ij]) [107]       h_i = MLP((1+ε)h_i + Σ(h_j + e_ij))
        +----------------------+----------------------+
                               |
                               v
                     [Multi-Scale Readout] [76]
                (Mean || Max || Attention Pooling) [76]
                               |
                               v
                 [Vulnerability Classification] ---> Focal Loss [88]
```

### 1. NNConv (Edge-Conditioned Message Passing)
To avoid the parameter explosion of assigning static weight matrices to all 21 edge types [106], we employ **NNConv** (Edge-Conditioned Convolution) [106]. An auxiliary Edge MLP dynamically calculates filter weights conditioned on the 21-dimensional one-hot edge attribute vector $e_{ij}$ [107]:
$$m_{ij} = \text{MLP}_{edge} \left( [ h_i \parallel h_j \parallel e_{ij} ] \right)$$

The local target node representation is then updated utilizing Batch Normalization (BN) and residual skip connections [107]:
$$h'_i = \text{BN} \left( \text{ReLU} \left( \text{MLP}_{node}(h_i) + \sum_{j \in \mathcal{N}(i)} m_{ij} \right) \right) + h_i$$

### 2. GINEConv (Graph Isomorphism Network with Edge Features)
While NNConv excels at learning separate multi-relational propagation rules, **GINEConv** maps local graph topology with a discriminative power equivalent to the Weisfeiler-Lehman (1-WL) graph isomorphism test [108]. GINEConv incorporates edge features into GIN's neighborhood summation [108]:
$$h_i^{(l+1)} = \text{MLP}_{GIN} \left( (1 + \epsilon) h_i^{(l)} + \sum_{j \in \mathcal{N}(i)} \text{ReLU}\left( h_j^{(l)} + e_{ij} \right) \right)$$

* $\epsilon$ is a learnable parameter scaling central node features against neighbors to prevent over-smoothing [108].
* $e_{ij}$ aligns AST syntax directly with data-flow edges.

### 3. Multi-Scale Readout & Classification
To classify the entire graph slice, node embeddings $H^{(L)}$ are aggregated into a single graph-level vector $h_G$ via a three-way concatenated readout [27, 76]:
$$h_G = [h_{mean} \parallel h_{max} \parallel h_{att}]$$

* **Global Mean Pooling:** Captures the structural baseline [76].
* **Global Max Pooling:** Highlights highly salient features (e.g., buffer writes) [76].
* **Global Attention Pooling:** Learns attention coefficients to focus on critical vulnerability contexts [76].

#### Classification & Focal Loss
The pooled embedding $h_G$ is mapped through a linear projection layer:
$$\hat{y} = \text{Sigmoid}\left(W_{class} h_G + b\right)$$

To combat class imbalance, the GNN is trained using **Focal Loss** [88]:
$$\mathcal{L}_{Focal} = -\alpha_t (1 - p_t)^\gamma \log(p_t)$$

---

## Phase 3: Structure-Aware Soft Prompt Alignment (CGP-Tuning)

If the GNN classifies the program slice as vulnerable ($\hat{y} \geq \text{threshold}$), the graph-level features must be mapped into the latent space of a Large Language Model. To achieve this with linear computational complexity, we utilize **CGP-Tuning** [29].

```
Word Embeddings (Xt) ---------> [ MHA Block 1 ] <--------- Soft Prompt Tokens (Xs)
                                       |
                            Text-Aligned Features (H4)
                                       |
Node Embeddings (H3) ---------> [ MHA Block 2 ]
                                       |
                            Graph-Aligned Features (H5)
                                       |
                                 [ FeedForward ]
                                       |
                                       v
                     Graph-Enhanced Soft Prompts (Z)
                                       |
                                       v
                         Pre-Trained Code LLM [49]
```

### 1. Word and Node Feature Extraction
* **Source Code Tokens:** Tokenized into word embeddings $X_t \in \mathbb{R}^{n_t \times d_{lm}}$ [42].
* **Graph Features:** The GNN's final layer node embeddings are pooled down to $n_k$ final node representation vectors: $H_3 \in \mathbb{R}^{n_k \times d_{lm}}$ [46].

### 2. Decoupled Cross-Modal Alignment
CGP-Tuning utilizes trainable soft prompt embeddings $X_s \in \mathbb{R}^{n_s \times d_{lm}}$ as a global query template [42, 47]. It decouples graph-text interactions into two sequential Multi-Head Attention (MHA) steps, reducing computational cost from multiplicative $\mathcal{O}(|V|N)$ [38] to linear $\mathcal{O}(n_k + n_t)$ [48]:

1. **Text Alignment:** Aligns global soft prompts with textual source code embeddings [47]:
   $$H_4 = \text{MHA}(Q = X_s, K = X_t, V = X_t)$$

2. **Graph Alignment:** Aligns the text-aligned prompts with GNN structural embeddings [47]:
   $$H_5 = \text{MHA}(Q = H_4, K = H_3, V = H_3)$$

3. **FFN Projection:** Projects the combined structural representations into the LLM's latent space [48]:
   $$Z = \text{FeedForwardNetwork}(H_5)$$

### 3. Autoregressive Explanation & Generation
The graph-enhanced soft prompt embeddings $Z$ are prepended to the original code word embeddings $X_t$ and passed into a frozen pre-trained Code LLM (e.g., Qwen2.5-Coder) [49]:
$$Y = \text{CodeLLM}([Z \parallel X_t])$$

Because $Z$ contains the aligned structural topology of the CPG (including data-flow paths, constraints, and pointer variables), the LLM generates a highly precise, hallucination-free explanation of the vulnerability and outputs a correct, syntactically-sound git patch [4, 5, 29].
