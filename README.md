# Daikin Madoka Controller BLE Protocol Reverse Engineering

`Madoka` is the friendly commercial name given by Daikin to the `BRC1H` controller. This controller integrates a BLE chip, allowing control from a smartphone. The application is available freely for iOS and Android, called `Madoka Assistant`. This application is coded in React Native.

![Madoka](master/images/brc1h.jpg)

My reverse engineering work was done by observing exchanges between the phone and my `Madoka` controllers, using a home-made BLE relay.
By playing with parameters and values, I managed to reverse most of the protocol.

This article assumes that you have basic BLE knowledge (services, characteristics...).

## Methodology and tools

  * I used iOs app `MLight` to list device BLE services and characteristics
  * I used `Bleno`, a `NodeJS` library/framework to emulate a `BRC1H` device on Linux and intercept/log mobile app calls.
  * I used `TinyB`, an `Ìntel` framework to implement a BLE central using `java`
  * I used [dbus-bluez](https://github.com/hypfvieh/bluez-dbus), an amazing library that works far better than TinyB, using DBus and Unix sockets (only on Linux - therefore)
  * a Linux PC and a Mac - which unfortunately allowed me to discover a bug on newest 2019 16" MacBook Pro (https://discussions.apple.com/thread/250944058)

The final goal of this study is to implement an [OpenHAB](https://www.openhab.org) binding in my home-automation infrastructure.

The protocol being made available here - it can be implemented by anybody to integrate it in its HomeAutomation.

I also used the protocol with an ESP32 and the amazing firmware [ESP32-BLE2MQTT(https://github.com/shmuelzon/esp32-ble2mqtt).

## BLE Advertising, Detection and Pairing

The `device` must be paired with its `central`, whatever the OS is (iOS, Linux...). I'm not covering how to pair a device with Linux (Bluetoothctl...).

The `BRC1H` device is advertising itself by sending advertisement data. In order to be detectable by the `Madoka Assistant` application, some specific data have to be broadcasted.

The structure of the advertisement packet is the following:

```
<size of section> <section type> <data>
```

Packets must have 3 sections to be recognized correctly by the app:

```
1. Flags
2. Local Name
3. UUID
4. Manufacturer Specific Data
```

Because BLENO (The Node library I used to emulate BLE device) restricts the size of advertising packet (or maybe it has to be split in multiple packets and I did not want to lose time investigating this), I used a very short device name of 1 letter.

It uses standard Bluetooth EIR : https://www.bluetooth.com/specifications/assigned-numbers/generic-access-profile/.

Here is the advertisement buffer I used to make the `Madoka Assistant` application recognize my fake device:

```

// Section Flags
0x02 0x01 0x06
 \    \    \_____ 0x04 | 0x02 = BR/EDR not supported and LE General Discoverable Mode
  \    \_________ Section type = 0x01 = Flags
   \_____________ Size of section


// local name - 'X'
0x02 0x09 0x58
 \    \    \_____ Hex value of letter 'X'
  \    \_________ Section type = 0x09 = Local Name
   \_____________ Size of section

// service uuid
0x11 0x07 0x77 0xae 0x8c 0x12 0x71 0x9e 0x7b 0xb6 0xe6 0x11 0x3a 0x21 0x10 0xe1 0x41 0x21
 \    \    \_____ Beginning of 16 bytes UUID
  \    \_________ Section type = 0x07 = 128-bit Service Class UUID
   \_____________ Size of section = 16 bytes + section type = 17 bytes = 0x11


// Manufacturer Specific data
0x05 0xff 0x93 0x00 0xe0 0x00
 \    \    \_____ Uknown Bytes - but have to be present to be recognized by the App
  \    \_________ Section type = 0xff = Manufacturer Specific Data
   \_____________ Size of section
```

The `Manufacturer Specific Data` acts like a "password" to make the device appearing in the mobile app. If the data is different or absent, he will not appear in the device list.

When the `BRC1H` is in communication with a BLE Central (smartphone, computer...) - it will stop broadcasting itself and will not accept any communication. It was found that an unproper communication termination leaded to a broken device - forcing to perform a power off of the entire air-conditioning installation to restart the thermostat.

## BLE device services and characteristics

The device is composed of 2 services. The first service is composed of 3 characteristics while the second is only composed of 2 characteristics. Each service and characteristc is designated by an UUID.

The first service is used for firmware upgrade, and is not covered by this research. I only focused on the second service, which allows controlling the AC unit.

1. Service `2141e100-213a-11e6-b67b-9e71128cae77` (Firmware Management)
   - characteristic `2141e101-213a-11e6-b67b-9e71128cae77` : Type = `Notify`
   - characteristic `2141e102-213a-11e6-b67b-9e71128cae77` : Type = `WriteWithoutResponse`
   - characteristic `2141e103-213a-11e6-b67b-9e71128cae77` : Type = `Notify`


2. Service `2141e110-213a-11e6-b67b-9e71128cae77` (AC Unit Management)
   - characteristic `2141e111-213a-11e6-b67b-9e71128cae77` : Type = `Notify`
   - characteristic `2141e112-213a-11e6-b67b-9e71128cae77` : Type = `WriteWithoutResponse`

The rest of this study will only focus on service (2).

## UART over BLE

It was found that the communication protocol is an emulated UART over Bluetooth Low Energy. The `Notify` characteristic can be considered as an `RX` while the `WriteWithoutResponse` can be considered as `TX`.

BLE `Notify` requests require the BLE Central to subscribe to notification - otherwise no notifaction will be sent by the device...

Some limitations have been found (device specific - not BLE):
   - Message size is 20 bytes (both `RX`and `TX`)

The process is asynchronous - which means that although the logical communication protocol is sync (request/response), it uses 2 distincts BLE characteristics that requires the developer to implement async message treatment.

The controller will **never** sends spontaneous messages. It only replies to requests/queries. As such, a synchronous protocol can be considered. The message size being limited to 20 bytes, protocol implements fragmentation. Messages needs to be re-assembled.

### Transport Message structure

For transport, the message structure is as follow - for both requests and responses. A message part is called a `chunk`. Messages are hex-strings.

```
|     0x00     |   0x.. 0x.. .. 0x.. |
|<chunk number>|       <payload>     |
```

The chunk number starts at `00`. The first chunk has a special additional header, which is the payload total length, including this field.

Examples of valid messages (from transport perspective):
```
0x00 0x05 0x00 0x01 0x02 0x03
 \    \    \___________________ 4 bytes of data
  \    \_______________________ Total payload size : 4 bytes of data + 1 byte of size length
   \___________________________ Chunk ID
```

If the payload to be carried on is larger than 18 bytes (max message size 20 bytes - 1 byte of Chunk Id - 1 byte of payload size), then the message is fragmented in multiple chunks. Example:

```
0x00 0x13 0x00 0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d 0x0e 0x0f 0x10 0x11
 \    \    \___________________ First 18 bytes of data
  \    \_______________________ Total payload size : 18 bytes of data + 1 byte of size length = 19 = 0x13
   \___________________________ Chunk ID

0x01 0x12
 \    \_______________________ Last payload byte
  \___________________________ Chunk ID
```

This message structure applies to both queries (initiated by the `central`) and responses (from the `device`).

Depending on the Bluetooth library/framework/stack - notifications *might* arrive in the wrong order (chunk `01` before chunk `00`). This is the developer's responsibility to reassemble correctly messages!

### Request / Response protocol

On top of this UART/BLE Protocol, the `payload` itself has a protocol. Let's discuss it now!

**Warning** : Here, I will not discuss anymore the chunks/headers, and will only mention the `payload`

Requests (from `central` to `device`):

```
Byte |  Length  | Description
0x00 |   0x03   | Function ID
0x03 | variable | Function argument(s)
```

The arguments are using the following structure:
```
Byte |  Length  | Description
0x00 |   0x01   | Argument ID
0x01 |   0x01   | Size of the argument (in bytes)
0x02 | variable | value
```

A function can take multiple arguments! If no arguments, then the argument structure is e0x00 0x00`

Responses :

Responses are formatted similarly as requests:
```
Byte |  Length  | Description
0x00 |   0x03   | Function ID
0x03 | variable | Returned object(s)
```

Each object is a structure, with an ID, a size, and a value:
```
Byte |  Length  | Description
0x00 |   0x01   | Argument ID
0x01 |   0x01   | Size of the argument (in bytes)
0x02 | variable | Value
```

This can seems a bit obscure but it is actually simple.

### Common errors

 * Misunderstanding decimal and hexadecimal values
 * Forgetting the "transport" layer (chunks fragmentation)

### Identified functions

I have not reversed the entire list of functions (the application also supports an "operator" mode which contains additional functions).

Here are the list of functions identified so far (I might update this list with time and needs).

The return functions usually return much more information that I memtion here - but when they are useless of unknown I don't report them ! Make your parsers "flexible".

For parameter functions, all parameters are not mandatory.

#### Function 0 (0x00) - "GetGeneralInfo()"

 * Takes **no** argument.

 * Returns:

```
 ID  |  Size  | Name                       | Description
0x10 |  0x01  | majorControlOperations     | <unknown>
0x11 |  0x01  | majorFunctionsSupported    | <unknown>
0x12 |  0x01  | majorSystemFunctions       | <unknown>
0x13 |  0x01  | majorConfigurationSettings | <unknown>
0x40 |  0x03  | apiVersion                 | <unknown>
```

#### Function 32 (0x00 0x20) - "GetSettingStatus()"

* Takes **no** argument.

* Returns:

```
 ID  |  Size  | Name                       | Description
0x20 |  0x01  | turnedOn                   | 0x01 : The unit is ON
     |        |                            | 0x00 : The unit is OFF
```

#### Function 16416 (0x40 0x20) - "SetSettingStatus()"

* Takes **several** argument(s).

```
 ID  |  Size  | Name                            | Description
0x20 |  0x01  | turnedOn                        | Same as GetSettingStatus()
```

#### Function 48 (0x00 0x30) - "GetOperationMode()"

* Takes **no** argument.

* Returns:

```
 ID  |  Size  | Name                       | Description
0x1f |  0x01  | autoCoolHeat               | <unknown>
0x20 |  0x01  | currentMode                | Current Mode of AC unit:
     |        |                            |   0 = FAN
     |        |                            |   1 = DRY
     |        |                            |   2 = AUTO
     |        |                            |   3 = COOL
     |        |                            |   4 = HEAT
     |        |                            |   5 = VENTILATION
```

#### Function 16432 (0x40 0x30) - "SetOperationMode()"

* Takes **several** argument(s).

```
 ID  |  Size  | Name                            | Description
0x20 |  0x01  | currentMode                     | Same as GetSetpoint()
```

#### Function 64 (0x00 0x40) - "GetSetpoint()"

* Takes **no** argument.

* Returns:

```
 ID  |  Size  | Name                            | Description
0x20 |  0x02  | coolingSetpoint                 | GFLOAT encoding
0x21 |  0x02  | heatingSetpoint                 | GFLOAT encoding
0x30 |  0x01  | setpointRangeEnabled            |
0x31 |  0x01  | setpointMode                    |
0x32 |  0x01  | minimumSetpointDifferential     |
0xa0 |  0x01  | minCoolingSetpointLowerLimit    |
0xa1 |  0x01  | minHeatingSetpointLowerLimit    |
0xa2 |  0x02  | coolingSetpointLowerLimit       |
0xa3 |  0x02  | heatingSetpointLowerLimit       |
0xa4 |  0x01  | coolingSetpointLowerLimitSymbol |
0xa5 |  0x01  | heatingSetpointLowerLimitSymbol |
0xb0 |  0x01  | maxCoolingSetpointUpperLimit    |
0xb1 |  0x01  | maxHeatingSetpointUpperLimit    |
0xb2 |  0x01  | coolingSetpointUpperLimit       |
0xb3 |  0x01  | heatingSetpointUpperLimit       |
0xb4 |  0x01  | coolingSetpointUpperLimitSymbol |
0xb5 |  0x01  | heatingSetpointUpperLimitSymbol |
```

#### Function 16448 (0x40 0x40) - "SetSetpoint()"

* Takes **several** argument(s).

```
 ID  |  Size  | Name                            | Description
0x20 |  0x02  | coolingSetpoint                 | GFLOAT encoding
0x21 |  0x02  | heatingSetpoint                 | GFLOAT encoding
```

#### Function 80 (0x00 0x50) - "GetFanSpeed()"

* Takes **no** argument.

* Returns:

```
 ID  |  Size  | Name                            | Description
0x20 |  0x01  | coolingFanSpeed                 |   5 = MAX
     |        |                                 |   4 = MEDIUM
     |        |                                 |   3 = MEDIUM
     |        |                                 |   2 = MEDIUM
     |        |                                 |   1 = LOW
0x21 |  0x01  | heatingFanSpeed                 | Same as coolingFanSpeed
```

#### Function 16464 (0x40 0x50) - "SetFanSpeed()"

* Takes **several** argument(s).

```
 ID  |  Size  | Name                            | Description
0x20 |  0x01  | coolingFanSpeed                 | Fan speed in COOL mode
0x21 |  0x01  | heatingFanSpeed                 | Fan speed in HEAT mode
```

#### Function 272 (0x01 0x10) - "GetSensorInformation()"

* Takes **no** argument.

* Returns:

```
 ID  |  Size  | Name                            | Description
0x40 |  0x01  | indoorTemperature               | Temperature in Celsius
0x41 |  0x02  | outdoorTemperature              | Temperature in Celsius. Not always supported - Then reports 0xff is unavailable.
```

#### Function 304 (0x01 0x30) - "GetMaintenanceInformation()"

* Takes **no** argument.

* Returns:

```
 ID  |  Size  | Name                            | Description
0x45 |  0x03  | toshibaVersion                  | Release - Ex: 3.2.0
0x46 |  0x02  | bleVersion                      | BLE Ctrl Version - Ex: 5.11
```
