&emsp;&emsp;本文描述了Android系统扫描连接BLE设备(蓝牙透传模块)并接收和发送数据的相关知识。

# 约定

&emsp;&emsp;以下代码中，未定义的变量为类变量。

# 注意事项

1. BluetoothAdapter.startLeScan已经弃用，替换为BluetoothLeScanner.startScan, BluetoothLeScanner实例通过BluetoothAdapter.getBluetoothLeScanner获取

# 相关Java类

| 类名                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| BluetoothManager                                             | 用于获取BluetoothAdapter实例                                 |
| BluetoothAdapter                                             | 用于获取BluetoothLeScanner实例                               |
| BluetoothLeScanner                                           | 用于扫描BLE设备                                              |
| ScannerCallback                                              | 扫描时候的回调函数                                           |
| [ScanSettings](https://developer.android.google.cn/reference/android/bluetooth/le/ScanSettings.html?hl=en) | 保存ble设备扫描相关设置                                      |
| [ScanSettings.Builder](https://developer.android.com/reference/android/bluetooth/le/ScanSettings.Builder#setReportDelay(long)) | 对ble设备扫描的相关属性进行设置并生成ScanSettings对象        |
| ScanResult                                                   | 包含扫描结果的类，用于获取BluetoothDevice实例                |
| BluetoothDevice                                              | 用于建立连接得到BluetoothGatt实例                            |
| BluetoothGatt                                                | 管理同BLE设备建立的一个连接                                  |
| BluetoothGattCallback                                        | 该连接发生变化时的回调函数                                   |
| BluetoothGattService                                         | service包含多个characteristic，代表一组功能，通过其获得characteristic |
| BluetoothGattCharacteristic                                  | characteristic表示一个功能，包含一个或多个Desciptor，通过对characteristic的操作可以发送或获取数据 |
| BluetoothGattDescriptor                                      | 对characteristic的功能进行调整                               |

# 权限

```bash
 <!-- 允许连接已配对过的设备-->
 <uses-permission android:name="android.permission.BLUETOOTH" /> 
 
 <!-- 允许扫描和连接新的设备-->
 <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" /> 
 
 <!-- 因为扫描蓝牙会暴露位置信息， 所以需要申请位置权限才能扫描蓝牙-->
 <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
 <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
 
 <!-- 需要BLE支持-->
 <uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/> 
 
 
```

# 扫描和连接

## 启动蓝牙并获取BluetoothAdapter

```java
private void bluetoothSetup() {
    // 首先获取系统的BluetoothManager
    BluetoothManager mBluetoothManager = (BluetoothManager)getSystemService(Context.BLUETOOTH_SERVICE);

    // 通过BluetoothManager获取BluetoothAdapter
    BluetoothAdapter mBluetoothAdapter = mBluetoothManager.getAdapter();

    // 检查蓝牙在设备上可用并且已启用, 如果没有启用，显示一个对话框，请求用户打开蓝牙, 并在Activity的回调函数
    // onActivityResult中继续完成扫描
    if (mBluetoothAdapter == null || !mBluetoothAdapter.isEnabled()) {
        Intent enableBtIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
        // REQUEST_ENABLE_BT为自定义的一个整数，会传递给回调函数onActivtyResult用以区分是哪一个			// Activity返回了
        startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT);
    }
}
```

## onActivityResult回调函数处理

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    Log.d(TAG, "onActivityResult: activity return, request code is " + requestCode);
    Log.d(TAG, "onActivityResult: activity return, result code is " + resultCode);
    if (requestCode == REQUEST_ENABLE_BT) {
        switch(resultCode) {
            // resultcode的值是测试得到的, 打开蓝牙的对话框在拒绝后返回0，成功开启时返回-1
            case 0:
                Toast.makeText(this, "您拒绝了开启蓝牙的请求， 请在设置中自行打开蓝牙", Toast.LENGTH_SHORT).show();
                break;
            case -1:
                // 用户选择开启蓝牙后重新获取BluetoothAdapter
                mBluetoothAdapter = mBluetoothManager.getAdapter();
                if (mBluetoothAdapter == null || !mBluetoothAdapter.isEnabled()) {
                    Toast.makeText(this, "蓝牙启动失败", Toast.LENGTH_SHORT).show();
                    return；
                }
                Toast.makeText(this, "蓝牙已开启", Toast.LENGTH_SHORT).show();
                }
                break;
            default:
                Toast.makeText(this, "未知结果", Toast.LENGTH_SHORT).show();
                break;
        }
    }
}
```

## 扫描设备

```java
public boolean scanBleDevice(boolean enable) {
		// 检查蓝牙是否开启
        if (mBluetoothAdapter == null || !mBluetoothAdapter.isEnabled()) {
            Log.d(TAG, "scanBleDevice: bluetooth not enabled");
            return false;
        }
		// 避免长时间扫描，设置计时器在一定时间后调用停止扫描的函数
        Timer mTimer;
   		// 获取BluetoothLeScanner
        mBluetoothLeScanner = mBluetoothAdapter.getBluetoothLeScanner();
    	// 根据enable参数决定是开始扫描还是停止扫描
        if (enable) {
            mTimer = new Timer();
           	// 设置一段时间（30s）后自动停止扫描
            mTimer.schedule(new TimerTask() {
                @Override
                public void run() {
                    mScanning = false;
                    mBluetoothLeScanner.stopScan(mBleScanCallback);
                }
            }, 30 * 1000);
            // 设置自定义的是否处于扫描中的标志变量
            mScanning = true;
            // 使用默认设置开始扫描，参数为扫描回调函数，如果不需要后台长时间扫描采用默认设置就可以了
            mBluetoothLeScanner.startScan(mBleScanCallback);
            
            /*//通过ScanSettings.Builder来配置扫描设置，详见ScanSettings.Builder类文档和ScanSetting类文档
            ScanSettings.Builder mScanSettingBuilder = new ScanSettings.Builder();
            ScanSettings settings = mScanSettingBuilder
                    .setScanMode(ScanSettings.SCAN_MODE_LOW_POWER)
                    .setReportDelay(1000).build();
            // 另一种开始扫描的方法重载
            mBluetoothLeScanner.startScan(null, settings, mBleScanCallback);
            */
        } else {
            // 停止扫描
            mScanning = false;
            mBluetoothLeScanner.stopScan(mBleScanCallback);
        }
        return true;
    }
```

## 扫描回调函数

扫描回调通过实现抽象类`ScanCallback`来实现

```java
 ScanCallback mBleScanCallback = new ScanCallback() {
     //##########################分割线#############################
     // 每次收到蓝牙设备的广播消息时（扫描到一个设备时）就会调用该方法
     @Override
     public void onScanResult(int callbackType, ScanResult result) {
         super.onScanResult(callbackType, result);
         // 通过ScanResult类型的参数result来获取BluetoothDevice
         BluetoothDevice mDevice = result.getDevice();
         // 获取device的名称
         Log.d(TAG, "onScanResult:" + mDevice.getName());

         if ("xxxx".equals(mDevice.getName())) {
             // 找到指定设备时停止扫描
             mBluetoothLeScanner.stopScan(this);
             // 开始连接指定设备（BleModule.this.context是我写的一个类中的一个变量，该变量就是MainActivity对象）
             mBluetoothGatt = mDevice.connectGatt(BleModule.this.context, false, gattCallback);
         }
     }
     
	 //##########################分割线#############################
     // 扫描设置设置为低功耗（ScanSettings.SCAN_MODE_LOW_POWER）并且设置结果报告延迟后(ScanSettings.BUilder.setReportDelay)后系统会把一段时间的搜索结果合并起来通过该方法回调
     @Override
     public void onBatchScanResults(List<ScanResult> results) {
         super.onBatchScanResults(results);
         for (ScanResult result : results) {
             BluetoothDevice mDevice =  result.getDevice();
             Log.d(TAG, "onBatchScanResults:" + mDevice.getName());
         }
     }
	 
     //##########################分割线#############################
     // 扫描故障
     @Override
     public void onScanFailed(int errorCode) {
         super.onScanFailed(errorCode);
         Log.d(TAG, "onScanFailed:" + String.valueOf(errorCode));
     }
 };
```

## 连接设备

```java
// 调用BluetoothDevice的connectGatt方法来连接到该设备，得到BluetoothGatt对象mBluetoothGatt，该对象代表了同蓝牙设备建立的一个连接，之后发送和接收数据都需要它， 连接结果通过BluetoothGattCallback类的对象gattCallback中的回调函数来处理
//参数为(上下文, 是否自动连接, 回调函数)
mBluetoothGatt = mDevice.connectGatt(BleModule.this.context, false, gattCallback);
```

##　连接回调函数

```java
BluetoothGattCallback gattCallback = new BluetoothGattCallback() {
    //##########################分割线#############################
    // 在成功连接或者断开时会调用该方法
    @Override
    public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
        super.onConnectionStateChange(gatt, status, newState);
        // 成功连接
        if (newState == BluetoothProfile.STATE_CONNECTED) {
            Log.d(TAG, "Connected to GATT server.");
            // 成功连接后需要调用BluetoothGatt的discoverService方法发现该蓝牙设备上的服务
            Log.d(TAG, "Attempting to start service discovery:" +
                  mBluetoothGatt.discoverServices());
		//　断开连接
        } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {
            Log.i(TAG, "Disconnected from GATT server.");
        }
    }
	
    //##########################分割线#############################
    // BluetoothGatt的discoverService方法扫描到蓝牙设备上的服务后会调用该方法
    @Override
    public void onServicesDiscovered(BluetoothGatt gatt, int status) {
        super.onServicesDiscovered(gatt, status);
        Log.d(TAG, "onServicesDiscovered received: " + status);
		
        // 使用UUID获取指定服务, UUID对应的功能说明由蓝牙设备制造商提供
        BluetoothGattService service = gatt.getService( UUID.fromString("0000ffe0-0000-1000-8000-00805f9b34fb"));
        if (service != null) {
            // 使用UUID获取指定服务中的指定特征(characteristic), 数据的发送和设置都需要操作BluetoothGattCharacteristic对象
            BluetoothGattCharacteristic characteristic = service.getCharacteristic(UUID.fromString("0000ffe1-0000-1000-8000-00805f9b34fb"));
            if (characteristic != null) {
                // 使用UUID获取特征(characteristic)的描述符(BluetoothGattDescriptor), 操作描述符可以对特征进行功能的设置
                BluetoothGattDescriptor des = mChar.getDescriptor(UUID.fromString("00002902-0000-1000-8000-00805f9b34fb"));
                // 在描述符中设置接收方式为NOTIFICATION(这样才能接收到蓝牙设备主动发送的信息)
                Log.d(TAG, "onServicesDiscovered: descriptor set " + des.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE));
                mBluetoothGatt.writeDescriptor(des);
                // 开启该特征的Notification方式接收数据功能
                mBluetoothGatt.setCharacteristicNotification(mChar, true);
            }
        }
    }
	
    //##########################分割线#############################
    // 在某些可读的特征调用readCharacteristic方法后读取结果会通过该回调函数传递
    @Override
    public void onCharacteristicRead(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic, int status) {
        super.onCharacteristicRead(gatt, characteristic, status);
        byte[] data = characteristic.getValue();
        Log.d(TAG, "onCharacteristicRead: read success" + data);
    }

    //##########################分割线#############################
    // 接收到蓝牙设备主动发送的数据时该方法会被调用, 通过读取characteristic的值得到数据
    @Override
    public void onCharacteristicChanged(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic) {
        super.onCharacteristicChanged(gatt, characteristic);
        byte[] data = characteristic.getValue();
        String message;

        try {
            message = new String(data, "ascii");
        } catch (UnsupportedEncodingException e) {
            Log.d(TAG, "onCharacteristicChanged: error coding");
            message = Arrays.toString(data);
        }
        Log.d(TAG, "onCharacteristicChanged: read success:" + message);
    }


};

```

# 获取服务(Service)和特征(Characteristic)

上一节的连接回调函数部分讲到了获取服务和特征的方式, 总结如下: 

1. 调用`BluetoothGatt.discoverService`方法发现服务

2. 当服务被发现后`onServicesDiscovered`回调函数会被调用, 此时调用`BluetooGatt`对象的`getService`或者`getServices`方法可以获取到`BluetoothGattService`类型的指定UUID的服务或者所有服务列表

   ```java
   // 获取所有服务并打印uuid
   List<BluetoothGattService> services = gatt.getServices();
   for(BluetoothGattService service : services) {
       UUID service_uuid = service.getUuid();
       Log.d(TAG, "onServicesDiscovered: service id is " + service_uuid);
   }
   // 获取指定UUID的服务
   BluetoothGattService service = gatt.getService( UUID.fromString("0000ffe0-0000-1000-8000-00805f9b34fb"));
   ```

3. 调用`BluetoothGattService`对象的`getCharacteristic`或者`getCharacteristics`方法可以获取到`BluetoothGattCharacteristic`类型的指定UUID的特征或者所有特征列表

   ```java
   // 获取所有特征并打印uuid
   List<BluetoothGattCharacteristic> characteristics = service.getCharacteristics()
   for (BluetoothGattCharacteristic characteristic : characteristics) {
       UUID characteristic_uuid = characteristic.getUuid();
       Log.d(TAG, "onServicesDiscovered: characteristic id is " + characteristic_uuid);
   }
   
   // 获取指定UUID的特征
   BluetoothGattCharacteristic characteristic = service.getCharacteristic(UUID.fromString("0000ffe1-0000-1000-8000-00805f9b34fb"));
   ```

   

## 数据接收和发送

上一节得到 `BluetoothGattCharacteristic`对象后就可以进行数据接收和发送操作了, 数据的接收和发送都需要操作`BluetoothGattCharacteristic`类型的对象.

###　数据的发送

```java
public boolean sendData(String data) {
    // mChar是BluetoothGattCharacteristic类型的变量
    if (mChar == null) {
        Log.d(TAG, "sendData: bluetooth connection not ready, can't send data");
        return false;
    }
    //将数据放到byte类型数组中, 使用小于20字节的数据兼容性最好, 当然也可以其他方式改变单次发送的最大数据量(MTU最大传输单元)
    byte[] byte_data = data.getBytes();
    if(byte_data.length > 20) {
        Log.d(TAG, "sendData: data too long to send");
        return false;
    }
    Log.d(TAG, "sendData: " + data);
    // 使用BluetoothGattCharacteristic.setValue方法来设置characteristic的值
    mChar.setValue(byte_data);
    //发送数据
    mBluetoothGatt.writeCharacteristic(mChar);
    return true;
}
```



### 数据的接收

前面的连接回调函数部分的讲到了通过nofity方式从蓝牙设备接收数据的方法,  总结一下就是

1. 设置对应Characteristic的descriptor的value为ENABLE_NOTIFICATION_VALUE

   ```java
   // des为BluetoothGattDescriptor类型变量
   des.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE))
   ```

2. 写入descriptor

   ```java
   mBluetoothGatt.writeDescriptor(des);
   ```

3. 开启对应Characteristic的Notification方式接收数据功能

   ```java
   // mChar为BluetoothGattCharacteristic类型的变量, true表示开启
   mBluetoothGatt.setCharacteristicNotification(mChar, true);
   ```

4. 这样在`onCharacteristicChanged`回调函数中就可以读取数据了

   ```java
   byte[] data = characteristic.getValue();
   // 使用String构造函数可以转化成字符串
   message = new String(data, "ascii");
   ```

   





















