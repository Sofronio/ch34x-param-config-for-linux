# CH343 USB-to-Serial Driver Installation and Configuration Guide for Linux

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Driver Installation](#driver-installation)
4. [CDC-ACM Driver Conflict Resolution](#cdc-acm-driver-conflict-resolution)
5. [Device Configuration Tool](#device-configuration-tool)
6. [Troubleshooting](#troubleshooting)
7. [References](#references)

## Overview

This guide provides comprehensive instructions for installing and configuring WCH CH343 series USB-to-serial drivers on Linux systems, particularly Raspberry Pi 5. The CH343 family includes:

- **Single-port chips**: CH343P, CH346C, CH347T/F, CH9101U/H/R/Y, CH9102F, CH9111L
- **Dual-port chips**: CH342F, CH346C, CH347T/F, CH9103M
- **Quad-port chips**: CH344Q/L, CH348Q/L, CH9104L, CH9114F/L/W, CH9344Q

**Critical Note**: These devices are Communications Device Class (CDC) compliant, causing automatic binding to the Linux `cdc-acm` driver. This creates conflicts that must be manually resolved.

## Prerequisites

### Hardware Requirements
- Raspberry Pi 5 (or compatible Linux system)
- CH343-based USB-to-serial adapter
- USB cable for connection

### Software Requirements
- Raspberry Pi OS or compatible Linux distribution
- Kernel headers: `sudo apt install raspberrypi-kernel-headers`
- Build tools: `sudo apt install build-essential`
- Git: `sudo apt install git`

### Required Files
~~~
ch343ser_linux/
├── driver/
│   ├── ch343.c          # Main driver source
│   ├── Makefile         # Build configuration
│   ├── ch343.ko         # Compiled module (after build)
│   └── other source files
├── param_config/        # Configuration utilities
│   ├── ch34x_demo_param_config.c
│   ├── libch34xcfg.so   # Configuration library
│   ├── libch343.so      # CH343 library
│   ├── libch9344.so     # CH9344 library
│   └── header files
└── tools/              # Additional utilities
~~~

## Driver Installation

### Step 1: Obtain Driver Source
~~~
# Clone the repository
git clone https://github.com/WCHSoftGroup/ch343ser_linux.git
cd ch343ser_linux/driver
~~~

### Step 2: Check Current Device Status
~~~
# Verify device connection
lsusb | grep 1a86
# Expected output: Bus 003 Device 005: ID 1a86:55d3 QinHeng Electronics

# Check for existing serial devices
ls /dev/ttyCH343USB* /dev/ttyACM* 2>/dev/null
~~~
If the linked device is shown as ttyACM*, you should first checkout the CDC-ACM Driver Conflict Resolution part below.

### Step 3: Compile the Driver
~~~
# Clean previous builds
make clean

# Compile the driver
make
# Successful compilation creates ch343.ko
~~~
When you're using Orangepi, big change that your linux header is not installed.
Chech here first: https://github.com/Sofronio/ch34x-param-config-for-linux/blob/main/README.md#how-to-install-kernel-header-files

### Step 4: Load the Driver
~~~
# Temporary load (until reboot)
sudo insmod ch343.ko

# Verify driver loading
lsmod | grep ch343
sudo dmesg | tail -5 | grep -i ch343
~~~

## CDC-ACM Driver Conflict Resolution

### Understanding the Conflict
CH343 devices implement the CDC-ACM standard, causing Linux to automatically bind them to the generic `cdc-acm` driver. This creates several issues:

1. **Loss of functionality**: CDC-ACM lacks hardware flow control and GPIO support
2. **Device naming**: Creates `/dev/ttyACMx` instead of `/dev/ttyCH343USBx`
3. **Driver priority**: System prefers CDC-ACM over vendor-specific drivers

### Resolution Steps

#### 1. Remove CDC-ACM Driver
~~~
# Unload the module
sudo rmmod cdc_acm

# Prevent automatic loading
echo "blacklist cdc_acm" | sudo tee /etc/modprobe.d/blacklist-cdc-acm.conf
sudo update-initramfs -u
~~~

#### 2. Force CH343 Driver Binding
Create a binding script to handle automatic binding:

~~~
#!/bin/bash
# /usr/local/bin/ch343-bind.sh
for iface in /sys/bus/usb/devices/*/idVendor; do
    if [ -f "$iface" ] && [ "$(cat $iface)" = "1a86" ]; then
        dev_path=$(dirname "$iface")
        if [ -f "$dev_path/idProduct" ]; then
            dev_name=$(basename "$dev_path")
            for sub_iface in /sys/bus/usb/devices/${dev_name}:*; do
                if [ -d "$sub_iface" ]; then
                    iface_name=$(basename "$sub_iface")
                    # Unbind from cdc_acm
                    if [ -L "$sub_iface/driver" ]; then
                        driver_name=$(basename $(readlink "$sub_iface/driver"))
                        [ "$driver_name" = "cdc_acm" ] && \
                        echo "$iface_name" > "/sys/bus/usb/drivers/cdc_acm/unbind" 2>/dev/null
                    fi
                    # Bind to ch343
                    echo "$iface_name" > "/sys/bus/usb/drivers/ch343/bind" 2>/dev/null
                fi
            done
        fi
    fi
done
sleep 0.5
chmod 666 /dev/ttyCH343USB* 2>/dev/null
~~~

Make executable:
~~~
sudo chmod +x /usr/local/bin/ch343-bind.sh
~~~

#### 3. Create Udev Rule
~~~
# /etc/udev/rules.d/99-ch343.rules
SUBSYSTEM=="usb", ATTRS{idVendor}=="1a86", ACTION=="add", RUN+="/usr/local/bin/ch343-bind.sh"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", MODE="0666"
SUBSYSTEM=="usb", ATTRS{idVendor}=="1a86", MODE="0666"
~~~

Reload udev:
~~~
sudo udevadm control --reload-rules
sudo udevadm trigger
~~~

#### 4. Verify Resolution
~~~
# Check for correct device naming
ls /dev/ttyCH343USB*
# Should show: /dev/ttyCH343USB0

# Verify no CDC-ACM devices
ls /dev/ttyACM* 2>/dev/null || echo "No CDC-ACM devices found"
~~~

## Device Configuration Tool

### About CH34xSerCfg
The CH34xSerCfg utility configures USB parameters for WCH USB-to-serial chips, including:
- Vendor ID (VID) and Product ID (PID)
- Maximum current and power mode
- String descriptors (manufacturer, product, serial)
- Hardware features (flow control, MODEM signals, sleep mode)

### Installation Requirements

#### Library Installation
[https://github.com/WCHSoftGroup/ch9344ser_linux/tree/main/lib](https://github.com/WCHSoftGroup/ch9344ser_linux/tree/main/lib)

[https://github.com/WCHSoftGroup/ch343ser_linux/tree/main/lib](https://github.com/WCHSoftGroup/ch343ser_linux/tree/main/lib)
~~~
# Copy libraries to system path
sudo cp libch34xcfg.so /usr/lib/aarch64-linux-gnu/
sudo cp libch343.so /usr/lib/aarch64-linux-gnu/
sudo cp libch9344.so /usr/lib/aarch64-linux-gnu/

# Update library cache
sudo ldconfig

# Verify libraries
ldconfig -p | grep ch34
~~~

#### Compile from Source (Alternative)
~~~
cd ch343ser_linux/param_config
gcc ch34x_demo_param_config.c -lch34xcfg -lch343 -lch9344 -o CH34xSerCfg
~~~
When you're using orangepi, it might not have the header installed.
You can try this(Quote from Orangepi Zero2w Manual):

#### How to install kernel header files
Debian11 system with Linux6.1 kernel will report GCC error when compiling
kernel module. So if you want to compile the kernel module, please use Debian12 or
Ubuntu22.04.
1) The Linux image released by OPi comes with the deb package of the kernel header
file by default, and the storage location is /opt/
~~~
orangepi@orangepi:~$ ls /opt/linux-headers*
/opt/linux-headers-xxx-sun50iw9_x.x.x_arm64.deb
~~~
3) Use the following command to install the deb package of the kernel header file
~~~
orangepi@orangepi:~$ sudo dpkg-i /opt/linux-headers*.deb
rangePiUserManual CopyrightreservedbyShenzhenXunlongSoftwareCo.,Ltd
~~~
3)After installation, you can see the folder where the kernel header file is located under
/usr/src.
~~~
orangepi@orangepi:~$ls/usr/src
linux-headers-x.x.x
~~~

### Configuration File Format
Create a configuration file (e.g., `my_config.ini`):

~~~
[Public]
VendorID=1a86
ProductID=55d3
MaxPower(HEX)=FA
PowerMode=Bus-Powered
Wakeup_Enable=0
CDC_CTSRTS_FlowControl=0
EEPROM_Disable=0
TNOW_DTR_SoftSet=0
UARTx_TNOW_DTR_SETBIT=00
CH9101RY_MODEM_Enable=0
CH9101RY_DSR_MUX_TXS=0
CH9101RY_RI_MUX_RXS=0
CH9101RY_DCD_MUX_TNOW=0
CH9101RY_DTR_MUX_SUSPEND=0
Serial_String_enable=1
Product_String_enable=1
Manufacturer_String_enable=1
Serial_String=535A000001
Product_String=Half Decent Scale
Manufacturer_String=Decent Espresso
~~~

### Configuration Options Table
| Configuration Item | Description | Valid Values | Supported Chips |
|-------------------|-------------|--------------|-----------------|
| VendorID | Vendor identification code | Legal USB VID (hex) | All |
| ProductID | Product identification code | Legal USB PID (hex) | All |
| MaxPower(HEX) | Maximum current | Hexadecimal value | All |
| PowerMode | USB power mode | Self-Powered, Bus-Powered | All |
| Wakeup_Enable | Sleep wake function | 1: Enable, 0: Disable | CH342/CH9102 |
| CDC_CTSRTS_FlowControl | Hardware flow control in CDC mode | 1: Enable, 0: Disable | All CDC chips |
| TNOW_DTR_SoftSet | TNOW/DTR pin function | 1: Enable, 0: Disable | CH344/CH348/CH9104 |
| CH9101RY_MODEM_Enable | MODEM pin functions | 1: Enable, 0: Disable | CH9101R/Y |
| Serial_String_enable | Serial number string enable | 1: Enable, 0: Disable | All |
| Product_String_enable | Product string enable | 1: Enable, 0: Disable | All |
| Manufacturer_String_enable | Manufacturer string enable | 1: Enable, 0: Disable | All |
| Serial_String | Serial number string | 0-24 bytes | All |
| Product_String | Product information string | 0-40 bytes | All |
| Manufacturer_String | Manufacturer string | 0-40 bytes | All |
| Sleep_Mode_enable | USB sleep function | 1: Enable, 0: Disable | CH348/CH344/CH9114/CH9111 |
| TXD_state | TXD pin high-impedance state | 1: Enable, 0: Disable | CH342/CH9102 |

### Using the Configuration Tool

#### Basic Usage
~~~
sudo ./CH34xSerCfg /dev/ttyCH343USB0 CONFIG.INI
~~~
if it shows
~~~
./CH34xSerCfg: error while loading shared libraries: libch343.so: cannot open shared object file: No such file or directory
~~~
then use this instead
~~~
sudo LD_LIBRARY_PATH=./ ./CH34xSerCfg /dev/ttyCH343USB1 CONFIG.INI
~~~


#### Interactive Commands
~~~
g - Get current configuration
s - Set new configuration
r - Write manufacturer default configuration
q - Quit program
~~~

#### Example Session
~~~
$ sudo ./CH34xSerCfg /dev/ttyCH343USB0 my_config.ini
press g to get usb config, s to set usb config, r to set default config, q to quit this app.
s
Update [ Serial_String_enable       ]----------> [ON ]
Update [ Product_String_enable      ]----------> [ON ]
Update [ Manufacturer_String_enable ]----------> [ON ]
Update [ VendorID                   ]----------> [1a86]
Update [ ProductID                  ]----------> [55d3]
Configuration successful!
~~~

### Important Notes for Configuration

1. **VID/PID Changes**: Modifying VID or PID requires updating the driver's device ID table:
   ~~~
   // In ch343.c, add to usb_device_id array:
   { USB_DEVICE(0x1a86, 0xNEW_PID) },
   ~~~

2. **EEPROM Persistence**: Configuration is written to chip EEPROM and persists across power cycles.

3. **Device Replug**: Always replug the USB device after configuration for changes to take effect.

4. **Default Restoration**: Use 'r' command to restore manufacturer defaults if needed.

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: No ttyCH343USB devices appear
~~~
# Check driver loading
lsmod | grep ch343

# Check device recognition
lsusb | grep 1a86

# Force manual binding
sudo /usr/local/bin/ch343-bind.sh

# Check kernel messages
sudo dmesg | tail -20 | grep -i "ch343\|usb\|1a86"
~~~

#### Issue 2: Permission denied errors
~~~
# Temporary fix
sudo chmod 666 /dev/ttyCH343USB0

# Verify udev rules
sudo udevadm test /sys/class/tty/ttyCH343USB0 2>&1 | grep -i "mode"

# Check permissions
ls -la /dev/ttyCH343USB0
~~~

#### Issue 3: Driver compilation fails
~~~
# Install kernel headers
sudo apt update
sudo apt install raspberrypi-kernel-headers

# Verify kernel version matches headers
uname -r
ls /lib/modules/$(uname -r)/build

# Clean rebuild
make clean
make
~~~

#### Issue 4: Configuration tool library errors
~~~
# Check library paths
echo $LD_LIBRARY_PATH

# Use explicit library path
sudo LD_LIBRARY_PATH=/path/to/libs ./CH34xSerCfg /dev/ttyCH343USB0 CONFIG.INI

# Install libraries system-wide
sudo cp *.so /usr/lib/aarch64-linux-gnu/
sudo ldconfig
~~~

#### Issue 5: Reverting to CDC-ACM driver
~~~
# Remove CH343 driver
sudo rmmod ch343
sudo rm /etc/modprobe.d/blacklist-cdc-acm.conf

# Load CDC-ACM
sudo modprobe cdc_acm

# Update initramfs
sudo update-initramfs -u

# Replug device
~~~

### Diagnostic Commands
~~~
#!/bin/bash
echo "=== CH343 Diagnostic Report ==="
echo "1. USB Devices:"
lsusb | grep -i "1a86\|qinheng"
echo ""
echo "2. Loaded Modules:"
lsmod | grep -E "ch343|cdc_acm|usbserial"
echo ""
echo "3. Device Nodes:"
ls -la /dev/ttyCH343USB* 2>/dev/null || echo "No CH343 devices"
ls -la /dev/ttyACM* 2>/dev/null || echo "No ACM devices"
echo ""
echo "4. Udev Status:"
sudo udevadm info /dev/ttyCH343USB0 2>/dev/null || echo "Device not found"
echo ""
echo "5. Kernel Messages:"
sudo dmesg | tail -30 | grep -i "usb\|tty\|ch343\|cdc"
~~~

## References

### Official Resources
- **GitHub Repository**: https://github.com/WCHSoftGroup/ch343ser_linux
- **Application Examples**: https://github.com/WCHSoftGroup/tty_uart
- **Manufacturer Website**: http://www.wch.cn
- **Technical Support**: tech@wch.cn

### Additional Information
- **Supported Device IDs**: Check `ch343_id_table[]` in ch343.c for complete list
- **CDC-ACM Specification**: https://www.usb.org/document-library/communications-class-specification
- **Linux USB Serial Programming**: https://www.kernel.org/doc/html/latest/driver-api/usb/usb-serial.html

### Version Information
- **Driver Version**: V2.0 (2025.09)
- **Supported Kernel**: 6.12.47+ (Raspberry Pi 5)
- **Tested Devices**: CH343P (1a86:55d3), CH9101RY (1a86:55e8)

---
**Disclaimer**: This guide is based on community testing and manufacturer documentation. Always verify compatibility with your specific hardware and software configuration.

**Last Updated**: 2024-01-19
**Contributors**: WCH Technical Support, Raspberry Pi Community
