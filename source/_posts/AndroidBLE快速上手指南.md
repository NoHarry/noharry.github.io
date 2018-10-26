
---
title: Android BLE 快速上手指南
# toc: true
tags:
- BLE
- bluetooth-low-energy
- Android
date: 2018/10/24 20:30:00
update: 2018/10/24 21:01:00
---

 本文旨在提供一个方便没接触过Android上低功耗蓝牙(Bluetooth Low Energy)的同学快速上手使用的简易教程，因此对其中的一些细节不做过分深入的探讨，此外，为了让没有Ble设备的同学也能模拟与设备的交互过程，本文还提供了中央设备(central)和外围设备(peripheral)的示例代码，只需2部手机大家就可以愉快的“左右互搏”了。

 ## 准备工作
### 角色
  上面我们提到了中央设备(central)和外围设备(peripheral),在这里我们可以这样简单的理解:
* 中央设备(central)：收到外围设备发出的广播信号后能主动发起连接的主设备，例如我们给摩拜单车开锁时我们的手机就是作为中央设备连接单车并进行开锁等一系列操作的，[通常情况](https://www.zhihu.com/question/25120915)下同一时间一台中央设备只能与最多7台外围设备建立连接。
* 外围设备(peripheral):能被中央设备连接的从设备，同一时间外围设备只能被一个中央设备连接。

**注**：Android从4.3(API Level 18) 开始支持低功耗蓝牙，但是刚开始只支持作为中央设备（central）模式，从 Android 5.0(API Level 21) 开始才支持作为外围设备(peripheral)的模式，因此我们最好使用Android 5.0以上版本的手机进行下面的操作。

### 需要的权限

```java
  <uses-permission android:name="android.permission.BLUETOOTH" />
  <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
  <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
  //使用ble扫描时还需要我们到’设置 > 安全性和位置信息 > 位置信息‘处打开位置信息，
  //否则将会搜索不到周围的设备
```
可能有人会问[为什么使用低功耗蓝牙还需要位置权限](https://source.android.google.cn/devices/bluetooth/ble)?简单来说就是蓝牙也有定位的功能。

### 示例代码
* [外围设备](https://github.com/NoHarry/BleServer)
* [中央设备](https://github.com/NoHarry/BleExample)

## 开始
  接下来我们就准备开始实际操作了，首先我们准备2台手机，手机A作为中央设备，手机B作为外围设备,在打开B手机的ble广播后，我们使用A手机进行**打开蓝牙**-->**扫描**-->**连接**-->**获取服务，特征**-->**打开通知**-->**写特征**-->**读特征**-->**断开连接**,通过这些步骤我们就能学会Android Ble 的基本方法的使用。

  > 从扫描开始，接下来的这些操作中你可能会遇到各种奇奇怪怪的问题，为了减少大家踩坑的概率，我会在后面的操作中分享一些可能会遇到的问题和解决方法，有的问题在官方文档中可能有提到，有的在一些论坛帖子中有提及，还有的一些就是自己的经验之谈。


### 打开蓝牙
打开蓝牙有以下两种方式：

```java
    //方法一
    BluetoothManager bluetoothManager= (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
    BluetoothAdapter mBluetoothAdapter = bluetoothManager.getAdapter();
    if (mBluetoothAdapter != null){
      mBluetoothAdapter.enable();
    }
```

```java
    //方法二
    BluetoothManager bluetoothManager= (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
    BluetoothAdapter mBluetoothAdapter = bluetoothManager.getAdapter();
    if (!mBluetoothAdapter.isEnabled() && !mBluetoothAdapter.isEnabled()) {
      Intent enableBtIntent = new Intent(
          BluetoothAdapter.ACTION_REQUEST_ENABLE);
      startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT);
    }
```
- 使用方法一将会直接打开蓝牙，使用方法二会跳转到系统Activity由用户手动打开蓝牙


### 扫描
扫描是一个非常耗电的操作，因此当我们找到我们需要的设备后应该马上停止扫描。官方提供了2个扫描的方法：

```java
  //旧API
  //启动扫描
  private void scan(){
    BluetoothManager bluetoothManager= (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
    bluetoothManager.getAdapter().startLeScan(mLeScanCallback);

    //如果想要指定搜索设备，可以使用下面这个构造方法，传入外围设备广播出的服务的UUID数组
    UUID[] uuids=new UUID[]{UUID_ADV_SERVER};
    bluetoothManager.getAdapter().startLeScan(uuids,mLeScanCallback);
  }

  //停止扫描
  private void stopScan(){
    BluetoothManager bluetoothManager= (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
    bluetoothManager.getAdapter().stopLeScan(mLeScanCallback);
  }

  //扫描结果回调
  LeScanCallback mLeScanCallback = new LeScanCallback() {
      @Override
      public void onLeScan(BluetoothDevice device, int rssi, byte[] scanRecord) {
        //device:扫描到的蓝牙设备对象
        //rssi：扫描到的设备的信号强度，这是一个负值，值越大代表信号强度越大
        //scanRecord:扫描到的设备广播的数据,包含设备名，服务UUID等
      }
    };
```
↑ 这是个在Android 5.0时被标注deprecated的API，该方法目前仍能使用。由于onLeScan中回调出的设备的广播数据需要自己手动解析,这是个比较麻烦的过程。

![advData](advData.png)

在新的API中已经封装了方法来解析广播数据，如果为了适配性使用这个旧的扫描方法，同时又希望解析得到广播中的数据，我们可以使用源码中[新API使用的解析方法](http://androidxref.com/8.0.0_r4/xref/frameworks/base/core/java/android/bluetooth/le/ScanRecord.java)(需要稍许修改，直接使用会报错),或者使用[我自己修改过的方法](https://github.com/NoHarry/BLELib/blob/master/blelib/src/main/java/cc/noharry/blelib/data/ScanRecord.java),如果你想了解更多关于广播数据的解析可以看[Core Specifications 5.0](https://www.bluetooth.com/specifications/bluetooth-core-specification)中Volume 3, Part C, Section 11这一节。

```java
  //新API,需要Android 5.0(API Level 21)及以上版本才能使用
  //启动扫描
  private void scanNew() {
    BluetoothManager bluetoothManager= (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
    //基本的扫描方法
    bluetoothManager
        .getAdapter()
        .getBluetoothLeScanner()
        .startScan(mScanCallback);


    //设置一些扫描参数
    ScanSettings settings=new ScanSettings
        .Builder()
        //例如这里设置的低延迟模式，也就是更快的扫描到周围设备，相应耗电也更厉害
        .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
        .build();

    //你需要设置的过滤条件,不只可以像旧API中的按服务UUID过滤
    //还可以按设备名称，MAC地址等条件过滤
    List<ScanFilter> scanFilters=new ArrayList<>();

    //如果你需要过滤扫描到的设备可以用下面的这种构造方法
    bluetoothManager
        .getAdapter()
        .getBluetoothLeScanner()
        .startScan(scanFilters,settings,mScanCallback);
  }

  //扫描结果回调
  ScanCallback mScanCallback = new ScanCallback() {
     @Override
     public void onScanResult(int callbackType, ScanResult result) {
        //callbackType:扫描模式
        //result：扫描到的设备数据，包含蓝牙设备对象，解析完成的广播数据等
     }
   };

  //停止扫描
  private void stopNewScan(){
    BluetoothManager bluetoothManager= (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
    bluetoothManager.getAdapter().getBluetoothLeScanner().stopScan(mScanCallback);
  }
```
  相比旧API,新API的功能更全面，但是需要Android 5.0以上才能使用，究竟需要使用哪种方法，大家可以根据自己的实际情况选择。


  > **注意坑来了：**

  - 1.如果搜索不到设备，请检查对于Android 6.0及以上版本ACCESS_COARSE_LOCATION或者ACCESS_FINE_LOCATION权限是否已经动态授予，同时检查位置信息(也就是GPS)是否已经打开，一般来说搜不到设备就是这两个原因。

  - 2.不管是新旧API的扫描结果回调都是不停的回调扫描到的设备，就算是相同的设备也会重复回调,直到你停止扫描，因此最好不要在回调方法中做过多的耗时操作,否则可能会出现[这个问题](https://issuetracker.google.com/issues/36989120)，如果需要处理回调的数据可以把数据放到另外一个线程处理，让回调尽快返回。

### 连接
