# PinDou 项目规则

**项目**：拼豆图案设计工具（纯前端单文件应用）

**最后更新**：2026-05-02

---

## 项目结构

```
PinDou/
├── index.html              # 单文件应用（所有代码内联）
├── RULES.md                 # 本规则文件
├── CHANGELOG.md             # 变更记录
├── docs/
│   └── superpowers/
│       └── specs/           # 设计文档
├── SPEC.md                  # 项目规格（链接到设计文档）
└── README.md                # 项目说明
```

---

## 代码组织

### 单文件内部结构（按顺序）

```javascript
// 1. 文件头部注释（项目名、版本、变更日志）
// 2. 常量配置（颜色数据、工具定义）
// 3. 状态管理（全局状态对象）
// 4. 工具函数（辅助函数）
// 5. 画布操作（Canvas 相关）
// 6. UI 交互（事件处理）
// 7. 初始化代码
```

---

## 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 函数 | camelCase | `initCanvas()`, `convertImage()` |
| 变量 | camelCase | `canvasWidth`, `currentColor` |
| 常量 | UPPER_SNAKE_CASE | `MAX_CANVAS_SIZE`, `DEFAULT_COLORS` |
| DOM 元素 ID | kebab-case | `color-palette`, `tool-bar` |
| CSS 类名 | kebab-case | `.btn-primary`, `.canvas-container` |

---

## HTML/CSS 规范

- **CSS**：全部放在 `<style>` 区块内，不使用外部 CSS 文件
- **语义化 HTML**：使用 `<button>` 而不是 `<div>` 做按钮
- **避免内联样式**：优先使用 CSS 类
- **可访问性**：按钮要有 `title` 属性，表单要有 `label`

---

## JavaScript 规范

- **严格模式**：文件开头使用 `'use strict';`
- **无 ES 模块**：所有代码在同一个 `<script>` 区块内
- **事件委托**：使用事件委托减少监听器数量
- **Canvas 渲染**：使用 `requestAnimationFrame` 优化性能
- **避免魔法值**：使用具名常量替代

---

## 变更记录（强制）

每次功能修改必须在文件顶部注释和 `CHANGELOG.md` 中记录：

### index.html 顶部注释格式

```javascript
/**
 * PinDou - 拼豆图案设计工具
 * 版本：v1.0.1
 * 更新日志：
 *   2026-05-02 v1.0.0 - 初始版本，像素画编辑器/图片转图案/模板库
 *   2026-05-03 v1.0.1 - 新增：导出PNG功能；修复：画布缩放偏移问题
 */
```

### CHANGELOG.md 格式

```markdown
# 变更日志

## [v1.0.1] 2026-05-03
- 新增：导出为PNG图片功能
- 修复：画布缩放时位置偏移问题

## [v1.0.0] 2026-05-02
- 初始版本
- 像素画编辑器（支持 10x10 到 200x200 画布）
- 图片转图案（PNG/JPG 支持）
- 模板库（内置 15 个模板）
- 内置 Mard 291 色颜色盘
```

---

## 提交规范（Git）

```
feat: 添加图片转图案功能
fix: 修复画布缩放时位置偏移问题
refactor: 重构颜色匹配算法
docs: 更新变更日志
style: 调整工具栏样式
test: 添加颜色匹配测试
```

格式：`<type>: <描述>`

---

## 调试与日志

- **开发阶段**：`console.log` 用于调试，完成后删除
- **生产代码**：不保留任何调试日志
- **错误处理**：使用 `try-catch` 捕获可预见的错误

---

## 性能要求

- Canvas 操作使用防抖/节流
- 大图处理时显示加载状态
- 避免频繁 DOM 操作，使用文档片段批量更新

---

## 隐私与安全

- 所有数据存储在浏览器 LocalStorage，不上传服务器
- 无外部依赖，杜绝 XSS 风险
- 用户导入的图片仅在本地处理