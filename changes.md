# changes in the program files

1) in acitivity_main.xml add some buttons and (Material) editText fields:

```plaintext
    <Button
        android:id="@+id/btnMainConnectHeartRateServiceDevices"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:text="connect to devices with a Heart Rate Service" />
        
    <Button
        android:id="@+id/btnMainDisconnectFromHeartRateServiceDevice"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:enabled="false"
        android:text="disconnect from device with a Heart Rate Service" />        
        
    <com.google.android.material.textfield.TextInputLayout
        android:id="@+id/etMainConnectedDeviceDecoration"
        style="@style/Widget.MaterialComponents.TextInputLayout.OutlinedBox"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginStart="16dp"
        android:layout_marginTop="16dp"
        android:layout_marginEnd="16dp"
        android:hint="connected device"
        android:visibility="visible"
        app:boxCornerRadiusBottomEnd="5dp"
        app:boxCornerRadiusBottomStart="5dp"
        app:boxCornerRadiusTopEnd="5dp"
        app:boxCornerRadiusTopStart="5dp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent">

        <com.google.android.material.textfield.TextInputEditText
            android:id="@+id/etMainConnectedDevice"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:ellipsize="end"
            android:focusable="false"
            android:text=""
            android:textSize="14sp"
            android:visibility="visible"
            tools:ignore="KeyboardInaccessibleWidget" />
    </com.google.android.material.textfield.TextInputLayout>
            
```

2) Expose the BluetoothHandler in MainActivity:

```plaintext
BluetoothHandler bluetoothHandler;

    private void initBluetoothHandler()
    {
        // BluetoothHandler.getInstance(getApplicationContext());
        // new in part 2
        bluetoothHandler = BluetoothHandler.getInstance(getApplicationContext());
    }

```

3) activate the buttons and editText in MainActivity:

```plaintext 
    Button connectToHrsDevices, disconnectFromHrsDevice;
    com.google.android.material.textfield.TextInputEditText connectedDevice;
    
    onCreate:
        connectToHrsDevices = findViewById(R.id.btnMainConnectToHeartRateServiceDevices);
        connectedDevice = findViewById(R.id.etMainConnectedDevice);    
```

4) in BluetoothHandler add and change the following

```plaintext 
    add this methods:
    
    public void connectToHeartRateServiceDevice() {
        startScanHrs();
    }
    
    // this will connect to HeartRateService devices only
    private void startScanHrs() {
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                central.scanForPeripheralsWithServices(new UUID[]{HEART_RATE_SERVICE_UUID});
            }
        },1000);
    }
    
    change these methods:
    
    BluetoothCentralManagerCallback
    
        @Override
        public void onBluetoothAdapterStateChanged(int state) {
            Timber.i("bluetooth adapter changed state to %d", state);
            if (state == BluetoothAdapter.STATE_ON) {
                // Bluetooth is on now, start scanning again
                // Scan for peripherals with a certain service UUIDs
                central.startPairingPopupHack();
                // changed in part 2
                // startScan();
                startScanHrs();
            }
        }
        
    private BluetoothHandler(Context context) {
        this.context = context;
        // Plant a tree
        Timber.plant(new Timber.DebugTree());
        // Create BluetoothCentral
        central = new BluetoothCentralManager(context, bluetoothCentralManagerCallback, new Handler());
        // Scan for peripherals with a certain service UUIDs
        central.startPairingPopupHack();
        // changed in part 2
        // the scanning is disabled here, it will be done by calling connectToHeartRateServiceDevice
        // startScan();
    }                
```

To get the peripherals mac address we need more constants for the BroadcastIntent:
```plaintext 
    public static final String BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS = "androidcrypto.bluetoothhandler.peripheralmacaddress";
    public static final String BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS_EXTRA = "androidcrypto.bluetoothhandler.peripheralmacaddress.extra";
```


In BluetoothCentralManagerCallback change these methods:
```plaintext 
        @Override
        public void onConnectedPeripheral(@NotNull BluetoothPeripheral peripheral) {
            Timber.i("connected to '%s'", peripheral.getName());
            Intent intent = new Intent(BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS);
            String returnString = peripheral.getAddress() + " (" +
            peripheral.getName() + ")";        
            intent.putExtra(BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS_EXTRA, returnString);
            context.sendBroadcast(intent);
        }

        @Override
        public void onConnectionFailed(@NotNull BluetoothPeripheral peripheral, final @NotNull HciStatus status) {
            Timber.e("connection '%s' failed with status %s", peripheral.getName(), status);
            Intent intent = new Intent(BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS);
            String returnString = "";
            intent.putExtra(BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS_EXTRA, returnString);
            context.sendBroadcast(intent);
        }

        @Override
        public void onDisconnectedPeripheral(@NotNull final BluetoothPeripheral peripheral, final @NotNull HciStatus status) {
            Timber.i("disconnected '%s' with status %s", peripheral.getName(), status);
            Intent intent = new Intent(BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS);
            String returnString = "";
            intent.putExtra(BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS_EXTRA, returnString);
            context.sendBroadcast(intent);
            
            // Reconnect to this device when it becomes available again
            handler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    central.autoConnectPeripheral(peripheral, peripheralCallback);
                }
            }, 5000);
        }
```



```plaintext 
  onCreate: 
  registerReceiver(getPeripheralMacAddressStateReceiver, new IntentFilter(BluetoothHandler.BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS));

  onDestroy:
  unregisterReceiver(getPeripheralMacAddressStateReceiver);
  
  add:
    private final BroadcastReceiver getPeripheralMacAddressStateReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String dataString = intent.getStringExtra(BluetoothHandler.BLUETOOTHHANDLER_PERIPHERAL_MAC_ADDRESS_EXTRA);
            if (dataString == null) return;
            connectedDevice.setText(dataString);
            // save the peripheralsMacAddress
            if (dataString.length() > 5) {
                peripheralMacAddress = dataString.substring(0, 17);
            } else {
                peripheralMacAddress = "";
            }
        }
    };

```



```plaintext 

```








