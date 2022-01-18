# raspberrypiBLEesp32
To create communication between a raspberry pi (e.g. Kidbright AI bot, box) and a Kidbright32 board, we can use GATT BLE since it does not require paring process. It can only use MAC address and UUIDs to establish a link between the raspberry pi and the Kidbright32 board. 

BLE protocol stack is shown below. 

![alt text](https://github.com/fllay/raspberrypiBLEesp32/blob/main/images/BLEprotocol.png?raw=true)
BLE Protocol stack

The process to establish the BLE link is shown below. When the peripheral is turn on, it will start advitise (broadcast) the services. Then, the central device  (Raspberry pi) will scan for BLE broadcasting signal to check for matching service by checking service UUID. Characteristics whithin the service are where the data is stored. They act like variables for exchnaging data. There are three types of commands to exchange data between peripherals and the central. The `write` command is used for sending data from central to peripheral. The `read` command is used for central to get data from server on-demand. The server can push data to the client by using either `notify` or `indicate` command. `notify` command will be used when acknownlagment is not required where as `notify` command is used when the client can acknownlage for received data.


![alt text](https://github.com/fllay/raspberrypiBLEesp32/blob/main/images/BLErpikb.png?raw=true)




Let us start with ESP32 (Kidbright32 board) code. The following scketch will create a BLE server on Kidbright32 board. User can send data using serial command on Ardiono IDE serial terminal. 

ESP32 Arduino code for BLE server

```
/*
    Video: https://www.youtube.com/watch?v=oCMOYS71NIU
    Based on Neil Kolban example for IDF: https://github.com/nkolban/esp32-snippets/blob/master/cpp_utils/tests/BLE%20Tests/SampleNotify.cpp
    Ported to Arduino ESP32 by Evandro Copercini

   Create a BLE server that, once we receive a connection, will send periodic notifications.
   The service advertises itself as: 6E400001-B5A3-F393-E0A9-E50E24DCCA9E
   Has a characteristic of: 6E400002-B5A3-F393-E0A9-E50E24DCCA9E - used for receiving data with "WRITE" 
   Has a characteristic of: 6E400003-B5A3-F393-E0A9-E50E24DCCA9E - used to send data with  "NOTIFY"

   The design of creating the BLE server is:
   1. Create a BLE Server
   2. Create a BLE Service
   3. Create a BLE Characteristic on the Service
   4. Create a BLE Descriptor on the characteristic
   5. Start the service.
   6. Start advertising.

   In this example rxValue is the data received (only accessible inside that function).
   And txValue is the data to be sent, in this example just a byte incremented every second. 
*/
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include "esp_bt_device.h"#include "esp_bt_device.h"

BLEServer *pServer = NULL;
BLECharacteristic * pTxCharacteristic;
bool deviceConnected = false;
bool oldDeviceConnected = false;
uint8_t txValue = 0;

// See the following for generating UUIDs:
// https://www.uuidgenerator.net/

#define SERVICE_UUID           "6E400001-B5A3-F393-E0A9-E50E24DCCA9E" // UART service UUID
#define CHARACTERISTIC_UUID_RX "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"
#define CHARACTERISTIC_UUID_TX "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"


void printDeviceAddress() {
 
  const uint8_t* point = esp_bt_dev_get_address();
 
  for (int i = 0; i < 6; i++) {
 
    char str[3];
 
    sprintf(str, "%02X", (int)point[i]);
    Serial.print(str);
 
    if (i < 5){
      Serial.print(":");
    }
 
  }
}

class MyServerCallbacks: public BLEServerCallbacks {
    void onConnect(BLEServer* pServer) {
      deviceConnected = true;
    };

    void onDisconnect(BLEServer* pServer) {
      deviceConnected = false;
    }
};

class MyCallbacks: public BLECharacteristicCallbacks {
    void onWrite(BLECharacteristic *pCharacteristic) {
      std::string rxValue = pCharacteristic->getValue();

      if (rxValue.length() > 0) {
        Serial.println("*********");
        Serial.print("Received Value: ");
        for (int i = 0; i < rxValue.length(); i++)
          Serial.print(rxValue[i]);

        Serial.println();
        Serial.println("*********");
      }
    }
};


void setup() {
  Serial.begin(115200);

  // Create the BLE Device
  BLEDevice::init("UART Service");

  // Create the BLE Server
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  // Create the BLE Service
  BLEService *pService = pServer->createService(SERVICE_UUID);

  // Create a BLE Characteristic
  pTxCharacteristic = pService->createCharacteristic(
                    CHARACTERISTIC_UUID_TX,
                    BLECharacteristic::PROPERTY_NOTIFY
                  );
                      
  pTxCharacteristic->addDescriptor(new BLE2902());

  BLECharacteristic * pRxCharacteristic = pService->createCharacteristic(
                       CHARACTERISTIC_UUID_RX,
                      BLECharacteristic::PROPERTY_WRITE
                    );

  pRxCharacteristic->setCallbacks(new MyCallbacks());

  // Start the service
  pService->start();

  // Start advertising
  pServer->getAdvertising()->start();
  Serial.print("BT MAC: ");
  printDeviceAddress();
  Serial.println("");
  Serial.println("Waiting a client connection to notify...");
}

void loop() {

    /*if (deviceConnected) {
        pTxCharacteristic->setValue(&txValue, 1);
        pTxCharacteristic->notify();
        txValue++;
    delay(1000); // bluetooth stack will go into congestion, if too many packets are sent
  }*/
  
  if (deviceConnected) {
    if (Serial.available()){
      char ch = Serial.read();
      txValue = (uint8_t) ch;
      pTxCharacteristic->setValue(&txValue, 1);
      pTxCharacteristic->notify();
    }
  }

    // disconnecting
    if (!deviceConnected && oldDeviceConnected) {
        delay(500); // give the bluetooth stack the chance to get things ready
        pServer->startAdvertising(); // restart advertising
        Serial.println("start advertising");
        oldDeviceConnected = deviceConnected;
    }
    // connecting
    if (deviceConnected && !oldDeviceConnected) {
    // do stuff here on connecting
        oldDeviceConnected = deviceConnected;
    }
}
```

After BLE server starts, the serial terninal will show that it is waiting for connection. It can be seen that the BLE MAC address is also shown in the terminal. This MAC address will be used later in python code for making a connection. In this example, the MAC address is `8C:AA:B5:8C:B7:1A`

![alt text](https://github.com/fllay/raspberrypiBLEesp32/blob/main/images/Screen%20Shot%202565-01-17%20at%2015.31.45.png?raw=true)


Now we are readly for python code on Raspbery pi. The required package, bluepy , can ne install by using command `sudo pip3 install bluepy`. The output is somewhat similar to the following

```
pi@KidBrightAI02:~$ sudo pip3 install bluepy
WARNING: The directory '/home/pi/.cache/pip' or its parent directory is not owned or is not writable by the current user. The cache has been disabled. Check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
Collecting bluepy
  Downloading bluepy-1.3.0.tar.gz (217 kB)
     |████████████████████████████████| 217 kB 1.8 MB/s 
Building wheels for collected packages: bluepy
  Building wheel for bluepy (setup.py) ... done
  Created wheel for bluepy: filename=bluepy-1.3.0-cp36-cp36m-linux_aarch64.whl size=563418 sha256=0c3289f178a427d8ab76230ac7097d46e77915b56df5ea338bc6b05b6cf35d92
  Stored in directory: /tmp/pip-ephem-wheel-cache-gxuwz1_h/wheels/16/43/3b/dbf93f0d5e1fb36c82d1632ae26d1e921e2a6d898f05411a58
Successfully built bluepy
Installing collected packages: bluepy
Successfully installed bluepy-1.3.0
WARNING: You are using pip version 20.3.1; however, version 21.3.1 is available.
You should consider upgrading via the '/usr/bin/python3 -m pip install --upgrade pip' command.
```

To create BLE client on the Raspberry pi, we need to set the following values for connection.
- BLE MAC Address
- Service UUID
- Characteristic UUIDs

These values need to be matched with the values in Arduino code. For this example,
- BLE MAC Address is 8C:AA:B5:8C:B7:1A
- Service UUID is 6E400001-B5A3-F393-E0A9-E50E24DCCA9E
- Characteristic UUIDs are 6E400002-B5A3-F393-E0A9-E50E24DCCA9E, 6E400003-B5A3-F393-E0A9-E50E24DCCA9E
Hence, python code for BLE connection initialization is 
```
p = btle.Peripheral("8C:AA:B5:8C:B7:1A")   

# Setup to turn notifications on, e.g.
svc = p.getServiceByUUID("6E400001-B5A3-F393-E0A9-E50E24DCCA9E")
ch_Tx = svc.getCharacteristics("6E400002-B5A3-F393-E0A9-E50E24DCCA9E")[0]
ch_Rx = svc.getCharacteristics("6E400003-B5A3-F393-E0A9-E50E24DCCA9E")[0]
```
Complete python code is given below.

```
from bluepy import btle
import time

class MyDelegate(btle.DefaultDelegate):
    def __init__(self):
        btle.DefaultDelegate.__init__(self)
        # ... initialise here

    def handleNotification(self, cHandle, data):
        #print("\n- handleNotification -\n")
        print(data)
        # ... perhaps check cHandle
        # ... process 'data'

# Initialisation  -------

p = btle.Peripheral("8C:AA:B5:8C:B7:1A")   #NodeMCU-32S
#p = btle.Peripheral("24:0a:c4:e8:0f:9a")   #ESP32-DevKitC V4

# Setup to turn notifications on, e.g.
svc = p.getServiceByUUID("6E400001-B5A3-F393-E0A9-E50E24DCCA9E")
ch_Tx = svc.getCharacteristics("6E400002-B5A3-F393-E0A9-E50E24DCCA9E")[0]
ch_Rx = svc.getCharacteristics("6E400003-B5A3-F393-E0A9-E50E24DCCA9E")[0]

p.setDelegate( MyDelegate())

setup_data = b"\x01\00"
p.writeCharacteristic(ch_Rx.valHandle+1, setup_data)

lasttime = time.localtime()

while True:
    """
    if p.waitForNotifications(1.0):
        pass  #continue

    print("Waiting...")
    """
    
    nowtime = time.localtime()
    if(nowtime > lasttime):
        lasttime = nowtime
        stringtime = time.strftime("%H:%M:%S", nowtime)
        btime = bytes(stringtime, 'utf-8')
        try:
            ch_Tx.write(btime, True)
        except btle.BTLEException:
            print("btle.BTLEException");
        #print(stringtime)
        #ch_Tx.write(b'wait...', True)
        
    # Perhaps do something else here
```

This code will send date-time to the BLE peripheral (Kidbright32 board) every 1 second. Once run this code at the terminal.

```
pi@KidBrightAI02:~/python$ python3 bleuart.py

```

The serial monitor will show somthing like this.

![alt text](https://github.com/fllay/raspberrypiBLEesp32/blob/main/images/Screen%20Shot%202565-01-18%20at%2009.57.37.png?raw=true)


Data can be sent to BLE client (Raspberry pi) by using the serial terminal. For example, if we put "Eru" in the serial terminal and click `Send`.
![alt text](https://github.com/fllay/raspberrypiBLEesp32/blob/main/images/Screen%20Shot%202565-01-18%20at%2009.56.07.png?raw=true)

At the terminal, we will see the follwing.

```
pi@KidBrightAI02:~/python$ python3 bleuart.py
b'E'
b'r'
b'u'
b'\n'
```

