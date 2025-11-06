*Tài liệu được biên soạn bởi **TIEU CHI** — 2025*  
# Viết driver đơn giản cho Linux kernel
Hướng dẫn viết driver điều khiển LED cơ bản đâuf tiên cho kernel
- in log mỗi khi insert hoặc remove module
- Driver cấu hình chân GPIO để điều khiển LED
- Viết app để điều đọc ghi driver điều khiển LED

Có thể xem qua các driver có sẵn trong linux-kernel
```bash
cd ~/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/board-support/ti-linux-kernel-6.1.119+gitAUTOINC+c490f4c0fe-ti/drivers
```
## Tạo driver cơ bản đầu tiên có thể hiện log khi insert và remove module
**Tạo project hello-kernel**
**Tạo file hello-kernel.c**
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void)
{
    printk("[HELLO KERNEL DRIVER]: module loaded\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk("[HELLO KERNEL]: module exited\n");
}

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Tieu Chi");
MODULE_DESCRIPTION("Hello kernel demo");
module_init(hello_init);
module_exit(hello_exit);
```

**Tạo một Makefile**
```Makefile
obj-m := Hello_kernel.o 
KERNELDIR=/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/board-support/ti-linux-kernel-6.1.119+gitAUTOINC+c490f4c0fe-ti
CURRENTDIR=$(shell pwd)
CROSS=/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/external-toolchain-dir/arm-gnu-toolchain-11.3.rel1-x86_64-arm-none-linux-gnueabihf/bin/arm-none-linux-gnueabihf-

all:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENTDIR) ARCH=arm CROSS_COMPILE=$(CROSS) modules
clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENTDIR) ARCH=arm CROSS_COMPILE=$(CROSS) clean
```
Viết 2 file vậy là xong
**Trên host PC**
*Build module hello-kernel.ko*
```bash
make
```
*Copy file module xuống BBB*
```bash
scp hello-kernel.ko root@192.168.8.8:/home/root
```
**Trên board BeagleBone**
*Thêm module vào kernel*
```bash
insmod Hello_kernel.ko
```
*Để remove module*
```bash
rmmod Hello_kernel.ko
```
*Để xem log mội khi insert/ remove*
```bash
dmesg
```
bạn sẽ thấy log 
```
[HELLO KERNEL DRIVER]: this module is loaded
[HELLO KERNEL]:module exited
```
To write driver for kernel
cd ti-sdk -> boardsupport -> ti-kernel -> include -> linux -> init.h

search for "linux/init.h"
-> the file have funtion use for init

## Fix magic version error
```
root@am335x-evm:~# insmod hello-kernel.ko                                                                             
[   93.497405] hello_kernel: version magic '6.1.119 preempt mod_unload ARMv7 p2v8 ' should be '6.1.119-ti-gc490f4c0fe'
insmod: ERROR: could not insert module hello-kernel.ko: Invalid module format        
```
Lỗi này là do module nạp vào khác version với zImage (version được boot)
Bạn cần build lại zImage và boot lại đúng file
- Check
ti-sdk -> boardsupport -> built-image
Nếu bạn thấy file zImage thì OK, nếu không thì build lại
```bash
cd ti-processor-sdk-linux-am335x-evm-09.03.05.02/
make linux
make linux_install
```
Kiểm tra lại trong built-image, sẽ thấy file zImage
Copy zImage to tftpboot
cd ti-sdk -> boardsupport -> built-image
```bash
cp zImage /tftpboot/
```
Kiểm tra trong */tftpboot*, thấy file zImage là OK

Sửa lại file setupBoard.minicom để boot zImage
```bash
sudo vim setupBoard.minicom
```
```
send "setenv bootfile zImage"
send setenv bootcmd 'run findfdt; run init_console; setenv autoload no;tftp ${loadaddr} zImage; tftp ${fdtaddr} ${fdtfile}; bootz ${loadaddr} - ${fdtaddr}'
```
**OK, now boot image again**
Insert module lại, thấy log dưới này là OK
```
root@am335x-evm:~# insmod hello-kernel.ko                                                                             
[  127.969741] hello_kernel: loading out-of-tree module taints kernel.                                                
[  127.976554] [HELLO KERNEL DRIVER]: this module loaded     
```
**Khi copy file hello-kernel vào BBB, nếu xuất hiện lỗi**
```
Add correct host key to home/tieuchi/.shh/known_host
```
Đang yêu cầu host key trong file Known_host, xóa luôn file này là được
```bash
rm home/tieuchi/.shh/known_host
```
___
## Tạo device trong /sys/class
*Chúng ta cần tạo thư mục hello-kernel trong /sys/class*
Sau khi khởi tạo module xong, chúng ta cần tạo tiếp một class, và class này mặc định nằm trong thư mục sys trên BBB
```c
#include <linux/device.h>
```
Thêm thư viên device.h vào hello-kernel.c
Trong file *device.h* có include *class.h*
*class.h* là thư viện cung cấp hàm để tạo class

**Update hello-kernel.c to create class**
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/device.h>


struct class* hello_kernel_class;

static int __init hello_init(void)
{

    printk("[HELLO KERNEL DRIVER]: module loaded\n");
    hello_kernel_class = class_create (THIS_MODULE,"hello-kernel");
    if (IS_ERR(hello_kernel_class)) {
        printk("[HELLO KERNEL DRIVER]: create hello-kernel class failed");
        return -1;
    }
    return 0;
}

static void __exit hello_exit(void)
{
    printk("[HELLO KERNEL]: module exited\n");
    class_destroy(hello_kernel_class);
}

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Tieu Chi");
MODULE_DESCRIPTION("Hello kernel demo");
module_init(hello_init);
module_exit(hello_exit);
```
**Trên host PC**
```bash
make
scp hello-kernel.ko root@192.168.8.8:/home/root
```
**Trên BBB**
```bash
cd /sys/class
 ```
 Sẽ thấy trong class có thư mục hello-kernel
___
## Tạo một file file test ở trong thư mục *hello-kernel*
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/device.h>
#include <linux/kdev_t.h>

struct class* hello_kernel_class;
struct device* test_device;

static int __init hello_init(void)
{

    printk("[HELLO KERNEL DRIVER]: module loaded\n");
    hello_kernel_class = class_create (THIS_MODULE,"hello-kernel");
    if (IS_ERR(hello_kernel_class)) {
        printk("[HELLO KERNEL DRIVER]: create hello-kernel class failed\n");
        return -1;
    }
    test_device = device_create(hello_kernel_class,NULL, MKDEV(0,0), NULL, "test");
    if (IS_ERR(test_device)) {
        printk("[HELLO KERNEL DRIVER]: create test device failed\n");
        class_destroy(hello_kernel_class);
        return -1;

    }
    return 0;
}

static void __exit hello_exit(void)
{
    printk("[HELLO KERNEL]: module exited\n");
    device_destroy(hello_kernel_class, MKDEV(0,0));
    class_destroy(hello_kernel_class);
}

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Tieu Chi");
MODULE_DESCRIPTION("Hello kernel demo");
module_init(hello_init);
module_exit(hello_exit);
```
## Tạo thư mục device0
Tạo thư mục device0 trong class hello-kernel, rồi cho file test vào trong device0. Sau đó có thể đọc ghi vào file test
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/device.h>
#include <linux/kdev_t.h>


struct class* hello_kernel_class;
struct device* test_device;

ssize_t test_show(struct device *dev, struct device_attribute *attr, char *buf) 
{
    sprintf(buf, "hello\n");
    printk("[HELLO KERNEL DRIVER]: you just read /sys/class/hello-kernel/device0/test \n");
    return 6;
}

ssize_t test_store(struct device *dev, struct device_attribute *attr, const char *buf, size_t count) 
{
    printk("[HELLO KERNEL DRIVER]: you just write [%s] to /sys/class/hello-kernel/device0/test \n", buf);
    return count;
}

DEVICE_ATTR_RW(test);

static int __init hello_init(void)
{

    printk("[HELLO KERNEL DRIVER]: module loaded\n");
    hello_kernel_class = class_create (THIS_MODULE,"hello-kernel");
    if (IS_ERR(hello_kernel_class)) {
        printk("[HELLO KERNEL DRIVER]: create hello-kernel class failed\n");
        return -1;
    }

    test_device = device_create(hello_kernel_class,NULL, MKDEV(0,0), NULL, "device0");
    if (IS_ERR(test_device)) {
        printk("[HELLO KERNEL DRIVER]: create device0 failed \n");
        class_destroy(hello_kernel_class);
        return -1;

    }

    int ret = device_create_file(test_device, &dev_attr_test);
    if (ret != 0) {
        printk("[HELLO KERNEL DRIVER]: create test file failed \n");
        device_destroy(hello_kernel_class, MKDEV(0,0));
        class_destroy(hello_kernel_class);
    }
    return 0;

}

static void __exit hello_exit(void)
{
    printk("[HELLO KERNEL]: module exited\n");
    device_remove_file(test_device, &dev_attr_test);
    device_destroy(hello_kernel_class, MKDEV(0,0));
    class_destroy(hello_kernel_class);
}

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Tieu Chi");
MODULE_DESCRIPTION("Hello kernel demo");
module_init(hello_init);
module_exit(hello_exit);
```
## Cập nhật Makefile 
Cập nhật Make file để quá làm nhanh bước copy module xuống bo BBB
**Update Makefile**
```Makefile
obj-m := hello-kernel.o 
KERNELDIR=/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/board-support/ti-linux-kernel-6.1.119+gitAUTOINC+c490f4c0fe-ti
CURRENTDIR=$(shell pwd)
CROSS=/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/external-toolchain-dir/arm-gnu-toolchain-11.3.rel1-x86_64-arm-none-linux-gnueabihf/bin/arm-none-linux-gnueabihf-

all:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENTDIR) ARCH=arm CROSS_COMPILE=$(CROSS) modules
install:
	scp hello-kernel.ko root@192.168.8.8:/home/root
clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENTDIR) ARCH=arm CROSS_COMPILE=$(CROSS) clean
help:
	$(MAKE) -C $(KERNELDIR) help
```
**Trên host PC**
```bash
make
make install
```
**Trên BBB**
Thử đọc file test
```bash
cd /sys/class/hello-kernel/device0
cat test
dmesg
```
Log hiện như thế này là OK
```
	[ 6312.502667] [HELLO KERNEL DRIVER]: you just read /sys/class/hello-kernel/dev 
```
Thử ghi gì đó vào file test
```bash
echo blabla>test
dmesg
```
Log hiện
```
[ 6379.523209] [HELLO KERNEL DRIVER]: you just write [blabla
```

> © 2025_TIEUCHI  



