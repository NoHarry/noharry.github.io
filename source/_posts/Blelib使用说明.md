
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
      .setLogEnable(true) //是否打开日志
      .setLogStyle(BleAdmin.LOG_STYLE_DEFAULT) //日志显示风格
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
### 3.读，写，通知等操作
因为Android对BLE设备的读，写等操作需要在上一个任务完成以后才能进行下一个任务，因此以下的任务在创建并加入任务队列后将会按入列的先后顺序一次执行

* 读

```java

```
