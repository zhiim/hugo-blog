+++

title = "机器学习 18 主成分分析"
date = 2025-09-26T19:57:00+08:00
slug = "ml_18_principal_components_analysis"
description = "无监督学习，主成分分析"
tags = ["机器学习"]
categories = ["Notes"]
image = "/p/ml_01_introduction/ml_header.webp"

+++

## 无监督学习

有时候我们的训练数据里面没有标签，但是我们还是希望寻找数据的结构信息

无监督学习的应用主要分为如下几个大类：

- 聚类（clustering）：将数据划分成几个相似的点集
- 数据降维（dimensionality reduction）：数据一般分布在特征空间的低纬子空间，或者矩阵一般有低阶秩近似
- 分布学习（density estimation）：使用离散数据拟合一个连续概率分布

## 主成分分析 Principal Components Analysis

目标：给定一些 $\mathbb{R}^d$ 中的样本点，寻找 k 个方向，可以捕获大部分特征变化

主成分分析一般有以下几个目的：

- 降低维度可以降低计算复杂度
- 移除了干扰维度可以防止过拟合，和特征子集选择类似，但是此时选出来的特征是其他特征的线性组合
- 寻找一个较小的可以表征复杂变化的基

假设 $X$ 是一个 $n \times d$ 的设计矩阵，并且 $X$ 的均值为 0。假设有一些正交基 ${} v_1, \dots, v_k {}$，那么 x 可以在正交基下表示为 $\widetilde{x} = \sum_{i = 1}^k (x \cdot v_i) v_i$，其中 $x \cdot v_i$ 表示主坐标，是 x 在 $v_i$ 方向投影的长度

在降维的时候我们希望最大程度保留原始信息，在主成分分析中信息被定义为方差，所以最佳的方向就是协方差矩阵 $X^TX$ 的特征向量。

假设 $X^TX$ 的特征特征值为 $0 \le \lambda_1 \le \lambda_2 \le \dots \le \lambda_d$，对应到特征向量 $v_1, v_2, \dots, v_k$ 就是主成分。计算每个数据点在主成分下的主坐标，就是实现了对数据的降维
