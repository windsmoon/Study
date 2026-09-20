# Anisotropic Filtering 各向异性过滤

## 1. 为什么需要各向异性过滤

当纹理以较大倾斜角度观察时，一个屏幕像素映射到纹理空间后，覆盖区域通常不是正方形，而是一个细长的椭圆：

```text
        长轴
<-------------------->

     ______________
   /                \
  |                  |
   \________________/
          ↑
         短轴
```

例如：

```text
短轴 = 2 texels
长轴 = 16 texels
```

各向异性比例：

$$
A = \frac{16}{2}=8
$$

---

## 2. 普通 Mipmap 的问题

普通 Bilinear / Trilinear Filtering 往往按照较大的纹理变化率选择 Mip。

例如：

```text
2 × 16 texels
```

如果按长轴 `16` 选择：

$$
LOD \approx \log_2(16)=4
$$

就会使用很低分辨率的 Mip。

这相当于把：

```text
2 × 16
```

近似成：

```text
16 × 16
```

短轴方向因此被过度过滤，最终看起来很糊。

---

## 3. 各向异性过滤的核心思想

各向异性过滤把两个方向分开处理：

```text
短轴 → 决定 Mip Level
长轴 → 决定额外采样数量 / 范围
```

即：

$$
LOD \approx \log_2(\text{短轴长度})
$$

而：

$$
Anisotropy \approx
\frac{\text{长轴}}{\text{短轴}}
$$

例如：

```text
短轴 = 2
长轴 = 16
```

那么：

$$
LOD \approx \log_2(2)=1
$$

$$
Anisotropy = 16/2=8
$$

可以粗略理解成：

```text
Mip ≈ 1
Aniso ≈ 8x
```

---

## 4. 短轴和长轴如何过滤

并不是：

```text
短轴 → 一个颜色
长轴 → 一个颜色

然后两个颜色混合
```

而是：

```text
长轴位置 0 → 短轴过滤 → C0
长轴位置 1 → 短轴过滤 → C1
长轴位置 2 → 短轴过滤 → C2
长轴位置 3 → 短轴过滤 → C3

C0、C1、C2、C3
       ↓
沿长轴加权
       ↓
Final Color
```

可以理解为：

$$
C=
\int_{\text{长轴}}
\left[
\int_{\text{短轴}}
T(x,y)\,dy
\right]dx
$$

也就是：

> **先短轴过滤，再沿长轴积分。**

---

## 5. Mipmap 在短轴过滤中的作用

GPU 一般不会真的沿短轴逐个 texel 现场积分。

MipMap 本身已经保存了不同尺度下的局部低通结果。

因此：

```text
短轴宽度
   ↓
选择合适 Mip
   ↓
Bilinear / Trilinear Sample
   ↓
近似得到该位置的短轴过滤结果 Ci
```

然后在长轴上重复多次。

---

## 6. 三线性过滤 + Anisotropic Filtering

假设计算出的：

$$
LOD=1.4
$$

那么长轴上的每一个采样点都会进行一次三线性过滤：

$$
C_i=
0.6\times Bilinear(Mip1)
+
0.4\times Bilinear(Mip2)
$$

例如：

```text
Mip1 Bilinear ─┐
               ├→ Trilinear → C0
Mip2 Bilinear ─┘

Mip1 Bilinear ─┐
               ├→ Trilinear → C1
Mip2 Bilinear ─┘

Mip1 Bilinear ─┐
               ├→ Trilinear → C2
Mip2 Bilinear ─┘
```

然后：

$$
C_{final}=\sum_i w_iC_i
$$

---

## 7. 长轴采样的权重

最简单的理解方式是等权平均。

例如 4 个长轴采样：

```text
●      ●      ●      ●
C0     C1     C2     C3
```

则：

$$
C_{final}
=
\frac{C_0+C_1+C_2+C_3}{4}
$$

即：

$$
w_i=\frac1N
$$

更复杂的过滤也可以让中心位置权重更大：

```text
●     ●     ●     ●     ●

0.1   0.2   0.4   0.2   0.1
```

一般形式：

$$
C_{final}
=
\frac{\sum_i w_iC_i}
{\sum_i w_i}
$$

权重通常来自某种 Filter Kernel：

$$
w_i=K(d_i)
$$

其中 `d_i` 是采样点到 footprint 中心的距离。

---

## 8. Aniso Level

`Aniso Level` 可以理解为允许处理的最大长宽比。

```text
Aniso 1  → ≈ 1:1
Aniso 2  → ≈ 2:1
Aniso 4  → ≈ 4:1
Aniso 8  → ≈ 8:1
Aniso 16 → ≈ 16:1
```

但：

```text
Aniso 8
```

并不意味着 GPU 一定严格采 8 次。

实际：

* Tap 数量
* Tap 位置
* 长轴权重
* Bilinear Sample 如何复用
* LOD 调整方式

通常属于 GPU 厂商的硬件实现细节。

---

# 核心总结

各向异性过滤本质上是在处理：

> **一个像素在纹理空间中不同方向缩放率不同的问题。**

核心可以记成：

$$
\boxed{\text{短轴决定 Mip LOD}}
$$

$$
\boxed{
\text{长轴决定额外采样范围和数量}
}
$$

最终：

```text
短轴
 ↓
选择 Mip
 ↓
每个长轴位置做 Bilinear / Trilinear
 ↓
得到 C0、C1、C2...
 ↓
沿长轴加权平均
 ↓
Final Color
```

相比普通 Trilinear：

> **MipMap 负责过滤短轴，各向异性的多次采样负责过滤长轴。**
