<template>
  <view class="vertical-slider-container">
    <!-- 垂直滑动条主体 -->
    <view 
      class="slider-rail"
      @touchstart="onTouchStart"
      @touchmove="onTouchMove"
      @touchend="onTouchEnd"
    >
      <!-- 进度条 -->
	  <view class="slider-process">
		<view class="slider-track" :style="{ height: trackHeight + '%' }">
			<!-- 滑块 -->
			<view 
			  class="slider-thumb" 
			  :style="{ bottom: thumbPosition + '%' }"
			></view>
		</view>
	  </view>
    </view>
  </view>
</template>

<script>
export default {
  props: {
    min: {
      type: Number,
      default: 0
    },
    max: {
      type: Number,
      default: 100
    },
    step: {
      type: Number,
      default: 1
    },
    unit: {
      type: String,
      default: '%'
    },
    defaultValue: {
      type: Number,
      default: 0
    }
  },
  data() {
    return {
      currentValue: this.defaultValue,
      sliderHeight: 0,
      startY: 0,
      startValue: 0
    }
  },
  computed: {
    // 计算进度条高度百分比
    trackHeight() {
      return ((this.currentValue - this.min) / (this.max - this.min)) * 100
    },
    // 计算滑块位置百分比
    thumbPosition() {
      return this.trackHeight
    },
    // 生成刻度线位置
    marks() {
      const marks = []
      const steps = (this.max - this.min) / this.step
      for (let i = 0; i <= steps; i++) {
        marks.push((i / steps) * 100)
      }
      return marks
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.initSliderHeight()
    })
  },
  methods: {
    // 初始化滑动条高度
    initSliderHeight() {
      const query = uni.createSelectorQuery().in(this)
      query.select('.slider-rail').boundingClientRect(rect => {
        if (rect) {
          this.sliderHeight = rect.height
        }
      }).exec()
    },
    
    // 触摸开始
    onTouchStart(e) {
      this.startY = e.touches[0].clientY
      this.startValue = this.currentValue
      this.updateValue(e)
    },
    
    // 触摸移动
    onTouchMove(e) {
      e.preventDefault()
      this.updateValue(e)
    },
    
    // 触摸结束
    onTouchEnd() {
      this.$emit('change', this.currentValue)
    },
    
    // 更新当前值
    updateValue(e) {
      const currentY = e.touches[0].clientY
      const query = uni.createSelectorQuery().in(this)
      query.select('.slider-rail').boundingClientRect(rect => {
        if (!rect) return
        
        // 计算相对位置 (从底部开始计算)
        const position = rect.bottom - currentY
        let percentage = position / rect.height
        
        // 限制在0-1范围内
        percentage = Math.max(0, Math.min(1, percentage))
        
        // 计算实际值
        let value = this.min + percentage * (this.max - this.min)
        
        // 应用步长
        value = Math.round(value / this.step) * this.step
        
        // 限制在min-max范围内
        value = Math.max(this.min, Math.min(this.max, value))
        
        this.currentValue = value
        this.$emit('input', value)
      }).exec()
    },
    
    // 外部设置值
    setValue(value) {
      this.currentValue = Math.max(this.min, Math.min(this.max, value))
    }
  }
}
</script>

<style lang="scss" scoped>
.vertical-slider-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 5rpx;
}

.value-display {
  font-size: 32rpx;
  color: #333;
  margin-bottom: 20rpx;
  font-weight: bold;
}

.slider-rail {
  position: relative;
  width: 50rpx;
  height: 100%;
  background-color: #f0f0f0;
  border-radius: 50%;
  touch-action: none;
}
.slider-process{
	height: 100%;
	border-radius: 20%;
	background-color:chocolate;
}
.slider-track {
  position: absolute;
  bottom: 0;
  width: 100%;
  background-color: #4cd964;
  border-radius: 20%;
  
}

.slider-thumb {
  position: absolute;
  left: 50%;
  width: 50rpx;
  height: 50rpx;
  background-color: #fff;
  border: 4rpx solid #4cd964;
  border-radius: 50%;
  transform: translate(-50%, 50%);
  z-index: 5;
  box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.2);
}

/* 响应式调整 */
@media (max-width: 768px) {
  .slider-rail {
    height: 300rpx;
  }
}
</style>