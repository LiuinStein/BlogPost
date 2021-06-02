### 0x00 索引目录

基于内容的图像检索（Content-Based Image Retrieval, CBIR），是计算机视觉方向上的一个研究领域。主要用于给定一张图片，在已知图像数据库中检索与之视觉内容最相似的图片。

### 0x01 论文笔记

[Block-based Image Matching for Image Retrieval](/anthologies/cbir/paper/block-based-image-matching-for-image-retrieval.md) 本文共4541字，阅读大约需要18分钟

> 本文介绍了一种以子图搜全图的方法。通过将原始图片分割为多个子图，然后对每个子图提取SURFCN特征并进行VLAD编码，以及使用VGG-f模型提取深度特征，将两个特征组合后作为块特征，使用这个块特征以及其所提出的相似度计算方法，检索时只需召回块特征相似度Top k的图片即可。
>
> 标称效果还算不错，但是SURFCN这个创新过于勉强，这点根据其在Oxford5K上的试验结果即可看出。方法可行，整体看上去还算可以。

