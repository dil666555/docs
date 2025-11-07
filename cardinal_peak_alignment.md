cardinal

**参数：**

object(mz, intensity等)，ref，tolerance，units，domain(感兴趣的m/z范围)，binfun，binratio

​	binfun(media, mean, min/max)，计算相邻峰间隙的方法（默认计算中位数，计算平均数，使用最大/最小）

​	binration，定义了容忍度是格子的几分之几

**总流程：**

1. ==如果没有指定tolerance，计算一个（分析所有峰之间的距离，使用binration计算）==
2. 创建或者使用ref
   1. 没提供ref：
      1. 将所有质谱图的峰位置（index）分箱（binning）
      2. 合并每个箱子里的峰，计算每个箱子的中心位置，形成ref
   2. 提供了ref：
      1. 使用该ref
3. 执行对齐
   1. 有了ref和tol
   2. 遍历每个质谱图，查看每个质谱图的m/z_list，如果在某个中心位置的容忍度范围内，将强度记录在中心位置上
   3. 生成生成一个稀疏矩阵
4. 将对齐后的spectraData，ref以及所有==操作的元数据（如使用的tolerance等）==，打包为一个SpectraImagingExperiment对象后返回

**计算ref：**

1. 定义分箱函数==fun（具体的逻辑是什么）==,fun函数通过domain(想保留的m/z范围处理)
2. 使用fun，遍历所有的质谱图，对所有峰进行分箱，完成后，返回peaks。==具体格子是如何计算的==
3. 清理掉无效值（没能成功分到任何格子里的峰），mergepeaks将peaks值分组，合并（计算平均数）
4. 合并好的值即为ref的峰，==且ref的每个m/z值附带一个n（这个m/z是由多少个原始峰合并成的）==
   1. n值很大的峰，可以肯定是一个真实存在的，普遍的生物分子信号
   2. n值很低的峰，很可能是随机产生的伪峰，之后可以用n<100的峰舍弃掉

**执行峰对齐：**

1. 定义输出矩阵的维度
2. tolerance = tol(计算或者提供而来)，使用sparse_mat函数进行匹配，该函数会检测m/z是否落在ref中某个r的（r-tol, r+tol）内

**具体算法**

1. 如何计算tol
   1. 通过binfun确定了四种方式，遍历所有谱所有峰，计算出一个典型间隙值width
   2. tol = width*binration
   3. 如果单位是ppm则，则tol值为tol乘2，四舍五入到第六位，然后除二
   4. 如果单位是da(mz)，则直接对tol进行四舍五入到第四位即可

1. 如何计算ref

   1. 遍历所有质谱，将所有的峰（每张谱里的都要）全部映射到Bins，并将其收集到中间列表peaks

   2. mergepeasks函数处理peaks，排序和合并，计算最后的参考值

   3. get_reference_mz_axis_cardinal是如何做的

      1. 创建固定大小的数组

      



**python实现**

**计算tol**，cardinal是==分块计算==，我是进行了==采样==，==逻辑不变==

**计算ref**

**1. 主函数caculate_reference_peaks**

​	功能：从未经处理的质谱数据到最终参考峰列表（包含m/z和频次计数）的全部计算过程。它将调用所有必要的辅助函数。

​	参数：ms_data, tol, units, N_sample, binfun, binration, domain(感兴趣的m/z范围)

​	返回：pandas.DataFrame: 一个包含两列的DataFrame。一列是m/z，一列是每个m/z是有多少原始峰合并而来的n

**2. 辅助函数**

1. **_create_domain_bins**

   功能：根据给定的m/z范围和分辨率，创建一个等间距的一维网格（bin）。

   参数：domain：touple[float, float]，tol(通常计算而来)

   返回：np.ndarray: 一个一维NumPy数组，包含所有箱子中心点m/z值

2. **_get_all_binned_peaks**

   功能：遍历所有质谱图，将每个m/z映射到网格上最近的bin中心点(可以想想什么算法可以节约内存)

   参数：ms_data, domain, 由_create_domain_bins创建的分箱网格

   返回：非常长的NumPy数组，包含了每一个原始山峰被映射到的domain网格点的m/z值，包含大量重复值

3. **_merge_binned_peaks**

   功能：核心功能，接受上一步产生的大量有重复的分箱后的m/z值，将数值上邻近的m/z分组，让后为每个组计算质心（平均值）和成员数量

   参数：all_binned_peaks: np.ndarray，tol

   返回：Touple[np.ndarray, np.ndarray]: 包含两个NumPy数组的元组，ref_mz，唯一的参考m/z值，ref_counts，与ref_mz一一对应的频次计数

4. **考虑加入chunk_lopply**

   减小内存压力，还是说我们的懒加载模式不需要呢

**峰对齐**

**阶段一**   参数估算与准备

1. 做一些检查，看传入内容是否合理以及计算tolerance

2. 计算典型间隙，峰之间的典型间隙
3. 没有提供ref时候使用domain，tolerance，binration计算分辨率，创建用作分箱基础的等间距m/z网格

**阶段二**   ref生成

**阶段三**   执行对齐

1. 函数sparse_met:

   1. sampler = 'none'时，目标：创建一个稀疏矩阵（M, N）M=参考峰数量，N=像素数量

      矩阵中每个单元格（r, c）（第r个参考峰，第c个谱图）的值是：在谱图c中找到一个m/z值，这个m/z最接近参考峰r的m/z，如果这个最近的m/z与参考峰r的m/z之间的距离tol范围内，就将这个值填入单元格（r, c）如果找不到，则该单元格为0。（只找最近的）

   2. sparse_met的决策过程

      对于ref peak（100.00），spectrum中哪个是最近的

      peak a（100.02）最近

      peak a是否在tol之内

      如果0.02 <= tol

      在矩阵该点填入

**阶段四**   输出对象











