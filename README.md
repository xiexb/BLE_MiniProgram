## 原帖地址
https://www.cnblogs.com/HintLee/p/9499423.html

## API简介
微信小程序目前有蓝牙 API 共 18 个

### 操作蓝牙适配器的共有 4 个，分别是
wx.openBluetoothAdapter 初始化蓝牙适配器
wx.closeBluetoothAdapter 关闭蓝牙模块
wx.getBluetoothAdapterState 获取本机蓝牙适配器状态
wx.onBluetoothAdapterStateChange 监听蓝牙适配器状态变化事件

### 连接前使用的共有 4 个，分别是
wx.startBluetoothDevicesDiscovery 开始搜寻附近的蓝牙外围设备
wx.stopBluetoothDevicesDiscovery 停止搜寻附近的蓝牙外围设备
wx.getBluetoothDevices 获取所有已发现的蓝牙设备
wx.onBluetoothDeviceFound 监听寻找到新设备的事件

### 连接和断开时使用的共有 2 个，分别是
wx.createBLEConnection 连接低功耗蓝牙设备
wx.closeBLEConnection 断开与低功耗蓝牙设备的连接

### 连接成功后使用的共有 8 个，分别是
wx.getConnectedBluetoothDevices 根据 uuid 获取处于已连接状态的设备
wx.getBLEDeviceServices 获取蓝牙设备所有 service（服务）
wx.getBLEDeviceCharacteristics 获取蓝牙设备所有 characteristic（特征值）
wx.readBLECharacteristicValue 读取低功耗蓝牙设备的特征值的二进制数据值
wx.writeBLECharacteristicValue 向低功耗蓝牙设备特征值中写入二进制数据
wx.notifyBLECharacteristicValueChange 启用低功耗蓝牙设备特征值变化时的 notify 功能
wx.onBLECharacteristicValueChange 监听低功耗蓝牙设备的特征值变化
wx.onBLEConnectionStateChange 监听低功耗蓝牙连接的错误事件

## 基本操作流程
最基本的操作流程是：初始化蓝牙适配器→开始搜寻附近的蓝牙外围设备→监听寻找到新设备的事件→连接低功耗蓝牙设备→获取蓝牙设备所有 service 和 characteristic →读取或写入低功耗蓝牙设备的特征值的二进制数据值。

## 踩过的几个坑

### 支持蓝牙 API 的版本
Android从微信 6.5.7 开始支持，iOS从微信 6.5.6 开始支持，因此小程序中需要做好版本检测，在 app.js 文件中加入以下代码，其中 wx.getSystemInfoSync 是一个获取系统信息的API。
```
onLaunch: function() {
    this.globalData.sysinfo = wx.getSystemInfoSync()
},
getModel: function () { //获取手机型号
    return this.globalData.sysinfo["model"]
},
getVersion: function () { //获取微信版本号
    return this.globalData.sysinfo["version"]
},
getSystem: function () { //获取操作系统版本
    return this.globalData.sysinfo["system"]
},
getPlatform: function () { //获取客户端平台
    return this.globalData.sysinfo["platform"]
},
getSDKVersion: function () { //获取客户端基础库版本
    return this.globalData.sysinfo["SDKVersion"]
}
```

在初始页面（一般是 index.wxml）对应的 js 文件中使用 app.getPlatform() 和 app.getVersion() 即可获取到客户端平台（安卓或 iOS）和微信版本号。在onLoad中获取这两个信息后进行比较即可，使用了下面的版本比较方法。
```
versionCompare: function (ver1, ver2) { //版本比较
    var version1pre = parseFloat(ver1)
    var version2pre = parseFloat(ver2)
    var version1next = parseInt(ver1.replace(version1pre + ".", ""))
    var version2next = parseInt(ver2.replace(version2pre + ".", ""))
    if (version1pre > version2pre)
        return true
    else if (version1pre < version2pre) 
        return false
    else {
        if (version1next > version2next)
            return true
        else
            return false
    }
}
if (app.getPlatform() == 'android' && this.versionCompare('6.5.7', app.getVersion())) {
    wx.showModal({
        title: '提示',
        content: '当前微信版本过低，请更新至最新版本',
        showCancel: false
    })
}
else if (app.getPlatform() == 'ios' && this.versionCompare('6.5.6', app.getVersion())) {
    wx.showModal({
        title: '提示',
        content: '当前微信版本过低，请更新至最新版本',
        showCancel: false
    })
}
```
### 安卓 6.0 及以上设备需打开定位服务
在测试中发现安卓 6.0 以上的手机未打开系统定位服务时，搜索不到蓝牙设备，因此最好在页面中提示用户打开定位服务。

### wx.onBluetoothDeviceFound 不兼容
安卓及iOS设备使用 wx.onBluetoothDeviceFound 时会出现不同的返回值，且有概率出现重复设备，所以使用以下代码可以清除重复的设备和解决 API 不兼容问题。
```
wx.onBluetoothDeviceFound(function (devices) {
    var isnotExist = true
    if (devices.deviceId) {
        for (var i = 0; i < foundDevice.length; i ++) {
            if (devices.deviceId == foundDevice[i].deviceId) {
                isnotExist = false
            }
        }
        if (isnotexist)
            foundDevice.push(devices)
    }
    else if (devices.devices) {
        for (var i = 0; i < foundDevice.length; i++) {
            if (devices.devices[0].deviceId == foundDevice[i].deviceId) {
                isnotExist = false
            }
        }
        if (isnotexist)
            foundDevice.push(devices.devices[0])
    }
    else if (devices[0]) {
        for (var i = 0; i < foundDevice.length; i++) {
            if (devices[0].deviceId == foundDevice[i].deviceId) {
                isnotExist = false
            }
        }
        if (isnotexist)
            foundDevice.push(devices[0])
    }
})
```
### 读取广播数据和特征值
小程序中读取 BLE 广播数据使用 wx.onBluetoothDeviceFound 接口中的 advertisData，对应上面兼容问题的 devices 格式，如 devices.advertisData，这个数据是 ArrayBuffer，需要转换，可以使用以下两种转换方法。另外 wx.getBLEDeviceCharacteristics 读取的特征值 characteristic.value 也是 ArrayBuffer，用同样的方法转换。
```
buf2string: function (buffer) {
    var arr = Array.prototype.map.call(new Uint8Array(buffer), x => x)
    var str = ''
    for (var i = 0; i < arr.length; i++) {
      str += String.fromCharCode(arr[i])
    }
    return str
}
buf2hex: function (buffer) {
    return Array.prototype.map.call(new Uint8Array(buffer), x => ('00' + x.toString(16)).slice(-2)).join('');
}
```
### 发送大于 20 字节的数据包
众所周知，BLE 4.0 中发送一个数据包只能包含 20 字节的数据，大于 20 字节只能分包发送。微信小程序提供的 API 中似乎没有自动分包的功能，这就只能自己手动分包了。调试中发现，在 iOS 系统中调用 wx.writeBLECharacteristicValue 发送数据包，回调 success 后紧接着发送下一个数据包，很少出现问题，可以很快全部发送完毕。而安卓系统中，发送一个数据包成功后紧接着发送下一个，很大概率会出现发送失败的情况，在中间稍做延时再发送下一个就可以解决这个问题（不同安卓手机的时间长短也不一致），照顾下一些比较奇葩的手机，大概需要延时 250 ms 。不太好的但是比较科学的办法是，只要成功发送一个数据包则发送下一个，否则不断重发，具体就是
wx.writeBLECharacteristicValue 回调 fail 则重新发送，直至发送完毕。

补充说明
此处补充说明一下，华为荣耀部分机型、还有蓝绿厂的部分机型，在蓝牙 API 有深坑，谨慎调试。另：发现挺多同学没有注意到官方文档最下方的错误码列表，顺便在此处贴出来。

## 蓝牙错误码 (errCode) 列表
错误码	说明	备注
0	ok	正常
10000	not init	未初始化蓝牙适配器
10001	not available	当前蓝牙适配器不可用
10002	no device	没有找到指定设备
10003	connection fail	连接失败
10004	no service	没有找到指定服务
10005	no characteristic	没有找到指定特征值
10006	no connection	当前连接已断开
10007	property not support	当前特征值不支持此操作
10008	system error	其余所有系统上报的异常
10009	system not support	Android 系统特有，系统版本低于 4.3 不支持BLE
## 注意
search 页面基本不需改动可直接使用，device 页面中固定了 services 和 characteristics，发送和接收都是字符串，根据需求修改即可。
