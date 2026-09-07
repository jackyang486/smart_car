<template>
  <view class="joystick-container" @touchstart="handleTouchStart" @touchmove="handleTouchMove" @touchend="handleTouchEnd">
    <!-- 摇杆背景 -->
    <view class="joystick-background" :style="backgroundStyle">
		<!-- 摇杆手柄 -->
		  <view class="joystick-handle" 
				:animation="animationData"
		        :style="handleStyle"
		        :class="{'joystick-active': isActive}"></view>
		</view>	
	</view>
</template>

<script>
export default {
  props: {
    size: {
      type: Number,
      default: 150
    },
    handleSize: {
      type: Number,
      default: 60
    },
    color: {
      type: String,
      default: '#ffff7f'
    }
  },
  data() {
    return {
      isActive: false,
      handleX: 0,
      handleY: 0,
	  animationData:{},
      maxDistance: 0
    }
  },
  computed: {
    backgroundStyle() {
      return {
        width: `${this.size}px`,
        height: `${this.size}px`,
        backgroundColor: `${this.color}20`, // 添加透明度
        borderRadius: '50%'
      }
    },
    handleStyle() {
      return {
        width: `${this.handleSize}px`,
        height: `${this.handleSize}px`,
        backgroundColor: this.color,
        borderRadius: '50%',
        transform: `translate(${this.handleX}px, ${this.handleY}px)`,
        transition: this.isActive ? 'none' : 'transform 0.2s ease-out'
      }
    }
  },
  mounted() {
    this.maxDistance = (this.size - this.handleSize) / 2;
    this.resetPosition();
  },
  methods: {
    resetPosition() {
      this.handleX = 0;
      this.handleY = 0;
	  // 强制视图更新
	  const animation = uni.createAnimation({
	      duration: 200,
	      timingFunction: 'ease-out'
	    });
	    animation.translate(0, 0).step();
	    this.animationData = animation.export();
    },
    handleTouchStart(e) {
	  e.stopPropagation(); // 阻止事件冒泡
      this.isActive = true;
      this.updateHandlePosition(e.touches[e.touches.length - 1]);
    },
    handleTouchMove(e) {
      if (!this.isActive) return;
      e.preventDefault();
	  e.stopPropagation(); // 阻止事件冒泡
      this.updateHandlePosition(e.touches[e.touches.length - 1]);
      // 计算方向并触发事件
	  this.emitDirection();
    },
    handleTouchEnd(e) {
		e.preventDefault();
		e.stopPropagation();
      this.isActive = false;
	    // 添加延迟确保触摸事件完全结束
	  setTimeout(() => {
			this.resetPosition();
	  }, 50);  
	  this.$emit('change', { x: 0, y: 0, angle: 0, direction: 'neutral', distance: 0});
    }, 
    updateHandlePosition(touch) {
	const rect = uni.createSelectorQuery().in(this).select('.joystick-container').boundingClientRect();
	      rect.exec(res => {
	        if (res[0]) {
	          const centerX = res[0].left + res[0].width/2;
	          const centerY = res[0].top + res[0].height/2;
	          
	          let deltaX = touch.clientX - centerX;
	          let deltaY = touch.clientY - centerY;
	          
	          const distance = Math.sqrt(deltaX*deltaX + deltaY*deltaY);
	          if (distance > this.maxDistance) {
	            deltaX = deltaX / distance * this.maxDistance;
	            deltaY = deltaY / distance * this.maxDistance;
	          }
	          
	          this.handleX = deltaX;
	          this.handleY = deltaY;
	        }
	      });	
    },
    emitDirection() {
      const x = this.handleX / this.maxDistance; // 归一化到[-1, 1]
      const y = this.handleY / this.maxDistance;
      const angle = Math.atan2(y, x) * 180 / Math.PI; // 角度(-180到180)
      const distance = Math.min(1, Math.sqrt(x * x + y * y)); // 距离(0到1)
      // 确定8个基本方向
      let direction = 'neutral';
      if (distance > 0.3) { // 添加死区阈值
        const sector = Math.floor((angle + 180 + 22.5) / 45) % 8;
        const directions = ['left', 'left-up', 'up', 'right-up', 'right', 'right-down', 'down', 'left-down'];
        direction = directions[sector];
      }
      this.$emit('change', { x, y, angle, direction, distance});
    }
  }
}
</script>

<style scoped>
.joystick-container {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  touch-action: none;
  user-select: none;
}

.joystick-background {
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
}

.joystick-handle {
  transform: translate(-50%, -50%);
  will-change: transform;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.joystick-active {
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
}
</style>