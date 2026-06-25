姚苏珂-202411081026-计算机师范
# 光线投射与光线追踪实验

## 实验概述

本实验实现了基于 Whitted-Style 光线追踪模型的光线追踪渲染器，通过迭代方式模拟光线在场景中的传播，实现了硬阴影和镜面反射效果。实验使用 Taichi 语言编写，充分利用 GPU 并行计算能力。

## 实验目标达成

### 1. 理论理解：光线投射 vs 光线追踪

- **光线投射（Ray Casting）**：从摄像机发射主光线到场景，计算最近的交点，仅计算该点的直接光照，光线不再继续传播。
- **光线追踪（Ray Tracing）**：在光线投射基础上，当光线击中表面时，根据材质特性继续发射次级射线（反射射线、折射射线等），模拟更复杂的光学现象。

### 2. 全局光照效果

- **硬阴影**：从交点向光源方向发射暗影射线（Shadow Ray），检测是否有物体遮挡，实现阴影效果
- **理想镜面反射**：镜面材质表面根据反射定律计算反射方向，生成新的反射射线继续传播

### 3. GPU 编程思维

将传统递归光线追踪改写为适合 GPU 的**迭代循环**模式：
- 定义 `throughput` 累积光线能量衰减
- 使用 `for` 循环替代递归调用
- 镜面材质：继续循环，更新射线起点和方向
- 漫反射材质：计算颜色后 `break` 终止

## 场景搭建

### 几何体定义（隐式曲面）

| 物体 | 位置 | 半径 | 材质 | 颜色 |
|------|------|------|------|------|
| 红色漫反射球 | (-1.2, 0.0, 0.0) | 1.0 | 漫反射 | RGB(0.8, 0.1, 0.1) |
| 银色镜面球 | (1.2, 0.0, 0.0) | 1.0 | 镜面反射 | RGB(0.9, 0.9, 0.9) |
| 无限大地平面 | y = -1.0 | - | 漫反射 | 黑白棋盘格 |

### 棋盘格纹理实现

```python
ix = ti.floor(p.x * grid_scale)
iz = ti.floor(p.z * grid_scale)
if (ix + iz) % 2 == 0:
    hit_c = ti.Vector([0.3, 0.3, 0.3])  # 灰色格子
else:
    hit_c = ti.Vector([0.8, 0.8, 0.8])  # 白色格子
```

## 关键技术实现

### 1. 迭代式光线追踪核心代码

```python
for bounce in range(max_bounces[None]):
    t, N, obj_color, mat_id = scene_intersect(ro, rd)
    
    if mat_id == MAT_MIRROR:
        # 镜面：生成反射射线，继续追踪
        ro = p + N * 1e-4  # 法线偏移防止自相交
        rd = normalize(reflect(rd, N))
        throughput *= 0.8 * obj_color
        # 继续下一次循环
        
    elif mat_id == MAT_DIFFUSE:
        # 漫反射：计算光照后终止
        final_color += throughput * direct_light
        break
```

### 2. 阴影检测

```python
# 发射暗影射线
shadow_ray_orig = p + N * 1e-4
shadow_t, _, _, _ = scene_intersect(shadow_ray_orig, L)

dist_to_light = (light_pos - p).norm()
if shadow_t < dist_to_light:
    in_shadow = True  # 处于阴影中
```

### 3. 反射向量计算

$$\mathbf{R} = \mathbf{L}_{in} - 2(\mathbf{L}_{in} \cdot \mathbf{N})\mathbf{N}$$

其中 $\mathbf{L}_{in}$ 为入射光线方向，$\mathbf{N}$ 为表面法向量。

### 4. 浮点数精度问题修复

**Shadow Acne（阴影痤疮）问题**：由于浮点精度误差，射线可能与自身表面相交。

**解决方案**：将射线起点沿法线方向偏移微小量

```python
ro = p + N * 1e-4  # 反射射线起点偏移
shadow_ray_orig = p + N * 1e-4  # 暗影射线起点偏移
```

## 用户交互界面

通过 `ti.ui.Window` 提供实时交互面板：

| 控件 | 范围 | 默认值 | 功能 |
|------|------|--------|------|
| Light X | -5.0 ~ 5.0 | 2.0 | 光源 X 坐标 |
| Light Y | 1.0 ~ 8.0 | 4.0 | 光源 Y 坐标 |
| Light Z | -5.0 ~ 5.0 | 3.0 | 光源 Z 坐标 |
| Max Bounces | 1 ~ 5 | 3 | 最大光线弹射次数 |

### 观察效果

- **Max Bounces = 1**：无反射效果，镜面球显示黑色
- **Max Bounces > 1**：出现镜中世界，镜面球反射周围环境
- <img width="480" height="395" alt="TFa9SwN3_converted" src="https://github.com/user-attachments/assets/651425d4-1a25-4c73-85de-2289f1447235" />

## 运行方法

```bash
# 安装依赖
pip install taichi

# 运行程序
python ray_tracing.py
```

## 技术参数

| 参数 | 值 |
|------|-----|
| 分辨率 | 800 × 600 |
| 摄像机位置 | (0, 1, 5) |
| 摄像机方向 | (0, -0.2, -1) |
| 镜面反射率 | 0.8 |
| 环境光强度 | 0.2 |
| 漫反射强度 | 0.8 |

## 实验总结

通过本实验，我们：

1. ✅ 理解了光线投射与光线追踪的本质区别
2. ✅ 实现了硬阴影和理想镜面反射效果
3. ✅ 掌握了 GPU 上的迭代式光线追踪编程模式
4. ✅ 解决了光线追踪中常见的浮点数精度问题
5. ✅ 构建了交互式实时渲染界面
