# CVNet:为图像检索重排序阶段提供相关验证
![image](D0E7807768184F9DA75891F00EDFBCBE)
- 论文名字：《Correlation Verification for Image Retrieval》
- 论文地址：https://arxiv.org/abs/2204.01458v1
- 代码地址：https://github.com/sungonce/CVNet

## 引言
图像检索的目的是根据图像与给定查询图的相似性对图像数据库进行排序。一个图像检索的框架通常包含两个组件：1.由全局描述子进行匹配的全局检索。2.局部特征匹配后的几何验证。它们相互补充，其中，全局检索在整个数据集中快速执行粗检索，而几何验证仅对前top-k的候选对象执行精确评估，从而对粗检索结果进行重新排序。

本篇工作侧重于重排序的研究，将重排序和全局特征提取在一起，提出一种新的端到端的图像检索重排序网络--相关验证网络（Correlation Verfication Networks, CVNet），以取代几何验证的角色。该网络通过卷积方式利用密集的特征相关性，直接评估语义和几何关系。具体来说，网络由深度堆叠的4D卷积层组成，逐步将密集的特征相关性压缩为图像相似度，同时从不同的图像对学习不同的几何匹配模式。为了确保鲁棒性（面对大尺度不同的问题），该网络将每幅图像的单尺度特征扩展成特征金字塔，形成特征金字塔之间的跨尺度相关性。同时为了挖掘困难样本，作者通过curriculum learning，将训练阶段使用困难负样本挖掘（hard negative mining）和捉迷藏（Hide-and-Seek）策略,旨在通过聚焦于困难样本的同时而不失去普通样本的普遍性来提高整体性能。

该工作在多个公开数据集（POxford、RParis、GLDv2）上达到SOTA水平，代码公开于：https://github.com/sungonce/CVNet.

## 动机&贡献
### 动机
- 针对图像检索中的重排序（re-ranking）阶段，在局部特征匹配之后进行几何验证（Geometric verification）是非常重要的。但是由于其verify-after-matching的结构，几何验证仅基于稀疏的特征进行验证，且这些特征还基于阈值进行筛选。如图1（a）所示：
![image](5CCFCACC1D8B4A8CA815BF1B5E5449BF)
- 此外，几何验证不能处理多尺度问题。因此，一些工作试图通过图像金字塔操作进行重复推理来提取多尺度的局部特征来解决尺度问题，但是这是一个非常耗时的过程。
- 图像检索存在困难样本问题。


## 贡献
- 提出相关验证匹配网络（Correlation Verification Networks, CVNet），它是一种基于密集特征相关性直接预测图像对相似度的重排序网络。
- 为了取代昂贵的多尺度推理，作者在模型中构造了跨尺度关联，并使用单个推理来执行跨尺度匹配。
- 提出curriculum learning，建议使用hard negative mining和Hide-and-Seek策略来处理困难样本，并且不失去一般性。

# 方法
![image](C2D3E610BCB344348306562FCFCC4F7F)
如图3所示，CVNet由两部分组成：CVNet-Global(灰色区域)和CVNet-Rerank(绿色区域)。其中，CVNet-Global负责提取特征作为图像的全局描述子，CVNet-Rerank负责对特征进行重排序，进一步精化特征。最终，在推理阶段，使用全局描述子作为图片的特征表示。


## 全局骨干网络（CVNet-Global）
### Structure
![image](311CF8471E514163B34CBFC138EA7521)
受到论文MoCo的启发，CVNet-Global采用动量对比结构（momentum-contrastive structrue）。具体来说，网络采用两个分支，即Global Network:` <img src="https://www.zhihu.com/equation?tex=f" alt="f" class="ee_img tr_noresize" eeimg="1"> `
（使用梯度更新）和Momentum Network:` <img src="https://www.zhihu.com/equation?tex=\bar{f}" alt="\bar{f}" class="ee_img tr_noresize" eeimg="1"> `（使用动量更新）。细节如下：
- 输入：两个分支每次迭代只接收单张图片` <img src="https://www.zhihu.com/equation?tex=\mathbf{I} \in \mathbb{R}^{3 \times H \times W}" alt="\mathbf{I} \in \mathbb{R}^{3 \times H \times W}" class="ee_img tr_noresize" eeimg="1"> `作为输入。
- ` <img src="https://www.zhihu.com/equation?tex=f" alt="f" class="ee_img tr_noresize" eeimg="1"> `和` <img src="https://www.zhihu.com/equation?tex=\bar{f}" alt="\bar{f}" class="ee_img tr_noresize" eeimg="1"> `基于ResNet,` <img src="https://www.zhihu.com/equation?tex=f_{i}" alt="f_{i}" class="ee_img tr_noresize" eeimg="1"> `表示第` <img src="https://www.zhihu.com/equation?tex=i" alt="i" class="ee_img tr_noresize" eeimg="1"> `个ResBlock。
- 使用GeM代替GAP。 
- 在池化层之后接whitening FC层以及L2归一化。

因此，输入一张query，经过Gloabl Network输出一个查询图的全局描述子` <img src="https://www.zhihu.com/equation?tex=d_{g}^{q}" alt="d_{g}^{q}" class="ee_img tr_noresize" eeimg="1"> `，该特征作为分类损失的输入。同时，在Momentum Network维护一个队列` <img src="https://www.zhihu.com/equation?tex=\mathbf{Q} \in\left\{\overline{\mathbf{d}}_{g}^{i}\right\}_{i=1}^{K}" alt="\mathbf{Q} \in\left\{\overline{\mathbf{d}}_{g}^{i}\right\}_{i=1}^{K}" class="ee_img tr_noresize" eeimg="1"> `,其中` <img src="https://www.zhihu.com/equation?tex=\bar{d}_{g}^{p}" alt="\bar{d}_{g}^{p}" class="ee_img tr_noresize" eeimg="1"> `表示Positive Momentum Global descriptor。该队列保存每次迭代的动量全局描述子，并且把它们作为对比损失的输入。

*注：队列的更新原则遵从先进先出原则，即每次更新，会将最开始的那个特征出列。*

### Traing Objective
**分类损失**：

![image](A7C77F6939D54226B44E3EEBEA97524C)

公式1为**CurricularFace-maigined classification loss**。

其中：
- ` <img src="https://www.zhihu.com/equation?tex=\mathbf{d}_{g}^{q}" alt="\mathbf{d}_{g}^{q}" class="ee_img tr_noresize" eeimg="1"> `表示查询图` <img src="https://www.zhihu.com/equation?tex=\mathbf{I}_{q}" alt="\mathbf{I}_{q}" class="ee_img tr_noresize" eeimg="1"> `经过gloabl network输出的全局描述子。
- ` <img src="https://www.zhihu.com/equation?tex=\mathbf{W}" alt="\mathbf{W}" class="ee_img tr_noresize" eeimg="1"> `表示类别权重
- ` <img src="https://www.zhihu.com/equation?tex=\mathbf y_{g}" alt="\mathbf y_{g}" class="ee_img tr_noresize" eeimg="1"> `表示ground-truth类别
- ` <img src="https://www.zhihu.com/equation?tex=\mathbb{1}_{q}^{i}" alt="\mathbb{1}_{q}^{i}" class="ee_img tr_noresize" eeimg="1"> `表示指示函数，指第i个类别` <img src="https://www.zhihu.com/equation?tex=y_{i}" alt="y_{i}" class="ee_img tr_noresize" eeimg="1"> `是否等于` <img src="https://www.zhihu.com/equation?tex=y_{g}" alt="y_{g}" class="ee_img tr_noresize" eeimg="1"> `。是为1，否为0.
- C是一个带margin的余弦相似度计算函数。
- N表示类别数量
- τ表示缩放参数。

该公式迫使当前特征与gt靠近。

*注：CurricularFace-maigined classification loss详细解释见论文《Curricularface: adaptive curriculum learning loss for deep
face recognition》*

**动量对比损失**：

![image](DA7B541173134A27A7D965DE60C0A565)

公式2表示动量对比损失。

其中：
- ` <img src="https://www.zhihu.com/equation?tex=\mathbf{d}_{g}^{q}" alt="\mathbf{d}_{g}^{q}" class="ee_img tr_noresize" eeimg="1"> `表示查询图全局描述子。
- ` <img src="https://www.zhihu.com/equation?tex=\overline{\mathbf{d}}_{g}^{p}" alt="\overline{\mathbf{d}}_{g}^{p}" class="ee_img tr_noresize" eeimg="1"> `表示与查询图同类别的图片经过momentum network输出的全局描述子。
- τ表示缩放参数
- ` <img src="https://www.zhihu.com/equation?tex=\overline{\mathcal{C}}\left(\mathbf{d}_{q}^{q} \cdot \overline{\mathbf{d}}_{q}^{p}, 1\right) " alt="\overline{\mathcal{C}}\left(\mathbf{d}_{q}^{q} \cdot \overline{\mathbf{d}}_{q}^{p}, 1\right) " class="ee_img tr_noresize" eeimg="1"> `表示两个描述子的余弦相似度。` <img src="https://www.zhihu.com/equation?tex=\overline{\mathcal{C}}" alt="\overline{\mathcal{C}}" class="ee_img tr_noresize" eeimg="1"> `与` <img src="https://www.zhihu.com/equation?tex=\mathcal{C}" alt="\mathcal{C}" class="ee_img tr_noresize" eeimg="1"> `相同，但用C单独更新其移动平均参数。
- ` <img src="https://www.zhihu.com/equation?tex=P(q) \text / N(q)" alt="P(q) \text / N(q)" class="ee_img tr_noresize" eeimg="1"> `分别表示正样本集合/负样本集合。与q同类别的样本为正，反之为负。

该公式希望在嵌入空间上，正样本的距离越近越好，负样本的距离越远越好。

**总损失**：

![image](E168F4E793D5456F9AF7FAEF37B6925D)

总损失为两个损失函数的加权和。

*注：优化器只更新gloabl network` <img src="https://www.zhihu.com/equation?tex= f" alt=" f" class="ee_img tr_noresize" eeimg="1"> `。momentum network使用动量` <img src="https://www.zhihu.com/equation?tex=\eta" alt="\eta" class="ee_img tr_noresize" eeimg="1"> `进行更新。*


## 重排序网络 (CVNet-Rerank)
![image](F4A9CF117E1348CBA43B170883630F88)

如图3绿色区域所示，重排序网络接收来自全局网络的局部特征图作为输入` <img src="https://www.zhihu.com/equation?tex=\left(\mathbf{F}_{q}, \mathbf{F}_{k}\right)" alt="\left(\mathbf{F}_{q}, \mathbf{F}_{k}\right)" class="ee_img tr_noresize" eeimg="1"> `，即图片` <img src="https://www.zhihu.com/equation?tex=\left(\mathbf{I}_{q}, \mathbf{I}_{k}\right)" alt="\left(\mathbf{I}_{q}, \mathbf{I}_{k}\right)" class="ee_img tr_noresize" eeimg="1"> `经过全局网络中间层输出的特征图。该网络利用深度堆叠的4D卷积层逐步压缩特征相关性，并利用分类器对图像相关性进行预测。

### Cross-scale Correlation Construction
- 动机：由于图像检索必须对尺度差具有鲁棒性，几种使用局部特征进行检索的方法使用图像金字塔进行多次推理来构建一个多尺度的局部特征集。而这是一个非常耗时间的操作。
- 解决方案：为了在单次推理时捕获多尺度信息。本文提出将提取的特征图扩展到一个多尺度的特征金字塔，以捕获模型内不同尺度的语义信息。
- 具体实施：给出一对图（query/key image）` <img src="https://www.zhihu.com/equation?tex=\mathbf{I}_{q}, \mathbf{I}_{k} \in \mathbb{R}^{3 \times H \times W}" alt="\mathbf{I}_{q}, \mathbf{I}_{k} \in \mathbb{R}^{3 \times H \times W}" class="ee_img tr_noresize" eeimg="1"> `,经过全局网络` <img src="https://www.zhihu.com/equation?tex=f" alt="f" class="ee_img tr_noresize" eeimg="1"> `，提取中间层的特征图` <img src="https://www.zhihu.com/equation?tex=\mathbf{F}_{q}, \mathbf{F}_{k} \in \mathbb{R}^{C_{l} \times H_{l} \times W_{l}}" alt="\mathbf{F}_{q}, \mathbf{F}_{k} \in \mathbb{R}^{C_{l} \times H_{l} \times W_{l}}" class="ee_img tr_noresize" eeimg="1"> `。然后构建一个特征金字塔` <img src="https://www.zhihu.com/equation?tex=\left\{\mathbf{F}^{s}\right\}_{s=1}^{S}" alt="\left\{\mathbf{F}^{s}\right\}_{s=1}^{S}" class="ee_img tr_noresize" eeimg="1"> `，其中` <img src="https://www.zhihu.com/equation?tex=S" alt="S" class="ee_img tr_noresize" eeimg="1"> `是尺度的数量，用` <img src="https://www.zhihu.com/equation?tex=1 / \sqrt{2}" alt="1 / \sqrt{2}" class="ee_img tr_noresize" eeimg="1"> `作为比例系数反复调整特征图` <img src="https://www.zhihu.com/equation?tex=F" alt="F" class="ee_img tr_noresize" eeimg="1"> `的大小。金字塔的每一层都通过按比率的3x3卷积将每一层的通道数调整到` <img src="https://www.zhihu.com/equation?tex=C_{l}^{\prime}" alt="C_{l}^{\prime}" class="ee_img tr_noresize" eeimg="1"> `。

如下图所示，在本文中，S=3. 

![image](166FBDA9A9F343889A7489313612A39A)

当获得query特征金字塔` <img src="https://www.zhihu.com/equation?tex=\left\{\mathbf{F}_{q}^{s}\right\}_{s=1}^{S}" alt="\left\{\mathbf{F}_{q}^{s}\right\}_{s=1}^{S}" class="ee_img tr_noresize" eeimg="1"> `以及key特征金字塔` <img src="https://www.zhihu.com/equation?tex=\left\{\mathbf{F}_{k}^{s}\right\}_{s=1}^{S}" alt="\left\{\mathbf{F}_{k}^{s}\right\}_{s=1}^{S}" class="ee_img tr_noresize" eeimg="1"> `之后，作者利用余弦相似度和ReLu函数计算大小为` <img src="https://www.zhihu.com/equation?tex=S^{2}" alt="S^{2}" class="ee_img tr_noresize" eeimg="1"> `的四维跨尺度的相关集合` <img src="https://www.zhihu.com/equation?tex=\left\{C_{q k}^{s_{q}, s_{k}}\right\}^{(S, S)}\left(s_{q}, s_{k}\right)=(1,1)" alt="\left\{C_{q k}^{s_{q}, s_{k}}\right\}^{(S, S)}\left(s_{q}, s_{k}\right)=(1,1)" class="ee_img tr_noresize" eeimg="1"> `。

![image](A2439116D1F4429BABFFE05CE035DCA9)

具体计算如公式4所示，其中` <img src="https://www.zhihu.com/equation?tex=\mathbf{P}_{q} \text { and } \mathbf{p}_{k}" alt="\mathbf{P}_{q} \text { and } \mathbf{p}_{k}" class="ee_img tr_noresize" eeimg="1"> `表示每个特征图上的像素位置。ReLu函数将特征点的相似度变为0-1之间的概率。最后，通过线性插值将所有相似度相关矩阵大小变为` <img src="https://www.zhihu.com/equation?tex=H_{l}\times  W_{l}" alt="H_{l}\times  W_{l}" class="ee_img tr_noresize" eeimg="1"> `，堆叠所有的相关矩阵，构建一个跨尺度的相关集合` <img src="https://www.zhihu.com/equation?tex=C_{qk}^{0}\in  \mathbb{R} ^{S^{2} \times H_{l} \times W_{l}\times H_{l}\times W_{l}}" alt="C_{qk}^{0}\in  \mathbb{R} ^{S^{2} \times H_{l} \times W_{l}\times H_{l}\times W_{l}}" class="ee_img tr_noresize" eeimg="1"> `。

你可以理解为红色的3个特征图分别与蓝色的3个特征图计算像素级别的相似度。

### 4D Correlation Encoder

![image](C9A231F21A0745F7B32F74BDB0AB4C1F)

如图所示，相关编码器接收5维数据：跨尺度的相关集合` <img src="https://www.zhihu.com/equation?tex=C_{qk}^{0}\in  \mathbb{R} ^{S^{2} \times H_{l} \times W_{l}\times H_{l}\times W_{l}}" alt="C_{qk}^{0}\in  \mathbb{R} ^{S^{2} \times H_{l} \times W_{l}\times H_{l}\times W_{l}}" class="ee_img tr_noresize" eeimg="1"> `作为输入，随后将其逐渐压缩为一个二分类` <img src="https://www.zhihu.com/equation?tex=Z_{QK}=\left \{ z_{0},z_{1} \right \}∈\mathbb{R}  ^{2} " alt="Z_{QK}=\left \{ z_{0},z_{1} \right \}∈\mathbb{R}  ^{2} " class="ee_img tr_noresize" eeimg="1"> `。编码器由4D卷积块堆叠而成（B1,B2,B3,B4）,随后接一个4D全局平均池化层和2层MLP层。4D卷积块细节如图4所示。

![image](1AF087EEC1D14E8BAE01759F3E7864ED)
每个4D卷积块由 (卷积层+GN+Relu) 组成，并且每一层（B1、B2、B3、B4）的最后一个块使用步长为2的卷积进行下采样，形成不同尺度的相关矩阵。每一层4D卷积块的数量根据尺度不同而堆叠不同数量的卷积块。如B1一个4D卷积块，B2两个4D卷积块，以此类推。B4不进行下采样。因此，
` <img src="https://www.zhihu.com/equation?tex=C_{qk}^{0}\in  \mathbb{R} ^{S^{2} \times H_{l} \times W_{l}\times H_{l}\times W_{l}}" alt="C_{qk}^{0}\in  \mathbb{R} ^{S^{2} \times H_{l} \times W_{l}\times H_{l}\times W_{l}}" class="ee_img tr_noresize" eeimg="1"> `经过相关掩码器，输出` <img src="https://www.zhihu.com/equation?tex=C_{qk}^{4}\in  \mathbb{R} ^{128 \times H_{l}/8 \times W_{l}/8\times H_{l}/8\times W_{l}/8}" alt="C_{qk}^{4}\in  \mathbb{R} ^{128 \times H_{l}/8 \times W_{l}/8\times H_{l}/8\times W_{l}/8}" class="ee_img tr_noresize" eeimg="1"> `。

利用这种4D卷积金字塔结构，跨尺度特征相关集合被编码为细粒度相关线索` <img src="https://www.zhihu.com/equation?tex=C_{qk}^{1:4}" alt="C_{qk}^{1:4}" class="ee_img tr_noresize" eeimg="1"> `。最后做一个二分类，进行相似度预测，输出` <img src="https://www.zhihu.com/equation?tex=Z_{qk}" alt="Z_{qk}" class="ee_img tr_noresize" eeimg="1"> `,判断两张图片是否属于同一类。

### Training Objective

![image](CD20C800B54D40EFB359795F0DB63C0B)
![image](C634A42241D646779DB53D505C65E9F6)

重排序网络的损失函数如公式5，6所示。其中` <img src="https://www.zhihu.com/equation?tex=\mathbb{I} _{q}^{k} " alt="\mathbb{I} _{q}^{k} " class="ee_img tr_noresize" eeimg="1"> `为指示函数，表示k图片是否与q图像属于相同类别。是为1，否为0。q,p,n:分别表示query,positive,negative。

### Training with Hard Samples
重排序是在第一眼上去相似的图像上进行的，因此它必须对困难样本具有鲁棒性。
![image](EA6620169D4B44AF948FD720FE6F3038)
![image](4CD6585343CF490D8043299490274C4A)

图5展示了查询图与困难负样本，它们乍一看很相似，但它们存在不同之处，属于不同类别。作者提出在课程学习（curriculum learning）方式中应用困难负样本挖掘（hard negative mining）和捉迷藏（Hide-and-Seek augmentation）策略来训练重排序网络，使其在专注于困难样本的同时，在正常情况下不失去一般性的原则下做出更准确的预测。

**Hard negative mining**.通过训练好的模型对数据集中所有图片提取全局特征描述子。按照全局特征描述子匹配得分，取前10个负样本。图5展示了一些样本。

**Hide-and-Seek**.

![image](69FF5342C23241B68A7121A820CA0F37)

随机失活图像区域，模拟遮挡的情景。让模型不仅关注那些高响应区域，同时关注那些平凡的区域，增加模型鲁棒性。

实际操作是随机失活特征图的某些像素，由于特征图是原始图片的某个尺度的下采样，因此，特征图上的一个像素对应原图的某个区域。如图6所示。

**Curriculum learning**.为了防止困难样本对早期的学习造成干扰，作者在课程学习中应用困难负样本挖掘和捉迷藏策略。具体来说，不是从一开始就关注困难样本，而是选择困难的比例和捉迷藏的概率随着学习的进展逐步增加。

## 实验结果展示

![image](4067C88CB17A4FB487F20CA22C450E83)

表1展示了该方法在公开数据集上与其他SOTA方法的比对。其结果高于现有方法。

![image](043E35D39B0F4CA9BA117477C1EA191B)
表2表3展示了与其他re-ranking方法的比对结果。

## 消融实验
![image](E56B9E4F933847329CDD7E3BFFE11637)

- (a)：# 表示重排序的样本数量，作者分别在不同数量的样本下应用跨尺度相关进行重排序。结果摆明，增加CSC，可以提高检索map，且重排序样本越多，效果越好。
- (b): 比对了CVNet-Global阶段两个损失函数带来的影响。
- (c)：比对了困难样本采样以及捉迷藏策略带来的提升。
- (d)：比较了与其他方法的特征提取速度以及内存占用。