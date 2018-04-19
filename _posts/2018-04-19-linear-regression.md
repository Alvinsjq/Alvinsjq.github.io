---
layout: post
title:  "Python实现线性回归"
tags: [机器学习]
comments: true
description: "这是对cs229 note1 的翻译笔记，这篇讲义主要是介绍了监督学习中的线性回归和逻辑谛斯回归，以及指数家族与生成线性模型。"
category: 机器学习
keywords: "Coursera,Machine learning"
date:   2018-04-19 13:40:22 +0800
---

最近重拾机器学习，想着学习网上的把Ng课涉及到的几个基础的机器学习算法用python实现以下，第一章的练习就是线性回归的实现。

![效果图](https://github.com/Alvinsjq/ML_ALGo_Basic/blob/master/Coursera_ML_Python/images/lr_gra.png?raw=true)

## 重要算法与函数

#### Cost Function

- 假设函数（关于的线性方程）

$$h_{\theta}(x)=\theta_{0}+\theta_{1}x_{1}+\theta_{2}x_{2}=\sum_{i=0}^{n}{\theta_{i}x_{i}}=\theta^{T}x $$

- 损失函数

$$J(\theta) = \frac{1}{2} \sum_{i=1}^{m}{(h_{\theta}(x^{(i)})-y^{(i)})^{2}}$$

用向量表示则可以是：

$$ J(\theta)=\frac{1}{2} (X\theta-\vec{y})^{T}(X\theta-\vec{y}) = \frac{1}{2} \sum_{i=1}^{m} (h_{\theta}(x^{(i)}) - y^{(i)} )^{2}$$

- Python实现

```python
def _ComputerCost(X,y,theta):
     #这里实现损失函数的计算方法，
     #theta是n维向量 n*1
     #X是增加了一个截矩项的特征向量矩阵 m*n
     #y是对应真实的估计值向量 m*1
     m = y.size;
     hyp = np.dot(X,theta);
     a = hyp - y;
     coef = 1./(2*m);
     J = coef * np.dot(a.T,a);#返回一个实数
     return float(J); 
```

#### 梯度下降法

- 公式推导

梯度下降法的理论可以看之前翻译的[讲义](http://alvinsjq.purposes.cn/2017/cs229note1/)，主要用的公式如下：

$$\theta_{j}:= \theta_{j} - \alpha \frac{\partial }{\partial \theta_{j}}  J(\theta) $$

将其中的导数求解出来后得到：

$$\frac{\partial}{\partial \theta_{j}}J(\theta) = (h_{\theta}(x)-y)x_{j}$$

由此得到梯度下降的函数规则公式：

$$\theta_{j}:= \theta_{j} + \alpha  (y^{(i)}-h_{\theta}(x^{(i)})) x_{j}^{(i)}$$

其中alpha是学习率，在作业代码中可以看到它对线性回归算法的影响。



- Python实现

```python
def _gradientDescent(X,y,theta,alpha,iterations):
     # 通过梯度下降来得到theta的估计值
     # 根据循环次数和学习率alpha来更新theta参数
      m = y.size;
      J_histroy = [];#初始化cost的数组，用于存放cost的值
      theta_histroy = [];
      #梯度下降公式实现
     
      for iter in range(iterations) :
          a = np.dot(np.transpose(X),np.dot(X,theta)-y);
          theta_histroy.append(list(theta[:,0]))
          theta = theta - alpha * (1/m) * a;
          J_histroy.append(_ComputerCost(X,y,theta));#将每一次梯度下降的损失函数结果存起来
      return theta,J_histroy,theta_histroy;
```


## Exercise 1 实现

Ng课后的作业提供了该算法的实现案例，我也用Python重新实现了一遍，并附上了一些注释：

- [Linear Regression 1](https://github.com/Alvinsjq/ML_ALGo_Basic/blob/master/Coursera_ML_Python/ex1/Linear_regression1.ipynb)
    + 假设函数 Hyp
    + 损失函数 Cost Function
    + 最小二次乘法
    + 梯度下降算法
- [Linear Regression 1](https://github.com/Alvinsjq/ML_ALGo_Basic/blob/master/Coursera_ML_Python/ex1/Linear_regression2.ipynb)
    + 多维变量的线性回归
    + 特征标准化
        * 均值、标准差计算法
    + 学习率与循环次数对Cost函数收敛的影响
    + 最小二次乘法直接计算theta
        * 注意当X不是满秩矩阵时的情况（详见西瓜书）

由于自己是初学者，如有错误之处欢迎指正！



