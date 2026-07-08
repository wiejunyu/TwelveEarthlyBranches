# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

单文件的 3D 十二面骰子（正十二面体）网页应用，骰面为十二地支（子丑寅卯辰巳午未申酉戌亥）。点击"摇一摇"后骰子在屏幕内弹跳、旋转并随机停在某一面。全部逻辑集中在 `骰子.html` 一个文件中。

## 运行方式

无构建、无依赖安装步骤。Three.js 通过 CDN import map 加载（`unpkg.com/three@0.160.0`），因此：

- 需要**联网**（获取 Three.js 模块）
- 需通过 HTTP(S) 打开，不能用 `file://`（ES module import map 限制）。例如：
  - `python -m http.server 8000` 然后访问 `http://localhost:8000/骰子.html`
- 无 lint、无测试、无 CI

## 架构要点

单个 `<script type="module">` 内的渲染管线，理解顺序：

1. **几何与面数据提取**（`DodecahedronGeometry`）：Three.js 的十二面体每个面由 3 个三角形（9 个顶点）组成。代码遍历顶点求出 12 个 `faceCenters` / `faceNormals`，并为每个面计算 `faceUpDir`（面内切向"朝上"方向）。

2. **面对齐 `alignQuat(faceIdx)`**：核心数学。用面法线(→+Z)和面朝上方向(→+Y)构造旋转矩阵并取逆，得到"让指定面正对相机且文字正立"的四元数。摇骰结果的确定性依赖此函数。

3. **贴字纹理与 UV 重映射**：`createTexture(char)` 用 Canvas 2D 逐字生成纹理；随后手动重写几何体默认 UV（默认是极坐标投影，会扭曲文字），把每个面的 5 个去重顶点按极角排序映射到正五边形 UV，使每个面完整显示一个字。修改字体/底色时注意底色必须铺满整块 canvas，否则 UV 边缘会露色。

4. **摇一摇动画 `shake()`**：三条独立轨迹叠加，均由 `mixEase(t)` 驱动时间：
   - **位置**：Catmull-Rom 样条 (`cr`/`sample`) 在屏幕像素空间生成随机弹跳点，经 `px2world` 换算为 3D 世界坐标（骰子移动，背景不动）。
   - **对齐**：`slerp(startQuat, targetQuat, pow(et,2.5))`，后半段才明显对齐到目标面。
   - **自旋**：绕随机轴，`spinProg = et*(1-et³)` 先升后降、结束归零。
   - 组合顺序 `quaternion = spin * align`，保证动画结束时自旋归零、目标面精确朝前。结束后有 200ms `snap` 补偿对齐。

5. **渲染循环状态机**：平时由 `startRenderLoop` 持续渲染；`shake()` 期间取消循环、改由动画帧独占控制（`animId` vs `renderLoopId`），动画完成后 `done()` 重启循环。改动画逻辑时必须维护这两个 requestAnimationFrame ID 的互斥，否则会出现双重渲染。

`updateCamera()` 按屏幕宽高比自适应相机距离，保证竖屏/横屏下骰子都不出视野。
