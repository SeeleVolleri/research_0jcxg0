# Researching Dell 0JCXG0
## 0x00 Abstract
Microsoft obviously do not satisfy with password or hardware passkeys back in 2015  
So they created Windows Hello and Windows Hello compatible standard.  
Manufacturers then begin cooperating Microsoft producing Windows Hello Devices
such as fingerprint and IR cameras.  
But what will happen if you try to plug such device onto Linux?
## 0x01 Introduction
The device is made by Realtek which has a Realtek vendor ID and REALSIL id (see below)  
and has two cameras and a IR emitter. Two streams are mutex locked.  
Back in the days when I'm using Windows 10, I saw this on Xianyu for 30 RMB.

Why not buy one just for fun?

It is fun, first time using my face unlock the computer, first time
see coke was transparent in IR vision, first time night vision.

Until I plug it on Linux using [Howdy](https://github.com/boltgolt/howdy)
trying to setup facial recognition. It failed.

Back in the good old days, this young man have no idea about Open Source.  
Young man do not know he need to send specific command to activate IR mode,  
So he didn't know he need reverse engineer to obtain vendor specific command(XU).  
So he didn't know he need usbmon and wireshark to reverse engineer.  
## 0x02 Get hands dirty
Wireshark is a tool that can capture, analyze frames that not limited to network.  
TCPDump is a tool that can capture a device and convert it to pcap format.  
SSH is a tool that can acts as a pipe.  
usbmon is a module that can let you capture USB communicate on a linux machine.

I believe you'll dive into manpages.

First, let's lookup the bus which device plugged onto.

```
root@hypervisor:~# lsusb
Bus 001 Device 021: ID 0bda:5767 Realtek Semiconductor Corp. Integrated_Webcam_HD
```

There. `Bus 001 Device 021`. Now let's plug it out and

Immediately start capturing USB bus 1 on hypervisor to view your loot.  
`ssh root@hypervisor tcpdump -i usbmon1 -w - | wireshark -k -i -`  
Please don't ask why usbmonx not exist.

We now see various frames, let's apply filter `usb.bus_id == 1 && usb.device_address == 22`
to view the fresh frames between device and host. The device address **must** add 1

What, nothing? Did you plug the cam back or messed the device address?

Now we see some noisy frames between device and host.

Now start the VM. libvirt, PVE, whatever. Go passthrough the usb device to a Windows Guest. 
Go install driver.  
Go reboot guest.  
Go start Windows Hello setup steps.  
Actually you cancel once you see your face to reduce data.  

Go back to wireshark, stop capture.
##0x03 What's on hands?
`1.22.0	host	0xffffffff	USB	813	GET DESCRIPTOR Response CONFIGURATION`  
includes `VIDEO CONTROL INTERFACE DESCRIPTOR [Extension Unit] (Entity 4)`  
includes `guid: 1229a78c-47b4-4094-b0ce-db07386fb938`
Now we got 

Fuck I gave up, you just throw all these to AI and he'll tell you

### Extension Unit info
- Unit ID: 0x04
- GUID: {1229a78c-47b4-4094-b0ce-db07386fb938}
- Selector 0x0a (8 bytes)
- Selector 0x0b (8 bytes)

### Extension Unit 0x04 uses a dual-control register access protocol:
- **Selector 0x0a** = Command/address register (SET_CUR sets target register address+length, or 0xff executes command)
- **Selector 0x0b** = Data register (GET_CUR reads data, SET_CUR writes data)

### Phase 1: Initial Register Reads (Frame 3005-3051)

| # | Frame | Line | Direction | Sel | Data (hex) | Response (hex) | Description |
|---|-------|------|-----------|-----|------------|----------------|-------------|
| 1 | 3005/3006 | 28381 | GET_CUR | 0x0a | - | `00 08 00 bb bb bb bb bb` | Read current value of 0x0a |
| 2 | 3007/3008 | 28534 | SET_CUR | 0x0a | `ff 00 00 00 00 00 00 00` | ACK | Reset/initialization command |
| 3 | 3009/3010 | 28691 | SET_CUR | 0x0a | `00 cd 00 00 08 00 00 00` | ACK | Set register address: 0xcd00, len=8 |
| 4 | 3011/3012 | 28848 | GET_CUR | 0x0b | - | `da 0b 75 58 22 42 10 10` | Read 0xcd00: Device ID |
| 5 | 3013/3014 | 29001 | SET_CUR | 0x0a | `00 cd 01 00 08 00 00 00` | ACK | Set register address: 0xcd01 |
| 6 | 3015/3016 | 29158 | GET_CUR | 0x0b | - | `07 12 01 c0 00 00 01 00` | Read 0xcd01: Hardware version |
| 7 | 3017/3018 | 29311 | SET_CUR | 0x0a | `00 cd 02 00 08 00 00 00` | ACK | Set register address: 0xcd02 |
| 8 | 3019/3020 | 29468 | GET_CUR | 0x0b | - | `30 38 2f 31 38 2f 31 35` | Read 0xcd02: Firmware date 2015-08-18 |
| 9 | 3021/3022 | 29621 | SET_CUR | 0x0a | `00 cd 03 00 08 00 00 00` | ACK | Set register address: 0xcd03 |
| 10 | 3023/3024 | 29778 | GET_CUR | 0x0b | - | `bb bb bb bb bb bb bb bb` | Read 0xcd03: Uninitialized |
| 11 | 3025/3026 | 29931 | SET_CUR | 0x0a | `00 cd 04 00 08 00 00 00` | ACK | Set register address: 0xcd04 |
| 12 | 3027/3028 | 30088 | GET_CUR | 0x0b | - | `01 00 10 01 00 00 00 00` | Read 0xcd04: Configuration data |
| 13 | 3029/3030 | 30241 | SET_CUR | 0x0a | `00 cd 05 00 08 00 00 00` | ACK | Set register address: 0xcd05 |
| 14 | 3031/3032 | 30398 | GET_CUR | 0x0b | - | `bb bb bb bb bb bb bb bb` | Read 0xcd05: Uninitialized |
| 15 | 3033/3034 | 30551 | SET_CUR | 0x0a | `00 cd 06 00 08 00 00 00` | ACK | Set register address: 0xcd06 |
| 16 | 3035/3036 | 30708 | GET_CUR | 0x0b | - | `bb bb bb bb bb bb bb bb` | Read 0xcd06: Uninitialized |
| 17 | 3037/3038 | 30861 | SET_CUR | 0x0a | `00 cd 07 00 08 00 00 00` | ACK | Set register address: 0xcd07 |
| 18 | 3039/3040 | 31018 | GET_CUR | 0x0b | - | `bb bb bb bb bb bb bb bb` | Read 0xcd07: Uninitialized |
| 19 | 3041/3042 | 31171 | SET_CUR | 0x0a | `00 cd 08 00 08 00 00 00` | ACK | Set register address: 0xcd08 |
| 20 | 3043/3044 | 31328 | GET_CUR | 0x0b | - | `11 80 01 10 22 42 81 81` | Read 0xcd08: Extended capabilities |
| 21 | 3045/3046 | 31481 | SET_CUR | 0x0a | `00 cd 09 00 08 00 00 00` | ACK | Set register address: 0xcd09 |
| 22 | 3047/3048 | 31638 | GET_CUR | 0x0b | - | `bb bb bb bb bb bb bb bb` | Read 0xcd09: Uninitialized |
| 23 | 3049/3050 | 31791 | SET_CUR | 0x0a | `00 9f 00 00 01 00 00 00` | ACK | Set register address: 0x9f00, len=1 |
| 24 | 3051/3052 | 31948 | SET_CUR | 0x0b | `01 00 00 00 00 00 00 00` | ACK | **Write 0x01 to 0x9f00 (Enable IR)** |

### Phase 2: Second Configuration Sequence (Frame 3062-3083)

| # | Frame | Line | Direction | Sel | Data (hex) | Response (hex) | Description |
|---|-------|------|-----------|-----|------------|----------------|-------------|
| 27 | 3062/3063 | 32480 | SET_CUR | 0x0a | `00 d0 00 00 08 00 00 00` | ACK | Set register address: 0xd000, len=8 |
| 28 | 3064/3067 | 32637 | GET_CUR | 0x0b | - | `52 45 41 4c 53 49 4c 00` | Read 0xd000: "REALSIL\0" (chip vendor) |
| 29 | 3068/3069 | 32790 | SET_CUR | 0x0a | `00 9f 00 00 01 00 00 00` | ACK | Set register address: 0x9f00 |
| 30 | 3070/3071 | 32947 | SET_CUR | 0x0b | `01 00 00 00 00 00 00 00` | ACK | Write 0x01 to 0x9f00 |
| 31 | 3072/3073 | 33104 | SET_CUR | 0x0a | `00 fb 00 00 06 00 00 00` | ACK | Set register address: 0xfb00, len=6 |
| 32 | 3074/3075 | 33261 | SET_CUR | 0x0a | `ff 00 00 00 00 00 00 00` | ACK | Execute command: 0xff (trigger firmware command) |
| 33 | 3076/3077 | 33418 | SET_CUR | 0x0a | `00 fb 00 00 05 00 00 00` | ACK | Set register address: 0xfb00, len=5 |
| 34 | 3078/3079 | 33575 | GET_CUR | 0x0b | - | `80 02 e0 01 11` | Read 0xfb00: **w=640, h=480, mode=0x11** |
| 35 | 3080/3081 | 33728 | SET_CUR | 0x0a | `00 c2 21 fe 02 00 00 00` | ACK | Set register address: 0xc221fe, len=2 |
| 36 | 3082/3083 | 33885 | GET_CUR | 0x0b | - | `10 10` | Read 0xc221fe: Sensor configuration |

### Phase 3: Final Pre-Stream Configuration (Frame 3314-3370)

Repeat the configuration sequence from Phase 2:

| # | Frame | Line | Direction | Sel | Data (hex) | Response (hex) | Description |
|---|-------|------|-----------|-----|------------|----------------|-------------|
| 39 | 3314/3315 | 34940 | SET_CUR | 0x0a | `00 d0 00 00 08 00 00 00` | ACK | Set register address: 0xd000 |
| 40 | 3316/3317 | 35097 | GET_CUR | 0x0b | - | `52 45 41 4c 53 49 4c 00` | Read "REALSIL\0" |
| 41 | 3318/3319 | 35250 | SET_CUR | 0x0a | `00 9f 00 00 01 00 00 00` | ACK | Set register address: 0x9f00 |
| 42 | 3320/3321 | 35407 | SET_CUR | 0x0b | `01 00 00 00 00 00 00 00` | ACK | Write 0x01 to 0x9f00 |
| 43 | 3322/3323 | 35564 | SET_CUR | 0x0a | `00 fb 00 00 06 00 00 00` | ACK | Set register address: 0xfb00, len=6 |
| 44 | 3324/3325 | 35721 | SET_CUR | 0x0a | `ff 00 00 00 00 00 00 00` | ACK | Execute command: 0xff |
| 45 | 3326/3327 | 35878 | SET_CUR | 0x0a | `00 fb 00 00 05 00 00 00` | ACK | Set register address: 0xfb00, len=5 |
| 46 | 3328/3329 | 36035 | GET_CUR | 0x0b | - | `80 02 e0 01 11` | Read w=640, h=480 |
| 47 | 3330/3331 | 36188 | SET_CUR | 0x0a | `00 c2 21 fe 02 00 00 00` | ACK | Set register address: 0xc221fe |
| 48 | 3332/3333 | 36345 | GET_CUR | 0x0b | - | `10 10` | Read sensor configuration |

### Phase 4: IR Enable + Video Stream Start (Frame 3349-3379)

| # | Frame | Line | Direction | Sel | Data (hex) | Response (hex) | Description |
|---|-------|------|-----------|-----|------------|----------------|-------------|
| 49 | 3349/3350 | 36498 | SET_CUR | 0x0a | `00 9f 00 00 01 00 00 00` | ACK | Set register address: 0x9f00 |
| 50 | 3351/3352 | 36655 | SET_CUR | 0x0b | `01 00 00 00 00 00 00 00` | ACK | Write 0x01 to 0x9f00 |
| 51 | 3353/3354 | 36812 | SET_CUR | 0x0a | `00 fb 00 00 06 00 00 00` | ACK | Set register address: 0xfb00, len=6 |
| 52 | 3355/3356 | 36969 | SET_CUR | 0x0a | `ff 00 00 00 00 00 00 00` | ACK | Execute command: 0xff |
| 53 | 3357/3358 | 37126 | SET_CUR | 0x0a | `ff 00 00 00 00 00 00 00` | ACK | Execute command: 0xff (second time) |
| 54 | 3359/3360 | 37283 | SET_CUR | 0x0a | `00 fb 00 00 05 00 00 00` | ACK | Set register address: 0xfb00, len=5 |
| 55 | 3361/3362 | 37440 | GET_CUR | 0x0b | - | `80 02 e0 01 11` | Read w=640, h=480 |
| 56 | 3363/3364 | 37593 | SET_CUR | 0x0a | `00 fb 00 00 05 00 00 00` | ACK | Re-read |
| 57 | 3365/3366 | 37750 | GET_CUR | 0x0b | - | `80 02 e0 01 11` | Confirm w=640, h=480 |
| 58 | 3367/3368 | 37903 | SET_CUR | 0x0a | `00 9f 00 00 01 00 00 00` | ACK | Set register address: 0x9f00 |
| 59 | 3369/3370 | 38060 | SET_CUR | 0x0b | `00 00 00 00 00 00 00 00` | ACK | **Write 0x00 to 0x9f00 (disable before stream)** |
| 60 | 3371/3372 | 38217 | SET_CUR | VS_PROBE | `00 00 02 01 15 16 05 00...` | ACK | **VS_PROBE_CONTROL** |
| 61 | 3373/3374 | 38393 | GET_CUR | VS_PROBE | - | `00 00 02 01 15 16 05 00...` | VS_PROBE response |
| 62 | 3375/3376 | 38563 | SET_CUR | VS_COMMIT | `00 00 02 01 15 16 05 00...` | ACK | **VS_COMMIT_CONTROL** |
| 63 | 3377/3378 | ~38730 | SET_INTERFACE | - | Intf=1, Alt=7 | ACK | **Activate stream alt setting** |
| 64 | **3379** | **38884** | - | - | (isochronous) | - | **IR video data starts** |

(todo)
Basiclly you only need send 

0x0a     │ ff 00 00 00 00 00 00 00  
0x0a     │ 00 fb 00 00 05 00 00 00  
0x0b     │ 00 9f 00 00 01 00 00 00  
0x0a     │ 00 9f 00 00 01 00 00 00  
0x0b     │ 00 00 00 00 00 00 00 00  

to activate 640x480 YUV420 IR stream.  
Beware application should not touch format, otherwise camera will fall back to RGB.

(todo)

eXtention Unit commands was a standard. for viewers, you don't really need to [read it](https://www.kernel.org/doc/html/latest/userspace-api/media/drivers/uvcvideo.html#extension-unit-xu-support).

(ending todo)
