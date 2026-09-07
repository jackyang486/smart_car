<template>
	<div>
		<uni-section title="动作类型" type="line" >
			<div class="action_class" >
				<button type="primary" class="item" @click="move('standUp')">起立</button>
				<button type="primary" class="item" @click="move('sitDown')">坐下</button>
				<button type="primary" class="item" @click="move('stayDown')">趴下</button>
				<button type="primary" class="item" @click="move('handShake')">握手</button>
				<button type="primary" class="item" @click="move('swing')">摇摆</button>
				<button type="primary" class="item" @click="move('leftSpin')">左转圈</button>
				<button type="primary" class="item" @click="move('rightSpin')">右转圈</button>
				<button type="primary" class="item" @click="move('stretch')">伸懒腰</button>
				<button type="primary" class="item" @click="move('wagging')">摇尾巴</button>
				<button type="primary" class="item" @click="move('stop')">停止</button>
			</div>
		</uni-section>
		<uni-section title="摇杆" type="line" >
			<view class="controls" >
				<joystick
				  ref="leftJoystick"
				  :size="150"
				  :handleSize="65"
						:reset="true"
				  color="#4CAF50"
				  @change="handleJoystickLeft"
				  class="left-joystick">
				</joystick>
			</view>
		</uni-section>
	</div>
</template>

<script>
import  joystick from '@/pages/joystick/joystick.vue'

export default {	
  name: 'remote_car',
  components: {joystick},
  props:{
	  moeTcpClient:{
		  type: Object,
		  default: null
	  },
	  connected:{
		  type: Boolean,
		  default: false
	  }
  },
  data() {
    return {
		moveParam:{
				direction:'neutral',
				angle:0,
				distance:0,
		}
    }
  },
  created() {
  },
  computed:{
  },
  methods: {
	  handleJoystickLeft(data) {
	    this.moveParam.angle = data.angle;
	    this.moveParam.distance = data.distance;
		if(this.moveParam.direction != data.direction){
			this.moveParam.direction = data.direction;
			switch (this.moveParam.direction){
				case 'up': this.move('forward'); break;
				case 'down': this.move('backward'); break;
				case 'left': this.move('left'); break;
				case 'right': this.move('right'); break;
				case 'neutral': this.move('stop'); break;
				default: this.move('stop');
			}
		}
	  },
    // 控制小车移动
    move(direction) {
       console.info(direction);
       if(!this.connected) return;
       this.moeTcpClient.sendStr({
    		message: direction+"\n"
    	});
    }
  }
};
</script>

<style>
	.container {
		padding: 20px;
		font-size: 14px;
		line-height: 24px;
	}
	.wifi-config {
	  display: flex;
	  justify-content: center;
	}
	.wifi-config input {
	  padding: 10px;
	  margin: 5px;
	  width: 200px;
	  border: 1px solid #ccc;
	  border-radius: 5px;
	}
	.action_class{
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		flex-wrap: wrap;
		width: 90%;
	}
	.controls {
	  display: flex;
	  justify-content: center;
	  padding: 10px;
	}
	.action_class .item{
		width: 100px;
		margin-top: 20px;
	}
	.connect-button {
	  background-color: #42b983;
	  color: white;
	  border: none;
	  border-radius: 5px;
	  cursor: pointer;
	}
	.connect-button:hover {
	  background-color: #369f6e;
	}
</style>
