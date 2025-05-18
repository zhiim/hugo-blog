+++

title = "DOA估计中均匀圆阵的模糊问题"
date = 2023-02-25T15:54:25+08:00
slug = "uca-doa-ambiguity"
description = "在使用MUSIC算法进行DOA估计时，必须保证流型矩阵中各个导向矢量线性无关，否则可能出现虚假谱峰值。本文讨论了均匀线阵和均匀线阵的DOA估计中，出现峰值模糊的条件。"
tags = ["DOA 估计"]
categories = ["Notes"]
image = ""

+++

## DOA 估计的模糊问题

一些经典的 DOA 估计算法，例如 MUSIC 算法，利用了阵列流型矩阵和噪声子空间的正交关系来确定入射信号的角度。因此在使用这些算法时，必须保证流型矩阵中各个导向矢量是线性无关的，这也是为什么要保证阵列天线的个数大于入射信号的个数。  
在讨论阵列的相位模糊时，如果某一个导向矢量可以表示成其他 K 个导向矢量的线性组合
$$a(\phi_k,\theta_k) = \sum_{i = 1}^K l_ia(\phi_i,\theta_i), \ \ {l_i} \ne 0$$
则称该阵列存在**K 阶模糊问题**，此时会存在测向模糊。当$K=1$时，称**一阶模糊**或**平凡模糊**。  
高阶模糊的识别比较复杂，发生的条件也更加有限，一般只讨论一阶模糊的问题，即流型矩阵中的导向矢量存在
$$a({\phi _i},{\theta _i}) = a({\phi _j},{\theta _j}),\ \ i \ne j$$

## 均匀线阵的相位模糊

在讨论均匀线阵的模糊问题时，主要考虑阵元间距带来的相位模糊。  
设$\theta$为实际的入射信号，如果存在$a(\hat{\theta})=a(\theta)$，此时在运行 MUSIC 算法时，在$\theta$和$\hat{\theta}$都会出现谱线峰值。  
在均匀线阵中 DOA 估计的角度范围为$[ - {\pi  \over 2},{\pi  \over 2}]$，根据导向矢量的表达式
$$a(\theta ) = \begin{bmatrix} e^{j\omega {\tau_0}} & e^{j\omega {\tau_1}} & \dots & e^{j\omega {\tau_{M-1}}}\end{bmatrix}$$
一阶模糊出现的条件为
$$2\pi \frac{d}{\lambda}sin\hat{\theta}=2\pi \frac{d}{\lambda}sin\theta+2k\pi,\ \ k\in Z$$
其$d$为相邻两阵元的间距，化简得
$$sin\hat{\theta}=sin\theta+\frac{k}{d / \lambda},\ \ k \in Z$$
根据$sin\theta$在$[ - {\pi  \over 2},{\pi  \over 2}]$上的单调性，我们可以知道一阶模糊是否出现，等价于是否存在除零以外的整数 k 使得$(sin\theta+\frac{k}{d / \lambda})\in[-1,1]$。于是可以推导出**均匀线阵出现一阶模糊的条件**：

- $d\le\frac{1}{2}\lambda$
  此时$|\frac{k}{d/\lambda}|\ge 2|k|$，为使$(sin\theta+\frac{k}{d / \lambda})\in[-1,1]$，k 只能取 0。
- $\frac{1}{2}\lambda<d<\lambda$
  此时$2|k|>|\frac{k}{d/\lambda}|> |k|$，取$|k|=1$可得如果入射角满足$|sin\theta|<\lambda /d -1$，不会出现多余峰值；否则会出现一个虚假谱峰。
- $d\ge \lambda$
  此时$|\frac{k}{d/\lambda}|\le |k|$，为使$(sin\theta+\frac{k}{d / \lambda})\in[-1,1]$，当$0\le sin\theta \le 1$时，k 至少还可以取-1；当$-1\le sin\theta \le 0$时，k 至少还可以取 1，所以此时一定会出现虚假谱峰。

## 均匀圆阵的相位模糊

均匀圆阵的相位模糊问题，一般只出现在阵列孔径较大，而阵元数较少的情况下。  
考虑一维均匀圆阵，即只考虑方位角$\theta$，俯仰角取 90 度。  
假设出现$a(\hat{\theta})=a(\theta)$，则在第 m 个阵元上有
$$2\pi\frac{r}{\lambda}cos(\hat{\theta}-(m-1)\frac{2\pi}{M})=2\pi\frac{r}{\lambda}cos(\theta-(m-1)\frac{2\pi}{M})+2k\pi$$
右式减左式可得
$$\rho_m=\frac{4\pi r}{\lambda}sin(\frac{\theta-\hat{\theta}}{2})sin(\frac{\theta+\hat{\theta}}{2}-(m-1)\frac{2\pi}{M})=2k\pi,k\in Z$$
同理，在第 m+1 个阵元上可以得到
$$\rho_{m+1}=\frac{4\pi r}{\lambda}sin(\frac{\theta-\hat{\theta}}{2})sin(\frac{\theta+\hat{\theta}}{2}-m\frac{2\pi}{M})=2k'\pi,k\in Z$$
所以
$$\frac{\rho_{m+1}}{\rho_m}=sin(\frac{\theta+\hat{\theta}}{2}-m\frac{2\pi}{M})/sin(\frac{\theta+\hat{\theta}}{2}-(m-1)\frac{2\pi}{M})=\frac{\phi_m}{\phi_{m+1}}=\frac{k}{k'}$$
即在存在一阶模糊时，上式的结果为有理数。  
根据三角函数的展开定理，可以将$\phi_{m+1}$用$\phi_m$表示
$$\phi_{m+1}=sin(\frac{\theta+\hat{\theta}}{2}-m\frac{2\pi}{M})=\phi_mcos\frac{2\pi}{M}-\sqrt{1-\phi_m^2}sin\frac{2\pi}{M}$$
于是可以得到**均匀圆阵出现一阶模糊的条件**（_必要条件_）：当$M\ge 5$且为奇数或者$M\ge 8$且为偶数时，$cos\frac{2\pi}{M}$和$\sin\frac{2\pi}{M}$为无理数，此时均匀圆阵不会出现一阶模糊。

[1] 跃郭, 王宏远, 陬周, Yue G. U. O., Hong-yuan Wang 和 Zou Zhou. 《阵元间距对 MUSIC 算法的影响》. 电子学报 35, 期 9 (2007 年 9 月 25 日): 1675.  
[2] Xiao, Wei, Xian-Ci Xiao 和 Heng-Ming Tai. 《Rank-1 ambiguity DOA estimation of circular array with fewer sensors》. 收入 The 2002 45th Midwest Symposium on Circuits and Systems, 2002. MWSCAS-2002., 3:III–III, 2002.
