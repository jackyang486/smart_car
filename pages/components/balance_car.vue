<template>
	<view class="controls">
    <view v-if="showBluetoothModal" class="bluetooth-modal">
      <view class="bluetooth-modal__content">
        <text class="bluetooth-modal__title">请选择蓝牙设备</text>
        <text v-if="isConnecting" class="bluetooth-modal__status">正在连接...</text>
        <text v-else-if="!blueToothList.length" class="bluetooth-modal__status">正在搜索蓝牙设备...</text>
        <view v-else class="bluetooth-device-list">
          <view
            v-for="device in blueToothList"
            :key="device.value"
            class="bluetooth-device"
            @tap="selectDevice(device.value)"
          >
            <text>{{ device.text }}</text>
            <text class="bluetooth-device__arrow">连接</text>
          </view>
        </view>
      </view>
    </view>
    <view class="connection-tools">
      <view class="connection-item">
        <text>连接状态：</text>
        <switch
          :checked="isConnected"
          :disabled="isConnecting"
          color="#09BB07"
          @change="toggleConnection"
        />
      </view>
      <view class="connection-item pid-control">
        <text>行驶速度</text>
        <uni-number-box
          v-model="drivingSpeedPid"
          :min="0"
          :max="100"
        />
      </view>
      <view class="connection-item pid-control">
        <text>转向速度</text>
        <uni-number-box
          v-model="steeringSpeedPid"
          :min="0"
          :max="100"
        />
      </view>
    </view>
    <view class="drive-control-zone">
      <view class="control-column left-column">
        <view class="controls_button updown-pad">
          <view class="up control-button" :class="{'buttion-active': buttonStatus.forwardPressed}" @touchstart="move($event,'forward')" @touchend="move($event,'forward')"></view>
          <view class="down control-button" :class="{'buttion-active': buttonStatus.backwardPressed}" @touchstart="move($event,'backward')" @touchend="move($event,'backward')"></view>
        </view>
      </view>
      <view class="control-column right-column">
        <view class="controls_button leftright-pad">
          <view class="left control-button" :class="{'buttion-active': buttonStatus.leftPressed}" @touchstart="move($event,'left')" @touchend="move($event,'left')"></view>
          <view class="right control-button" :class="{'buttion-active':  buttonStatus.rightPressed}" @touchstart="move($event,'right')" @touchend="move($event,'right')"></view>
        </view>
      </view>
    </view>
	</view>
</template>

<script>
export default {
  name: 'banlance_car',
  data() {
    return {
      value: '',
      checkBluetooth: null,
      deviceId: '',
      deviceName: 'JDY-31-SPP',
      serviceId: '',
      writeCharacteristicId: '',
      readCharacteristicId: '',
      isConnecting: false,
      isConnected: false,
      scanSuccess: false,
      dataType: Object.freeze({
        joystick: 0x32,
        speed: 0x02,
      }),
      speed: 0,
      drivingSpeedPid: 50,
      steeringSpeedPid: 50,
      joystickData:{
        ax_robot_vx: this.drivingSpeedPid,
        ax_robot_vw: this.steeringSpeedPid,
      },
      deviceList: [],
      blueToothList: [],
      showBluetoothModal: true,
      buttonStatus:{
        forwardPressed: false,
        backwardPressed: false,
        leftPressed: false,
        rightPressed: false
      },
    };
  },
  created() {
    plus.screen.lockOrientation('landscape');
    this.initialize();
  },
  computed: {},
  methods: {
    initialize() {
      this.requestPermission();
      this.checkBluetoothStatus();
      this.scanDevice();
    },
    requestPermission() {
      if (typeof plus === 'undefined' || !plus.android) {
        return;
      }

      uni.getSystemInfo({
        success: (res) => {
          const systemVersion = Number((res.system || '').replace(/[^0-9.]/g, '').split('.')[0] || 0);
          if (res.platform === 'android' && systemVersion >= 12) {
            plus.android.requestPermissions(
              [
                'android.permission.ACCESS_FINE_LOCATION',
                'android.permission.BLUETOOTH_SCAN',
                'android.permission.BLUETOOTH_CONNECT',
              ],
              (result) => {
                console.log('蓝牙权限申请结果', result);
              },
              (error) => {
                console.error('蓝牙权限申请失败', error);
              }
            );
          }
        },
      });
    },
    checkBluetoothStatus() {
      uni.openBluetoothAdapter({
        success: () => {
          console.log('蓝牙已开启');
          this.checkBluetooth = true;
        },
        fail: (err) => {
          this.checkBluetooth = false;
          console.error('蓝牙未开启或不可用', err);
          if (err && err.errCode === 10001) {
            uni.showToast({
              title: '蓝牙未开启',
              icon: 'error',
            });
          }
        },
      });
    },
    scanDevice() {
      this.deviceList = [];
      this.blueToothList = [];

      uni.showLoading({
        title: '蓝牙扫描中',
        mask: false,
      });

      uni.stopBluetoothDevicesDiscovery({
        success: () => console.log('已停止旧的 BLE 扫描'),
      });

      uni.startBluetoothDevicesDiscovery({
        allowDuplicatesKey: false,
        services: [],
        success: () => {
          console.log('开始扫描 BLE 设备');
          uni.onBluetoothDeviceFound((res) => {
            const devices = Array.isArray(res.devices) ? res.devices : [];
            devices.forEach((device) => {
              if (!device || !device.deviceId) {
                return;
              }
              const hasDevice = this.deviceList.some((item) => item.deviceId === device.deviceId);
              if (!hasDevice) {
                this.deviceList.push(device);
              }
            });

            this.blueToothList = this.deviceList
              .filter((item) => item.name || item.localName)
              .map((item) => ({
                value: item.deviceId,
                text: item.name || item.localName,
              }));

            this.scanSuccess = this.blueToothList.length > 0;
            if (this.scanSuccess) {
              uni.hideLoading();
            }
          });
        },
        fail: (err) => {
          uni.hideLoading();
          console.error('BLE 扫描失败', err);
        },
      });
    },
    stopScan() {
      uni.stopBluetoothDevicesDiscovery({
        success: () => console.log('停止扫描成功'),
      });
    },
    selectDevice(index) {
      this.value = index;
      if (!this.isConnecting && !this.isConnected) {
        this.connect(index);
      }
    },
    toggleConnection(event) {
      if (event.detail.value) {
        if (!this.value) {
          uni.showToast({
            title: '请先选择设备',
            icon: 'none',
          });
          return;
        }
        this.connect(this.value);
        return;
      }

      this.closeConnect();
    },
    connect(index) {
      const device = typeof index === 'number'
        ? this.deviceList[index]
        : this.deviceList.find((item) => item.deviceId === index);

      if (!device) {
        console.log('未找到设备', index);
        return;
      }

      this.deviceId = device.deviceId;
      this.deviceName = device.name || device.localName || '未知设备';
      this.isConnecting = true;
      this.stopScan();

      uni.createBLEConnection({
        deviceId: this.deviceId,
        success: () => {
          this.isConnected = true;
          console.log('BLE 连接成功');
		  //uni.setBLEMTU({
		  //    deviceId: this.deviceId,
		  //    mtu: 23, // 根据硬件支持设置，一般256-512
		  //    success: () => console.log('MTU设置成功'),
		  //    fail: (err) => console.error('MTU设置失败', err)
		  //});
		  setTimeout(() => {
		        this.getBLEDeviceServicesAndCharacteristics();
		  }, 500);
        },
        fail: (err) => {
          this.isConnecting = false;
		  this.isConnected = false;
          console.error('BLE 连接失败', err);
        },
      });
    },
    getBLEDeviceServicesAndCharacteristics() {
      uni.getBLEDeviceServices({
        deviceId: this.deviceId,
        success: ({ services }) => {
		console.info("serviceId:"+this.deviceId);	
		  // 精确匹配 FFE0 服务
		const service = services.find(s => 
		                  s.uuid.toUpperCase().includes('FFE0')
		              ) || services[0];
		              
    if (!service) {
      this.isConnecting = false;
      this.isConnected = false;
      uni.showToast({ title: '未找到蓝牙服务', icon: 'none' });
      return;
    }
		this.serviceId = service.uuid;
        uni.getBLEDeviceCharacteristics({
            deviceId: this.deviceId,
            serviceId: this.serviceId,
            success: ({ characteristics }) => {
			  console.info(JSON.stringify(characteristics));	
			  const writeChar = characteristics.find(item => 
			        item.uuid.toUpperCase().includes('FFE2')
			 );
			// ✅ 精确匹配 FFE1（通知特征值）
			 const readChar = characteristics.find(item => 
			        item.uuid.toUpperCase().includes('FFE1')
			);
              if (writeChar) {
                this.writeCharacteristicId = writeChar.uuid;
              }
              if (readChar) {
                this.readCharacteristicId = readChar.uuid;
              }
        if (!writeChar) {
          uni.showToast({ title: '未找到写入特征', icon: 'none' });
          this.isConnecting = false;
          this.isConnected = false;
          return;
        }
        this.sendConnectionFrame();
        setTimeout(() => {
          this.setDataReceive();
        }, 	500);
            },
            fail: (err) => {
              console.error('获取 BLE 特征失败', err);
			  this.isConnecting = false;
			  this.isConnected = false;
			  uni.showToast({ title: '获取蓝牙特征失败', icon: 'none' });
            },
          });
        },
        fail: (err) => {
          console.error('获取 BLE 服务失败', err);
		  this.isConnecting = false;
		  this.isConnected = false;
		  uni.showToast({ title: '获取蓝牙服务失败', icon: 'none' });
        },
      });
    },
    sendData(data, dataType) {
      if (!this.isConnected || !this.deviceId || !this.serviceId || !this.writeCharacteristicId) {
        console.warn('BLE 未准备好，无法发送');	
        return Promise.reject(new Error('BLE 未准备好'));
      }
      if (!Array.isArray(data)) {
        console.warn('BLE 数据必须是数组');
        return Promise.reject(new Error('BLE 数据必须是数组'));
      }
      const frame = new Uint8Array([
        0xAA,
        0x55,
        0x00,
        dataType,
        ...data.map((item) => item & 0xFF),
      ]);
      frame[2] = frame.length+1;
      const checksum = frame.reduce((sum, byte) => sum + byte, 0) & 0xFF;
      const dataBuffer = new Int8Array([...frame, checksum]).buffer;
    console.info('BLE 发送数据帧:', this.bytesToHex(dataBuffer));
    return this.safeWrite(this.deviceId, this.serviceId, this.writeCharacteristicId, dataBuffer);
    },
    sendConnectionFrame() {
      this.sendData([0x55], 0x30)
        .then(() => {
          this.isConnecting = false;
          this.showBluetoothModal = false;
          uni.hideLoading();
          uni.showToast({ title: '连接成功', icon: 'success' });
        })
        .catch((error) => {
          this.isConnecting = false;
          this.isConnected = false;
          console.error('蓝牙连接帧发送失败', error);
          uni.showToast({ title: '连接初始化失败', icon: 'none' });
        });
    },
  bytesToHex(buffer) {
    return Array.from(new Uint8Array(buffer))
      .map((byte) => byte.toString(16).padStart(2, '0').toUpperCase())
      .join(' ');
  },
	async safeWrite(deviceId, serviceId, characteristicId, buffer) {
	    try {
	        await new Promise((resolve, reject) => {
	            uni.writeBLECharacteristicValue({
	                deviceId,
	                serviceId,
	                characteristicId,
	                value: buffer,
	                // ⚠️ JDY-31 模块建议优先用 writeNoResponse
	                writeType: 'writeNoResponse',
	                success: resolve,
	                fail: reject
	            });
	        });
	        console.log('BLE 写入成功');
	        // 给模块 MCU 处理时间
	        await new Promise(r => setTimeout(r, 20));
	    } catch (e) {
	        console.error('BLE 写入失败:', e);
          throw e;
	    }
	},
    setDataReceive() {
      if (!this.isConnected || !this.deviceId || !this.serviceId || !this.readCharacteristicId) {
        return;
      }

      uni.notifyBLECharacteristicValueChange({
        deviceId: this.deviceId,
        serviceId: this.serviceId,
        characteristicId: this.readCharacteristicId,
        state: true,
        success: () => {
          console.log('开启 BLE 通知成功');
          uni.onBLECharacteristicValueChange((res) => {
            if (!res || !res.value) {
              return;
            }
            console.log('BLE 接收数据帧:', this.bytesToHex(res.value));
            const receivedFrame = new Uint8Array(res.value);
            if (receivedFrame.length >= 6 && receivedFrame[0] === 0xAA && receivedFrame[1] === 0x55) {
              const expectedChecksum = receivedFrame
                .slice(0, receivedFrame.length - 1)
                .reduce((sum, byte) => sum + byte, 0) & 0xFF;
              if (expectedChecksum !== receivedFrame[receivedFrame.length - 1]) {
                console.warn('BLE 接收帧校验失败');
              }
            }
          });
        },
        fail: (err) => {
          console.error('开启 BLE 通知失败', err);
        },
      });
    },
    cancelDataReceive() {
      if (!this.isConnected || !this.deviceId || !this.serviceId || !this.readCharacteristicId) {
        return;
      }

      uni.notifyBLECharacteristicValueChange({
        deviceId: this.deviceId,
        serviceId: this.serviceId,
        characteristicId: this.readCharacteristicId,
        state: false,
        success: () => {
          console.log('关闭 BLE 通知成功');
        },
        fail: (err) => {
          console.error('关闭 BLE 通知失败', err);
        },
      });
    },
    closeConnect() {
      if (!this.deviceId) {
        return;
      }

      uni.closeBLEConnection({
        deviceId: this.deviceId,
        success: () => {
          this.isConnected = false;
          console.log('断开 BLE 连接成功');
        },
        fail: (err) => {
          console.error('断开 BLE 连接失败', err);
        },
      });
    },
    str2ab(str) {
      const buffer = new ArrayBuffer(str.length);
      const view = new Uint8Array(buffer);
      for (let i = 0; i < str.length; i++) {
        view[i] = str.charCodeAt(i);
      }
      return buffer;
    },
    ab2str(buffer) {
      const view = new Uint8Array(buffer);
      let str = '';
      for (let i = 0; i < view.length; i++) {
        str += String.fromCharCode(view[i]);
      }
      return str;
    },
    move(e, direction) {
      e.stopPropagation();
      e.preventDefault();
      if(e.type === 'touchstart'){ //按钮按下
         switch(direction){
          case 'forward': this.buttonStatus.forwardPressed = true;break;
          case 'backward': this.buttonStatus.backwardPressed = true;break;
          case 'left': this.buttonStatus.leftPressed = true;break;
          case 'right': this.buttonStatus.rightPressed = true;break;
          default: 'stop';
         } 
      }else if(e.type === 'touchend'){
        switch(direction){
          case 'forward': this.buttonStatus.forwardPressed = false;break;
          case 'backward': this.buttonStatus.backwardPressed = false;break;
          case 'left': this.buttonStatus.leftPressed = false;break;
          case 'right': this.buttonStatus.rightPressed = false;break;
          default: 'stop';
         } 
      }
      //前进和后退
      if(this.buttonStatus.forwardPressed){
        this.joystickData.ax_robot_vx = this.drivingSpeedPid;
      }else if(this.buttonStatus.backwardPressed){
        this.joystickData.ax_robot_vx = -this.drivingSpeedPid;
      }else {
        this.joystickData.ax_robot_vx = 0x00;
        this.joystickData.ax_robot_vw = 0x00; 
      }
      //左转和右转
       if(this.buttonStatus.leftPressed){
        this.joystickData.ax_robot_vw = this.steeringSpeedPid;
      }else if(this.buttonStatus.rightPressed){
        this.joystickData.ax_robot_vw = -this.steeringSpeedPid;
      }else {
        this.joystickData.ax_robot_vw = 0x00; 
      }
      this.sendData([this.joystickData.ax_robot_vx,0x00,0x00,this.joystickData.ax_robot_vw], this.dataType.joystick); //手柄操作
    },
  },
};
</script>

<style scoped>
  .bluetooth-modal {
    position: fixed;
    z-index: 1000;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(0, 0, 0, 0.45);
  }
  .bluetooth-modal__content {
    width: min(520px, 82vw);
    max-height: 75vh;
    padding: 24px;
    overflow: auto;
    border-radius: 12px;
    background: #ffffff;
    box-shadow: 0 12px 40px rgba(0, 0, 0, 0.2);
  }
  .bluetooth-modal__title {
    display: block;
    margin-bottom: 16px;
    color: #222222;
    font-size: 20px;
    font-weight: 600;
    text-align: center;
  }
  .bluetooth-modal__status {
    display: block;
    padding: 24px 0;
    color: #666666;
    text-align: center;
  }
  .bluetooth-device {
    display: flex;
    align-items: center;
    justify-content: space-between;
    min-height: 48px;
    padding: 0 12px;
    border-bottom: 1px solid #eeeeee;
    color: #333333;
  }
  .bluetooth-device__arrow {
    color: #09bb07;
    font-size: 14px;
  }
	.controls {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    gap: 16px;
	  padding: 10px 30px;
	  background-color: #f0f0f0;
	}
  .connection-tools {
    display: flex;
    align-items: center;
    justify-content: flex-start;
    width: 100%;
  }
  .drive-control-zone {
    display: flex;
    justify-content: space-between;
    align-items: center;
    width: 100%;
    min-height: 220px;
    padding: 10px 12px 12px;
    box-sizing: border-box;
  }
  .control-column {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 50%;
    min-height: 200px;
  }
  .left-column {
    justify-content: flex-start;
    padding-left: 8px;
  }
  .right-column {
    justify-content: flex-end;
    padding-right: 8px;
  }
  .connection-item {
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .connection-item:first-child {
    flex: 1;
    min-width: 0;
  }
  .connection-tools uni-data-select {
    flex: 1;
    min-width: 0;
  }
  .speed-control slider {
    width: 160px;
  }
  .pid-control {
    white-space: nowrap;
  }
	.controls_button{ 
	  display: flex;
    align-items: center;
    justify-content: center;
    gap: 18px;
	  width: 170px;
    height: 170px;
    padding: 12px;
    border-radius: 36px;
    background: rgba(255,255,255,0.28);
    box-shadow: inset 0 0 0 1px rgba(0,0,0,0.06);
  }
	.controls_button .control-button {
	  display: block;
	  width: 62px;
	  height: 62px;
	  border-radius: 20px;
	  background-size: cover;
	  background-position: center;
	  box-shadow: inset 0 0 0 1px rgba(0,0,0,0.06), 0 4px 10px rgba(0,0,0,0.08);
	  transition: transform 0.12s ease, box-shadow 0.12s ease;
	}
	.controls_button .control-button:active,
	.controls_button .control-button.active,
	.controls_button .control-button.buttion-active {
	  transform: scale(0.96);
	  box-shadow: inset 0 0 0 1px rgba(0,0,0,0.08), 0 2px 8px rgba(0,0,0,0.18);
	}
	.controls_button.updown-pad {
	  flex-direction: column;
	}
	.controls_button.leftright-pad {
	  flex-direction: row;
	}
	.controls_button .up{
		background-image: url('/static/front.png');
	}
	.controls_button .down{
		background-image: url('/static/back.png');
	}
	.controls_button .left{
		background-image: url('/static/left.png');
	}
	.controls_button .right{
		background-image: url('/static/right.png');
	}
	.controls_button .stop{
		border-radius: 50%;
		background-size: cover;
		background-position: center;
		background-color:  #f0f0f0;
	}
	.otherButton {
		display: flex;
		justify-content: flex-start;
		flex-direction: column;
		width: 50px;
	}
	.otherButton .bulb-btn{
		width: 100%;
		height: 50px;
		border-radius: 15px;
		overflow: hidden;
	}
	.buttion-active {
	  box-shadow: 0px 4px 15px rgba(0, 0, 0, 0.3), inset 0 0 0 1px rgba(0,0,0,0.08);
	  transform: scale(0.96);
	}
</style>