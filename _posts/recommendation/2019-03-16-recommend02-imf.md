---
layout: post
title:  推荐技术系列02：隐式矩阵分解(IMF)详解
categories: [IMF,Recommendation]
---


## 背景

显示反馈在生产环境中随处可见，比如用户对道具的购买量，视频的观看时间，文章的阅读时长等。为了获取这些信息，业务测无需做额外工作，所以获取成本非常低廉，广泛的存在我们的线上生产环境中。相比而言，获取显示反馈就比较困难，因为业务测必须实现对物品的打分功能，而这些功能可能不会对业务产生价值，尤其是在线游戏中。笔者自从工作以来，还没有在实际工作中接触到显示反馈相关的真实数据。所以，研究基于隐式反馈的推荐技术，在我们的生产环境中，具有比较重要的应用价值。

## 1 显式(SVD)反馈矩阵分解原理简介
在介绍隐式反馈之前，很有必要简单介绍显示反馈矩阵分解(简称EMF)的算法原理，因为隐反馈算法基本上是在显示反馈的算法框架下进行调整而得到的，便于后面算法的理解。详细的显示反馈算法原理，参考文章[FunkSVD](https://sifter.org/simon/journal/20061211.html)。
在EMF中，将用户对物品的打分看做一个矩阵 $ D_{m \times n} ​$ ，行代表用户 $ u ​$ ，列代表物品 $ i ​$ 。每一个元素代表用户对特定物品的打分，打分区间最常见的是1到5。用户不可能对每个物品都打分，这些数据就忽略，需要特别强调的是这些分数不是0，而是未知。所以，我们希望找到一种矩阵分解，
$$
	D_{m \times n} = U_{m \times f} \times I_{f \times n} 
$$

其中，U所有用户的隐向量组成的向量，每行表示一个用户；I的每一列是物品的隐向量。对于那些没有打分的物品，系统通过两个隐向量的点积得到相应的打分。为了计算这些隐向量，我们需要通过对已打分的物品学习，求解 $ (m +n ) \times f $ 个参数，来推导用户对没接触过的物品的打分。损失函数使用平方差错误MSE，最终目标函数为，

$$
G(p_\star,q_\star) = \sum_{u,i} I_{ui} (r_{ui} - p_uq_i)^2 + \lambda (\sum_u \Vert p_u\Vert^2 + \sum_i \Vert q_i\Vert^2 )
$$

其中 $ I_{ui} = 1 $ 当用户 $ u $ 对物品 $ i $ 有过评分，否则为0。 $ \lambda $ 是正则化系数。 $ r_{ui} $ 是用户 $ u $ 对物品 $ i $ 的评分。 $ p_u $ 和 $ q_i $ 分别是用户和物品的隐向量。 然后使用随机梯度下降等优化方法求解所有的 $ p_u $ 和 $ q_i $ 。此算法的计算量与打分数量成正比，时间复杂度可以接受。SVD还有很多衍生算法，比较著名的是[SVD++](https://zhuanlan.zhihu.com/p/42269534)，都是在其基础上添加一些改进，但是总体上仍属于显式矩阵分解类推荐算法。以上就是简要的显式矩阵分解算法介绍。


## 2 隐式反馈举证分解原理详解

IMF与EMF最根本的区别在于输入数据的意义。
对于EMF，输入矩阵的每个元素 $ r_{ui} $ 代表用户 $ u $ 对物品 $ i $ 的打分，一般在1-5之间，1代表不喜欢，5代表非常喜欢，其他分数以此类推。如果没有评分，建模过程不予考虑。
在IMF中，输入矩阵的每个值 $ p_{ui} $ 代表用户 $ u $ 对物品 $ i $ 的偏好(Preference)，该值越高，我们就与有信心(Confidence)认为用户 $ u $ 更偏好物品 $ i $ 。偏好一般用交互次数或程度表示，比如购买次数，观看比例，在线时长等。即使偏好很低，比如为0，也需要考。因为可能由于一些客观原因，用户和物品无法相遇，从数据上不能很自信的认为用户 $ u $ 喜欢物品 $ i $ ，这并不代表用户对物品有负向的反馈。 IMF中 $ r_{ui} $ 真正代表的含义，只能猜测，不像EMF中的 $ r_{ui} $ 那样“爱憎分明“。

### 启发式目标函数构建

基于上面提到的区别，作者启发式的构造了偏好和信心的关系

$$
c_{ui} = 1 + \alpha r_{ui} \qquad (1) 
$$

公式(1)比较简单，只有一个超参数 $ \alpha $ ，如果需要更加粒度的刻画，可以用下面的加强版本，

$$
c_{ui} = 1 + \alpha \log(1 + \frac{r_{ui}}{\epsilon}) \qquad (2) 
$$

此版本对高交互的物品附加了一个 $ log $ 惩罚，但是惩罚之前给予了一个 $ 1/ \epsilon $ 的提升，此方法有2个超参数。当然变种还有很多，但是从实战角度而言，超参数越少，需要调试的地方越少，所以后面的推导都是基于公司(1)进行。

用户对物品是否有打分使用 $ p_{ui} $ 表示，

$$
p_{ui} = \begin{equation}
   \begin{cases}
    1 & r_{ui} > 0 \\
    0 & r_{ui} = 0
  \end{cases} \qquad (3)
\end{equation}
$$

我们的优化目的是希望 $ x_u^T y_i  = p_{ui} $，但是同时需要考虑数据的自信程度，即越自信，惩罚力需要越大；如果不自信，惩罚力度相对较轻，因为不确定，所以不能随便惩罚。基于这种朴素的启发式思想，最终的目标函数如下

$$
G(x_\star, y_\star) = \left(\sum_{u,i} c_{ui}(p_{ui}-x_u^Ty_i)^2\right) + \lambda \left( \sum_u \Vert x_u \Vert^2 + \sum_i \Vert y_i \Vert^2 \right) \qquad (4)
$$

最小化 $ G(x_u,y_i) $ 可得到用户和道具隐式矩阵。公式(4)中第二部分是正则化项，用于解决过拟合问题。 

### 优化求解以及时间复杂度分析
对于公式(4)，有 $ m*n $ 项需要计算，这种时间复杂度无法使用随机梯度下降，而且梯度计算较复杂。但是，一旦固定所有 $ y_\star $ 求解 $ x_\star $ ，那么目标函数就变成2次，是**凸函数**，且在最优点有**解析解**。所以，我们可以使用交替最小二乘法(Alternating-Least-Squares)来求解这个问题。因为每次固定 $ x_\star $ 或 $ y_\star $ ，整体损失都可以保证朝较小方向走，直到收敛为止。虽然不是最为精确的解，但是在实际问题上够用。下面观察固定 $ y_\star $ 时，$ x_u $ 在最优点的形式，

$$
x_u^T = (Y^TC^uY+\lambda I)^{-1} Y^TC^up(u) \qquad (5)
$$

公式(5)具体的推导形式参见附录。对于每一个用户，复杂度为 $ O(n) $ ，所以整体的复杂度仍然是 $ O(m*n) $ 。但是，我们可以利用公式(5)的结构，进行一些优化。 

*  $ Y^TC^uY = Y^TY + Y^T(C^u-I)Y $  ，其中 $ Y^TY $ 只需要算一次，所有用户可以共用，所以对每个用户的复杂度为 $ O(\frac{1}{m} nf^2) $ ,其中 $ f $ 为隐式分解维度。 
*  $ (C^u-I) ​$  只有 $ n_u ​$ 个非0项，且 $ N = \sum_u n_u ​$ ，所以 $  Y^T(C^u-I)Y ​$ 的复杂度 $ O(n_u f^2 ) ​$ 。
* 逆矩阵的复杂度为 $ O(f^3) $ 。
*  $ Y^TC^up(u) ​$ 的复杂度为 $ O(n_uf) ​$ 。

所以，m个用户的整体复杂度为 $ O(mf^3+N f^2) $ 。固定 $ x_\star $ 时，对 $ y_\star $ 的复杂度以此类推，为 $ O(nf^3+N f^2) $  。所以，当 $ f $ 不大时，整体复杂度为 $ O(N) $ 与显式矩阵分解复杂度一样，均是用户打分项的线性复杂度。

上面是IMF的核心内容，如果读者还想了解其他细节，可以参考[原始论文](http://yifanhu.net/PUB/cf.pdf)。

## 3 IMF实战

如果希望在较大数据集上使用IMF，笔者推荐使用spark mllib的[ALS模块](https://spark.apache.org/docs/latest/mllib-collaborative-filtering.html)，该模块实现了IMF和EMF，使用spark分布式架构，可以很高效的处理千万量级的数据集。R和python也有对应的实现，有兴趣的读者可以google或bing一下相关内容。

 


## 总结

以上就是IMF的原理，以及实战相关的分析。该算法使用成本较低，无需开发用户宽表，以及物品特征，适合快速上线，十分适合我们的敏捷开发模式。建议大家后续在工作中可以尝试使用该技术，推荐以下几种适合使用IMF的场景，

* Case 1: 活动上线压力大，需要快速上线，没有时间开发宽表和计算道具特征，可以使用IMF。 
* Case 2: 热销作为base line太弱，需要一个较强的base line，而且成本不高，可以使用IMF。
* Case 3: 需要道具embedding或者用户embedding，可以使用IMF。

使用中有任何问题欢迎和笔者讨论。



## 附录：IMF目标函数求导
原始论文中，直接给出了目标函数，以及基于用户隐式向量的求导结果，没有给出推导过程。这里给出详细的推导过程，作为补充。


针对目标函数，在Y固定情况下，对分量 $ x_{uk} $ 求偏导

$$
\frac {\partial G(x_\star,y_\star)}{\partial x_{uk}} = \Sigma_i 2c_{ui}(x_uy_i^T-p_{ui})y_{ik} + 2\lambda x_{uk} = 0 \Rightarrow  \Sigma_i c_{ui}(p_{ui}-x_uy_i^T)y_{ik} = \lambda x_{uk} 
$$

扩展到用户 $ u $ 的所有隐式分量，

$$
\begin{bmatrix}
\Sigma_i c_{ui}(p_{ui}-x_uy_i^T)y_{i1} \\
\Sigma_i c_{ui}(p_{ui}-x_uy_i^T)y_{i2}\\
 \vdots \\
\Sigma_i c_{ui}(p_{ui}-x_uy_i^T)y_{if}\\
\end{bmatrix} = 
\begin{bmatrix}
\lambda x_{u1}\\
\lambda x_{u2}\\
 \vdots \\
\lambda x_{uf}
\end{bmatrix} 
$$

令 $ z_u = p(u)-Yx_u^T $ ，上面可以简化为（变化关键是将三个分量抽象成矩阵形式），

$$
\begin{bmatrix}
z_u^T C^u Y_{\star 1} \\ 
z_u^T C^u Y_{\star 2} \\ 
 \vdots \\
z_u^T C^u Y_{\star f} \\ 
\end{bmatrix} 
=
\begin{bmatrix}
 Y_{\star 1}^TC^uz_u \\ 
 Y_{\star 2}^TC^uz_u \\ 
 \vdots \\
 Y_{\star f}^TC^uz_u \\ 
\end{bmatrix} = 
\begin{bmatrix}
 Y_{\star 1}^T\\ 
 Y_{\star 2}^T \\ 
 \vdots \\
 Y_{\star f}^T \\ 
\end{bmatrix} C^uz_u 
=Y^TC^uz_u = \lambda x_u^T \Rightarrow Y^TC^u(p(u) - Yx_u^T) = \lambda x_u^T 
$$

其中 $ Y_{\star k} $ 表示道具矩阵Y的第k列。最后提取 $ x_u^T $ 向量，可以得到

$$
x_u^T = (Y^TC^uY+\lambda I)^{-1} Y^TC^up(u)
$$

针对道具隐式向量 $ y_i^T $ 的推导与上面非常类似，这里省略，有兴趣的读者可以自行推导。










