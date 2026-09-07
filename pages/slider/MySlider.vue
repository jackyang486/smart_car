<template>
  <view class="container">
    <!-- 显示当前速度百分比 -->
    <view>当前速度: {{speed}}%</view>
    
    <!-- 垂直滑动条容器 -->
    <view class="vertical-slider-container">
      <!-- 通过旋转实现垂直滑动条 -->
      <slider 
        :value="speed" 
        min="0" 
        max="100" 
		:step="5"
        activeColor="#4cd964" 
        block-size="20"
        @changing="onSliderChanging" 
        @change="onSliderChange"
        style="transform: rotate(270deg); width: 50rpx; height: 60rpx;"
      />
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      speed: 0 // 初始速度为0%
    }
  },
  methods: {
    onSliderChanging(e) {
      this.speed = e.detail.value
    },
    onSliderChange(e) {
      this.speed = e.detail.value
      this.sendSpeedToServer(this.speed)
    },
    sendSpeedToServer(speed) {
      // 发送速度到服务器的代码
    }
  }
}
</script>

<style>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 5rpx;
}

.vertical-slider-container {
  height: 50rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 1rpx 0;
}

</style>