# 背景

最近工作在把论文QuaRot中的方法应用到端到端泊车模型的W8A8量化上，泊车模型W8A16量化后评测指标基本上是与浮点模型对齐的，但W8A8量化后崩得特别厉害。于是打算将Attention等模块的激活值进行随机Hadamard矩阵旋转从而抑制outlier。

主要的几个模块长这样：

<img src="C:\Users\a1097\AppData\Roaming\Typora\typora-user-images\image-20260808232254518.png" alt="image-20260808232254518" style="zoom: 80%;" />

在适配的过程中遇到了以下两个问题：

1.   论文中用的是RMSNorm，泊车模型中的LayerNorm不满足$LayerNorm(XH)=LayerNorm(X)H$,其中H为随机Hadamard矩阵。
2.   在解决问题1后，最终的精度提升却微乎其微，完全没有论文中的效果，问题出在哪？也就是说，为什么随即旋转这种在LLM量化中几乎成为标配的方法在小参数模型上却几乎无效？

本文的主要目的就是记录这两个问题的解决过程。

























