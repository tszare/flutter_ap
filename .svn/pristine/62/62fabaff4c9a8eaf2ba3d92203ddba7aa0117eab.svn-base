package com.newsky.flutterbluetoothplugin;

import android.Manifest;
import android.bluetooth.BluetoothAdapter;
import android.bluetooth.BluetoothDevice;
import android.bluetooth.BluetoothManager;
import android.bluetooth.BluetoothSocket;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Build;
import android.util.Log;

import com.tbruyelle.rxpermissions2.Permission;
import com.tbruyelle.rxpermissions2.RxPermissions;

import java.io.IOException;
import java.io.PrintWriter;
import java.io.StringWriter;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.Timer;
import java.util.TimerTask;
import java.util.UUID;
import java.io.OutputStream;
import java.io.InputStream;

import io.flutter.plugin.common.EventChannel;
import io.flutter.plugin.common.MethodCall;
import io.flutter.plugin.common.MethodChannel;
import io.flutter.plugin.common.MethodChannel.MethodCallHandler;
import io.flutter.plugin.common.MethodChannel.Result;
import io.flutter.plugin.common.PluginRegistry.Registrar;
import io.reactivex.functions.Consumer;

/** FlutterBluetoothPlugin */
public class FlutterBluetoothPlugin implements MethodCallHandler{

    private static final String TAG = "FlutterBluetoothPlugin";
    private static final String NAMESPACE = "flutter_bluetooth_serial";
    private static final UUID MY_UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB");
    private Registrar registrar;

    private HashMap<String, ConnectedThread> connectedThreadMap = new HashMap<String, ConnectedThread>();
    private HashMap<String, ConnectThread> connectThreadMap = new HashMap<String, ConnectThread>();

    private BluetoothAdapter btAdapter;

    private RxPermissions rxPermissions;

    private EventChannel.EventSink statusSink;
    private EventChannel.EventSink readSink;

    // all devices can be bonded
    private List<BluetoothDevice> btDevicePairingByHand = new ArrayList<>();

    /** Plugin registration. */
    public static void registerWith(Registrar registrar) {
        new FlutterBluetoothPlugin(registrar);
    }

    @Override
    public void onMethodCall(MethodCall call, final Result result) {
        if (btAdapter == null && !"isAvailable".equals(call.method)) {
            result.error("bluetooth_unavailable", "the device does not have bluetooth", null);
            return;
        }

        final Map<String, Object> arguments = call.arguments();

        switch (call.method) {
            case "checkAvailable":
                result.success(btAdapter.isEnabled());
                break;
            case "enableBT":
                result.success(btAdapter.enable());
                break;
            case "checkPermission":
                if (rxPermissions == null) {
                    rxPermissions = new RxPermissions(registrar.activity());
                }

                if (!rxPermissions.isGranted(Manifest.permission.ACCESS_FINE_LOCATION)) {
                    rxPermissions.requestEach(Manifest.permission.ACCESS_FINE_LOCATION).subscribe(
                            new Consumer<Permission>() {
                                @Override
                                public void accept(Permission permission) {
                                    if (permission.granted) {
                                        // 用户已经同意该权限
                                        result.success(true);
                                    } else if (permission.shouldShowRequestPermissionRationale) {
                                        // 用户拒绝了该权限，没有选中『不再询问』（Never ask again）,那么下次再次启动时，还会提示请求权限的对话框
                                        result.error("permission_denied",
                                                permission.name + " is denied. More info should be provided.",
                                                null);
                                    } else {
                                        // 用户拒绝了该权限，并且选中『不再询问』
                                        result.error("permission_denied",
                                                permission.name + " is denied.",
                                                null);
                                    }
                                }
                            }
                    );
                } else {
                    result.success(true);
                }
                break;
            case "discover":
                if (btAdapter.isEnabled()) {

                    Log.d(TAG, "is discovering: " + btAdapter.isDiscovering() + "\r\n");

                    if (!btAdapter.isDiscovering()) {
                        btAdapter.startDiscovery();
                    }
//                    else {
//                        result.error("bluetooth_enable", "BluetoothAdapter is not enabled", null);
//                    }
                }
                break;
            case "connect":
                if (arguments.containsKey("address")) {
                    String address = (String) arguments.get("address");
                    connect(result, address);
                }
                break;
            case "disconnect":
                if (arguments.containsKey("address")) {
                    String address = (String) arguments.get("address");
                    disconnect(result, address);
                }
                break;
            case "write":
                if (arguments.containsKey("message") && arguments.containsKey("address")) {
                    String message = (String) arguments.get("message");
                    String address = (String) arguments.get("address");
                    write(result, message, address, false);
                } else {
                    registrar.activity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            result.error("invalid_argument", "argument 'message' not found", null);
                        }
                    });
                }

                break;
            case "writeHex":
                if (arguments.containsKey("message") && arguments.containsKey("address")) {
                    String message = (String) arguments.get("message");
                    String address = (String) arguments.get("address");
                    write(result, message, address, true);
                } else {
                    registrar.activity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            result.error("invalid_argument", "argument 'message' not found", null);
                        }
                    });
                }

                break;
            case "isConnected":
                if (arguments.containsKey("address")) {
                    String address = (String) arguments.get("address");
                    isConnected(result, address);
                }
                break;
            case "getBondedBTDevices":
                getBondedBTDevices(result);
                break;
            case "createBond":
                if (arguments.containsKey("address")) {
                    String address = (String) arguments.get("address");
                    createBond(result, address);
                }
                break;
            default:
                result.notImplemented();
                break;
        }
    }

    FlutterBluetoothPlugin(Registrar registrar) {
        this.registrar = registrar;
        MethodChannel channel = new MethodChannel(registrar.messenger(), NAMESPACE + "/methods");
        EventChannel stateChannel = new EventChannel(registrar.messenger(), NAMESPACE + "/state");
        EventChannel readChannel = new EventChannel(registrar.messenger(), NAMESPACE + "/read");

        BluetoothManager mBluetoothManager = (BluetoothManager) registrar.activity()
              .getSystemService(Context.BLUETOOTH_SERVICE);
        assert mBluetoothManager != null;
        this.btAdapter = mBluetoothManager.getAdapter();
        channel.setMethodCallHandler(this);
        stateChannel.setStreamHandler(streamHandler);
        readChannel.setStreamHandler(readResultsHandler);
    }

    private void createBond(Result result, String address) {

        boolean succeed = false;
        try {
            for (int i = 0; i < btDevicePairingByHand.size() ; i++) {
                if (btDevicePairingByHand.get(i).getAddress().equals(address)) {
                    btAdapter.startDiscovery();
                    succeed = BluetoothUtils.createBond(btDevicePairingByHand.get(i).getClass(), btDevicePairingByHand.get(i));
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        result.success(succeed);
    }

    private void getBondedBTDevices(Result result) {
        Set<BluetoothDevice> deviceSet = btAdapter.getBondedDevices();

        ArrayList<String> deviceAddressList = new ArrayList<>();

        if (deviceSet.size() != 0) {

            for (BluetoothDevice device : deviceSet) {
                if (device.getType() == 1 && device.getBluetoothClass().getMajorDeviceClass() == 7936) {
                    deviceAddressList.add(
                            device.getName() + "," + device.getAddress() + "," +
                                    device.getType() + "," +
                                    device.getBluetoothClass().getMajorDeviceClass());
                }

                for (int i = 0; i < btDevicePairingByHand.size(); i++) {
                    if (btDevicePairingByHand.get(i).getAddress().equals(device.getAddress())) {
                        btDevicePairingByHand.remove(i);
                    }
                }
            }
        }

        result.success(deviceAddressList);
    }

    private void isConnected(Result result, String address) {
        if (connectThreadMap.get(address) != null && connectedThreadMap.get(address) != null) {
            result.success(true);
        } else {
            result.success(false);
        }
    }

    private void disconnect(Result result, String address) {
        try {
            connectedThreadMap.get(address).cancel();
            connectedThreadMap.put(address, null);

            connectThreadMap.get(address).cancel();
            connectThreadMap.put(address, null);

            result.success(true);
        } catch (Exception e) {
            result.success(false);
        }
    }

    private void connect(Result result, String address) {

        if (connectThreadMap.containsKey(address) && connectedThreadMap.get(address) != null) {
        } else {
            connectThreadMap.put(address, new ConnectThread(address, result));
        }
        connectThreadMap.get(address).start();
    }

    private void write(Result result, String message, String address, boolean isHex) {

        try {
            if (connectedThreadMap.get(address) == null) {
            } else {
                connectedThreadMap.get(address).write(message, isHex);
//                result.success(true);
            }

        } catch (Exception ex) {
            Log.e(TAG, ex.getMessage(), ex);
//            result.error("write_error", ex.getMessage(), exceptionToString(ex));
        }
    }

    private class ConnectThread extends Thread {

        private String address;
        private Result result;

        private BluetoothSocket btSocket;

        ConnectThread(String address, Result result) {
            this.address = address;
            this.result = result;
        }

        public void run() {
            try {
                BluetoothDevice device = btAdapter.getRemoteDevice(address);

                if (device == null) {
                    registrar.activity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            result.error("connect_error", "can not find device", null);
                        }
                    });
                    return;
                }

//                int sdk = Build.VERSION.SDK_INT;
//                if (sdk >= 10) {
//                    btSocket = device.createInsecureRfcommSocketToServiceRecord(MY_UUID);
//                } else {
//                    btSocket = device.createRfcommSocketToServiceRecord(MY_UUID);
//                }

                btSocket = device.createRfcommSocketToServiceRecord(MY_UUID);

//                btAdapter.cancelDiscovery();

                if (btSocket == null) {
                    registrar.activity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            result.error("connect_error", "bluetoothSocket is not established", null);
                        }
                    });
                    return;
                }

                try {
                    btSocket.connect();
                } catch (Exception e) {
                    Log.e(TAG, e.getMessage(), e);
                    registrar.activity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            result.success(false);
                        }
                    });
                    return;
                }

                if (connectedThreadMap.containsKey(address) && connectedThreadMap.get(address) != null) {

                    registrar.activity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            result.success(true);
                        }
                    });
                } else  {
                    connectedThreadMap.put(address, new ConnectedThread(btSocket, address));
                    registrar.activity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            result.success(true);
                        }
                    });
                }

                connectedThreadMap.get(address).start();
            } catch (Exception ex) {
                Log.e(TAG, ex.getMessage(), ex);
            }
        }

        public void cancel() {
//            try {
//                if (btSocket != null) {
//                    btSocket.close();
//                }
//            } catch (Exception e) {
//                Log.e(TAG, e.getMessage(), e);
//            }
        }
    }

    private String exceptionToString(Exception ex) {
        StringWriter sw = new StringWriter();
        PrintWriter pw = new PrintWriter(sw);
        ex.printStackTrace(pw);
        return sw.toString();
    }

    private class ConnectedThread extends Thread {
        private final BluetoothSocket mmSocket;
        private final InputStream mmInStream;
        private final OutputStream mmOutStream;
        private final String btAddress;
        private List<byte[]> cache;
        private List<Integer> lengthList;
        private boolean isHex;
        int cacheLength = 0;

        ConnectedThread(BluetoothSocket socket, final String address) {
            mmSocket = socket;
            InputStream tmpIn = null;
            OutputStream tmpOut = null;
            btAddress = address;
            cache = new ArrayList<>();
            lengthList = new ArrayList<>();
            isHex = false;

            TimerTask task = new TimerTask() {
                @Override
                public void run() {
                    if (cacheLength == 0) {
                        cacheLength = cache.size();
                    } else {

                        if (cacheLength != cache.size()) {
                            cacheLength = cache.size();
                        } else {
                            if (cache.size() != 0 && lengthList.size() != 0) {
                                int allBytesLength = 0;

                                for (int i = 0; i < lengthList.size(); i++) {
                                    allBytesLength += lengthList.get(i);
                                }

                                System.out.print("all bytes size:  " + allBytesLength + "\r\n");

                                byte[] result = new byte[allBytesLength];

                                int countLength = 0;
                                for (int i = 0; i < cache.size(); i++) {

                                    byte[] b = cache.get(i);
                                    System.arraycopy(b, 0, result, countLength, lengthList.get(i));
                                    countLength += lengthList.get(i);
                                }

                                String resultStr = null;
                                try {
                                    if (!isHex) {
                                        resultStr = new String(result, 0, allBytesLength, "gb2312");
                                    } else {
                                        resultStr = HexUtils.BinaryToHexString(result);
                                    }
                                } catch (Exception e) {
                                    e.printStackTrace();
                                }

//                                System.out.println("From :   " + address + "     " + resultStr);
                                Log.i(TAG, "From :   " + address + "     " + resultStr);

                                final String finalResultStr = resultStr;
                                registrar.activity().runOnUiThread(new Runnable() {
                                    @Override
                                    public void run() {
                                        readSink.success(btAddress + "," + finalResultStr);
                                    }
                                });

                                cache.clear();
                                lengthList.clear();
                                cacheLength = 0;
                            }
                        }
                    }
                }
            };

            try {
                tmpIn = socket.getInputStream();
                tmpOut = socket.getOutputStream();

                Timer timer = new Timer();
                long delay = 0;
                long PeriodTime = 150;

                timer.scheduleAtFixedRate(task, delay, PeriodTime);
            } catch (IOException e) {
                e.printStackTrace();
            }

            mmInStream = tmpIn;
            mmOutStream = tmpOut;
        }

        public void run() {
//            byte[] buffer = new byte[1024];
            int bytes;

            while (true) {
                byte[] buffer = new byte[1024];
                try {
                    bytes = mmInStream.read(buffer);

                    cache.add(buffer);

                    lengthList.add(bytes);
                } catch (NullPointerException e) {
                    break;
                } catch (IOException e) {
                    break;
                }
            }
        }

        public void write(String message, boolean isHexSend) {
            try {
                isHex = isHexSend;
                if (isHex) {

                    mmOutStream.write(HexUtils.hex2byte(message));
                } else {
                    mmOutStream.write(message.getBytes("gb2312"));
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        public void cancel() {
            try {
                mmOutStream.flush();
                mmOutStream.close();

                mmInStream.close();

                mmSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private final EventChannel.StreamHandler streamHandler = new EventChannel.StreamHandler() {

        private final BroadcastReceiver mReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {


                String action = intent.getAction();

//                System.out.println(action);
                BluetoothDevice device = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);

                if (device != null) {
//                    String name = "未命名";
//
//                    if (device.getName()!= null) {
//                        name = device.getName();
//                    }

//                    Short rssi = intent.getExtras().getShort(BluetoothDevice.EXTRA_RSSI);
//
//                    if (rssi != 0) {
//                        statusSink.success(name + "," + device.getAddress() + "," +
//                                rssi + "," + device.getType() + "," +
//                                device.getBluetoothClass().getMajorDeviceClass());
//                    }
                }

//                statusSink.error("connect error", "set pin failed", device.getAddress());

                if (BluetoothDevice.ACTION_FOUND.equals(action)) {
                    if (device != null && device.getBluetoothClass().getMajorDeviceClass() == 7936 && device.getType() == 1) {
                        if (BluetoothDevice.BOND_NONE == device.getBondState()) {
                            try {

                                // check if this device in black sheet
                                boolean canbeAutoPinned = true;
                                if (btDevicePairingByHand.size() != 0) {
                                    for (int i = 0; i < btDevicePairingByHand.size(); i++) {
                                        if (btDevicePairingByHand.get(i).getAddress().equals(device.getAddress())) {
                                            canbeAutoPinned = false;
                                            break;
                                        }
                                    }
                                }

                                if (canbeAutoPinned) {
//                                    System.out.println("try to create bond" + ":" + device.getAddress());
                                    BluetoothUtils.createBond(device.getClass(), device);
                                }

                            } catch (Exception ex) {

                                final String address = device.getAddress();
                                registrar.activity().runOnUiThread(new Runnable() {
                                    @Override
                                    public void run() {
                                        statusSink.error("connect error", "bond device failed", address);
                                    }
                                });
                            }
                        }

                        if (BluetoothDevice.BOND_BONDING == device.getBondState()) {
                            try {
                                boolean canceled = BluetoothUtils.cancelBondProcess(device.getClass(), device);
//                                if (canceled) {
//                                    BluetoothUtils.createBond(device.getClass(), device);
//                                }
                            } catch (Exception ex) {
                            }
                        }

//                        else {
//                            statusSink.success(device.getName() + "," + device.getAddress() + "," + String.valueOf(intent.getExtras().getShort(BluetoothDevice.EXTRA_RSSI)));
//                        }
                    }
                }

                if (BluetoothDevice.ACTION_PAIRING_REQUEST.equals(action)) {

                    final String deviceAddress = device.getAddress();
                    try {
                        boolean needToAbortBroadcast = true;
                        if (btDevicePairingByHand.size() != 0) {
                            for (int i = 0; i < btDevicePairingByHand.size(); i++) {
                                if (btDevicePairingByHand.get(i).getAddress().equals(device.getAddress())) {
                                    needToAbortBroadcast = false;
                                    break;
                                }
                            }
                        }

                        boolean setPinOk = false;
                        if (needToAbortBroadcast) {
                            abortBroadcast();
                            setPinOk = BluetoothUtils.setPin(device.getClass(), device, "1234");
                        }

                        final String deviceName = device.getName();
                        final String rssi = String.valueOf(intent.getExtras().getShort(BluetoothDevice.EXTRA_RSSI));
                        if (setPinOk) {
                            registrar.activity().runOnUiThread(new Runnable() {
                                @Override
                                public void run() {
                                    statusSink.success(deviceName + "," + deviceAddress + "," + rssi + "," + "bonded");
                                }
                            });
                        } else {
                            btDevicePairingByHand.add(device);

                            registrar.activity().runOnUiThread(new Runnable() {
                                @Override
                                public void run() {
                                    statusSink.error("connect error", "set pin failed", deviceAddress);
                                }
                            });
                        }
                    } catch (Exception ex) {

                        registrar.activity().runOnUiThread(new Runnable() {
                            @Override
                            public void run() {
                                statusSink.error("connect error", "set pin failed", deviceAddress);
                            }
                        });
                    }
                }

                if (BluetoothAdapter.ACTION_DISCOVERY_FINISHED.equals(action)) {
                    if (btAdapter.isDiscovering()) {
                        btAdapter.cancelDiscovery();
                    }
                }
            }
        };

        @Override
        public void onListen(Object o, EventChannel.EventSink eventSink) {
            statusSink = eventSink;

            registrar.activity().registerReceiver(
                    mReceiver, new IntentFilter(BluetoothDevice.ACTION_FOUND));
            registrar.activity().registerReceiver(
                    mReceiver, new IntentFilter(BluetoothDevice.ACTION_PAIRING_REQUEST));
            registrar.activity().registerReceiver(
                    mReceiver, new IntentFilter(BluetoothAdapter.ACTION_DISCOVERY_FINISHED));
        }

        @Override
        public void onCancel(Object o) {

            System.out.println("Cancel receiver");
            statusSink = null;
            registrar.activity().unregisterReceiver(mReceiver);
        }
    };

    private final EventChannel.StreamHandler readResultsHandler = new EventChannel.StreamHandler() {
        @Override
        public void onListen(Object o, EventChannel.EventSink eventSink) {
            readSink = eventSink;
        }

        @Override
        public void onCancel(Object o) {
            readSink = null;
        }
    };

}
