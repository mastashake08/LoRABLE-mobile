<script lang="ts" setup>
import {
  ref,
  onMounted,
  onUnmounted,
  $navigateTo,
} from 'nativescript-vue';
import { Bluetooth } from '@nativescript-community/ble';
import { Application, Dialogs } from '@nativescript/core';
import Details from './Details.vue';

interface BLEDevice {
  UUID: string;
  name: string;
  RSSI: number;
  state: string;
}

const bluetooth = new Bluetooth();
const devices = ref<BLEDevice[]>([]);
const isScanning = ref(false);
const statusMessage = ref('Ready to scan');

onMounted(async () => {
  console.log('BLE Scanner mounted');
  
  // Check if Bluetooth is enabled
  const enabled = await bluetooth.isBluetoothEnabled();
  if (!enabled) {
    statusMessage.value = 'Bluetooth is disabled';
  }
});

onUnmounted(() => {
  console.log('BLE Scanner unmounted');
  if (isScanning.value) {
    stopScanning();
  }
});

async function requestPermissions() {
  try {
    const hasPermission = await bluetooth.hasLocationPermission();
    if (!hasPermission) {
      const granted = await bluetooth.requestLocationPermission();
      if (!granted) {
        await Dialogs.alert({
          title: 'Permission Required',
          message: 'Location permission is required for BLE scanning on Android.',
          okButtonText: 'OK'
        });
        return false;
      }
    }
    return true;
  } catch (error) {
    console.error('Permission error:', error);
    return false;
  }
}

async function startScanning() {
  const hasPermission = await requestPermissions();
  if (!hasPermission) {
    statusMessage.value = 'Permission denied';
    return;
  }

  const enabled = await bluetooth.isBluetoothEnabled();
  if (!enabled) {
    statusMessage.value = 'Please enable Bluetooth';
    await Dialogs.alert({
      title: 'Bluetooth Disabled',
      message: 'Please enable Bluetooth to scan for devices.',
      okButtonText: 'OK'
    });
    return;
  }

  devices.value = [];
  isScanning.value = true;
  statusMessage.value = 'Scanning for devices...';

  try {
    await bluetooth.startScanning({
      seconds: 10, // Scan for 10 seconds
      onDiscovered: (peripheral) => {
        console.log('Device discovered:', peripheral.name || peripheral.UUID);
        
        // Check if device already exists
        const exists = devices.value.find(d => d.UUID === peripheral.UUID);
        if (!exists) {
          devices.value.push({
            UUID: peripheral.UUID,
            name: peripheral.name || 'Unknown Device',
            RSSI: peripheral.RSSI || -100,
            state: 'disconnected'
          });
        }
      },
      skipPermissionCheck: false
    });
    
    // Scanning will stop automatically after 10 seconds
    setTimeout(() => {
      isScanning.value = false;
      statusMessage.value = `Found ${devices.value.length} device(s)`;
    }, 10000);
    
  } catch (error) {
    console.error('Scanning error:', error);
    isScanning.value = false;
    statusMessage.value = 'Scanning failed';
    await Dialogs.alert({
      title: 'Scan Error',
      message: String(error),
      okButtonText: 'OK'
    });
  }
}

async function stopScanning() {
  try {
    await bluetooth.stopScanning();
    isScanning.value = false;
    statusMessage.value = `Found ${devices.value.length} device(s)`;
  } catch (error) {
    console.error('Stop scanning error:', error);
  }
}

async function connectToDevice(device: BLEDevice) {
  try {
    statusMessage.value = `Connecting to ${device.name}...`;
    
    await bluetooth.connect({
      UUID: device.UUID,
      onConnected: (peripheral) => {
        console.log('Connected to:', peripheral.name);
        statusMessage.value = `Connected to ${device.name}`;
        
        // Navigate to details page with device info
        $navigateTo(Details, {
          props: {
            deviceUUID: device.UUID,
            deviceName: device.name,
            bluetooth: bluetooth
          }
        });
      },
      onDisconnected: (peripheral) => {
        console.log('Disconnected from:', peripheral.name);
        statusMessage.value = 'Device disconnected';
        Dialogs.alert({
          title: 'Disconnected',
          message: `Lost connection to ${device.name}`,
          okButtonText: 'OK'
        });
      }
    });
  } catch (error) {
    console.error('Connection error:', error);
    statusMessage.value = 'Connection failed';
    await Dialogs.alert({
      title: 'Connection Error',
      message: String(error),
      okButtonText: 'OK'
    });
  }
}
</script>

<template>
  <Frame>
    <Page>
      <ActionBar>
        <Label text="BLE Scanner" class="font-bold text-lg" />
      </ActionBar>

      <GridLayout rows="auto, auto, *" class="px-4">
        <!-- Status and Scan Button -->
        <StackLayout row="0" class="py-4">
          <Label
            :text="statusMessage"
            class="text-center text-base text-gray-600 mb-4"
          />
          
          <GridLayout columns="*, *" class="gap-2">
            <Button
              col="0"
              @tap="startScanning"
              :isEnabled="!isScanning"
              class="px-4 py-3 rounded-lg"
              :class="isScanning ? 'bg-gray-300' : 'bg-blue-500 text-white'"
            >
              {{ isScanning ? 'Scanning...' : 'Start Scan' }}
            </Button>
            
            <Button
              col="1"
              @tap="stopScanning"
              :isEnabled="isScanning"
              class="px-4 py-3 rounded-lg"
              :class="!isScanning ? 'bg-gray-300' : 'bg-red-500 text-white'"
            >
              Stop Scan
            </Button>
          </GridLayout>
        </StackLayout>

        <!-- Divider -->
        <StackLayout row="1" class="h-px bg-gray-300 my-2" />

        <!-- Device List -->
        <ScrollView row="2" v-if="devices.length > 0">
          <StackLayout>
            <GridLayout
              v-for="device in devices"
              :key="device.UUID"
              columns="*, auto"
              rows="auto, auto"
              class="p-4 mb-2 bg-white rounded-lg border border-gray-200"
              @tap="connectToDevice(device)"
            >
              <Label
                row="0"
                col="0"
                :text="device.name"
                class="text-lg font-semibold text-gray-900"
              />
              <Label
                row="0"
                col="1"
                :text="`${device.RSSI} dBm`"
                class="text-sm text-gray-500"
              />
              <Label
                row="1"
                col="0"
                colSpan="2"
                :text="device.UUID"
                class="text-xs text-gray-400 mt-1"
                textWrap="true"
              />
            </GridLayout>
          </StackLayout>
        </ScrollView>

        <!-- Empty State -->
        <StackLayout row="2" v-else class="vertical-middle">
          <Label
            text="No devices found"
            class="text-center text-gray-400 text-lg"
          />
          <Label
            text="Tap 'Start Scan' to discover BLE devices"
            class="text-center text-gray-400 text-sm mt-2"
          />
        </StackLayout>
      </GridLayout>
    </Page>
  </Frame>
</template>

<style scoped>
.gap-2 {
  column-gap: 8;
}
</style>
