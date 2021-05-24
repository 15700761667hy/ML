
### Transformer源代码解释之PyTorch篇

**章节**

- [词嵌入](#embed)
- [位置编码](#pos)
- [多头注意力](#multihead)
- [残差相连](#add&norm)
- [总结](#conclusions)
- [参考文献](#references)

![](https://github.com/sherlcok314159/ML/blob/main/nlp/Images/transformer.png)

**<div id='embed'>词嵌入</div>**

Transformer本质上是一种Encoder，以翻译任务为例，原始数据集是以两种语言组成一行的，在应用时，应是Encoder输入源语言序列，Decoder里面输入需要被转换的语言序列（训练时）。一个文本常有许多序列组成，常见操作为将序列进行一些预处理（如词切分等）变成列表，一个序列的列表的元素通常为词表中不可切分的最小词，整个文本就是一个大列表，元素为一个一个由序列组成的列表。如一个序列经过切分后变为["am", "##ro", "##zi", "accused", "his", "father"]，接下来按照它们在词表中对应的索引进行转换，假设结果如[23, 94, 13, 41, 27, 96]。假如整个文本一共100个句子，那么就有100个列表为它的元素，因为每个序列的长度不一，需要设定最大长度，这里不妨设为128，那么将整个文本转换为数组之后，形状即为100 x 128，这就对应着batch_size和seq_length。

输入之后，紧接着进行词嵌入处理，词嵌入就是将每一个词用预先训练好的向量进行映射，参数为词表的大小和被映射的向量的维度，通俗来说就是向量里面有多少个数。注意，第一个参数是词表的大小，如果你目前有4个词，就填4，你后面万一进入与这4个词不同的词，还需要重新映射，为了统一，一开始的也要重新映射，因此这里填词表总大小。假如我们打算映射到512维（num_features或者embed_dim），那么，整个文本的形状变为100 x 128 x 512。接下来举个小例子解释一下：假设我们词表一共有10个词，文本里有2个句子，每个句子有4个词，我们想要把每个词映射到8维的向量。于是2，4，8对应于batch_size, seq_length, embed_dim（如果batch在第一维的话）。

另外，一般深度学习任务只改变num_features，所以讲维度一般是针对最后特征所在的维度。

```python
import torch
import torch.nn as nn
X = torch.zeros((2,4),dtype=torch.long)
embed = nn.Embedding(10,8)
print(embed(X).shape)
# torch.Size([2, 4, 8])
```

***

**<div id='pos'>位置编码</div>**

词嵌入之后紧接着就是位置编码，位置编码用以区分不同词以及同词不同特征之间的关系。代码中需要注意：X_只是初始化的矩阵，并不是输入进来的；完成位置编码之后会加一个dropout。另外，位置编码是最后加上去的，因此输入输出形状不变。

```python
Tensor = torch.Tensor
def positional_encoding(X, num_features, dropout_p=0.0, max_len=512) -> Tensor:
    r'''
        给输入加入位置编码
    参数：
        - num_features: 输入进来的维度
        - dropout_p: dropout的概率，当其为非零元素时执行dropout
        - max_len: 句子的最大长度，默认512
    
    形状：
        - 输入： [batch_size, seq_length, num_features]
        - 输出： [batch_size, seq_length, num_features]

    例子：
        >>> X = torch.randn((2,4,10))
        >>> X = positional_encoding(X, 10)
        >>> print(X.shape)
        >>> torch.Size([2, 4, 10])
    '''

    dropout = nn.Dropout(dropout_p)
    P = torch.zeros((1,max_len,num_features))
    X_ = torch.arange(max_len,dtype=torch.float32).reshape(-1,1) / torch.pow(
        10000,
        torch.arange(0,num_features,2,dtype=torch.float32) /num_features)
    P[:,:0::2] = torch.sin(X_)
    P[:,:,1::2] = torch.cos(X_)
    X = X + P[:,:X.shape[1],:].to(X.device)
    return dropout(X)
```
***

**<div id='multihead'>多头注意力</div>**

多头注意力大概分为三个部分讲，分别为参数初始化，q,k,v从何来，遮挡机制，点积注意力

- 初始化参数

query，key，value是源语言序列（本文记为src）乘以对应的矩阵得到的，那么，那些矩阵从何而来（注意，因为大部分代码都是从源码中抽离出来的，因而常带有self等，最后会呈现组成好的，而行文过程中不会将整个结构呈现出来）：

```python
from torch.nn.parameter import Parameter
factory_kwargs = {'device': device, 'dtype': dtype}
if self._qkv_same_embed_dim is False:
    # 初始化前后形状维持不变
    # (seq_length x embed_dim) x (embed_dim x embed_dim) ==> (seq_length x embed_dim)
    self.q_proj_weight = Parameter(torch.empty((embed_dim, embed_dim), **factory_kwargs))
    self.k_proj_weight = Parameter(torch.empty((embed_dim, self.kdim), **factory_kwargs))
    self.v_proj_weight = Parameter(torch.empty((embed_dim, self.vdim), **factory_kwargs))
    self.register_parameter('in_proj_weight', None)
else:
    self.in_proj_weight = Parameter(torch.empty((3 * embed_dim, embed_dim), **factory_kwargs))
    self.register_parameter('q_proj_weight', None)
    self.register_parameter('k_proj_weight', None)
    self.register_parameter('v_proj_weight', None)

if bias:
    self.in_proj_bias = Parameter(torch.empty(3 * embed_dim), **factory_kwargs)
else:
    self.register_parameter('in_proj_bias', None)
# 后期会将所有头的注意力拼接在一起然后乘上权重矩阵输出
# out_proj是为了后期准备的
self.out_proj = nn.Linear(embed_dim, embed_dim, bias=bias, **factory_kwargs)
self._reset_parameters()
```

torch.empty是按照所给的形状形成对应的tensor，特点是填充的值还未初始化，类比torch.randn（标准正态分布），这就是一种初始化的方式。在PyTorch中，变量类型是tensor的话是无法修改值的，而Parameter()函数可以看作为一种类型转变函数，将不可改值的tensor转换为可训练可修改的模型参数，即与model.parameters绑定在一起，register_parameter的意思是是否将这个参数放到model.parameters，None的意思是没有这个参数。每个参数其实还有device和dtype两个属性，因此**factory_kwargs的意思是这两个参数是可变的。

这里有个if判断，用以判断q,k,v的最后一维是否一致，若一致，则一个大的权重矩阵全部乘然后分割出来，若不是，则各初始化各的，其实初始化是不会改变原来的形状的（如![](http://latex.codecogs.com/svg.latex?q=qW_q+b_q)，见注释）。

可以发现最后有一个_reset_parameters()函数，这个是用来初始化参数数值的。xavier_uniform意思是从[连续型均匀分布](https://zh.wikipedia.org/wiki/%E9%80%A3%E7%BA%8C%E5%9E%8B%E5%9D%87%E5%8B%BB%E5%88%86%E5%B8%83)里面随机取样出值来作为初始化的值，xavier_normal_取样的分布是正态分布。正因为初始化值在训练神经网络的时候很重要，所以才需要这两个函数。

constant_意思是用所给值来填充输入的向量。

另外，在PyTorch的源码里，似乎projection代表是一种线性变换的意思，in_proj_bias的意思就是一开始的线性变换的偏置

```python
from torch.nn.init import xavier_uniform_
from torch.nn.init import constant_
from torch.nn.init import xavier_normal_

def _reset_parameters(self):
    if self._qkv_same_embed_dim:
        xavier_uniform_(self.in_proj_weight)
    else:
        xavier_uniform_(self.q_proj_weight)
        xavier_uniform_(self.k_proj_weight)
        xavier_uniform_(self.v_proj_weight)
    if self.in_proj_bias is not None:
        constant_(self.in_proj_bias, 0.)
        constant_(self.out_proj.bias, 0.)

```
以上便是参数初始化过程，接下来是进行query,key和value的赋值

***
- q,k,v从何来？

对于`nn.functional.linear`函数，其实就是一个线性变换，与`nn.Linear`不同的是，前者可以提供权重矩阵和偏置，执行![](http://latex.codecogs.com/svg.latex?y=xW^T+b)，而后者是可以自由决定输出的维度，因为linear函数多层调用且无太大意义，这里省略。

```python

def _in_projection_packed(
    q: Tensor,
    k: Tensor,
    v: Tensor,
    w: Tensor,
    b: Optional[Tensor] = None,
) -> List[Tensor]:
    r"""
    用一个大的权重参数矩阵进行线性变换

    参数:
        q, k, v: 对自注意来说，三者都是src；对于seq2seq模型，k和v是一致的tensor。
                 但它们的最后一维(num_features或者叫做embed_dim)都必须保持一致。
        w: 用以线性变换的大矩阵，按照q,k,v的顺序压在一个tensor里面。
        b: 用以线性变换的偏置，按照q,k,v的顺序压在一个tensor里面。

    形状:
        输入:
        - q: shape:`(..., E)`，E是词嵌入的维度（下面出现的E均为此意）。
        - k: shape:`(..., E)`
        - v: shape:`(..., E)`
        - w: shape:`(E * 3, E)`
        - b: shape:`E * 3` 

        输出:
        - 输出列表 :`[q', k', v']`，q,k,v经过线性变换前后的形状都一致。
    """
    E = q.size(-1)
    # 若为自注意，则q = k = v = src，因此它们的引用变量都是src
    # 即k is v和q is k结果均为True
    # 若为seq2seq，k = v，因而k is v的结果是True
    if k is v:
        if q is k:
            return nn.functional.linear(q, w, b).chunk(3, dim=-1)
        else:
            # seq2seq模型
            w_q, w_kv = w.split([E, E * 2])
            if b is None:
                b_q = b_kv = None
            else:
                b_q, b_kv = b.split([E, E * 2])
            return (nn.functional.linear(q, w_q, b_q),) + nn.functional.linear(k, w_kv, b_kv).chunk(2, dim=-1)
    else:
        w_q, w_k, w_v = w.chunk(3)
        if b is None:
            b_q = b_k = b_v = None
        else:
            b_q, b_k, b_v = b.chunk(3)
        return nn.functional.linear(q, w_q, b_q), nn.functional.linear(k, w_k, b_k), nn.functional.linear(v, w_v, b_v)

q, k, v = _in_projection_packed(query, key, value, in_proj_weight, in_proj_bias)

```
***

- 遮挡机制

对于attn_mask来说，若为2D，形状如`(L, S)`，L和S分别代表着目标语言和源语言序列长度，若为3D,形状如`(N * num_heads, L, S)`，N代表着batch_size，num_heads代表注意力头的数目。若为ByteTensor，非0的位置会被忽略不做注意力；若为BoolTensor，True对应的位置会被忽略；若为数值，则会直接加到attn_weights。

因为在decoder解码的时候，只能看该位置和它之前的，如果看后面就犯规了，所以需要attn_mask遮挡住。

下面函数直接复制PyTorch的，意思是确保不同维度的mask形状正确以及不同类型的转换


```python
if attn_mask is not None:
    if attn_mask.dtype == torch.uint8:
        warnings.warn("Byte tensor for attn_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
        attn_mask = attn_mask.to(torch.bool)
    else:
        assert attn_mask.is_floating_point() or attn_mask.dtype == torch.bool, \
            f"Only float, byte, and bool types are supported for attn_mask, not {attn_mask.dtype}"
    # 对不同维度的形状判定
    if attn_mask.dim() == 2:
        correct_2d_size = (tgt_len, src_len)
        if attn_mask.shape != correct_2d_size:
            raise RuntimeError(f"The shape of the 2D attn_mask is {attn_mask.shape}, but should be {correct_2d_size}.")
            attn_mask = attn_mask.unsqueeze(0)
    elif attn_mask.dim() == 3:
        correct_3d_size = (bsz * num_heads, tgt_len, src_len)
        if attn_mask.shape != correct_3d_size:
            raise RuntimeError(f"The shape of the 3D attn_mask is {attn_mask.shape}, but should be {correct_3d_size}.")
    else:
        raise RuntimeError(f"attn_mask's dimension {attn_mask.dim()} is not supported")

```
与`attn_mask`不同的是，`key_padding_mask`是用来遮挡住key里面的值，详细来说应该是`<PAD>`，被忽略的情况与attn_mask一致。

```python
# 将key_padding_mask值改为布尔值
if key_padding_mask is not None and key_padding_mask.dtype == torch.uint8:
    warnings.warn("Byte tensor for key_padding_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
    key_padding_mask = key_padding_mask.to(torch.bool)
```

先介绍两个小函数，`logical_or`，输入两个tensor，并对这两个tensor里的值做`逻辑或`运算，只有当两个值均为0的时候才为`False`，其他时候均为`True`，另一个是`masked_fill`，输入是一个mask，和用以填充的值。mask由1，0组成，0的位置值维持不变，1的位置用新值填充。
```python
a = torch.tensor([0,1,10,0],dtype=torch.int8)
b = torch.tensor([4,0,1,0],dtype=torch.int8)
print(torch.logical_or(a,b))
# tensor([ True,  True,  True, False])
```

```python
r = torch.tensor([[0,0,0,0],[0,0,0,0]])
mask = torch.tensor([[1,1,1,1],[0,0,0,0]])
print(r.masked_fill(mask,1))
# tensor([[1, 1, 1, 1],
#         [0, 0, 0, 0]])
```
其实attn_mask和key_padding_mask有些时候对象是一致的，所以有时候可以合起来看。`-inf`做softmax之后值为0，即被忽略。
```python
if key_padding_mask is not None:
    assert key_padding_mask.shape == (bsz, src_len), \
        f"expecting key_padding_mask shape of {(bsz, src_len)}, but got {key_padding_mask.shape}"
    key_padding_mask = key_padding_mask.view(bsz, 1, 1, src_len).   \
        expand(-1, num_heads, -1, -1).reshape(bsz * num_heads, 1, src_len)
    # 若attn_mask为空，直接用key_padding_mask
    if attn_mask is None:
        attn_mask = key_padding_mask
    elif attn_mask.dtype == torch.bool:
        attn_mask = attn_mask.logical_or(key_padding_mask)
    else:
        attn_mask = attn_mask.masked_fill(key_padding_mask, float("-inf"))

# 若attn_mask值是布尔值，则将mask转换为float
if attn_mask is not None and attn_mask.dtype == torch.bool:
    new_attn_mask = torch.zeros_like(attn_mask, dtype=torch.float)
    new_attn_mask.masked_fill_(attn_mask, float("-inf"))
    attn_mask = new_attn_mask

```

***
-  点积注意力

```python
from typing import Optional, Tuple
def scaled_dot_product_attention(
    q: Tensor,
    k: Tensor,
    v: Tensor,
    attn_mask: Optional[Tensor] = None,
    dropout_p: float = 0.0,
) -> Tuple[Tensor, Tensor]:
    r'''
    在query, key, value上计算点积注意力，若有注意力遮盖则使用，并且应用一个概率为dropout_p的dropout

    参数：
        - q: shape:`(B, Nt, E)` B代表batch size， Nt是目标语言序列长度，E是嵌入后的特征维度
        - key: shape:`(B, Ns, E)` Ns是源语言序列长度
        - value: shape:`(B, Ns, E)`与key形状一样
        - attn_mask: 要么是3D的tensor，形状为:`(B, Nt, Ns)`或者2D的tensor，形状如:`(Nt, Ns)`

        - Output: attention values: shape:`(B, Nt, E)`，与q的形状一致;attention weights: shape:`(B, Nt, Ns)`
    
    例子：
        >>> q = torch.randn((2,3,6))
        >>> k = torch.randn((2,4,6))
        >>> v = torch.randn((2,4,6))
        >>> out = scaled_dot_product_attention(q, k, v)
        >>> out[0].shape, out[1].shape
        >>> torch.Size([2, 3, 6]) torch.Size([2, 3, 4])
    '''
    B, Nt, E = q.shape
    q = q / math.sqrt(E)
    # (B, Nt, E) x (B, E, Ns) -> (B, Nt, Ns)
    attn = torch.bmm(q, k.transpose(-2,-1))
    if attn_mask is not None:
        attn += attn_mask 
    # attn意味着目标序列的每个词对源语言序列做注意力
    attn = nn.functional.softmax(attn, dim=-1)
    if dropout_p:
        attn = nn.functional.dropout(attn, p=dropout_p)
    # (B, Nt, Ns) x (B, Ns, E) -> (B, Nt, E)
    output = torch.bmm(attn, v)
    return output, attn 
```

接下来将三个部分连起来：
```python
def multi_head_attention_forward(
    query: Tensor,
    key: Tensor,
    value: Tensor,
    num_heads: int,
    in_proj_weight: Tensor,
    in_proj_bias: Optional[Tensor],
    dropout_p: float,
    out_proj_weight: Tensor,
    out_proj_bias: Optional[Tensor],
    training: bool = True,
    key_padding_mask: Optional[Tensor] = None,
    need_weights: bool = True,
    attn_mask: Optional[Tensor] = None,
    use_seperate_proj_weight = None,
    q_proj_weight: Optional[Tensor] = None,
    k_proj_weight: Optional[Tensor] = None,
    v_proj_weight: Optional[Tensor] = None,
) -> Tuple[Tensor, Optional[Tensor]]:
    r'''
    形状：
        输入：
        - query：`(L, N, E)`
        - key: `(S, N, E)`
        - value: `(S, N, E)`
        - key_padding_mask: `(N, S)`
        - attn_mask: `(L, S)` or `(N * num_heads, L, S)`
        输出：
        - attn_output:`(L, N, E)`
        - attn_output_weights:`(N, L, S)`
    '''
    tgt_len, bsz, embed_dim = query.shape
    src_len, _, _ = key.shape
    head_dim = embed_dim // num_heads
    q, k, v = _in_projection_packed(query, key, value, in_proj_weight, in_proj_bias)

    if attn_mask is not None:
        if attn_mask.dtype == torch.uint8:
            warnings.warn("Byte tensor for attn_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
            attn_mask = attn_mask.to(torch.bool)
        else:
            assert attn_mask.is_floating_point() or attn_mask.dtype == torch.bool, \
                f"Only float, byte, and bool types are supported for attn_mask, not {attn_mask.dtype}"

        if attn_mask.dim() == 2:
            correct_2d_size = (tgt_len, src_len)
            if attn_mask.shape != correct_2d_size:
                raise RuntimeError(f"The shape of the 2D attn_mask is {attn_mask.shape}, but should be {correct_2d_size}.")
            attn_mask = attn_mask.unsqueeze(0)
        elif attn_mask.dim() == 3:
            correct_3d_size = (bsz * num_heads, tgt_len, src_len)
            if attn_mask.shape != correct_3d_size:
                raise RuntimeError(f"The shape of the 3D attn_mask is {attn_mask.shape}, but should be {correct_3d_size}.")
        else:
            raise RuntimeError(f"attn_mask's dimension {attn_mask.dim()} is not supported")

    if key_padding_mask is not None and key_padding_mask.dtype == torch.uint8:
        warnings.warn("Byte tensor for key_padding_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
        key_padding_mask = key_padding_mask.to(torch.bool)
    
    # reshape q,k,v将Batch放在第一维以适合点积注意力
    # 同时为多头机制，将不同的头拼在一起组成一层
    q = q.contiguous().view(tgt_len, bsz * num_heads, head_dim).transpose(0, 1)
    k = k.contiguous().view(-1, bsz * num_heads, head_dim).transpose(0, 1)
    v = v.contiguous().view(-1, bsz * num_heads, head_dim).transpose(0, 1)
    if key_padding_mask is not None:
        assert key_padding_mask.shape == (bsz, src_len), \
            f"expecting key_padding_mask shape of {(bsz, src_len)}, but got {key_padding_mask.shape}"
        key_padding_mask = key_padding_mask.view(bsz, 1, 1, src_len).   \
            expand(-1, num_heads, -1, -1).reshape(bsz * num_heads, 1, src_len)
        if attn_mask is None:
            attn_mask = key_padding_mask
        elif attn_mask.dtype == torch.bool:
            attn_mask = attn_mask.logical_or(key_padding_mask)
        else:
            attn_mask = attn_mask.masked_fill(key_padding_mask, float("-inf"))
    # 若attn_mask值是布尔值，则将mask转换为float
    if attn_mask is not None and attn_mask.dtype == torch.bool:
        new_attn_mask = torch.zeros_like(attn_mask, dtype=torch.float)
        new_attn_mask.masked_fill_(attn_mask, float("-inf"))
        attn_mask = new_attn_mask

    # 若training为True时才应用dropout
    if not training:
        dropout_p = 0.0
    attn_output, attn_output_weights = _scaled_dot_product_attention(q, k, v, attn_mask, dropout_p)
    attn_output = attn_output.transpose(0, 1).contiguous().view(tgt_len, bsz, embed_dim)
    attn_output = linear(attn_output, out_proj_weight, out_proj_bias)
    if need_weights:
        # average attention weights over heads
        attn_output_weights = attn_output_weights.view(bsz, num_heads, tgt_len, src_len)
        return attn_output, attn_output_weights.sum(dim=1) / num_heads
    else:
        return attn_output, None

```


https://www.jianshu.com/p/d8b77cc02410