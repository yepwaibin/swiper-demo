# DocumentViewer 使用指南

## 快速开始

已经为你创建了一个完整的 Swiper 文档查看器 demo，满足所有需求。

### 当前实现的功能清单

| 需求 | 实现状态 | 说明 |
|------|---------|------|
| 1. Mock 图片接口 | ✅ | `mockImageApi(page)` 函数模拟后端接口 |
| 2. 自适应展示 | ✅ | 图片自上而下，占满宽度，自适应屏幕 |
| 3. 使用 Swiper | ✅ | 使用 Swiper 作为滚动组件 |
| 4. 实例化方式 | ✅ | 使用 `new Swiper()` 实例化 |
| 5. 显示滚动条 | ✅ | 自定义样式的滚动条，记录滚动高度 |
| 6. 自然滚动 | ✅ | 可以停在任意位置，包括两页中间 |
| 7. 高度计算 | ✅ | 首张图片高度 × 总页数 = 预估总高度 |
| 8. 懒加载 | ✅ | 滚动时自动加载附近的页面 |

## 核心代码解析

### 1. 接口 Mock 实现

```javascript
// 在 DocumentViewer.vue 中
const mockImageApi = async (page) => {
  const width = 800;
  const heights = [1000, 950, 1100, 880, 1050, 920, 1000, 980, 1120, 900, 1040, 960, 1010, 990, 1080];
  const height = heights[page - 1] || 1000;
  
  return `https://picsum.photos/${width}/${height}?random=${page}`;
};
```

**真实接口对接示例：**

```javascript
// 修改为真实的 API 调用
const fetchDocumentPage = async (page) => {
  try {
    const response = await fetch(`https://your-api.com/api/document/page/${page}`, {
      method: 'GET',
      headers: {
        'Authorization': 'Bearer your-token',
        'Content-Type': 'application/json',
      }
    });
    
    if (!response.ok) {
      throw new Error('Failed to fetch page');
    }
    
    const data = await response.json();
    return data.imageUrl; // 假设接口返回 { imageUrl: "..." }
  } catch (error) {
    console.error(`Error loading page ${page}:`, error);
    // 返回占位图
    return 'https://via.placeholder.com/800x1000?text=Load+Failed';
  }
};
```

### 2. Swiper 配置详解

```javascript
swiperInstance = new Swiper(swiperContainer.value, {
  // 垂直方向滚动
  direction: "vertical",
  
  // 自动计算每次显示多少个 slide
  slidesPerView: "auto",
  
  // 自由模式 - 关键配置！实现自然滚动
  freeMode: {
    enabled: true,              // 启用自由模式
    sticky: false,              // 不自动对齐到 slide
    momentum: true,             // 启用惯性滚动
    momentumRatio: 1,           // 惯性滚动的比例
    momentumVelocityRatio: 1,   // 惯性滚动的速度比例
  },
  
  // 鼠标滚轮支持
  mousewheel: {
    enabled: true,              // 启用鼠标滚轮
    sensitivity: 1,             // 滚轮灵敏度（1为正常）
    releaseOnEdges: false,      // 不在边缘释放滚动
  },
  
  // 键盘支持
  keyboard: {
    enabled: true,
  },
  
  // 观察器 - 自动更新
  observer: true,
  observeParents: true,
  observeSlideChildren: true,
  
  // 事件监听
  on: {
    init: function() {
      // Swiper 初始化完成
      console.log("Swiper initialized");
    },
    scroll: handleScroll,        // 滚动事件
    slideChange: handleScroll,   // Slide 变化
    touchMove: handleScroll,     // 触摸移动
    setTranslate: handleScroll,  // 位置改变
    transitionEnd: handleScroll, // 过渡结束
  },
});
```

### 3. 高度计算逻辑

```javascript
// 第一张图片加载完成
const onImageLoad = (event, page) => {
  const img = event.target;
  const actualHeight = img.naturalHeight * (img.offsetWidth / img.naturalWidth);
  
  // 记录实际高度
  imageHeights.value[page] = actualHeight;
  
  // 如果是第一张图片
  if (page === 1 && !isFirstImageLoaded.value) {
    isFirstImageLoaded.value = true;
    estimatedHeight.value = actualHeight;
    
    // 关键！用第一张图片的高度估算总高度
    totalHeight.value = actualHeight * totalPages;
    
    console.log(`预估总高度: ${totalHeight.value}px = ${actualHeight}px × ${totalPages}页`);
  }
  
  // 更新总高度（混合实际高度和预估高度）
  updateTotalHeight();
};

// 更新总高度的函数
const updateTotalHeight = () => {
  let total = 0;
  for (let page = 1; page <= totalPages; page++) {
    if (imageHeights.value[page]) {
      // 已加载：使用实际高度
      total += imageHeights.value[page];
    } else {
      // 未加载：使用预估高度
      total += estimatedHeight.value || 1000;
    }
  }
  totalHeight.value = total;
};
```

### 4. 懒加载实现

```javascript
const checkAndLoadPages = () => {
  if (!swiperInstance) return;
  
  // 获取当前滚动位置
  const scrollTop = Math.abs(swiperInstance.translate || 0);
  const viewportHeight = window.innerHeight;
  
  // 预加载范围：上下各 1.5 屏
  const preloadOffset = viewportHeight * 1.5;
  
  const startLoadPosition = Math.max(0, scrollTop - preloadOffset);
  const endLoadPosition = scrollTop + viewportHeight + preloadOffset;
  
  // 遍历所有页面，判断是否在预加载范围内
  let accumulatedHeight = 0;
  for (let page = 1; page <= totalPages; page++) {
    const pageHeight = imageHeights.value[page] || estimatedHeight.value || 1000;
    const pageStartPosition = accumulatedHeight;
    const pageEndPosition = accumulatedHeight + pageHeight;
    
    // 页面在预加载范围内 -> 加载它
    if (pageEndPosition >= startLoadPosition && pageStartPosition <= endLoadPosition) {
      if (!loadedPages.value.has(page)) {
        loadedPages.value.add(page);  // 标记为已加载
        console.log(`懒加载页面: ${page}`);
      }
    }
    
    accumulatedHeight += pageHeight;
  }
};
```

## 滚动条的实现

滚动条是通过 CSS 实现的，支持 Webkit 和 Firefox：

```css
/* Webkit 浏览器（Chrome, Safari, Edge） */
.swiper-container::-webkit-scrollbar {
  width: 12px;
  background: #f5f5f5;
}

.swiper-container::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 6px;
  margin: 4px 0;
}

.swiper-container::-webkit-scrollbar-thumb {
  background: linear-gradient(180deg, #888 0%, #666 100%);
  border-radius: 6px;
  border: 2px solid #f1f1f1;
}

.swiper-container::-webkit-scrollbar-thumb:hover {
  background: linear-gradient(180deg, #666 0%, #444 100%);
}

/* Firefox 浏览器 */
.swiper-container {
  scrollbar-width: thin;
  scrollbar-color: #888 #f1f1f1;
}
```

## 滚动位置记录

滚动位置通过 Swiper 的 `translate` 属性获取：

```javascript
const handleScroll = () => {
  if (!swiperInstance) return;
  
  // Swiper 的 translate 是负值，取绝对值得到滚动距离
  const scrollTop = Math.abs(swiperInstance.translate || 0);
  scrollPosition.value = scrollTop;
  
  // 触发懒加载检查
  checkAndLoadPages();
};
```

## 当前页码计算

```javascript
const updateCurrentPage = (scrollTop) => {
  let accumulatedHeight = 0;
  
  for (let page = 1; page <= totalPages; page++) {
    const pageHeight = imageHeights.value[page] || estimatedHeight.value || 1000;
    const pageMiddle = accumulatedHeight + pageHeight / 2;
    
    // 如果滚动位置在页面中点之前，当前就是该页
    if (scrollTop < pageMiddle) {
      currentPage.value = page;
      break;
    }
    
    accumulatedHeight += pageHeight;
  }
};
```

## 自定义配置参数

你可以在 `DocumentViewer.vue` 中调整这些参数：

```javascript
// === 文档配置 ===
const totalPages = 15;                     // 总页数

// === 预加载配置 ===
const preloadOffset = viewportHeight * 1.5; // 预加载范围（1.5屏）

// === 滚动配置 ===
mousewheel: {
  sensitivity: 1,  // 调整滚轮灵敏度（默认1，可改为0.5-2）
}

// === 高度配置 ===
const estimatedHeight = ref(0);  // 预估高度（自动从首张图片获取）
```

## 进阶扩展

### 1. 添加加载进度条

```vue
<template>
  <div class="loading-progress" v-if="isLoading">
    <div class="progress-bar" :style="{ width: loadingProgress + '%' }"></div>
  </div>
</template>

<script setup>
const isLoading = ref(false);
const loadingProgress = ref(0);

const onImageLoad = (event, page) => {
  // ... 原有逻辑
  
  // 更新进度
  loadingProgress.value = (loadedPages.value.size / totalPages) * 100;
  
  if (loadedPages.value.size === totalPages) {
    isLoading.value = false;
  }
};
</script>
```

### 2. 添加页码跳转功能

```vue
<template>
  <div class="page-jumper">
    <input 
      v-model.number="jumpToPage" 
      type="number" 
      :min="1" 
      :max="totalPages"
      placeholder="跳转到..."
    />
    <button @click="jumpTo(jumpToPage)">跳转</button>
  </div>
</template>

<script setup>
const jumpToPage = ref(1);

const jumpTo = (page) => {
  if (page < 1 || page > totalPages) return;
  
  // 计算目标位置
  let targetPosition = 0;
  for (let p = 1; p < page; p++) {
    targetPosition += imageHeights.value[p] || estimatedHeight.value;
  }
  
  // 滚动到目标位置
  if (swiperInstance) {
    swiperInstance.setTranslate(-targetPosition);
    swiperInstance.update();
  }
};
</script>
```

### 3. 添加缩放功能

```vue
<script setup>
const scale = ref(1);

const zoomIn = () => {
  scale.value = Math.min(scale.value + 0.1, 3);
};

const zoomOut = () => {
  scale.value = Math.max(scale.value - 0.1, 0.5);
};
</script>

<template>
  <img
    :style="{ transform: `scale(${scale})` }"
    class="page-image"
  />
</template>
```

## 常见问题

### Q1: 滚动不够流畅？

调整 `freeMode` 的参数：

```javascript
freeMode: {
  enabled: true,
  sticky: false,
  momentum: true,
  momentumRatio: 1.5,           // 增加惯性
  momentumVelocityRatio: 1.5,   // 增加速度
}
```

### Q2: 图片加载太慢？

调整预加载范围：

```javascript
const preloadOffset = viewportHeight * 2; // 从 1.5 改为 2
```

### Q3: 想要隐藏滚动条？

```css
.swiper-container::-webkit-scrollbar {
  width: 0;
  display: none;
}
```

### Q4: 如何禁用某些滚动方式？

```javascript
mousewheel: false,  // 禁用鼠标滚轮
keyboard: false,    // 禁用键盘
```

## 总结

这个 demo 完整实现了你要求的所有功能：

1. ✅ Mock 图片接口（可按页数获取）
2. ✅ 图片自适应展示（占满宽度，自上而下）
3. ✅ 使用 Swiper 滚动组件
4. ✅ 实例化方式使用
5. ✅ 显示滚动条并记录高度
6. ✅ 自然滚动（可停在任意位置）
7. ✅ 智能高度计算（首张图片 × 总页数）
8. ✅ 懒加载机制

现在你可以：
- 直接使用这个 demo
- 根据 `USAGE.md` 的说明进行扩展
- 对接真实的后端接口

祝使用愉快！🎉

