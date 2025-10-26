# 新功能说明

## 🎉 v3.0.0 新增功能

本次更新新增了两个重要功能：**全屏模式** 和 **图片旋转**。

---

## 🖥️ 功能 1：全屏模式

### 功能说明

点击全屏按钮后，文档查看器会进入全屏模式，占据整个屏幕，提供更沉浸的阅读体验。

### 使用方法

1. **进入全屏**：点击左上角的全屏按钮（四角扩展图标）
2. **退出全屏**：
   - 点击全屏按钮（四角收缩图标）
   - 或按 `ESC` 键
   - 或按 `F11` 键（部分浏览器）

### 技术实现

```javascript
// 进入全屏
toggleFullscreen() {
  const elem = this.$el;
  
  if (!this.isFullscreen) {
    // 兼容不同浏览器的全屏 API
    if (elem.requestFullscreen) {
      elem.requestFullscreen();
    } else if (elem.webkitRequestFullscreen) {
      elem.webkitRequestFullscreen(); // Safari
    } else if (elem.mozRequestFullScreen) {
      elem.mozRequestFullScreen(); // Firefox
    } else if (elem.msRequestFullscreen) {
      elem.msRequestFullscreen(); // IE/Edge
    }
  } else {
    // 退出全屏
    if (document.exitFullscreen) {
      document.exitFullscreen();
    } else if (document.webkitExitFullscreen) {
      document.webkitExitFullscreen();
    } else if (document.mozCancelFullScreen) {
      document.mozCancelFullScreen();
    } else if (document.msExitFullscreen) {
      document.msExitFullscreen();
    }
  }
}
```

### 状态监听

```javascript
// 监听全屏状态变化
mounted() {
  document.addEventListener("fullscreenchange", this.handleFullscreenChange);
  document.addEventListener("webkitfullscreenchange", this.handleFullscreenChange);
  document.addEventListener("mozfullscreenchange", this.handleFullscreenChange);
  document.addEventListener("MSFullscreenChange", this.handleFullscreenChange);
}

handleFullscreenChange() {
  this.isFullscreen = !!(
    document.fullscreenElement ||
    document.webkitFullscreenElement ||
    document.mozFullScreenElement ||
    document.msFullscreenElement
  );
}
```

### 样式变化

```css
/* 全屏时的样式 */
.document-viewer.is-fullscreen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 9999;
}
```

### 浏览器兼容性

| 浏览器 | 支持版本 | API |
|--------|---------|-----|
| Chrome | 15+ | `requestFullscreen` |
| Firefox | 10+ | `mozRequestFullScreen` |
| Safari | 5.1+ | `webkitRequestFullscreen` |
| Edge | 12+ | `requestFullscreen` / `msRequestFullscreen` |

---

## 🔄 功能 2：图片旋转

### 功能说明

点击旋转按钮，可以让**每张图片**顺时针旋转 90 度。旋转不影响页面的垂直滚动方向，只是图片本身被旋转。

### 使用方法

1. 点击旋转按钮（带箭头的循环图标）
2. 每次点击旋转 90 度
3. 按钮显示当前旋转角度：0° → 90° → 180° → 270° → 0°（循环）

### 旋转角度说明

| 角度 | 效果 | 适用场景 |
|------|------|---------|
| **0°** | 原始方向 | 正常的纵向文档 |
| **90°** | 顺时针 90° | 横向扫描的文档（如横版证书） |
| **180°** | 倒置 | 倒置扫描的文档 |
| **270°** | 顺时针 270°（逆时针 90°） | 特殊方向的文档 |

### 技术实现

```javascript
// 切换旋转
toggleRotation() {
  // 每次点击增加 90 度，循环 0 -> 90 -> 180 -> 270 -> 0
  this.rotationDegree = (this.rotationDegree + 90) % 360;
  
  // 旋转后需要重新计算高度
  this.$nextTick(() => {
    this.recalculateAllHeights();
  });
}
```

### CSS 实现

```css
.page-image {
  transform: rotate(90deg); /* 动态值 */
  transition: transform 0.3s ease;
  transform-origin: center center;
}
```

### 高度自适应

旋转 90° 或 270° 时，图片的高度和宽度会互换，需要重新计算容器高度：

```javascript
getImageWrapperHeight(page) {
  if (this.rotationDegree === 90 || this.rotationDegree === 270) {
    // 旋转 90 或 270 度时，使用原始图片的宽度作为容器高度
    if (this.imageWidths[page]) {
      return this.imageWidths[page] + "px";
    }
    return this.estimatedHeight + "px";
  } else {
    // 0 或 180 度时，使用正常高度
    return this.estimatedHeight + "px";
  }
}
```

### 重要特性

1. **页面保持垂直滚动** ✅
   - 无论图片如何旋转
   - 页面始终是垂直滚动
   - 不会变成横向滚动

2. **每张图片独立旋转** ✅
   - 所有图片同步旋转
   - 保持统一的旋转角度
   - 可以单独处理某张图片（代码可扩展）

3. **高度自动调整** ✅
   - 旋转后容器高度自动调整
   - 滚动条位置自动重新计算
   - 保持滚动流畅性

### 使用场景

**场景 1：横向文档**
```
原始：竖版文档 (800×1000)
旋转 90°：变成横版显示
适合：横向扫描的文档、证书、表格等
```

**场景 2：倒置文档**
```
原始：倒置的文档
旋转 180°：正常显示
适合：扫描时放反的文档
```

**场景 3：特殊方向**
```
原始：特殊角度的文档
旋转 270°：调整到正常方向
适合：特殊方向扫描的文档
```

---

## 🎨 界面设计

### 工具栏位置

```
┌─────────────────────────────────────┐
│  [全屏] [旋转 0°]           [信息面板] │
│                                       │
│                                       │
│           文档内容区域                │
│                                       │
│                                       │
└─────────────────────────────────────┘
```

- **工具栏**：左上角
- **信息面板**：右上角
- **按钮样式**：半透明黑色背景，白色图标

### 按钮设计

```css
.toolbar-btn {
  background: rgba(0, 0, 0, 0.8);  /* 半透明黑色 */
  color: white;
  padding: 12px 16px;
  border-radius: 12px;
  backdrop-filter: blur(10px);     /* 毛玻璃效果 */
  transition: all 0.3s ease;
}

.toolbar-btn:hover {
  background: rgba(0, 0, 0, 0.9);
  transform: translateY(-2px);      /* 悬浮效果 */
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.4);
}
```

### 图标说明

**全屏按钮：**
- 进入全屏：四角向外扩展的图标 ⛶
- 退出全屏：四角向内收缩的图标 ⛶

**旋转按钮：**
- 图标：循环箭头 ↻
- 文字：显示当前角度（0°, 90°, 180°, 270°）

---

## 📱 键盘快捷键

| 快捷键 | 功能 | 说明 |
|--------|------|------|
| `ESC` | 退出全屏 | 浏览器原生支持 |
| `F11` | 切换全屏 | 部分浏览器支持（整个浏览器全屏） |
| ↑ / ↓ | 滚动页面 | Swiper 的键盘支持 |

**未来可扩展：**
- `R` 键：旋转图片
- `1-4` 键：快速设置旋转角度
- `Space` 键：滚动一页

---

## 🔧 自定义配置

### 修改旋转步长

如果想每次旋转 45° 而不是 90°：

```javascript
toggleRotation() {
  // 改为每次旋转 45°
  this.rotationDegree = (this.rotationDegree + 45) % 360;
}
```

### 添加逆时针旋转

```javascript
// 添加一个逆时针旋转按钮
rotateCounterclockwise() {
  this.rotationDegree = (this.rotationDegree - 90 + 360) % 360;
}
```

### 禁用某些功能

```javascript
data() {
  return {
    enableFullscreen: true,  // 设为 false 禁用全屏
    enableRotation: true,    // 设为 false 禁用旋转
  };
}
```

```vue
<template>
  <div class="toolbar">
    <button v-if="enableFullscreen" @click="toggleFullscreen">...</button>
    <button v-if="enableRotation" @click="toggleRotation">...</button>
  </div>
</template>
```

---

## 🎯 使用示例

### 示例 1：查看横向文档

```
1. 打开页面，显示竖版文档
2. 发现文档实际是横向的（字是横着的）
3. 点击旋转按钮
4. 旋转 90°，文档现在正常显示
5. 继续阅读，可以正常滚动
```

### 示例 2：全屏阅读

```
1. 打开页面
2. 点击全屏按钮
3. 进入全屏模式，占据整个屏幕
4. 阅读文档，更沉浸
5. 按 ESC 键退出全屏
```

### 示例 3：组合使用

```
1. 进入全屏模式（更大的显示区域）
2. 旋转图片 90°（调整方向）
3. 开始阅读
4. 滚动查看所有页面
5. 完成后退出全屏
```

---

## 🐛 已知问题

### 1. 旋转后高度计算

**问题：** 旋转 90° 或 270° 时，容器高度可能不完全精确。

**原因：** CSS `transform` 不改变元素的布局空间，只改变视觉效果。

**解决方案：** 
- 已实现：记录图片宽度，旋转时用宽度作为高度
- 可能需要微调 `getImageWrapperHeight` 函数

### 2. 全屏兼容性

**问题：** 部分旧浏览器不支持全屏 API。

**解决方案：**
- 已添加多个浏览器的兼容性代码
- 如果浏览器不支持，按钮可以隐藏或禁用

```javascript
mounted() {
  // 检查浏览器是否支持全屏
  this.fullscreenSupported = !!(
    document.fullscreenEnabled ||
    document.webkitFullscreenEnabled ||
    document.mozFullScreenEnabled ||
    document.msFullscreenEnabled
  );
}
```

---

## 💡 最佳实践

### 1. 性能优化

```javascript
// 使用 CSS transition 而不是 JavaScript 动画
.page-image {
  transition: transform 0.3s ease;
}

// 避免频繁重新计算高度
toggleRotation() {
  this.rotationDegree = (this.rotationDegree + 90) % 360;
  
  // 使用 nextTick 确保 DOM 更新后再计算
  this.$nextTick(() => {
    this.recalculateAllHeights();
  });
}
```

### 2. 用户体验

```javascript
// 添加加载提示
toggleRotation() {
  this.isRotating = true; // 显示加载动画
  
  this.rotationDegree = (this.rotationDegree + 90) % 360;
  
  this.$nextTick(() => {
    this.recalculateAllHeights();
    this.isRotating = false; // 隐藏加载动画
  });
}
```

### 3. 状态持久化

```javascript
// 保存用户的旋转偏好
watch: {
  rotationDegree(newValue) {
    localStorage.setItem('rotation', newValue);
  }
},

mounted() {
  // 恢复上次的旋转角度
  const savedRotation = localStorage.getItem('rotation');
  if (savedRotation) {
    this.rotationDegree = parseInt(savedRotation);
  }
}
```

---

## 🎉 总结

### 新增功能

1. ✅ **全屏模式** - 提供沉浸式阅读体验
2. ✅ **图片旋转** - 支持 0°/90°/180°/270° 旋转
3. ✅ **工具栏** - 美观的操作界面
4. ✅ **状态监听** - 自动更新按钮状态
5. ✅ **高度自适应** - 旋转后自动调整布局

### 技术亮点

- 🎨 **优雅的 UI** - 半透明按钮 + 毛玻璃效果
- ⚡ **流畅动画** - CSS transition 动画
- 🔧 **高度兼容** - 支持主流浏览器
- 📱 **响应式** - 适配不同屏幕尺寸
- 🎯 **用户友好** - 直观的图标和提示

---

**现在你的文档查看器功能更加强大了！** 🚀

试试全屏模式和图片旋转，享受更好的阅读体验吧！

