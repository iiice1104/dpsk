# dpsk-R1

# 1.支持128K输入上下文长度

RoPE(Rotary Position Embedding)：旋转位置编码

外推性：模型对超出训练范围的数据的泛化性能，对于RoPE指的是处理比训练时更长的序列时的有效性

核心问题：比训练时更长的序列意味着模型需要处理未见过的位置索引
位置编码的重要性：模型通过位置编码来理解token的顺序关系，若外推性不足，模型难以捕捉长距离位置依赖或相对位置关系

如果外推性较差：则模型处理长文本生成任务时可能出现语段重复、逻辑混乱的情况

RoPE通过几何旋转将位置信息融入向量，旋转角度具有周期性，即使位置索引m超出范围，也能通过旋转将其处理到合理范围内，内积结果依赖相对位置差而不是绝对位置，从而外推性表现好(暂时不理解为什么RoPE的外推性好)

YaRN(Yet another RoPE extensioN):结合NTK(动态调整旋转角度基数)和位置插值（将位置索引m压缩成m/λ到训练范围），进一步扩展上下文窗口（模型单次处理时可以考虑的最大token数量，即捕捉信息的范围）
RoPE的外推性具有局限性，YaRN在RoPE的基础上采用多阶段插值策略和动态调整机制，具有更优的外推性，在超出长度的序列任务上表现优异

优化措施中的一些概念：  
高频维度：RoPE中旋转角度较大，旋转速度较快的维度，一般是隐藏层的低维部分，索引较小，擅长捕捉局部细节和近距离依赖关系  
低频维度：相应地，旋转角度较小，旋转较慢，维度索引较大，捕捉长距离依赖和全局结构  
波长：设旋转角度为θi，则旋转一周是2Π，波长λ是在维度i上旋转一周（2Π）需要的序列长度。例如当λ=100，则每100个token后改维度的位置编码会循环一次  
短波长（高频维度）：旋转角度大，波长短，适合编码局部位置  
长波长（低频维度）：旋转角度小，波长长，适合编码全局位置  

优化策略：根据波长λ与预训练长度的关系，将序列分为三个部分，较短波长即索引较小部分保留原始编码，中等波长部分插值，较长波长即索引较大部分完全插值

# 2.多头潜在注意力机制Mutile-Head-Latent-Attention MLA

传统的Transformer采用多头注意力机制MHA，但键值（KV）缓存限制推理速度，由此dpsk引入了MLA，性能更优且KV缓存大大减少

首先介绍多头注意力机制MHA:将输入序列通过多组独立的注意力计算处理，每组称为一个“头”，每个头关注不同的内容不同的依赖关系，所有头的结果合并输出为最终注意力输出

1.首先输入为X，经过三种不同的线性变换，即使用权重矩阵将输入向量X投影到不同子空间分别得到查询Q，键K，值V，按照头的数量h，在特征维度上分为h份，每一份的维度为d_k=d_models/h  
2.单个头注意力计算：分数矩阵=Q·K^T，防止点积结果太大导致softmax梯度消失，进行缩放，得到缩放分数=Q·K^T/√d_k，注意力权重=softmax(缩放分数），输出=注意力权重·V  
3.将所有头的输出拼接起来，拼接输出=[head1,head2,...headh],然后使用线性变换Wo将其融合，最终输出=拼接输出·Wo,映射到原始维度  

● 由于MHA的KV缓存太大，有提出多查询注意力机制MQA，分组查询注意力机制GQA，都是多个查询对应一个KV，虽然KV缓存减小了，但是性能也降低了  

接下来介绍MLA:MHA需要为每个注意力头缓存完整的KV矩阵，显存占用随着头数线性增长，MLA通过低秩联合压缩技术，将KV矩阵压缩为潜在向量，显著降低空间占用  
1.下投影：对于输入h，使用共享的降维矩阵W<sup>dkv</sup>，将原始高维特征映射到低维潜在空间，得C<sup>KV</sup>=W<sup>DKV</sup>h,同理C<sup>Q</sup>=W<sup>DKV</sup>h，缓存的是降维结果  
2.向上投影：后续推理使用时，需要将潜在向量升维，使用头特定的升维矩阵W<sup>kk</sup>、W<sup>vv</sup>等，k<sup>C</sup>=W<sup>kk</sup>C<sup>KV</sup>，例如计算分数矩阵时，Q·K^T=Q·(W<sup>kk</sup>C<sup>KV</sup>)^T  
3.与RoPE兼容：  
RoPE不是应用于上投影得到的q<sup>C</sup>，而是直接从C<sup>Q</sup>生成新的Q嵌入:q<sup>R</sup>=RoPE(W<sup>QR</sup>C<sup>Q</sup>)  
对于新的k嵌入，也不是应用于上投影后的K，与q不同，也不是下投影的K(C<sup>K</sup>)，而是由输入h生成：k<sup>R</sup>=RoPE(W<sup>KR</sup>h)  
4.计算注意力输出：q为{q<sup>C</sup>,q<sup>R</sup>}，k为{k<sup>C</sup>,k<sup>R</sup>}，v为{v<sup>C</sup>}，连接时Q和K的维数增加了，模型可以增加注意力头的数量或者调整每个头的维数来适应这个增加  
注意力的输出如下：  
![image](https://github.com/user-attachments/assets/aec69435-c56b-408c-88bb-d9910c676a15)

# 3.混合专家MOE

在学习MoE之前，需要先学习前馈神经网络FFN  
FFN位于多头注意力机制之后，存在于每一个编码器和解码器层中，由两个全连接层（线性变换）和中间的一个ReLU激活函数组成：FFN(x)=max(0,xW<sub>1</sub>+b<sub>1</sub>)W<sub>2</sub>+b<sub>2</sub>  
x：输入向量，为前一层的输出，维度为d_model;  
W<sub>1</sub>，b<sub>1</sub>：第一个全连接层的权重矩阵和偏置项，维度：d_model->d_ff;  
max(0,·)：ReLU激活函数，引入非线性，将所有负值置0；  
W<sub>2</sub>，b<sub>2</sub>：第二个全连接层的权重矩阵和偏置项，维度d_ff->d_model;(一般d_model为512，d_ff为2048)  
注意：FFN与注意力机制很大的不同，FFN是位置独立处理，不同位置不共享信息  

FFN的核心作用与功能：  
1.引入非线性表达能力：FFN的ReLU为模型提供了强大的非线性变换能力，如果没有FFN，那么Transformer将退化为一系列线性变换的组合，极大限制其表达能力；  
2.特征增强与维度变换：FFN通过“扩展-压缩”的维度变换实现特征的增强与精炼，升维时，在高维空间挖掘潜在的复杂模式，降维时，对高维信息进行筛选和整合，保留最有用的信息；  
3.与注意力机制互补：注意力层专注于不同位置（词与词)之间的关系，FFN层则专注于单个词内部特征的转换与增强；  
4.参数存储与知识保留：FFN占据了约2/3的总参数量，是模型寸尺知识和模式的主要场所。  

提问：为什么注意力层的softmax的非线性表达能力有限，而FFN的ReLU提供强大的非线性变换能力？  
softmax的非线性表达能力有限：softmax的输出是输入的单调函数，近似线性变换；softmax仅对注意力分数做归一化，不改变原始特征的空间维度。  
FFN引入非线性表达能力：全连接层FC的本质是线性变换，多个FC仍等价于单层线性变换，而ReLU（max(0,x)）通过分段非线性设计，正区间梯度为1，缓解梯度消失问题，负区间阶段归零，引入了稀疏性，且计算量小；此外FFN的扩展-压缩带来维度变换，在高维空间通过ReLU实现特征重组。  
ReLU的作用：1.在正区间的梯度恒为1，避免传统激活函数导致的梯度消失问题，尤其适合深层网络训练；2.稀疏激活特性：负输入输出0的特性使约50%的神经元在训练中被抑制，既能减少活跃神经元数量加速训练，还能迫使不同神经元学习差异化特征  

深度学习结构的黄金法则：线性变换提供可扩展性，非线性激活赋予模型深度表达能力。  

接下来学习MoE：  
每个MoE层有多个专家（Expert）组成，每个专家在结构上保持类似FFN的结构（两层FC以及中间的ReLU层），但通过动态路由机制选择性地激活部分专家。  

deepseek的MoE层的架构相比传统的MoE有两个特点：细粒度专家分割与共享隔离。  
1.细粒度专家分割：将专家进一步拆分成更小的子专家，每次激活一定数量的子专家，避免单一专家被迫学习混杂知识；  
2.共享专家隔离：固定保留部分专家作为共享专家，始终被激活，用于捕捉跨任务的通用知识，如语法、基础逻辑等，其余专家专注于领域特异性知识。  

如何计算激活哪几个专家？  
每个专家有一个质心向量e<sub>i</sub>,计算MoE的输入u<sub>t</sub>与每个e<sub>i</sub>之间的相似度：s<sub>i,t</sub> = Sigmoid(u<sub>t</sub><sup>T</sup>e<sub>i</sub>)，这个值决定每个专家与给定输入的相关程度，仅激活具有最高s<sub>i,t</sub>的Top-K专家进行处理，其余专家输出置0。  
最终的输出h<sub>t</sub>包含三部分：输入u<sub>t</sub>，共享专家的输出，被激活的Top-K专家的输出（需处理）。







