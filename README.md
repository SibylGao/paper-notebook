# paper-notebook
记录这段时间的论文，MVS相关

2020.7.20：
昨天看了一下patch match的算法，感觉就是随机一个patch中心位置然后选择一个最接近的offset进行衰减半径的搜索。
今天看了patch match stero这篇的论文，感觉有点难理解，它主要是为了解决匹配区域内视差不一致的问题，由于深度不一样会导致视差不一样，这篇文章将patch match用于MVS方法中，对每一个像素都搜索了一个最优平面，即最优的视差，用点法式来表示最优的平面（点坐标+法向量），上次讲论文小格也提到了迭代的问题，在这篇文章中是分别对图像从左上角到右下角，或者右下角到左上角进行遍历，用的也是patch match中随机+搜索更新的规则，并且定义了四个更新的规则，有一点不太懂的地方是不知道他计算代价里面的一对匹配点p，q是分别是左图和右图的一组点，还是同一幅图中位于同一3D平面的点，这个算法属于局部匹配算法，并且由于对像素都是进行了好几次的遍历，所以速度会比较慢

2020.7.21：
Massively Parallel Multiview Stereopsis by Surface Normal Diffusion这篇论文主要为了解决patch match stereo 里面迭代传播比较慢的问题，原始的patch match是从左上到右下或者反过来，这样需要遍历6遍全图，这篇文章是将像素格分为红格和黑格，并且是并行的处理这两类的格子，在处理黑格的时候，所有黑格可以并行处理，因为他们法向量和深度的传播是靠周围的红格，相邻的黑格更新平面不会互相影响，这样的方法非常新颖也大大提高了更新的速度，并且还可以通过调整周围红格的搜索范围来提高速度。作者还将这种方法推广到了多张图像，具体就是用单应性矩阵来完成不同平面坐标系之间的变换，将patch match中左右图之间的cost function推广到所有与参考图像有重合的图像，并且将所有的视图的cost进行一个排序，取代价值最低的几个cost进行累加，作为更新最优平面时判断的依据，分别将每个视图作为参考图像重复上面的步骤，就能分别计算每个视图的深度图，最后将所有深度图进行融合，并且也像patch match一样最后要做一个左右一致性检查，不满足的点将被删除。这种并行处理的思路大大提高了patch match的速度，作者提到它的速度和准确率都比PMVS要高。

2020.7.22
MVSNet: Depth Inference for Unstructured Multi-view Stereo
这篇文章跟之前看过的端到端重建深度的框架很像，都是提取特征---warp构建feature volume---构建cost volume---输出深度图---refine这样的一个结构。并且作者在构建feature volume这一步划分256个深度平面（跟我之前讲的那篇深度学习+平面扫描的文章很像），分别计算对应的平面坐标转换的单应性矩阵，然后再将其它图像warp到参考图像的坐标系下，根据方差来算cost volume，再通过U-Net结构来生成probability volume，计算深度概率图的数学期望作为估计的深度图，最后将初步估计的深度图和参考图像经过神经网络求出一个残差，再加到初始的深度图上得到refine之后的深度图，最后作者还根据深度概率图过滤掉一些不可靠的深度点，至此完成了深度图的估计。作者再DTU数据集上做了测试，跟patch match的Gipuma算法做了一个对比，虽然MVSNet的准确率是要低于Gipuma的，但是完整度要高于Gipuma，并且速度也更快。

2020.7.23
Recurrent MVSNet for High-resolution Multi-view Stereo Depth Inference
这篇论文是在MVSNet基础之上的改进，MVSNet是使用U-Net来计算probability volume，这样的方式对内存的消耗十分大，因此作者提出了使用循环神经网络的一种GRU来对cost volume进行正则化，关于GRU我去查了一些资料，感觉跟LSTM很像都是对数据有选择性的遗忘和记忆，作者将cost volume根据深度的不同拆分成多个input volume，作为一个序列输入GRU中，用循环神经网络的结构解决输入序列之间的相关性，首先通过一个2D卷积层将32-channel转化为16-channel作为GRU层的输入。之后，每一层GRU输出作为下一层输入，最后一层的GRU输出为1，再通过一个softmax层来输出深度概率，用这种方法代替了MVSNet中的U-Net以及残差结构。最后作者实验出来的结果是准确率虽然仍然不如Gipuma，但是完整性和准确率的都要高于MVSNet，对内存的占用也比3D的CNN更少。

2020.7.24
Point-MVSNet
这篇文章在前面部分是和MVSNet一致的，用MVSNet来提取出粗略的深度信息，在MVSNet和Recurrent MVSNet中到了这一步之后都会有一个refine的步骤，MVSNet是将彩色图像+深度图生成一个残差来得到更好的结果，Recurrent MVSNet是使用GRU来代替后面的这些步骤，在之前看的论文对比中可以发现MVSNet系列的方法对深度估计的完整度并不高，因此作者这篇论文将2D和3D特征进行融合以求得到更加完整的深度结果。这篇文章最核心的地方就是pointflow这个结构，pointflow首先是将2D的深度图“unproject”成3D的点云，但是我有去查过DGCNN生成点云的方法，里面说的是从通用的高斯模型中进行采样，再通过K近邻来聚合（？）这里细节的地方没有来得及看，准备明天再去仔细看一下DGCNN的方法，里面还用到了边卷积，感觉是将概率作为每一条边的权重然后用求质心的方法来计算出每个unproject point的偏移，最后得到参考图像的残差值，对MVSNet输出的深度图进行加强。作者再使用MVSNet的时候，作者将特征图下采样到了原图的1/8，因此再进行3D卷积的时候占用的内存更少，但这样初始生成的深度图应该会比MVSNet更稀疏。这篇文章最后的结果是完整性和准确率都要高于MVSNet，并且内存占用更少。

2020.7.30
Dynamic Graph CNN for Learning on Point Clouds
这篇文章是用edgeconv的方法来对输入的3D点云进行聚类和分割，是基于PointNet的一个工作，整个网络分为classification和segmentation两个部分，classification的网络输出一个1D的用于分类的描述子，然后segmentation网络将这个描述子和每一层edgeconv输出的特征综合起来计算出每个点的classification score。对于每个edgeconv模块共享边缘函数，并且这个计算特征的边缘函数是用MLP来表示的，也就是一个多层感知机结构，在一个set中点与点之间是全连接的一个关系。总体上来说，我感觉是将KNN的聚合和分类过程中用到的特征空间中的距离用这个边缘函数来代替，并且不断地更新边缘函数来动态地更新每一个set中所含的点，因此也有了动态图这个概念。edgeconv就是一个多层感知机结构（？但是为什么专门提出来是edgeconv，跟多层感知机有什么区别？这里似乎是每一个神经元之间的连线代表两个点之间连接的边），此外网络中还有一个超参数K，跟KNN中的K相对应。还有一个对应的Point cloud transform block，这个模块是为了将一个输入的set变换到标准的三维空间（？边卷积想象起来还是有一点太抽象，但是整个网络的输入应该是标准三维空间的一个点云）一个edgeconv模块如果是a层的一个MLP，经过池化之后，输出的特征也是a维的，如果有n个点，那输出张量大小就是n*a。作者分别实验了分类任务和分割任务，在分类任务中，表现是要优于PointNet系列的结果的，并且参数量也比pointnet更少；在分割任务上也优于了PointNet。看完了DGCNN感觉还是不太理解PointMVS里面的unproject点以及它是如何计算偏移和图像残差的，DGCNN感觉更像是一个分割算法，本身并不会对点云的位置进行矫正。

2020.7.31
Polarimetric Multi-View Inverse Rendering
这篇文章中作者提出了使用偏振光相机图像作为输入来对点云进行一个refine让三维重建的结果更加的精细。之前看到的很多refine的方法都是基于彩色图像的photometric，还是第一次看见用偏振光图像来做的。整个算法的输入是一系列的彩色图像以及对应的偏振光图像，先根据彩色图像序列使用SFM的方法解算出相机的姿态，再根据解算出来的相机姿态使用MVS系列算法得到一个初步的点云，最后用彩色图像+偏振光图像（Photometric and polarimetric optimization）的方法来对点云进行一个refine得到更精细的结果，在SFM和MVS算法上面作者选择的是COLMAP+OpenMVS，作者着重介绍了一下photometric+polarimetric优化的这一部分。优化的参数分别是三角网格的顶点，每个顶点的albedo（albedo不是相对于一个平面的吗？）以及场景的光照；cost function包括了四项，分别是Photometric rendering term（这一项用来表示实际的光强和用参数估计的光照模型下三角网格顶点光强的差值），Polarimetric term（这一部分函数很复杂，但感觉是衡量根据投影计算出来的法向量跟偏振光估计出来的法向量的差距，并且同时消除了pi和pi/2 ambiguities），Geometric smoothness term（这一项是通过约束三角网格极其周围的网格法向量来控制几何形状的平滑），Photometric smoothness term（这一项是约束相邻三角网格的patch之间的albedo差异），用数值方法优化这个函数，就能得到refine之后精细的三维模型。通过实验可以看出本文的方法比其他三维重建的方法得到的结果都要精细，在准确率和完整度上也高于了CMPMVS和OpenMVS，这证明了使用偏振光的方法对于提升三维重建的精度是十分有效的。


2020.8.3
Planar Prior Assisted PatchMatch Multi-View Stereo
这篇文章作者主要旨在解决patch match算法里面用基于光度一致性的cost function对低纹理区域的估计不准确的问题，因此提出了一种基于平面先验的patch match算法。整个的算法流程为，先用基于光度一致性的patch match方法中的cost function（例如COLMAP,ACMH）来初步得到一个重建的结果，再根据这个结果生成三角网格，同一个网格平面中的点法向量相同，再将三角网格平面的法向量作为先验推导出一个新的cost function，这个cost function有两项，一项是基于光度一致性假设，另一项是三角网格平面的假设，并且这个先验包含了三项（分别是三角网格平面距离之间的差值为指数，三角网格法向量夹角为指数，和一个常数项，这个公式是作者自己定义的，感觉是为了衡量假设与先验之间的区别，有一点点意会，但是不知道为什么要这样设）；并且在进行光度一致性假设估计的时候，cost function里面有一组权重系数，为了表示patch l 周围的neighbor patch的可见性，也将其作为概率的先验，并且是通过蒙特卡洛采样的方式来统计这个概率，作为权重（应该就是直接采样然后通过频数计算频率以频率来代替概率，说实话这一步我不知道为什么要加这一个权重和先验）。本文中更新的方法也是使用了棋盘并行更新的方法，但是它在更新的最后都会有一步refine（？这一步也没有看懂），总的来说感觉整个方法还是基于patch match的框架，只是用了大量统计学的方法来对cost function和propagation进行了优化，最后作者在ETH3D数据集上的结果准确率没有COLMAP高，但是F1 score要高于现有的算法，作者也提到了它的性能跟ACMM相当。

2020.8.5-8.6
Confidence-Based Large-Scale Dense Multi-View Stereo
 整个算法特别复杂工作量也特别多（这就是期刊和会议的不同吗...）整个算法的流程可以概括为：baseline Patchmatch，根据初始生成的深度生成法向量，基于神经网络的confidence prediction，根据置信度来进行插值，（对于outdoor场景下）用mask和检查法向量和光线的夹角来去掉天空的深度值，最后再使用patch match的方法+先验来进行refine，最后再对生成的各个视点的图进行fusion。在baseline patchmatch这一步，首先是使用了仿射系数omega对光强进行了一个加权，再求一个q和p之间的correlation计算，这里的q和p的光强是指实际光强减去加权光强（这一步不知道为什么要这样做），最后再除以一个表示模的量，因此是正则化之后的cost即ANCC。为了增强鲁棒性，取了前Kb个cost值来计算，这一步使用棋盘格的propagation方法。
计算法向量这一步，取的是像素p上下左右四个像素做逆投影变换之后的差值，取这两个向量的叉乘方向作为p的法向量。在confidence prediction这一步里面，为了得到连续的confidence的值使用了神经网络的方式来得到一个confidence值，为了生成正负样本，作者定义了一个算法来评价样本是可靠还是不可靠（这个算法中的参数是作者自己手动设置的）判断一个点的深度是否可靠有两个条件分别是geometric consistency（p点逆投影到neighbor img q的深度值和q点附近四个像素插值得到的深度之间的差距）和spatial consistency（source img逆投影到reference img上的投影距离和法向量夹角的误差，使用投票机制）满足spatial consistency函数的投票值大于一个阈值，以及geometric consistency函数小于一个阈值被认为是正样本，否则是负样本，将其输入到一个神经网络，计算出置信值。然后就是基于confidence值进行插值，这一步是基于weight least base方法，这一部分实在是没有看懂，后面再去查一下WLB-based方法，但是看能量函数是两项分别是深度置信，以及彩色图像和深度图的二次导数（应该是在置信度低的区域由彩色图像和深度图像中导数变化比较大的区域传递到导数低的区域？）。最后是refinement和fusion，在上一步中得到的插值结果是作为这一步的先验，感觉跟上一篇看的论文Planar Prior Assisted PatchMatch Multi-View Stereo这里的方法很像，这里就不再赘述了。（但是整个算法中用了两次patch match感觉很奇怪，最后花费的时间肯定也会很多）最后作者在ETH3D数据集上的结果，优于了PMVS,OpenMVS等算法，在ETH3D和DTU数据集上都能达到state-of-the-art，但代价是会花费很长的时间。

2020.8.10
Learning Stereo from Single Images
这几天看了一下ECCV的oral文章，选了几篇MVS相关的来精读，这是一篇关于用单目相机的单张图像生成stereo数据的文章，作者首先是用单目估计深度的算法估计出深度，再根据深度生成视差，最后将原图像作为左图，通过视差图和原图像来生成不同的右图。首先是估计深度，在这一步里面作者没有详细描述，就是单纯的使用一个神经网络来估计深度，在每个不同的数据集上会进行不同的finetune，深度和视差是一一对应的关系，因此直接通过深度换算到视差就行。通过左图生成右图则是使用不同的baseline和不同的焦距根据视差进行转换就行，作者这里使用的是forward warping的方法，将左图的像素warp到右图，再根据插值得到右图像素的值，这里面会分别出现Occlusion（右图有的像素左图没有）和Collisions（与前述相反）的问题，分别用填充（参考的是另一篇论文的方法）和选择左图中的重叠像素的最大视差对应的像素来解决，为了解决预测深度的网络在深度不连续的地方会产生模糊边界的问题，作者提出了一个Depth Sharpening的步骤，使用 Sobel 边缘检测算子，将滤波值大于3的像素点作为不准确的边缘点（这里不明白，sobel算子的值越大应该是边缘越清晰啊？）再用周围的视差来矫正这些点。作者将单目图像生成深度的算法跟基于深度学习的MVS方法结合在一起，用单目图像生成的stereo和估计的深度作为MVS的输入，从而不需要大量的ground-truth的图像和对应的深度值来进行训练。作者提出由于这样生成的训练数据比较多，这样训练出来的结果比stereo matching networks baseline更好，并且得到的性能提升跟MVS使用什么样的网络结构没有关系。

2020.8.11
今天继续看ECCV的论文，有一篇oral的文章叫做Domain-invariant Stereo Matching Networks，这篇文章是做Domain Adaptation的，属于迁移学习的领域，主要是为了解决 stereo matching networks在不同环境下的表现不一致的问题，这篇感觉跟MVS算法本身关系不大，而且也不是熟悉的领域没看太懂，准备先记录一下后面如果有用再来翻看这篇文章。
Du2Net: Learning Depth Estimation from Dual-Cameras and Dual-Pixels
这篇文章用dual-camera+dual-pixel的方式来获取输入数据，用神经网络来提取和融合特征，生成cost volume最后预测出深度图。作者使用的相机是一对dual camera，并且右眼是dual-pixel，dual pixel的baseline是垂直于dual camera的baseline的。这样做的好处有两个，首先是当texture平行于某个baseline的时候它会被令一个系统检测出来，其次，DP系统对近处的深度更敏感，DC系统对远处的深度更敏感，两者之间可以形成一个互补。文章的整个过程跟之前看到的端到端网络很像，并且我感觉也是跟基于平面扫描的深度学习方法很像，都是根据不同的视差（逆深度）来聚集形成cost volume，但是这里有一个问题是DC系统和DP系统的尺度是不同的，因此在将两者的cost volume融合之前将其用softmax函数都变换到了（0，1）之间，并且推导出来DC和DP系统之间的一个仿射变换，将其变换到同一个尺度下才进行了fusion（除了仿射变换之外还有左右图之间的warp）。然后是跟常规的基于深度学习的MVS方法一样定义lost进行训练。由于作者的输入是需要特定的相机，因此作者自己采集了一个数据集来进行训练，作者跟普通的stereo算法和基于DP的算法都做了比较，用作者采集的数据集的情况下，Du^2Net的表现优于了state-of-the-art的算法，如PSMNet，StereoNet等。

2020.8.14
Cost Volume Pyramid Based Depth Inference for Multi-View Stereo
这篇文章小格有讲过，但我还是重新把它看了一遍，这篇文章主要的创新点在于用cost volume 金字塔的方式从低分辨率到高分辨率这样的coarse-to-fine的方法来对图像进行三维重建，主要也是为了解决MVSNet系列方法（MVSNet，Point-MVS）在计算残差的时候对cost volume使用的是3D卷积这种方法对内存的消耗非常大，因此作者使用这种金字塔递推的方式来节约内存。整个pipeline中有好几个金字塔（图像，特征，cost volume和概率金字塔）
首先是图像金字塔，作者对图像进行下采样得到图像金字塔并且从分辨率最低的图像开始估计深度。其次是feature金字塔，这个是通过将图像通过一个提取特征的神经网络得到的，并且每一层都share参数。
然后是cost volume金字塔，cost volume分两种，对于第一层，由于要估计出一个绝对值的深度，所以是使用MVSNet的方法直接对特征进行堆叠并且根据平面扫面算法对对每个不同的深度将ref img逆投影得到src img中匹配的特征，堆叠在一起形成不同深度的cost volume；对于其它层，则使用ref img和src img平均特征之差，方法跟上述的一致，只不过这里的深度是作者定义的一个残差深度，是一个相对值。（整个结构里面只有第一层是绝对值，其它全是相对于上一层的值）
在概率金字塔这里，作者使用的是一个3D卷积生成概率金字塔，这一步也和MVSNet一致并且每一层是share系数的。生成深度图的就是对概率volume做一个softmax，最上层的深度和下面的深度计算不一样，在金字塔最上面一层，对每一个像素用输出的概率对深度进行加权得到该像素的深度；对于下面的金字塔，则使用上一层上采样之后的深度图+残差概率估计这样的方法。
最后整个网络的loss是对于每一层和ground truth之间的差。
作者在DTU数据集上的实验表明，这种方法在准确率完整度和整体表现上都优于了MVSNet系列的算法，只在准确率上低于了Gipuma算法，但是完整度是要远远高于Gipuma的。其实看完这篇文章也不知道它为什么内存占用会小很多...在训练过程中还要储存每一层的结果和ground truth进行比较，并且每一层都还是需要进行3D卷积。但是感觉这种用金字塔递推的思路值得借鉴，在顶层低分辨率的地方可不可以就视为高置信的区域？ 

2020.8.15
A Novel Recurrent Encoder-Decoder Structure for Large-Scale Multi-view Stereo Reconstruction from An Open Aerial Dataset
这篇文章作者主要介绍了一种基于aerial img的MVS算法，由于这篇文章中作者要使用基于深度学习的方法，而现有的数据集不足以支撑这样的运算，所以作者自己用无人机（5个相机一个竖直，4个40°角倾斜）采集了图像的数据再用Smart3D来数据进行重建得到3D模型（这里有一个问题，这样重建出来的数据能算是ground truth吗，尽管作者提到他们用手工对数据进行了修正），这样采集了6.7*2.2km^2的范围，并且作者为了得到更好的训练效果将采集的区域进行了分区，用纹理较为复杂的区域和建筑较为多的区域来进行训练，其它的用作测试集，并且由于一共有5个相机因此每张图片相当于一个场景对应着一张ref img，4张src img，（这里不太能理解的是在采集的时候相机是有倾斜的，但是作者在这里说0号和2号照片是heading方向，作者还将数据集分为muliti-view数据集和stereo数据集，只是在每个场景的图像数量和大小上有区别）最后训练集有261张图像，测试集93张。
在算法方面感觉大部分还是比较常规的，跟R-MVSNet很像，并且也使用了GRU单元，但是不同的是在正则化cost volume的时候作者使用的是一个encoder-decoder的网络结构，并且两边是完全对称的，不像R-MVSNet对图像进行了一个1/4下采样，而且分别在encoder和decoder的不同层使用了GRU这样可以更好的从各个尺度和各个感受野来获得信息，同时GRU也保持了记忆功能，将深度方向的上下文信息联系在一起，代替了3D卷积，大大减小了内存占用。在WHU 数据集上的结果优于了state-of-the-art的MVSNet和R-MVSNet，并且在5视图的情况下运行时间是R-MVSNet的一半。我觉得这篇文章值得借鉴的地方是他构建数据集的方法，以及将GRU应用到encoder-decoder结构中。

2020.8.28
Attention-Aware Multi-View Stereo
这篇文章主要介绍了一种基于注意力机制的MVS算法，整个程序大的框架跟普通的基于深度学习的框架相同，即生成cost map，cost聚集，cost正则化生成probability volume最后是回归得到预测的深度图。不同的是作者在cost聚集和cost正则化这两个步骤中加入了注意力机制。
在进行cost聚集的时候，作者使用了 squeeze-and-excitation block结构，将cost volume沿着深度方向对每一张图像的cost map做一个全局池化（得到0-N个全局描述子），将这个全局描述子取方差，这样就得到了一个1X1XZ的向量（其中Z是平面扫描的深度数目也是通道数），然后再通过两层的全连接对这个注意力向量进行正则化，按照通道乘回最开始堆叠在一起的那个raw cost volume，就得到了一个基于注意力机制的cost volume。
在正则化的步骤中，作者也使用了注意力机制和ray fusion module。作者先是通过卷积神经网络的不同步长将cost volume encode成大小不同的volume，再对这些volume下采样两次得到了一个类似于金字塔的结构。从最底部对volume进行一个正则化，并且上层volume的正则化是基于下层volume正则化之后的结果。具体算法就是RFM模块，RFM模块中分别是pre-contextual模块，attention模块，和post-contextual模块，其中较为复杂的是ray attention模块，这个模块的注意力是基于pre-contextual的输出和下层cost volume正则化结果的残差，注意力权重的计算也是使用的squeeze-and-excitation算法，算出注意力加权后的残差再加上上一层正则化的结果再通过一个post-contextual模块就得到了这一层正则化的最后结果。
loss function有两项，分别是深度正则化误差和xy方向上分布的梯度误差。最后在DTU数据集上的结果在深度数目Z=384的时候达到的效果优于了MVSNet系列的方法，增加深度数目的同时，精度和完整度都增加了。


2020.8.31
BlendedMVS: A Large-scale Dataset for Generalized Multi-view Stereo Networks
这篇文章主要介绍了一个可以用于MVS深度学习训练的数据集和生成数据集的pipeline。作者提出，现有的MVS数据集中，DTU是采集了100多个室内的物体和它们的点云，再通过点云render出ground truth的训练图像，但是DTU的相机轨迹是固定的，会给基于深度学习的方法带来一定的泛化困难。Tanks and Temples既有室内的也有室外的场景，但是只有7个场景有ground truth的深度图，训练集较小也会给基于深度学习的方法带来泛化误差。而ETH3D数据集也有同样的问题，就是ground truth的场景比较少。
基于以上情况，作者构建了一个新的数据集，整个流程为：首先在Altizure.com线上平台上人工选择了113个构建的比较完好的3维模型（每个场景有20-100张ground truth图像），然后将三维模型投影到不同的相机位置得到render img和render depth map；但是在这一步中作者提到3D模型中render出来的图像缺乏光照的信息，因此作者用低通滤波器将ground truth img中的光照信息提取出来，并且用高通滤波器对render img进行处理，最后将二者相加得到最后的blended img。（低通滤波器主要是为了提取光照条件，高通滤波器是为了提取texture信息）。最后作者为了增强数据集，提高泛化能力，使用了随机改变图像亮度（一定范围内），随机改变对比度，高斯运动模糊。
最后作者使用MVSNet，R-MVSNet，Point-MVSNet等来验证数据集的泛化能力，发现用BlendedMVS训练出来的模型，在DTU,ETH3D上都能获得不错的验证准确率，因此证明了BlendedMVS数据集在泛化能力上的提升。此外作者还对比了将ground truth img，rendered img和blended img作为输入对准备率的影响，发现使用blended img的准确率最高。
