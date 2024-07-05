[[Android]对于Android：Layout_weight的深刻理解-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1596187)

layout_weight属性的真正含义是将当前屏幕宽度或高度的**剩余空间**按比例分配然后加到控件尺寸上。

例如有3个控件A、B、C，最终在屏幕呈现的宽度公式为：

1. 剩余空间宽度 = 屏幕宽度 - 控件A宽度 - 控件B宽度 - 控件C宽度

可见剩余空间宽度可能为负数，比如控件A是match_parent模式

1. 控件A在屏幕上呈现的宽度 = 控件A宽度 + layout_weight_A / （layout_weight_A + layout_weight_B + layout_weight_C）* 剩余空间宽度
2. 同理控件B、控件C在屏幕上呈现的宽度计算方式和控件A相同
3. 注意，因为剩余空间宽度可能为负数，那么根据第2点的计算公式，那么可能导致计算后控件的宽度为0，也就是不展示。


