# SwiftTransformer 

## 计划模块

暂定为(SeeleTransformer)，具体要实现的模块包括四部分，Cuda Base Operator, Layer implementation, Model implementation,  Utils(Pytorch interface, weight convert bash)

### Model Implementation

暂定首先实现GPT模型，推理流程分为如下几个过程：初始化通信库，加载权重，推理，解码结果

+ 模型超参数：
  + GPTHyperParam: 
    + vocab_size
    + max_position_embeddings
    + hidden_size
    + num_layers
    + num_q_heads
    + num_kv_heads
    + head_dim
    + ffn_inter_dim
    + is_rotary_posi_embedding（for rope alternative）
    + is_rmsnorm(for rmsnorm)
    + is_attn_qkv_biased
    + is_attn_out_biased
    + ffn_activation_type
  + GptParallelismParam
    + tensor_para_size
    + tensor_para_rank
    + pipeline_para_size
    + pipeline_para_rank
      + layer_begin, layer_end, local_layer_num
  + GptPagedAttnParam
    + block_size
    + max_num_block_per_req
+ 模型类定义:
  + weight_params
  + communicator_params
  + GPU buffers for requests:
    + inputs: decoder_input/output, d_input_lens, d_sum_prev_input_lens?
    + input embedding: d_token_ids, d_position_ids
    + input indexing: ith_context_req_req_index, ith_context_req_token_index, ith_decoding_req_req_index, ith_decoding_req_token_index
    + layer internal computation: qkv_buf, attn_out_buf, ffn_inter_buf_1, ffn_inter_buf_2, context_stage_kernel_m_buf, context_stage_kernel_l_buf
    + forwardDecoder: attention_out
    + output projection: output_projection_last_tokens_buf, *output_projection_tokens_buf*, output_projection_result_buf

+ 通信库初始化：暂定实现tp和pp，需要初始化MPI，NCCL，其中MPI划分为每个pp和tp通信组(暂时不用PP)，NCCL分为TP和PP 通信器(暂时不用PP)
+ 加载权重：
  + 权重转换脚本：从torch.load()转换为SwiftTransformer的格式
    + 转换逻辑：
      + 首先从input中读取文件，如果权重未被分片，直接返回模型权重(state_dict)，否则进行处理，tensorMergeFunc是一个在不同维度上对不同权重进行拼接的函数，不同类型的权重需要在不同的维度上进行cat来拼接成一个state_dict。具体过程是，首先是resharedWeight函数，首先将所有相同key(weight)的tensor放到一起得到`sharded_tensors: dict[str, List[torch.Tensor]]`，然后调用`tensor_merge_func`将这个每个list进行拼接，得到`dict[str, torch.Tensor]`，就是转换权重前所用的`state_dict`。
      + 然后进行具体的转换逻辑，对于llama来说，就是从state_dict中通过正则得到模型层数，然后对state_dict中wqkv拼接到一起，并将wo进行转置，最后将tensor_dict分割并保存。在进行分割和保存时，以llama2为例，定义一个命名池，对于不同类型的权重，将其在不同维度上拆成8份存到文件中，而对于shared_tensor，则直接存到文件中
  + 具体加载权重：
    + Embedding:
      + embed_token_weight
      + embed_positioins_weight
    + Transformer
      + attn_qkv_weight/bias
      + attn_out_weight/bias
      + attn_layernorm_weight
      + attn_layernorm_bias(rmsnorm不要)
      + ffn1_weight/bias
      + ffn2_weight/bias
      + ffn3_weight?(gated_ffn需要)
      + final_layernorm_weight/bias(GeluFFN前的layernorm)
    + Final:
      + final_layernorm_weight/bias
      + final_logitgemm_weight
  + 加载逻辑：首先根据是否有embedding_layer加载embedding weights，然后根据当前layer范围加载layer权重，最后根据是否是pp最后一阶段加载final_layernorm权重
  + 其中output_weight，ffn1_weight, ffn2_weight, ffn3_weight需要进行tp
  + 初始化权重
+ 推理过程：
  + 分配k/v cache内存：采用PagedAttention，一开始就先在GPU上分配所有的Block, 记录了每个Request=分配的block数量，然后初始化block_table, 这个block_table存储了每个request所分配的块的索引(从0-max_num_block_per_req) 
  + 为decoding stage的数组分配内存，保存了output of the last token
  + Context Stage， 执行gpt.forward
  + Decoding Stage: 在Decoding stage中，input只用包括所有unfinished request的last token，判断每个request(input tokens + output tokens)是否需要分配新的Block，用一个二维数组记录每个Request的下一轮的input tokens。但是由于request可能在本轮中结束，所以kvcache的位置需要移动
    + 一次forward的实现：首先初始化三个host/device数组，h_input_lens, h_is_context_stage, h_sum_prev_input_lens和num_tokens。然后因为PP，所以对第一个Stage进行特殊处理，进行EmbedingandPositionEncode, 同理最后一个Stage需要特殊处理，需要调用layernorm。中间调用forwardDecoder，首先从上一个阶段收取数据，并将数据广播给tp内所有的设备。
    + forwardDecoder的实现：分配内存后，调用之前写的attention和layernorm, addbias
+ 解码结果：
  + VocabDecoder

### Layer implementation

+ attention.cc
+ ffn.cc
+ gated_ffn.cc

### Operator

+ addbias.cu
+ count_nan.cu
+ embedding.cu
+ findmax.cu
+ fused_activ_multiply.cu
+ fused_addbias_activ.cu
+ fused_context_stage_attention.cu
+ fused_decoding_stage_attention.cu
+ fused_decoding_stage_attention_mha.cu
+ gather_last_tokens.cu
+ kvcache_mgmt.cu
+ layernorm.cu
+ reduction.cu
+ rmsnorm.cu
+ rotary_posi_embedding.cu
+ unfused_attention.cu
+ xformers_attention.cu

### Utils

+ Cublas Wrapper

## 框架代码分析

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



