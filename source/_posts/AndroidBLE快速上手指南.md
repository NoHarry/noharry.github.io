
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
  接下来我们就准备开始实际操作了，首先我们准备2台手机，手机A作为中央设备，手机B作为外围设备
