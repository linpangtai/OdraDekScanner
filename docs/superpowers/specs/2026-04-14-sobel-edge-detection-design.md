# Sobel 边缘检测后处理设计

## 概述

为 OdraDekScanner 项目实现一个基于 Sobel 算子的全屏后处理边缘检测效果。作为 Post Process Material 应用到全场景，对所有物体进行全局描边。

## 方案

**单通道全屏后处理**：在一个 Post Process Material 中同时对 Scene Depth 和 Scene Color 做 Sobel 边缘检测，加权混合后得到边缘遮罩，叠加到最终画面上。

## 材质架构

```
Post Process Material
Domain: Post Process
Blendable Location: After Tonemapping
│
├── Scene Texture: SceneColor (3x3 邻域采样)
├── Scene Texture: SceneDepth (3x3 邻域采样)
│
├── Sobel Depth (3x3 卷积) ──┐
│                             ├── 加权混合 ──→ 阈值化 ──→ 边缘遮罩
├── Sobel Color (3x3 卷积) ──┘
│
├── 边缘颜色参数
├── 边缘强度参数
│
└── Lerp(原始颜色, 边缘颜色, 边缘遮罩 * 强度) → Emissive Color 输出
```

## Sobel 算子

标准 3x3 Sobel 卷积核：

```
Gx = [-1  0  1]    Gy = [-1 -2 -1]
     [-2  0  2]         [ 0  0  0]
     [-1  0  1]         [ 1  2  1]

梯度幅值 = sqrt(Gx^2 + Gy^2)
```

对深度图和颜色图分别计算梯度幅值，结果加权混合。

## 可调参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| EdgeColor | LinearColor | (0, 1, 0) | 轮廓线颜色（绿色） |
| EdgeIntensity | Scalar | 2.0 | 轮廓线强度 |
| EdgeThreshold | Scalar | 0.1 | 检测阈值，越低越灵敏 |
| DepthSensitivity | Scalar | 1.0 | 深度边缘权重 |
| ColorSensitivity | Scalar | 1.0 | 颜色边缘权重 |

## 材质节点实现细节

### UV 偏移计算

使用 ViewSize 节点获取屏幕分辨率，计算单个像素的 UV 偏移量：
- `PixelOffset = 1.0 / ViewSize`

### 3x3 邻域采样

对 SceneDepth 和 SceneColor 分别在 9 个位置采样（中心 + 8 个邻域），用 UV 偏移定位。

### 深度 Sobel

1. 对 9 个深度采样值应用 Gx 和 Gy 卷积核
2. 计算梯度幅值 `sqrt(Gx^2 + Gy^2)`
3. 乘以 DepthSensitivity

### 颜色 Sobel

1. 对 SceneColor 转灰度（Luminance）
2. 对 9 个灰度采样值应用 Gx 和 Gy 卷积核
3. 计算梯度幅值 `sqrt(Gx^2 + Gy^2)`
4. 乘以 ColorSensitivity

### 混合与输出

1. 深度边缘 + 颜色边缘 → 混合边缘强度
2. `step(EdgeThreshold, 混合边缘强度)` → 二值化边缘遮罩
3. `lerp(SceneColor, EdgeColor, 边缘遮罩 * EdgeIntensity)` → 最终输出

## 部署方式

1. 创建 Post Process Material 资产
2. 在场景中添加 Post Process Volume
3. 设置 Volume 为 Unbound（全场景生效）
4. 将材质添加到 Volume 的 Post Process Materials 数组中

## 实现方式

纯材质节点实现，不涉及 C++ 代码修改。所有逻辑在 UE 材质编辑器中通过节点连线完成。

## 文件清单

- `Content/PostProcess/M_SobelEdgeDetection.uasset` — 后处理材质资产
- 场景中放置 Post Process Volume（通过 Level 配置）
