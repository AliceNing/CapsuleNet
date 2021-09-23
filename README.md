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

主要思想：把每个位置的scence提取成不同的层，包含不同的特征，加入一个包含agent的层，然后把这些扔到胶囊网络里进行学习。  

**输入**：agent的历史状态矩阵S（包括速度，加速度，航向角），St=[Vtx,Vty,atx,aty,yaw].    
       地图数据M=[L0,L1……,lt]，包含t个时刻的map_chunk组，每个时刻的局部map会抽象出n层，Lt=[Lt0,Lt1,Lt2,……,Ltn]，其中包含一个agent层。   
       agent的历史位置Pt=(xt,yt)，Lt用Pt来定义和抽取，每十米抽30个像素，并且从RGB转成灰度图，主要关注 road segment, drivable area, lane ， walkway  和 agent部分。  
**输出**：预测的结果  
![image](https://github.com/AliceNing/CapsuleNet/blob/main/images/map_chunk.png)  


**模型框架**：
![image](https://github.com/AliceNing/CapsuleNet/blob/main/images/motion_caps.png)  

输入:M是map chunk，包含t个时刻的map_chunk组，每个时刻的局部Map会抽象出n层，其中包含一个agent层。  
       S是agent的t个时刻里的状态矩阵，描述其速度，加速度和航向角  

第一层是base卷积：每个层分别经过相同卷积和ELU(k=9，s=2，o=64)，卷积核大小为9，步长为2，输出通道数为64，得到的数据为64\*28\*28。  
第二层是低层胶囊:包含4组，每组里面有两层卷积，分别为(k=9,s=2,o=32)和(k=2,=2,o=16),不同层之间使用squash函数，而非激活函数，每组里面第二层的输出是16\*5\*5，  
第三层是高层胶囊，输出的维度是32维，（五个子图用不同的来提取信息）  
第四层是连接层，把五个分图计算得到的结果拼接起来，再用一个胶囊进行计算，得到128维数据，即Zt。

解码部分：每个状态St，经过一个全连接层，变128维，和胶囊网络的128维输出Zt, 拼接起来成时间长度\*256的矩阵，使用LSTM进行编码（隐层128维），最后使用一个全连接层进行解码，变成一个2(x,y)\*时间步的预测结果

损失函数：L=aL1+bL2，L1是平均绝对值误差，L2是平方差误差，ab都为1

缺点：没有考虑到不同agent之间的交互信息