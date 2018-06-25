# 量化策略

![](http://file.elecfans.com/web1/M00/55/79/pIYBAFssV_SAPOcSAACWBTome1c039.png)


# 量化(quantization)。
    对象：对权重量化，对特征图量化(神经元输出)，对梯度量化(训练过程中)
    过程：在inference网络前传，在训练过程(反传)
    一步量化(仅对权重量化)，
    两步量化(对神经元与特征图量化，第一步先对feature map进行量化，第二步再对权重量化)
    
    32位浮点和16位浮点存储的时候，
    第一位是符号位，中间是指数位，后面是尾数。
    英特尔在NIPS2017上提出了把前面的指数项共享的方法，
    这样可以把浮点运算转化为尾数的整数定点运算，从而加速网络训练。
![](https://github.com/Ewenwan/MVision/blob/master/CNN/Deep_Compression/img/flexpoint.jpg)

    分布式训练梯度量化：
![](https://github.com/Ewenwan/MVision/blob/master/CNN/Deep_Compression/img/gradient_quant.jpg)
    

    对权重数值进行聚类，
    量化的思想非常简单。
    CNN参数中数值分布在参数空间，
    通过一定的划分方法，
    总是可以划分称为k个类别。
    然后通过储存这k个类别的中心值或者映射值从而压缩网络的储存。

    量化可以分为
    Low-Bit Quantization(低比特量化)、
    Quantization for General Training Acceleration(总体训练加速量化)和
    Gradient Quantization for Distributed Training(分布式训练梯度量化)。

    由于在量化、特别是低比特量化实现过程中，
    由于量化函数的不连续性，在计算梯度的时候会产生一定的困难。
    对此，阿里巴巴冷聪等人把低比特量化转化成ADMM可优化的目标函数，从而由ADMM来优化。

    从另一个角度思考这个问题，使用哈希把二值权重量化，再通过哈希求解.

    用聚类中心数值代替原权重数值，配合Huffman编码，
    具体可包括标量量化或乘积量化。
    但如果只考虑权重自身，容易造成量化误差很低，
    但分类误差很高的情况。
    因此，Quantized CNN优化目标是重构误差最小化。
    此外，可以利用哈希进行编码，
    即被映射到同一个哈希桶中的权重共享同一个参数值。

    聚类例子：
        例如下面这个矩阵。

        1.2  1.3  6.1
        0.9  0.7  6.9
        -1.0 -0.9 1.0
        设定类别数k=3，通过kmeans聚类。得到：
        A类中心： 1.0 , 映射下标： 1
        B类中心： 6.5 , 映射下标： 2
        C类中心： -0.95 , 映射下标： 3

        所以储存矩阵可以变换为(距离哪个中心近，就用中心的下标替换)：
        1  1  2
        1  1  2
        3  3  1
        当然，论文还提出需要对量化后的值进行重训练，挽回一点丢失的识别率 
        基本上所有压缩方法都有损，所以重训练还是比较必要的。
        
# 1. 深度神经网络压缩 Deep Compression
    为了进一步压缩网络，考虑让若干个权值共享同一个权值，
    这一需要存储的数据量也大大减少。
    在论文中，采用kmeans算法来将权值进行聚类，
    在每一个类中，所有的权值共享该类的聚类质心，
    因此最终存储的结果就是一个码书和索引表。
    
    1.对权值聚类 
        论文中采用kmeans聚类算法，
        通过优化所有类内元素到聚类中心的差距（within-cluster sum of squares ）来确定最终的聚类结果.
        
    2. 聚类中心初始化 

        常用的初始化方式包括3种： 
        a) 随机初始化。
           即从原始数据种随机产生k个观察值作为聚类中心。 

        b) 密度分布初始化。
           现将累计概率密度CDF的y值分布线性划分，
           然后根据每个划分点的y值找到与CDF曲线的交点，再找到该交点对应的x轴坐标，将其作为初始聚类中心。 

        c) 线性初始化。
            将原始数据的最小值到最大值之间的线性划分作为初始聚类中心。 

        三种初始化方式的示意图如下所示： 

![](https://img-blog.csdn.net/20161026183710142)

    由于大权值比小权值更重要（参加HanSong15年论文），
    而线性初始化方式则能更好地保留大权值中心，
    因此文中采用这一方式，
    后面的实验结果也验证了这个结论。 
    
    3. 前向反馈和后项传播 
        前向时需要将每个权值用其对应的聚类中心代替，
        后向计算每个类内的权值梯度，
        然后将其梯度和反传，
        用来更新聚类中心，
        如图： 
        
![](https://img-blog.csdn.net/20161026184233327)

        共享权值后，就可以用一个码书和对应的index来表征。
        假设原始权值用32bit浮点型表示，量化区间为256，
        即8bit，共有n个权值，量化后需要存储n个8bit索引和256个聚类中心值，
        则可以计算出压缩率compression ratio: 
            r = 32*n / (8*n + 256*32 )≈4 
            可以看出，如果采用8bit编码，则至少能达到4倍压缩率。

[通过减少精度的方法来优化网络的方法总结](https://arxiv.org/pdf/1703.09039.pdf)


 
# 降低数据数值范围。
        其实也可以算作量化
        默认情况下数据是单精度浮点数，占32位。
        有研究发现，改用半精度浮点数(16位)
        几乎不会影响性能。谷歌TPU使用8位整型来
        表示数据。极端情况是数值范围为二值
        或三值(0/1或-1/0/1)，
        这样仅用位运算即可快速完成所有计算，
        但如何对二值或三值网络进行训练是一个关键。
        通常做法是网络前馈过程为二值或三值，
        梯度更新过程为实数值。

## 2. 二值量化网络 
[二值化神经网络介绍](https://blog.csdn.net/tangwei2014/article/details/55077172)

![](http://file.elecfans.com/web1/M00/55/79/pIYBAFssV_SAdU6BAACcvDwG5pU677.png)

    上图是在定点表示里面最基本的方法：BNN和BWN。
    在网络进行计算的过程中，可以使用定点的数据进行计算，
    由于是定点计算，实际上是不可导的，
    于是提出使用straight-through方法将输出的估计值直接传给输入层做梯度估计。
    在网络训练过程中会保存两份权值，用定点的权值做网络前向后向的计算，
    整个梯度累积到浮点的权值上，整个网络就可以很好地训练，
    后面几乎所有的量化方法都会沿用这种训练的策略。
    前面包括BNN这种网络在小数据集上可以达到跟全精度网络持平的精度，
    但是在ImageNet这种大数据集上还是表现比较差。


![](https://img-blog.csdn.net/20170214003827832?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGFuZ3dlaTIwMTQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

    二值化神经网络，是指在浮点型神经网络的基础上，
    将其权重矩阵中权重值(线段上) 和 各个 激活函数值(圆圈内) 同时进行二值化得到的神经网络。
        1. 一个是存储量减少，一个权重使用 1bit 就可以，而原来的浮点数需要32bits。
        2. 运算量减少， 原先浮点数的乘法运算，可以变成 二进制位的异或运算。
        
[Binarized Neural Networks BNN](https://arxiv.org/pdf/1602.02830.pdf)

    BNN的 激活函数值 和 权重参数 都被二值化了, 前向传播是使用二值，反向传播时使用全精度梯度。 
**二值化方法**

    1. 阈值二值化，确定性(sign()函数）
       x =   +1,  x>0
             -1,  其他
![](https://img-blog.csdn.net/20170214005016493?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGFuZ3dlaTIwMTQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)    
             
    2. 概率二值化随机（基于概率）两种二值化方式。
       x = +1,  p = sigmod(x) ,  
           -1,  1-p
![](https://img-blog.csdn.net/20170214005110619?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGFuZ3dlaTIwMTQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![](https://raw.githubusercontent.com/stdcoutzyx/Blogs/master/blogs2016/imgs_binarized/3.png)

    第二种方法虽然看起来比第一种更合理，但是在实现时却有一个问题，
    那就是每次生成随机数会非常耗时，所以一般使用第一种方法.
    
    
           
**训练二值化网络**

**前向传播时**

    对权重值W 和 激活函数值a 进行二值化
    Wk = Binary(Wb)   // 权重二值化
    Sk = ak-1 * Wb    // 计算神经元输出
    ak = BN(Sk, Ck)   // BN 方式进行激活
    ak = Binary(ak)   // 激活函数值 二值化

![](https://img-blog.csdn.net/20170214010139607?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGFuZ3dlaTIwMTQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**在反传过程时**

    使用sign函数时，
    对决定化方式中的Sign函数进行松弛化，即前传中是： 
![](https://img-blog.csdn.net/20170214005740059?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGFuZ3dlaTIwMTQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

    sign函数不可导，
    使用直通估计（straight-through estimator）(即将误差直接传递到下一层):
    反传中在已知q的梯度，对r求梯度时，Sign函数松弛为：
    gr=gq1|r|≤1
![](https://img-blog.csdn.net/20170214005816256?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGFuZ3dlaTIwMTQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
    
    即当r的绝对值小于等于1时，r的梯度等于q的梯度，否则r的梯度为0。 
    
    最后求得各层浮点型权重值对应的梯度和浮点型激活函数值对应的残差，
    然后用SGD方法或者其他梯度更新方法对浮点型的权重值进行更新，
    以此不断的进行迭代，直到loss不再继续下降。

    BNN中同时介绍了基于移位（而非乘法）的BatchNormailze和AdaMax算法。 
    实验结果： 
    在MNIST，SVHN和CIFAR-10小数据集上几乎达到了顶尖的水平。 
    在ImageNet在使用AlexNet架构时有较大差距（在XNOR-Net中的实验Δ=29.8%） 
    在GPU上有7倍加速.

**求各层梯度方式如下：**

![](https://img-blog.csdn.net/20170214005928789?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGFuZ3dlaTIwMTQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
    
**梯度更新方式如下：**

![](https://img-blog.csdn.net/20170214010005900?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGFuZ3dlaTIwMTQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
    
    

[BinaryConnect: Training Deep Neural Networks with binary weights](https://arxiv.org/pdf/1511.00363.pdf)

[论文笔记](https://blog.csdn.net/weixin_37904412/article/details/80618102)

[BinaryConnect 代码](https://github.com/Ewenwan/BinaryConnect)


[BWN(Binary-Weights-Networks) ](https://arxiv.org/pdf/1603.05279.pdf)

![](http://file.elecfans.com/web1/M00/55/79/pIYBAFssV_SAaYgnAACz9cXw6vE854.png)

    上图展示了ECCV2016上一篇名为XNOR-Net的工作，
    其思想相当于在做量化的基础上，乘了一个尺度因子，这样大大降低了量化误差。
    他们提出的BWN，在ImageNet上可以达到接近全精度的一个性能，
    这也是首次在ImageNet数据集上达到这么高精度的网络。
    

    BWN(Binary-Weights-Networks) 仅有参数二值化了，激活量和梯度任然使用全精度。XNOR-Net是BinaryNet的升级版。 
    主要思想： 
        1. 二值化时增加了缩放因子，同时梯度函数也有相应改变：
        W≈W^=αB=1n∑|W|ℓ1×sign(W)
        ∂C∂W=∂C∂W^(1n+signWα)

        2. XNOR-Net在激活量二值化前增加了BN层 
        3. 第一层与最后一层不进行二值化 
    实验结果： 
        在ImageNet数据集AlexNet架构下，BWN的准确率有全精度几乎一样，XNOR-Net还有较大差距(Δ=11%) 
        减少∼32×的参数大小，在CPU上inference阶段最高有∼58× 的加速。
        
[QNN](https://arxiv.org/pdf/1609.07061.pdf)

        对BNN的简单扩展，
        量化激活函数，
        有线性量化与log量化两种，
        其1-bit量化即为BinaryNet。
        在正向传播过程中加入了均值为0的噪音。 
        BNN约差于XNOR-NET（<3%），
        QNN-2bit activation 略优于DoReFaNet 2-bit activation


#### 二值约束低比特量化
![](http://file.elecfans.com/web1/M00/55/79/pIYBAFssV_WAdFmiAACFxVTKLmQ760.png)

    上图展示了阿里巴巴冷聪等人做的通过ADMM算法求解binary约束的低比特量化工作。
    从凸优化的角度，在第一个优化公式中，f(w)是网络的损失函数，
    后面会加入一项W在集合C上的loss来转化为一个优化问题。
    这个集合C取值只有正负1，如果W在满足约束C的时候，它的loss就是0；
    W在不满足约束C的时候它的loss就是正无穷。
    为了方便求解还引进了一个增广变量，保证W是等于G的，
    这样的话就可以用ADMM的方法去求解。
    
#### 哈希函数两比特缩放量化 BWNH
![](http://file.elecfans.com/web1/M00/55/79/pIYBAFssV_WAE7dRAACHJnpcRMk945.png)

    通过Hashing方法做的网络权值二值化工作。
    第一个公式是我们最常用的哈希算法的公式，其中S表示相似性，
    后面是两个哈希函数之间的内积。
    我们在神经网络做权值量化的时候采用第二个公式，
    第一项表示输出的feature map，其中X代表输入的feature map，W表示量化前的权值，
    第二项表示量化后输出的feature map，其中B相当于量化后的权值，
    通过第二个公式就将网络的量化转化成类似第一个公式的Hashing方式。
    通过最后一行的定义，就可以用Hashing的方法来求解Binary约束。


## 3. 三值化网络 
[Ternary Neural Networks TNN](https://arxiv.org/pdf/1609.00222.pdf)

    训练时激活量三值化，参数全精度 
    infernce时，激活量，参数都三值化（不使用任何乘法） 
    用FPGA和ASIC设计了硬件

[Ternary weight networks](https://arxiv.org/pdf/1605.04711.pdf)

    主要思想就是三值化参数（激活量与梯度精度），参照BWN使用了缩放因子。
    由于相同大小的filter，
    三值化比二值化能蕴含更多的信息，
    因此相比于BWN准确率有所提高。
    
    

[Trained Ternary Quantization  TTQ](https://arxiv.org/pdf/1612.01064.pdf)

    与TWN类似，
    只用参数三值化，
    但是正负缩放因子不同，
    且可训练，由此提高了准确率。
    ImageNet-18模型仅有3%的准确率下降。

#### 三值 矩阵分解和定点变换
![](http://file.elecfans.com/web1/M00/55/79/pIYBAFssV_aAHHAsAACFv5V6ARc330.png)

    借助了矩阵分解和定点变换的优势，
    对原始权值矩阵直接做一个定点分解，限制分解后的权值只有+1、-1、0三个值。
    将网络变成三层的网络，首先是正常的3×3的卷积，对feature map做一个尺度的缩放，
    最后是1×1的卷积，所有的卷积的操作都有+1、-1、0。
    

## 4. 二进制位量化网络 哈希函数的味道啊 
[ShiftCNN](http://cn.arxiv.org/pdf/1706.02393v1)

[博客](https://blog.csdn.net/shuzfan/article/details/77856900)

![](https://img-blog.csdn.net/20170905204744197?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc2h1emZhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


    一个利用低精度和量化技术实现的神经网络压缩与加速方案。
    
>最优量化

    量化可以看作用离散码本描述样本分布。 
    优化目标(最大概率准则)和优化方法(L1和L2正则化)通常导致了神经网络参数呈现中心对称的非均匀分布。
    因此，一个最佳的量化码本应当是一个非均匀分布的码本。 
    这也是为什么BinaryNet(-1,+1)、ternary quantization(-1,0,+1)这种策略性能不足的一个重要原因。
    
    需要注意的是，
    量化之前需要对参数进行范围归一化，
    即除以最大值的绝对值，这样保证参数的绝对值都小于1。
    该量化方法具有码本小、量化简单、量化误差小的优点。
    
>量化

    ShiftCNN所采用是一种相当巧妙的类似于残差量化的方法。

    完整的码本包含 N 个子码本。 
    每个码本包含 M=2^B−1 个码字，即每一个码字可以用 B bit 表示。 
    每个码本定义如下：

     Cn=0, ±2^−n+1, ±2^−n, …, ±2^−n−⌊M/2⌋+2
    假设 N=2，B=4，则码本为

    C1=0, ±2^−1, ±2^−2, ±2^−3, ±2^−4, ±2^−5, ±2^−6
    C2=0, ±2^−2, ±2^−3, ±2^−4, ±2^−5, ±2^−6, ±2^−7
    
    于是，每一个权重都可以使用 N*B bit 的索引通过下式求和计算得到：
    wi' = sum(Cn[id(n)])

>卷积计算

    卷积计算的复杂度主要来自乘法计算。
    ShiftCNN采用简单的移位和加法来实现乘法，从而减少计算量。
    比如计算 y=wx, 而 w 通过量化已经被我们表示成了,
    类似于 2^−1 + 2^−2 + 2^−3 这种形式，
    于是 y = x>>1 + x>>2 + x>>3 


## 5. 固定点多比特量化
[Fixed Point Quantization of Deep Convolutional Networks ](https://arxiv.org/pdf/1511.06393.pdf)

    r=S(q-Z) 其中q为定点结果，r为对应的浮点数据，S和Z分别为范围和偏移参数
## 6. Quantized Convolutional Neural Networks for Mobile Devices  8bit
[Quantized Convolutional Neural Networks for Mobile Devices](https://arxiv.org/pdf/1512.06473.pdf)


## 7.

[Ristretto: Hardware-Oriented Approximation of Convolutional Neural Networks](https://arxiv.org/pdf/1605.06402.pdf)


[Fixed-point Factorized Networks](https://arxiv.org/pdf/1611.01972.pdf)

[Deep Learning with Low Precision by Half-wave Gaussian Quantization](https://arxiv.org/pdf/1702.00953.pdf)

[INCREMENTAL NETWORK QUANTIZATION: TOWARDS LOSSLESS CNNS WITH LOW-PRECISION WEIGHTS](https://arxiv.org/pdf/1702.03044.pdf)


[Network Sketching: Exploiting Binary Structure in Deep CNNs ](https://arxiv.org/pdf/1706.02021.pdf)


[Training Quantized Nets: A Deeper Understanding ](https://arxiv.org/pdf/1706.02379.pdf)


[Balanced Quantization: An Effective and Efficient Approach to Quantized Neural Networks](https://arxiv.org/pdf/1706.07145.pdf)


[Performance Guaranteed Network Acceleration viaHigh-Order Residual Quantization](https://arxiv.org/pdf/1708.08687.pdf)

[Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference](https://arxiv.org/abs/1712.05877)


[ALTERNATING MULTI-BIT QUANTIZATION FOR RECURRENT NEURAL NETWORKS](https://arxiv.org/pdf/1802.00150.pdf)


[Deep Neural Network Compression with Single and Multiple Level Quantization](https://arxiv.org/pdf/1803.03289.pdf)

[Expectation Backpropagation: Parameter-Free Training of Multilayer Neural Networks with Continuous or Discrete Weights](http://papers.nips.cc/paper/5269-expectation-backpropagation-parameter-free-training-of-multilayer-neural-networks-with-continuous-or-discrete-weights.pdf)