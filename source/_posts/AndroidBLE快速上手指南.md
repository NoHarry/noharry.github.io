---
title: Android BLE 快速上手指南
tags:
- BLE
- bluetooth-low-energy
- Android
header-img: "bg.jpg"
date: 2018/10/24 20:30:00
update: 2018/10/29 21:01:00
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
同一时间我们只能对一个外围设备发起连接，如果需要对多个设备连接可以等上一个连接成功后再进行下一个连接，否则如果前面的某个连接操作失败了没有回调，后面的操作会被一直阻塞。

```java
  //发起连接
  private void connect(BluetoothDevice device){
    mBluetoothGatt = device.connectGatt(context, false, mBluetoothGattCallback);
  }

  //Gatt操作回调，此回调很重要，后面所有的操作结果都会在此方法中回调
  BluetoothGattCallback mBluetoothGattCallback = new BluetoothGattCallback() {
     @Override
     public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
       //gatt:GATT客户端
       //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
       //newState:当前连接处于的状态，例如连接成功，断开连接等

       //当连接状态改变时触发此回调
     }

     @Override
     public void onServicesDiscovered(BluetoothGatt gatt, int status) {
       //gatt:GATT客户端
       //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常

       //成功获取服务时触发此回调，“获取服务，特征”一节会介绍
     }

     @Override
     public void onCharacteristicRead(BluetoothGatt gatt,
         final BluetoothGattCharacteristic characteristic, final int status) {
           //gatt:GATT客户端
           //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
           //characteristic:被读的特征

           //当对特征的读操作完成时触发此回调，“读特征”一节会介绍
     }

     @Override
     public void onCharacteristicWrite(BluetoothGatt gatt,
         final BluetoothGattCharacteristic characteristic, final int status) {
           //gatt:GATT客户端
           //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
           //characteristic:被写的特征

           //当对特征的写操作完成时触发此回调，“写特征”一节会介绍
     }

     @Override
     public void onCharacteristicChanged(BluetoothGatt gatt,
         final BluetoothGattCharacteristic characteristic) {
           //gatt:GATT客户端
           //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
           //characteristic：特征值改变的特征

           //当特征值改变时触发此回调，“打开通知”一节会介绍
     }

     @Override
     public void onDescriptorRead(BluetoothGatt gatt, BluetoothGattDescriptor descriptor,
         int status) {
           //gatt:GATT客户端
           //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
           //descriptor:被读的descriptor

           //当对descriptor的读操作完成时触发
     }

     @Override
     public void onDescriptorWrite(BluetoothGatt gatt, BluetoothGattDescriptor descriptor,
         int status) {
           //gatt:GATT客户端
           //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
           //descriptor:被写的descriptor

           //当对descriptor的写操作完成时触发，“打开通知”一节会介绍
     }
   };

```
当我们调用connectGatt方法后会触发onConnectionStateChange这个回调，回调中的status我们用来判断这次操作的成功与否，newState用来判断当前的连接状态。

> **注意坑来了：**

* 我们在调用**连接**和**断开连接**这两方法的时候最好放到**主线程**调用，否则可能会在一些手机上遇到[奇怪的问题](https://stackoverflow.com/questions/20069507/gatt-callback-fails-to-register/20507449#20507449)

### 获取服务，特征
当我们连接成功后，GATT客户端(手机A)可以通过发现方法检索GATT服务端(手机B)的服务和特征，以便后面操作使用。

![Stack_GATT_Profile_Structure](Stack_GATT_Profile_Structure.png)

```java
  //连接成功后掉用发现服务
  gatt.discoverServices();

      //当服务检索完成后会回调该方法,检索完成后我们就可以拿到需要的服务和特征
      @Override
      public void onServicesDiscovered(BluetoothGatt gatt, int status) {

        //获取特定UUID的服务
        BluetoothGattService service = gatt.getService(UUID_SERVER);

        //获取所有服务
        List<BluetoothGattService> services = gatt.getServices();

        if (service!=null){

          //获取该服务下特定UUID的特征
          mCharacteristic = service.getCharacteristic(UUID_CHARWRITE);

          //获取该服务下所有特征
          List<BluetoothGattCharacteristic> characteristics = service.getCharacteristics();

        }
      }
```

### 打开通知
打开通知[官方的标准做法](https://developer.android.com/guide/topics/connectivity/bluetooth-le#notification)分两步:
```java
//官方文档做法
private BluetoothGatt mBluetoothGatt;
BluetoothGattCharacteristic characteristic;
boolean enabled;
...
//第一步，开启手机A(本地)对这个特征的通知
mBluetoothGatt.setCharacteristicNotification(characteristic, enabled);
...
//第二步，通过对手机B(远程)中需要开启通知的那个特征的CCCD写入开启通知命令，来打开通知
BluetoothGattDescriptor descriptor = characteristic.getDescriptor(
        UUID.fromString(SampleGattAttributes.CLIENT_CHARACTERISTIC_CONFIG));
descriptor.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);
mBluetoothGatt.writeDescriptor(descriptor);
```
> 由于Android7.0以前版本存在一个bug:对descriptor的写操作会复用父特征的写入类型，这个bug在7.0之后进行了[修复](https://android.googlesource.com/platform/frameworks/base/+/942aebc95924ab1e7ea1e92aaf4e7fc45f695a6c%5E%21/),为了提高兼容性，我们可以对官方做法稍许修改：

```java
private BluetoothGatt mBluetoothGatt;
BluetoothGattCharacteristic characteristic;
boolean enabled;
...
//第一步，开启手机A(本地)对这个特征的通知
mBluetoothGatt.setCharacteristicNotification(characteristic, enabled);
...
//第二步，通过对手机B(远程)中需要开启通知的那个特征的CCCD写入开启通知命令，来打开通知
BluetoothGattDescriptor descriptor = characteristic.getDescriptor(
        UUID.fromString(SampleGattAttributes.CLIENT_CHARACTERISTIC_CONFIG));
//获取特征的写入类型，用于后面还原
int parentWriteType = characteristic.getWriteType();
//设置特征的写入类型为默认类型
characteristic.setWriteType(BluetoothGattCharacteristic.WRITE_TYPE_DEFAULT);
descriptor.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);
mBluetoothGatt.writeDescriptor(descriptor);
//还原特征的写入类型
characteristic.setWriteType(parentWriteType);
```
接下来我们来看看回调

```java
      @Override
      public void onCharacteristicChanged(BluetoothGatt gatt,
          final BluetoothGattCharacteristic characteristic) {
          //当手机B的通知发过来的时候会触发这个回调
      }

      @Override
      public void onDescriptorWrite(BluetoothGatt gatt, BluetoothGattDescriptor descriptor,
          int status) {
        //第二步会触发此回调
      }
```

> **注意:**

* 对于有的设备可能我们只需要执行第一步就能收到通知，但是为了保险起见我们最好两步都做，以防出现[通知开启无效](https://stackoverflow.com/questions/22817005/why-does-setcharacteristicnotification-not-actually-enable-notifications)的情况。
* 再次**强调**读、写、通知等这些GATT的操作都只能串行的使用，并且在执行下一个任务前必须保证上一个任务已经完成并且成功回调，否则可能出现后面的任务都阻塞无法进行的情况。
* 对于开启通知这个操作触发**onDescriptorWrite**时代表任务完成，可以进行下一个GATT操作。

### 写特征

```java
//默认的写入类型，需要外围设备响应
mCharacteristic.setWriteType(BluetoothGattCharacteristic.WRITE_TYPE_DEFAULT);
//无需设备响应的写入类型
mCharacteristic.setWriteType(BluetoothGattCharacteristic.WRITE_TYPE_NO_RESPONSE);

mCharacteristic.setValue(data);
mBluetoothGatt.writeCharacteristic(mCharacteristic);


      //写入特征回调
      @Override
      public void onCharacteristicWrite(BluetoothGatt gatt,
          final BluetoothGattCharacteristic characteristic, final int status) {

      }

```
写特征的用法和前面打开通知中的写descriptor类似。

> **注意:**

* 上面提到了2种写入类型，他们的区别是：
  * WRITE_TYPE_DEFAULT:写入数据后需要外围设备给出响应才会回调onCharacteristicWrite
  * WRITE_TYPE_NO_RESPONSE:写入数据后无需外围设备给出响应就会回调onCharacteristicWrite
> 如果使用WRITE_TYPE_DEFAULT这种类型写入，而外围设备没有回应，那后面的操作都会被阻塞。因此，使用哪种方式需要大家根据自己的外围设备决定,大家可以尝试把示例工程中的[这一行](https://github.com/NoHarry/BleServer/blob/master/app/src/main/java/cc/noharry/bleserver/ble/BLEAdmin.java#L323)注释掉然后在来写入数据，结合日志看看会能更好的理解。

* 一次写入最多能写入20字节的数据，如果需要写入更多的数据可以分包多次写入，或者如果设备支持[更改MTU](https://developer.android.com/reference/android/bluetooth/BluetoothGatt#requestMtu)的话一次最多可以传输512字节。

### 读特征

```java
//读特征
mBluetoothGatt.readCharacteristic(mCharacteristic);

//读特征的回调
@Override
public void onCharacteristicRead(BluetoothGatt gatt,
          final BluetoothGattCharacteristic characteristic, final int status) {

}
```
读特征这个操作没多少坑，只是需要前面提到的成功回调以后才算执行完成

### 断开连接

```java
private void disConnect(){
    if (mBluetoothGatt!=null){
      //断开连接
      mBluetoothGatt.disconnect();
      // mBluetoothGatt.close();
    }
  }

@Override
public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
    if (newState==BluetoothProfile.STATE_DISCONNECTED){
        //关闭GATT客户端
        gatt.close();
      }
}
```
> **注意:**

* **断开连接**和**连接**一样最好都在**主线程**执行
* BluetoothGatt.disConnect()方法和BluetoothGatt.close()方法要成对配合使用，有一点需要注意：如果调用disConnect()方法后立即调用close()方法(就像上面注释掉的代码那样)蓝牙能正常断开，只是在onConnectionStateChange中我们就收不到newState为BluetoothProfile.STATE_DISCONNECTED的状态回调，因此，可以在收到断开连接的回调后在关闭GATT客户端。
* 如果断开连接后没调用close方法，在多次重复连接-断开之后可能你就再也连不上设备了。

## 总结
其实这篇文章除了给大家列举了一些使用的API和可能遇到的问题外，最主要是要强调一个蓝牙操作的节奏，也就是一个任务完成下一个任务才能开始的原则，为了便于大家入门，上面这些使用简化了很多需要考虑的逻辑，例如：读、写、通知一直没回调怎么办？(可以给这些操作都加上超时时间)等等，不过如果大家按照本文提供的方法使用就已经能避开很多可能会遇到的奇怪问题了。

如果大家需要了解更多更详细的使用方法，这里给大家推荐2个开源的ble库：

* [Android-BLE-Library](https://github.com/NordicSemiconductor/Android-BLE-Library):NordicSemiconductor官方的Android ble库。
* [BLELib](https://github.com/NoHarry/BLELib):我自己封装的ble库,大家喜欢的话可以顺手star一下。
