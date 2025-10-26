<template>
  <div class="document-viewer" :class="{ 'is-fullscreen': isFullscreen }">
    <!-- 滚动信息显示 -->
    <div class="scroll-info">
      <p>当前滚动位置: {{ scrollPosition.toFixed(0) }}px</p>
      <p>总高度: {{ totalHeight.toFixed(0) }}px</p>
      <p>当前页数: {{ currentPage }} / {{ totalPages }}</p>
      <p>已加载页数: {{ loadedPages.size }}</p>
    </div>

    <!-- 工具栏 -->
    <div class="toolbar">
      <button
        @click="toggleFullscreen"
        class="toolbar-btn"
        :title="isFullscreen ? '退出全屏' : '进入全屏'"
      >
        <svg
          v-if="!isFullscreen"
          width="20"
          height="20"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path
            d="M8 3H5a2 2 0 0 0-2 2v3m18 0V5a2 2 0 0 0-2-2h-3m0 18h3a2 2 0 0 0 2-2v-3M3 16v3a2 2 0 0 0 2 2h3"
          />
        </svg>
        <svg
          v-else
          width="20"
          height="20"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path
            d="M8 3v3a2 2 0 0 1-2 2H3m18 0h-3a2 2 0 0 1-2-2V3m0 18v-3a2 2 0 0 1 2-2h3M3 16h3a2 2 0 0 1 2 2v3"
          />
        </svg>
      </button>

      <button
        @click="toggleRotation"
        class="toolbar-btn"
        :title="`旋转 ${rotationDegree}°`"
      >
        <svg
          width="20"
          height="20"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path
            d="M21.5 2v6h-6M2.5 22v-6h6M2 11.5a10 10 0 0 1 18.8-4.3M22 12.5a10 10 0 0 1-18.8 4.2"
          />
        </svg>
        <span class="rotation-text">{{ rotationDegree }}°</span>
      </button>
    </div>

    <!-- Swiper容器 -->
    <div class="swiper-container" ref="swiperContainer">
      <div class="swiper-wrapper" ref="swiperWrapper">
        <div
          v-for="page in totalPages"
          :key="page"
          class="swiper-slide"
          :data-page="page"
        >
          <div
            class="image-wrapper"
            :style="{ minHeight: getImageWrapperHeight(page) }"
          >
            <img
              v-if="shouldLoadPage(page)"
              :src="getImageUrl(page)"
              :alt="`Page ${page}`"
              @load="onImageLoad($event, page)"
              @error="onImageError($event, page)"
              class="page-image"
              :style="{ transform: `rotate(${rotationDegree}deg)` }"
            />
            <div v-else class="image-placeholder">
              <p>页面 {{ page }} - 待加载</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Swiper from "swiper";
import "swiper/css";

export default {
  name: "DocumentViewer",

  data() {
    return {
      // 配置
      totalPages: 15, // 总页数
      estimatedHeight: 0, // 预估每页高度（首次加载后更新）
      scrollPosition: 0,
      currentPage: 1,
      loadedPages: new Set(), // 已加载的页面集合
      imageHeights: {}, // 存储每张图片的实际高度
      imageWidths: {}, // 存储每张图片的实际宽度
      totalHeight: 0, // 总高度
      isFirstImageLoaded: false, // 第一张图片是否已加载

      // 功能状态
      isFullscreen: false, // 是否全屏
      rotationDegree: 0, // 旋转角度 (0, 90, 180, 270)

      // Swiper 实例
      swiperInstance: null,
    };
  },

  mounted() {
    console.log("📱 组件挂载，总页数:", this.totalPages);
    this.$nextTick(() => {
      this.initSwiper();
    });

    // 监听全屏变化事件
    document.addEventListener("fullscreenchange", this.handleFullscreenChange);
    document.addEventListener(
      "webkitfullscreenchange",
      this.handleFullscreenChange
    );
    document.addEventListener(
      "mozfullscreenchange",
      this.handleFullscreenChange
    );
    document.addEventListener(
      "MSFullscreenChange",
      this.handleFullscreenChange
    );
  },

  beforeUnmount() {
    if (this.swiperInstance) {
      // 移除滚动监听
      const container = this.$refs.swiperContainer;
      if (container) {
        container.removeEventListener("scroll", this.handleNativeScroll);
      }

      this.swiperInstance.destroy(true, true);
      this.swiperInstance = null;
      console.log("🔚 Swiper 实例已销毁");
    }

    // 移除全屏事件监听
    document.removeEventListener(
      "fullscreenchange",
      this.handleFullscreenChange
    );
    document.removeEventListener(
      "webkitfullscreenchange",
      this.handleFullscreenChange
    );
    document.removeEventListener(
      "mozfullscreenchange",
      this.handleFullscreenChange
    );
    document.removeEventListener(
      "MSFullscreenChange",
      this.handleFullscreenChange
    );
  },

  methods: {
    // 获取图片URL
    getImageUrl(page) {
      // 使用更稳定的图片源
      const width = 800;
      const heights = [
        1000, 950, 1100, 880, 1050, 920, 1000, 980, 1120, 900, 1040, 960, 1010,
        990, 1080,
      ];
      const height = heights[page - 1] || 1000;

      // 生成本地 SVG 图片（最稳定）
      const colors = [
        "#FF6B6B",
        "#4ECDC4",
        "#45B7D1",
        "#FFA07A",
        "#98D8C8",
        "#6C5CE7",
        "#A29BFE",
        "#FD79A8",
        "#FDCB6E",
        "#74B9FF",
        "#55EFC4",
        "#DFE6E9",
        "#FFB8B8",
        "#81ECEC",
        "#A29BFE",
      ];
      const color = colors[page - 1] || "#4A90E2";

      const svg = `
        <svg xmlns="http://www.w3.org/2000/svg" width="${width}" height="${height}">
          <rect width="100%" height="100%" fill="${color}"/>
          <text x="50%" y="50%" font-family="Arial, sans-serif" font-size="48" fill="white" text-anchor="middle" dominant-baseline="middle">
            Page ${page}
          </text>
          <text x="50%" y="60%" font-family="Arial, sans-serif" font-size="24" fill="rgba(255,255,255,0.7)" text-anchor="middle" dominant-baseline="middle">
            ${width}×${height}
          </text>
        </svg>
      `;

      // 转换为 Data URL
      return `data:image/svg+xml;charset=UTF-8,${encodeURIComponent(svg)}`;
    },

    // 判断是否应该加载某一页
    shouldLoadPage(page) {
      return this.loadedPages.has(page);
    },

    // 图片加载完成
    onImageLoad(event, page) {
      const img = event.target;
      const actualHeight = img.offsetHeight;
      const actualWidth = img.offsetWidth;

      console.log(
        `页面 ${page} 图片加载完成，实际显示高度: ${actualHeight.toFixed(
          0
        )}px, 宽度: ${actualWidth.toFixed(0)}px`
      );

      // 记录实际高度和宽度
      this.imageHeights[page] = actualHeight;
      this.imageWidths[page] = actualWidth;

      // 如果是第一张图片，用它的高度来估算总高度
      if (page === 1 && !this.isFirstImageLoaded) {
        this.isFirstImageLoaded = true;
        this.estimatedHeight = actualHeight;

        // 计算预估总高度
        this.totalHeight = actualHeight * this.totalPages;

        console.log(
          `✅ 第一张图片加载完成，高度: ${actualHeight.toFixed(0)}px`
        );
        console.log(
          `📏 预估总高度: ${this.totalHeight.toFixed(
            0
          )}px (${actualHeight.toFixed(0)}px × ${this.totalPages}页)`
        );

        // 更新 Swiper
        this.$nextTick(() => {
          if (this.swiperInstance) {
            this.swiperInstance.update();
          }
        });
      }

      // 重新计算总高度
      this.updateTotalHeight();
    },

    // 图片加载失败
    onImageError(event, page) {
      console.error(`页面 ${page} 图片加载失败`);
    },

    // 更新总高度
    updateTotalHeight() {
      let total = 0;
      for (let page = 1; page <= this.totalPages; page++) {
        if (this.imageHeights[page]) {
          total += this.imageHeights[page];
        } else {
          total += this.estimatedHeight || 800;
        }
      }
      this.totalHeight = total;
    },

    // 原生滚动事件处理（关键修复！）
    handleNativeScroll(event) {
      const container = event.target;
      const scrollTop = container.scrollTop;

      // 更新滚动位置
      this.scrollPosition = scrollTop;

      // 检查并加载页面
      this.checkAndLoadPages(scrollTop);

      // 更新当前页码
      this.updateCurrentPage(scrollTop);
    },

    // 检查哪些页面需要加载（懒加载逻辑）
    checkAndLoadPages(scrollTop) {
      if (!this.$refs.swiperContainer) return;

      const viewportHeight = window.innerHeight;

      // 预加载范围：当前可视区域上下各 1.5 屏
      const preloadOffset = viewportHeight * 1.5;

      const startLoadPosition = Math.max(0, scrollTop - preloadOffset);
      const endLoadPosition = scrollTop + viewportHeight + preloadOffset;

      // 计算需要加载的页面
      let accumulatedHeight = 0;
      for (let page = 1; page <= this.totalPages; page++) {
        const pageHeight =
          this.imageHeights[page] || this.estimatedHeight || 800;
        const pageStartPosition = accumulatedHeight;
        const pageEndPosition = accumulatedHeight + pageHeight;

        // 如果页面在预加载范围内，加载它
        if (
          pageEndPosition >= startLoadPosition &&
          pageStartPosition <= endLoadPosition
        ) {
          if (!this.loadedPages.has(page)) {
            this.loadedPages.add(page);
            console.log(
              `🔄 懒加载页面: ${page} (位置: ${pageStartPosition.toFixed(
                0
              )}px - ${pageEndPosition.toFixed(0)}px)`
            );
          }
        }

        accumulatedHeight += pageHeight;
      }
    },

    // 更新当前页码
    updateCurrentPage(scrollTop) {
      let accumulatedHeight = 0;
      for (let page = 1; page <= this.totalPages; page++) {
        const pageHeight =
          this.imageHeights[page] || this.estimatedHeight || 800;
        const pageMiddle = accumulatedHeight + pageHeight / 2;

        // 如果滚动位置在页面中点之前，说明当前在该页
        if (scrollTop < pageMiddle) {
          this.currentPage = page;
          break;
        }

        accumulatedHeight += pageHeight;
      }

      // 确保不超过总页数
      if (this.currentPage > this.totalPages) {
        this.currentPage = this.totalPages;
      }
    },

    // 初始化 Swiper
    initSwiper() {
      if (!this.$refs.swiperContainer) return;

      // 创建 Swiper 实例
      this.swiperInstance = new Swiper(this.$refs.swiperContainer, {
        direction: "vertical", // 垂直方向
        slidesPerView: "auto", // 自动计算显示的slide数量
        freeMode: {
          enabled: true, // 启用自由模式，允许自然滚动
          sticky: false, // 不自动对齐到slide
          momentum: true, // 启用惯性滚动
          momentumRatio: 1, // 惯性滚动比例
          momentumVelocityRatio: 1, // 惯性滚动速度比例
        },
        mousewheel: {
          enabled: true, // 启用鼠标滚轮
          sensitivity: 1, // 滚轮灵敏度
          releaseOnEdges: false,
        },
        keyboard: {
          enabled: true,
        },
        observer: true,
        observeParents: true,
        observeSlideChildren: true,
      });

      console.log("🚀 Swiper 实例已创建");

      // 关键修复：监听原生滚动事件而不是 Swiper 的 translate
      const container = this.$refs.swiperContainer;
      if (container) {
        container.addEventListener("scroll", this.handleNativeScroll);
        console.log("✅ 原生滚动监听已添加");
      }

      // 初始加载第一页和周围的页面
      this.loadedPages.add(1);
      this.$nextTick(() => {
        this.checkAndLoadPages(0);
      });
    },

    // 切换全屏
    toggleFullscreen() {
      const elem = this.$el;

      if (!this.isFullscreen) {
        // 进入全屏
        if (elem.requestFullscreen) {
          elem.requestFullscreen();
        } else if (elem.webkitRequestFullscreen) {
          elem.webkitRequestFullscreen();
        } else if (elem.mozRequestFullScreen) {
          elem.mozRequestFullScreen();
        } else if (elem.msRequestFullscreen) {
          elem.msRequestFullscreen();
        }
        console.log("🖥️ 进入全屏模式");
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
        console.log("📱 退出全屏模式");
      }
    },

    // 处理全屏变化
    handleFullscreenChange() {
      this.isFullscreen = !!(
        document.fullscreenElement ||
        document.webkitFullscreenElement ||
        document.mozFullScreenElement ||
        document.msFullscreenElement
      );
      console.log(`全屏状态: ${this.isFullscreen ? "是" : "否"}`);
    },

    // 切换旋转
    toggleRotation() {
      // 每次点击增加 90 度，循环 0 -> 90 -> 180 -> 270 -> 0
      this.rotationDegree = (this.rotationDegree + 90) % 360;
      console.log(`🔄 旋转到: ${this.rotationDegree}°`);

      // 旋转后需要重新计算高度
      this.$nextTick(() => {
        this.recalculateAllHeights();
      });
    },

    // 获取图片包装器的高度（考虑旋转）
    getImageWrapperHeight(page) {
      if (this.rotationDegree === 90 || this.rotationDegree === 270) {
        // 旋转 90 或 270 度时，高度和宽度互换
        // 使用原始图片的宽度作为容器的高度
        if (this.imageWidths[page]) {
          return this.imageWidths[page] + "px";
        }
        return this.estimatedHeight + "px";
      } else {
        // 0 或 180 度时，使用正常高度
        return this.estimatedHeight + "px";
      }
    },

    // 重新计算所有高度（旋转后）
    recalculateAllHeights() {
      // 触发重新计算
      this.updateTotalHeight();

      // 更新 Swiper
      if (this.swiperInstance) {
        this.swiperInstance.update();
      }
    },
  },
};
</script>

<style scoped>
.document-viewer {
  width: 100%;
  height: 100vh;
  position: relative;
  overflow: hidden;
  background: #f0f0f0;
}

/* 全屏样式 */
.document-viewer.is-fullscreen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 9999;
}

.scroll-info {
  position: fixed;
  top: 20px;
  right: 20px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 15px 20px;
  border-radius: 12px;
  z-index: 1000;
  font-size: 14px;
  font-family: "Courier New", monospace;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(10px);
}

.scroll-info p {
  margin: 6px 0;
  line-height: 1.5;
}

/* 工具栏 */
.toolbar {
  position: fixed;
  top: 20px;
  left: 20px;
  display: flex;
  gap: 10px;
  z-index: 1000;
}

.toolbar-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  border: none;
  padding: 12px 16px;
  border-radius: 12px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.3s ease;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(10px);
}

.toolbar-btn:hover {
  background: rgba(0, 0, 0, 0.9);
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.4);
}

.toolbar-btn:active {
  transform: translateY(0);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

.toolbar-btn svg {
  flex-shrink: 0;
}

.rotation-text {
  font-family: "Courier New", monospace;
  font-weight: bold;
  min-width: 35px;
  text-align: center;
}

.swiper-container {
  width: 100%;
  height: 100vh;
  overflow: auto;
}

.swiper-wrapper {
  display: flex;
  flex-direction: column;
}

.swiper-slide {
  width: 100% !important;
  height: auto !important;
  flex-shrink: 0;
}

.image-wrapper {
  width: 100%;
  position: relative;
  background: #ffffff;
  display: flex;
  align-items: center;
  justify-content: center;
  border-bottom: 2px solid #e0e0e0;
}

.page-image {
  width: 100%;
  height: auto;
  display: block;
  object-fit: contain;
  background: white;
  transition: transform 0.3s ease;
  transform-origin: center center;
}

.image-placeholder {
  width: 100%;
  height: 100%;
  min-height: 600px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  color: #adb5bd;
  font-size: 20px;
  font-weight: 500;
}

/* 美化滚动条 */
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
  transition: background 0.3s ease;
}

.swiper-container::-webkit-scrollbar-thumb:hover {
  background: linear-gradient(180deg, #666 0%, #444 100%);
}

.swiper-container::-webkit-scrollbar-thumb:active {
  background: linear-gradient(180deg, #555 0%, #333 100%);
}

/* Firefox 滚动条样式 */
.swiper-container {
  scrollbar-width: thin;
  scrollbar-color: #888 #f1f1f1;
}
</style>

