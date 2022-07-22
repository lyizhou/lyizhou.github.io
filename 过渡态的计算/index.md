# 过渡态的计算


## 原理

![](https://pic-1302704451.cos.ap-nanjing.myqcloud.com/20220712102108.png)

> Arrhenius Equation 反应速率对于温度的依赖性，计算化学反应速率和活化能
$$
k(T)=Ae^{-\frac{E_a}{RT}} \to E_a = - \frac{\partial \ln k(T)}{\partial (\frac{1}{RT})}=constant
$$

> Eyring Equation 化学反应速率随反应混合物温度变化的方程，用于过渡态理论
$$
k = \frac{kk_BT}{h}e \frac{\Delta G_+^+}{RT}
$$

![](https://pic-1302704451.cos.ap-nanjing.myqcloud.com/20220712104206.png)
<center>Arrhenius 和 Eyring 方程的区别</center>

**过渡态的定义：**
    过渡态是指势能面反应路径上的能量最高点，它通过最小能量路径(minimum energy path,MEP)连接着反应物和产物的结构

**过渡态只有一个虚频：**
    在势能面上，过渡态结构的能量对坐标的一阶导数为0，只有在反应坐标方向上曲率（对坐标二阶导数）为负，而其它方向上皆为正，是能量面上的一阶鞍点。过渡态结构的能量二阶导数矩阵（Hessian矩阵）的本征值仅有一个负值，这个负值也就是过渡态拥有唯一虚频的来源。若将分子振动简化成谐振子模型，这个负值便是频率公式中的力常数，开根号后即得虚数。

**过渡态搜索算法 CI-NEB方法：**
    NEB与String等方法都可以结合Climbing Image方法，它专门考虑到了定位过渡态问题。CI-NEB与NEB的关键区别是能量最高的点受力的定义，在CI-NEB中这个点不会受到相邻点的弹簧力，避免位置被拉离过渡态，而且将此点平行于路径方向的势能力分量的符号反转，促使此点沿着路径往能量升高的方向上爬到过渡态。这个方法只需要很少的点，比如包含初、末态总共5个甚至3个点就能准确定位过渡态，是最有效率的寻找过渡态的方法之一。如果还需要精确描述MEP，可以在此过渡态上使用Stepwise descent方法、最陡下降法、RK4等方法沿势能面下坡走出MEP，整个过程比直接使用很多点的NEB方法能在更短时间内得到更准确的MEP。


## CI-NEB过渡态计算
**锂离子在石墨上的迁移路径**

1. 优化初态和末态。
<center class="half">
    <img src="https://pic-1302704451.cos.ap-nanjing.myqcloud.com/20220710174037.png" width=360>
    <img src="https://pic-1302704451.cos.ap-nanjing.myqcloud.com/20220710174118.png" width=360>
</center>

2. 用dist.pl检查两个优化后的结构(两个CONTCAR)的相似程度。在执行前给可执行文件权限`chmod +x dist.pl`
    ```bash
    dist.pl ini/CONTCAR fin/CONTCAR
    ```
    ![](https://pic-1302704451.cos.ap-nanjing.myqcloud.com/20220711203021.png)
3. nebmake.pl 插点
    插点数目取$\frac{dist.pl\ 返回值}{0.8}$ 
    ```bash
    nebmake.pl ini/CONTCAR fin/CONTCAR 3
    ```
    ![](https://pic-1302704451.cos.ap-nanjing.myqcloud.com/20220711203131.png)
    
    生成了`./00 ./01 ./02 ./03 ./04`五个文件夹。
    其中00表示初态，里面为ini/CONTCAR，04表示末态，放入的为fin/CONTCAR，其它为插入的点，三个文件夹里面文件名都是POSCAR。
4. 初末态对应的OUTCAR复制到相应的文件夹。
5. 检查插入点的合理性
    ```bash
    nebmovie.pl 0   #参数0表示用POSCAR生成xyz文件；1，用CONTCAR生成。
    ```
6. 准备INCAR
    ```bash
    #### initial I/O ####
    SYSTEM = GLi-NEB
    ISTART = 0    
    ICHARG = 2
    NCORE = 4
    #### Ele Relaxation ####
    ISPIN = 2
    ISMEAR = 0
    SIGMA = 0.01
    ENCUT = 500
    PREC = N
    ALGO= V
    LREAL = .Auto.
    ISYM = 0
    EDIFF = 1E-5      # 更精准的电子步会加速收敛
    #### Geo opt ####
    EDIFFG = -0.03    # 过渡态可以适当放宽结构优化的收敛精度到-0.03
    POTIM = 0
    IBRION = 3        # IBRION = 3, POTIM = 0, VTST识别并启动VTST优化算法
    NSW = 300
    ISIF = 2
    #### VTST ####
    IOPT = 1          # 优化算法 IOPT = 1, 2(适合精收敛) IOPT = 7(适合粗收敛)
    ICHAIN = 0        # 开启NEB方法
    LCLIMB = .T.      # CI-NEB
    IMAGES = 3        # 插入的点个数
    SPRING =-5        # 弹簧力常数，默认值-5   
    #### DFT-D ####
    IVDW = 11
    ```
7. 准备POTCAR和KPOINTS
   
   同结构优化一致

8. 提交任务

    核的个数必须整除插点的数量
9. 跟踪检查收敛结果
    ```bash
    nebmovie.pl 1
    ```
    ![](https://pic-1302704451.cos.ap-nanjing.myqcloud.com/20220711202823.png)

10. 频率计算验证虚频
    ```bash
    #### inital I/O ####
    SYSTEM = Gli-NEB-Freq
    ISTART = 0
    ICHARG = 2
    NCORE = 4
    #### Ele Relaxation ####
    ISPIN = 2
    ENCUT = 500
    ISMEAR = 0
    SIGMA = 0.01
    PREC = A
    ALGO = N
    EDIFF = 1E-5
    ISYM = 0
    LREAL = .FALSE.
    NFREE = 2
    LPLANE = .TRUE.
    #### Geo opt ####
    EDIFFG = -0.01
    POTIM = 0.015
    IBRION = 5
    NSW = 1
    ISIF = 2
    #### DFT-D ####
    IVDW = 11
    ```
11. nebresults.pl总结结果
    通过执行一系列的脚本，来总结neb结果
    <center class="half">

    <img src="https://pic-1302704451.cos.ap-nanjing.myqcloud.com/20220711202535.png" width=310>
    
    <img src="https://pic-1302704451.cos.ap-nanjing.myqcloud.com/mep.png" width=400>

    距离和能量势垒图
    </center class="half">
    nebresults.pl会把所有OUTCAR都打包成.gz的文件，如果不想打包可以用如下命令解压。
    `gunzip 0*/OUTCAR.gz`

12.  查看虚频 `grep THz OUTCAR` 查看频率，f/i为虚频 
    
     ![](https://pic-1302704451.cos.ap-nanjing.myqcloud.com/20220711202337.png)

     下载OUTCAR，用Jmol打开，工具-震动-开始振动，跳转到最后一个频率，左箭头跳转到倒数第二个频率

     <img src="https://pic-1302704451.cos.ap-nanjing.myqcloud.com/OUTCAR.png" width=700>

13. 沿着不想要的虚频方向调整过渡态结构，重新执行过渡态计算，即把结构向着较小的那个虚频的方向做微小的位移重新作为初始结构计算

## 参考文章
1. [Difference Between Arrhenius and Eyring Equation](https://www.differencebetween.com/difference-between-arrhenius-and-eyring-equation/)
2. [过渡态、反应路径的计算方法及相关问题](http://sobereva.com/44)
3. [vasp-vtst计算过渡态--NEB方法](http://blog.wangruixing.cn/2019/08/19/cineb/)
4. [vasp-vtst计算过渡态(NEB方法)具体过程](https://zhuanlan.zhihu.com/p/375723525)



---

> 作者: 之航  
> https://lyizhou.github.io/%E8%BF%87%E6%B8%A1%E6%80%81%E7%9A%84%E8%AE%A1%E7%AE%97/
