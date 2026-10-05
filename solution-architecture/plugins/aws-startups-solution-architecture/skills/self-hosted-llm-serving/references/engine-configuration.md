# Engine Configuration

Run one documented engine configuration per replica, written in infrastructure as code with every memory and concurrency setting explicit. Without a platform team, nobody will notice that a new image tag changed a default until the replica runs out of memory, so defaults are not a configuration.

## Pin the engine

Versions checked for this file on 2026-10-05:

- vLLM v0.31.0 (released 2026-10-05). Docs for that version: [docs.vllm.ai/en/v0.31.0](https://docs.vllm.ai/en/v0.31.0/).
- SGLang v0.5.21 (released 2026-10-02). Docs at that tag: [server arguments](https://github.com/sgl-project/sglang/blob/v0.5.21/docs/docs/advanced_features/server_arguments.mdx).

Both engines move fast. Pin the container image by digest, record the engine version next to the model revision, and treat an engine upgrade like a model change: re-read the startup log, re-run `references/benchmark-on-target.md`, and compare against the previous result before shifting traffic. Flag names and defaults below are from these versions; check them against the pinned version's docs before use.

Text Generation Inference (TGI) is archived on GitHub and its README says it is in maintenance mode, recommending vLLM or SGLang instead ([repository](https://github.com/huggingface/text-generation-inference)). Treat an existing TGI deployment as a migration source. Do not translate its flags one to one; derive the new engine's settings from the measured context and concurrency in `references/fit-one-gpu.md`.

## Settings that decide memory and concurrency

| Decision                     | vLLM                                                       | SGLang                                                            | What it trades                                                                                                                                                   |
| ---------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GPU memory the engine claims | `--gpu-memory-utilization` (fraction for the whole engine) | `--mem-fraction-static` (fraction for weights plus KV cache pool) | Higher means more KV cache and concurrency, less headroom for activations and CUDA graphs                                                                        |
| Longest request              | `--max-model-len`                                          | `--context-length`                                                | Defaults come from the model config, often far above real traffic; set from measured p99 request length                                                          |
| Requests in flight           | `--max-num-seqs`                                           | `--max-running-requests`                                          | Cap at the concurrency the latency target allows; above it, requests queue                                                                                       |
| Prefill chunk per step       | `--max-num-batched-tokens`                                 | `--chunked-prefill-size`                                          | Smaller favors inter-token latency for running requests, larger favors time to first token for new ones                                                          |
| KV cache precision           | `--kv-cache-dtype`                                         | `--kv-cache-dtype`                                                | 8-bit roughly halves cache memory per token; needs an evaluation run                                                                                             |
| Weight quantization          | `--quantization`                                           | `--quantization`                                                  | Less weight memory; supported formats depend on GPU architecture and engine version                                                                              |
| GPUs per replica             | `--tensor-parallel-size`                                   | `--tp-size`                                                       | Leave at 1 unless `references/fit-one-gpu.md` reached its last step                                                                                              |
| Fixed KV cache size          | `--kv-cache-memory-bytes`                                  | `--max-total-tokens`                                              | Makes the budget explicit; vLLM's value is only valid on the GPU and free memory it was measured on, and SGLang documents its flag for development and debugging |
| Metrics endpoint             | `/metrics` on the API server                               | `--enable-metrics`                                                | Needed for `references/benchmark-on-target.md` and for scaling signals                                                                                           |

Sources: vLLM [engine arguments](https://docs.vllm.ai/en/v0.31.0/configuration/engine_args/), [conserving memory](https://docs.vllm.ai/en/v0.31.0/configuration/conserving_memory/), [optimization and tuning](https://docs.vllm.ai/en/v0.31.0/configuration/optimization/) and [production metrics](https://docs.vllm.ai/en/v0.31.0/usage/metrics/); SGLang [server arguments](https://github.com/sgl-project/sglang/blob/v0.5.21/docs/docs/advanced_features/server_arguments.mdx) and [hyperparameter tuning](https://github.com/sgl-project/sglang/blob/v0.5.21/docs/docs/advanced_features/hyperparameter_tuning.mdx).

Two defaults worth knowing in these versions: vLLM's `gpu_memory_utilization` defaults to 0.92, and its prefix caching is on by default (`vllm/config/cache.py` at v0.31.0). Set both explicitly anyway, so the configuration does not change when the default does.

SGLang's tuning guide is written for throughput: it calls a queue of 100 to 2000 waiting requests healthy and aims for KV cache usage above 0.9. That is the right target for offline work and the wrong one for an online latency target, where any sustained queue shows up as time to first token. Tune online replicas against the latency target in `references/benchmark-on-target.md`.

## Confirm the budget at startup

After every configuration change, read the startup log before sending traffic:

- vLLM: `GPU KV cache size: <N> tokens, Maximum concurrency for <L> tokens per request: <X>x`. X must cover the concurrency the latency target needs at context L.
- SGLang: `max_total_num_tokens=..., max_running_requests=..., context_len=..., available_gpu_mem=...`.

Record the line in the deployment's change record. It is the measured memory budget the skill's deliverables ask for.

## Serve the model with a predictable start

Online replicas restart: on deploys, node replacement and scale-out. Make the start time short and known.

- Stage model weights in Amazon S3 at an immutable revision with a manifest, as the batch skill's `prepare-model-and-input.md` describes, and read them from there at start rather than from a public model hub. Use `Skill("aws-core:aws-storage")` for the storage mechanics.
- Keep the engine image in Amazon ECR, close to the cluster.
- Measure the start in parts: capacity acquisition, node boot, image pull, weight load, and engine warm-up (compilation and CUDA graph capture). vLLM documents ways to shorten repeated boots, including reusing its compile cache and `--enforce-eager`, which skips CUDA graph capture at a cost in steady-state decode speed ([optimization and tuning](https://docs.vllm.ai/en/v0.31.0/configuration/optimization/)).
- The EKS user guide lists cold-start reductions for inference Pods: SOCI parallel image pull (on by default in EKS Auto Mode for GPU instances), streaming weights from S3 to GPU memory, ECR over a VPC endpoint, and instance store caching ([Run AI/ML inference workloads on Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/ml-inference.html)).
- Gate traffic on readiness. Both engines expose `/health` on their HTTP server. On ECS with a load balancer, set the service's health check grace period longer than the measured start; the default is 0 ([ECS service properties](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs.CfnServiceProps.html)). On Kubernetes, use a startup probe sized the same way.

## GPU node prerequisites by platform

Route the platform mechanics to `Skill("aws-core:aws-containers")` (EKS and ECS) or `Skill("aws-core:aws-compute")` (EC2). What the engine needs from each:

- EKS: the EKS accelerated AMI ships the NVIDIA driver. On Bottlerocket the NVIDIA device plugin is pre-installed; on AL2023 it is not and must be installed separately, typically as a DaemonSet. Check that the node advertises `nvidia.com/gpu` ([EKS best practices, AI/ML compute](https://docs.aws.amazon.com/eks/latest/best-practices/aiml-compute.html)).
- ECS on EC2 container instances: use the ECS GPU-optimized AMI, set `ECS_ENABLE_GPU_SUPPORT=true` in the agent configuration, and request GPUs in the task definition ([ECS task definitions for GPU workloads](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-gpu.html)).
- ECS Managed Instances: supports GPU instance types with NVIDIA drivers and CUDA pre-installed; check the supported list for the family chosen in `references/fit-one-gpu.md` ([Use GPUs with Amazon ECS Managed Instances](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/managed-instances-gpu.html)).
- EC2 directly: an AMI with a driver that matches the engine image's CUDA version, and a process supervisor that restarts the engine. One model server per instance keeps the memory budget valid.

## Load balancer in front of a streaming engine

Responses from an LLM can stream for longer than typical web requests, and a queued request or a long prompt can send nothing until its first token. An Application Load Balancer closes a connection that sends no data for its idle timeout, 60 seconds by default and configurable from 1 to 4000 seconds ([ALB attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html)). Set it above the longest measured time to first token plus margin, and keep the engine's keep-alive timeout above the load balancer's.
