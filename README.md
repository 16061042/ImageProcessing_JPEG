# ImageProcessing

##Dependencies

[ujmp-complete.jar ](https://github.com/ujmp/universal-java-matrix-package/releases/download/0.3.0/ujmp-complete-0.3.0.jar)



#JPEG图像压缩算法流程详解

JPEG代表**J**oint **P**hotographic **E**xperts **G**roup（联合图像专家小组）。此团队创立于1986年，1992年发布了JPEG的标准而在1994年获得了ISO10918-1的认定。 

JPEG是一种有损压缩。

##色彩空间转换

图片由`RGB`色彩空间转换到`YUV`色彩空间，转换关系如下
$$
Y=0.299R+0.587G+0.114B
$$
$$
U=-0.169R-0.331G+0.500B+128
$$
$$
V=0.500R-0.419G-0.081B+128
$$

一般来说U,V是有符号的数字，但这里通过加上128，使其变为无符号数，方便存储和计算

## 采样

研究发现，人眼对亮度变换的敏感度比色彩变换的敏感度高。因此，可以认为Y分量比U,V分量更为重要。故采样时通常会降低`Y,U`分量的采样率，这里采用411采样方式，即`Y,U,V`三个分量的取样比例为$4:1:1$，其含义为在$2*2$的单元中，`Y`分量采样4次，`U,V`分量各采样一次。这样采样的优点是虽然损失了一定精度，但也在人眼几乎不可见的前提下减小了数据存储量。

## 分块

DCT变换是对$8*8$的子块进行处理的，且`U,V`分量在$2*2$的单元中采样一次，故在DCT变换之前将原图像长宽分别用0补齐到16的倍数。之后再对`Y,U,V`三个分量分别分为$8*8$的块便进行后续操作。

##离散余弦变换 

离散余弦变化的公式为
$$
F(u,v)=C(u,v)\sum_{x=0}^{N-1}\sum_{y=0}^{N-1}f(x,y)\cos{\frac{\pi u(2x+1)}{2N}}\cos{\frac{\pi v(2y+1)}{2N}}
$$

其中$\begin{equation}C(x,y)=\begin{cases}{\frac{1}{N}}& \text{(x,y)=(0,0)}\\{\frac{1}{2N}}& text{(x,y)!=(0,0)}\end{cases}\end{equation}$

由于已经明确每次进行离散余弦变化的矩阵大小为$8*8$，故这里不再采用上述方法进行离散余弦变换，而是利用$DCT$变换矩阵实现$DCT$变换以降低运算量。

$DCT$变换矩阵计算公式为
$$
\begin{equation}T_{ij}=\begin{cases}{\frac{1}{\sqrt{M}}}& {i = 0, 0 \leq j \leq M - 1}\\{\sqrt{\frac{2}{M}}\cos{\frac{\pi(2j + 1)i}{2M}}}& {1\leq i\leq M - 1, 0\leq j \leq M - 1}\end{cases}\end{equation}
$$
故$8*8$的$DCT$变换矩阵
$$
T = \left[\begin{matrix}
0.35355&0.35355&0.35355&0.35355&0.35355&0.35355&0.35355&0.35355&\\
0.49039&0.41573&0.27779&0.09755&-0.09755&-0.27779&-0.41573&-0.49039&\\
0.46194&0.19134&-0.19134&-0.46194&-0.46194&-0.19134&0.19134&0.46194&\\
0.41573&-0.09755&-0.49039&-0.27779&0.27779&0.49039&0.09755&-0.41573&\\
0.35355&-0.35355&-0.35355&0.35355&0.35355&-0.35355&-0.35355&0.35355&\\
0.27779&-0.49039&0.09755&0.41573&-0.41573&-0.09755&0.49039&-0.27779&\\
0.19134&-0.46194&0.46194&-0.19134&-0.19134&0.46194&-0.46194&0.19134&\\
0.09755&-0.27779&0.41573&-0.49039&0.49039&-0.41573&0.27779&-0.09755&\\
\end{matrix}\right]
$$
其转置矩阵
$$
T' = \left[\begin{matrix}
0.35355&0.49039&0.46194&0.41573&0.35355&0.27779&0.19134&0.09755&\\
0.35355&0.41573&0.19134&-0.09755&-0.35355&-0.49039&-0.46194&-0.27779&\\
0.35355&0.27779&-0.19134&-0.49039&-0.35355&0.09755&0.46194&0.41573&\\
0.35355&0.09755&-0.46194&-0.27779&0.35355&0.41573&-0.19134&-0.49039&\\
0.35355&-0.09755&-0.46194&0.27779&0.35355&-0.41573&-0.19134&0.49039&\\
0.35355&-0.27779&-0.19134&0.49039&-0.35355&-0.09755&0.46194&-0.41573&\\
0.35355&-0.41573&0.19134&0.09755&-0.35355&0.49039&-0.46194&0.27779&\\
0.35355&-0.49039&0.46194&-0.41573&0.35355&-0.27779&0.19134&-0.09755&\\
\end{matrix}\right]
$$
$DCT$可简化为$T * B * T'$，其中$B$为$8*8$的原矩阵。

实施二维$DCT$可将图像的能量集中在极少的几个系数之上，其他系数相比于这些系数，绝对值要小很多。这些系数大都集中在左上角，即低频分量区。

## 数据量子化

量子化是JPEG算法中损失图像精度的根源， 也是产生压缩效果的源泉。

人类眼睛在一个相对大范围区域，辨别亮度上细微差异是相当的好，但是在一个高频率亮度变动之确切强度的分辨上，却不是如此地好。这个事实让我们能在高频率成分上极佳地降低信息的数量。简单地把频率领域上每个成分，除以一个对于该成分的常量就可完成，且接着舍位取最接近的整数。这是整个过程中的主要有损运算。以这个结果而言，经常会把很多更高频率的成分舍位成为接近0，且剩下很多会变成小的正或负数。

JPEG提供的量子化算法如下 
$$
B_{i,j} = round(\frac{G_{i, j}}{G_{i,j}})		j, j = 0,1,2,...,7
$$
其中$G$是我们需要处理的图像矩阵，$Q$称作量化系数矩阵 ，$round$函数是取整函数。JPEG算法提供了两张标准的量化系数矩阵，分别用于处理亮度数据$Y$和色差数据$U$以及$V$。 

标准亮度量化表$$Q_Y =\left[\begin{matrix}16&11&10&16&24&40&51&61&\\12&12&14&19&26&58&60&55&\\14&13&16&24&40&57&69&56&\\14&17&22&29&51&87&80&62&\\18&22&37&56&68&109&103&77&\\24&35&55&64&81&104&113&92&\\49&64&78&87&103&121&120&101&\\72&92&95&98&112&100&103&99&\\\end{matrix}\right]$$

标准色差量化表$$Q_C =\left[\begin{matrix}17&18&24&47&99&99&99&99&\\18&21&26&66&99&99&99&99&\\24&26&56&99&99&99&99&99&\\47&66&99&99&99&99&99&99&\\99&99&99&99&99&99&99&99&\\99&99&99&99&99&99&99&99&\\99&99&99&99&99&99&99&99&\\99&99&99&99&99&99&99&99&\\\end{matrix}\right]$$

$DCT$系数矩阵中的不同位置的值代表了图像数据中不同频率的分量，这两张表中的数据时人们根据人眼对不不同频率的敏感程度的差别所积累下的经验制定的，一般来说人眼对于低频的分量比高频分量更加敏感，所以两张量化系数矩阵左上角的数值明显小于右下角区域。 这样就使得量化后的矩阵更多地保留高频信息。

量化后的一个$8*8$矩阵：$\left[\begin{matrix}-24&2&-1&1&0&0&0&0&\\-33&-1&1&0&0&0&0&0&\\9&-1&0&0&0&0&0&0&\\2&1&0&0&0&0&0&0&\\-1&0&0&0&0&0&0&0&\\0&0&0&0&0&0&0&0&\\0&0&0&0&0&0&0&0&\\0&0&0&0&0&0&0&0&\\\end{matrix}\right]$

通常一张图片中的一个$8*8$矩阵经过上述变化之后，矩阵都如上矩阵一样，其中的一大部分数据都会变成0，这非常有利于后面数据的压缩。

## 编码
### 差值编码和Zig-zag扫描

量化后矩阵左上角的值被称为直流分量$DC$，其他63个值被称为交流分量$AC$。其中$DC$不参与Z字形扫描，而是与前一矩阵的$DC$系数进行差分编码($DPCM$)；$AC$分量则采用$Z$字形扫描排列并进行游程长度编码($RLE$)。

### 游程长度编码(Run-Length Encoding, RLE)

量化$AC$系数的特点是包含很多连续的0，故使用$RLE$对其进行编码。JPEG使用了1个字节的高4位来表示连续的0的个数，而使用其低四位来表示下一个非0系数需要的位数，紧随其后的是量化$AC$系数的值。

假设$Zig-zag​$扫描后的一组向量的$AC​$分量为
$$
31,45,0,0,0,0,23,0,-30,-8,0,0,1,0,0,0,0,0,0,0,…,0
$$

经$RLE$压缩后如下
$$
(0,31);(0,45);(4,23);(1,-30);(0,-8);(2,1);EOB
$$

其中`EOB`表示后面都是0。实际上，用$(0,0)$表示`EOB`。若这组数字不以0结束，则不需要`EOB`。

### Huffman编码

JEPG压缩编码时，通过查表实现霍夫曼编码器，本次实验中我使用了ISO/IEC International Standard 10918-1中JPEG推荐的典型霍夫曼表(Typical Huffman tables)，我已经将4张表格发表在[个人博客](https://www.cnblogs.com/buaaxhzh/p/9119870.html)，篇幅所限这里就不再给出了。

之所以需要四张Huffman 编码表是因为，编码时每个矩阵数据的1个$DC$值与63个$AC$值分别使用不同的Huffman 编码表，而且亮度Y与色度U,V也要使用不同的Huffman 编码表。

为提高储存效率，JEPEG里并不直接保存数值，而是将数值按实际值所需要的位数分成16组，如下表所示

|               Value                | Size |                     Bits                     |
| :--------------------------------: | :--: | :------------------------------------------: |
|                 0                  |  0   |                      -                       |
|               -1, 1                |  1   |                     0, 1                     |
|            -3, -2, 2, 3            |  2   |                00, 01, 10, 11                |
|     -7, -6, -5, -4, 4, 5, 6, 7     |  3   |    000, 001, 010, 011, 100, 101, 110, 111    |
|        -15, …, -8, 8, …, 15        |  4   |         0000, …, 0111, 1000, …, 1111         |
|       -31, …, -16, 16, …, 31       |  5   |     0 0000, …, 0 1111, 1 0000, …, 1 1111     |
|       -63, …, -32, 32, …, 63       |  6   |            00 0000, …, …, 11 1111            |
|      -127, …, -64, 64, …, 127      |  7   |           000 0000, …, …, 111 1111           |
|     -255, …, -128, 128, …, 255     |  8   |          0000 0000, …, …, 1111 1111          |
|     -511, …, -256, 256, …, 511     |  9   |        0 0000 0000, …, …, 1 1111 1111        |
|    -1023, …, -512, 512, …, 1023    |  10  |       00 0000 0000, …, …, 11 1111 1111       |
|   -2047, …, -1024, 1024, …, 2047   |  11  |      000 0000 0000, …, …, 111 1111 1111      |
|   -4095, …, -2048, 2048, …, 4095   |  12  |     0000 0000 0000, …, …, 1111 1111 1111     |
|   -8191, …, -4096, 4096, …, 8191   |  13  |   0 0000 0000 0000, …, …, 1 1111 1111 1111   |
|  -16383, …, -8192, 8192, …, 16383  |  14  |  00 0000 0000 0000, …, …, 11 1111 1111 1111  |
| -32767, …, -16348, 16348, …, 32767 |  15  | 000 0000 0000 0000, …, …, 111 1111 1111 1111 |



设一个一维化后的亮度的数据块为
$$
data = 5,31,45,0,0,0,0,23,0,-30,-8,0,0,1,0,0,0,0,0,0,0,…,0
$$
其中第一个数字代表本数据块$DC$值与前一数据块$DC$值之差为5。

那么$RLE$压缩后的$data$变为
$$
data' = (5);(0,31);(0,45);(4,23);(1,-30);(0,-8);(2,1);EOB
$$

其中$EOB = (0,0)$。

$$
变换函数s = f(v), 其中s\in Size, v\in Value, 且s,v位于表格的同一行
$$

对$data'$中每个数对的第二个数$v$求对应的$s$值，并将$s$置于数对中$v$之前，得到$data''$
$$
data'' = (3,5);(0/5,31);(0/6,45);(4/5,23);(1/5,-30);(0/4,-8);(2/1,1);(0/0,0)
$$
其中第一个数对为$DC$分量运算得到的结果。

由于假设该以为数据由亮度数据块变换得来，因此对于$DC$和$AC$分量，分别对$data''$数对中的第一个数字查Huffman DC亮度表和Huffman AC亮度表，再对数对中的第二个数字查上表，即可得到一维数据块$data$编码后的序列。$data''$查表得结果为$011,101;11010,11111;111000,101101;1111111110011000,10111;11111110110,00001;1011,0111;11100,1;1010 $

其中的标点是为了方便对照，实际编码中没有标点。

### 压缩比

最终共用$70973bit$保存压缩后的数据，压缩前的数据大小为$256 * 256 * 8+256 * 256/4 * 2*8 = 786432bit$，故压缩比为$70973/786432 *100\% =9.02\% $。

## 解码

解码过程为编码过程的逆过程，这里不再赘述。

### 解码后的图片与原图片比较

![原图片](D:\JAVA WorkSpace\ImageProcessing_JPEG\pic\lane.jpg)

<center>原图片</center>

![经过编码、解码的图片](D:\JAVA WorkSpace\ImageProcessing_JPEG\pic\lena_after_encoding_and_decoding.jpg)

<center>经过编码、解码的图片</center>

可以看出，处理后的图片与原图片有一定差异，推测差异是在量化和反量化过程中产生的。

## 代码说明

主类为JPEGEncoding

四张$Huffman Table$位于`./HuffamTable`

原图片和生成的图片位于`./pic`

处理过程中的部分结果位于`./debug`

注：`.`为工程目录


