

## 目录：

1. TGI 工程简介
2. 涉及相关技术
3. 代码结构和开发
4. 思考和入手点




## TGI 工程简介

### 介绍：

Text Generation Inference (TGI) is a toolkit for deploying and serving Large Language Models (LLMs). TGI enables high-performance text generation for the most popular open-source LLMs, including Llama, Falcon, StarCoder, BLOOM, GPT-NeoX, and T5.

![[Pasted image 20231025162444.png]]



### 架构：

![[Pasted image 20231023112439.png]]


### 组件：

启动代码
``` shell
 text-generation-launcher --model-id  /home/admin/.cache/huggingface/hub/models--THUDM--chatglm2-6b/snapshots/b1502f4f75c71499a3d566b14463edd62620ce9f/ --trust-remote-code --max-input-length 2048 --max-total-tokens 12000
```

![[Pasted image 20231024234402.png]]
包括三个服务：
1. text-generation-launcher （启动器）
2. text-generation-router       （路由服务）
3. text-generation-server        （模型服务）

这是一个类似lightllm的架构，这个router服务，里面提供了prefill（预加载技术）
但是缺少了（detokenizer独立服务，可成为优化点）

### 特性：

- Simple launcher to serve most popular LLMs
- Production ready (distributed tracing with Open Telemetry, Prometheus metrics) -----
- Tensor Parallelism for faster inference on multiple GPUs 
- Token streaming using Server-Sent Events (SSE) ------
- Continuous batching of incoming requests for increased total throughput
- Optimized transformers code for inference using [Flash Attention](https://github.com/HazyResearch/flash-attention) and [Paged Attention](https://github.com/vllm-project/vllm) on the most popular architectures。 -----
- Quantization with [bitsandbytes](https://github.com/TimDettmers/bitsandbytes) and [GPT-Q](https://arxiv.org/abs/2210.17323)  -----
- [Safetensors](https://github.com/huggingface/safetensors)。weight loading
- Watermarking with [A Watermark for Large Language Models](https://arxiv.org/abs/2301.10226)  ------
- Logits warper (tempera。ure scaling, top-p, top-k, repetition penalty)
- Stop sequences
- Log probabilities
- Custom Prompt Generation: Easily generate text by providing custom prompts to guide the model’s output. -----
- Fine-tuning Support: Utilize fine-tuned models for specific tasks to achieve higher accuracy and performance.

## 相关技术简介
### FlashAttention
![[flashattn_banner.jpg]]
这是个将tensor计算转化为一种更利用并行计算和分片计算的算法。 
在使用中需要理解模型中Q、K、V怎么获取

### PageAttention vs TokenAttention

![[Pasted image 20231025011222.png]]
![[att.gif]]
这两种Attention是一类提升显存使用率，减少显存碎片和提升吞吐量的计算。
在真实业务中实际提升的效果相近

### Continous batching vs Static batching

![[cb_02_diagram-static-batching.png]]
静态的batch需要等待整批都处理完，才能开始下批任务

![[cb_03_diagram-continuous-batching.png]]
动态的batch，是任何一个完成了都可以接受新的任务


重点推论：
1.  推理计算是个IO密集型任务，而不是计算密集型任务。因为对于GPU来说加载1MB的速度远比计算1MB的速度要慢。
2. 显存资源是包含模型本身 + 输入输出的token。对于一个13B模型，每个token是需要1MB左右的显存。（看起来不大，但是要是有一个需要4096token的问题，意味着就需要4G的显存）。BatchSize就很难开大


### Safetensors
![[Pasted image 20231025002525.png]]
这是一个编码技术，从性能角度考虑，零拷贝和懒加载会有所帮助
### Watermark
![[Pasted image 20231025160249.png]]
watermark技术对模型调优有些帮助
## 代码结构和开发思路
### 代码结构：
![[Pasted image 20231025163852.png]]
模型服务相关的代码都在server目录下，模型适配的代码都在models，以及可以自定义的模型目录

### 重点代码：
=> server/text_generation_server/models/custom_modeling/flash_llama_modeling.py （以llama的模型为例）
![[Pasted image 20231025155221.png]]
``` python
class FlashLlamaAttention(torch.nn.Module):
    def __init__(
        self,
        prefix: str,
        config,
        weights,
    ):
        super().__init__()
        self.num_heads = config.num_attention_heads
        self.hidden_size = config.hidden_size
        self.head_size = self.hidden_size // self.num_heads

        # self.rotary_emb = PositionRotaryEmbedding.load(
        #     config=config, prefix=f"{prefix}.rotary_emb", weights=weights
        # )
        self.rotary_emb = PositionRotaryEmbedding.static(
            config=config,
            dim=self.head_size,
            base=config.rope_theta,
            device=weights.device,
        )

        self.softmax_scale = self.head_size**-0.5

        if self.num_heads % weights.process_group.size() != 0:
            raise ValueError(
                f"`num_heads` must be divisible by `num_shards` (got `num_heads`: {self.num_heads} "
                f"and `num_shards`: {weights.process_group.size()}"
            )
        self.num_heads = self.num_heads // weights.process_group.size()
        self.num_key_value_heads = (
            config.num_key_value_heads // weights.process_group.size()
        )

        self.query_key_value = load_attention(config, prefix, weights)

        self.o_proj = TensorParallelRowLinear.load(
            config,
            prefix=f"{prefix}.o_proj",
            weights=weights,
            bias=False,
        )
        self.num_groups = self.num_heads // self.num_key_value_heads
        self.kv_head_mapping = torch.arange(
            0, self.num_key_value_heads, dtype=torch.int32, device=weights.device
        ).repeat_interleave(self.num_groups)

    def forward(
        self,
        hidden_states,
        cos,
        sin,
        cu_seqlen_prefill,
        kv_cache,
        block_tables,
        slots,
        input_lengths,
        max_s,
    ):
        qkv = self.query_key_value(hidden_states)
        query, kv = qkv.split(
            [
                self.head_size * self.num_heads,
                2 * self.head_size * self.num_key_value_heads,
            ],
            dim=1,
        )
        query = query.view(-1, self.num_heads, self.head_size)
        kv = kv.view(-1, 2, self.num_key_value_heads, self.head_size)

        self.rotary_emb(query, cos, sin)
        self.rotary_emb(torch.select(kv, dim=1, index=0), cos, sin)

        vllm_cache_ops.reshape_and_cache(
            kv[:, 0], kv[:, 1], kv_cache[0], kv_cache[1], slots
        )

        # output tensor
        attn_output = torch.empty_like(query)

        # Prefill
        if cu_seqlen_prefill is not None:
            # flash attention
            attention(
                query,
                torch.select(kv, dim=1, index=0),
                torch.select(kv, dim=1, index=1),
                attn_output,
                cu_seqlen_prefill,
                max_s,
                self.softmax_scale,
            )
        # Decode
        else:
            # kv_cache[1] => [num_blocks, num_heads, head_size, block_size]
            block_size = kv_cache[1].shape[3]
            vllm_attention_ops.single_query_cached_kv_attention(
                attn_output,
                query,
                kv_cache[0],
                kv_cache[1],
                self.kv_head_mapping,
                self.softmax_scale,
                block_tables,
                input_lengths,
                block_size,
                max_s,
            )
        return self.o_proj(attn_output.view(-1, self.num_heads * self.head_size))
```
这个代码里面的三个方法：
- vllm_cache_ops.reshape_and_cache()
- vllm_attention_ops.single_query_cached_kv_attention()
- attention()
就是使用 FlashAttention和 PageAttention两大技术


![[Pasted image 20231025154637.png]]
```python
def load_attention(config, prefix, weights):
    if config.num_attention_heads != config.num_key_value_heads:
        return _load_gqa(config, prefix, weights)
    else:
        if config.model_type == "baichuan":
            return TensorParallelColumnLinear.load_qkv(
                config,
                prefix=f"{prefix}.W_pack",
                weights=weights,
                bias=False,
            )
        else:
            return TensorParallelColumnLinear.load_multi(
                config,
                prefixes=[f"{prefix}.q_proj", f"{prefix}.k_proj", f"{prefix}.v_proj"],
                dim=0,
                weights=weights,
                bias=False,
            )


def _load_gqa(config, prefix: str, weights):
    assert config.hidden_size % config.num_attention_heads == 0
    assert config.num_attention_heads % weights.process_group.size() == 0

    weight = weights.get_multi_weights_col(
        prefixes=[f"{prefix}.q_proj", f"{prefix}.k_proj", f"{prefix}.v_proj"],
        quantize=config.quantize,
        dim=0,
    )

    if config.quantize not in ["gptq", "awq"]:
        weight = weight.to(dtype=weights.dtype).to(device=weights.device)

        head_size = config.hidden_size // config.num_attention_heads
        num_heads = config.num_attention_heads // weights.process_group.size()
        num_key_value_heads = config.num_key_value_heads // weights.process_group.size()
        assert list(weight.shape) == [
            (num_heads + 2 * num_key_value_heads) * head_size,
            config.hidden_size,
        ], f"{list(weight.shape)} != {[(num_heads + 2 * config.num_key_value_heads) * head_size, config.hidden_size]}"

    return TensorParallelColumnLinear(
        get_linear(weight, bias=None, quantize=config.quantize)
    )
```
quantize 加载模型代码


## 思考
### 总结出的特点：

1. 支持量化(gptq， bitsandbytes)
2. 显示实现了PageAttention(vllm)和FlashAttention(flash_attn_cuda_v2)，这点对我们判断是否使用上和开发适配新模型都比较友好，参数好调整
3. 跟huggingface的集成度高，模型的加载是基于transformers代码的，对引入新模型的开发相对比较容易
4. 生产环境完备，也就是监控指标这些完毕。对生产环境的稳定性有帮助
5. 代码结构上支持custom prompt，对我们使用的场景友好

### 问题点：

1. 各个技术代码迭代很快
2. page_attention_v1 和 page_attention_v2 （编译过程中发现是写死了flash_attention代码库的两个commit，后续flash_attention代码的更新不会跟上）
3. vllm 包使用的是vllm上一版本(0.17.0)的代码，新代码完全不支持
4. 没有对国产软件有真实实践（代码里有出现baichuan的实现，测试过baichuan2的代码，不能直接使用）

### 思路：

各个优化技术的特点：
1. FlashAttention 是通过重新组织attention的计算来达到计算上的技术
2. PageAttention 和 TokenAttention是通过更好的使用显存排布来提升任务吞吐量
3. 量化技术是降低模型显存占用来提升batch size

测试和迭代方向
1. 借鉴baichuan模型的加载，测试实现TGI上对chatglm2-6B的调优（FlashAttention）
2. 测试实现4bit量化
3. 考虑引入类似lightllm的detokenizer异步进程

总结：
TGI框架几乎涵盖了所有我们之前做过chatglm2的测试，性能不是很好。

参考
https://vllm.ai -- vllm
https://www.anyscale.com/blog/continuous-batching-llm-inference -- continuous-batching
https://arxiv.org/pdf/2205.14135.pdf -- FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness
https://tridao.me/publications/flash2/flash2.pdf -- FlashAttention2
https://arxiv.org/pdf/2309.06180.pdf -- Efficient Memory Management for Large Language Model Serving with PagedAttention
https://arxiv.org/pdf/2301.10226.pdf -- A Watermark for Large Language Models
https://github.com/TimDettmers/bitsandbytes -- bitsnadbytes
https://arxiv.org/pdf/2210.17323.pdf -- GPTQ: ACCURATE POST-TRAINING QUANTIZATION FOR GENERATIVE PRE-TRAINED TRANSFORMERS