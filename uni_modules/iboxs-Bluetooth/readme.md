# uni-Bluetooth(安卓经典蓝牙)
### 注意：本插件仅支持Android，请注意相关的条件编译。
### 本插件尝试了多次发布源码版，但是依旧不显示，若有需要的请直接联系作者获取

### 开发文档
* 注意：首次使用请打自定义基座，打自定义基座，打自定义基座。
* 注意：发送数据无需手动分包，函数已做分包处理，按1KB为一个包自动分包发送。
* Android蓝牙不但需要蓝牙权限，还需要位置权限，且开启相关服务，否则无法使用，请注意权限上的处理。
* 更多功能请参考示例项目

### 用到的自定义数据类型
* blueDevice(蓝牙设备)
	* 定义
```
export type blueDevice={
	name?:String,    //设备名称
	address?:String, //地址
	type?:Int,       //类型
	uuids?:Any,      //UUID
	rssi?:Int        //RSSI
}
```

## 函数目录
#### init
* 功能：初始化蓝牙模块
	* 注意：所有函数都需要在init之后调用
* 参数：无
* 返回：布尔型，true为初始化成功，false为初始化失败
	* 初始化失败可能的原因有：无蓝牙模块、无权限、未开启蓝牙、未开启位置服务等。
* 使用示例
```
console.log('蓝牙初始化结果',init()); //输出初始化结果(true为初始化成功)
```

#### checkBluetoothPermissions
* 功能：检查蓝牙权限(包括蓝牙权限、位置权限)
* 使用示例：
```
checkBluetoothPermissions(function(res:Boolean){ //检查权限
	if(res){
		console.log('已获得权限');
	} else{
		console.log('权限获取失败');
	}
})
```
* 权限获取失败的可能有：用户未授权、用户取消或拒绝、相关服务未开启。

#### checkBluetooth
* 功能：检查蓝牙是否已开启
* 使用示例：
```
var res=checkBluetooth(); //返回布尔型
console.log('蓝牙状态',res);
```

#### checkBluetoothStatus
* 功能：检查蓝牙状态，若关闭则请求开启蓝牙
* 使用示例：
```
checkAndEnableBluetooth(function(res:Boolean){ //检查蓝牙是否已开启，未开启会请求开启，并将结果返回
	if(res){
		console.log('蓝牙已开启');
	} else{
		console.log('蓝牙未开启并请求开启失败');
	}
})
```

#### isLocationServiceEnabled
* 功能：检查位置服务是否已开启(Android蓝牙扫描设备需要开启位置服务)
* 使用示例：
```
var res=isLocationServiceEnabled(); //布尔型
console.log('位置服务开启状态',res);
```

#### startDiscovery
* 功能：扫描设备
* 注意：本函数自1.3.0版本起已设置@UTSJS.keepAlive，注意内存回收策略
* 使用示例
```
this.deviceList=[] as blueDevice[]; //存储扫描到的设备列表
let that=this;
var includePaired=true; //是否包括已配对的设备
let status=startDiscovery(includePaired,function(res:blueDevice){
	console.log('扫描到设备',res);
	if(res.name==null){  //排除name为null的设备
		return;
	}
	that.deviceList.push(res);
})
console.log('开始扫描状态',status); //false为开启扫描失败
```

#### stopDiscovery
* 功能：停止扫描设备
* 使用示例：
```
console.log('停止扫描设备',stopDiscovery()); //true为停止成功
```

#### setBluetoothStateCallback
* 功能：设置蓝牙状态监听（蓝牙状态改变时触发）
* 使用示例：
```
setBluetoothStateCallback(function(res:Boolean){
	console.log('监听到蓝牙状态变化,当前状态:',res); //true为蓝牙已开启、false为蓝牙被关闭
})
```

### cancelBluetoothStateCallback（1.3.0版本新增）
* 功能：取消设置蓝牙状态监听
* 使用示例：
```
cancelBluetoothStateCallback();  //无返回值
```

#### checkDevicePaired
* 功能：检查设备是否已配对
* 使用示例：
```
var address="5C:C3:36:02:56:04";
let res=isDevicePaired(address); //布尔型
console.log('设备配对情况',res);
```

#### getPairedDevices
* 功能：获取已配对设备列表
* 使用示例：
```
let list=getPairedDevices();  //blueDevice[]类型
console.log('已配对设备列表',list)
this.deviceList2=list;
```

#### unpairDevice
* 功能：取消配对设备
* 使用示例：
```
var device:blueDevice=this.deviceList2[index];
//参数device数据类型为自定义类型blueDevice
let r=unpairDevice(device)
console.log('取消配对结果',r);
if(r){
	console.log('取消配对成功');
	this.deviceList2.splice(index,1)
}
```

#### paried
* 功能：配对设备
* 使用示例：
```
//参数device数据类型为自定义类型blueDevice
let status=pairDevice(device,function(res:Boolean){
	//配对结果回调，res为true为配对成功，false为配对失败
})
//status为布尔型，代表配对是否发起成功，不代表配对结果
```

#### connectDevice
* 功能：连接设备
* 使用示例：
```
//参数device数据类型为自定义类型blueDevice
connectDevice(device,function(res:Boolean){
	console.log('连接结果',res);
	//res为true为连接成功，false为连接失败
})
```

#### sendData
* 功能：发送数据（字符串方式）
* 使用示例：
```
sendData('hello 老满');
```
* 注意，发送数据函数有自动分包功能，当数据长度大于1KB时，数据会自动分包发送。

#### sendBufferData
* 功能：发送数据（ArrayBuffer方式）
* 使用示例：
```
let base64 = 'H28B8QE=';
let data = uni.base64ToArrayBuffer(base64);
sendBufferData(data);
```

#### setDataReceiveListener
* 功能：设置数据接收监听
* 注意：本函数自1.3.0版本起已设置@UTSJS.keepAlive，注意内存回收策略
* 使用示例：
```
var returnArrayBuffer=true; //接收数据方式，true为ArrayBuffer方式，false为字符串方式，若传入true则回调函数的data类型为ArrayBuffer，false为字符串
setDataReceiveListener(function(data:Any){
	console.log('接收到数据',data);
	if(returnArrayBuffer){
		var base64=uni.arrayBufferToBase64(data as ArrayBuffer);
		console.log('接收到的ArrayBuffer数据转为base64',base64)
	}
},returnArrayBuffer)
```

### stopListeningForData (v1.3.0版本新增)
* 功能：取消设置数据接收监听
* 使用示例：
```
stopListeningForData(); //无返回值
```

#### closeConnection
* 功能：关闭连接
* 使用示例
```
let res=closeConnection();
if(res){
	console.log('断开连接成功')
} else{
	console.log('断开失败');
}
```

#### setReceiveListener
* 功能：设置链接状态监听
* 注意：本函数自1.3.0版本起已设置@UTSJS.keepAlive，注意内存回收策略
* 使用示例
```
//connected:是否连接成功,true为链接成功或正常，false为链接失败或断开
//msg:连接成功或断开时的信息
setReceiveListener(function(connected,msg){
	console.log('链接状态变化为:',connected,'原因',msg);
})
```

#### cancelReceiveListener  (v1.3.0版本新增)
* 功能：设置链接状态监听
* 使用示例
```
cancelReceiveListener();  //无返回值
```