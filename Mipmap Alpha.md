# MipMap 中 Alpha 的处理总结

## 1. 普通 Alpha Blend

Alpha 表示连续的透明度权重。

生成下一层 Mipmap 时，Alpha 通常直接做过滤/平均：

$$
\alpha_{mip} = \text{Average}(\alpha_i)
$$

例如：

$$
1,\ 1,\ 0,\ 0 \rightarrow 0.5
$$

对于 Alpha Blend 来说，这个结果通常是合理的，因为 `0.5` 就表示大约 50% 的透明度贡献。

---

## 2. Alpha Test / Cutout

Alpha Test 不是连续混合，而是根据阈值决定：

```hlsl
clip(alpha - cutoff);
```

* `alpha > cutoff`：保留像素
* `alpha <= cutoff`：丢弃像素

因此普通平均可能改变纹理的实际覆盖面积。

例如：

```text
1 1
1 0
```

平均后：

$$
\alpha=0.75
$$

如果：

$$
cutoff=0.5
$$

原本覆盖率是：

$$
3/4=75\%
$$

缩成一个像素后，该像素整体通过 Alpha Test，覆盖率会变成 100%。

---

## 3. Alpha Coverage Preservation

对于 Cutout 纹理：

1. 先正常过滤生成 Alpha Mip
2. 计算这一层通过 `cutoff` 的像素比例
3. 和原始纹理的 Coverage 比较
4. 调整这一层 Alpha，使 Coverage 尽量保持一致

Coverage 定义：

$$
Coverage=
\frac{\text{通过 Alpha Test 的像素数}}
{\text{总像素数}}
$$

常见做法是调整：

$$
\alpha'=\alpha \cdot s
$$

直到：

$$
Coverage_{mip}\approx Coverage_{原图}
$$

注意：

> Coverage 不是直接存进 Alpha，而是用来校正 Mipmap 的 Alpha。

---

# 4. RGB 与 Alpha 不能总是独立平均

假设：

* 红色：`RGB=(1,0,0), α=1`
* 蓝色：`RGB=(0,0,1), α=0`

如果直接平均 RGB：

$$
RGB=(0.5,0,0.5)
$$

会得到紫色。

但蓝色像素完全透明，本来不应该影响结果。

---

# 5. Premultiplied Alpha

先把 RGB 乘 Alpha：

$$
C'=C\alpha
$$

于是：

```text
红色 α=1   → 红色
蓝色 α=0   → 黑色
```

完全透明像素就不会污染 Mipmap 的颜色。

因此生成 Mipmap 时可以：

```text
Straight Alpha
      ↓ × Alpha
Premultiplied Alpha
      ↓ 过滤
Premultiplied Mip
```

---

# 6. 如果最终仍然要保存 Straight Alpha

Mip 过滤后得到：

$$
C'_{mip}
$$

和：

$$
\alpha_{mip}
$$

如果最终纹理需要 Straight Alpha，则反预乘：

$$
C_{mip}
=
\frac{C'_{mip}}{\alpha_{mip}}
$$

例如：

$$
C'_{mip}=(0.5,0,0.25)
$$

$$
\alpha_{mip}=0.75
$$

则：

$$
C_{mip}
=
(0.667,0,0.333)
$$

---

## 7. Alpha = 0 的情况

不能直接：

$$
RGB/\alpha
$$

因为会除以 0。

通常：

```c
if (alpha > epsilon)
    rgb /= alpha;
else
    rgb = 0;
```

但这里有一个前提：

> 整个 Mipmap 链必须使用 Alpha-aware / Premultiplied 的方式生成。

否则，即使 `alpha=0`，透明像素中的 RGB 仍可能在下一层普通平均时污染附近颜色。

---

# 8. Alpha Bleeding / Edge Padding

如果使用：

* Straight Alpha
* 普通 RGB 平均生成 Mipmap

那么透明区域最好不要填黑色，而是把边缘可见颜色向外扩展。

例如树叶：

```text
推荐：

绿色 α=1 | 绿色 α=0

而不是：

绿色 α=1 | 黑色 α=0
```

这样即使透明像素参与普通过滤，也不容易产生黑边。

---

# 9. Premultiplied Alpha 的 Blend

Straight Alpha：

$$
C_{out}
=
C_{src}\alpha
+
C_{dst}(1-\alpha)
$$

Blend Factor：

```text
SrcAlpha, OneMinusSrcAlpha
```

Premultiplied Alpha：

RGB 已经提前乘过 Alpha：

$$
C'=C\alpha
$$

所以：

$$
C_{out}
=
C'_{src}
+
C_{dst}(1-\alpha)
$$

Blend Factor：

```text
One, OneMinusSrcAlpha
```

如果 Premultiplied Alpha 又使用普通 Alpha Blend，就会再次乘 Alpha：

$$
C\alpha^2
$$

结果会偏暗。

---

# 最终记忆

```text
Alpha Blend
→ Alpha 直接平均通常没问题

Alpha Test / Cutout
→ 先平均
→ 再通过 Coverage 校正 Alpha
→ 保持缩小时的覆盖面积

RGB + Alpha
→ 最好先 Premultiply
→ 再生成 Mipmap
→ 避免透明像素 RGB 污染边缘

如果最终要 Straight Alpha
→ Mip 过滤完成后再除以 mip alpha
```

一句话总结：

> **Blend 关注 Alpha 的平均值，Cutout 关注通过阈值后的覆盖率；RGB 过滤则要考虑 Alpha 权重。**
