<template>
	<view class="controls">
	 <view class="controls_button">
		  <div class="up" :class="{'buttion-active': direction!= 'forward'}" @touchstart="move($event,'forward')" 
		   @touchend="move($event,'stop')" ></div>
		  <div class="down" :class="{'buttion-active': direction!= 'backward'}" @touchstart="move($event,'backward')"
		   @touchend="move($event,'stop')" ></div>
		  <div class="left" :class="{'buttion-active': direction!= 'left'}" @touchstart="move($event,'left')" 
		   @touchend="move($event,'stop')" ></div> 
		  <div class="right" :class="{'buttion-active': direction!= 'right'}" @touchstart="move($event,'right')" 
		   @touchend="move($event,'stop')" ></div>
		   <div class="stop" @click="move($event,'stop')" >
			 <image style="width: 100%;height: 100%;" src="/static/stop.png"></image>
			</div>
	  </view>
	  <!-- 图传 -->
	  <view>
		<img  ref="mjpegStream" :src="videoUrl" style="width: 380px;height: 296px;" alt="ESP32-CAM Stream">
	  </view>
	  <!-- 其他功能按钮 -->
	  <div class="otherButton">
	  				  <div  class="bulb-btn" @click="speedControl($event,'add')">
	  					<image style="width: 100%;height: 100%;" src="/static/front.png"></image>
	  				   </div>
	  				  <div class="bulb-btn"  @click="speedControl($event,'dec')">
	  					<image style="width: 100%;height: 100%;"  src="/static/back.png"></image>
	  				  </div>
	  				  <div class="bulb-btn" @click="turnLight" >
	  					 <img style="width: 100%; height: 100%;" :src="lambUrl" alt="灯泡" class="bulb-image">
	  				  </div>
						<switch checked color="#FFCC33" style="transform:scale(0.7)" @change="changeMode"/>
	  </div>
	  <joystick
	    ref="rightJoystick"
	    :size="150"
	    :handleSize="60"
	    color="#ffff00"
	    @change="handleJoystickRight"
	    >
	  </joystick>
	</view>
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
	  videoUrl: {
		  type: String,
		  default: ''
	  },
	  connected:{
		  type: Boolean,
		  default: false
	  }
  },
  data() {
    return {
	  lambState:false,
	  speed:0,
	  direction: '',
	  carParam:{
		direction:'neutral',
		angle:0,
		distance:0,
	  },
	  panAngle: 90,  // 水平舵机角度 (0-180)
	  tiltAngle: 90, // 垂直舵机角度 (0-180)
    };  
  },
  created() {
	  plus.screen.lockOrientation('landscape');
  },
  computed:{
	lambUrl(){
		let url = ''
		if(this.lambState){
			url = 'static/turn_on.jpeg';
		}else{
			url = 'static/turn_off.jpeg';
		}
		return url;
	}  
  },
  methods: {
	  handleJoystickRight(data) {
		 const x = data.x;
		 const y = data.y;
		 this.panAngle = -Math.round(data.x*90)+90;
		 this.tiltAngle = Math.round(data.y*90)+90;
		 this.servoControl(this.panAngle,this.tiltAngle);
	  },
    // 控制小车移动
    move(e,direction) {
	   e.stopPropagation();
	   e.preventDefault();
	   console.info(direction);
	   this.direction = direction;
	   if(!this.connected) return;
	   	const cmd = { 
	   		          type: 'move_control',
	   		          direction: direction,
	   		        };
	   	this.moeTcpClient.sendStr({
	   		 			message: JSON.stringify(cmd)+"\n"
	   	});
    },
	// 舵机控制
	servoControl(pan, tilt) {
		if(!this.connected) return;
		const cmd = { 
		          type: 'servo_control',
		          pan: pan,
		          tilt: tilt
		        };
			this.moeTcpClient.sendStr({
				 			message: JSON.stringify(cmd)+"\n"
			});		
	},
	//速度控制
	speedControl(e,type){
		 e.stopPropagation();
		 e.preventDefault();
		  if(!this.connected) return;
		  const cmd = { 
		            type: 'speed_control',
		            speed: type
		          };
		  	this.moeTcpClient.sendStr({
		  		 			message: JSON.stringify(cmd)+"\n"
		  	});	
	},
	//开关灯
	turnLight(){
		if(this.lambState){
			this.lambState = false;
		}else{
			this.lambState = true;
		}
		const cmd = {
		          type: 'lamb_control',
		          lambState: this.lambState
		        };
			this.moeTcpClient.sendStr({
				 			message: JSON.stringify(cmd)+"\n"
			});	
	},
	//模式切换
	changeMode(e){
	    const cmd = {
		          type: 'mode_change',
		          enable: e.target.value
		        };
			this.moeTcpClient.sendStr({
				 			message: JSON.stringify(cmd)+"\n"
		});
	},
  },
};
</script>

<style scoped>
	.controls {
	  display: flex;
	  justify-content: space-between;
	  padding: 5px;
	  background-color: #f0f0f0;
	}
	.controls_button{
	  display: grid;
	  grid-template-columns: 33.3% 33.3% 33.3%;
	  grid-template-rows: 33.3% 33.3% 33.3%;
	  width: 220px;
	},
	.controls_button .up{
		border-radius: 15px;
		grid-column: 2 / 2;
		grid-row: 1 / 1;
		background-image: url('/static/front.png');
		background-size: cover;
		background-position: center;
	}
	.controls_button .down{
		border-radius: 15px;
		grid-column: 2 / 2;
		grid-row: 3 / 3;
		background-image: url('/static/back.png');
		background-size: cover;
		background-position: center;
	}
	.controls_button .left{
		border-radius: 15px;
		grid-column: 1 / 1;
		grid-row: 2 / 2;
		background-image: url('/static/left.png');
		background-size: cover;
		background-position: center;
	}
	.controls_button .right{
		border-radius: 15px;
		grid-column: 3 / 3;
		grid-row: 2 / 2;
		background-image: url('/static/right.png');
		background-size: cover;
		background-position: center;
	}
	.controls_button .stop{
		border-radius: 50%;
		grid-column: 2 / 2;
		grid-row: 2 / 2;
		background-size: cover;
		background-position: center;
		background-color:  #f0f0f0;
	}
	.otherButton {
		display: flex;
		justify-content: flex-start;
		flex-direction: column;
		columns: 1;
		width: 50px;
	}
	.otherButton .bulb-btn{
		width: 100%;
		height: 50px;
		border-radius: 15px;
		overflow: hidden;
	}
	.buttion-active {
	  box-shadow: 0px 4px 15px rgba(0, 0, 0, 0.3);
	}
</style>