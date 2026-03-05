# BLE Implementation Guide

## Overview
This NativeScript Vue 3 app implements full BLE (Bluetooth Low Energy) scanning and connection functionality using the `@nativescript-community/ble` library.

## Features

### Home Screen (Scanner)
- ✅ Auto-request BLE permissions
- ✅ Check Bluetooth status
- ✅ Scan for BLE devices (10-second scan duration)
- ✅ Display found devices with name, UUID, and signal strength (RSSI)
- ✅ Connect to discovered devices
- ✅ Real-time status messages

### Details Screen (Connected Device)
- ✅ Auto-discover services and characteristics
- ✅ Display all services with expandable characteristics
- ✅ Show characteristic properties (Read, Write, Notify)
- ✅ Read characteristic values
- ✅ Write to characteristics
- ✅ Enable notifications for characteristics
- ✅ Auto-disconnect on navigation back

## Installation

The BLE library is already installed in `package.json`:
```json
"@nativescript-community/ble": "^3.1.22"
```

## Permissions Configured

### Android (AndroidManifest.xml)
- ✅ `BLUETOOTH` - Basic Bluetooth access
- ✅ `BLUETOOTH_ADMIN` - Bluetooth management
- ✅ `ACCESS_COARSE_LOCATION` / `ACCESS_FINE_LOCATION` - Required for BLE scanning
- ✅ `BLUETOOTH_SCAN` - Android 12+ scanning permission
- ✅ `BLUETOOTH_CONNECT` - Android 12+ connection permission
- ✅ `bluetooth_le` hardware feature declaration

### iOS (Info.plist)
- ✅ `NSBluetoothAlwaysUsageDescription` - Bluetooth usage description
- ✅ `NSBluetoothPeripheralUsageDescription` - Peripheral usage description
- ✅ `bluetooth-central` background mode

## Usage

### Running the App

```bash
# Android
ns run android

# iOS
ns run ios

# Clean build (recommended after permission changes)
ns clean
ns run android
```

### Scanning for Devices

1. Launch the app
2. Tap "Start Scan" button
3. Wait up to 10 seconds for devices to appear
4. Discovered devices will appear in a list showing:
   - Device name (or "Unknown Device")
   - Signal strength (RSSI in dBm)
   - Device UUID

### Connecting to a Device

1. Tap any device in the list
2. The app will attempt to connect
3. Upon successful connection, you'll be navigated to the Details screen
4. If connection fails, an error dialog will appear

### Interacting with Device Characteristics

On the Details screen:

1. **View Services**: Tap a service header to expand/collapse characteristics
2. **Read**: Tap "Read" button to read the current value
3. **Write**: Tap "Write" button, enter a value in the prompt, and submit
4. **Notify**: Tap "Notify" button to enable notifications for value changes

### Disconnecting

- Tap the "← Back" button to disconnect and return to the scanner
- The app automatically disconnects when leaving the Details screen

## Code Structure

```
src/
├── app.ts                 # App entry point
└── components/
    ├── Home.vue          # BLE Scanner screen
    └── Details.vue       # Connected device details
```

### Home.vue
- Manages BLE scanning lifecycle
- Handles permissions
- Displays discovered devices
- Initiates connections

### Details.vue
- Discovers services and characteristics
- Provides read/write/notify operations
- Manages device disconnection

## Common Issues & Solutions

### "Location permission is required"
- On Android, BLE scanning requires location permissions
- The app automatically requests these on first scan
- If denied, go to Settings > Apps > Your App > Permissions and enable Location

### "Please enable Bluetooth"
- Ensure Bluetooth is enabled in device settings
- The app will detect and prompt you

### "Failed to discover services"
- Some devices require pairing before service discovery
- Try connecting again
- Check that the device is in pairing/advertising mode

### Device not appearing in scan
- Ensure the device is powered on and advertising
- Move closer to the device (check RSSI)
- Try stopping and restarting the scan

### Android 12+ Issues
- Ensure you've run `ns clean` after adding Android 12+ permissions
- Check that `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT` are granted in app settings

## API Reference

### Bluetooth Instance Methods

```typescript
// Check if Bluetooth is enabled
await bluetooth.isBluetoothEnabled()

// Start scanning
await bluetooth.startScanning({
  serviceUUIDs: [],      // Empty = scan all
  seconds: 10,           // Scan duration
  onDiscovered: (peripheral) => { /* ... */ }
})

// Stop scanning
await bluetooth.stopScanning()

// Connect to device
await bluetooth.connect({
  UUID: deviceUUID,
  onConnected: (peripheral) => { /* ... */ },
  onDisconnected: (peripheral) => { /* ... */ }
})

// Disconnect
await bluetooth.disconnect({ UUID: deviceUUID })

// Discover services
await bluetooth.discoverServices({ peripheralUUID })

// Discover characteristics
await bluetooth.discoverCharacteristics({ 
  peripheralUUID, 
  serviceUUID 
})

// Read characteristic
await bluetooth.read({
  peripheralUUID,
  serviceUUID,
  characteristicUUID
})

// Write characteristic
await bluetooth.write({
  peripheralUUID,
  serviceUUID,
  characteristicUUID,
  value: [/* byte array */]
})

// Start notifications
await bluetooth.startNotifying({
  peripheralUUID,
  serviceUUID,
  characteristicUUID,
  onNotify: (result) => { /* ... */ }
})

// Stop notifications
await bluetooth.stopNotifying({
  peripheralUUID,
  serviceUUID,
  characteristicUUID
})
```

## Data Conversion Examples

### String to Bytes
```typescript
const encoder = new TextEncoder()
const bytes = encoder.encode('Hello')
// Use: Array.from(bytes)
```

### Bytes to String
```typescript
const decoder = new TextDecoder()
const text = decoder.decode(new Uint8Array(result.value))
```

### Number to Bytes
```typescript
const value = 42
const bytes = [value]
```

## Testing Tips

1. **Use a BLE Scanner App**: Install nRF Connect (iOS/Android) to verify your device is advertising
2. **Check Signal Strength**: RSSI values closer to 0 indicate stronger signal
3. **Service UUIDs**: Standard Bluetooth services use short UUIDs (e.g., "180F" for Battery Service)
4. **Monitor Console**: Enable debug mode and watch console logs for detailed BLE events

## Next Steps

- Add service/characteristic name mappings for common UUIDs
- Implement persistent device connections
- Add device pairing/bonding support
- Store recently connected devices
- Add filtering by service UUID
- Implement custom characteristic parsers for specific data formats

## Resources

- [NativeScript BLE Plugin Docs](https://github.com/NativeScript/plugins/tree/main/packages/ble)
- [Bluetooth SIG Assigned Numbers](https://www.bluetooth.com/specifications/assigned-numbers/)
- [NativeScript Vue 3 Docs](https://nativescript-vue.org/)

---

**Library Version**: @nativescript-community/ble ^3.1.22
**NativeScript Version**: ~8.9.1
**Vue Version**: 3.0.2
