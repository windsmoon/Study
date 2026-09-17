# 顺序无关透明度（OIT）学习总结

## 1. 为什么需要 OIT

普通 Alpha Blend 依赖绘制顺序，通常要求透明物体按照 **远到近** 渲染。

当透明几何互相穿插时，不同像素的前后关系可能不同，无法只靠物体级排序彻底解决。

OIT（Order Independent Transparency）的目标是：

> 不依赖透明物体提交顺序，也能得到正确或近似正确的透明结果。

---

## 2. Depth Peeling

每一遍渲染都找到：

> 比上一层更远的所有 Fragment 中，最近的那一层。

可表示为：

\[
z_i = \min \{z \mid z > z_{i-1}\}
\]

特点：

- 天然按照 **近到远** 获取透明层
- 每剥一层就需要再次绘制透明场景
- 层数越多，Pass 越多，性能越高
- 可以配合 Front-to-Back 合成

---

## 3. Front-to-Back 透明合成

按近到远处理透明层。

颜色累计：

\[
C += T \alpha C_i
\]

剩余透过率：

\[
T *= (1-\alpha)
\]

其中：

- \(C\)：累计颜色
- \(T\)：剩余透过率
- 初始 \(T=1\)

当 \(T\) 接近 0 时，可以提前结束后续透明层计算。

---

## 4. A-Buffer

A-Buffer 的核心思想是：

> 每个像素保存多个透明 Fragment，而不是只保存一个。

每个 Fragment 通常需要保存：

- Depth
- Color
- Alpha

之后再进行排序和透明合成。

---

## 5. PPLL

PPLL（Per-Pixel Linked List）是 A-Buffer 的一种 GPU 实现。

基本结构：

- 每个像素保存一个 `Head`
- 全局 `Node Buffer` 保存所有透明 Fragment
- 每个 Node 保存：
  - Color
  - Alpha
  - Depth
  - Next

收集阶段：

1. 用原子操作从全局 Node Pool 分配 Node
2. 用原子操作修改当前像素的链表头
3. 保存 Fragment 数据

链表收集顺序不重要，因为 Resolve 阶段会：

1. 遍历链表
2. 按 Depth 排序
3. 按正确顺序进行 Alpha Blend

优点：

- 可以得到较精确的透明结果
- 能处理复杂交叉透明几何

缺点：

- Node Buffer 占显存
- 原子操作成本
- 链表随机显存访问
- 每像素 Fragment 数量不同，GPU 负载不均
- Resolve 时需要像素内排序
- Node Pool 仍然可能溢出

---

## 6. Weighted Blended OIT（WBOIT）

WBOIT 不保存和排序所有 Fragment。

核心思想：

> 使用满足交换律的累积运算，再用权重近似透明层的前后关系。

### 颜色累积

\[
Accum.rgb += C \alpha w
\]

权重累积：

\[
Accum.a += \alpha w
\]

最终透明颜色：

\[
C_{trans} =
\frac{Accum.rgb}{Accum.a}
\]

其中：

- \(C\)：Fragment 颜色
- \(lpha\)：透明度
- \(w\)：人为设计的权重

真正参与加权平均的权重是：

\[
\alpha w
\]

### Revealage

Revealage 表示背景剩余透过率。

初始值：

\[
R=1
\]

每经过一个透明 Fragment：

\[
R *= (1-\alpha)
\]

最终：

\[
R = \prod_i(1-\alpha_i)
\]

### 最终合成

\[
C_{final}
=
C_{trans}(1-R)
+
C_{opaque}R
\]

因此：

- `Accum` 决定透明层整体是什么颜色
- `Revealage` 决定背景还能透出多少

颜色使用加法累计，Revealage 使用乘法累计，两者都满足交换律，因此不依赖透明 Fragment 的提交顺序。

---

## 7. WBOIT 的权重

权重 \(w\) 用来近似“谁更靠前”。

通常希望：

\[
距离越近 \Rightarrow w越大
\]

\[
距离越远 \Rightarrow w越小
\]

简单例子：

\[
w=(1-d)^n
\]

其中 \(d\) 是归一化深度。

\(n\) 越大：

- 越强调近处 Fragment
- 越能减少远处颜色泄漏
- 但远处透明层贡献可能过弱
- 深度变化可能导致更明显的颜色变化

实际使用时还要注意：

- 线性深度
- 非线性 Depth Buffer
- Reversed-Z

---

## 8. WBOIT 的 MRT 与 Blend State

WBOIT 通常需要两个 Render Target：

### RT0：Accum

保存：

\[
(C\alpha w,\ \alpha w)
\]

Blend：

```text
One One Add
```

初始清为：

```text
0
```

### RT1：Revealage

目标：

\[
R_{new}=R_{old}(1-\alpha)
\]

使用对应的乘法式 Blend State。

初始清为：

```text
1
```

之所以使用 MRT，是因为：

> 一次透明 Pass 同时写入 Accum 和 Revealage，避免把透明物体再绘制一遍。

---

## 9. WBOIT 深度状态

透明 Pass 通常：

```text
ZTest On
ZWrite Off
```

原因：

### ZTest On

透明 Fragment 仍然需要和不透明物体的深度比较。

被不透明物体挡住的透明 Fragment 应该被剔除。

### ZWrite Off

透明 Fragment 之间不能互相写深度，否则先绘制的透明物体可能把后面的透明 Fragment 提前剔除，结果重新依赖绘制顺序。

---

## 10. 各方案对比

### Depth Peeling

特点：

- 精确
- 多 Pass
- 层数越多成本越高

适合：

- 透明层数较少
- 对结果精度要求高的情况

### PPLL

特点：

- 像素级保存 Fragment
- Resolve 时排序
- 结果较精确
- 显存、原子操作和排序成本较高

适合：

- 玻璃
- 高 Alpha 透明表面
- 复杂交叉透明物体
- 对前后关系精度要求较高的情况

### Weighted Blended OIT

特点：

- 不排序
- 性能稳定
- 一次透明累积 + Resolve
- 颜色遮挡是近似结果

适合：

- 烟雾
- 粒子
- 半透明特效
- 大量透明层
- 对颜色精度要求不是特别严格的场景

---

## 11. 核心记忆

```text
Depth Peeling
= 多次绘制，逐层剥离

PPLL
= 乱序收集，像素内排序，最后合成

WBOIT
= 不排序，用权重近似前后关系
```

WBOIT：

\[
Accum.rgb += C\alpha w
\]

\[
Accum.a += \alpha w
\]

\[
R *= (1-\alpha)
\]

\[
C_{trans} = \frac{Accum.rgb}{Accum.a}
\]

\[
C_{final}
=
C_{trans}(1-R)
+
C_{opaque}R
\]
