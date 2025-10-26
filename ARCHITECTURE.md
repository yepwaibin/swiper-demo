# 架构说明

## 系统架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     DocumentViewer 组件                       │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Scroll Info Panel (右上角)                │  │
│  │  - 滚动位置   - 总高度   - 当前页   - 已加载数        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │               Swiper Container (滚动区域)              │  │
│  │                                                         │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  Swiper Wrapper (垂直布局)                      │  │  │
│  │  │                                                   │  │  │
│  │  │  ┌───────────────────────────────────────┐      │  │  │
│  │  │  │  Slide 1 - Page 1 (已加载)           │      │  │  │
│  │  │  │  ┌─────────────────────────────────┐ │      │  │  │
│  │  │  │  │     <img> 图片 1               │ │      │  │  │
│  │  │  │  └─────────────────────────────────┘ │      │  │  │
│  │  │  └───────────────────────────────────────┘      │  │  │
│  │  │                                                   │  │  │
│  │  │  ┌───────────────────────────────────────┐      │  │  │
│  │  │  │  Slide 2 - Page 2 (已加载)           │      │  │  │
│  │  │  │  ┌─────────────────────────────────┐ │      │  │  │
│  │  │  │  │     <img> 图片 2               │ │      │  │  │
│  │  │  │  └─────────────────────────────────┘ │      │  │  │
│  │  │  └───────────────────────────────────────┘      │  │  │
│  │  │                                                   │  │  │
│  │  │  ┌───────────────────────────────────────┐      │  │  │
│  │  │  │  Slide 3 - Page 3 (待加载)           │ ← 懒加载
│  │  │  │  ┌─────────────────────────────────┐ │      │  │  │
│  │  │  │  │     Placeholder               │ │      │  │  │
│  │  │  │  └─────────────────────────────────┘ │      │  │  │
│  │  │  └───────────────────────────────────────┘      │  │  │
│  │  │                                                   │  │  │
│  │  │  ... (更多页面)                                 │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │                                                         │  │
│  │                                          [滚动条] ━━━━▊  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 核心工作流程

### 1. 初始化流程

```
组件挂载 (onMounted)
    ↓
创建 Swiper 实例 (initSwiper)
    ↓
配置垂直滚动 + 自由模式 + 鼠标滚轮
    ↓
监听滚动事件 (on: { scroll, slideChange, ... })
    ↓
加载第一页 (loadedPages.add(1))
    ↓
等待第一张图片加载完成
```

### 2. 首张图片加载流程

```
第一张图片加载完成 (onImageLoad, page=1)
    ↓
获取图片实际高度 (actualHeight)
    ↓
设置预估高度 (estimatedHeight = actualHeight)
    ↓
计算预估总高度 (totalHeight = actualHeight × totalPages)
    ↓
更新 Swiper (swiperInstance.update())
    ↓
触发懒加载检查 (checkAndLoadPages)
```

### 3. 滚动 + 懒加载流程

```
用户滚动页面
    ↓
触发滚动事件 (handleScroll)
    ↓
获取当前滚动位置 (scrollTop = |translate|)
    ↓
更新滚动位置显示 (scrollPosition.value = scrollTop)
    ↓
检查并加载页面 (checkAndLoadPages)
    ↓
计算预加载范围
    startPosition = scrollTop - viewportHeight × 1.5
    endPosition = scrollTop + viewportHeight + viewportHeight × 1.5
    ↓
遍历所有页面，判断是否在预加载范围内
    ↓
如果在范围内且未加载 → loadedPages.add(page)
    ↓
更新当前页码 (updateCurrentPage)
```

## 数据流

```
Mock API (mockImageApi)
    ↓
返回图片 URL
    ↓
    ├─→ getImageUrl(page)
    │       ↓
    │   Template 中的 <img :src="getImageUrl(page)">
    │       ↓
    │   图片开始加载
    │       ↓
    └─→ onImageLoad(event, page)
            ↓
        记录图片高度 (imageHeights[page] = height)
            ↓
        更新总高度 (updateTotalHeight)
            ↓
        如果是第一张 → 计算预估总高度
            ↓
        更新 Swiper
```

## 状态管理

### 响应式状态

| 状态变量 | 类型 | 说明 | 初始值 |
|---------|------|------|--------|
| `totalPages` | `const` | 总页数 | 15 |
| `estimatedHeight` | `ref(Number)` | 预估每页高度 | 0 |
| `scrollPosition` | `ref(Number)` | 当前滚动位置 | 0 |
| `currentPage` | `ref(Number)` | 当前页码 | 1 |
| `loadedPages` | `ref(Set)` | 已加载的页面集合 | `new Set()` |
| `imageHeights` | `ref(Object)` | 每页实际高度 | `{}` |
| `totalHeight` | `ref(Number)` | 文档总高度 | 0 |
| `isFirstImageLoaded` | `ref(Boolean)` | 第一张图是否已加载 | false |

### 非响应式状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `swiperInstance` | `Swiper | null` | Swiper 实例 |
| `swiperContainer` | `ref(HTMLElement)` | Swiper 容器 DOM |
| `swiperWrapper` | `ref(HTMLElement)` | Swiper 包装器 DOM |

## 关键算法

### 1. 懒加载算法

```javascript
function checkAndLoadPages() {
  // 当前滚动位置
  scrollTop = |translate|
  
  // 预加载范围
  preloadStart = scrollTop - 1.5 × viewportHeight
  preloadEnd = scrollTop + 1.5 × viewportHeight + viewportHeight
  
  // 遍历所有页面
  accumulatedHeight = 0
  for each page from 1 to totalPages:
    pageHeight = imageHeights[page] || estimatedHeight || 1000
    pageStart = accumulatedHeight
    pageEnd = accumulatedHeight + pageHeight
    
    // 判断页面是否在预加载范围内
    if (pageEnd >= preloadStart && pageStart <= preloadEnd):
      if page not in loadedPages:
        loadedPages.add(page)  // 标记为待加载
    
    accumulatedHeight += pageHeight
}
```

### 2. 当前页码计算算法

```javascript
function updateCurrentPage(scrollTop) {
  accumulatedHeight = 0
  
  for each page from 1 to totalPages:
    pageHeight = imageHeights[page] || estimatedHeight
    pageMiddle = accumulatedHeight + pageHeight / 2
    
    // 如果滚动位置在页面中点之前
    if scrollTop < pageMiddle:
      currentPage = page
      break
    
    accumulatedHeight += pageHeight
}
```

**逻辑说明：**

- 以每页的中点作为判断标准
- 滚动位置在页面中点之前，说明当前在该页
- 这样当你滚动到两页之间时，显示的是上面的页码

### 3. 总高度计算算法

```javascript
function updateTotalHeight() {
  total = 0
  
  for each page from 1 to totalPages:
    if imageHeights[page] exists:
      // 已加载：使用实际高度
      total += imageHeights[page]
    else:
      // 未加载：使用预估高度
      total += estimatedHeight || 1000
  
  totalHeight = total
}
```

**优化策略：**

1. **初始阶段**：所有页面都用预估高度
   - `totalHeight = estimatedHeight × totalPages`

2. **加载过程中**：混合使用实际高度和预估高度
   - `totalHeight = Σ(已加载的实际高度) + Σ(未加载的预估高度)`

3. **全部加载后**：使用所有实际高度
   - `totalHeight = Σ(所有实际高度)`

## 性能优化策略

### 1. 懒加载

- **原理**：只加载可视区域附近的图片
- **范围**：上下各 1.5 屏
- **效果**：减少初始加载时间和内存占用

```
不加载区域
───────────
   ↑
预加载区域 (1.5 屏)
   ↑
═══════════  ← scrollTop - 1.5 × viewportHeight
可视区域 (1 屏)
═══════════  ← scrollTop + viewportHeight
   ↓
预加载区域 (1.5 屏)
   ↓
───────────
不加载区域
```

### 2. 高度预估

- **原理**：首张图片加载后，用其高度估算总高度
- **公式**：`总高度 = 首张图片高度 × 总页数`
- **效果**：快速渲染滚动条，提供准确的滚动范围

### 3. Set 数据结构

- **用途**：记录已加载的页面
- **优势**：O(1) 查询时间，避免重复加载
- **操作**：
  - `loadedPages.has(page)` - 检查是否已加载
  - `loadedPages.add(page)` - 标记为已加载

### 4. 事件监听优化

监听多个 Swiper 事件，确保不会遗漏任何滚动情况：

```javascript
on: {
  scroll: handleScroll,        // 主要滚动事件
  slideChange: handleScroll,   // slide 切换
  touchMove: handleScroll,     // 触摸滑动
  setTranslate: handleScroll,  // 位置改变
  transitionEnd: handleScroll, // 过渡结束
}
```

## Swiper 配置解析

```javascript
{
  direction: "vertical",        // 垂直滚动
  slidesPerView: "auto",        // 自动计算 slide 数量
  
  // 自由模式 - 核心配置
  freeMode: {
    enabled: true,              // 启用
    sticky: false,              // 不自动对齐（关键！）
    momentum: true,             // 惯性滚动
    momentumRatio: 1,           // 惯性比例
    momentumVelocityRatio: 1,   // 惯性速度
  },
  
  // 鼠标滚轮
  mousewheel: {
    enabled: true,
    sensitivity: 1,             // 灵敏度
    releaseOnEdges: false,
  },
  
  // 键盘支持
  keyboard: {
    enabled: true,
  },
  
  // 观察器 - 自动更新
  observer: true,
  observeParents: true,
  observeSlideChildren: true,
}
```

### 为什么选择这些配置？

1. **`direction: "vertical"`**
   - 文档是纵向阅读的，垂直滚动符合用户习惯

2. **`slidesPerView: "auto"`**
   - 每张图片高度不同，需要自动计算

3. **`freeMode.sticky: false`** ⭐ 关键！
   - 允许停在任意位置（包括两页中间）
   - 如果设为 `true`，会自动对齐到某一页

4. **`mousewheel.sensitivity: 1`**
   - 默认灵敏度，滚动体验自然

5. **`observer: true`**
   - 图片加载后自动更新布局

## 技术栈

```
Vue 3
  ├─ Composition API
  │    ├─ ref / reactive
  │    ├─ onMounted / onBeforeUnmount
  │    └─ nextTick
  │
  └─ Template 语法
       ├─ v-for
       ├─ v-if / v-else
       └─ :style / @load

Swiper
  ├─ 实例化 API
  ├─ 垂直滚动
  ├─ 自由模式
  └─ 事件系统

CSS
  ├─ Flexbox 布局
  ├─ 自定义滚动条
  └─ 响应式设计
```

## 浏览器兼容性

| 特性 | Chrome | Firefox | Safari | Edge |
|------|--------|---------|--------|------|
| Swiper | ✅ | ✅ | ✅ | ✅ |
| 自定义滚动条 | ✅ | ⚠️ (简化) | ✅ | ✅ |
| Flexbox | ✅ | ✅ | ✅ | ✅ |
| Composition API | ✅ | ✅ | ✅ | ✅ |

⚠️ Firefox 的滚动条自定义能力有限，但仍然可用

## 总结

这个架构的核心优势：

1. **自然滚动体验** - Swiper 的 freeMode 配置
2. **性能优化** - 懒加载 + 高度预估
3. **灵活扩展** - 模块化设计，易于修改
4. **用户友好** - 实时信息显示，自定义滚动条

你可以基于这个架构：
- 对接真实后端 API
- 添加更多交互功能（缩放、跳转等）
- 优化加载策略（预加载更多页面）
- 自定义样式主题

