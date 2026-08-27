# Final Year Project

## Title: Hybrid GNN-LLM Pipeline for C/C++ Vulnerability Detection and Patch suggetion

## Objectives
- #### **Convert** the cached C/C++ parent codebase and new code into **a** graph representation.

- #### **Isolate** the subgraphs affected **by the** new code changes.

- #### **Construct a** GNN **classifier using** available **datasets** (e.g., MegaVul).

- #### **Perform** classification (vulnerable or safe) **on** the subgraphs using **the** GNN.

- #### If classified as vulnerable, pass the **subgraph** to **an LLM to explain** the vulnerability and suggest **a** patch.
