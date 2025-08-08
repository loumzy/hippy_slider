<template>
  <div id="demo-swiper">
    <swiper
      id="swiper"
      :style="{ width: width, height: (150 * width) / 375 }"
      ref="swiper"
      need-animation
      :current="currentSlide"
      @dragging="onDragging"
      @dropped="onDropped"
      @stateChanged="onStateChanged"
    >
      <swiper-slide
        v-for="(item, index) in infiniteSliders"
        :key="index"
        @click="handleClick(item)"
      >
        <img
          id="image"
          :src="item.img"
          :style="{ 
            width: width * 0.94, 
            height: (150 * width * 0.94) / 375,
            marginLeft: width * 0.03,
            marginRight: width * 0.03,
            borderRadius: '8px'
          }"
        />
      </swiper-slide>
    </swiper>
    <div 
      id="swiper-dots"
      :style="{ 
        marginLeft: width * 0.03,
        marginRight: width * 0.03
      }"
    >
      <div
        v-for="(item, index) in sliders"
        :key="index"
        class="dot"
        :class="{ highlight: realSlideIndex === index }"
      ></div>
    </div>
  </div>
</template>

<script>
import {
  defineComponent,
  watch,
  onMounted,
  onUnmounted,
  ref,
  toRefs,
  computed,
  nextTick,
} from "@vue/runtime-core";

export default {
  props: {
    sliders: {
      type: Object,
      required: true,
    },
    width: {
      type: Number,
      required: true,
    },
  },

  setup(props) {
    const { sliders } = toRefs(props);
    const maxIndex = ref(sliders.value.length - 1);
    const currentSlide = ref(0); // 当前显示的slide索引
    const currentSlideNum = ref(0);
    const state = ref("idle");
    const swiper = ref(null); // swiper组件的ref
    let intervalId = null;
    let isJumping = false; // 防止跳转时重复触发

    // 创建无限滚动数据源：使用多个副本减少跳转频率
    const infiniteSliders = computed(() => {
      if (!sliders.value || sliders.value.length === 0) return [];
      
      const original = sliders.value;
      const len = original.length;
      
      if (len === 1) {
        // 只有一张图片时，不需要无限滚动
        return original;
      }
      
      // 创建更大的数据集：使用5组数据进一步减少跳转频率
      // 这样可以大幅减少跳转的频率，让用户更难察觉到跳转
      return [
        ...original, // 第一组
        ...original, // 第二组  
        ...original, // 第三组
        ...original, // 第四组
        ...original  // 第五组
      ];
    });

    // 计算真实的slide索引（用于指示器显示）
    const realSlideIndex = computed(() => {
      if (!sliders.value || sliders.value.length <= 1) {
        return currentSlide.value;
      }
      
      const len = sliders.value.length;
      // 对于三组数据结构，直接取模即可
      return currentSlide.value % len;
    });

    watch(sliders, (newSliders) => {
      if (newSliders && newSliders.length > 0) {
        maxIndex.value = newSliders.length - 1;
        // 数据更新时从中间的一组开始（第三组的第一张），这样向前向后都有足够的空间
        currentSlide.value = newSliders.length > 1 ? newSliders.length * 2 : 0;
        currentSlideNum.value = 0;
        isJumping = false;
        // 重新启动轮播
        nextTick(() => {
          startCycle();
        });
      }
    });

    onMounted(() => {
      startCycle();
    });

    onUnmounted(() => {
      stopCycle();
    });

    const onDropped = (evt) => {
      // 手动拖拽结束时，更新当前页码
      currentSlide.value = evt.currentSlide;
      currentSlideNum.value = realSlideIndex.value;
    };

    const onDragging = (evt) => {};

    // 处理无缝跳转
    const handleInfiniteJump = () => {
      if (isJumping || !sliders.value || sliders.value.length <= 1) return;
      
      const len = sliders.value.length;
      const totalSlides = infiniteSliders.value.length;
      
      // 5组数据结构下，只在接近边界时才进行跳转，减少跳转频率
      // 如果在第一组，跳转到第三组的相同位置
      if (currentSlide.value < len) {
        isJumping = true;
        const targetSlide = currentSlide.value + len * 2; // 跳转到第三组
        nextTick(() => {
          if (swiper.value && typeof swiper.value.setSlideWithoutAnimation === 'function') {
            swiper.value.setSlideWithoutAnimation(targetSlide);
            currentSlide.value = targetSlide;
          }
          setTimeout(() => { 
            isJumping = false; 
          }, 100);
        });
      }
      // 如果在第五组，跳转到第三组的相同位置
      else if (currentSlide.value >= len * 4) {
        isJumping = true;
        const targetSlide = currentSlide.value - len * 2; // 跳转到第三组
        nextTick(() => {
          if (swiper.value && typeof swiper.value.setSlideWithoutAnimation === 'function') {
            swiper.value.setSlideWithoutAnimation(targetSlide);
            currentSlide.value = targetSlide;
          }
          setTimeout(() => { 
            isJumping = false; 
          }, 100);
        });
      }
    };

    const onStateChanged = (evt) => {
      state.value = evt.state;
      
      // 根据状态控制轮播
      if (evt.state === 'dragging') {
        // 拖拽中，暂停轮播
        stopCycle();
      } else if (evt.state === 'idle') {
        // 空闲状态，处理无缝跳转并重新开始轮播
        nextTick(() => {
          if (state.value === 'idle' && !isJumping) {
            handleInfiniteJump();
            // 延长延迟时间，确保跳转完全完成后再启动轮播
            setTimeout(() => {
              if (state.value === 'idle' && !isJumping) {
                startCycle();
              }
            }, 200);
          }
        });
      }
    };

    // 开始循环播放
    const startCycle = () => {
      if (intervalId) {
        clearInterval(intervalId);
      }
      
      // 确保有数据且多于1张图片才开始轮播，并且不在跳转过程中
      if (!sliders.value || sliders.value.length <= 1 || isJumping) {
        return;
      }
      
      intervalId = setInterval(() => {
        if (state.value === 'idle' && !isJumping) {
          // 直接更新currentSlide，通过:current属性触发swiper切换
          currentSlide.value = currentSlide.value + 1;
          // 同步更新指示器显示
          currentSlideNum.value = realSlideIndex.value;
        }
      }, 3000);
    };

    // 停止循环播放
    const stopCycle = () => {
      if (intervalId) {
        clearInterval(intervalId);
        intervalId = null;
      }
    };

    const handleClick = (item) => {
      console.log(item);
    };

    return {
      infiniteSliders,
      realSlideIndex,
      currentSlide,
      currentSlideNum,
      swiper,
      onDropped,
      onDragging,
      onStateChanged,
      startCycle,
      stopCycle,
      handleClick,
    };
  },
};
</script>

<style scoped>
#demo-swiper {
  flex: 1;
}

#demo-swiper #swiper {
  height: 150px;
}

#demo-swiper #image {
  height: 150px;
  resize-mode: cover;
}

#demo-swiper #swiper-dots {
  flex-direction: row;
  align-items: center;
  justify-content: center;
  height: 6px;
  position: relative;
  top: -16px;
}

#demo-swiper .dot {
  width: 6px;
  height: 6px;
  border-radius: 3px;
  background-color: rgb(202, 201, 201);
  margin-left: 5px;
  margin-right: 5px;
}

#demo-swiper .dot.highlight {
  background-color: rgb(60, 117, 215);
}
</style>
