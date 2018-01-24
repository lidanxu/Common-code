本文是XGBoost模型[介绍](https://xgboost.readthedocs.io/en/latest/model.html)的翻译。

# Introduction to Boosted Trees

XGBoost是“Extreme Gradient Boosting”的缩写，
其中‘Gradient Boosting’是Friedman的论文Greedy Function Approximation：Gradient Boosting Machine中提出的。

XGBoost基于这个原始模型。 本文关于渐变增强树（gradient boosted trees）的教程，大部分内容是基于xgboost作者的这些[幻灯片](https://homes.cs.washington.edu/~tqchen/pdf/BoostedTree.pdf)。

GBM（boosted trees）提出已经有一段时间了，关于这个话题有很多材料。本教程试图用监督式学习元素以独立和有原则（直译，没懂是什么意思）的方式来解释提升树。 我们认为这个解释更清晰，更正式，并鼓励使用xgboost延伸的变体。

## Elements of Supervised Learning(监督学习元素)

XGBoost用于有监督的问题，使用训练数据（包含很多特征）x<sub>i</sub>预测目标变量y<sub>i</sub>.在我们深入trees之前，首先回顾一下监督学习的基本要素。
 
### Model and Parameters

监督学习模型通常指的是根据已给的x<sub>i</sub>预测y<sub>i</sub>的数学结构。举个例子，一个通用的模型如一个线性模型，其中的预测结果由下式给出：![linear model](https://github.com/lidanxu/Common-code/blob/master/images/linear_model.png)
预测结果y<sub>i</sub>是输入特征x<sub>i</sub>的加权线性组合。 预测值y可以具有不同的解释，这取决于任务，即回归或分类。 例如，可以通过logistic transformed来获取逻辑回归中正例的概率，也可以作为排序得分当我们需要对结果进行排序。

参数是我们需要从训练数据中学习的未确定部分。在线性回归问题中，参数是系数θ。 通常我们将用θ来表示参数（模型中有很多参数，我们这里的定义是粗略的）。

### Objective Function:Training Loss + Regularization

目标函数：训练误差 + 规则化

基于对y<sub>i</sub>不同的理解，我们可能会遇到不同的问题。如y为离散整型时，可能是分类问题；y为连续数值型，则可能是回归问题。我们需要找到一种方法根据训练数据的得到模型的最佳参数。 为了做到这一点，我们需要定义一个所谓的目标函数，用以给定一组参数来衡量模型的表现。

关于目标函数的一个非常重要的事实是它必须总是包含两个部分：训练损失和基于对yi的不同理解，我们可能会遇到不同的问题，如回归，分类，排序等。我们需要找到一种方法来找到训练数据的最佳参数。 为了做到这一点，我们需要定义一个所谓的目标函数，以给定一组参数来衡量模型的性能。

关于目标函数的一个非常重要的事实是它必须总是包含两个部分：训练损失和正规化。

![obj(θ)](https://github.com/lidanxu/Common-code/blob/master/images/objection_function.png)

其中L是训练误差，Ω是regularization term 正规化项。训练损失衡量我们的模型如何预测训练数据。 例如，常用的训练损失是均方误差squared error。

![squared error](https://github.com/lidanxu/Common-code/blob/master/images/squared_error.png)

另一种常用的损失函数是逻辑回归的logistic loss：

![logistic loss](https://github.com/lidanxu/Common-code/blob/master/images/logistic_loss.png)

通常我们会忘记添加正规化项regularization term.regularization term控制模型的复杂度，从而避免过拟合。这听起来有些抽象，所以让我们在下面的图片中考虑下面的问题。假设你需要根据左上角的图片中的数据点拟合出一个阶梯函数，下面三种解决方案中那种最合适？ 

![User's interest](https://github.com/lidanxu/Common-code/blob/master/images/user_interest.png)

正确的结果被标记为红色。请考虑对你而言视觉上这样拟合也是合理的。 一般原则是我们想要一个简单又预测能力强的模型。
【简单：规则化；预测能力强：训练误差小】 两者之间的折衷也被称为机器学习中的偏差 - 方差折衷。


### why introduce the general principle

上面介绍的要素构成了监督学习的基本要素，它们是机器学习工具包的基石。 例如，你应该能够描述强化树木boosted trees 和随机森林random forests之间的差异和共同点。 以正式的方式(直译)理解这个过程也有助于我们理解我们正在学习的目标以及启发式算法如修剪和平滑背后的原因。


## Tree Ensemble
树集成算法

既然我们已经介绍了监督学习的基本内容，让我们开始学习真正的tree算法。 首先，让我们先了解一下xgboost：tree集成模型。 树集成模型是一组分类和回归树（CART）。 下图是一个CART的简单例子，它可以分类是否有人喜欢电脑游戏。

![a example of CART 1](https://github.com/lidanxu/Common-code/blob/master/images/example_CART_1.png)

我们将一个家庭的成员分到不同的叶子中去，并给相应的叶子分配分数。 CART与决策树有些不同，在决策树中叶子只包含决策值。 在CART中，每个叶子都有一个真实的分数，这给了我们更丰富的解释，超越了分类。这也使得统一的优化步骤更容易，我们将在本教程的后面部分看到。

通常情况下，一棵树在实践中使用不够强大。 实际使用的是所谓的树集成模型，它将结合多棵树的预测结构。

![a example of CART 2](https://github.com/lidanxu/Common-code/blob/master/images/example_CART_2.png)

上图是两颗树集成的例子。将两颗树的预测分数加起来构成模型的最后分数。在这个例子中可以看出一个重要的事实就是这两棵树互相补充。 在数学上，我们可以中这样表示模型：

![a example of CART 3](https://github.com/lidanxu/Common-code/blob/master/images/example_CART_3.png)

其中K是树的个数，f是函数空间（假设空间）F中的函数，其中F是所有可能的CART树集合。所以我们优化的目标可以写成

![obj(θ)](https://github.com/lidanxu/Common-code/blob/master/images/example_CART_4.png)

现在问题来了，随机森林的模型是什么？ 这正是树集成模型！ 所以随机森林和boosted trees在模型方面没有什么不同，不同之处在于我们如何训练它们。 这意味着如果你写一个树集成的预测模型，你只需要编写其中的一个，他们应该直接能为随机森林和boosted trees工作。 


## Tree Boosting
在介绍模型之后，让我们从真正的训练部分开始。 我们应该如何学习树木？或者说应该如何训练树模型？ 答案就像所有的监督学习模型一样：定义一个目标函数并对其进行优化！

假设我们有以下目标函数（记住它总是需要包含训练损失和正则化）

![objective function](https://github.com/lidanxu/Common-code/blob/master/images/Tree_Boosting_obj.png)

### Additive Training
累积训练

我们首先要问的是树的参数是什么？ 你可以发现我们需要学习的是那些函数f<sub>i</sub>，每个函数都包含树的结构和叶子的分数。 这比传统的优化问题困难得多，在这个问题上你可以采取渐变gradient的方式。一次训练所有的树木并不容易。 相反，我们使用一个additive(添加/累积)策略：修复我们所学的内容，并一次添加一个新的树。 我们设步骤t写出预测值为 y<sup>^</sup> <sub>i</sub>(t)，所以我们有

![Additive Training 1](https://github.com/lidanxu/Common-code/blob/master/images/Additive_Training_1.png)

每棵树之间是有联系的，根据前面产生的树来产生后面的树。

那么每一步我们想要哪棵树？一个自然而然的事情是添加可以优化目标函数的树。

![Additive Training 2](https://github.com/lidanxu/Common-code/blob/master/images/Additive_Training_2.png)

如果我们考虑使用MSE（均方差）作为损失函数，那么目标函数变为下图形式：

![Additive Training 3](https://github.com/lidanxu/Common-code/blob/master/images/Additive_Training_3.png)

MSE的形式form是友好的，具有一阶项（通常称为残差residual）和二次项quadratic term。 对于其他形式的损失（例如logistic loss），并不能轻易获得如此好的目标函数形式。 所以在一般情况下，我们将损失函数进行泰勒展开到二阶：

![Additive Training 4](https://github.com/lidanxu/Common-code/blob/master/images/Additive_Training_4.png)

其中g<sub>i</sub> 和 h<sub>i</sub>定义如下：

![Additive Training 5](https://github.com/lidanxu/Common-code/blob/master/images/Additive_Training_5.png)

当我们移除所有的约束，t步的目标函数是：

![Additive Training 6](https://github.com/lidanxu/Common-code/blob/master/images/Additive_Training_6.png)

上式成为新tree的优化目标函数。 这个定义的一个重要优点是它只取决于g<sub>i</sub>和h<sub>i</sub>。 这就是xgboost支持自定义损失的功能。 我们可以使用完全相同的求解器，以g<sub>i</sub>和h<sub>i</sub>作为输入，来优化每个损失函数，包括逻辑回归logistic regression 和加权逻辑回归weighted logistic regression。

注：XGBoost每步迭代过程都有新的损失函数，根据新的损失函数选择/构造新树

### Model Complexity

我们已经介绍了训练过程，但是等等！还有一个重要的事情，regularization.我们需要定义树的复杂度Ω（f）
。为了做到这一点，让我们首先提炼refine树f（x）的定义为 

![Model Complexity 1](https://github.com/lidanxu/Common-code/blob/master/images/Model_Complexity_1.png)

上式中w是叶节点的分数的矢量，q是一个函数将每个数据点分配给相应的叶节点。T是叶节点的数量。 在XGBoost中，我们将复杂度定义为

![Model Complexity 2](https://github.com/lidanxu/Common-code/blob/master/images/Model_Complexity_2.png)

当然不止一种方法来定义复杂性，但是上式定义的复杂度在实践中运作良好。 正规化regularization是大多数tree模型/模块不那么谨慎对待或者容易被忽略的一部分。
这是因为传统的tree学习策略只强调提高impurity，而复杂性控制则是启发式的。 通过正式的定义，我们可以更好地了解我们正在学习什么，同时证实了regularization在实践中起到良好的作用。

### The Structure Score

这是求导的神奇部分。将树模型重新格式化之后，我们可以将第t个树模型的目标值表示如下：

![Structure Score 1](https://github.com/lidanxu/Common-code/blob/master/images/Structure_Score_1.png)

 其中I<sub>j</sub>={i|q(x<sub>i</sub>)=j}表示被分配给第j个叶节点的数据点索引的集合。注意在第二行中，我们改变求和的索引，因为同一叶节点上的所有数据点都得到相同的分数。我们可以通过定义G<sub>j</sub> =Σ<sub>i∈I<sub>j</sub></sub>g<sub>i</sub>和H<sub>j</sub> =Σ<sub>i∈I<sub>j</sub></sub>h<sub>i</sub>来进一步压缩目标函数的表达式：
 
 ![Structure Score 2](https://github.com/lidanxu/Common-code/blob/master/images/Structure_Score_2.png)
 
 在上式方程中，w<sub>j</sub>是相互独立的，方程中G<sub>j</sub>w<sub>j</sub> + 1/2*(H<sub>j</sub>+λ)w<sub>j</sub><sup>2</sup>是一个给定结构q(x)的二次型和最佳的w<sub>j</sub>，我们可以得到最佳的目标减少是：

 ![Structure Score 3](https://github.com/lidanxu/Common-code/blob/master/images/Structure_Score_3.png)
 
 上图中最后一个等式衡量树结构q(x)到底有多好。
 
 ![Structure Score 4](https://github.com/lidanxu/Common-code/blob/master/images/Structure_Score_4.png)
 如果这一切听起来有些复杂，那么看看上图，看看分数是如何计算的。 基本上，对于一个给定的树结构，我们把统计信息g<sub>i</sub>和h<sub>i</sub>推到/放到它们所属的叶子上，叠加叶子节点的分数并使用公式计算树的好坏。这个分数就像决策树中的杂质测量impurity measure一样，区别就是score计算需要考虑到模型的复杂性，而决策树的impurity measure计算不需要。

### Learn the tree structure

现在我们有了方法来衡量一棵树的表现，理想情况下我们会列举所有可能的树木并挑选最好的树木，但在实践中这是非常棘手的，所以我们将尽量一次优化树的一个层次（直译）。 具体来说，我们试图将一片叶子分成两片，得到的分数增量是

![Score it gain](https://github.com/lidanxu/Common-code/blob/master/images/Score_Gain.png)

这个公式可以分解为：1）新左叶上的得分2）新右叶上的得分3）原叶上的得分4）附加叶上的正则化。

在这里我们可以看到一个重要的事实：如果增益小于γ，我们最好不要增加那个分支。 这正是树型模型中的修剪技术！ 通过使用监督学习的原理，我们自然会想到这些技术背后的原理:)

对于真实有价值的数据，我们通常要寻找一个最佳的分割。 为了有效地做到这一点，我们把所有的实例按排序顺序排列，如下图所示。

![sorted instance](https://github.com/lidanxu/Common-code/blob/master/images/sorted_instance.png)

从左到右扫描足以计算所有可能的特征分割方案的结构分数structure score，并且可以有效地找到最佳特征分割方案。

## Final words on XGBoost

现在你已经知道了什么是boosted tree,你可能还要问，关于XGBoost的介绍在哪里？其实XGBoost就是本教程介绍的formal principle所激发的工具，或者说XGBoost是基于formal principle产生的tool. 更重要的是，它在系统优化systems optimization和机器学习原理principles in machine learning方面都得到了深入的发展、研究。 这个库的目标是推动机器计算局限的极限，以提供一个可扩展的，可移植的、准确的库。 请你尝试调用这个库，最重要的是向社区贡献你的智慧（代码，例子，教程）！

