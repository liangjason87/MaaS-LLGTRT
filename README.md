# MaaS-LLGTRT

Clone this fork of llgtrt https://github.com/natehofmann/llgtrt/tree/nathof/spec-decode-support, and switch to the branch `nathof/spec-decode-support`. Please note this is still a WIP so subject to change. If something doesn't seem right, this commit should be stable: [a95315adb0d7ed836bc59ac709e9984cb82c3d5d](https://github.com/guidance-ai/llgtrt/commit/a95315adb0d7ed836bc59ac709e9984cb82c3d5d)

From here, follow the same steps here: https://github.com/guidance-ai/llgtrt/tree/main?tab=readme-ov-file#building-or-pulling-docker-container to build a Docker image based off this fork/branch and setup a container based on this image.

## Target/Draft Models

In our benchmarks, we worked with Llama 3.1 8b/Llama 3.2 1b and Llama 3.3 70b/Llama 3.1 8b as our target/draft models respectively. In this example, we'll use 70b/8b. 

### Quantization

Download weights for both 70b/8b from HuggingFace. We are mostly following the quant/build steps here: https://github.com/datdo-msft/azureai-maas-trtllm/blob/main/Azure%20AI%20MaaS%20-%20Llama%20%2B%20TRT-LLM%20162bb3b8f56180fba84adb4286d8c59c.md, but for both models/additional parameters. Run the following command for both models:

`python /opt/TensorRT-LLM-examples/quantization/quantize.py --model_dir <path to hugging face weights> --dtype auto --qformat fp8 --kv_cache_dtype fp8 --output_dir <output directory for checkpoints> --calib_size 512 --tp_size 2 --pp_size 1`

### Build

- 70b (target model):
`trtllm-build --checkpoint_dir <path to checkpoints>  --output_dir  <output directory for engines>  --max_num_tokens 4096 --max_input_len 127000 --max_seq_len 128000 --max_batch_size 128 --use_fused_mlp enable  --use_paged_context_fmha enable --gemm_plugin fp8 --use_fp8_context_fmha enable --gemm_swiglu_plugin fp8 --max_draft_len=10 --speculative_decoding_mode=draft_tokens_external --gather_generation_logits`

- 8b (draft model):
`trtllm-build --checkpoint_dir <path to checkpoints>  --output_dir  <output directory for engines>  --max_num_tokens 4096 --max_input_len 127000 --max_seq_len 128000 --max_batch_size 128 --use_fused_mlp enable  --use_paged_context_fmha enable --gemm_plugin fp8 --use_fp8_context_fmha enable --gemm_swiglu_plugin fp8 --gather_generation_logits`

### Runtime Configuration

Mostly following the steps here: https://github.com/datdo-msft/azureai-maas-trtllm/blob/main/Azure%20AI%20MaaS%20-%20Llama%20%2B%20TRT-LLM%20162bb3b8f56180fba84adb4286d8c59c.md#run, with the exception of the `runtime.json`. Adjust the `kv_cache_free_gpu_mem_fraction` field accordingly to accomadate the GPU memory (in our tests we were using two H100s).

- `runtime.json`
    - Create this file like so:
        
        ```json
        {
          "guaranteed_no_evict": false,
          "max_batch_size": 128,
          "max_num_tokens": 8192,
          "max_queue_size": 0,
          "enable_chunked_context": true,
          "enable_kv_cache_reuse": true,
          "kv_cache_free_gpu_mem_fraction": 0.6,
          "kv_cache_host_memory_megabytes": 0,
          "enable_batch_size_tuning": true,
          "enable_max_num_tokens_tuning": true,
          "kv_cache_onboard_blocks": true,
          "cross_kv_cache_fraction": null,
          "secondary_offload_min_priority": null,
          "event_buffer_max_size": null
        }
        ```
### Starting the inference server

To start the inference server, after you have followed the steps above to set up the runtime configurations (i.e. move tokenizers, chat_template.j2, etc) the paths to both engine files for target/draft will be needed. Follow this command:

`/usr/local/bin/launch-llgtrt.sh <path to target engine> <path to draft engine> --port <port number> --debug`

If using 70b/8b in TP2 setup and two H100s, this will load the weights between the two GPUs, and spawn Draft/Target executors on both GPUS (for a total of 4 in this example).

### Benchmarking

Follow the same steps here: https://github.com/datdo-msft/azureai-maas-trtllm/blob/main/Azure%20AI%20MaaS%20-%20Llama%20%2B%20TRT-LLM%20162bb3b8f56180fba84adb4286d8c59c.md#running-benchmarks to setup a commonbench process to send requests to the inference server port specified above. 

  


