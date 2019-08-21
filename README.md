# ifly-algorithm_challenge
讯飞移动反欺诈算法竞赛,目前分数只有94.48

讯飞移动反欺诈算法数据竞赛网址： http://challenge.xfyun.cn/2019/gamedetail?type=detail/mobileAD

### 总体流程
```
  | EDA
  | 数据预处理
  | 数据特征构造
  | 模型搭建
  | 模型参数的调优以及特征筛选
```
#### 一、EDA

 在做数据竞赛的时候，当我们拿到数据集的时候，我们首先要做的，也是最重要的事情那就是进行数据探索分析，这部分做的好，对接下来的每个部分的工作都是有力的参考，而不会盲目的去进行数据预处理，以及特征构造。数据探索分析主要做以下几个部分的工作:
   - 查看每个数据特征的缺失情况、特征中是否含有异常点，错误值
   - 查看每个特征的分布情况，主要包括连续特征是否存在偏移，也就是不服从正态分布的情况；离散特征的值具体分布如何
   - 查看一般特征之间的相关性如何
   - 查看一般特征与目标特征之间的相关性


#### 二、数据预处理

通过EDA,我们对数据进行了初步分析，接下来就针对EDA部分得出的结果，来进行数据的预处理工作,主要做了以下工作:
  
  - 先是对特征中不符合类型的数据进行类别转换
  - 利用众数来对缺失值填充，异常值的剔除，以及错误值进行修正
  - 对于连续特征，如果存在偏移现象，使用对数或者Box-Cox进行转换，使其满足正太分布，降低拖尾带来的影响
  - 对于离散值，一般是进行独热编码，但由于本身数据集离散特征的取值类别数较多，如果使独热编码(OneHotEncoder)，会使得数据得维度变得很高，会降低模型运行的速度；这里我使用的是类别编码(LabelEncoder)来对离散特征进行编码
  - 对部分连续特征进行分箱操作，将其离散化。

#### 补充一点(比较重要)
```
面对数据量很大时候，如何解决大规模数据建模问题？一般会有三种基本方法:
  1. 对原始样本进行抽样，不过这样会存在一个问题，那就是会导致正负样本可能失衡，影响模型最终的效果
  2. 对数据结构或者类型进行优化来降低内存的消耗：
     (1) 当特征取值没有负数的时候，我们可以将int32类型的数据转换为uint8。
     (2) 将float64类型的数据转换为float32
     (3) 在不影响模型效果的前提下，可以将object类型的数据转化为category类型的数据，这种适合离散特征取值较少的情况，一般特征的的取值类别数占特征总的数量的比例小于5%。这次采用的是这种方法，具体实现见代码.
  3. 利用online learning等相关方法。
```
#### 数据特征构造

```
特征主要分为：
   1. 原始特征  
   2. 统计特征 ： count, max, min,std, nunique,mean,...
   3. 组合特征 : 将重要性较高的特征与其他的特征进行组合
   4. 叶子节点特征：利用lgb和xgb模型来生成部分叶子节点特征
```

#### 模型的选择
```
  这里主要是选用了四个模型，三个机器学习模型，一个深度学习模型
    1. lightgbm
    2. xgboost
    3. catboost
    4. MLP 
  最后发现catboost效果最好，catboost有一个参数cat_features,通过这个参数，我们可以指定离散特征的索引.效果的确很好，但也很吃内存，最后因为没有机器跑模型，到后面基本上很多想法都不能实现。。。。
```
#### 模型参数的寻优
可直接参考：  https://www.cnblogs.com/pinard/p/11114748.html

#### 特征的筛选
```
  1. 使用了随机森林，lgb来进行特征重要性的输出
  2. 使用了基于过滤器的卡方，基于包装器的递归特征消除，以及皮尔逊相关系数来进行特征选择
```

#### 模型的融合

在模型的输出时，我们输出的是概率，这里采用的加权融合的方法,```catboost*0.5 + lgb*0.2 + xgb*0.2+ MLP * 0.1```,然后再进行最终类别的输出.

#### 最后再补充一下

我们在做分类任务时，如果正负样本分布均衡，我们如何处理呢？

Q: 首先要说下，为什么正负样本分布不均衡，会影向模型的最终效果呢？

A: 本质原因是在训练优化的目标寒素与测试时使用的评价标准不一致，主要分为样本分布不一致，类别权重不同。

Q: 如何处理?

A: 一般会进行```随机采样```操作，具体分```过采样```和```欠采样```操作。但是如果单纯的增加类别少的样本，增加模型的复杂度的同时，也容易造成过拟合，一般我们选择```SMOTE```方法来生成新样本。在进行欠采样的时候，如果单纯的从类别多的样本中选取数量和样本少的样本进行训练，这样容易丢失部分有用信息，一般我们使用```Easy Ensamble``` 和```Balance Cascade```方法来进行采样，其中```Easy Ensemble```思想就是对数据集进行多次欠采样，训练多个基模型，然后将这些模型的输出进行有效结合作为最的结果。```Balance Cascade```是一个级联结构，在每一级中从多数类S1中随机抽取子集E,用E+S2(少数类)训练该级分类器，然后将S1中能够被当前分类器正确判别的样本剔除掉，继续下一级的操作，重复若干次得到级联结构(有点像Adaboost思想降低被正确分类样本的权重，只不过是直接将权重设为0)，最终的结果也是各级分类器结果的融合。

### 参考资料xiang

- [x] GBDT、xgboost对比分析：

https://wenku.baidu.com/view/f3da60b4951ea76e58fafab069dc5022aaea463e.html

- [x] xgboost论文：

https://arxiv.org/pdf/1603.02754.pdf

- [x] lightgbm论文:

http://papers.nips.cc/paper/6907-lightgbm-a-highly-efficient-gradient-boosting-decision-tree.pdf

- [x] catboost论文：

  - https://arxiv.org/pdf/1706.09516.pdf

  - http://learningsys.org/nips17/assets/papers/paper_11.pdf
