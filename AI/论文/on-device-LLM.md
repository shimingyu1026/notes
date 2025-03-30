# On-Device Language Models: A Comprehensive Review

## 大模型基础

### 大模型结构

* Traditional text-based LLMs: Transformer,widely used to process sequential data。由encoder和decoder组成，mainly use decoder-only architecture：GPT (Generative Pre-trained Transformer),GPT模型使用标准的自注意力机制。

    MOE：混合专家模型。用于预训练模型。两个部分：一个包含多个“专家”的稀疏MoE层，每个“专家”本身就是一个独立的神经网络；以及一个门控网络或路由：这个组件用于确定哪些令牌被发送到哪个专家模型进行处理。

* 多模态：A）使用标准的交叉注意层对模型内部层的多模态输入进行深度融合；B）使用定制设计的层在模型的内部层中执行多模态输入的深度融合；C）在模型的输入阶段进行多模态输入的早期融合，使用特定于模态的编码器；D）在输入阶段进行早期融合，但使用分词技术（例如分词器）来处理模态

### On-Device 大模型的训练

用于解决计算和内存资源受限的问题

* Quantization-aware scaling（感知量化缩放）：自动调整不同比特精度张量的梯度大小，解决量化图中不同比特宽度张量梯度不一致的问题。
* Sparse update（稀疏更新）：选择性地更新网络中部分层的权重，跳过不那么重要的层和子张量的梯度计算。
* Tiny Training Engine (TTE)：在反向图中包含冗余节点，例如冻结权重的梯度节点，并重新排序操作以实现就地更新。
* Contribution analysis：确定哪些参数（权重/偏置）对下游准确率贡献最大，以便在有限的内存预算下选择哪些层或张量的部分应该被更新。

### On-Device LLM模型性能指标

TTFT (Time-to-First-Token)用于衡量模型的延迟。

Inference speed（推理速度）：影响用户体验。

RAM使用大小

存储空间与功耗大小

## On-Device LLMs 的高效架构

### 设计方法与创新

1）参数共享

2）模块化架构：单独或者可以并行处理

3）紧凑表示：使用量化和权重剪枝减少内存占用

### 模型压缩与参数共享

AWQ（引文）：保护一小部分关键权重（0.1%-1%），量化其他权重，不需要反向传播或重建。减小模型架构

MobileLLM：没看懂，反正加速就对了。架构优化和权重共享

### 协作和层次化模型

协作和分层模型方法通过分配计算负载并利用具有不同能力的多个模型，突破有限的内存和计算能力的限制。

* EdgeShard框架（引文）：将模型划分为较小的片段。
* LLMCad（引文）：新颖的推理引擎；内存驻留小模型+大模型
* WDMoE（引文）

### 内存与计算的高效性
