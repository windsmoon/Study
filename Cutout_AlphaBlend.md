# Cutout + Alpha Blend 透明处理方案

## 核心思路

将同一个透明物体分两次绘制：

1. **不透明主体：Alpha Test / Cutout**
2. **半透明边缘：Alpha Blend**

这样可以让主体正常写入深度，同时让边缘保持平滑透明。

## Pass 1：绘制不透明主体

对 Alpha 较高的像素保留，其余像素丢弃：

```text
alpha >= threshold -> 保留
alpha < threshold  -> discard
```

典型状态：

```text
Blend Off
ZWrite On
ZTest On
```

作用：

- 主体写入 Depth Buffer
- 遮挡关系稳定
- 大部分区域不需要参与透明排序

## Pass 2：绘制半透明边缘

再绘制 Alpha 较低的边缘像素：

```text
0 < alpha < threshold
```

使用 Alpha Blend：

```text
Blend SrcAlpha OneMinusSrcAlpha
ZWrite Off
ZTest On
```

作用：

- 保留纹理边缘的半透明效果
- 边缘更加平滑
- 不写深度，避免把后面的透明像素错误挡掉

## 为什么半透明部分不写深度

如果半透明像素写入深度：

```text
透明 A -> 写入深度
透明 B -> ZTest Failed -> 无法参与混合
```

因此半透明 Pass 通常使用：

```text
ZTest On
ZWrite Off
```

即：

> 可以利用已有深度判断自己是否被遮挡，但自己不修改深度。

## 整体流程

```text
Pass 1
Cutout / Alpha Test
不透明主体
ZWrite On

↓

Pass 2
Alpha Blend
半透明边缘
ZWrite Off
```

## 优点

- 主体拥有正确的深度遮挡
- 大部分像素不用参与透明排序
- 边缘仍能保持平滑透明效果

## 局限

该方案不能彻底解决透明排序问题。

第二个 Pass 中的半透明像素之间，依然可能存在透明排序问题。

本质上是：

> 把大部分接近不透明的区域交给 Depth Buffer，只留下少量边缘像素进行 Alpha Blend。
