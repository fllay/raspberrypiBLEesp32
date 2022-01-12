# raspberrypiBLEesp32


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
