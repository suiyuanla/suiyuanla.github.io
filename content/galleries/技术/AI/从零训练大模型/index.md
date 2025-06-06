---
title: LLM From Scratch（正在施工🚧）
date: 2025-04-17T14:22:30+08:00
draft: false
categories:
  - AI
tags:
  - LLM
series:
  - LLM From Scratch
lastmod: 2025-06-06T09:04:50.143Z
---
## 简介

本文尝试从零训练一个大模型，以学习大模型相关的训练流程和知识。

## 前置知识

### RMS Norm

1. 为什么要有 Norm？\
   规范化（Norm）本质是将非标准数据统一为指定格式的过程，个人理解是将数据重新映射到某个区间内。\
   这么做的一个原因是，随着网络深度的增加，各层的特征值会逐渐趋近激活函数的上下限附近，导致激活函数饱和，进而导致梯度消失。归一化（规范化）可以使特征值的分布重新回到激活函数对输入敏感的部分，从而避免梯度消失、加快收敛速度。\
   对训练数据和测试数据进行规范化可以防止不同数据分布对模型训练的影响，提高模型的泛化能力。

2. Batch Norm 和 Layer Norm

   > 此处参考李沐 Transformer 视频:【Transformer 论文逐段精读【论文精读】】[精准空降到 26:05](https://www.bilibili.com/video/BV1pu411o7BE/?share_source=copy_web\&vd_source=4b96d541e7b734eb933e137270be06ab\&t=1565)

   Batch Norm 是对一个 batch 内的数据按特征进行规范化：\
   ![Batch Norm](/obsidian/%E6%8A%80%E6%9C%AF/AI/%E4%BB%8E%E9%9B%B6%E8%AE%AD%E7%BB%83%E5%A4%A7%E6%A8%A1%E5%9E%8B/images/BatchNorm.png)\
   Layer Norm 则是对一个样本内所有特征进行规范化：\
   ![Layer Norm](/obsidian/%E6%8A%80%E6%9C%AF/AI/%E4%BB%8E%E9%9B%B6%E8%AE%AD%E7%BB%83%E5%A4%A7%E6%A8%A1%E5%9E%8B/images/LayerNorm.png)

   在图中可以直观看到，对于文本生成任务，Batch Norm 当不同 Batch 间的序列长度差异较大时，Batch Norm 由于空白的存在，导致算出的均值和方差波动较大。Batch Norm 在训练时使用当前 Batch 的均值和方差，在推理时使用整个训练集的均值和方差。

   在推理时当一个 Batch 内有一个过长的序列，我们之前训练得到的均值和方差可能就不能很好的应用在这个没见过的长序列上，导致预测效果不好。而 Layer Norm 由于是每个样本内部做 Norm 相对来说没有这些问题，能够更好的应用在序列生成任务上。

   Batch Norm 在计算机视觉领域更加有效，因为它消除了不同特征之间的大小关系，保留了不同样本间的大小关系。对于文本生成任务，Layer Norm 更加有效，因为单个样本的不同特征是词语随时间的变化，而且样本内的特征关系非常密切，它能保留样本内特征的大小关系，并且和它的计算和 batch 无关，能更好的应用在序列生成任务上。

3. 为什么要有 RMS Norm？\
   RMS Norm 提出的动机是 LayerNorm 的运算量比较大，因此对 LayerNorm 做了运算上的简化。

   原有的 LayerNorm 计算公式：

   1. 计算均值$\mu$和方差$\sigma^2$：
      <div>
      $$
       \mu_L=\frac{1}{d}\sum_{i=1}^d x_{i}
      $$
      </div>
   2. 对每个元素进行归一化：
      <div>
      $$
        \hat{x}_i=\frac{x_i-\mu_L}{\sqrt{\sigma_L^2+\epsilon}}
      $$
      </div>
   3. 对归一化后的结果进行缩放和平移，$\gamma$和$\beta$是可学习的参数：
      <div>
      $$
        y_i=\gamma\hat{x}_i+\beta
      $$
      </div>

   RMS Norm 的计算公式：\
   对于输入向量$x=(x_1,x_2,...,x_d)$，计算其均方根(RMS)：

   1. 计算输入向量的均方根(RMS)：

      <div>
      $$
        RMS(x) = \sqrt{\frac{1}{d}\sum_{i=1}^d x_i^2}
      $$
      </div>

   2. 对每个元素$x_i$进行归一化：

      <div>
      $$
        \hat{x}_i=\frac{x_i}{RMS(x)+\epsilon}
      $$
      </div>

   3. 对归一化后的结果进行缩放，$\gamma$是可学习的参数：
      <div>
      $$
        y_i=\gamma\hat{x}_i
      $$
      </div>

4. 为什么 RMS Norm 对 Layer Norm 简化后效果依然很好？

   1. Transformer 架构中使用了残差连接，直接将输入传递到输出端，天然保留了均值信息，使得 RMSNorm 不必显式计算均值进行中心化也可以通过残差路径维持分布特征。
   2. 对于自然语言处理这类高位数据来说，向量的方向比绝对位置更能表征语义信息，$\gamma$通过缩放向量模长直接影响特征方向，而$\beta$调整 layerNorm 的位置偏移，相比之下显得没有那么重要。
   3. 实验结果也表明，RMSNorm 在多项任务中性能与 LayerNorm 相当，甚至更优。这表明平移参数在某些场景下可能被过参数化，或其对模型性能的影响被模型中其他机制补偿。

5. RMSNorm 的实现

   > 此处参照：https://github.com/jingyaogong/minimind

   ```py
    import torch
    import torch.nn as nn

    class RMSNorm(torch.nn.Module):
        def __init__(self, dim: int, eps: float = 1e-6):
            super().__init__()
            self.eps = eps  # 小常数
            self.weight = nn.Parameter(torch.ones(dim)) # γ矩阵

        # x * 1/√(Σ(x^2) + ε)
        def _norm(self, x):
            return x * torch.rsqrt(x.pow(2).mean(-1, keepdim=True) + self.eps)

        # γ * _norm(x)
        def forward(self, x):
            return self.weight * self._norm(x.float()).type_as(x)
   ```

### 位置编码

位置编码是大模型理解输入 Token 不同位置信息的关键，如果没有位置编码，那么“你好呀”和“呀你好”在大模型的视角看将没有区别，这一点是因为 Attention 的计算公式里不同 Token 计算时并没有设计位置信息的参与，因此需要额外的位置信息附加在 Token 的向量上。

1. Attention 的位置编码：

   **Attention is all you need**这篇论文在刚提出 Transformer 的时候提出了一个位置编码的方案：

   <div>
   $$
   PE(pos,2i) =sin(\frac{pos}{10000^{2i/d}}) \\
   PE(pos,2i+1) =cos(\frac{pos}{10000^{2i/d}})
   $$
   </div>

   一个具体的例子,X 每一行是一个词的词向量：

   <div>
   $$
   X = \begin{bmatrix}
   w_{00} & w_{01} & w_{02} & w_{03} \\
   w_{10} & w_{11} & w_{12} & w_{13} \\
   w_{20} & w_{21} & w_{22} & w_{23}
   \end{bmatrix}
   $$
   </div>

   <div>
   $$
   PE = \begin{bmatrix}
   \sin\left(\frac{0}{10000^{(2*0/4)}}\right) & \cos\left(\frac{0}{10000^{(2*0/4)}}\right) & \sin\left(\frac{0}{10000^{(2*1/4)}}\right) & \cos\left(\frac{0}{10000^{(2*1/4)}}\right) \\
   \sin\left(\frac{1}{10000^{(2*0/4)}}\right) & \cos\left(\frac{1}{10000^{(2*0/4)}}\right) & \sin\left(\frac{1}{10000^{(2*1/4)}}\right) & \cos\left(\frac{1}{10000^{(2*1/4)}}\right) \\
   \sin\left(\frac{2}{10000^{(2*0/4)}}\right) & \cos\left(\frac{2}{10000^{(2*0/4)}}\right) & \sin\left(\frac{2}{10000^{(2*1/4)}}\right) & \cos\left(\frac{2}{10000^{(2*1/4)}}\right)
   \end{bmatrix}
   $$
   </div>

   可以看到，绝对位置编码的计算相对来说简单，且位置编码可以提前计算好，在实际附加时，只需要将词向量与位置编码相加即可。

   绝对位置编码的一些问题：

   * 假设在训练时都用的是短序列，在推理时出现了长序列的推理，由于没有见过长序列部分的位置编码，会导致模型性能的大幅下降。
   * 对于句子中的一个词来说，它的语义和出现在句子中绝对位置的关系不大，而和词中几个 token 的相对位置关系较大，但是不同位置的相同词的 Token 间关系，在绝对位置编码中并不一致。

2. RoPE 位置编码\
   为了折中绝对位置编码与相对位置编码的需求，融合两者的优点，以及提高模型在长序列上的泛化能力，提出了 RoPE 位置编码。

   1. RoPE 的核心思想：\
      在注意力计算时，计算第 m 个词和第 n 个词的注意力为：

      <div>
      $$
         q_m^T \cdot k_n
      $$
      </div>

      我们希望能够对 q，k 进行绝对位置(m,n)编码，同时当它们进行注意力计算后的结构能够反应(m-n)的关系，即我们希望($f_q，f_k$是对 q，k 进行位置编码的函数)：

      <div>
      $$
         f_q(x_m,m) \cdot f_k(x_n,n) = g(x_m,x_n,m-n)
      $$
      </div>

      跳过推导过程，我们直接看结论，假设 q，k 都是 2 维的向量，则：

      <div>
      $$
         f_{q,k}(x_m,m) = \begin{pmatrix}
            \cos{m\theta} & - \sin{m\theta} \\
            \sin{m\theta} & \cos{m\theta}
         \end{pmatrix}
         \begin{pmatrix}
         W_{q,k}^{(11)} & W_{q,k}^{(12)} \\
         W_{q,k}^{(21)} & W_{q,k}^{(22)}
         \end{pmatrix}
         \begin{pmatrix}
         x_m^{(1)} \\
         x_m^{(2)}
         \end{pmatrix}
      $$
      </div>

      旋转矩阵就是：

      <div>
      $$
      R(\theta) = \begin{pmatrix}
            \cos{\theta} & - \sin{\theta} \\
            \sin{\theta} & \cos{\theta}
         \end{pmatrix}
      $$
      </div>

      这个旋转矩阵有一些很好的性质：

      <div>
      $$
      R(\alpha)^T = R(-\alpha)\\
      R(\alpha) \cdot R(\beta)= R(\alpha + \beta)
      $$
      </div>

      因此，如果我们对$\vec{q}_m、\vec{k}_n$进行位置编码再进行注意力运算，得到的结果就能够很好的反应 q 和 k 的相对位置关系：

      <div>
      $$
      (R(m\theta)\vec{q}_m)^T \cdot R(n\theta)\vec{k}_n = \vec{q}_m^T R(-m\theta)R(n\theta)\vec{k}_n = \vec{q}_m^T R((n-m)\theta)\vec{k}_n
      $$
      </div>

   2. RoPE 的通用形式：\
      在上面，我们介绍了 RoPE 在词向量是 2 维的时候的形式，如果是多维的情况下该如何推广呢？\
      当 q，k 是多维时，我们可以对其维度进行两两分组，分别应用旋转位置编码，具体的位置编码矩阵就是下面这样：

      <div>
      $$
      R_{\Theta, m}^d = \begin{pmatrix}
      \cos m\theta_1 & -\sin m\theta_1 & 0 & 0 & \cdots & 0 & 0 \\
      \sin m\theta_1 & \cos m\theta_1 & 0 & 0 & \cdots & 0 & 0 \\
      0 & 0 & \cos m\theta_2 & -\sin m\theta_2 & \cdots & 0 & 0 \\
      0 & 0 & \sin m\theta_2 & \cos m\theta_2 & \cdots & 0 & 0 \\
      \vdots & \vdots & \vdots & \vdots & \ddots & \vdots & \vdots \\
      0 & 0 & 0 & 0 & \cdots & \cos m\theta_{d/2} & -\sin m\theta_{d/2} \\
      0 & 0 & 0 & 0 & \cdots & \sin m\theta_{d/2} & \cos m\theta_{d/2}
      \end{pmatrix}
      $$
      </div>

      这样的话相当于对词向量（q，k）的维度进行了两两分组，分别应用旋转位置编码。实际计算时，由$R_{\Theta, m}^d \cdot x$的结果如下：

      <div>
      $$
      \begin{equation}
      R_{\Theta,m}^d \cdot x = \begin{pmatrix}
      x_1 \cos m\theta_1 - x_2 \sin m\theta_1 \\
      x_1 \sin m\theta_1 + x_2 \cos m\theta_1 \\
      x_3 \cos m\theta_2 - x_4 \sin m\theta_2 \\
      x_3 \sin m\theta_2 + x_4 \cos m\theta_2 \\
      \vdots \\
      x_{d-1} \cos m\theta_{d/2} - x_d \sin m\theta_{d/2} \\
      x_{d-1} \sin m\theta_{d/2} + x_d \cos m\theta_{d/2}
      \end{pmatrix}
      \end{equation}
      $$
      </div>

      在实际计算时，$R_{\Theta, m}^d \cdot x$ 等价于如下的计算，避免了矩阵中的 0 带来的额外计算开销：

      <div>
      $$
         R_{\Theta, m}^d \cdot x =
         \begin{pmatrix}
         x_1 \\
         x_2 \\
         x_3 \\
         x_4 \\
         \vdots \\
         x_{d-1} \\
         x_d
         \end{pmatrix}
         \odot
         \begin{pmatrix}
         \cos{m\theta_1} \\
         \cos{m\theta_2} \\
         \cos{m\theta_3} \\
         \cos{m\theta_4} \\
         \vdots \\
         \cos{m\theta_{d-1}} \\
         \cos{m\theta_d}
         \end{pmatrix}
         +
         \begin{pmatrix}
         -x_2 \\
         x_1 \\
         -x_4 \\
         x_3 \\
         \vdots \\
         -x_d \\
         x_{d-1}
         \end{pmatrix}
         \odot
         \begin{pmatrix}
         \sin{m\theta_1} \\
         \sin{m\theta_2} \\
         \sin{m\theta_3} \\
         \sin{m\theta_4} \\
         \vdots \\
         \sin{m\theta_{d-1}} \\
         \sin{m\theta_d}
         \end{pmatrix}
      $$
      </div>

   3. RoPE 计算实例

   ```txt
   θ = 1e6 , dim = 4, end = 3
   词向量 x1 = [1,2,3,4]
         x2 = [5,6,7,8]
         x3 = [9,10,11,12]
   预计算位置编码
      θ1 = (1e6)^0/4 = 1
      θ2 = (1e6)^2/4 = 0.001
       t = [0,1,2]
      θ * t = [0,    0] // 外积
              [1,0.001]
              [2,0.002]
        pos => [cos0+sin0i, cos0.000+sin0.000i]
               [cos1+sin1i, cos0.001+sin0.001i]
               [cos2+sin2i, cos0.002+sin0.002i]

      实际位置编码时：
      x1 => [1+2i,3+4i]
      x2 => [5+6i,7+8i]
      x3 => [9+10i,11+12i]
      实际位置编码运算：
      x1 * pos[1] = [(1+2i)(cos0+sin0i), (3+4i)(cos0.000+sin0.000i)] = [1+2i,3+4i]
      x2 * pos[2] = [(5+6i)(cos1+sin1i), (7+8i)(cos0.001+sin0.001i)] = [(5cos1-6sin1)+(5sin1+6cos1)i,...]
      x3 * pos[3] = ...
      结果：
      x1_out = [1,2,3,4]
      x2_out = [5cos1-6sin1,5sin1+6cos1,7cos0.001-8sin0.001,7sin0.001+8cos0.001]
      x3_out = [...]
   ```

   4. RoPE 具体实现（来自 minimind）

   ```py
   import torch


   def precompute_pos_cis(dim: int, end: int = int(32 * 1024), theta: float = 1e6):
       freqs = 1.0 / (theta ** (torch.arange(0, dim, 2)
                      [: (dim // 2)].float() / dim))
       t = torch.arange(end, device=freqs.device)
       freqs = torch.outer(t, freqs).float()
       pos_cis = torch.polar(torch.ones_like(freqs), freqs)
       return pos_cis


   def apply_rotary_emb(xq, xk, pos_cis):
       def unite_shape(pos_cis, x):
           ndim = x.ndim
           assert 0 <= 1 < ndim
           assert pos_cis.shape == (x.shape[1], x.shape[-1])
           shape = [d if i == 1 or i == ndim -
                    1 else 1 for i, d in enumerate(x.shape)]
           return pos_cis.view(*shape)

       xq_ = torch.view_as_complex(xq.float().reshape(*xq.shape[:-1], -1, 2))
       xk_ = torch.view_as_complex(xk.float().reshape(*xk.shape[:-1], -1, 2))
       pos_cis = unite_shape(pos_cis, xq_)
       xq_out = torch.view_as_real(xq_ * pos_cis).flatten(3)
       xk_out = torch.view_as_real(xk_ * pos_cis).flatten(3)
       return xq_out.type_as(xq), xk_out.type_as(xk)
   ```

### 注意力机制

注意力机制是 Transformer 模型成功的关键，注意力机制使得大语言模型能够计算当前位置 Token 与之前所有 token 的注意力分数，从而能够更好的理解到对话内容的信息。

1. 公式
   <div>
   $$
   Attention(Q,K,V)=softmax(\frac{Q K^T}{\sqrt{d_k}})V
   $$
   </div>

2. 例子\
   假设输入一个序列：“Llama is a large model…”，对应的词向量矩阵为 X：

   <div>
   $$
   X = \begin{aligned}
   \text{Llama} & \quad \begin{bmatrix} \vec{x}_1^T \end{bmatrix} \\
   \text{is} & \quad \begin{bmatrix} \vec{x}_2^T \end{bmatrix} \\
   \text{a} & \quad \begin{bmatrix} \vec{x}_3^T \end{bmatrix} \\
   \text{...} & \quad \begin{bmatrix} \vec{x}_{...}^T \end{bmatrix} \\
   \end{aligned}
   $$
   </div>

   Q、K、V 分别为：

   <div>
   $$
      \begin{aligned}
   Q = X \cdot W^Q = \begin{bmatrix}
   \vec{x}_1^T W_Q \\
   \vec{x}_2^T W_Q \\
   \vec{x}_3^T W_Q \\
   \vec{x}_{...}^T W_Q \\
   \end{bmatrix}
   =\begin{bmatrix}
   \vec{Q}_1 \\
   \vec{Q}_2 \\
   \vec{Q}_3 \\
   \vec{Q}_{...} \\
   \end{bmatrix} && \\
   K=X \cdot W^K  = \begin{bmatrix}
   \vec{x}_1^T W_K \\
   \vec{x}_2^T W_K \\
   \vec{x}_3^T W_K \\
   \vec{x}_{...}^T W_K \\
   \end{bmatrix}
   =\begin{bmatrix}
   \vec{K}_1 \\
   \vec{K}_2 \\
   \vec{K}_3 \\
   \vec{K}_{...} \\
   \end{bmatrix} && \\
   V=X \cdot W^V  = \begin{bmatrix}
   \vec{x}_1^T W_V \\
   \vec{x}_2^T W_V \\
   \vec{x}_3^T W_V \\
   \vec{x}_{...}^T W_V \\
   \end{bmatrix}
   =\begin{bmatrix}
   \vec{V}_1 \\
   \vec{V}_2 \\
   \vec{V}_3 \\
   \vec{V}_{...} \\
   \end{bmatrix} \\
   \end{aligned}
   $$
   </div>

   计算 QK^T 矩阵：

   <div>
   $$
   \begin{aligned}
   K^T &= \begin{bmatrix}
   W_K^T \vec{x}_1 &
   W_K^T \vec{x}_2 &
   W_K^T \vec{x}_3 &
   W_K^T \vec{x}_{...} &
   \end{bmatrix} = \begin{bmatrix}
   \vec{K}_1^T &
   \vec{K}_2^T &
   \vec{K}_3^T &
   \vec{K}_{...}^T &
   \end{bmatrix} \\
   Q K^T &= XW_Q \cdot W_K^T X^T \\
   &=\begin{bmatrix}
   \vec{Q}_1 \vec{K}_1^T & \vec{Q}_1 \vec{K}_2^T & \vec{Q}_1 \vec{K}_3^T & \vec{Q}_1 \vec{K}_{...}^T \\
   \vec{Q}_2 \vec{K}_1^T & \vec{Q}_2 \vec{K}_2^T & \vec{Q}_2 \vec{K}_3^T & \vec{Q}_2 \vec{K}_{...}^T \\
   \vec{Q}_3 \vec{K}_1^T & \vec{Q}_3 \vec{K}_2^T & \vec{Q}_3 \vec{K}_3^T & \vec{Q}_3 \vec{K}_{...}^T \\
   \vec{Q}_{...} \vec{K}_1^T & \vec{Q}_{...} \vec{K}_2^T & \vec{Q}_{...} \vec{K}_3^T & \vec{Q}_{...} \vec{K}_{...}^T \\
   \end{bmatrix} \\
   \end{aligned}
   $$
   </div>

   正常的大模型是 Decoder Only 的大模型，Decoder 的自注意力机制是 Masked，原因是大模型在推理时不知道下一个字，所以计算注意力时只能与前面已知的 Token 进行计算。对上面的计算结果进行 Mask，得到的是下三角矩阵：

   <div>
   $$
   Q K^T = \begin{bmatrix}
   \vec{Q}_1 \vec{K}_1^T \\
   \vec{Q}_2 \vec{K}_1^T & \vec{Q}_2 \vec{K}_2^T \\
   \vec{Q}_3 \vec{K}_1^T & \vec{Q}_3 \vec{K}_2^T & \vec{Q}_3 \vec{K}_3^T  \\
   \vec{Q}_{...} \vec{K}_1^T & \vec{Q}_{...} \vec{K}_2^T & \vec{Q}_{...} \vec{K}_3^T & \vec{Q}_{...} \vec{K}_{...}^T \\
   \end{bmatrix}
   $$
   </div>

   计算完结果后，再进行 softmax，得到概率：

   <div>
   $$
   softmax(\frac{Q K^T}{d_k}) =
   \begin{bmatrix}
   p_{11} &  \\
   p_{21} & p_{22}  \\
   p_{31} & p_{32} & p_{33}  \\
   p_{{...}1} & p_{{...}2} & p_{{...}3} & p_{{...}{...}} \\
   \end{bmatrix}
   $$
   </div>

   注意力机制的完整结果为：

   <div>
   $$
   Attention(Q,K,V) = softmax(\frac{Q K^T}{d_k})V \\
   = \begin{bmatrix}
   p_{11} \vec{V}_1 \\
   p_{21} \vec{V}_1 + p_{22} \vec{V}_2  \\
   p_{31} \vec{V}_1 + p_{32} \vec{V}_2  + p_{33} \vec{V}_3  \\
   p_{{...}1} \vec{V}_1 + p_{{...}2} \vec{V}_2  + p_{{...}3} \vec{V}_3  + p_{{...}{...}} \vec{V}_{...} \\
   \end{bmatrix}  \\
   = \begin{bmatrix}
   \vec{R}_1 \\
   \vec{R}_2 \\
   \vec{R}_3 \\
   \vec{R}_{...} \\
   \end{bmatrix}
   $$
   </div>

3. 注意力机制的一些思考
   * 注意力机制形式上是每个 Token 与之前的 Token 进行注意力计算求加权平均，在数学上本质是算相关性，点积反映的是当前 Token 与前文 Token 的相关性。Q，K，V 是当前 Token 的不同表现形式。注意力计算后的结果是我们从词向量中提取出的更关注的信息（与下一个 token 更相关的信息，通过训练的权重矩阵提取得出）
   * 为什么 Tranformer 能够通过已知的句子预测下一个 Token？\
     我的理解是，当我们说一句话的时候，由于语言本身是因果相关的，前文的句子已经隐含了接下来可能说的词，因此通过注意力计算，我们能够计算出接下来可能的词的概率，以此来递推来生成句子。

4. 代码实现

   ```py
   # 此函数直接复制自minimind的repeat函数
   def repeat_kv(x: torch.Tensor, n_rep: int) -> torch.Tensor:
       """torch.repeat_interleave(x, dim=2, repeats=n_rep)"""
       bs, slen, n_kv_heads, head_dim = x.shape
       if n_rep == 1:
           return x
       return (
           x[:, :, :, None, :]
           .expand(bs, slen, n_kv_heads, n_rep, head_dim)
           .reshape(bs, slen, n_kv_heads * n_rep, head_dim)
       )

   # 注意力机制的具体实现（分组注意力机制GQA）
   class MyAttention(nn.Module):
       def __init__(self, dim: int, n_heads: int = 8, n_kv_heads: int = 4, max_seq_len: int = 1024, dropout: float = 0.1):
           super().__init__()

           assert (dim % n_heads == 0)
           assert (n_heads % n_kv_heads == 0)

           self.dim = dim                      # 隐层向量维度
           self.head_dim = dim // n_heads      # 单头的向量维度
           self.n_heads = n_heads              # 注意力头数量
           self.n_kv_heads = n_kv_heads            # 注意力头分组数量
           self.n_rep = n_heads // n_kv_heads    # kv重复次数
           self.wq = nn.Linear(self.dim, self.head_dim * self.n_heads, bias=False)
           self.wk = nn.Linear(self.dim, self.head_dim *
                               self.n_kv_heads, bias=False)
           self.wv = nn.Linear(self.dim, self.head_dim *
                               self.n_kv_heads, bias=False)
           self.wo = nn.Linear(self.n_heads * self.head_dim, self.dim, bias=False)

           self.attn_dropout = nn.Dropout(dropout)
           self.resid_dropout = nn.Dropout(dropout)

           mask = torch.full((1, 1, max_seq_len, max_seq_len), float("-inf"))
           mask = torch.triu(mask, diagonal=1)

           self.register_buffer("mask", mask, persistent=False)

       def forward(self, x: torch.Tensor, pos_cis: torch.Tensor, past_kv: Optional[Tuple[torch.Tensor, torch.Tensor]] = None) -> torch.Tensor:
           batch_size, seq_len, _ = x.size()
           # 计算Q、K、V
           xq, xk, xv = self.wq(x), self.wk(x), self.wv(x)
           xq = xq.view(batch_size, seq_len, self.n_heads, self.head_dim)
           xk = xk.view(batch_size, seq_len, self.n_kv_heads, self.head_dim)
           xv = xv.view(batch_size, seq_len, self.n_kv_heads, self.head_dim)
           # 应用旋转位置编码
           xq,xk = apply_rotary_emb(xq, xk, pos_cis)
           # kv cache
           if past_kv is not None:
               xk = torch.cat([past_kv[0], xk], dim=1)
               xv = torch.cat([past_kv[1], xv], dim=1)
           kv = (xk, xv)
           # kv重复
           xq, xk, xv = (
               xq.transpose(1, 2),
               repeat_kv(xk, self.n_rep).transpose(1, 2),
               repeat_kv(xv, self.n_rep).transpose(1, 2),
           )
           # 计算注意力分数
           scores = (xq @ xk.transpose(-2, -1)) / math.sqrt(self.head_dim)
           scores += self.mask[:, :, :seq_len, :seq_len]
           scores = F.softmax(scores.float(), dim=-1).type_as(xq)
           scores = self.attn_dropout(scores)
           # 计算注意力结果
           output = scores @ xv
           output = output.transpose(1, 2).reshape(batch_size, seq_len, -1)
           output = self.resid_dropout(self.wo(output))
           return output, kv
   ```
