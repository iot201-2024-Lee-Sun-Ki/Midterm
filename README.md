Midterm
=======

### About

This is your project's README.md file. It helps users understand what your
project does, how to use it and anything else they may need to know.

1. AWS Clean up Screen shots
 <img width="1440" alt="스크린샷 2024-04-26 오전 9 36 21" src="https://github.com/iot201-2024-Lee-Sun-Ki/Midterm/assets/141006849/c5c1a950-4d54-4a4d-bd53-dc551c2a7761">
<img width="1440" alt="스크린샷 2024-04-26 오전 9 46 02" src="https://github.com/iot201-2024-Lee-Sun-Ki/Midterm/assets/141006849/dfce953c-407a-4b12-a7a7-55643d2d1b36">


2. EC2 instance Domain name
   
<img width="1440" alt="스크린샷 2024-04-26 오전 9 40 28" src="https://github.com/iot201-2024-Lee-Sun-Ki/Midterm/assets/141006849/bf865477-6568-4273-8336-c98718578798">


3. Io7 Management Web Device screen shot
   
<img width="1440" alt="스크린샷 2024-04-26 오전 9 50 00" src="https://github.com/iot201-2024-Lee-Sun-Ki/Midterm/assets/141006849/e22ea4e8-3dd5-481b-bf6f-d13c60a7db09">
<img width="1440" alt="스크린샷 2024-04-26 오전 9 49 47" src="https://github.com/iot201-2024-Lee-Sun-Ki/Midterm/assets/141006849/9d81ef3a-1094-405b-b81a-d30a37843cb9">


4. Demonstration Video





https://github.com/iot201-2024-Lee-Sun-Ki/Midterm/assets/141006849/7db84869-3f18-4780-9742-d71a7b960305


5. Micropython Codes for rotary, thermometer, valve

```python
###Thermostat.py
from IO7FuPython import ConfiguredDevice
import json
import time
import uComMgr32
from machine import Pin, ADC, Timer
import st7789
import tft_config
import vga2_8x16 as font1
import vga1_bold_16x32 as font2




rotaryA = Pin(43, Pin.IN, Pin.PULL_UP)
rotaryB = Pin(44, Pin.IN, Pin.PULL_UP)

old_encode = 0
right = [13, 32, 20, 01]
left = [10, 02, 23, 31]

value = 0
def rotated(p):
    global old_encode, value, lastPub
    new_state = rotaryA.value() << 1| rotaryB.value()
    if new_state != old_encode:
        new_combined = old_encode * 10 + new_state
        old_encode = new_state
        if new_combined in right:
            value = (value + 1) if value < 255 else 255
        elif new_combined in left:
            value = (value - 1) if value > 0 else 0
        lastPub = time.ticks_ms() - device.meta['pubInterval'] + 50

rotaryA.irq(trigger=Pin.IRQ_FALLING | Pin.IRQ_RISING, handler=rotated)
rotaryB.irq(trigger=Pin.IRQ_FALLING | Pin.IRQ_RISING, handler=rotated)

tft = tft_config.config(3, buffer_size=64*64*2)

tft.init()
tft.fill(st7789.BLACK)
tft.text(font2, 'Thermostat', 85, 5, st7789.WHITE)
tft.text(font1, 'Target      : ', 65, 50, st7789.WHITE)


def r2t(r):
#     0.1960784 = (60 - 10) / 255 - 0
    return r * 0.1960784 + 10

def t2r(t):
#     0.1960784 = (60 - 10) / 255 - 0
     
    return (t - 10) / 0.1960784




def handleCommand(topic, msg):
    global lastPub, value
    jo = json.loads(str(msg,'utf8'))
    if ("target" in jo['d']):
        value = t2r(int(jo['d']['target']))
        lastPub = - device.meta['pubInterval']
        
nic = uComMgr32.startWiFi('io7thermostat')
device = ConfiguredDevice()
device.setUserCommand(handleCommand)

device.connect()

lastPub = time.ticks_ms() - device.meta['pubInterval']

while True:
    if not device.loop():
        tft.deinit()
        break
    
    if (time.ticks_ms() - device.meta['pubInterval']) > lastPub:
        lastPub = time.ticks_ms()
        device.publishEvent('status',
            json.dumps({
                'd' : {
                        'target': round(r2t(value),2),
                      }
                 }
            )
        )
    tft.text(font1, f'{round(r2t(value),2)}', 200, 50, st7789.WHITE)
```

```python
###dht.py

from IO7FuPython import ConfiguredDevice
import json
import time
import uComMgr32
from machine import Pin, ADC, Timer
import dht

sensor = dht.DHT22(Pin(15))


temerature = 0
humidity = 0




lastMeasured = 0
def measureData():
    global temperature, humidity, lastMeasured
    if (time.ticks_ms() - 2000) > lastMeasured:
        lastMeasured = time.ticks_ms()
        sensor.measure()
        temperature = sensor.temperature()
        humidity = sensor.humidity()

def handleCommand(topic, msg):
    global lastPub
    jo = json.loads(str(msg,'utf8'))
    if ("temperature" in jo['d']):
        lastPub = - device.meta['pubInterval']
        
nic = uComMgr32.startWiFi('io7thermostat')
device = ConfiguredDevice()
device.setUserCommand(handleCommand)

device.connect()

lastPub = time.ticks_ms() - device.meta['pubInterval']

while True:
    if not device.loop():
        break
    measureData()
    if (time.ticks_ms() - device.meta['pubInterval']) > lastPub:
        lastPub = time.ticks_ms()
        device.publishEvent('status',
            json.dumps({
                'd' : {
                        'temperature' : round(temperature, 2),
                        'humidity' : round(humidity, 2),
                      }
                 }
            )
        )

```

```python
###valve.py
from IO7FuPython import ConfiguredDevice
import json
import time
import uComMgr32
from machine import Pin

valve = Pin(15, Pin.OUT)
lastPub = time.ticks_ms() - device.meta['pubInterval']

def handleCommand(topic, msg):
    global lastPub
    jo = json.loads(str(msg,'utf8'))
    if ("valve" in jo['d']):
        if jo['d']['valve'] is 'on':
            valve.on()
        else:
            valve.off()
        lastPub = - device.meta['pubInterval']

nic = uComMgr32.startWiFi('valve')
device = ConfiguredDevice()
device.setUserCommand(handleCommand)

device.connect()



while True:
    # default is JSON format with QoS 0
    if not device.loop():
        break
    if (time.ticks_ms() - device.meta['pubInterval']) > lastPub:
        lastPub = time.ticks_ms()
        device.publishEvent('status', json.dumps({'d':{'valve': 'on' if valve.value() else 'off'}}))


```

6. github링크

https://github.com/iot201-2024-Lee-Sun-Ki/Midterm


