
title: Blelib 使用说明
toc: true
tags: BLE
date: 2018/6/28 20:30:00
update: 2018/4/5 21:01:00

---
# Blelib
---
[![Build Status](https://travis-ci.org/NoHarry/BLELib.svg?branch=dev)](https://travis-ci.org/NoHarry/BLELib)
[ ![Download](https://api.bintray.com/packages/l2011louhanyu/maven/BleLib/images/download.svg) ](https://bintray.com/l2011louhanyu/maven/BleLib/_latestVersion)
<a href="https://play.google.com/store/apps/details?id=cc.noharry.bledemo">
  <img alt="Android app on Google Play"
       src="https://developer.android.com/images/brand/en_app_rgb_wo_45.png" />
</a>

Blelib是一个Android端的低功耗蓝牙(Bluetooth Low Energy)库，该库包含许多与BLE外围设备进行
交互时需要用到的功能。

## 功能
* 支持与外围设备的扫描、连接、读、写、通知等基本功能
* 支持自定义扫描
* 支持自定义连接超时时间
* 支持指定符号速率PHY(Physical Layer)的连接(PHY_LE_1M_MASK,PHY_LE_2M_MASK,PHY_LE_CODED_MASK)
* 支持设置连接优先级
* 支持设置最大传输单元(MTU)
* 支持日志开关

## 配置
### Gradle

```
// 在你项目的Project级别下的build.gradle文件中添加
repositories {
  jcenter()
}

// 在你项目的Module级别下的build.gradle文件中添加下面的依赖
dependencies {
    implementation 'cc.noharry.blelib:blelib:0.0.5'
}
```

### Maven
```
<dependency>
  <groupId>cc.noharry.blelib</groupId>
  <artifactId>blelib</artifactId>
  <version>0.0.5</version>
  <type>pom</type>
</dependency>
```
### 权限

```
<uses-permission android:name="android.permission.BLUETOOTH"/>
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
```
第三个权限在Android API 23 及以上的版本需要[在运行时请求权限](https://developer.android.com/training/permissions/requesting?hl=zh-cn)

**注意**：该库需要java1.8版本的支持

```
// 在你项目的Module级别下的build.gradle文件中配置下面的属性
android {
compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
  }
```


## 使用


### 1.扫描
* 设定扫描规则

|方法|描述|
| ---                                               | ---                                                                            |
| setUUID(UUID[] uuid)| 过滤出广播中含有传入的UUID的设备|
| setDeviceName(String[] deviceName,boolean fuzzzy) | 过滤出传入设备名的设备，第二个参数true表示只要设备名中包含传入的名称都进行过滤 |
|setDeviceMac(String[] deviceMac)|过滤出含有传入MAC地址的设备,MAC格式例如：11:22:33:44:55:66|
|setScanTime(long scanTime)|设定扫描时长,单位:毫秒；不调用该方法或传入0,会进入持续扫描模式|
```java
BleScanConfig scanConfig = new Builder()
       .setDeviceName(new String[]{Name}, isFuzzy) //过滤的设备名和是否模糊搜索
       .setDeviceMac(new String[]{Mac}) //过滤的设备的Mac地址
       .setUUID(new UUID[]{UUID.fromString(uuid)}) //过滤的设备的广播UUID
       .setScanTime(scanTime) //扫描时长，不设定该选项则是一直扫描，单位：毫秒
       .build();
```
* 开始扫描

```java
//扫描结果回调
BleScanCallback mBleScanCallback = new BleScanCallback() {
      @Override
      public void onScanStarted(boolean isStartSuccess) {
        //开始扫描
      }

      @Override
      public void onFoundDevice(BleDevice bleDevice) {
        //发现设备，这个回调会在扫描过程中不停回调出发现的设备，其中会出现重复的设备
        //因此，使用者需要根据自身的使用情况进行过滤
      }

      @Override
      public void onScanCompleted(List<BleDevice> deviceList) {
        //扫描完成
        //会在扫描时间结束或者主动调用stopScan()时发生回调
        //回调出的设备为该过程中扫描到的所有设备，已过滤重复设备
      }
    };
BleAdmin
      .getINSTANCE(getApplication())
      .scan(scanConfig, mBleScanCallback);

```
* 停止扫描

```java
BleAdmin.getINSTANCE(getApplication()).stopScan();
```
### 2.连接

* 连接回调

```java
BleConnectCallback mBleConnectCallback = new BleConnectCallback() {
      @Override
      public void onDeviceConnecting(BleDevice bleDevice) {
          //开始连接
      }

      @Override
      public void onDeviceConnected(BleDevice bleDevice) {
        //连接成功
      }

      @Override
      public void onServicesDiscovered(BleDevice bleDevice
      , BluetoothGatt gatt, int status) {
        //发现服务
      }

      @Override
      public void onDeviceDisconnecting(BleDevice bleDevice) {
        //正在断开连接
      }

      @Override
      public void onDeviceDisconnected(BleDevice bleDevice, int status) {
        //断开连接
      }
    };
```
* 开始连接

|参数|是否必填|描述|
| --- |---| ---   |
| BleDevice  |是| 需要连接的设备 |
| isAutoConnect |否| 是否使用自动连接,true：使用自动连接，连接慢；false：立即连接 |
|BaseBleConnectCallback|是|如果想全面的自己处理连接后的回调可使用:BaseBleConnectCallback;如果只想简单的使用就用：BleConnectCallback|
|preferredPhy|否|使用指定符号速率PHY(Physical Layer)的连接(PHY_LE_1M_MASK,PHY_LE_2M_MASK,PHY_LE_CODED_MASK)
|timeOut|否|连接的超时时间，单位:毫秒|


```java
BleAdmin
          .getINSTANCE(getApplication())
          .connect(bleDevice               //需要连接的设备
              , false                      //是否自动连接，如果想立即连接传入false
              , mBleConnectCallback        //连接结果回调
              ,timeOut);                   //超时时间，单位:毫秒
```
* 断开连接

```java
BleAdmin
        .getINSTANCE(getApplication())
        .disconnect(bleDevice);
```
### 3.读、写、通知等操作
因为Android对BLE设备的读，写等操作需要在上一个任务完成以后才能进行下一个任务，因此以下的任务在创建并加入任务队列后将会按入列的先后顺序依次执行

* 读
    - 创建读任务

  ```java
  ReadTask task = Task.newReadTask(bleDevice
  , characteristic)                       //读取的特征
      .with(mReadCallback);               //传入回调
  ```
  |参数|是否必须|描述|
  |---|---|---|
  |bleDevice|是|读任务执行的设备|
  |bluetoothGattCharacteristic|是|需要读的特征|

  * 加入任务队列

  ```java
    BleAdmin.getINSTANCE(getApplication()).addTask(task);
  ```

  * 结果回调

  ```java
  ReadCallback mReadCallback = new ReadCallback() {
        @Override
        public void onDataRecived(BleDevice bleDevice, Data data) {
          //读到的数据
        }

        @Override
        public void onOperationSuccess(BleDevice bleDevice) {
          //操作成功
        }

        @Override
        public void onFail(BleDevice bleDevice, int statuCode, String message) {
          //失败回调
        }

        @Override
        public void onComplete(BleDevice bleDevice) {
          //完成回调
        }
      };
  ```



* 写

```java
//结果回调
WriteCallback mWriteCallback = new WriteCallback() {
      @Override
      public void onDataSent(BleDevice bleDevice, Data data, int totalPackSize,
          int remainPackSize) {

      }

      @Override
      public void onOperationSuccess(BleDevice bleDevice) {

      }

      @Override
      public void onFail(BleDevice bleDevice, int statuCode, String message) {

      }

      @Override
      public void onComplete(BleDevice bleDevice) {

      }
    };

    WriteTask task = Task.newWriteTask(bleDevice, characteristic, writeData)
        .with(mWriteCallback);
    BleAdmin.getINSTANCE(getApplication()).addTask(task);
```
