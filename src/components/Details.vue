<script lang="ts" setup>
import { ref, onMounted, onUnmounted, $navigateBack } from 'nativescript-vue';
import { Dialogs } from '@nativescript/core';

interface Props {
  deviceUUID: string;
  deviceName: string;
  bluetooth: any;
}

interface Service {
  UUID: string;
  characteristics: Characteristic[];
  expanded: boolean;
}

interface Characteristic {
  UUID: string;
  properties: {
    read: boolean;
    write: boolean;
    notify: boolean;
  };
  value?: string;
}

const props = defineProps<Props>();

const services = ref<Service[]>([]);
const isLoading = ref(true);
const statusMessage = ref('Loading services...');

onMounted(async () => {
  console.log('Connected device details mounted');
  await discoverServices();
});

onUnmounted(async () => {
  console.log('Disconnecting from device');
  try {
    await props.bluetooth.disconnect({
      UUID: props.deviceUUID
    });
  } catch (error) {
    console.error('Disconnect error:', error);
  }
});

async function discoverServices() {
  try {
    isLoading.value = true;
    statusMessage.value = 'Discovering services...';

    const peripheralServices = await props.bluetooth.discoverServices({
      peripheralUUID: props.deviceUUID
    });

    console.log('Services discovered:', peripheralServices);

    // Build services array with characteristics
    const servicesList: Service[] = [];
    
    for (const service of peripheralServices) {
      const characteristicsList: Characteristic[] = [];
      
      // Discover characteristics for each service
      const characteristics = await props.bluetooth.discoverCharacteristics({
        peripheralUUID: props.deviceUUID,
        serviceUUID: service.UUID
      });

      for (const char of characteristics) {
        characteristicsList.push({
          UUID: char.UUID,
          properties: {
            read: char.properties?.read || false,
            write: char.properties?.write || char.properties?.writeWithoutResponse || false,
            notify: char.properties?.notify || char.properties?.indicate || false
          },
          value: undefined
        });
      }

      servicesList.push({
        UUID: service.UUID,
        characteristics: characteristicsList,
        expanded: false
      });
    }

    services.value = servicesList;
    isLoading.value = false;
    statusMessage.value = `Found ${servicesList.length} service(s)`;
  } catch (error) {
    console.error('Service discovery error:', error);
    isLoading.value = false;
    statusMessage.value = 'Failed to discover services';
    await Dialogs.alert({
      title: 'Error',
      message: `Failed to discover services: ${error.toString()}`,
      okButtonText: 'OK'
    });
  }
}

function toggleService(service: Service) {
  service.expanded = !service.expanded;
}

async function readCharacteristic(service: Service, characteristic: Characteristic) {
  try {
    statusMessage.value = 'Reading...';
    
    const result = await props.bluetooth.read({
      peripheralUUID: props.deviceUUID,
      serviceUUID: service.UUID,
      characteristicUUID: characteristic.UUID
    });

    // Convert value to string (assuming text data)
    const decoder = new TextDecoder();
    const value = decoder.decode(new Uint8Array(result.value));
    characteristic.value = value || 'Empty';
    
    statusMessage.value = 'Read successful';
    
    await Dialogs.alert({
      title: 'Characteristic Value',
      message: `Value: ${characteristic.value}`,
      okButtonText: 'OK'
    });
  } catch (error) {
    console.error('Read error:', error);
    statusMessage.value = 'Read failed';
    await Dialogs.alert({
      title: 'Read Error',
      message: error.toString(),
      okButtonText: 'OK'
    });
  }
}

async function writeCharacteristic(service: Service, characteristic: Characteristic) {
  try {
    const result = await Dialogs.prompt({
      title: 'Write Value',
      message: 'Enter value to write:',
      okButtonText: 'Write',
      cancelButtonText: 'Cancel',
      defaultText: ''
    });

    if (result.result && result.text) {
      statusMessage.value = 'Writing...';
      
      // Convert string to bytes
      const encoder = new TextEncoder();
      const bytes = encoder.encode(result.text);
      
      await props.bluetooth.write({
        peripheralUUID: props.deviceUUID,
        serviceUUID: service.UUID,
        characteristicUUID: characteristic.UUID,
        value: Array.from(bytes)
      });

      statusMessage.value = 'Write successful';
      
      await Dialogs.alert({
        title: 'Success',
        message: 'Value written successfully',
        okButtonText: 'OK'
      });
    }
  } catch (error) {
    console.error('Write error:', error);
    statusMessage.value = 'Write failed';
    await Dialogs.alert({
      title: 'Write Error',
      message: error.toString(),
      okButtonText: 'OK'
    });
  }
}

async function toggleNotify(service: Service, characteristic: Characteristic) {
  try {
    statusMessage.value = 'Setting up notifications...';
    
    await props.bluetooth.startNotifying({
      peripheralUUID: props.deviceUUID,
      serviceUUID: service.UUID,
      characteristicUUID: characteristic.UUID,
      onNotify: async (result) => {
        const decoder = new TextDecoder();
        const value = decoder.decode(new Uint8Array(result.value));
        console.log('Notification received:', value);
        
        await Dialogs.alert({
          title: 'Notification',
          message: `New value: ${value}`,
          okButtonText: 'OK'
        });
      }
    });

    statusMessage.value = 'Notifications enabled';
    
    await Dialogs.alert({
      title: 'Success',
      message: 'Notifications enabled for characteristic',
      okButtonText: 'OK'
    });
  } catch (error) {
    console.error('Notify error:', error);
    statusMessage.value = 'Notify failed';
    await Dialogs.alert({
      title: 'Notify Error',
      message: error.toString(),
      okButtonText: 'OK'
    });
  }
}

function getShortUUID(uuid: string): string {
  // If it's a standard Bluetooth UUID, show short form
  if (uuid.length > 8 && uuid.includes('-')) {
    return uuid.substring(4, 8);
  }
  return uuid;
}
</script>

<template>
  <Page>
    <ActionBar>
      <GridLayout columns="auto, *">
        <Button
          col="0"
          text="← Back"
          @tap="$navigateBack"
          class="text-white bg-transparent"
        />
        <Label
          col="1"
          :text="deviceName"
          class="text-white font-bold text-base text-center"
        />
      </GridLayout>
    </ActionBar>

    <GridLayout rows="auto, *">
      <!-- Status Bar -->
      <StackLayout row="0" class="p-4 bg-blue-50">
        <Label
          :text="`Device: ${deviceName}`"
          class="text-base font-semibold text-gray-800"
        />
        <Label
          :text="`UUID: ${deviceUUID}`"
          class="text-xs text-gray-500 mt-1"
          textWrap="true"
        />
        <Label
          :text="statusMessage"
          class="text-sm text-blue-600 mt-2"
        />
      </StackLayout>

      <!-- Services List -->
      <ScrollView row="1" v-if="!isLoading && services.length > 0">
        <StackLayout class="p-4">
          <StackLayout
            v-for="(service, sIndex) in services"
            :key="service.UUID"
            class="mb-4 bg-white rounded-lg border border-gray-300"
          >
            <!-- Service Header -->
            <GridLayout
              columns="*, auto"
              class="p-3 bg-gray-100 rounded-t-lg"
              @tap="toggleService(service)"
            >
              <StackLayout col="0">
                <Label
                  text="Service"
                  class="text-xs text-gray-500 uppercase"
                />
                <Label
                  :text="getShortUUID(service.UUID)"
                  class="text-sm font-mono text-gray-800 mt-1"
                />
              </StackLayout>
              <Label
                col="1"
                :text="service.expanded ? '▼' : '▶'"
                class="text-gray-600 text-lg"
              />
            </GridLayout>

            <!-- Characteristics List (Expandable) -->
            <StackLayout v-if="service.expanded" class="p-2">
              <StackLayout
                v-for="(char, cIndex) in service.characteristics"
                :key="char.UUID"
                class="p-3 mb-2 bg-gray-50 rounded"
              >
                <Label
                  text="Characteristic"
                  class="text-xs text-gray-500 uppercase"
                />
                <Label
                  :text="getShortUUID(char.UUID)"
                  class="text-sm font-mono text-gray-800 mt-1"
                />
                
                <!-- Properties -->
                <GridLayout columns="auto, auto, auto" class="mt-2 gap-1">
                  <Label
                    col="0"
                    v-if="char.properties.read"
                    text="📖 Read"
                    class="text-xs bg-green-100 text-green-700 px-2 py-1 rounded"
                  />
                  <Label
                    col="1"
                    v-if="char.properties.write"
                    text="✏️ Write"
                    class="text-xs bg-blue-100 text-blue-700 px-2 py-1 rounded"
                  />
                  <Label
                    col="2"
                    v-if="char.properties.notify"
                    text="🔔 Notify"
                    class="text-xs bg-purple-100 text-purple-700 px-2 py-1 rounded"
                  />
                </GridLayout>

                <!-- Action Buttons -->
                <GridLayout columns="*, *, *" class="mt-3 gap-2">
                  <Button
                    col="0"
                    v-if="char.properties.read"
                    text="Read"
                    @tap="readCharacteristic(service, char)"
                    class="text-xs px-2 py-2 bg-green-500 text-white rounded"
                  />
                  <Button
                    col="1"
                    v-if="char.properties.write"
                    text="Write"
                    @tap="writeCharacteristic(service, char)"
                    class="text-xs px-2 py-2 bg-blue-500 text-white rounded"
                  />
                  <Button
                    col="2"
                    v-if="char.properties.notify"
                    text="Notify"
                    @tap="toggleNotify(service, char)"
                    class="text-xs px-2 py-2 bg-purple-500 text-white rounded"
                  />
                </GridLayout>

                <!-- Value Display -->
                <Label
                  v-if="char.value"
                  :text="`Value: ${char.value}`"
                  class="text-xs text-gray-600 mt-2 p-2 bg-white rounded"
                  textWrap="true"
                />
              </StackLayout>
            </StackLayout>
          </StackLayout>
        </StackLayout>
      </ScrollView>

      <!-- Loading State -->
      <StackLayout row="1" v-else-if="isLoading" class="vertical-middle">
        <ActivityIndicator busy="true" class="text-blue-500" />
        <Label
          :text="statusMessage"
          class="text-center text-gray-500 mt-4"
        />
      </StackLayout>

      <!-- Empty State -->
      <StackLayout row="1" v-else class="vertical-middle">
        <Label
          text="No services found"
          class="text-center text-gray-400 text-lg"
        />
      </StackLayout>
    </GridLayout>
  </Page>
</template>

<style scoped>
.gap-1 {
  column-gap: 4;
}

.gap-2 {
  column-gap: 8;
}
</style>
