---
layout: post
title:  "Implement a naive Bayes classifier for spam classification"
tags: [机器学习,cs229]
comments: true
description: "这是cs229的homework中的一道题，用朴素贝叶斯算法实现垃圾邮件的分类，并且使用多项事件模型和拉普拉斯平滑。"
keywords: ""
category: 机器学习
date:   2017-07-22 00:38:22 +0800
---


这是一道来自CS 229, Autumn 2016 Problem Set #2 的作业题，根据给定的数据，使用朴素贝叶斯算法来对垃圾邮件进行分类。


#### 概述

垃圾邮件分类中的特征向量一般都是离散值，根据lecture note上对垃圾邮件过滤器的讲述，我们知道特征向量x可以用来表示一封邮件，$ x=[x_{1},...,x_{n}]^{T}$中$x_{i}=1$就代表这封邮件包含在词汇集中的第i个字符。类似地，在这道习题中，也会有一个已经处理好的词汇集，列出了邮件中会出现的单词，但是这里面只存了一些出现频率为中等的词汇，因为偶尔出现和频繁出现的（例如of，the这类的）词汇分类的价值有限。当然还预先使用了一些调控算法使得形近的单词转为一样的词，例如“price”，“prices”，“priced”都看作是“price“。那么该题提供的词汇集有1448个单词。


#### 朴素贝叶斯思想

**条件独立**

朴素贝叶斯用到的假设就是条件独立假设：

>**Definition**: Given three sets of random variables X,Y and Z, we say X is **conditionally independent** of Y given Z, if and only if the probability distribution governing X is independent of the value of Y given Z; that is:

$$(\forall i,j,k) P(X=x_{i}|Y=y_{j},Z=z_{k})=P(X=x_{i}|Z=z_{k})$$


因此我们就可以将$P(X|Y)$的概率写成：
$$P(X_{1},...,X_{n}|Y) = \prod_{i=1}^{n} P(X_{i}|Y)$$

<!--more-->

**参数**

正如讲义与作业中设定的，我们设一些参数：

$$\phi_{i| y=1} & = p(x_{i}=1|\ y=1)$$

$$\phi_{i| y=0} & = p(x_{i}=1|\ y=0)$$

$$\phi_{i| y=1} & = p(y=1)$$

由于似然概率

$$L(\phi_{i| y=1},\phi_{i| y=0}, \phi_{i| y=1}) = \prod_{i=1}^{m} p(x^{(i)},y^{(i)})$$

因此取对数并求偏导设为零可求得参数值

$$\phi_{j|y=1} & = \frac{\sum_{i=1}^{m}1\{x_{j}^{(i)}=1 \land y^{(i)}=1 \}}{ \sum_{i=1}^{m} 1 \{ y^{(i)}=1\}}$$

$$\phi_{j|y=0} & = \frac{\sum_{i=1}^{m}1\{x_{j}^{(i)}=1 \land y^{(i)}=0 \}}{ \sum_{i=1}^{m} 1 \{ y^{(i)}=0\}}$$

$$\phi_{y} & = \frac{\sum_{i=1}^{m} 1 \{ y^{(i)}=1\}}{m}$$


**测试**

那么这样给定一个测试向量$X^{new}$，那么通过上面的参数我们就可以进行预测：

$$ Y^{new} = arg\ max_{y} \frac{p(y)\prod_{i}p(x_{i}|y)}{p(y=0)\prod_{i}p(x_{i}|y=0)+p(y=1)\prod_{i}p(x_{i}|y=1)} $$



#### 优化——拉普拉斯平滑

为防止概率事件为0，则需要加上一些平滑参数，从而得到该题下新的参数估计：

$$\phi_{j|y=1} & = \frac{\sum_{i=1}^{m}1\{x_{j}^{(i)}=1 \land y^{(i)}=1 \}+1}{ \sum_{i=1}^{m} 1 \{ y^{(i)}=1\}+2}$$

$$\phi_{j|y=0} & = \frac{\sum_{i=1}^{m}1\{x_{j}^{(i)}=1 \land y^{(i)}=0 \}+1}{ \sum_{i=1}^{m} 1 \{ y^{(i)}=0\}+2}$$

$$\phi_{y} & = \frac{\sum_{i=1}^{m} 1 \{ y^{(i)}=1\}+1}{m+2}$$



#### 垃圾邮件分类

**预处理**

习题给出了一些必要的调用函数，其中readMatrix函数对训练数据进行了预处理，从而得到spmatrix, tokenlist, trainCategory。

- spmatrix
readMatrix先得到一个稀疏矩阵matrix，用sparse实现，

```
...
(39,1033)     2
(10,1035)     1
(31,1035)     1
(18,1036)     4
(38,1036)     2
(2,1037)      1
...
```

该矩阵的行代表一个样例，列代表不同的的词汇token，而矩阵(i,j)元素则是在邮件i中第j个token出现的数量；

- tokenlist
这个就是词汇token向量；

- trainCategory
trainCategory也是一个向量。

**训练函数nb_train.m**

下面是朴素贝叶斯的训练算法，这里的trainMatrix将矩阵spmatrix转换为稀疏矩阵。

```c
[spmatrix, tokenlist, trainCategory] = readMatrix('MATRIX.TRAIN');

trainMatrix = full(spmatrix);
numTrainDocs = size(trainMatrix, 1); % trainMatrix矩阵的行数
numTokens = size(trainMatrix, 2);    % trainMatrix矩阵的列数  

% YOUR CODE HERE

     V = size(trainMatrix, 2); % trainMatrix矩阵的列数，特征向量X的维数
     neg = trainMatrix(find(trainCategory == 0), :); % neg矩阵表示的是那些标签为0的样本
     pos = trainMatrix(find(trainCategory == 1), :); % neg矩阵表示的是那些标签为1的样本
     neg_words = sum(sum(neg)); % 存储neg矩阵的元素之和
     pos_words = sum(sum(pos)); % 存储pos矩阵的元素之和
     neg_log_prior = log(size(neg,1) / numTrainDocs);  % y=1的先验概率
     pos_log_prior = log(size(pos,1) / numTrainDocs);  % y=0的先验概率

     for k=1:V,  % 对特征向量的每一个token计算它们的log先验概率，
       neg_log_phi(k) = log((sum(neg(:,k)) + 1) / (neg_words + V));  % 对应公式(1)，注意这里的token计算的是出现的个数
       pos_log_phi(k) = log((sum(pos(:,k)) + 1) / (pos_words + V));  % 对应公式(2) 
end
```

**测试函数nb_test.m**

```c
for k=1:numTestDocs,
       [i,j,v] = find(testMatrix(k,:)); 
       % 得到第k个测试样本 
       % i向量则是非零元素所在的行，这种情况下都是1
       % j向量则是非零元素所在的列，这就意味着可以知道有哪些token的个数非零 
       % v向量则是非零元素的值

       % 计算后验概率，由于是取log，因此这里有‘+’
       neg_posterior = sum(v .* neg_log_phi(j)) + neg_log_prior; 
       pos_posterior = sum(v .* pos_log_phi(j)) + pos_log_prior;

       if (neg_posterior > pos_posterior)
         output(k) = 0;
       else
         output(k) = 1;
       end
end
```

最后如果对所有训练集都进行训练的话，可以得到测试误差为1.63%。



>该博文为学习机器学习时的笔记，由于水平有限，可能会存在错误，欢迎指正。

