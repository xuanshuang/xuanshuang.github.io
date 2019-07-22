---
title: 使用Vue实现高仿低配Swiper
date: 2019-7-07 13:14:15
tags: 
- Vue
- Swiper
---
最近要用Vue来进行开发，想到前一段时间投入了支付宝小程序的开发，使用到了久违的双向绑定语法，这次又开始使用Vue，居然还有点小亲切。
从写点什么下手呢？想着想着，移动端H5怎么也得有个轮播图吧，于是立马就打开了新浪首页，盯着它的轮播，抄袭了起来。
<!--more-->

### 分析
新浪首页的轮播用了大名鼎鼎的Swiper组件，支持左右滑、无限轮播等特性。对着devTool观察了一整个轮播后，对它的实现方式有一定的了解了。
* 通过CSS translate属性，把每一帧按顺序排在同一行中，设置定时器改变当前显示的帧
* 处理用户滑动的事件
* 在首帧前增加一帧尾帧的备份，在尾帧后加一帧首帧，以达到无限轮播的效果

### 实现
``` html
<template>
  <div class="swiper" v-if="data.length">
    <div :class="swiperContainerCls" :style="swiperContainerSty" ref="swiperDiv">
      <div
        class="swiper-item"
        v-for="(item, i) in data"
        :key="i"
        :style="{ transform: `translate3D(${i * 100}%, 0, 0)`, backgroundImage: `url(${item.serviceImg})` }"
        @click="handleBannerClick(item.url)"
      />
      <template v-if="data.length > 1">
        <div class="swiper-item" :style="{ transform: `translate3D(${data.length * 100}%, 0, 0)`, backgroundImage: `url(${data[0].serviceImg})` }" />
        <div class="swiper-item" :style="{ transform: `translate3D(-100%, 0, 0)`, backgroundImage: `url(${data[data.length - 1].serviceImg})` }" />
      </template>
    </div>
    <div v-if="showIndicator && data.length > 1" class="swiper-indicator">
      <div v-for="(item, index) in data" :class="`swiper-dot ${isActived(index)}`" :key="`${index}-dot`" />
    </div>
  </div>
</template>

<script>
import { addEvent, removeEvent, getTouch, getTouchIdentifier } from '../utils';

const MINIMUM_SLIDE_INTERVAL = 300;

export default {
  name: 'swiper',
  props: {
    data: {
      type: Array,
      default() {
        return [];
      },
    },
    showIndicator: {
      type: Boolean,
      default: true
    }
  },
  data() {
    return {
      touchStartPointX: 0,
      touchStartTime: 0,
      deltaX: 0,
      touchIdentifier: -1,
      currentTranslateX: 0,
      isInTouchMoveAction: false,
      intervalId: undefined,
      isClickEvent: true,
      isDuplicateMove: false,
    };
  },
  watch: {},
  computed: {
    swiperContainerSty() {
      const { isInTouchMoveAction, currentTranslateX, deltaX } = this;
      const translateX = isInTouchMoveAction ? currentTranslateX + deltaX : currentTranslateX;
      return ({
        transform: `translateX(${translateX}px)`
      });
    },
    swiperContainerCls() {
      const { isInTouchMoveAction } = this;
      return `swiper-container ${isInTouchMoveAction ? '' : 'swiper-container-transition' }`;
    }
  },
  methods: {
    handleBannerClick(url) {
      ...
    },
    onTransitionend(e) {
      removeEvent(this.$refs.swiperDiv, 'transitionend', this.onTransitionend);
      const { perWidth, data, isDuplicateMove, currentTranslateX, isNeedOnceMore } = this;
      if (isDuplicateMove) {
        // 在重复的swiper滚动动画完成后，立刻改变到其真实的位置，且这时候不要运用动画
        this.isInTouchMoveAction = true;
        const minTranslateX = this.getMinTranslateX();
        if (currentTranslateX > 0) {
          this.currentTranslateX = minTranslateX;
        } else if (currentTranslateX < minTranslateX) {
          this.currentTranslateX = 0;
        }
      }
      this.isDuplicateMove = false;
      this.setSwiperInterval();
    },
    isActived(index) {
      const { currentTranslateX, perWidth, data } = this;
      const activeItem = !perWidth ? 0 : currentTranslateX / perWidth;
      // 当前位置是最后一张swiper的镜像
      if (activeItem === 1) {
        return index === data.length - 1 ? 'swiper-dot-active' : '';
      } else if (Math.abs(activeItem) === data.length) {
        // 当前位置是第一张swiper的镜像
        return index === 0 ? 'swiper-dot-active' : '';
      }
      return Math.abs(activeItem) === index ? 'swiper-dot-active' : '';
    },
    onTouchStart(e) {
      // e.preventDefault();
      const { currentTranslateX, perWidth, data } = this;
      if (currentTranslateX === perWidth || currentTranslateX + perWidth * data.length === 0) {
        return;
      }
      if (this.intervalId) {
        clearInterval(this.intervalId);
      }
      const touchIdentifier = getTouchIdentifier(e);
      const { clientX } = getTouch(e, touchIdentifier);
      this.touchIdentifier = touchIdentifier;
      this.touchStartPointX = clientX;
      this.touchStartTime = new Date().getTime();
      this.isInTouchMoveAction = true;
      this.intervalId = undefined;
    },
    onTouchMove(e) {
      e.preventDefault();
      const { currentTranslateX, perWidth, isInTouchMoveAction } = this;
      if (!isInTouchMoveAction) {
        return;
      }
      const { clientX } = getTouch(e, this.touchIdentifier);
      const deltaX = clientX - this.touchStartPointX;
      if (this.touchStartPointX + deltaX > perWidth) {
        return;
      }
      this.deltaX = deltaX;
      this.isClickEvent = false;
    },
    onTouchEnd(e) {
      const {
        touchIdentifier,
        touchStartPointX,
        currentTranslateX,
        perWidth,
        touchStartTime,
        isClickEvent,
        isInTouchMoveAction
      } = this;
      if (!isInTouchMoveAction) {
        return;
      }
      if (isClickEvent) {
        this.resetState();
        this.setSwiperInterval();
        return;
      }
      e.preventDefault();
      const touchEndTime = new Date().getTime();
      const { clientX } = getTouch(e, touchIdentifier);
      const deltaX = clientX - touchStartPointX;
      const isSlide2Left = deltaX < 0;
      let offset = 0;
      // 快速滑动，默认滑一屏
      if (touchEndTime - touchStartTime < MINIMUM_SLIDE_INTERVAL) {
        offset = isSlide2Left ? -perWidth : perWidth;
      } else if (deltaX) {
        const passBy = deltaX / perWidth;
        const up = Math.ceil(passBy);
        const down = Math.floor(passBy);
        if (up - passBy > passBy - down) {
          offset = down * perWidth;
        } else {
          offset = up * perWidth;
        }
      }

      let nextTranslateX = offset + currentTranslateX;
      const dataLen = this.data.length;
      if (0 < nextTranslateX <= perWidth || nextTranslateX < this.getMinTranslateX()) {
        this.isDuplicateMove = true;
      }
      this.currentTranslateX = nextTranslateX;
      this.resetState();
      addEvent(this.$refs.swiperDiv, 'transitionend', this.onTransitionend);
    },
    // 不算重复swiper的最大负偏移
    getMinTranslateX() {
      const { perWidth, data } = this;
      return - perWidth * (data.length - 1);
    },
    resetState() {
      this.deltaX = 0;
      this.touchStartPointX = 0;
      this.touchIdentifier = -1;
      this.isInTouchMoveAction = false;
      this.touchStartTime = 0;
      this.isClickEvent = true;
    },
    getAutoPlayTranslateX() {
      if (!this.$refs.swiperDiv) {
        return;
      }
      // 总是自右向左滑动
      const { currentTranslateX, isInTouchMoveAction, data, perWidth } = this;
      if (isInTouchMoveAction) {
        this.isInTouchMoveAction = false;
      }
      const length = data.length;
      const nextIndex = Math.abs(currentTranslateX) / perWidth + 1;
      this.currentTranslateX = - nextIndex * perWidth;
      // 为了实现无缝滚动效果
      if (nextIndex === length) {
        this.isDuplicateMove = true;
        addEvent(this.$refs.swiperDiv, 'transitionend', this.onTransitionend);
      }
    },
    cacNextIndex() {
      const { currentTranslateX, data, perWidth } = this;
      if (currentTranslateX > 0) {
        this.currentTranslateX = -perWidth * (data.length - 1);
      } else if (currentTranslateX < -perWidth * (data.length - 1)) {
        this.currentTranslateX = 0;
      }
    },
    setSwiperInterval() {
      if (this.intervalId) {
        clearInterval(this.intervalId);
      }
      this.intervalId = setInterval(() => {
        this.getAutoPlayTranslateX();
      }, 3000);
    },
    handleResize() {
      const newPerWidth = this.getPerWidth();
      if (newPerWidth !== this.perWidth) {
        this.perWidth = newPerWidth;
        this.currentTranslateX = 0;
      }
    },
    getPerWidth() {
      return this.$refs.swiperDiv.querySelector('.swiper-item').offsetWidth || screen.width - 32;
    }
  },
  mounted() {
    if (this.$refs.swiperDiv && this.data.length > 1) {
      addEvent(this.$refs.swiperDiv, 'touchstart', this.onTouchStart);
      addEvent(this.$refs.swiperDiv, 'touchmove', this.onTouchMove);
      addEvent(this.$refs.swiperDiv, 'touchend', this.onTouchEnd);
      this.perWidth = this.getPerWidth();
      this.setSwiperInterval();
      window.addEventListener('resize', this.handleResize);
    }
  },
  beforeDestroy() {
    if (this.$refs.swiperDiv) {
      removeEvent(this.$refs.swiperDiv, 'touchstart', this.onTouchStart);
      removeEvent(this.$refs.swiperDiv, 'touchmove', this.onTouchMove);
      removeEvent(this.$refs.swiperDiv, 'touchend', this.onTouchEnd);
      if (this.intervalId) {
        clearInterval(this.intervalId);
      }
      window.removeEventListener('resize', this.handleResize);
    }
  }
};
</script>

<style lang='less' scoped>
@swiper-height: 100px;

.swiper {
  position: relative;
  display: block;
  visibility: visible;
  overflow: hidden;
  width: 100%;
  height: @swiper-height;
  box-sizing: border-box;

  &-container {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    
    &-transition {
      transition: all .5s ease;
    }
  }

  &-item {
    position: absolute;
    will-change: auto;
    width: 100%;
    height: 100%;
    box-sizing: border-box;
    border-radius: 4px;
    overflow: hidden;
    background-size: cover;
  }

  &-indicator {
    position: absolute;
    bottom: 0px;
    height: 14px;
    width: 100%;
    text-align: center;
    justify-content: center;
    display: -webkit-box;
    display: -webkit-flex;
    display: flex;
    transform: translate3d(0, 0, 0);
  }

  &-dot {
    display: inline-block;
    zoom: 1;
    width: 4px;
    height: 3px;
    margin: 0 2px;
    background: #e8e8e8;
    transition: width .5s;

    &-active {
      width: 11px;
      background: #fff;
    }
  }
}
</style>
```
### 完工
自己的Swiper实现的很粗糙，也没有市面上的已有的组件那般功能强大，不过作为一个练手的小东西，还是很实在啦。
写的过程中没有最后贴代码这么顺利，遇到了许多问题：比如怎么区分点击和滑动；事件的绑定与解绑；vue的computed、method有啥不一样；镜像的那两帧什么时机变成真实的两帧等等。最后好赖都解决了，也算顺利完成。
PS：现在的日子，啥都写一点，就比如小程序和Vue，也不知道上手的顺序是不是搞错了。是该先写小程序呢，还是先写Vue呢？