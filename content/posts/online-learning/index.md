---
date: 2025-08-12
lastmod: 2026-06-01
name: online-learning
title: 在线学习与 FTRL 算法
draft: false
enableKatex: true
tags:
    - 人工智能
    - 论文阅读
---

## 前言

{{< notice note >}}
本文中包含大量公式，阅读时请做好心理准备。
{{< /notice >}}

最近在看在线学习（Online Learning）相关的东西，这篇主要参考 Google 2013 年的论文 [Ad Click Prediction: a View from the Trenches](https://doi.org/10.1145/2487575.2488200)，其给出了一份 Follow the Regularized Leader (FTRL) 的工程实现。

不过我的应用场景和 Google 还是差挺多的，但在考虑调整算法之前还是先看一下最近的研究进展比较好。此外查阅文档时发现 PyTorch 没有 FTRL 实现，就打算自己写一份，顺便也讲一下构建 PyTorch 模块（不单指 torch.nn.Module）时的一些注意点。

## 在线学习

与在线学习相对应的是离线学习，也是机器学习中最常见的情况：模型在线下训练，线上部署过程中不对模型的参数进行更新。而在线学习通常用于实时获取大量样本数据的场景，对模型的实时性要求较高，训练和推理过程同步进行，线上部署过程中模型的参数实时更新。

在线学习与离线学习的主要区别在于参数的优化过程，使用的模型完全相同，与之对应的是在线梯度下降（Online Gradient Descent, OGD）和随机梯度下降（Stochastic Gradient Descent, SGD）。OGD 相当于 SGD 在 batch size 为 1 时的情况：
$$
    \bm{w}_{t + 1} = \bm{w}_t - \eta_t \bm{g}_t
$$
其中 $\bm{w}$ 表示可优化参数，$\bm{g}$ 表示梯度，$\eta$ 表示学习率， $t$ 表示迭代次数。其等价于：
$$
    \bm{w}_{t + 1} = \argmin_{\bm{w}} (\bm{g}_t^{\mathsf{T}} \bm{w} + \frac{1}{2} \frac{1}{\eta_t} \| \bm{w} - \bm{w}_t \|_2^2)
$$
读者自证不难。

OGD 的主要问题是简单的 L1 正则化无法有效引入参数的稀疏，这也是 FTRL 算法要解决的主要问题。

## FTRL

### L1 正则与稀疏性

L1 正则通过在优化目标中添加参数的 L1 范数作为正则项：
$$
    \mathcal{L}_r = \mathcal{L} + \lambda \| \bm{w}\|_1
$$
其中 $\mathcal{L}$ 和 $\mathcal{L}_r$ 分别表示正则化前和正则化后的优化目标，$\lambda$ 是一个超参数，用于控制正则项的强度。

由于 L1 范数梯度的不连续性，当 $|g_i| < \lambda, (g_i = \frac{\partial \mathcal{L}}{\partial w_i})$ 时，会在 $w_i = 0$ 处产生极小值点，从而在参数中引入稀疏性。

### FTRL 的改进

按照传统 OGD 的方法，正则项的梯度被包含进 $\bm{g}_t$ 中，以下我们使用 $\bm{g}_{t, r}$ 以避免混淆。此外，OGD 中稀疏性的失效也来源于每步优化中求取局部最优，且这也会导致模型额外的不稳定。

针对以上问题，FTRL 进行了两点改进：一是将正则项从优化目标中分离出来，二是累加各步的梯度以求取全局最优。
$$
    \bm{w}_{t + 1} = \argmin_{\bm{w}} (\bm{g}_{1: t}^{\mathsf{T}} \bm{w} + \frac{1}{2} \sum_{s = 1}^t {\sigma_s \| \bm{w} - \bm{w}_s \|_2^2} + \lambda_1 \| \bm{w} \|_1 + \frac{1}{2} \lambda_2 \| \bm{w} \|_2^2)
$$
其中 $\bm{g}_{1: t} = \sum_{s = 1}^t \bm{g}_s$ 为累计梯度，$\sigma_s = \frac{1}{\eta_s} - \frac{1}{\eta_{s - 1}}$，故有 $\sigma_{1: t} = \sum_{s = 1}^t \sigma_s = \frac{1}{\eta_t}$。
整理后可得：
$$
    \bm{w}_{t + 1} = \argmin_{\bm{w}} ((\bm{g}_{1: t} - \sum_{s = 1}^t {\sigma_s \bm{w}_s})^{\mathsf{T}} \bm{w} + \frac{1}{2 \eta_t} (1 + \eta_t \lambda_2) \| \bm{w} \|_2^2 + \lambda_1 \| \bm{w} \|_1)
$$
右侧的梯度为 $\bm{0}$ 时，有：
$$
    \bm{w}_{t+1} = -\frac{\eta_t}{1 + \eta_t \lambda_2}(\bm{z}_t + \lambda_1 \nabla \| \bm{w} \|_1 )
$$
其中 $\bm{z}_t = \bm{g}_{1: t} - \sum_{s = 1}^t {\sigma_s \bm{w}_s} = \bm{z}_{t - 1} + \bm{g}_t - \sigma_t \bm{w}_t$，每一步迭代通过累加更新。

考虑到 L1 范数梯度的不连续性，对于 $\bm{w}_t$ 的每个分量 $w_{t, i}$，有：
$$
    w_{t, i} = \begin{cases}
        0 &\text{if} \ |z_{t, i}| \leq \lambda_1, \\
        -\frac{\eta_t}{1 + \eta_t \lambda_2} (z_{t, i} - \operatorname{sign}(z_{t, i}) \lambda_1) & \text{otherwise}.
    \end{cases}
$$

当 $\lambda_1 = \lambda_2 = 0$ 时，有：
$$
    \bm{w}_{t + 1} = \eta_t \sum_{s=1}^t {\sigma_s \bm{w}_s} - \eta_t \bm{g}_{1: t}
$$
由于 $\eta_t \sigma_{1: s} = 1$，第一项相当于历史参数的加权平均，记为 $\bar{\bm{w}_t}$：
$$
    \bm{w}_{t + 1} = \bar{\bm{w}_t} - \eta_t \bm{g}_{1: t}
$$

### 动态学习率

论文中使用的方案是对梯度较大的参数使用较小的学习率，不同参数的学习率不同，并在优化过程中动态更新：
$$
    \eta_{t, i} = \frac{\alpha}{\beta + \sqrt{\sum_{s = 1}^t {g_{s, i}^2}}}
$$
这是论文中的原式，对其参数进行调整以便于理解：
$$
    \eta_{t, i} = \frac{\eta}{1 + \alpha \sqrt{\sum_{s = 1}^t {g_{s, i}^2}}}
$$
这里的 $\eta$ 更符合对优化器指定学习率的习惯，$\alpha$ 则可以控制动态调整的幅度。

## 一些其他的内容

前文主要基于 Google 的论文，这更多是一份在推荐算法领域的工程实现，而非算法本身，很多地方都没有讲清楚 ~~（前人讲过的东西肯定不用再讲了啊，直接引用就好）~~。此外，这篇论文也比较老了，一些后续的研究也有了很多启发式的观点。

### 补充 FTRL 定义

一般形式定义：
$$
\begin{gathered}
    \bm{\Delta}_t = \argmin_{\bm{x}} (\Phi_t(\bm{x}) + \bm{v}_{1: t} \cdot \bm{x}) \\
    \bm{w}_{t + 1} = \bm{w}_{t} + \bm{\Delta}_t
\end{gathered}
$$
这是基于增量的视角，其等效为基于更新的视角:
$$
    \bm{w}_{t + 1} = \argmin_{\bm{w}} (\Phi_t(\bm{w} - \bm{w}_{t}) + \bm{v}_{1: t} \cdot \bm{w})
$$
更一般的情况下不具备马尔科夫性:
$$
    \bm{w}_{t + 1} = \argmin_{\bm{w}} (\Phi_t(\bm{w}, \{\bm{w}_{1: t}\}) + \bm{v}_{1: t} \cdot \bm{w})
$$
其中 $\Phi_t(\cdot)$ 表示正则函数，$\{\bm{w}_{1: t}\}$ 表示历史权重状态集合，$\bm{v}_t$ 表示时刻 $t$ 的在线更新损失向量，通常基于历史梯度 $\{\bm{g}_{1: t}\}$。

### L2 正则与权重衰减

首先考虑 SGD中的 L2 正则项：
$$
\begin{gathered}
    \tilde{L} = L + \frac{1}{2} \lambda_2 \|\bm{w}\|_2^2 \\
    \bm{\Delta}_t = -\eta_t (\bm{g}_t + \lambda_2 \bm{w}_t)
\end{gathered}
$$

在 SGD 中，L2 正则引入了两个效果，一是权重衰减，这是核心的正则化，二是收敛点偏移，从 $\bm{g} = \bm{0}$ 移动到 $\bm{g} + \lambda_2 \bm{w} = \bm{0}$，考虑到目标函数的局部凸性，新的收敛点更接近原点。

考虑 FTRL-Proximal：
$$
\begin{aligned}
    \bm{w}_{t + 1} &= \argmin_{\bm{w}}(\bm{g}_{1: t}^{\mathsf{T}} \bm{w} + \frac{1}{2} \sum_{s = 1}^t {\|\bm{w} - \bm{w}_s\|_{\mathbf{Q}_s}^2} + \lambda_1 \|\bm{w}\|_1 + \frac{1}{2} \lambda_2 \|\bm{w}\|_2^2) \\
    &= \argmin_{\bm{w}}((\bm{g}_{1: t}^{\mathsf{T}} - \sum_{s = 1}^t {\bm{w}_s^{\mathsf{T}} \mathbf{Q}_s}) \bm{w} + \frac{1}{2} \bm{w}^{\mathsf{T}} \mathbf{Q}_{1: t} \bm{w} + \lambda_1 \|\bm{w}\|_1 + \frac{1}{2} \lambda_2 \|\bm{w}\|_2^2) \\
    &= (\mathbf{Q}_{1: t} + \lambda_2 \mathbf{I})^{-1} \operatorname{shrink}(\sum_{s = 1}^t {\mathbf{Q}_s \bm{w}_s} - \bm{g}_{1: t}, \lambda_1)
\end{aligned}
$$
其中 $\|\bm{x}\|_{\mathbf{Q}}^2 = \bm{x}^{\mathsf{T}} \mathbf{Q} \bm{x}$， $\mathbf{Q}_{1: t}$ 为半正定矩阵。当 $\lambda_1 = 0$ 时，有：
$$
\begin{gathered}
    (\mathbf{Q}_{1: t} + \lambda_2 \mathbf{I}) \bm{w}_{t + 1} = \sum_{s = 1}^t {\mathbf{Q}_s \bm{w}_s} - \bm{g}_{1: t} \\
    (\mathbf{Q}_{1: t} + \lambda_2 \mathbf{I}) \bm{w}_{t + 1} - (\mathbf{Q}_{1: t - 1} + \lambda_2 \mathbf{I}) \bm{w}_t = \mathbf{Q}_t \bm{w}_t - \bm{g}_t \\
    \bm{w}_{t + 1} = \bm{w}_t - (\mathbf{Q}_{1: t} + \lambda_2 \mathbf{I})^{-1} \bm{g}_t
\end{gathered}
$$
由此可见，L2 正则项的实际作用是限制动态学习率，避免因 $\mathbf{Q}_{1: t}$ 奇异导致发散。

若需要权重衰减，预期的结果为：
$$
\begin{gathered}
    \bm{w}_{t + 1} = \bm{w}_t - (\mathbf{Q}_{1: t} + \lambda_2 \mathbf{I})^{-1} (\bm{g}_t + \lambda_d \bm{w}_t) \\
    (\mathbf{Q}_{1: t} + \lambda_2 \mathbf{I}) \bm{w}_{t + 1} - (\mathbf{Q}_{1: t - 1} + \lambda_2 \mathbf{I}) \bm{w}_t = \mathbf{Q}_t \bm{w}_t - \bm{g}_t - \lambda_d \bm{w}_t \\
    \bm{w}_{t + 1} = (\mathbf{Q}_{1: t} + \lambda_2 \mathbf{I})^{-1} (\sum_{s = 1}^t {\mathbf{Q}_s \bm{w}_s} - \bm{g}_{1: t} - \lambda_d \bm{w}_{1: t})
\end{gathered}
$$
补充 L1 正则，有如下对应：
$$
\begin{gathered}
    \Phi_t(\bm{w}) = \frac{1}{2} \sum_{s = 1}^t {\|\bm{w} - \bm{w}_s\|_{\mathbf{Q}_s}^2} + \lambda_1 \|\bm{w}\|_1 + \frac{1}{2} \lambda_2 \|\bm{w}\|_2^2 \\
    \bm{v}_t = \bm{g}_t + \lambda_d \bm{w}_t
\end{gathered}
$$
这实际上与 SGD 中的权重衰减等效。

### $\beta$-FTRL

$\beta$-FTRL 是 Adam 的等价形式：
$$
\begin{gathered}
    \bm{w}_{t + 1} = \argmin_{\bm{w}}(\bm{v}_{1: t}^{\mathsf{T}} \bm{w} + \frac{1}{2} \|\bm{w} - \bm{w}_t\|_{\mathbf{Q}_t}^2 + \frac{1}{2} \epsilon \|\bm{w}\|_2^2) \\
    \mathbf{Q}_t = \operatorname{diag} \left(\frac{1}{\alpha_t} (\frac{\beta_2}{\beta_1})^t \sqrt{\sum_{s = 1}^t (\beta_2^{-s} \bm{g}_s)^2} \right) \\
    \bm{v}_t = \beta_1^{-t} \bm{g}_t \\
\end{gathered}
$$
其更新量为：
$$
\begin{gathered}
    \bm{\Delta}_t = -\frac{\alpha_t \sum_{s = 1}^t {\beta_1^{t - s} \bm{g}_s}}{\sqrt{\sum_{s = 1}^t (\beta_2^{t-s} \bm{g}_s)^2} + \epsilon'}
\end{gathered}
$$

## PyTorch 实现

首先要明确一点，FTRL 的本质是优化器而不是模型，因此要实现 ```torch.optim.Optimizer```而非 ```torch.nn.Module```。实现的核心是```torch.optim.Optimizer.step()```方法，类似```torch.nn.Module.forward()```。代码参考 [随附仓库](https://github.com/ppq1024/FTRL)。

*~~仓库更新是好久之前的了，我真的忘记当时是怎么写的了，找时间重新推导吧。~~*

## 参考文献

1. [McMahan, H. B., Holt, G., Sculley, D., Young, M., Ebner, D., Grady, J., ... & Badger, E. "Ad Click Prediction: a View from the Trenches." In *Proceedings of the 19th ACM SIGKDD International Conference on Knowledge Discovery and Data Mining (KDD 2013)*, pp. 1222-1230.](https://doi.org/10.1145/2487575.2488200)

2. [Ahn et al. "Understanding Adam Optimizer via Online Learning of Updates: Adam is FTRL in Disguise." arXiv:2402.01567, 2024. (In: *Proceedings of the 41st International Conference on Machine Learning (ICML 2024)*)](https://arxiv.org/abs/2402.01567v2)