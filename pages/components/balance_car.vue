<template>
	<view class="controls">
    <view class="connection-tools">
      <view class="connection-item">
        <text>蓝牙列表：</text>
        <uni-data-select v-if="scanSuccess" v-model="value"
          :localdata="blueToothList"
          @change="selectDevice"
        ></uni-data-select>
      </view>
      <view class="connection-item">
        <text>连接状态：</text>
        <switch
          :checked="isConnected"
          :disabled="isConnecting"
          color="#09BB07"
          @change="toggleConnection"
        />
      </view>
      <view class="connection-item speed-control">
        <text>速度：</text>
        <slider
          :value="speed"
          min="0"
          max="10"
          step="2"
          activeColor="#09BB07"
          @change="changeSpeed"
        />
        <text>{{ speed }}</text>
      </view>
    </view>
    <view class="controls_button">
		  <div class="up" :class="{'buttion-active': direction!= 'forward'}" @touchstart="move($event,'forward')" 
		   @touchend="move($event,'stop')" ></div>
		  <div class="down" :class="{'buttion-active': direction!= 'backward'}" @touchstart="move($event,'backward')"
		   @touchend="move($event,'stop')" ></div>
		  <div class="left" :class="{'buttion-active': direction!= 'left'}" @touchstart="move($event,'left')" 
		   @touchend="move($event,'stop')" ></div> 
		  <div class="right" :class="{'buttion-active': direction!= 'right'}" @touchstart="move($event,'right')" 
		   @touchend="move($event,'stop')" ></div>
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
      direction: '',
      deviceId: '',
      deviceName: 'JDY-31-SPP',
      serviceId: '',
      writeCharacteristicId: '',
      readCharacteristicId: '',
      isConnecting: false,
      isConnected: false,
      scanSuccess: false,
      dataType: Object.freeze({
        direction: 0x01,
        speed: 0x02,
      }),
      speed: 0,
      deviceList: [],
      blueToothList: [],
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

      uni.createBLEConnection({
        deviceId: this.deviceId,
        success: () => {
          this.isConnected = true;
          this.isConnecting = false;
          console.log('BLE 连接成功');
      uni.showToast({
        title: '连接成功',
        icon: 'success',
      });
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
		              
		if (!service) return;
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
				setTimeout(() => {
					this.setDataReceive();
				}, 	500);
            },
            fail: (err) => {
              console.error('获取 BLE 特征失败', err);
            },
          });
        },
        fail: (err) => {
          console.error('获取 BLE 服务失败', err);
        },
      });
    },
    sendData(data, dataType) {
      if (!this.isConnected || !this.deviceId || !this.serviceId || !this.writeCharacteristicId) {
        console.warn('BLE 未准备好，无法发送');	
        return;
      }
      if (!Array.isArray(data)) {
        console.warn('BLE 数据必须是数组');
        return;
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
      const dataBuffer = new Uint8Array([...frame, checksum]).buffer;
    console.info('BLE 发送数据帧:', this.bytesToHex(dataBuffer));
    this.safeWrite(this.deviceId, this.serviceId, this.writeCharacteristicId, dataBuffer);
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
      const directionData = {
        forward: 0x01,
        backward: 0x02,
        left: 0x04,
        right: 0x08,
        stop: 0x00,
      };
      this.direction = direction;
      this.sendData([directionData[direction] || 0x00], this.dataType.direction);
    },
    changeSpeed(e) {
      this.speed = e.detail.value;
      this.sendData([this.speed], this.dataType.speed);
    }
  },
};
</script>

<style scoped>
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
    justify-content: center;
    gap: 16px;
    width: 100%;
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
	.controls_button{ 
	  display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-template-rows: repeat(3, 1fr);
	  width: 220px;
    height: 220px;
  }
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