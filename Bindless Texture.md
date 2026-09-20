# Bindless Texture 学习总结

## 1. Bindless Texture 是什么

Bindless Texture（无绑定纹理）不是一种特殊的纹理类型，而是一种**资源访问方式**。

传统方式：

```text
CPU 将纹理绑定到固定 Slot
        ↓
Shader 从 t0 / t1 / t2 访问
```

Bindless：

```text
大量资源放入 Descriptor 集合
        ↓
材质保存 textureIndex
        ↓
Shader 根据 index 动态选择资源
```

例如：

```hlsl
uint index = material.textureIndex;
Texture2D tex = ResourceDescriptorHeap[index];
```

核心：

> 将“每次 Draw Call 绑定具体纹理”，变成“Shader 根据索引自行选择纹理”。

---

## 2. Descriptor

Descriptor 是 GPU 资源的**描述信息**。

它本身不存储纹理像素，而是描述：

* 资源位置
* 资源类型
* Texture2D / Buffer / Cubemap 等
* Format
* Mip 信息
* GPU 如何访问资源

可以简单理解为：

```text
Texture 数据
    ↑
Descriptor
```

即：

> Descriptor = GPU 资源的说明书 + 引用。

---

## 3. Descriptor Heap

DirectX 12 使用 **Descriptor Heap**。

可以理解为一个大型 Descriptor 数组：

```text
Descriptor Heap

[0] → Texture A
[1] → Texture B
[2] → Texture C
[3] → Buffer A
...
```

Shader 可以根据索引访问：

```hlsl
ResourceDescriptorHeap[index]
```

---

## 4. Descriptor Set

Vulkan 使用 **Descriptor Set**。

大致组织结构：

```text
Descriptor Pool
    ↓
Descriptor Set
    ↓
Binding
    ↓
Descriptor / Descriptor Array
```

例如：

```text
Set 0
 └─ Binding 0
      ├─ Texture 0
      ├─ Texture 1
      ├─ Texture 2
      └─ ...
```

Shader 可以使用：

```text
set + binding + index
```

找到具体资源。

---

## 5. Descriptor Heap 不等于 Bindless

这是最重要的区别之一。

即使使用 Descriptor Heap，也可以仍然采用传统绑定。

例如：

```text
Heap:

[100] Albedo
[101] Normal
[102] Metallic
```

传统方式：

```text
这次 Draw：

t0 → Descriptor[100]
t1 → Descriptor[101]
t2 → Descriptor[102]
```

仍然属于传统资源绑定。

Bindless 则是：

```text
Material:

albedoIndex = 100
normalIndex = 101
```

Shader：

```hlsl
ResourceDescriptorHeap[albedoIndex]
ResourceDescriptorHeap[normalIndex]
```

因此：

> 是否 Bindless 的关键，不是“有没有 Descriptor Heap”，而是 Shader 是否可以通过动态索引自由访问大量资源。

---

## 6. Unity 是否使用 Descriptor

使用。

Unity 在不同图形 API 下会使用对应机制：

```text
DirectX 12
→ Descriptor Heap

Vulkan
→ Descriptor Set / Descriptor Pool

Metal
→ Metal 自己的资源绑定机制
```

例如 Unity 中：

```csharp
material.SetTexture("_BaseMap", texture);
```

底层到了 DX12，最终还是需要通过 Descriptor 将 Texture 提供给 GPU。

只是 Unity 将这些操作封装起来了。

---

## 7. Unity 是否支持 Bindless

Unity 底层具备 Descriptor 管理，但普通 Unity API / Shader 并没有把完整的 Bindless 资源访问能力直接开放出来。

通常开发者使用的是：

```text
Material Texture
Texture2DArray
Texture Atlas
Virtual Texturing
GPU Instancing
SRP Batcher
```

这些技术都不能直接等同于 Bindless。

---

## 8. Texture2DArray 和 Bindless 的区别

Texture2DArray：

```text
一个 Texture Resource

Layer 0
Layer 1
Layer 2
Layer 3
...
```

所有 Layer 属于同一个纹理资源。

Bindless：

```text
Descriptor Heap

[0] → Texture2D
[1] → 另一张 Texture2D
[2] → Cubemap
[3] → Buffer
...
```

这些可以是完全独立的 GPU Resource。

所以：

> Texture2DArray = 一个资源里的多个 Layer
> Bindless = 从大量独立资源中动态选择一个资源

---

## 9. 是否所有贴图都是 Bindless

不是。

Bindless 是**访问方式**，不是纹理自身属性。

同一张 Texture：

```text
Texture A
```

既可以：

```text
固定绑定到 t0
```

也可以：

```text
Descriptor Heap[125]
```

然后 Shader 使用：

```text
textureIndex = 125
```

动态访问。

实际引擎中通常也会混用：

```text
大量材质纹理
→ 可能适合 Bindless

Shadow Map
Depth Texture
Render Target
临时纹理
→ 可能继续固定绑定
```

---

# 核心关系

```text
纹理 / Buffer 数据
        ↑
    Descriptor
        ↑
Descriptor Heap / Descriptor Set
        ↑
     资源索引
        ↑
      Shader
```

最核心的一句话：

> **Bindless 的本质，是 Shader 使用动态索引，从一个大型资源描述符集合中直接选择需要访问的 GPU Resource。**
