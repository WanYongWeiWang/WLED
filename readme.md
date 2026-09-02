# WLED

This is a fork of [WLED](https://github.com/wled/WLED), mainly focusing on integrating WLED's JSON APIs into the FQM protocol.

For original `readme`, see [here](/readme.md.archive).

## Control over serial

WLED supports JSON commands over serial. However, on ESP32-S3, the serial interface used by WLED depends on how the firmware was built.

With builds that enable USB CDC on boot (`ARDUINO_USB_CDC_ON_BOOT=1`), WLED uses the native USB CDC interface as `Serial`. The board's regular UART pins may still print ESP-ROM boot messages, but they are not necessarily monitored by WLED for JSON commands. Seeing boot logs alone therefore does not confirm that the port can control WLED.

On Linux, the native USB CDC port commonly appears as `/dev/ttyACM*`, while an external USB-to-UART bridge usually appears as `/dev/ttyUSB*`. If your board provides separate `USB/OTG` and `UART` ports, connect to the USB/OTG port and use the corresponding `/dev/ttyACM*` device.

WLED does not currently provide a web UI setting to select another UART. To use hardware UART RX/TX pins for serial control, WLED must be custom-built to use `Serial1`, or USB CDC must be disabled during compilation.

References: [WLED Serial documentation](https://kno.wled.ge/interfaces/serial/) and [WLED ESP32-S3 UART discussion](https://wled.discourse.group/t/uart-over-serial/15377).

## Env

Wled is built using **PlatformIO** / **Adruino**, it does not supports **ESP-IDF** framework.

## Modifications

