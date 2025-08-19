# RustOllama
### A Rust port of Ollama with GKE auto-orchestration for hosting Large Language Models
* Ver: 0.1 (early PoC)
* Author: Ash Mozano
  
## TL;DR

I started “rustollama” (not to be confused with ollama-rs, a Rust library for interacting with the Ollama API) as a private hobby project a while back for porting ollama from Go to Rust (ollama-rs APIs + some of the pieces I'd refactored already show compelling performance and overhead deltas on my distributed hybrid hyperscaler + home GPU rigs).

I'm exploring options for container auto-orcehstrations through ollama, so that it can efficiently host specific models on specialized containers created for various discovered devices (e.g., GPU, TPU, APU, CPU, etc.).  The original reason was that some NVIDIA GPU drivers and CUDA libraries do not support all the NVIDIA Tesla GPUs on the same OS, and some were also not supported by the latest NGC containers.  Adding non-NVIDIA devices to the mix adds a whole different level of challenge of setting up distributed rigs with lower-cost, legacy hardware.

**To be clear, the goal is not to compete with DGX (and NVLink, etc.), but to efficiently automate the orchestration of all discoverable, ML-capable devices in a given networked environment.**

## Project Purpose

This project provides a reference architecture for deploying and managing multiple Ollama models on a single Google Kubernetes Engine (GKE) cluster with heterogeneous GPU hardware.

The primary challenge this solves is the operational complexity of hosting AI models with diverse and sometimes conflicting hardware and software requirements. 

For example:

Legacy models may require older NVIDIA drivers (e.g., for K-series GPUs) that are incompatible with modern GPUs.
Newer models may depend on recent CUDA libraries only available in the latest NGC (NVIDIA GPU Cloud) containers.
Running all models in a single, monolithic environment is often impossible or highly inefficient.

This solution implements an auto-orchestration or smart routing layer that automatically directs incoming API requests to the specific, containerized Ollama instance best equipped to serve the requested model, ensuring optimal resource utilization and compatibility.

## Solution Overview

The architecture leverages core Kubernetes concepts to create isolated, specialized environments for different GPU types within a single GKE cluster. A central routing service acts as the single entry point, inspecting each request and forwarding it to the appropriate backend Ollama service.

## Key Features

* Heterogeneous Hardware Support: Seamlessly run models on different GPU architectures (e.g., NVIDIA K80, M40, M60, T4, and A100) within the same cluster.
* Automated Driver Management: Uses Kubernetes DaemonSets to ensure the correct NVIDIA drivers and CUDA libraries are installed on each node based on its assigned GPU.
* Intelligent Request Routing: A central proxy automatically routes requests based on the model name in the API call, abstracting the backend complexity from the client.
* Scalability & Isolation: Node pools with taints and tolerations ensure that GPU resources are isolated and that workloads scale independently.
* Simplified Management: Centralizes the configuration of model-to-hardware mapping in a simple Kubernetes ConfigMap, making it easy to add or reassign models.

## Architecture Flow
* GKE Cluster Setup: The GKE cluster is configured with multiple node pools, where each pool is equipped with a specific type of GPU (e.g., k80-pool, t4-pool).
* Hardware Isolation: Each node pool is "tainted," which prevents pods from being scheduled on it unless they have an explicit "toleration." This guarantees that only designated workloads run on the specialized hardware.
* Driver Installation: A DaemonSet is deployed for each node pool to install the precise NVIDIA drivers required for that GPU architecture.
* Specialized Ollama Deployments: Separate Ollama instances are deployed, each targeting a specific node pool using nodeSelector and tolerations. Each instance can be pre-loaded with models compatible with its environment.

## Smart Routing:

* A client sends a request to a single, public-facing LoadBalancer Service.
* This service directs traffic to a central "Ollama Router" proxy.
* The router reads the model field from the request payload.
* It looks up the model in a ConfigMap to determine which internal Ollama service to use (e.g., ollama-k80-service or ollama-t4-service).
* The request is forwarded to the correct service, which processes it using the appropriate GPU.
* The response is streamed back to the client through the router.
* This setup creates a robust, scalable, and maintainable system for serving a diverse library of Ollama models efficiently.

To implement auto-orchestration for Ollama on GKE, we'll combine several Kubernetes features: multiple Node Pools, Node Taints and Labels, DaemonSets for drivers, and a custom routing layer.

## Core Strategy

The plan is to create a separate, specialized environment for each of our GPU types. A central router will then inspect incoming Ollama requests and forward them to the correct environment that has the right GPU and drivers to serve the requested model.

* Isolate Hardware: Create distinct GKE Node Pools for each GPU architecture (e.g., one for K-series, one for M-series, one for a modern series like T4 or A100).
* Assign Specific Drivers: Use Kubernetes DaemonSets to automatically install the correct NVIDIA driver and CUDA libraries onto each node in a specific pool.
* Deploy Specialized Ollama Instances: Run separate Ollama deployments, each configured to run on a specific node pool and serve a specific set of models compatible with that hardware.
* Implement Smart Routing: Deploy a lightweight proxy service that acts as the single entry point. This service will read the model name from the incoming API request and route it to the correct backing Ollama service.
