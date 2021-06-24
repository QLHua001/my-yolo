# 网络结构

![image-20210624103753688](/yolo.assets/image-20210624103753688.png)



# loss function

$loss function = \lambda_{coord}\sum^{S^2}_{i=0}\sum_{j=0}^{B}\mathbb{I}^{obj}_{ij}\Big[(x_i-\hat{x_i})^2 + (y_i - \hat{y_i})^2\Big]\\ \quad \quad \quad \quad \quad \quad  + \lambda_{coord}\sum_{i=0}^{S^2}\sum_{j=0}^{B}\mathbb{I}_{ij}^{obj}\Big[ (\sqrt{w_i} - \sqrt{\hat{w}_i})^2 + (\sqrt{h_i} - \sqrt{\hat{h}_i})^2 \Big] \\ \quad \quad \quad \quad \quad \quad + \sum_{i=0}^{S^2}\sum_{j=0}^{B}\mathbb{I}^{obj}_{ij}(C_i-\hat{C}_i)^2\\ \quad \quad \quad \quad \quad \quad + \lambda_{noobj}\sum_{i=0}^{S^2}\sum_{j=0}^{B}\mathbb{I}_{ij}^{noobj}(C_i - \hat{C}_i)^2 \\ \quad \quad \quad \quad \quad \quad + \sum_{i=0}^{S^2}\mathbb{I}_i^{obj}\sum_{c\in classes} (p_i(c) - \hat{p}_i(c))^2$



![image-20210624113203044](/yolo.assets/image-20210624113203044.png)

==（请注意：图片解释有误，仅用于理解 loss 设计）==

损失函数的设计目标就是让坐标（x,y,w,h），confidence，classification 这个三个方面达到很好的平衡。简单的全部采用了sum-squared error loss来做这件事会有以下不足： a) 8维的localization error和20维的classification error同等重要显然是不合理的； b) ==如果一个网格中没有object（一幅图中这种网格很多），那么就会将这些网格中的box的confidence push到0，相比于较少的有object的网格，这种做法是overpowering的，这会导致网络不稳定甚至发散。== 解决方案如下：

- 更重视8维(2个bounding box)的坐标预测，给这些损失前面赋予更大的loss weight, 记为$\lambda_{coord}$,在pascal VOC训练中取5。

- 对没有object的bbox的confidence loss，赋予小的loss weight，记为 $\lambda_{noobj}$ ，在pascal VOC训练中取0.5。

- 有object的bbox的confidence loss和类别的loss 的loss weight正常取1。

- 对不同大小的bbox预测中，相比于大bbox预测偏一点，小box预测偏一点更不能忍受。而sum-square error loss中对同样的偏移loss是一样。 ==为了缓和这个问题，作者用了一个比较取巧的办法，就是将box的width和height取平方根代替原本的height和width。== 如下图：small bbox的横轴值较小，发生偏移时，反应到y轴上的loss（下图绿色）比big box(下图红色)要大。

  <img src="/yolo.assets/image-20210624113440836.png" alt="image-20210624113440836" style="zoom:50%;" />

- 一个网格预测多个bounding box，在训练时我们希望每个object（ground true box）只有一个bounding box专门负责（一个object 一个bbox）。具体做法是与ground true box（object）的IOU最大的bounding box 负责该ground true box(object)的预测。这种做法称作bounding box predictor的specialization(专职化)。每个预测器会对特定（sizes,aspect ratio or classed of object）的ground true box预测的越来越好。（个人理解：IOU最大者偏移会更少一些，可以更快速的学习到正确位置）

# 测试

**Test的时候**，**每个网格预测的class信息($ Pr(Class_i | Object)$)**和**bounding box预测的confidence信息( $Pr(Object)*IOU_{pred}^{truth}$)** 相乘，就得到每个bounding box的class-specific confidence score。

$Pr(Class_i|Object)*Pr(Object)*IOU_{truth}^{pred} = Pr(Class_i)*IOU_{truth}^{pred}$

等式左边第一项就是每个网格预测的类别信息，第二三项就是每个bounding box预测的confidence。这个乘积即encode了预测的box属于某一类的概率，也有该box准确度的信息。

<img src="/yolo.assets/image-20210624114640914.png" alt="image-20210624114640914" style="zoom: 50%;" />

<img src="/yolo.assets/image-20210624114738445.png" alt="image-20210624114738445" style="zoom:50%;" />

# 不足

- YOLO对相互靠的很近的物体（挨在一起且中点都落在同一个格子上的情况），还有很小的群体 检测效果不好，这是因为一个网格中只预测了两个框，并且只属于一类。
- 测试图像中，当同一类物体出现的不常见的长宽比和其他情况时泛化能力偏弱。
- 由于损失函数的问题，定位误差是影响检测效果的主要原因，尤其是大小物体的处理上，还有待加强。