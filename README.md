# Motion prediction use CapsuleNet
> Reference:  https://arxiv.org/pdf/2103.01644v1.pdf
>
> Reference: https://arxiv.org/abs/1710.09829
> 
> Reference: https://github.com/Riroaki/CapsNet

### 胶囊网络
在手写识别问题中，使用的三层网络结构如下图：
![image](https://github.com/AliceNing/CapsuleNet/blob/main/images/caps_mnist.png)
假如输入的图片大小为28\*28，通道数为1。  
**第一层**是包含激活的二维卷积层，输出的数据维度是为256\*20\*20。  
**第二层**是PrimaryCaps胶囊层，使用二维卷积处理，并且将输出分成32组，每组8个通道，卷积核的大小为9，移动步长是2。使用squash激活函数来使短向量趋于0，长向量趋于1，最后输出的数据维度是32\*8\*6\*6.  
![image](https://github.com/AliceNing/CapsuleNet/blob/main/images/squash.png)  
**第三层**是DigitCaps胶囊层，使用了论文提出的动态路由的方法，计算过程如下图，输入x，先经过一个线性变换u^，b初始化为0，（b经过softmax之后为c，s是u^和c的积，s经过squash激活函数得到v，u^和v的积累加到b上。）括号内部分迭代r次，即得到最后的输出10\*16。  
![image](https://github.com/AliceNing/CapsuleNet/blob/main/images/Routing_algorithm.png)  
DigitCaps的输出经过norm即可得到预测的结果logits(10*16)和target(1*16)，再把这个target经过三层的全连接层重构回784维(28\*28)。  
**解码器**如下图：输入是10*16，除了target，其他都mask掉了。  
![image](https://github.com/AliceNing/CapsuleNet/blob/main/images/cap_decoder.png)  
**损失函数部分**  
包括两部分Margin_loss和reconstruction_loss  
重构loss是根据decoder的重构结果和原始的像素值使用MSEloss(求差，平方，再求和)计算得到。  
Margin_loss计算如下，用输出向量的长度表示概率，并且加入了上下界，使正样本的概率大于0.9，负样本的概率小于0.1：  
![image](https://github.com/AliceNing/CapsuleNet/blob/main/images/cap_loss.png)  

### 使用胶囊网络进行轨迹预测

把每个位置的scence提取成不同的层，包含不同的特征，，同时加入一个包含agent的层，然后把这些扔到胶囊网络里进行学习

使用的特征：速度，加速度，航向角

**输入**：agent的历史状态矩阵S（包括速度，加速度，航向角），地图数据M（大小：时刻长度*提取出的层数）

 agent的历史位置P（x,y），Lt用Pt来定义和抽取，（方法如公式2所示，lamda是提取的范围）

 最终模型的输入由Pt和S来确定，M包含了所有时刻和层数的L

**输出**：一个向量

**模型框架**：类似胶囊网络在手写识别数据上的结构。

1.输入是每个map chunk

2.base卷积：每个层经过相同卷积？k9s2o64+ ELU

3.low低级胶囊网络：包括4组，每组有两个卷积层，两层之间使用squash

4.高层级胶囊网络：

5.预测部分：每个状态St，经过一个全连接层，变128维，和胶囊网络拼接起来成时间长度\*256的矩阵，使用LSTM进行编码（隐层128维），最后使用一个全连接层进行解码，变成一个2(x,y)\*时间步的预测结果

6.损失函数：L=aL1+bL2，L1是平均绝对值误差，L2是平方差误差，ab都为1

缺点：没有考虑到不同agent之间的交互信息？