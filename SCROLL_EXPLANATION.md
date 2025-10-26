# 原生滚动 vs Swiper Translate 详解

## 📌 核心问题

为什么我们使用**原生滚动监听**（`container.addEventListener('scroll')`）而不是 **Swiper 的 translate**？

---

## 🎯 简单回答

**因为在 Swiper 的 `freeMode` 下，Swiper 使用的是原生滚动，而不是 CSS transform translate。**

---

## 📖 详细解释

### 1. Swiper 的两种工作模式

#### 模式 A：默认模式（Slide Mode）
- **工作原理**：使用 CSS `transform: translateY()` 或 `translateX()` 来移动容器
- **滚动方式**：通过改变 transform 的值来"模拟"滚动
- **特点**：
  - ✅ 每次滑动会自动对齐到某一个 slide
  - ✅ 有吸附效果
  - ❌ 不能停在两个 slide 之间
  - ✅ 可以通过 `swiper.translate` 获取当前位置

**示例代码：**
```javascript
// 默认模式（不使用 freeMode）
new Swiper('.swiper', {
  direction: 'vertical',
  slidesPerView: 1,
  // freeMode 未启用
});

// 滚动监听
swiperInstance.on('setTranslate', function() {
  console.log(this.translate); // ✅ 可以获取位置
});
```

#### 模式 B：自由模式（Free Mode）⭐ 我们使用的模式
- **工作原理**：使用**原生的 CSS `overflow: auto`** 和 **`scrollTop`** 属性
- **滚动方式**：真正的浏览器原生滚动
- **特点**：
  - ✅ 可以停在任意位置
  - ✅ 没有吸附效果
  - ✅ 可以停在两个 slide 之间
  - ❌ `swiper.translate` 可能不准确或为 0

**示例代码：**
```javascript
// 自由模式（我们的配置）
new Swiper('.swiper', {
  direction: 'vertical',
  slidesPerView: 'auto',
  freeMode: {
    enabled: true,  // ⭐ 启用自由模式
    sticky: false,  // ⭐ 不自动对齐
  }
});

// Swiper 的 translate 不工作
swiperInstance.on('setTranslate', function() {
  console.log(this.translate); // ❌ 可能为 0 或不准确
});

// 使用原生滚动监听 ✅
container.addEventListener('scroll', function() {
  console.log(this.scrollTop); // ✅ 准确获取滚动位置
});
```

---

## 🔍 实际测试对比

### 测试 1：使用 Swiper translate（错误方法）

```javascript
// ❌ 错误的方式
swiperInstance.on('scroll', function() {
  const scrollTop = Math.abs(this.translate);
  console.log('滚动位置:', scrollTop);
});

// 结果：
// - scrollTop 始终为 0
// - 或者数值不准确
// - 页面数据无法更新
```

### 测试 2：使用原生滚动（正确方法）⭐

```javascript
// ✅ 正确的方式
const container = document.querySelector('.swiper-container');
container.addEventListener('scroll', function() {
  const scrollTop = this.scrollTop;
  console.log('滚动位置:', scrollTop);
});

// 结果：
// - scrollTop 实时更新
// - 数值准确
// - 页面数据正常更新
```

---

## 📊 对比表格

| 特性 | Swiper Translate | 原生滚动 (scrollTop) |
|------|-----------------|---------------------|
| **适用模式** | 默认模式（Slide Mode） | 自由模式（Free Mode）⭐ |
| **工作原理** | CSS transform | 原生 overflow: auto |
| **滚动位置获取** | `swiper.translate` | `container.scrollTop` ✅ |
| **在 freeMode 下是否准确** | ❌ 不准确 | ✅ 准确 |
| **能否停在中间** | ❌ 不能 | ✅ 可以 |
| **性能** | 较好（GPU 加速） | 很好（原生滚动） |
| **兼容性** | 需要 Swiper 库 | 所有浏览器原生支持 |

---

## 💡 为什么在 freeMode 下 translate 不工作？

### 原因分析

1. **Swiper 的设计**
   - `freeMode` 是为了提供类似原生滚动的体验
   - Swiper 内部会让浏览器接管滚动
   - 此时 Swiper 不再使用 `transform` 来移动容器

2. **技术实现**
   - 在 `freeMode` 下，Swiper 会给容器添加 `overflow: auto`
   - 浏览器使用原生的滚动机制
   - `translate` 属性不再被更新

3. **源码证据**
   ```javascript
   // Swiper 源码（简化）
   if (params.freeMode && params.freeMode.enabled) {
     // 使用原生滚动
     swiper.$wrapperEl.css({ overflow: 'auto' });
     // translate 不再更新
   } else {
     // 使用 transform translate
     swiper.$wrapperEl.css({ transform: `translate3d(0, ${translate}px, 0)` });
   }
   ```

---

## 🎓 最佳实践

### 选择合适的监听方式

#### 场景 1：使用默认模式（每次滑动对齐到 slide）
```javascript
const swiper = new Swiper('.swiper', {
  direction: 'vertical',
  slidesPerView: 1,
  // freeMode: false （默认）
});

// ✅ 使用 Swiper 的事件
swiper.on('slideChange', function() {
  console.log('当前 slide:', this.activeIndex);
});

swiper.on('setTranslate', function() {
  console.log('当前位置:', Math.abs(this.translate));
});
```

#### 场景 2：使用自由模式（可以停在任意位置）⭐ 我们的场景
```javascript
const swiper = new Swiper('.swiper', {
  direction: 'vertical',
  slidesPerView: 'auto',
  freeMode: {
    enabled: true,
    sticky: false,
  }
});

// ✅ 使用原生滚动事件
const container = document.querySelector('.swiper-container');
container.addEventListener('scroll', function() {
  const scrollTop = this.scrollTop;
  console.log('滚动位置:', scrollTop);
  
  // 更新你的状态
  updateScrollPosition(scrollTop);
  checkLazyLoad(scrollTop);
  updateCurrentPage(scrollTop);
});
```

---

## 🔧 完整实现（我们的代码）

```javascript
export default {
  methods: {
    initSwiper() {
      // 创建 Swiper 实例
      this.swiperInstance = new Swiper(this.$refs.swiperContainer, {
        direction: "vertical",
        slidesPerView: "auto",
        freeMode: {
          enabled: true,   // ⭐ 启用自由模式
          sticky: false,   // ⭐ 不自动对齐
          momentum: true,  // 启用惯性滚动
        },
        mousewheel: { enabled: true },
      });

      // ❌ 不使用 Swiper 的 translate
      // swiper.on('setTranslate', ...) 

      // ✅ 使用原生滚动监听
      const container = this.$refs.swiperContainer;
      container.addEventListener('scroll', this.handleNativeScroll);
    },

    handleNativeScroll(event) {
      // ✅ 获取准确的滚动位置
      const scrollTop = event.target.scrollTop;
      
      // 更新状态
      this.scrollPosition = scrollTop;
      this.checkAndLoadPages(scrollTop);
      this.updateCurrentPage(scrollTop);
    },
  }
};
```

---

## 🤔 常见问题 FAQ

### Q1: 为什么不能两者都用？

**A:** 可以，但没必要。在 `freeMode` 下：
- `swiper.translate` 不准确，用了也是错的
- `scrollTop` 完全准确，满足所有需求
- 两者同时用会增加代码复杂度

### Q2: 如果不用 freeMode，可以用 translate 吗？

**A:** 可以！如果你不用 `freeMode`，那么：
- ✅ `swiper.translate` 是准确的
- ✅ 可以使用 `swiper.on('setTranslate', ...)`
- ❌ 但是不能停在两页中间（不符合你的需求）

### Q3: 原生滚动性能会差吗？

**A:** 不会！反而可能更好：
- ✅ 浏览器原生滚动经过高度优化
- ✅ 不需要 JavaScript 计算
- ✅ GPU 加速（现代浏览器）
- ✅ 更流畅的用户体验

### Q4: 如果我想既能对齐又能自由滚动怎么办？

**A:** 这是矛盾的需求。你可以：
- **方案 1**：使用 `freeMode.sticky: true`（会自动对齐到最近的 slide）
- **方案 2**：不用 Swiper，直接用原生滚动 + Intersection Observer

### Q5: Swiper 在 freeMode 下还有什么用？

**A:** 很多！Swiper 提供：
- ✅ 鼠标滚轮支持（mousewheel）
- ✅ 键盘控制（keyboard）
- ✅ 触摸滑动（touch）
- ✅ 惯性滚动（momentum）
- ✅ 统一的 API 和配置
- ✅ 跨浏览器兼容性

只是在 `freeMode` 下，滚动位置要用 `scrollTop` 而不是 `translate`。

---

## 📝 总结

### 核心要点

1. **Swiper 有两种工作模式**
   - 默认模式：使用 `transform translate`
   - 自由模式：使用原生 `scrollTop`

2. **我们使用自由模式**
   - 因为需求是"可以停在两页中间"
   - 必须使用 `freeMode`

3. **在自由模式下**
   - ❌ `swiper.translate` 不准确
   - ✅ `container.scrollTop` 准确

4. **正确的做法**
   ```javascript
   // ❌ 错误
   swiper.on('setTranslate', ...)
   
   // ✅ 正确
   container.addEventListener('scroll', ...)
   ```

### 记忆口诀

```
自由模式用 scrollTop，
默认模式用 translate。
需求决定工作模式，
模式决定监听方式。
```

---

## 🎯 实践建议

1. **根据需求选择模式**
   - 需要对齐 → 默认模式 → 用 `translate`
   - 需要自由滚动 → 自由模式 → 用 `scrollTop`

2. **不要混用**
   - 在自由模式下不要用 `translate`
   - 在默认模式下不要用 `scrollTop`

3. **测试验证**
   - 打印 `swiper.translate` 和 `scrollTop`
   - 看哪个值在实时更新
   - 用实时更新的那个

4. **阅读文档**
   - Swiper 官方文档有详细说明
   - 特别关注 `freeMode` 部分

---

## 🔗 参考资源

- [Swiper 官方文档 - Free Mode](https://swiperjs.com/swiper-api#param-freeMode)
- [MDN - Element.scrollTop](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTop)
- [CSS Transform](https://developer.mozilla.org/en-US/docs/Web/CSS/transform)

---

**希望这个解释能帮助你理解为什么我们使用原生滚动监听！** 🎉

