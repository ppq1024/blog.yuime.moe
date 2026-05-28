---
date: 2025-08-12
update: 2025-09-25
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
    \boldsymbol{w}_{t + 1} = \boldsymbol{w}_t - \eta_t \boldsymbol{g}_t
$$
其中 $\boldsymbol{w}$ 表示可优化参数，$\boldsymbol{g}$ 表示梯度，$\eta$ 表示学习率， $t$ 表示迭代次数。其等价于：
$$
    \boldsymbol{w}_{t + 1} = \argmin_{\boldsymbol{w}} (\boldsymbol{g}_t \cdot \boldsymbol{w} + \frac{1}{2} \frac{1}{\eta_t} \| \boldsymbol{w} - \boldsymbol{w}_t \|_2^2)
$$
读者自证不难。

OGD 的主要问题是简单的 L1 正则化无法有效引入参数的稀疏，这也是 FTRL 算法要解决的主要问题。

## FTRL

### L1 正则与稀疏性

L1 正则通过在优化目标中添加参数的 L1 范数作为正则项：
$$
    \mathcal{L}_r = \mathcal{L} + \lambda \| \boldsymbol{w}\|_1
$$
其中 $\mathcal{L}$ 和 $\mathcal{L}_r$ 分别表示正则化前和正则化后的优化目标，$\lambda$ 是一个超参数，用于控制正则项的强度。

由于 L1 范数梯度的不连续性，当 $|g_i| < \lambda, (g_i = \frac{\partial \mathcal{L}}{\partial w_i})$ 时，会在 $w_i = 0$ 处产生极小值点，从而在参数中引入稀疏性。

### FTRL 的改进

按照传统 OGD 的方法，正则项的梯度被包含进 $\boldsymbol{g}_t$ 中，以下我们使用 $\boldsymbol{g}_{t, r}$ 以避免混淆。此外，OGD 中稀疏性的失效也来源于每步优化中求取局部最优，且这也会导致模型额外的不稳定。

针对以上问题，FTRL 进行了两点改进：一是将正则项从优化目标中分离出来，二是累加各步的梯度以求取全局最优。
$$
    \boldsymbol{w}_{t + 1} =
    \argmin_{\boldsymbol{w}} (
        \boldsymbol{g}_{1: t} \cdot \boldsymbol{w}
        + \frac{1}{2} \sum_{s = 1}^t {
            \sigma_s
            \| \boldsymbol{w} - \boldsymbol{w}_s \|_2^2
        }
        + \lambda_1 \| \boldsymbol{w} \|_1
        + \lambda_2 \| \boldsymbol{w} \|_2^2
    )
$$
其中 $\boldsymbol{g}_{1: t} = \sum_{s = 1}^t \boldsymbol{g}_s$ 为累计梯度，$\sigma_s = \frac{1}{\eta_s} - \frac{1}{\eta_{s - 1}}$，故有 $\sigma_{1: t} = \sum_{s = 1}^t \sigma_s = \frac{1}{\eta_t}$。
整理后可得：
$$
    \boldsymbol{w}_{t + 1} =
    \argmin_{\boldsymbol{w}} (
        (\boldsymbol{g}_{1: t} - \sum_{s = 1}^t {
            \sigma_s \boldsymbol{w}_s
        })
        \cdot \boldsymbol{w}
        + (\frac{1}{\eta_t} + \lambda_2)
        \| \boldsymbol{w} \|_2^2
        + \lambda_1 \| \boldsymbol{w} \|_1
    )
$$
右侧的梯度为 $\boldsymbol{0}$ 时，有：
$$
    \boldsymbol{w}_{t+1} =
    -\frac{\eta_t}{1 + \eta_t \lambda_2}(
        \boldsymbol{z}_t
        + \lambda_1 \nabla \| \boldsymbol{w} \|_1 
    )
$$
其中 $\boldsymbol{z}_t = \boldsymbol{g}_{1: t} - \sum_{s = 1}^t {\sigma_s \boldsymbol{w}_s} = \boldsymbol{z}_{t - 1} + \boldsymbol{g}_t - \sigma_t \boldsymbol{w}_t$，每一步迭代通过累加更新。

考虑到 L1 范数梯度的不连续性，对于 $\boldsymbol{w}_t$ 的每个分量 $w_{t, i}$，有：
$$
    w_{t, i} = \begin{cases}
        0 &\text{if} \ |z_{t, i}| \leq \lambda_1, \\
        -\frac{\eta_t}{1 + \eta_t \lambda_2} (
            z_{t, i} - \operatorname{sign}(z_{t, i}) \lambda_1
        ) & \text{otherwise}.
    \end{cases}
$$

当 $\lambda_1 = \lambda_2 = 0$ 时，有：
$$
    \boldsymbol{w}_{t + 1} = \eta_t \sum_{s=1}^t {
        \sigma_s \boldsymbol{w}_s
    } - \eta_t \boldsymbol{g}_{1: t}
$$
由于 $\eta_t \sigma_{1: s} = 1$，第一项相当于历史参数的加权平均，记为 $\bar{\boldsymbol{w}_t}$：
$$
    \boldsymbol{w}_{t + 1}
    = \bar{\boldsymbol{w}_t}
    - \eta_t \boldsymbol{g}_{1: t}
$$

### 动态学习率

论文中使用的方案是对梯度较大的参数使用较小的学习率，不同参数的学习率不同，并在优化过程中动态更新：
$$
    \eta_{t, i} = \frac{\alpha}{
        \beta + \sqrt{\sum_{s = 1}^t {g_{s, i}^2}}
    }
$$
这是论文中的原式，对其参数进行调整以便于理解：
$$
    \eta_{t, i} = \frac{\eta}{
        1 + \alpha \sqrt{\sum_{s = 1}^t {g_{s, i}^2}}
    }
$$
这里的 $\eta$ 更符合对优化器指定学习率的习惯，$\alpha$ 则可以控制动态调整的幅度。

## PyTorch 实现

首先要明确一点，FTRL 的本质是优化器而不是模型，因此要实现 ```torch.optim.Optimizer```而非 ```torch.nn.Module```。实现的核心是```torch.optim.Optimizer.step()```方法，类似```torch.nn.Module.forward()```。这里只放核心代码，完整代码参考 [随附仓库](https://github.com/ppq1024/FTRL)。

```python
def ftrl(
    params: list[Tensor],
    grads: list[Tensor],
    n_buffer: list[Tensor],
    z_buffer: list[Tensor],
    *,
    lr: float,
    alpha: float,
    sparse: float,
    weight_decay: float,
    maximize: bool,
):
    for i, param in enumerate(params):
        grad = grads[i] if not maximize else -grads[i]
        n = n_buffer[i]
        z = z_buffer[i]
        eta_old = (torch.sqrt(n) * alpha + 1) / lr if n.sum() > 0 else 0
        n.addcmul_(grad, grad, value=1)
        eta_new = (torch.sqrt(n) * alpha + 1) / lr
        sigma = eta_new - eta_old
        z.add_(grad).addcmul_(sigma, param, value=-1)
        param.set_(-torch.nn.functional.softshrink(z, sparse) / (eta_new + weight_decay))
```

仓库里还有一份 FTRLAdam 实现，用 $\hat{\boldsymbol{m}}_t / (\sqrt{\hat{\boldsymbol{v}}_t} + \epsilon)$ 和 $\hat{\boldsymbol{v}}_t$ 代替 $\boldsymbol{g}_t$ 和 $\boldsymbol{g}_t^2$。
