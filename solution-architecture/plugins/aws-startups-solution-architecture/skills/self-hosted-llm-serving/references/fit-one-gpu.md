# Fit One GPU First

Online serving needs GPU memory for three things: the model weights, the KV cache for every request in flight, and the engine's own overhead (activations, CUDA graphs, the runtime). The KV cache is the part that grows with traffic, so a model that loads cleanly can still fail under real concurrency. Size all three before choosing an instance, and choose the smallest single GPU that holds them.

## Why one GPU is the default here

A second GPU is a large step on AWS list prices. Price List GetProducts for Linux on-demand, shared tenancy, us-east-1 and eu-central-1, re-pulled 2026-10-05: in every current-generation G family that has a multi-GPU size, the smallest multi-GPU size lists at least twice as high as the family's smallest single-GPU size. The lowest ratio is exactly 2x, the Blackwell families g7 and g7e sit below 3x, and g4dn, g5, g6 and g6e sit above 5x, because those four families jump from one GPU straight to four.

With a committed fleet, that jump is absorbed by capacity already paid for. From a fixed ceiling it at least doubles the idle floor computed in `references/self-host-or-managed.md`. So exhaust the single-GPU options below before crossing it, and re-check the ratio for the Region in scope with `Skill("aws-core:aws-billing-and-cost-management")` rather than trusting this date.

Two related traps visible in the same Price List data:

- Larger sizes within a G family often keep one GPU. In g5, g6 and g6e, every size from xlarge to 16xlarge has a single GPU with the same GPU memory; moving up a size buys vCPUs and host memory, not KV cache.
- A large single-GPU size can list above the family's multi-GPU size (g4dn.16xlarge lists above g4dn.12xlarge in us-east-1 on 2026-10-05). Compare per GPU memory, not per size name.

## Measure the inputs

- GPU memory per GPU from EC2 `DescribeInstanceTypes` (`GpuInfo.Gpus[].MemoryInfo.SizeInMiB`), not from a marketing page. On 2026-10-05 in us-east-1 it returned 16384 MiB for the T4 in g4dn, 22888 MiB for the A10G in g5 and the L4 in g6, 32768 MiB for the RTX PRO 4500 in g7, 45776 MiB for the L40S in g6e, and 98304 MiB for the RTX PRO Server 6000 in g7e. The Price List shows the A10G and L4 as "24 GB", which is decimal gigabytes and the same 22888 MiB, or about 22.4 GiB in the units engine logs use. Keep one unit throughout the budget. Get candidates from `Skill("aws-core:aws-compute")`.
- Model weights size from the staged model files, after the chosen dtype or quantization. Parameter count times bytes per parameter is a first filter only: a 13B model at 2 bytes per parameter is about 24796 MiB of weights (derived), which already exceeds one A10G before any KV cache, and fits one L40S with room left for KV cache.
- KV cache bytes per token from the model's `config.json`: 2 (keys and values) times layers times key-value heads times head dimension times bytes per element. Models with sliding-window, latent or hybrid attention use less than this formula says; the engine log below is authoritative.
- Context per request from measured traffic: p99 input tokens plus p99 output tokens, not the model's maximum context.
- Concurrency from `references/benchmark-on-target.md`: the number of requests in flight that the latency target allows. Online, this comes from traffic under a latency target; it is not the largest batch the GPU can hold.

Do the arithmetic in a script. The formula shortlists GPUs; only the engine's startup log confirms one.

## Confirm with the engine

Start the engine on the candidate GPU with the context and concurrency settings from `references/engine-configuration.md`, then read the startup log:

- vLLM logs `GPU KV cache size: <N> tokens` and `Maximum concurrency for <L> tokens per request: <X>x`. If X is below the concurrency the latency target needs at context L, the GPU does not fit this workload.
- vLLM refuses to start when one request at the configured maximum context does not fit, with an error that states the estimated maximum model length the memory allows.
- SGLang logs `max_total_num_tokens`, `max_running_requests`, `context_len` and `available_gpu_mem` once the server is ready. Divide `max_total_num_tokens` by the measured context per request to get the concurrency it can hold.

Sources: vLLM v0.31.0 `vllm/v1/core/kv_cache_utils.py`; SGLang v0.5.21 [hyperparameter tuning](https://github.com/sgl-project/sglang/blob/v0.5.21/docs/docs/advanced_features/hyperparameter_tuning.mdx).

## When it does not fit, in this order

1. Bound context. Set the maximum model length to the measured p99 request length plus a margin, not the model's advertised maximum. KV cache reserved for contexts nobody sends is memory taken from concurrency.
2. Bound concurrency. Cap requests in flight at the concurrency the latency target needs. Past that point extra requests queue rather than evict others from the cache. vLLM documents that running short of KV cache causes preemption and recomputation, which hurts latency ([optimization guide](https://docs.vllm.ai/en/v0.31.0/configuration/optimization/)).
3. Quantize the KV cache. An 8-bit KV cache dtype roughly halves cache memory per token against 16-bit. Run the evaluation set before and after.
4. Quantize the weights. Check the pinned engine version's [quantization hardware table](https://docs.vllm.ai/en/v0.31.0/features/quantization/) against the GPU architecture (T4 is Turing, A10G is Ampere, L4 and L40S are Ada). Prefer a published quantized checkpoint of the same model to quantizing at load time. Accept it only if the evaluation set shows no regression the product cares about.
5. Take a larger single GPU. Moving from a 22888 MiB GPU to a 45776 MiB or 98304 MiB GPU keeps one GPU and one replica. On 2026-10-05 in us-east-1, g6e.xlarge (one L40S, 45776 MiB) listed at about a third of g5.12xlarge (four A10G, 91552 MiB in total) (Price List, derived ratio).
6. Only then, tensor parallelism across the GPUs of one instance. It is in scope for this skill when nothing above fits. Record why each earlier step failed, because the decision doubles the floor at minimum. Serving one model across several nodes is out of scope.

## Leave headroom

Keep measured headroom between the configured memory fraction and the GPU total for activations, CUDA graphs and the runtime. SGLang's tuning guide suggests reserving 5 to 8 GB for activations as a rule of thumb, which is a large share of a 22888 MiB GPU; measure it on the target instead. A replica that runs out of memory at the first long request is worse than one sized for slightly less concurrency, because with one warm replica there is no second replica to absorb the restart.
