# 如何实现YOLO

## 前言

深度学习极大的促进了物品检测算法的发展。人们研发了很多物品检测的算法：OLO、SSD、Mask RCNN、RetinaNet。

如果想要真正的掌握一个算法，就让我们亲自的实现它。因此，让我们来实现YOLO。

全文分成五个部分：
- 理解YOLO的工作原理
- 生成网络
- 实现前向传播
- 阈值和非最大抑制
- 输入和输出流水线

## 什么是YOLO

YOLO，You only look once，是最快的物品检测算法之一。YOLO的核心是基于深度卷积神经网络来检测一个物品。虽然RetinaNet、SSD等算法已经比YOLO更加精确，但是YOLO特别适合于实时监测，并且也有很好的精确度。

YOLO是一个完全的卷积网络。

YOLOv2并不擅长于检测小物品。

YOLOv3引入了残差模块、省略连接、上采样。YOLOv3使用的是Darknet的变体，一个在Imagenet上训练出来的53层的神经网络。在检测物品的时候，我们组合成了106层的架构。因此，YOLOv3相比于YOLOv2速度有所下降，但是精度有了上升。


![YOLOv3](i/YOLOv3Architecture.png)

(来源: [What’s new in YOLO v3?](https://towardsdatascience.com/yolo-v3-object-detection-53fb7d3bfe6b))

YOLOv3最大的变化是能够在三个不同的级别去进行计算。YOLO是一个完全的卷积网络，它的最后一步是在特征图上使用```1*1```的内核进行扫描。在YOLOv3中，这个扫描过程会应用在三个不同的大小和位置。

具体来说，检测的内核是1 x 1 x (B x (5 + C) )。其中的B是在特征图中一个单元可以检测的边框的数量。5是指边框的四个属性和对于检测的信任度。C是类的数量。在YOLOv3针对COCO的训练中，B = 3，C = 80，因此内核的大小是1 x 1 x 255。这个产生的新的特征图的长宽都与之前的相同，只是检测出来的属性会做为深度。

YOLOv3通过32、16、8的下采样来进行预测。第一个检测是在第82层，在前81层，我们不断的下采样从而保证在第82层是步长为32的下采样。如果我们的图片是416x416，那么我们的特征图是13x13。我们使用1x1的内核进行检测，得到了13x13x255的内核。

类似的过程发生在94层和106层，只是步长分别是16和8。

因为在不同的层进行检测，从而优化了对小物品的检测情况。13x13适合检测大物品，52x52适合检测小物品，26x26适合检测中等物品。

针对每个图片，YOLOv3产生了更多的接近10倍边框。


## 学习资料

首先访问官方网站，观看视频，并且了解相关的资料。
- [Yolo官方网站](https://pjreddie.com/darknet/yolo/)

然后可以阅读YOLOV3的最新更新
- [What’s new in YOLO v3?](https://towardsdatascience.com/yolo-v3-object-detection-53fb7d3bfe6b)

之后等着我更新内部原理的讲解吧。

## 参考资料
- [What’s new in YOLO v3?](https://towardsdatascience.com/yolo-v3-object-detection-53fb7d3bfe6b)
- [How to implement a YOLO (v3) object detector from scratch in PyTorch](https://blog.paperspace.com/how-to-implement-a-yolo-object-detector-in-pytorch/)
