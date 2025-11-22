*Tài liệu được biên soạn bởi **TIEU CHI** — 2025*  
> © 2025_TIEUCHI  
### QT creator
*QTcreator is a framework to make GUI for embeded system, mobile app, desktop app (linux,window,macos)*

In this tutorial, we will make first, simple project for BeagleBone Black, then advance to use GUI to control GPIO
___
**First, to install QTcreator (use opensource QT) on Linux host**
```bash
sudo apt update
sudo apt install qtcreator -y
```
**To run QT creator:**
```bash
qtcreator 
```
**Before you  open qtcreator, source ti-sdk enviroment first**
```bash
tieuchi@tieuchi:~$ source ti-processor-sdk-linux-am335x-evm-09.03.05.02/linux-devkit/environment-setup
```
___
> © 2025_TIEUCHI  
<!-- © 2025 — Authored by TIEU CHI. -->  
**If you install QT with command and open source, then you the instruction below**
In QT
- tools -> option
- kits -> add -> beaglebone black device
- build device: local PC (default for desktop)
- manage -> host name: 192.168.8.8 (BBB IP) *this is device host name, so you input BBB IP address
- private key -> create new (*if you already have -> deploy key)
- test
- compiler:
	- C: C/arm/ti-sdk/linux..... (*if you source before, this will auto match for you)
	- C++: (*if you source before, this will auto match for you)
	
- Debuger: /home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/linux-devkit/sysroots/x86_64-arago-linux/usr/bin/arm-oe-linux-gnueabi/arm-oe-linux-gnueabi-gdb
	(*this is also auto match, if not, write it yourself)
- QT version: QT 5.15.7 (system)
	(*ti-sdk/linux-devkit/sysroots/x86_64-arago-linux/usr/bin/qmake)

___

**Create project**
- QT widget application
- name project
- qmake
- build for BBB
___

**Test GUI**
- insert label: hello BBB
- insert button
(*anything you want)
___
<!-- © 2025 — Authored by TIEU CHI. -->  
- build (it may have error, no problem)
- deploy (the button run)
*note the size of your screen, in this instruction is 800x600*
OK now see in your BBB, it may have a program with your project name
> © 2025_TIEUCHI  

If you do it until now, you may see error relate to library
```bash
-> lld <project_name>
```
**You don't have this library, the BBB library is different to the host library**
```
/lub/ld-linux.so.3 => /lib/ld-linux-armf.so.3
```
**You have to link library, use this cmd**
```bash
ln -s /lub/ld-linux.so.3 /lib/ld-linux-armf.so.3
```
___

**To run the program, you have to download realVNC on host PC**
```bash
sudo apt update
sudo apt install realvnc-vnc-viewer
```
___
**On BBB**
```bash
./<project_name> -platform vnc:size=800x600:port=5900
```

open VNC on host desktop

VNC: 192.168.8.8:5900


Youtube instruction:
https://www.youtube.com/watch?v=4c6MZ34kLhc

> © 2025_TIEUCHI  

<!--
© 2025 — Authored by TIEU CHI  
Giữ attribution để tôn trọng quyền tác giả nhé.  
-->


















	
	



















