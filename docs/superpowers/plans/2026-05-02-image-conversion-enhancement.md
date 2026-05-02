# 图片转换算法增强实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 改进图片转换算法，解决线条缺失、主体变白、背景误判问题，增加可调参数

**Architecture:** 在现有 v1.1.4 代码基础上进行保守改进：
1. 修改 `isBackgroundColor` 增加位置上下文判断
2. 新增 `isEdgeRegion` 判断是否为边缘区域
3. 修改 `enhanceColorBlocks` 使用可调参数
4. 添加 UI 参数控件

**Tech Stack:** 纯 JavaScript + Canvas API（无外部依赖）

---

## 文件映射

**修改文件：** `D:\其他项目\PinDou\index.html`

**涉及函数：**
- `isBackgroundColor` (行 ~1457) - 修改：增加边缘判断和敏感度参数
- `convertImageToPattern` (行 ~1276) - 修改：传递参数给算法
- `enhanceColorBlocks` (行 ~1344) - 修改：使用可调参数
- 新增 `isEdgeRegion` - 判断是否为边缘区域
- 新增 UI 控件 - 添加三个滑块

---

## Task 1: A2/A3 - 修改背景检测逻辑

**Files:**
- Modify: `index.html:1456-1465` (isBackgroundColor 函数)

- [ ] **Step 1: 查看当前 isBackgroundColor 实现**

```javascript
// 当前实现 (v1.1.4)
function isBackgroundColor(r, g, b, a) {
    if (a < 128) return true;
    const brightness = (r + g + b) / 3;
    const saturation = Math.max(r, g, b) - Math.min(r, g, b);
    if (brightness > 240 && saturation < 20) return true;
    return false;
}
```

- [ ] **Step 2: 替换为增强版 isBackgroundColor**

```javascript
/**
 * 检测是否为背景色
 * @param {number} r - 红色通道
 * @param {number} g - 绿色通道
 * @param {number} b - 蓝色通道
 * @param {number} a - Alpha 通道
 * @param {number} x - 像素 X 坐标
 * @param {number} y - 像素 Y 坐标
 * @param {number} width - 画布宽度
 * @param {number} height - 画布高度
 * @param {number} sensitivity - 背景敏感度 (0-10)
 * @returns {boolean}
 */
function isBackgroundColor(r, g, b, a, x, y, width, height, sensitivity = 5) {
    // 透明像素一定是背景
    if (a < 128) return true;

    const brightness = (r + g + b) / 3;
    const saturation = Math.max(r, g, b) - Math.min(r, g, b);

    // 根据敏感度计算阈值
    // sensitivity 0-10: 亮度阈值从 250 降到 235, 饱和度阈值从 25降到 10
    const brightnessThreshold = 250 - (sensitivity * 1.5);
    const saturationThreshold = 25 - (sensitivity * 1.5);

    // 非常亮的单色区域
    if (brightness > brightnessThreshold && saturation < saturationThreshold) {
        // 只有在边缘区域才判定为背景
        if (isEdgeRegion(x, y, width, height, 3)) {
            return true;
        }
    }

    return false;
}

/**
 * 判断是否为边缘区域
 * @param {number} x - 像素 X 坐标
 * @param {number} y - 像素 Y 坐标
 * @param {number} width - 画布宽度
 * @param {number} height - 画布高度
 * @param {number} margin - 边缘宽度（像素）
 * @returns {boolean}
 */
function isEdgeRegion(x, y, width, height, margin = 3) {
    return x < margin || x >= width - margin || y < margin || y >= height - margin;
}
```

- [ ] **Step 3: 提交代码**

```bash
git add index.html
git commit -m "feat: 增强背景检测逻辑，添加位置上下文判断"
```

---

## Task 2: A1 - 修改 enhanceColorBlocks 使用保守策略

**Files:**
- Modify: `index.html:1344-1454` (enhanceColorBlocks 函数)

- [ ] **Step 1: 查看当前 enhanceColorBlocks 实现**

当前 v1.1.4 使用固定阈值 `region.length <= 3` 来判断是否为小区域。

- [ ] **Step 2: 替换为参数化的 enhanceColorBlocks**

```javascript
/**
 * 色块增强处理
 * @param {number} denoiseLevel - 去噪强度 (0-10)
 * @param {number} edgeSharpness - 边缘锐度 (0-10)
 */
function enhanceColorBlocks(denoiseLevel = 3, edgeSharpness = 5) {
    const width = state.canvasWidth;
    const height = state.canvasHeight;

    // 根据去噪强度计算小区域阈值
    // denoiseLevel 0-10: minRegionSize 从 2 到 8
    const minRegionSize = Math.max(2, Math.floor(denoiseLevel * 0.6));

    // 颜色差异阈值
    const threshold = 15;

    const tempGrid = state.gridData.map(row => [...row]);
    const visited = Array.from({ length: height }, () => Array(width).fill(false));

    const directions = [[0, 1], [0, -1], [1, 0], [-1, 0]];

    function colorDistance(c1, c2) {
        if (!c1 || !c2) return Infinity;
        const rgb1 = hexToRgb(c1);
        const rgb2 = hexToRgb(c2);
        return Math.sqrt(
            Math.pow(rgb1.r - rgb2.r, 2) +
            Math.pow(rgb1.g - rgb2.g, 2) +
            Math.pow(rgb1.b - rgb2.b, 2)
        );
    }

    function floodFillMerge(startX, startY, targetColor) {
        const queue = [[startX, startY]];
        const region = [];

        while (queue.length > 0) {
            const [x, y] = queue.shift();
            if (x < 0 || x >= width || y < 0 || y >= height) continue;
            if (visited[y][x]) continue;
            if (tempGrid[y][x] !== targetColor) continue;

            visited[y][x] = true;
            region.push([x, y]);

            for (const [dx, dy] of directions) {
                queue.push([x + dx, y + dy]);
            }
        }

        return region;
    }

    // 遍历所有格子进行区域生长
    for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
            if (visited[y][x]) continue;

            const currentColor = tempGrid[y][x];
            const region = floodFillMerge(x, y, currentColor);

            // 使用参数化的小区域阈值
            if (region.length <= minRegionSize) {
                // 找到周围的主要颜色
                const neighborColors = {};
                for (const [rx, ry] of region) {
                    for (const [dx, dy] of directions) {
                        const nx = rx + dx, ny = ry + dy;
                        if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
                            const nc = tempGrid[ny][nx];
                            if (nc !== currentColor) {
                                neighborColors[nc] = (neighborColors[nc] || 0) + 1;
                            }
                        }
                    }
                }

                let maxCount = 0;
                let mainColor = null;
                for (const [color, count] of Object.entries(neighborColors)) {
                    if (count > maxCount) {
                        maxCount = count;
                        mainColor = color;
                    }
                }

                // 只有在边缘区域且差异不大时才合并
                if (mainColor &&
                    colorDistance(currentColor, mainColor) < threshold * 2 &&
                    isEdgeRegion(region[0][0], region[0][1], width, height, 2)) {
                    for (const [rx, ry] of region) {
                        state.gridData[ry][rx] = mainColor;
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 3: 提交代码**

```bash
git add index.html
git commit -m "feat: 参数化 enhanceColorBlocks，支持去噪强度和边缘锐度调节"
```

---

## Task 3: C - 添加 UI 参数控件

**Files:**
- Modify: `index.html:220-245` (图片转换面板 HTML)
- Modify: `index.html:1276` (convertImageToPattern 函数)

- [ ] **Step 1: 在图片转换面板中添加滑块控件**

找到图片转换面板（id="imageConvertPanel"），在网格尺寸选择下方添加：

```html
<div style="margin:16px 0;">
    <label>网格尺寸: <select id="gridSizeSelect">
        <option value="8">8×8</option>
        <option value="16">16×16</option>
        <option value="29">29×29</option>
        <option value="50">50×50</option>
        <option value="100" selected>100×100</option>
        <option value="custom">自定义</option>
    </select></label>
</div>

<!-- 新增：算法参数调节 -->
<div style="margin:16px 0; border-top:1px solid #0f3460; padding-top:16px;">
    <label style="display:block; margin-bottom:8px;">算法参数：</label>
    <div style="margin:8px 0;">
        <label style="font-size:12px;">去噪强度：<span id="denoiseValue">3</span></label><br>
        <input type="range" id="denoiseLevel" min="0" max="10" value="3"
            style="width:200px;"
            oninput="document.getElementById('denoiseValue').textContent=this.value" />
        <span style="font-size:11px; color:#888;">(值越大越保守，保留更多细节)</span>
    </div>
    <div style="margin:8px 0;">
        <label style="font-size:12px;">背景敏感度：<span id="sensitivityValue">5</span></label><br>
        <input type="range" id="bgSensitivity" min="0" max="10" value="5"
            style="width:200px;"
            oninput="document.getElementById('sensitivityValue').textContent=this.value" />
        <span style="font-size:11px; color:#888;">(值越大背景判断越严格)</span>
    </div>
    <div style="margin:8px 0;">
        <label style="font-size:12px;">边缘锐度：<span id="edgeValue">5</span></label><br>
        <input type="range" id="edgeSharpness" min="0" max="10" value="5"
            style="width:200px;"
            oninput="document.getElementById('edgeValue').textContent=this.value" />
        <span style="font-size:11px; color:#888;">(值越大线条越锐利)</span>
    </div>
</div>
```

- [ ] **Step 2: 修改 convertImageToPattern 传递参数**

找到 `convertImageToPattern` 函数，在获取尺寸设置后添加参数获取：

```javascript
function convertImageToPattern() {
    if (!uploadedImage) return;

    let gridWidth, gridHeight;

    // 获取尺寸设置
    const sizeSelect = document.getElementById('gridSizeSelect').value;
    if (sizeSelect === 'custom') {
        gridWidth = parseInt(document.getElementById('customWidth').value) || 50;
        gridHeight = parseInt(document.getElementById('customHeight').value) || 50;
        gridWidth = Math.max(8, Math.min(200, gridWidth));
        gridHeight = Math.max(8, Math.min(200, gridHeight));
    } else {
        gridWidth = gridHeight = parseInt(sizeSelect);
    }

    // 获取算法参数
    const denoiseLevel = parseInt(document.getElementById('denoiseLevel')?.value || '3');
    const bgSensitivity = parseInt(document.getElementById('bgSensitivity')?.value || '5');
    const edgeSharpness = parseInt(document.getElementById('edgeSharpness')?.value || '5');

    // ... 现有 canvas 绘制代码保持不变 ...

    // 修改像素转换循环，传递背景敏感度参数
    for (let y = 0; y < gridHeight; y++) {
        for (let x = 0; x < gridWidth; x++) {
            const idx = (y * gridWidth + x) * 4;
            const r = imageData.data[idx];
            const g = imageData.data[idx + 1];
            const b = imageData.data[idx + 2];
            const a = imageData.data[idx + 3];

            // 传递位置信息和敏感度给背景检测
            const isEmpty = isBackgroundColor(r, g, b, a, x, y, gridWidth, gridHeight, bgSensitivity);
            if (isEmpty) {
                state.gridData[y][x] = '#FFFFFF';
            } else {
                const matchedColor = findClosestColorLab(r, g, b);
                state.gridData[y][x] = matchedColor;
            }
        }
    }

    // 对小尺寸图像应用色块增强算法，传递参数
    if (gridWidth <= 100 || gridHeight <= 100) {
        enhanceColorBlocks(denoiseLevel, edgeSharpness);
    }

    renderCanvas();
    saveToHistory();
    hideImageConvertPanel();
}
```

- [ ] **Step 3: 提交代码**

```bash
git add index.html
git commit -m "feat: 添加图片转换算法参数控件（去噪/背景敏感度/边缘锐度）"
```

---

## Task 4: 验证和测试

- [ ] **Step 1: 语法检查**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('D:/其他项目/PinDou/index.html', 'utf8');
const scriptMatch = html.match(/<script>([\s\S]*?)<\/script>/);
if (scriptMatch) {
    try {
        new Function(scriptMatch[1]);
        console.log('JavaScript syntax is valid!');
    } catch(e) {
        console.log('Syntax error:', e.message);
    }
}
"
```

Expected: `JavaScript syntax is valid!`

- [ ] **Step 2: 手动测试场景**

1. 打开浏览器加载 index.html
2. 上传一张人物照片
3. 尝试不同的参数组合
4. 检查：线条是否保留、主体是否不变白、背景是否正确

- [ ] **Step 3: 提交最终版本**

```bash
git add -A
git commit -m "feat: 完成图片转换算法增强（v1.2.0）
- A部分：保守修复（边缘判断、色彩完整性）
- B部分：主体识别（边缘区域识别）
- C部分：可调参数（去噪/背景敏感度/边缘锐度）"
```

---

## 验证清单

| 功能 | 测试方法 | 预期结果 |
|------|----------|----------|
| 线条保留 | 上传动漫图片 | 线条清晰不断裂 |
| 主体不变白 | 上传白色衣服人物照 | 白色衣服保持原色 |
| 背景正确 | 上传纯色背景图片 | 背景正确识别并填白 |
| 参数可调 | 拖动滑块 | 效果随参数变化 |
