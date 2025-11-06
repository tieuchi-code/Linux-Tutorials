*Tài liệu được biên soạn bởi **TIEU CHI** — 2025*  
# Cài đặt SDK manager  
**Cài đặt Texas Instrument SDK manager để build Linux**  
https://software-dl.ti.com/processor-sdk-linux/esd/AM335X/09_03_05_02/exports/docs/linux/Overview/Download_and_Install_the_SDK.html  
<!-- © 2025 — Authored by TIEU CHI. -->  
**Hướng dẫn build SDK, u-boot, dtb,...**  
https://software-dl.ti.com/processor-sdk-linux/esd/AM335X/09_03_05_02/exports/docs/linux/Overview/Top_Level_Makefile.html  
> © 2025_TIEUCHI  
<!--
© 2025 — Authored by TIEU CHI  
Giữ attribution để tôn trọng quyền tác giả nhé.  
-->
___

# Build rootfs sử dụng Yocto  

**Hướng dẫn build từ TI**  
https://software-dl.ti.com/processor-sdk-linux/esd/AM335X/09_03_05_02/exports/docs/linux/Overview_Building_the_SDK.html  
**Lưu ý**:  
-Trong link hướng dẫn clone và build version mặc định (500GB). Nhưng chỉ nên build base version  
-Lần build đầu tiên sẽ mất vài giờ,  hoặc lấu hơn tuy cấu hình máy  
-Sẽ có rất nhiều warning, điều này là bình thường  
<!-- © 2025 — Authored by TIEU CHI. -->  
### Hoặc theo dõi và làm theo hướng dẫn dưới này  
*Vừa làm vừa xem hướng dẫn từ TI*
```bash
sudo apt-get update
sudo apt-get -f -y install \
  git build-essential diffstat texinfo gawk chrpath socat doxygen \
  dos2unix python3 bison flex libssl-dev u-boot-tools mono-devel \
  mono-complete curl python3-distutils repo pseudo python3-sphinx \
  g++-multilib libc6-dev-i386 jq git-lfs pigz zstd liblz4-tool \
  cpio file lz4 debianutils iputils-ping python3-git python3-jinja2 \
  python3-subunit locales libacl1 unzip gcc python3-pip python3-pexpect \
  xz-utils wget \
```
<!-- © 2025 — Authored by TIEU CHI. -->  
```bash
sudo locale-gen en_US.UTF-8
sudo dpkg-reconfigure dash
```
<!-- © 2025 — Authored by TIEU CHI. -->  

```bash
git clone https://git.ti.com/git/arago-project/oe-layersetup.git tisdk
cd tisdk
./oe-layertool-setup.sh -f configs/processor-sdk/processor-sdk-09.01.00-legacy-config.txt
cd build
. conf/setenv
MACHINE=am335x-evm bitbake -k tisdk-base-image
```
> © 2025_TIEUCHI  
<!--
© 2025 — Authored by TIEU CHI  
Giữ attribution để tôn trọng quyền tác giả.
-->
___

# Boot Image sử dụng Ethernet  
**Truy cập link để xem thông tin boot Image qua TFTP/NFS/Ethernet**
https://software-dl.ti.com/processor-sdk-linux/esd/AM335X/08_02_00_24/exports/docs/linux/Overview/Run_Setup_Scripts.html  

Using TFTP and NFS to boot image to BBB board  
**TFTP**: Kernel image
**NFS**: Root filesystem
  
**Theo hướng dẫn từ TI, sẽ có 2 cách để boot Image tới BBB**  

**Ethernet**  

	PC <-> BBB (ethernet)  

*Máy host và BBB phải được cấu hình IP tĩnh*  

**Wifi (DHCP/NFS)**  

	PC <-> router <-> BBB (ethernet)  

*Router wifi được xem như DHCP server và tự động cấu hình IP động cho cả máy tính và BBB.**  

**Trong hướng dẫn này sẽ dùng Ethernet**  

Phần cứng cần có:  
USB Mini cable (included with BBBlack)  
FTDI serial caple ( USB UART)  
> © 2025_TIEUCHI  
<!-- © 2025 — Authored by TIEU CHI. -->  

**Cấu hình IP tĩnh cho host ethernet**
*Vì trong hướng dẫn này sẽ dùng ethernet, không dùng wifi*
```bash
ifconfig
```
```
ifconfig
enp4s0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether d8:43:ae:09:7c:da  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 8451  bytes 723046 (723.0 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8451  bytes 723046 (723.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlo1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.123.45  netmask 255.255.255.0  broadcast 192.168.123.255
        inet6 2403:e200:115:497e:ffe4:f55:fae1:b06a  prefixlen 64  scopeid 0x0<global>
        inet6 2403:e200:115:497e:9c97:ba61:5866:6bb8  prefixlen 64  scopeid 0x0<global>
        inet6 fe80::1643:1458:3eee:63bd  prefixlen 64  scopeid 0x20<link>
        ether 04:ec:d8:65:d6:f5  txqueuelen 1000  (Ethernet)
        RX packets 184676  bytes 185952991 (185.9 MB)
```
<!-- © 2025 — Authored by TIEU CHI. -->  

*Host ethernet là enp4s0, chưa được set IP tĩnh*
*Có thể tùy ý không cần chính xác*
```
sudo ifconfig enp4s0 192.168.8.9
```
*ifconfig để kiểm tra lại*
```bash
ifconfig
```
```
enp4s0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.8.9  netmask 255.255.255.0  broadcast 192.168.8.255
        ether d8:43:ae:09:7c:da  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
*IP tĩnh cho ethernet đã được set*
<!-- © 2025 — Authored by TIEU CHI. -->  

Chạy file setup.sh  
```bash
cd ti-processor-sdk-linux-am335x-evm-09.03.05.02 
sudo ./setup.sh  
```
Nhập password  
*Script sẽ giúp thiết lập môi trường (minicom, telnet, TFTP, NFS...)*  
<!-- © 2025 — Authored by TIEU CHI. -->  

```
Are you running this script using sudo? The detected username is 'root'.
Verify and enter your Linux username below
[ root ] 
```
-> enter  
<!-- © 2025 — Authored by TIEU CHI. -->  

```
User 'root' is already apart of the 'dialout' group
Do you wish to install required host packages (Press (Y) to run, (n) to skip)? 
```
-> y -> enter  
*Nhận thấy bạn đang dùng quyền root*  
<!-- © 2025 — Authored by TIEU CHI. -->  

```
Do you wish to run nfs setup (Press (Y) to run, (n) to skip) ?
```
->y -> enter  
<!-- © 2025 — Authored by TIEU CHI. -->  

```
In which directory do you want to install the target filesystem?(if this directory does not exist it will be created)
[ /home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/targetNFS ] 
```
-> enter 
<!-- © 2025 — Authored by TIEU CHI. -->  

*Dùng thư mục mặc đinh, hoặc có thể chọn chỗ khác (không khuyến khích)*
```
Note! This command requires you to have administrator priviliges (sudo access) 
on your host.
Press return to continue
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/targetNFS already exists
(r) rename existing filesystem (o) overwrite existing filesystem (s) skip filesystem extraction
[r] 
```
*Thông báo này nghĩa là thư mục đã tồn tại. Nó sẽ không hiện trong lần đầu tiên bạn tạo. Bạn có thể đổi tên, ghi đè hoặc bỏ qua*

```
This step will set up the SDK to install binaries in to:
/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/targetNFS/home/root/am335x-evm

The files will be available from /home/root/am335x-evm on the target.

This setting can be changed later by editing Rules.make and changing the
EXEC_DIR or DESTDIR variable (depending on your SDK).

Press return to continue
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
This step will export your target filesystem for NFS access.

Note! This command requires you to have administrator priviliges (sudo access) 
on your host.
Press return to continue
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
Stopping nfs-kernel-server (via systemctl): nfs-kernel-server.service.
Starting nfs-kernel-server (via systemctl): nfs-kernel-server.service.
Do you wish to run tftp setup (Press (Y) to run, (n) to skip) ? 
```
-> y -> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
Which directory do you want to be your tftp root directory?(if this directory does not exist it will be created for you)
[ /tftpboot ] 
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
This step will set up the tftp server in the /tftpboot directory.

Note! This command requires you to have administrator priviliges (sudo access) 
on your host.
Press return to continue
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
This step will set up the tftp server in the /tftpboot directory.

Note! This command requires you to have administrator priviliges (sudo access) 
on your host.
Press return to continue
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
/tftpboot already exists, not creating..

/tftpboot/*Image-am335x-evm.bin already exists. The existing installed file can be renamed and saved under the new name.
(r) rename (o) overwrite (s) skip copy 
[r] 
```
*Thông báo này sẽ không xuất hiện nếu chạy lần đầu tiên.
Tuy nhiên nếu có thông báo này, sẽ ra rất rất nhiều câu hỏi, có thể chọn skip (s) hoặc overwrite (o) tất cả*
<!-- © 2025 — Authored by TIEU CHI. -->  

```
Restarting tftp server
Stopping xinetd (via systemctl): xinetd.service.
Starting xinetd (via systemctl): xinetd.service.
Do you wish to run minicom setup (Press (Y) to run, (n) to skip) ? 
```
-> y -> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
This step will set up minicom (serial communication application) for
SDK development
For boards that contain a USB-to-Serial converter on the board such as:
	* BeagleBone
	* Beaglebone Black
	* AM335x EVM-SK
	* AM57xx EVM
	* K2H, K2L, and K2E EVMs
the port used for minicom will be automatically detected. By default Ubuntu
will not recognize this device. Setup will add a udev rule to
/etc/udev/ so that from now on it will be recognized as soon as the board is
plugged in.
For other boards, the serial will defualt to /dev/ttyS0. Please update based
on your setup.

NOTE: If your using any of the above boards simply hit enter
and the correct port will be determined automatically at a
later step.  For all other boards select the serial port
that the board is connected to.
Which serial port do you want to use with minicom?
[ /dev/ttyS0 ] 
(**we are using ubuntu, so it's not ttyS0, we use ttyUSB0)
-> /dev/ttyUSB0

Configuration saved to /root/.minirc.dfl. You can change it further from inside
minicom, see the Software Development Guide for more information.

Do you wish to run uboot setup (Press (Y) to run, (n) to skip) ? 
```
-> y -> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
This step will set up the U-Boot variables for booting the EVM.
Autodetected the following ip address of your host, correct it if necessary
```
*Ở bước này u-boot sẽ kết nối tới host Ethernet, không dùng router wifi, nên phải config IP tĩnh*

<!-- © 2025 — Authored by TIEU CHI. -->  

Quay lại setup.sh terminal

```This step will set up the U-Boot variables for booting the EVM.
Autodetected the following ip address of your host, correct it if necessary
```
-> 192.268.8.9
*Nhập IP tĩnh bạn đã set*
```
Select Linux kernel location:
 1: TFTP
 2: SD card

[ 1 ] 
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

*Chọn TFTP để boot qua ethernet*
```
Select root file system location:
 1: NFS
 2: SD card

[ 1 ] 
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
Which kernel image do you want to boot from TFTP?
[ zImage-am335x-evm.bin ] 
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

*Chọn theo mặc định*
```
Would you like to create a minicom script with the above parameters (y/n)?
[ y ] 
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
Successfully wrote /home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/bin/setupBoard.minicom

No BeagleBone (Black) or StarterKit detected. Assuming
general purpose evm is being used. Is this correct?
(y/n) 
```
(I haven't connect the BBB board, so we're done here)
```
After successfully executing this script, your board will be set up. You will be 
able to connect to it by executing 'minicom -w' or if you prefer a windows host
you can set up Tera Term as explained in the Software Developer's Guide.
If you connect minicom or Tera Term and power cycle the board Linux will boot.

[ y ] 
```
-> enter
<!-- © 2025 — Authored by TIEU CHI. -->  

```
No BeagleBone (Black) or StarterKit detected. Assuming
general purpose evm is being used. Is this correct?
(y/n) 
```
*TIsdk yêu cầu dùng TFTP cable, nhưng tôi đang dùng USB UART cp2102*
-> cần thay đổi script
-> tìm từ khóa  "No BeagleBone"
```bash
grep -r 'No BeagleBone' ./bin/
```
```
./bin/setup-uboot-env.sh:		echo "No BeagleBone (Black) or StarterKit detected. Assuming"
```
-> script đang ở trong file "./bin/setup-uboot-env.sh"
<!-- © 2025 — Authored by TIEU CHI. -->  

```bash
vim ./bin/setup-uboot-env.sh
```
Tìm từ khóa "No BeagleBone"
```
/No BeagleBone
```
Thấy đoạn script dưới này
```
echo ""
echo "No BeagleBone (Black) or StarterKit detected. Assuming"
echo "general purpose evm is being used. Is this correct?"
read -p "(y/n) " validevm
echo ""
```
Đoạn này nằm trong hàm *check_for_beaglebone*

->tìm *check_for_beaglebone*

**First check if there is a rev A3 board which uses the custom VID/PID**
Tìm BeagleBone Black
```bash
sudo lsusb -vv -d 0403:6001 > /dev/null
```
```
    if [ "$?" = "0" ]
    then
        isBB="y"
        isBBBlack="y"
        return
     fi
```
-> Đoạn script sẽ tìm VID/PID cho TFTP
-> Thay đổi IP cho VID/PID 
<!-- © 2025 — Authored by TIEU CHI. -->  

**Mở một tab Terminal mới**
```bash
lsusb
```
```
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 007: ID 1c4f:0002 SiGma Micro Keyboard TRACER Gamma Ivory
Bus 001 Device 003: ID 5986:211b Acer, Inc HD Webcam
Bus 001 Device 002: ID 3151:3020 YICHIP Wireless Device
Bus 001 Device 008: ID 10c4:ea60 Silicon Labs CP210x UART Bridge
Bus 001 Device 005: ID 8087:0026 Intel Corp. AX201 Bluetooth
Bus 001 Device 006: ID 1d6b:0104 Linux Foundation Multifunction Composite Gadget
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
-> chip UART cp2102 có  VID/PID:  10c4:ea60
<!-- © 2025 — Authored by TIEU CHI. -->  

Quay lại hàm *check_for_beaglebone*

Comment dòng
```
# sudo lsusb -vv -d 0403:6001 > /dev/null
```
Thêm dòng này ngay bên dưới
```
sudo lsusb -vv -d 10c4:ea60 > /dev/null
```
Chạy lại file setup.sh
<!-- © 2025 — Authored by TIEU CHI. -->  

```
Successfully wrote /home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/bin/setupBoard.minicom

A BeagleBone (Black) or StarterKit board has been detected
Do you want to configure U-Boot for one of the boards mentioned
above? An answer of 'n' will configure U-Boot for the
General Purpose EVM instead
(y/n) 
```
-> y -> enter
<!-- © 2025 — Authored by TIEU CHI. -->  


Bây giờ đã tìm thấy
```
Detecting connection to board...

Unable to detect which port the board is connected to.
Please reconnect your board.
Press 'y' to attempt to detect your board again or press 'n' to continue...
(y/n)
```
Xuất hiện lỗi, bây giờ sẽ tìm lỗi
```
~/ti-processor-sdk-linux-am335x-evm-09.03.05.02$ 
grep -r 'Unable to detect which port the board is connected to.' bin/

bin/common.sh:			echo "Unable to detect which port the board is connected to."
bin/setup-uboot-env.sh:		    echo "Unable to detect which port the board is connected to."
```
Nằm ở đâu đó trong *bin/common.sh* hoặc *bin/setup-uboot-env.sh*

-> Tìm trong file *bin/setup-uboot-env.sh*
```bash
sudo vim bin/setup-uboot-env.sh
```
Tìm từ khóa "Unable to detect which port the board is connected to."
```
/Unable to detect which port the board is connected to.
```
```
    while [ yes ]
    do
            echo "Detecting connection to board..."
            loopCount=0
            port=`dmesg | grep FTDI | grep "tty" | tail -1 | grep "attached" |  awk '{ print $NF }'`
            while [ -z "$port" ] && [ "$loopCount" -ne "10" ]
            do
                    #count to 10 and timeout if no connection is found
                    loopCount=$((loopCount+1))

                    sleep 1
                    port=`dmesg  | grep FTDI | grep "tty" | tail -1 | grep "attached" |  awk '{ print $NF }'`
            done
```
(sctipt sẽ tự động tìm USB)
Thay đổi tên "FTDI" bằng "cp210x"
> © 2025_TIEUCHI  
<!-- © 2025 — Authored by TIEU CHI. -->  

**Chạy lại setup.sh**

Nhấn giữ nút cách
Nhấn nút reset để khởi động lại board

Log sẽ dừng lại ở dòng
```
"Press SPACE to abort autoboot in 0 seconds"
```

Phải sửa lại script để setup in bin/setupBoard.minicom
```bash
/ti-processor-sdk-linux-am335x-evm-09.03.05.02/bin$ ls
add-to-group.sh     setup-host-check.sh       setup-tftp.sh
common.sh           setup-minicom.sh          setup-uboot-env.sh
create-sdcard.sh    setup-package-install.sh
setupBoard.minicom  setup-targetfs-nfs.sh
```
<!-- © 2025 — Authored by TIEU CHI. -->  

```bash
sudo vim setupBoard.minicom
```
```
timeout 300
verbose on
expect {
    "stop autoboot:"
}
send " "
```
Đoạn script này có nghĩa
Chờ 300 giây để board reset
Sau khi reset sẽ chờ "stop autoboot:"
Rồi gửi " "
Thay cụm"stop autoboot:" bằng "Press SPACE to abort autoboot in 0 seconds"
<!-- © 2025 — Authored by TIEU CHI. -->  

**Không dùng SD card, nên xóa đoạn script này**
```
send "env default -f -a"

expect {
    "=>"
}
send "saveenv"

expect {
    "=>"
}
send "reset"

expect {
    "stop autoboot:"
}
send " "

expect {
    "=>"
}
```
<!-- © 2025 — Authored by TIEU CHI. -->  
DHCP method là dùng cho wifi, nên xóa đoạn dưới này luôn
```
send "setenv ip_method dhcp"

expect {
    "=>"
}
```
Set IP tĩnh cho BBB, thay số  x từ 2 -> 255, NHƯNG kHÔNG được giống với IP ethernet của máy host, còn khác số trước đó thì phải giống IP host ethernet
```
send "setenv ipaddr 192.168.8.x"

expect {
    "=>"
}
```
Set địa chỉ gateway các số  phía trước như 191.168.8 phải gióng IP host, số cuối phải bằng 1
<!-- © 2025 — Authored by TIEU CHI. -->  

```
send "setenv gatewayip 192.168.8.1"

expect {
    "=>"
}
```
Set net mask bắt buộc là 255.255.255.0
```
send "setenv netmask 255.255.255.0"

expect {
    "=>"
}
```

xóa từ *dhcp*
```
send setenv bootcmd 'run findfdt; run init_console; setenv autoload no;tftp ${loadaddr} zImage-am335x-evm.bin; tftp ${fdtaddr} ${fdtfile}; run netargs; bootz ${loadaddr} - ${fdtaddr}'
```
Xóa saveevn 
```
send "saveenv"

expect {
    "=>"
}
```
> © 2025_TIEUCHI  
<!-- © 2025 — Authored by TIEU CHI. -->  

**Bây giờ setup BBB**
```bash
sudo minicom -D /dev/ttyUSB0 -S setupBoard.minicom 
```
Nhấn giữ space key
Nhấn nút reset BBB board
<!-- © 2025 — Authored by TIEU CHI. -->  

Hiện log như dưới này là thành công
```
Press SPACE to abort autoboot in 0 seconds                                                                                      
=>setenv serverip 192.168.8.9                                                                                                   
=>setenv ipaddr 192.168.8.8                                                                                                     
=> setenv gatewayip 192.168.8.1                                                                                                 
=>setenv netmask 255.255.255.0                                                                                                  
=>setenv rootpath '/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/targetNFS'                                       
=>setenv bootfile zImage-am335x-evm.bin                                                                                         
=>setenv nfsopts 'nolock,v3,tcp,rsize=4096,wsize=4096'                                                                          
=>setenv bootcmd 'run findfdt; run init_console; setenv autoload no;tftp ${loadaddr} zImage-am335x-evm.bin; tftp ${fdtaddr} ${f'
=>boot                                                                                                                          
board_name=[A335BNLT] ...                                                                                                       
board_rev=[00C0] ...                                                                                                            
link up on port 0, speed 100, full duplex                                                                                       
Using cpsw device                                                                                                               
TFTP from server 192.168.8.9; our IP address is 192.168.8.8                                                                     
Filename 'zImage-am335x-evm.bin'.                                                                                               
Load address: 0x82000000                                                                                                        
Loading: #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #################################################################                                                      
         #########                                                                                                              
         4.7 MiB/s                                                                                                              
done                                                                                                                            
Bytes transferred = 7696896 (757200 hex)                                                                                        
link up on port 0, speed 100, full duplex                                                                                       
Using cpsw device                                                                                                               
TFTP from server 192.168.8.9; our IP address is 192.168.8.8                                                                     
Filename 'am335x-boneblack.dtb'.                                                                                                
Load address: 0x88000000                                                                                                        
Loading: ####################                                                                                                   
         2.9 MiB/s                                                                                                              
done                                                                                                                            
Bytes transferred = 98157 (17f6d hex)                                                                                           
## Flattened Device Tree blob at 88000000                                                                                       
   Booting using the fdt blob at 0x88000000                                                                                     
   Loading Device Tree to 8ffe5000, end 8fffff6c ... OK                                                                         
                                                                                                                                
Starting kernel ...                                                                                                             
                                                                                                                                
[    0.000000] Booting Linux on physical CPU 0x0                                                                                
[    0.000000] Linux version 6.1.119-ti-gc490f4c0fe51 (oe-user@oe-host) (arm-oe-linux-gnueabi-gcc (GCC) 11.5.0, GNU ld (GNU Bin4
[    0.000000] CPU: ARMv7 Processor [413fc082] revision 2 (ARMv7), cr=10c5387d                                                  
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache                                         
[    0.000000] OF: fdt: Machine model: TI AM335x BeagleBone Black                                                               
[    0.000000] Memory policy: Data cache writeback                                                                              
[    0.000000] efi: UEFI not found.                                                                                             
[    0.000000] cma: Reserved 64 MiB at 0x9b800000                                                                               
[    0.000000] Zone ranges:                                                                                                     
[    0.000000]   Normal   [mem 0x
...
```
Trong log sẽ thấy dòng
```
[    0.000000] Linux version 6.1.119-ti-gc490f4c0fe51 
```
*Có nghĩa BBB đang chạy version được boot từ host (version mới hơn version gốc trên board)*
> © 2025_TIEUCHI  
<!-- © 2025 — Authored by TIEU CHI. -->  

**Tuy nhiên, ta vẫn mắc kẹt lại dòng này**
```
[  246.494206] cpsw-switch 4a100000.switch eth0: Link is Up - 100Mbps/Full - flow control off             
```
*Chưa chạy được version hoàn toàn*
<!-- © 2025 — Authored by TIEU CHI. -->  

Tìm trong log
```
Kernel command line: console=ttyO0,115200n8 root=/dev/nfs nfsroot=192.168.8.9:/home/tieuchi/ti-processor-sdk-linp
```
Chạy sai *netarg* (net argument)

**Sửa file *setupBoard.minicom***
```
send setenv bootcmd 'run findfdt; run init_console; setenv autoload no;tftp ${loadaddr} zImage-am335x-evm.bin; tftp ${fdtaddr} ${fdtfile}; run netargs; bootz ${loadaddr} - ${fdtaddr}'
```
Xóa cụm *run netargs;*
<!-- © 2025 — Authored by TIEU CHI. -->  
Thêm những dòng này dưới *setenv bootfile*
```
send "setenv bootargs 'console=ttyO0,115200n8 root=/dev/nfs nfsroot=192.168.8.9:/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/targetNFS,nolock,v3,tcp,rsize=4096,wsize=4096 rw ip=192.168.8.8:192.168.8.9:192.168.8.1:255.255.255.0::eth0'"

expect {
    "=>"
}
```
> © 2025_TIEUCHI  
<!-- © 2025 — Authored by TIEU CHI. -->  

___

Log này hiện ra là hoàn toàn thành công rồi
```
  OK  ] Finished User Runtime Directory /run/user/1000.
         Starting User Manager for UID 1000...
 _____                    _____           _         _   
|  _  |___ ___ ___ ___   |  _  |___ ___  |_|___ ___| |_ 
|     |  _| .'| . | . |  |   __|  _| . | | | -_|  _|  _|
|__|__|_| |__,|_  |___|  |__|  |_| |___|_| |___|___|_|  
              |___|                    |___|            

Arago Project am335x-evm -

Arago 2023.10 am335x-evm -

am335x-evm login: ***************************************************************
***************************************************************
NOTICE: This file system contains the following GPL-3.0 packages:

NOTE: If the package is a dependency of another package you
      will be notified of the dependent packages.  You should
      use the --force-removal-of-dependent-packages option to
      also remove the dependent packages as well
***************************************************************
***************************************************************
```
CONGRATULATION
> © 2025_TIEUCHI  
> <!-- © 2025 — Authored by TIEU CHI. -->  
<!--
© 2025 — Authored by TIEU CHI  
Giữ attribution để tôn trọng quyền tác giả nhé.  
-->






















































