### Recurrent Neutral Network

章节

- [RNN概述](#summary)
- [梯度困区](#problems)
- [LSTM](#lstm)
- [应用](#application)

### <div id='summary'>RNN概述</div>

为什么它叫做递归神经网络呢？与其他网络有何不同？接下来用简单例子阐述：

这是比较简单的示意图，比如说一个网络只有一层，那么，那一层代表的函数方法就是这个网络实际对输入所起的作用，即Y = Funtion(X)，我们实际上想找出那个function它究竟是什么。

![](https://github.com/sherlcok314159/ML/blob/main/Images/network.png)

可以从下图看出，RNN得到一个输出不仅仅靠输入的X，同时还依赖于h，h在RNN中被叫做cell state，那么h如何得出呢？由公式（1）可知，h_t是由h_(t-1)经过某种函数变换得到的，换句话说，我要得到目前这一个的，我还必须经过前一个才能做到。这里我们可以类比一下斐波那契数列，f(t) = f(t-1) + f(t-2)，某一项需要由前两项一起才能完成，RNN是某一个h需要前面一个h来完成，这也是为什么被叫做递归神经网络。顺带一提，这里的function有权重参数，即为W,而这个W是共享的，意思是无论是h_1到h2还是h_2到h_3，它们用的function其实是一样的。

![](https://github.com/sherlcok314159/ML/blob/main/Images/rnn.png)

![](https://github.com/sherlcok314159/ML/blob/main/Images/ht.png)

所以，复杂一点的RNN长这样：

![](https://github.com/sherlcok314159/ML/blob/main/Images/rnn_1.png)

每次输出完一个y，它同时还会有一个h出来，作为下一层的参数一起使用。从这一点来看，RNN跟其他网络不同的一点是前一层的输出同时可以作为后一层的输入，经过一层就会更新一次h，那么，h究竟是如何更新的呢？tanh是一种常用的激活函数，可见[Activation](../activation.md)。

![](https://github.com/sherlcok314159/ML/blob/main/Images/htt.png)

y_t可以由此得出：

![](https://github.com/sherlcok314159/ML/blob/main/Images/y_t.png)

从上述公式中可以看出有不同的W，即不同的权重矩阵，这些矩阵是机器自己去从数据中去学出来，同时也可以是人为设置的。注意，这些不同类之间的矩阵不同，但是如果说是同一个function，那么权重矩阵都是共享的。

传统的DNN，CNN的输入和输出都是固定的向量，而RNN与这些网络的最大不同点是它的输入和输出都是不定长的，具体因不同任务而定。

![](https://github.com/sherlcok314159/ML/blob/main/Images/many_lengths.png)

***
### <div id='problems'>梯度困区</div>

上述的RNN都是正向的，我们通过梯度下降来找出最优解，在[backpropagation](../bp.md)中讲到，我们要求损失函数关于某个参数的梯度时，需要用到反向传播。我们是基于链式法则来进行反向传播的，而当RNN的网络层数越来越多的时候，链就会越来越长，乘的东西也就越来越多，那么就很容易导致梯度爆炸和梯度消失这两类问题。举个例子，一开始有个数0.1，看起来还行吧，我们把0.1连续乘上三次0.1，0.0001，这个数就已经相当的小了。

- 梯度削减（Gradient Clipping）

对于梯度爆炸问题，我们可以采取梯度削减，首先设置一个clip_gradient作为梯度阈值，然后按照往常一样求出各个梯度，不一样的是，我们没有立马进行更新，而是求出这些梯度的L2范数，注意这里的L2范数与岭回归中的L2惩罚项不一样，前者求平方和之后开根号而后者不需要开根号。如果L2范数大于设置好的clip_gradient，则求clip_gradient除以L2范数，然后把除好的结果乘上原来的梯度完成更新。当梯度很大的时候，作为分母的结果就会很小，那么乘上原来的梯度，整个值就会变小，从而可以有效地控制梯度的范围。有一点疑惑的就是，梯度削减会使得原来的梯度过大的部分发生变化，方向既然发生了变化，为什么最后还能使得loss收敛呢？Deep Learning大概结果反推出解释吧。

接下来的部分是将如何解决梯度削减的问题：

- 切换激活函数

我们知道在CNN中出现梯度削减的原因是使用了sigmoid函数，

***
### <div id='lstm'>LSTM</div>

1
***
### <div id='application'>应用</div>

1
***
