<template>
  <div >
    <!-- WiFi 连接配置 -->
	<uni-section class="main" title="连接配置" v-if="!connected" type="line">
    <div class="wifi-config" >
	 <div>
	  <div class="radio-button">
			<button :class="{ active: selected === 'car_remote' }" @click="selectOption('car_remote')">监控小车</button>
			<button :class="{ active: selected === 'electronic_dog' }" @click="selectOption('electronic_dog')">电子宠物狗</button>
			<button :class="{ active: selected === 'balance_car' }" @click="selectOption('balance_car')">平衡小车</button>
	  </div> 
      <input v-model="serverIP" placeholder="输入IP" />
      <input v-model="serverPort" placeholder="输入端口" /> 
      <button size="default" @click="connectToEsp8266" class="connect-button">连接</button>
	  </div>	 
    </div>  
	</uni-section>
	<uni-section title="控制面板" type="line" v-if="connected">
	  <remote_car v-if="connected && selected === 'car_remote' "
	  :moe-tcp-client.sync="moeTcpClient" 
	  :connected.sync="connected" 
	  :videoUrl.sync="videoUrl"
	  ></remote_car>
	  <electronic_dog  v-if="connected && selected === 'electronic_dog' "
	  :moe-tcp-client.sync="moeTcpClient" 
	  :connected.sync="connected" 
	  ></electronic_dog>		
	  <balance_car v-if="connected && selected === 'balance_car'" ></balance_car>
	</uni-section>
  </div>
</template>
 
<script>
import remote_car from '@/pages/components/remote_car.vue';
import balance_car from '@/pages/components/balance_car.vue';
import electronic_dog from '@/pages/components/electronic_dog.vue';
export default { 
  name: 'WifiCarRemote',
  components: {remote_car,electronic_dog,balance_car},
  data() {
    return {
      serverIP: '', // ESP8266 IP
      serverPort: '', // ESP8266 端口
	  videoUrl:'',
	  moeTcpClient: null,
	  connected:false,
	  selected:'car_remote',
    }; 
  },
  created() {
	  this.moeTcpClient = uni.requireNativePlugin("moe-tcp-client");
	  this.moeTcpClient.onDisconnect(res => {
			this.connected = false;
	  });
	  this.moeTcpClient.onReceive(res => {
		 let result =  JSON.parse(res); 
		 if(result.code === 1){
		 } 
	  });
  },
  methods: {
    // 连接ESP8266
    connectToEsp8266() {
      if (!this.serverIP || !this.serverPort) {
        uni.showToast({
            title: '请输入IP和端口',
            icon: 'none',
        })
        return;  
      }
	  this.videoUrl = `http://${this.serverIP}:81/stream?t=${Date.now()}`;
	  uni.showLoading({ title: '连接中...' });
	   this.moeTcpClient.connect({
	    ip: this.serverIP,
	    port: this.serverPort
	  }, result => {
		 uni.hideLoading();
		 const res  = JSON.parse(result)
	     if(res.code == 1){
			 uni.showToast({
			     title: res.msg,
			 })
			  this.connected = true;
		 }else{
			 uni.showToast({
			     title: res.msg,
				 icon: 'error',
			 });
			 plus.screen.unlockOrientation();
		 }
	  });
    },
	//功能选择
	selectOption(option) {
		this.selected = option;
		if(option === 'balance_car'){
		   this.connected = true;	
		}
	     
	}
  },
};
</script>

<style scoped>
.main{
	background-image: url('/static/baby.jpg');
	background-repeat: no-repeat;
	background-position: center;
	background-size: cover;
	width: 100%;
} 
.wifi-config {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

.wifi-config input {
  padding: 10px;
  margin: 5px;
  width: 200px;
  border: 1px solid #ccc;
  border-radius: 5px;
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

.radio-button{
	display: flex;
	flex-direction: row;	
}
button {
  border: 1px solid #ccc; /* 边框 */
  background-color: #f9f9f9; /* 背景颜色 */
  cursor: pointer; /* 鼠标指针 */
}

.active {
  background-color: lightgreen; /* 选中时背景颜色 */
}
</style>