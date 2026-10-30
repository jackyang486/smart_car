<template>
	<view class="gamepad">
		<!-- 状态栏 -->
		<view class="status-bar">
			<text class="status-text">通道：{{ transportText }}</text>
			<text class="status-text">状态：{{ statusText }}</text>
			<text class="status-text frame">帧：{{ frameHex }}</text>
			<view class="switch-item">
				<text>BLE 直连</text>
				<switch :checked="transport === 'ble'" color="#FFCC33" style="transform:scale(0.7)" @change="toggleTransport" />
			</view>
			<view class="switch-item">
				<text>JSON 封装</text>
				<switch :checked="protocol === 'json'" color="#FFCC33" style="transform:scale(0.7)" @change="toggleProtocol" />
			</view>
		</view>

		<!-- BLE 设备列表（可选通道） -->
		<view v-if="transport === 'ble' && !bleReady" class="ble-panel">
			<text class="ble-panel__title">BLE 设备（ESP32S3 NES Gamepad）</text>
			<view v-if="bleDevices.length" class="ble-device-list">
				<view
					v-for="device in bleDevices"
					:key="device.deviceId"
					class="ble-device"
					@tap="connectBle(device.deviceId)"
				>
					<text>{{ device.name }}</text>
					<text class="ble-device__action">连接</text>
				</view>
			</view>
			<text v-else class="ble-panel__empty">正在搜索蓝牙设备...</text>
		</view>

		<!-- 手柄本体 -->
		<view class="pad-body">
			<!-- 方向键 -->
			<view class="dpad">
				<view
					class="dpad-btn dpad-up"
					:class="{ 'btn-active': pressed.up }"
					@touchstart.stop.prevent="press('up')"
					@touchend.stop.prevent="release('up')"
					@touchcancel.stop.prevent="release('up')"
				></view>
				<view
					class="dpad-btn dpad-down"
					:class="{ 'btn-active': pressed.down }"
					@touchstart.stop.prevent="press('down')"
					@touchend.stop.prevent="release('down')"
					@touchcancel.stop.prevent="release('down')"
				></view>
				<view
					class="dpad-btn dpad-left"
					:class="{ 'btn-active': pressed.left }"
					@touchstart.stop.prevent="press('left')"
					@touchend.stop.prevent="release('left')"
					@touchcancel.stop.prevent="release('left')"
				></view>
				<view
					class="dpad-btn dpad-right"
					:class="{ 'btn-active': pressed.right }"
					@touchstart.stop.prevent="press('right')"
					@touchend.stop.prevent="release('right')"
					@touchcancel.stop.prevent="release('right')"
				></view>
				<view class="dpad-center"></view>
			</view>

			<!-- Select / Start -->
			<view class="center-btns">
				<view
					class="pill-btn"
					:class="{ 'btn-active': pressed.select }"
					@touchstart.stop.prevent="press('select')"
					@touchend.stop.prevent="release('select')"
					@touchcancel.stop.prevent="release('select')"
				>
					<text>Select</text>
				</view>
				<view
					class="pill-btn"
					:class="{ 'btn-active': pressed.start }"
					@touchstart.stop.prevent="press('start')"
					@touchend.stop.prevent="release('start')"
					@touchcancel.stop.prevent="release('start')"
				>
					<text>Start</text>
				</view>
			</view>

			<!-- 功能键 A/B/X/Y -->
			<view class="action-btns">
				<view class="action-row">
					<view
						class="round-btn"
						:class="{ 'btn-active': pressed.x }"
						@touchstart.stop.prevent="press('x')"
						@touchend.stop.prevent="release('x')"
						@touchcancel.stop.prevent="release('x')"
					>
						<text>X</text>
					</view>
					<view
						class="round-btn"
						:class="{ 'btn-active': pressed.y }"
						@touchstart.stop.prevent="press('y')"
						@touchend.stop.prevent="release('y')"
						@touchcancel.stop.prevent="release('y')"
					>
						<text>Y</text>
					</view>
				</view>
				<view class="action-row">
					<view
						class="round-btn"
						:class="{ 'btn-active': pressed.b }"
						@touchstart.stop.prevent="press('b')"
						@touchend.stop.prevent="release('b')"
						@touchcancel.stop.prevent="release('b')"
					>
						<text>B</text>
					</view>
					<view
						class="round-btn"
						:class="{ 'btn-active': pressed.a }"
						@touchstart.stop.prevent="press('a')"
						@touchend.stop.prevent="release('a')"
						@touchcancel.stop.prevent="release('a')"
					>
						<text>A</text>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
// NES 按键位图：bit0-9 => 上 下 左 右 Select Start A B X Y
const BUTTON_BITS = Object.freeze({
  up: 1 << 0,
  down: 1 << 1,
  left: 1 << 2,
  right: 1 << 3,
  select: 1 << 4,
  start: 1 << 5,
  a: 1 << 6,
  b: 1 << 7,
  x: 1 << 8,
  y: 1 << 9,
});

// BLE 直连（可选通道）使用的服务与写入特征
const BLE_SERVICE_UUID = '6E400001-B5A3-F393-E0A9-E50E24DCCA9E';
const BLE_WRITE_CHAR_UUID = '6E400002-B5A3-F393-E0A9-E50E24DCCA9E';
const BLE_DEVICE_NAME = 'ESP32S3 NES Gamepad';

export default {
  name: 'nes_gamepad',
  props: {
    moeTcpClient: {
      type: Object,
      default: null,
    },
    connected: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      transport: 'tcp', // tcp：TCP Server（默认） / ble：BLE 直连（可选）
      protocol: 'hex', // hex：2 字节小端位图 / json：JSON 文本
      pressed: {
        up: false,
        down: false,
        left: false,
        right: false,
        select: false,
        start: false,
        a: false,
        b: false,
        x: false,
        y: false,
      },
      bitmap: 0,
      lastSent: -1,
      frameHex: '00 00',
      // BLE 相关（可选通道）
      bleDevices: [],
      bleDeviceId: '',
      bleServiceId: '',
      bleWriteCharId: '',
      bleReady: false,
    };
  },
  computed: {
    transportText() {
      return this.transport === 'ble' ? 'BLE 直连' : 'TCP Server';
    },
    statusText() {
      if (this.transport === 'ble') {
        return this.bleReady ? '已连接' : '未连接';
      }
      return this.connected ? '已连接' : '未连接';
    },
  },
  created() {
    if (typeof plus !== 'undefined' && plus.screen) {
      plus.screen.lockOrientation('landscape');
    }
  },
  destroyed() {
    this.releaseAll();
    if (this.transport === 'ble') {
      this.stopBleScan();
    }
  },
  methods: {
    // 按下 / 松开
    press(key) {
      if (this.pressed[key]) return;
      this.pressed[key] = true;
      this.refreshBitmap();
    },
    release(key) {
      if (!this.pressed[key]) return;
      this.pressed[key] = false;
      this.refreshBitmap();
    },
    releaseAll() {
      Object.keys(this.pressed).forEach((key) => {
        this.pressed[key] = false;
      });
      this.refreshBitmap();
    },
    // 计算 10 位按键状态位图
    refreshBitmap() {
      let value = 0;
      Object.keys(BUTTON_BITS).forEach((key) => {
        if (this.pressed[key]) {
          value |= BUTTON_BITS[key];
        }
      });
      this.bitmap = value & 0xffff;
      this.sendFrame();
    },
    toHex(byte) {
      return ('0' + (byte & 0xff).toString(16)).slice(-2).toUpperCase();
    },
    // 发送 2 字节小端按键位图
    sendFrame() {
      const lo = this.bitmap & 0xff;
      const hi = (this.bitmap >> 8) & 0xff;
      this.frameHex = `${this.toHex(lo)} ${this.toHex(hi)}`;
      if (this.bitmap === this.lastSent) return;
      this.lastSent = this.bitmap;

      if (this.transport === 'ble') {
        this.writeBle([lo, hi]);
        return;
      }
      if (!this.connected || !this.moeTcpClient) {
        return;
      }
      if (this.protocol === 'json') {
        this.moeTcpClient.sendStr({
          message: `${JSON.stringify({ type: 'nes_gamepad', bitmap: this.bitmap })}\n`,
        });
      } else {
        this.moeTcpClient.sendHexStr({
          message: `${this.toHex(lo)}${this.toHex(hi)}`,
        });
      }
    },
    // 通道切换：BLE 键盘模式仅作可选项，默认使用 TCP Server
    toggleTransport(e) {
      const useBle = e.detail.value;
      if (useBle) {
        this.transport = 'ble';
        this.lastSent = -1;
        this.initBle();
      } else {
        if (this.bleDeviceId) {
          this.closeBle();
        }
        this.bleDevices = [];
        this.bleReady = false;
        this.transport = 'tcp';
        this.lastSent = -1;
        this.sendFrame();
      }
    },
    toggleProtocol(e) {
      this.protocol = e.detail.value ? 'json' : 'hex';
      this.lastSent = -1;
      this.sendFrame();
    },
    // ---------- BLE 可选通道 ----------
    initBle() {
      uni.openBluetoothAdapter({
        success: () => this.startBleScan(),
        fail: (err) => {
          console.error('蓝牙不可用', err);
          uni.showToast({ title: '蓝牙不可用', icon: 'none' });
        },
      });
    },
    startBleScan() {
      this.bleDevices = [];
      const devices = [];
      uni.stopBluetoothDevicesDiscovery({ success: () => {} });
      uni.startBluetoothDevicesDiscovery({
        allowDuplicatesKey: false,
        services: [],
        success: () => {
          uni.onBluetoothDeviceFound((res) => {
            (res.devices || []).forEach((device) => {
              const name = device.name || device.localName || '';
              if (!name || !device.deviceId) return;
              if (name.toUpperCase().indexOf(BLE_DEVICE_NAME.toUpperCase()) === -1) return;
              if (devices.some((item) => item.deviceId === device.deviceId)) return;
              devices.push({ deviceId: device.deviceId, name });
              this.bleDevices = devices.slice();
            });
          });
        },
        fail: (err) => console.error('BLE 扫描失败', err),
      });
    },
    stopBleScan() {
      uni.stopBluetoothDevicesDiscovery({ success: () => {} });
    },
    connectBle(deviceId) {
      this.bleDeviceId = deviceId;
      this.stopBleScan();
      uni.showLoading({ title: '连接中...' });
      uni.createBLEConnection({
        deviceId,
        success: () => {
          setTimeout(() => this.discoverBle(), 500);
        },
        fail: (err) => {
          console.error('BLE 连接失败', err);
          uni.hideLoading();
          uni.showToast({ title: '连接失败', icon: 'none' });
        },
      });
    },
    discoverBle() {
      uni.getBLEDeviceServices({
        deviceId: this.bleDeviceId,
        success: ({ services }) => {
          const service =
            services.find((item) => item.uuid.toUpperCase().indexOf(BLE_SERVICE_UUID) !== -1) ||
            services[0];
          if (!service) {
            uni.hideLoading();
            uni.showToast({ title: '未找到服务', icon: 'none' });
            return;
          }
          this.bleServiceId = service.uuid;
          uni.getBLEDeviceCharacteristics({
            deviceId: this.bleDeviceId,
            serviceId: this.bleServiceId,
            success: ({ characteristics }) => {
              uni.hideLoading();
              const writeChar =
                characteristics.find(
                  (item) => item.uuid.toUpperCase().indexOf(BLE_WRITE_CHAR_UUID) !== -1
                ) || characteristics.find((item) => item.properties && item.properties.write);
              if (!writeChar) {
                uni.showToast({ title: '未找到写入特征', icon: 'none' });
                return;
              }
              this.bleWriteCharId = writeChar.uuid;
              this.bleReady = true;
              this.lastSent = -1;
              this.sendFrame();
              uni.showToast({ title: '连接成功', icon: 'success' });
            },
            fail: (err) => {
              uni.hideLoading();
              console.error('获取 BLE 特征失败', err);
            },
          });
        },
        fail: (err) => {
          uni.hideLoading();
          console.error('获取 BLE 服务失败', err);
        },
      });
    },
    writeBle(bytes) {
      if (!this.bleReady || !this.bleDeviceId || !this.bleServiceId || !this.bleWriteCharId) return;
      const buffer = new Uint8Array(bytes).buffer;
      uni.writeBLECharacteristicValue({
        deviceId: this.bleDeviceId,
        serviceId: this.bleServiceId,
        characteristicId: this.bleWriteCharId,
        value: buffer,
        writeType: 'write',
        fail: (err) => console.error('BLE 写入失败', err),
      });
    },
    closeBle() {
      uni.closeBLEConnection({ deviceId: this.bleDeviceId, success: () => {} });
    },
  },
};
</script>

<style scoped>
.gamepad {
  display: flex;
  flex-direction: column;
  padding: 8px 16px 16px;
  background-color: #f0f0f0;
}
.status-bar {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
}
.status-text {
  color: #333;
  font-size: 14px;
}
.status-text.frame {
  font-family: monospace;
  color: #007aff;
}
.switch-item {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 14px;
}
.ble-panel {
  max-height: 140px;
  margin-bottom: 8px;
  padding: 8px 12px;
  overflow: auto;
  border-radius: 10px;
  background: #ffffff;
}
.ble-panel__title {
  display: block;
  margin-bottom: 6px;
  font-weight: 600;
  color: #333;
}
.ble-panel__empty {
  color: #999;
  font-size: 13px;
}
.ble-device {
  display: flex;
  align-items: center;
  justify-content: space-between;
  min-height: 40px;
  padding: 0 8px;
  border-bottom: 1px solid #eee;
}
.ble-device__action {
  color: #09bb07;
  font-size: 13px;
}
.pad-body {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px;
  border-radius: 28px;
  background: #2f3136;
}
.dpad {
  position: relative;
  width: 168px;
  height: 168px;
}
.dpad-btn {
  position: absolute;
  width: 56px;
  height: 56px;
  border-radius: 12px;
  background-color: #5a5f66;
  box-shadow: inset 0 0 0 1px rgba(0, 0, 0, 0.15);
}
.dpad-up {
  top: 0;
  left: 56px;
}
.dpad-down {
  bottom: 0;
  left: 56px;
}
.dpad-left {
  top: 56px;
  left: 0;
}
.dpad-right {
  top: 56px;
  right: 0;
}
.dpad-center {
  position: absolute;
  top: 56px;
  left: 56px;
  width: 56px;
  height: 56px;
  background-color: #5a5f66;
}
.center-btns {
  display: flex;
  align-items: center;
  gap: 14px;
}
.pill-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 74px;
  height: 34px;
  border-radius: 17px;
  background-color: #8a8f96;
  color: #ffffff;
  font-size: 14px;
}
.action-btns {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.action-row {
  display: flex;
  gap: 12px;
}
.round-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 62px;
  height: 62px;
  border-radius: 50%;
  background-color: #d0333b;
  color: #ffffff;
  font-size: 20px;
  font-weight: 600;
}
.btn-active {
  transform: scale(0.92);
  background-color: #ffd24d;
  color: #222222;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.35);
}
</style>
