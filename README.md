# Swiper 文档查看器 Demo

这是一个使用 Swiper 实现的文档图片查看器示例，模拟了将文档转换为图片并进行懒加载展示的场景。

## 功能特性

✅ **完整实现的功能：**

1. **Mock 图片接口** - 使用 `mockImageApi` 函数模拟后端接口，根据页数返回不同的图片
2. **自适应展示** - 图片自上而下展示，占满宽度，自适应屏幕尺寸
3. **Swiper 实例化** - 使用 `new Swiper()` 方式实例化组件
4. **自然滚动** - 支持自然滚动方式，可以停在任意位置（包括两页之间）
5. **滚动条显示** - 页面有可见的自定义滚动条，实时记录滚动高度
6. **智能高度计算** - 首次加载第一张图片后，使用其高度乘以总页数计算预估总高度
7. **懒加载机制** - 滚动时自动加载可视区域附近的图片，优化性能
8. **实时信息显示** - 右上角显示当前滚动位置、总高度、当前页数、已加载页数

## 技术实现

### 1. Mock 图片接口

```javascript
const mockImageApi = async (page) => {
  // 模拟不同高度的页面（800-1200px之间）
  const heights = [1000, 950, 1100, 880, 1050, ...];
  const height = heights[page - 1] || 1000;
  
  return `https://picsum.photos/800/${height}?random=${page}`;
};
```

### 2. Swiper 实例化配置

```javascript
swiperInstance = new Swiper(swiperContainer.value, {
  direction: "vertical",        // 垂直滚动
  slidesPerView: "auto",        // 自动计算显示数量
  freeMode: {
    enabled: true,              // 自由模式，自然滚动
    sticky: false,              // 不自动对齐
    momentum: true,             // 惯性滚动
  },
  mousewheel: {
    enabled: true,              // 支持鼠标滚轮
    sensitivity: 1,
  },
  // ... 更多配置
});
```

### 3. 高度计算逻辑

当第一张图片加载完成后：

```javascript
if (page === 1 && !isFirstImageLoaded.value) {
  estimatedHeight.value = actualHeight;
  // 预估总高度 = 单页高度 × 总页数
  totalHeight.value = actualHeight * totalPages;
}
```

### 4. 懒加载策略

- 首次进入页面，加载第一页
- 滚动时，预加载当前可视区域上下各 1.5 屏的内容
- 使用 `Set` 数据结构记录已加载的页面，避免重复加载

```javascript
const preloadOffset = viewportHeight * 1.5;
const startLoadPosition = Math.max(0, scrollTop - preloadOffset);
const endLoadPosition = scrollTop + viewportHeight + preloadOffset;
```

## 项目结构

```
swiper-demo/
├── src/
│   ├── components/
│   │   └── DocumentViewer.vue    # 主要组件
│   ├── App.vue                   # 根组件
│   └── main.js                   # 入口文件
├── package.json
└── README.md
```

## 安装与运行

### 1. 安装依赖

```bash
npm install
```

### 2. 启动开发服务器

```bash
npm run dev
```

### 3. 访问应用

打开浏览器访问：`http://localhost:5173`

## 使用说明

### 基本操作

- **鼠标滚轮** - 上下滚动查看文档
- **拖动滚动条** - 快速跳转到指定位置
- **键盘方向键** - 也可以进行滚动操作

### 界面信息

右上角的信息面板实时显示：

- **当前滚动位置** - 当前滚动的像素位置
- **总高度** - 文档的总高度（预估值）
- **当前页数** - 当前正在查看的页数
- **已加载页数** - 已经加载的图片数量

### 自定义配置

在 `DocumentViewer.vue` 中可以调整的参数：

```javascript
const totalPages = 15;                    // 总页数
const preloadOffset = viewportHeight * 1.5; // 预加载范围
```

## 核心特性说明

### 1. 自然滚动体验

使用 Swiper 的 `freeMode` 配置，实现了：
- 不会自动对齐到某一页
- 可以停在任意位置（包括两页中间）
- 支持惯性滚动
- 滚动体验流畅自然

### 2. 性能优化

- **懒加载** - 只加载可视区域附近的图片
- **高度预估** - 使用第一页的高度预估总高度，快速渲染滚动条
- **事件防抖** - 滚动事件中的加载检查有优化机制

### 3. 自适应布局

- 图片宽度自动占满容器
- 高度按比例自适应
- 支持不同尺寸的屏幕
- 响应式设计

## 技术栈

- **Vue 3** - 前端框架
- **Swiper** - 滑动/滚动组件库
- **Vite** - 构建工具

## 注意事项

1. **图片加载** - 使用的是 `picsum.photos` 图片服务，需要网络连接
2. **首屏加载** - 第一次加载时会请求第一张图片，加载完成后才能正确计算高度
3. **懒加载时机** - 图片会在滚动到附近时才开始加载，这是正常的性能优化行为

## 扩展建议

如果要接入真实的后端接口，可以这样修改：

```javascript
// 将 mockImageApi 替换为真实的 API 调用
const mockImageApi = async (page) => {
  const response = await fetch(`/api/document/page/${page}`);
  const data = await response.json();
  return data.imageUrl;
};
```

## 浏览器兼容性

- Chrome / Edge (推荐)
- Firefox
- Safari
- 支持现代浏览器的滚动条自定义样式

## License

MIT
