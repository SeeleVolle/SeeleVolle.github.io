# SwiftTransformer 

### 推理流程

Prompts -> Tokenizer -> Embeding+positional encoding -> LLM(TransformerxN) -> Samplings -> New Token -> Embedding+positional encoding

GPT模型结构：

<img src="../../assets/image_GPT.png" style="zoom:100%;" />

两阶段的Attention结构 + Tensor Parallelism

<img src="../../assets/image_GPT2.png" style="zoom:100%;" />

### 项目依赖

该项目添加了Conda, Cudatookit, MPI, NCCL, Python, LibTorch, GoogleTest, nlohmann_json, argPrase, Pybinding, tpqd库，使用xformers的cuda kernel作加速

### 模块划分

模块主要分为kernel, layer、model和utils四个文件夹

+ Kernel: 引用了xformers库，基于该库实现了以下文件
  + activation_types.cuh: 激活函数在GPU上的具体实现，inline+constexpr, 包括了RELU, SILU和GELU
  + addbias.cu: 两个数组point-wise的相加, and batched version
  + count_nan.cu: 计算一个数组内的非零元素个数
  + embedding.cu: batched embedding and positional encoding
  + findmax.cu: 在一个数组中寻找最大元素，返回下标，并且实现了batched版本
  + fused_activ_multiply.cu: 实现了`output[i] = activation(input1[i]) * input2[i]`
  + fused_addbias_activ.cu: 实现了batched的`output[i] = activation(input[i] + bias[i])`
  + fused_context_stage_attention.cu: 实现了context stage的attention操作，即考虑输入`Q,K,V`，输出`softmax(QK^T·scale+mask)*V`，优化包括flash-attention, tensorcore, group query attention，计算了input中每个token的attention score
  + fused_decoding_stage_attention.cu: 实现了decoding stage的attention操作，具体来说， i.e. the decoding stage. It takes input tokens, k cache and v cache, calculate softmax(qK^T)V (Here q is a vector of length num_heads*head_dim), and store the result in `result`。和context阶段不同的是，decoding阶段每个token/prompts只用考虑最近的一个token
  + fused_decoding_stage_attention_mha.cu: 实现了decoding stage的attention_mha操作，针对MHA做了优化
  + gather_last_token.cu: 获取每个request的最后一个token加入到arary中, 由于每个request可能处于不同的阶段，因此需要获取不同阶段的最后一个token
  + kvcache_mgmt.cu: 实现了将刚刚计算出的q/k/vs存入Kvcache的操作
  + layernom.cu: 实现了layernorm的操作
  + reduction.cuh: 实现了warpReduceSum, blockReduceSum, warpReduceMax, blockReduceMax 
  + rmsnorm.cu: 实现了rmsnorm的操作，rmsnorm是layernorm的平替...
  + rotary_posi_embedding.cu: 实现了rotary positional embedding的token操作
  + unfused_attention.cu: 从QKV buffer中读出q,k,v并存入q_buf, k_buf, and v_buf, 以及将contextDecode产生的output attention matrix融入到一个矩阵中
  + xformers_attention.cu: xformers实现的attention操作

+ Layer:
  + attention.cc: attention layer的具体实现，包括了Selective batching, Paged Attention, FlashAttention和Tensor Parallelism
    > attention操作的具体过程是首先计算QKV Gemm得到QKV matrix, 第二步是给qkv加上Bias并进行embedding，第三步是进行fusedattention计算(softmax(QK^T)V)，具体是首先保存ContextStage的Kvcache, 然后进行attention计算，如果是DecodingStage就直接进行Attention计算。最后一步是Output GEMM
  + ffn.cc: FFN的具体实现，包括了Tensor Parallelism的优化。具体包括了两个线性层，最后用nccl规约
  + gated_ffn.cc：实现了gatedFFN(llama2)

+ Model: (仅考虑llama2)
  + llama2op.cc: llama2OP的实现，
  + llama2op.h: llama2OP的定义
  + gpt_base.h: GPT的抽象类，注意Pytorch binding需要一个non-template class, 对于一个GPT抽象类来说，需要实现GptHyperParam, GptPagedAttnParam, GptParallelismParam三个参数类，loadWeight操作，initDummyWeight操作, forward操作，communicator初始化
  + gpt_hyper_param.h: 模型本身的超参数和一些配置，包括vocab_size(positional embedding之前的embedding矩阵维度)，max_positional_embeddings, hidden_size, num_layers, num_q_heads, num_kv_heads, head_dim,ffn_inter_dim. 配置：is_pre_layernorm, is_rotary_posi_embedding, is_gated_ffn, is_rmsnorm, is_attn_qkv_biased, is_attn_out_biased
  + gpt_pagedattn_param.h: pageattention的参数，block_size, max_num_block_per_req
  + gpt_parallelism_param.h: Parallelism的参数，tensor_para_size, tensor_para_rank, pipeline_para_size, pipeline_para_rank, layer_begin, layer_end(当前pipeline stage的layer range)
  + gptop_base.cc: 对于操作的包装类
  + gpt_weight.cc: 实现了单个GPT Layer的Weight的操作，包括init, loadWeight, initDummyWeight, weight包括attn_qkv_kernel/bias attn_out_kernel/bias,  attn_layernorm_weight/bias,ffn_fc1_weight/bias, 
  ffn_fc2_weight/bias,ffn_fc3_weight, final_layernorm_weiht/bias,(分为四部分，pre_layernorm_weights, attention_weights, attention_layernorm_weight, ffn_weight)
  + gpt.cc: 模型的具体实现，具体包括`inputBatchEmbedAndPosiEncode`，`selectOutputTokenBatched`，`forwardDecoder`,`forward(decoderxN+layernorm)`
    

+ Utils:
  + cublas_wrapper.cc: 对cublas GEMM的封装，包括`gemmStridedBatched`, `gemmBatched`和`gemm`, `getCublasComputeType`
  + cuda_utils.h：定义了常用的cuda宏
  + debug_utils.h：提供了assert函数，可以参考
  + torch_utils.h：定义了torch tensor相关的一些操作
  + nccl_utils.cc：定义了nccl的相关操作，没写过
  + py_block_migrations.cc：cudaIPCMemHandle_t和python
  + py_nccl.cc：torch function
  + py_swapping.cc：Perform swapping between GPU blocks and CPU blocks
  + st_datatypes.h：定义了一些原型

+ pybinding.cc: 定义自定义的PyTorch，没写过