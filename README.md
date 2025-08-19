# RustOllama
### A Rust port of Ollama with GKE auto-orchestration for hosting Large Language Models
* Ver: 0.1 (early PoC)
* Author: Ash Mozano
  
## Background

I started “rustollama” (not to be confused with ollama-rs, a Rust library for interacting with the Ollama API) as a private hobby project a while back for porting ollama from Go to Rust (ollama-rs APIs + some of the pieces I'd refactored already show compelling performance and overhead deltas on my distributed hybrid hyperscaler + home GPU rigs).

I'm exploring options for auto-orchestration of Ollama nodes, each catering to a specific category of discovered devices (e.g., GPU, TPU, APU, CPU, etc.) to efficiently host specific models.  

The original reason for the project was to workaround the fact that some NVIDIA GPU drivers and CUDA libraries do not support all the families of NVIDIA Tesla GPUs on the same OS, including on the latest NGC containers.  Adding non-NVIDIA devices to the mix adds an even bigger challenge in setting up distributed MLOps rigs with lower-cost, legacy hardware.

Finally, Rust excels at creating fast, resilient, distributed systems with very little overhead.  Hence, RustOllama is born.

To be clear, the goal is not to compete with DGX (and NVLink, etc.), as those tightly-integrated systems will always be orders of magnitude more performant for enterprise use cases.  Rather, the goal is to enable the hobbyist community with older and less expensive hardware to efficiently automate the orchestration of all discoverable, ML-capable devices in a given networked environment, equip them with the right open source LLMs, and dynamically route the workload (training, inference, etc.) appropriately, somewhat akin to a dynamic weighted ensemble. 

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

## Kubeflow v. RustOllama

Both the RustOllama custom GKE auto-orchestration architecture and a Kubeflow-based setup aim to solve the same core problem: Serving diverse ML models on heterogeneous hardware. 
However, they approach it with different levels of abstraction and complexity.

The key difference is that RustOllama solution is intended to be a lightweight, bespoke implementation using fundamental Kubernetes objects, while Kubeflow is a comprehensive, opinionated MLOps platform.

### Which One to Choose?

**Choose RustOllama with custom GKE Auto-Orchestration approach if:**
* Your primary need is to solve the immediate problem of routing models to specific hardware.
* You value simplicity, transparency, and direct control over the infrastructure.
* You don't need the overhead of a full MLOps platform.
* You have a small team or limited Kubernetes expertise and want a solution that is easy to understand and manage.

**Choose a Kubeflow/KServe implementation if:**
* You are building a long-term, scalable MLOps platform for a data science team.
* You need advanced features like automated pipelines, experiment tracking, and serverless, scale-to-zero model serving.
* Your team is comfortable with the operational overhead of managing a more complex, feature-rich platform.
* You want to standardize your entire ML lifecycle, from development in notebooks to production deployment.

In essence, RustOllama is a pragmatic, focused solution for hobbyists, whereas Kubeflow offers a powerful, all-encompassing, enterprise-grade platform at the cost of increased complexity.

**And last but not least, RustOllama is written in Rust!**

Here’s a detailed comparison based on the architecture described:

| Feature | Rustollama | Kubeflow with KServe |
| :--- | :--- | :--- |
| **Core Idea** | Use a simple, stateless proxy to route requests to specialized, isolated Ollama deployments. | Use a dedicated model-serving component (KServe) that manages the entire lifecycle of model deployment. |
| **Complexity** | Lower. The architecture is built from standard, well-understood Kubernetes components (Deployments, Services, ConfigMaps, DaemonSets). The logic is transparent and easy to trace. | Higher. Kubeflow is a full-featured platform with many components (Pipelines, Katib, Notebooks, KServe). It requires a more significant initial investment in setup, configuration, and learning. |
| **Flexibility** | Very High. You have complete control over every aspect of the deployment—the routing logic, the container images, the driver installation, and the specific Ollama configurations. | Moderate. Kubeflow provides a structured, standardized way to do MLOps. While powerful, it imposes its own conventions and abstractions. Customizations may require a deeper understanding of Kubeflow's internal workings. |
| **Model Routing** | Explicit & Manual. You manually define the routing logic in a ConfigMap and implement the proxy yourself. This is simple but requires direct intervention to update. | Automated & Integrated. KServe (formerly KFServing), Kubeflow's model serving component, handles routing internally. It introduces its own Custom Resource Definition (InferenceService) that manages traffic splitting, canary rollouts, and routing to different model versions automatically. |
| **GPU & Driver Management** | Explicit. You manually create tainted node pools and deploy DaemonSets for specific drivers. This gives you precise control, which is ideal for legacy or conflicting driver requirements. | Integrated. Kubeflow leverages the underlying Kubernetes cluster's GPU support. While it works seamlessly with GKE's automatic driver installation, managing highly specific or conflicting driver versions might require similar manual DaemonSet configurations. |
| **Scalability** | Good. It scales well using standard Kubernetes Horizontal Pod Autoscalers (HPAs) on each Ollama deployment. You can autoscale node pools as well. | Excellent. KServe is built on top of Knative, which provides powerful event-driven, request-based autoscaling, including scale-to-zero. This is more resource-efficient, as models that are not in use will not consume any resources. |
| **Feature Set** | Lean & Focused. The solution does one thing: route requests. It does not include features for experiment tracking, pipelining, or hyperparameter tuning. | Rich & Comprehensive. Kubeflow is a complete MLOps toolkit. It includes: <br>- Kubeflow Pipelines: For orchestrating complex ML workflows. <br>- Katib: For hyperparameter tuning. <br>- Notebook Servers: For interactive development. <br>- KServe: For advanced model serving with features like A/B testing, explainability, and payload logging. |
| **Maintainability** | Simple. The small number of moving parts makes it easy to debug and maintain for a small team or a specific use case. | Complex. Maintaining a full Kubeflow installation is a significant operational task. It involves managing multiple controllers, CRDs, and a more complex dependency chain. |
