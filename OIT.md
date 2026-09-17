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

$$
z_i = \min \{z \mid z > z_{i-1}\}
$$

特点：

- 天然按照 **近到远** 获取透明层
- 每剥一层就需要再次绘制透明场景
- 层数越多，Pass 越多
- 可以配合 Front-to-Back 合成

---

## 3. Front-to-Back 透明合成

颜色累计：

$$
C += T \alpha C_i
$$

剩余透过率：

$$
T *= (1-\alpha)
$$

当 \(T\) 接近 0 时，可以提前结束。

---

## 4. A-Buffer

核心思想：

> 每个像素保存多个透明 Fragment，而不是只保存一个。

通常保存：

- Depth
- Color
- Alpha

之后再排序和透明合成。

---

## 5. PPLL

PPLL（Per-Pixel Linked List）是 A-Buffer 的一种 GPU 实现。

结构：

- 每个像素保存 `Head`
- 全局 `Node Buffer` 保存 Fragment
- Node 保存 Color、Alpha、Depth、Next
- 用原子操作分配 Node 和修改链表头
- Resolve 阶段遍历、按 Depth 排序、再混合

优点：

- 结果较精确
- 能处理复杂交叉透明几何

缺点：

- Node Buffer 占显存
- 原子操作
- 随机显存访问
- GPU 负载不均
- Resolve 阶段需要排序
- Node Pool 仍可能溢出

---

## 6. Weighted Blended OIT（WBOIT）

WBOIT 不保存和排序所有 Fragment，而是用权重近似前后关系。

### 颜色累计

$$
Accum.rgb += C \alpha w
$$

$$
Accum.a += \alpha w
$$

最终透明颜色：

$$
C_{trans} = \frac{Accum.rgb}{Accum.a}
$$

其中：

- \(C\)：Fragment 颜色
- \(\alpha\)：透明度
- \(w\)：人为设计的权重

真正参与加权平均的权重是：

$$
\alpha w
$$

### Revealage

Revealage 表示背景剩余透过率。

初始：

$$
R=1
$$

每个 Fragment：

$$
R *= (1-\alpha)
$$

最终：

$$
R = \prod_i (1-\alpha_i)
$$

### 最终合成

$$
C_{final}
=
C_{trans}(1-R)
+
C_{opaque}R
$$

因此：

- `Accum` 决定透明层整体是什么颜色
- `Revealage` 决定背景还能透出多少

---

## 7. WBOIT 权重

通常希望：

$$
距离越近 \Rightarrow w越大
$$

$$
距离越远 \Rightarrow w越小
$$

简单例子：

$$
w=(1-d)^n
$$

\(n\) 越大，越强调近处 Fragment，但远处贡献会更弱。

需要注意：

- 线性深度
- 非线性 Depth Buffer
- Reversed-Z

---

## 8. WBOIT 的 MRT 与 Blend State

### RT0：Accum

保存：

$$
(C\alpha w,\ \alpha w)
$$

Blend：

```text
One One Add
```

初始清为 0。

### RT1：Revealage

目标：

$$
R_{new}=R_{old}(1-\alpha)
$$

初始清为 1。

使用 MRT 是为了：

> 一次透明 Pass 同时写入 Accum 和 Revealage。

---

## 9. WBOIT 深度状态

通常：

```text
ZTest On
ZWrite Off
```

- `ZTest On`：和不透明物体深度比较
- `ZWrite Off`：透明 Fragment 之间不互相写深度，否则结果重新依赖绘制顺序

---

## 10. 各方案对比

### Depth Peeling
- 多 Pass
- 逐层剥离
- 精确但层数多时成本高

### PPLL
- 收集所有 Fragment
- 像素内排序
- 精确但显存、原子操作和排序成本较高

### WBOIT
- 不排序
- 用权重近似前后关系
- 性能稳定
- 颜色遮挡存在近似误差

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

WBOIT 核心公式：

$$
Accum.rgb += C\alpha w
$$

$$
Accum.a += \alpha w
$$

$$
R *= (1-\alpha)
$$

$$
C_{trans} = \frac{Accum.rgb}{Accum.a}
$$

$$
C_{final}
=
C_{trans}(1-R)
+
C_{opaque}R
$$
