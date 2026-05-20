# GPU Learning & AI Architecture Summary

This document summarizes the key architectural learnings regarding GPUs, their ecosystem, and their role in modern AI infrastructure from our recent architecture discussions.

## 1. GPU vs. NPU: The Engineering Perspective
*   **GPU (Cloud & Training):** Flexible, massive parallel processing cores, and programmable. Used for heavy model training and cloud-based inferencing. Drawback: High power consumption and cost.
*   **NPU (Edge & AI PC):** Application-specific circuits designed purely for neural network math. Highly energy-efficient. Used for local inferencing on laptops or edge devices (e.g., IoT kiosks). 
*   **Architectural Migration:** You can prototype on massive Cloud GPUs (A100s), then quantize models (e.g., FP16 to INT4) and export them (e.g., ONNX format) to run locally on Edge NPUs, saving massive cloud compute costs and enabling offline capabilities.

## 2. Cognitive Infrastructure & Massive Parallelism
*   **Standard Infrastructure Limitation:** Sequential processing on CPUs fails for real-time NLP because transformers require billions of operations per second (e.g., recognizing sentiment on 10,000 simultaneous audio calls).
*   **The GPU Solution:** A single GPU has over 18,000+ cores. It slices massive workloads into tiny matrix math problems and processes them simultaneously. For the call center example, audio is converted to text and run through an NLP model directly in the GPU VRAM, returning results in milliseconds.

## 3. GPU Starvation & Parallel File Systems (Lustre)
*   **The Problem:** $16,000/hour GPU clusters sit idle (0% utilization) if the storage network cannot feed them data fast enough. Standard NAS, Hadoop, or Iceberg are too slow.
*   **The Solution (FSx for Lustre):** Lustre is a POSIX Parallel File System. It splits (stripes) training data across hundreds of NVMe drives. When the GPU requests data, all drives transmit their pieces simultaneously over high-speed networks (InfiniBand), delivering Terabytes per second.

## 4. Sizing GPU Clusters & VRAM Math
*   **The Golden Rule:** 1 Billion Parameters ≈ 2GB of VRAM (at 16-bit precision), plus ~20% overhead for context (KV Cache). Training requires 3x-4x more memory than inferencing.
*   **Sizing Example:** A 70B parameter model requires ~168GB of VRAM. A single GPU (e.g., 24GB or 80GB) cannot hold this. You must rent multi-GPU instances (e.g., an 8-GPU `g5.48xlarge` cluster) and shard the model across them using Tensor Parallelism.
*   **Budget Constraints:** With limited budgets (e.g., $1,000), running an 8-GPU cluster 24/7 is impossible. The architectural pivot is to use smaller SLMs (Small Language Models like Llama 3 8B) on single GPUs, or move to Serverless/Pay-Per-Token APIs (like Bedrock).

## 5. Deployment Ecosystems
*   **OPEA (Open Platform for Enterprise AI):** Standardizes microservices. You don't build CUDA environments from scratch; you pull OPEA-compliant Docker containers pre-configured for NVIDIA GPUs (using vLLM or TGI) and orchestrate them via a FastAPI backend.
*   **Vector DBs (Pinecone) vs Lustre:** Lustre is used to hydrate GPUs during the *Training* phase. Pinecone (or other Vector DBs) is used during the *Inferencing* phase as an external memory bank for RAG architectures, retrieving context before the LLM generates an answer.
