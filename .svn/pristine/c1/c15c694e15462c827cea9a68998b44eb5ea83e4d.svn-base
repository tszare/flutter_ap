import 'dart:async';

import 'package:flutter/services.dart';

typedef void EventHandler(Object event);

class FlutterBluetoothPlugin {

  //fake
  static const String platformVersion = '1.0';

  static const String namespace = 'flutter_bluetooth_serial';

  static const MethodChannel _channel = const MethodChannel('$namespace/methods');

  static const EventChannel _readChannel = const EventChannel('$namespace/read');

  static const EventChannel _stateChannel = const EventChannel('$namespace/state');

  final StreamController<MethodCall> _methodStreamController = new StreamController.broadcast();

  Stream<MethodCall> get _methodStream => _methodStreamController.stream;

  FlutterBluetoothPlugin._() {
    _channel.setMethodCallHandler((MethodCall call) {
      _methodStreamController.add(call);
    });
  }

  static FlutterBluetoothPlugin _instance = new FlutterBluetoothPlugin._();

  static FlutterBluetoothPlugin get instance => _instance;

  onRead(onEvent, onError) {
    _readChannel.receiveBroadcastStream().listen(onEvent, onError: onError);
  }

  listenStatus(onEvent, onError) {
    _stateChannel.receiveBroadcastStream().listen(onEvent, onError: onError);
  }

  Future<bool> checkAvailable() {
    return _channel.invokeMethod("checkAvailable");
  }

  Future<bool> enableBT() {
    return _channel.invokeMethod("enableBT");
  }

  Future<bool> isConnected(String address) {
    return _channel.invokeMethod('isConnected', {'address': address});
  }

  Future<bool> checkPermission(){
    return _channel.invokeMethod('checkPermission');
  }

  Future<bool> createBond(String address) {
    return _channel.invokeMethod('createBond', {'address': address});
  }

//  void discover() => _channel.invokeMethod('discover');

  void discover() {
    try {
      _channel.invokeMethod('discover');
    } catch (e) {
      print(e);
//      throw(e);
    }
  }
  
  Future<dynamic> getBondedBTDevices() {
    return _channel.invokeMethod('getBondedBTDevices');
  }

  Future<bool> connect(String address) =>
      _channel.invokeMethod('connect', {'address': address});

  Future<bool> disconnect(String address) => _channel.invokeMethod('disconnect', {'address': address});

  void write(String message, String address, bool hexSend) {
    if (hexSend) {
      _channel.invokeMethod('writeHex', {'message': message, 'address': address});
    } else {
      _channel.invokeMethod('write', {'message': message, 'address': address});
    }
  }


//  _onEvent(String event) {
//    connect(event).then(onSucceed());
//  }
//
//  onSucceed() {
//    write("time" + "\r\n");
//  }
//
//  _onError(Object event) {
//    print(event.toString());
//    print(event);
//  }
}

//class BluetoothDevice {
//  final String name;
//  final String address;
//  final int type;
//  bool connected = false;
//
//  BluetoothDevice.fromMap(Map map)
//      : name = map['name'],
//        address = map['address'],
//        type = map['type'];
//
//  Map<String, dynamic> toMap() => {
//    'name': this.name,
//    'address': this.address,
//    'type': this.type,
//    'connected': this.connected,
//  };
//
//  operator ==(Object other) {
//    return other is BluetoothDevice && other.address == this.address;
//  }
//
//  @override
//  int get hashCode => address.hashCode;
//}
