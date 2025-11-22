*Tài liệu được biên soạn bởi **TIEU CHI** — 2025*  
> © 2025_TIEUCHI  
# Viết driver đơn giản cho Linux kernel
Hướng dẫn viết driver điều khiển LED cơ bản đâuf tiên cho kernel
- in log mỗi khi insert hoặc remove module
- Driver cấu hình chân GPIO để điều khiển LED
- Viết app để điều đọc ghi driver điều khiển LED

Có thể xem qua các driver có sẵn trong linux-kernel
```bash
cd ~/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/board-support/ti-linux-kernel-6.1.119+gitAUTOINC+c490f4c0fe-ti/drivers
```
<!-- © 2025 — Authored by TIEU CHI. -->  
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
<!-- © 2025 — Authored by TIEU CHI. -->  
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
> © 2025_TIEUCHI  
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
<!-- © 2025 — Authored by TIEU CHI. -->  
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
> © 2025_TIEUCHI  
## Tạo class trong sys
*Chúng ta cần tạo thư mục hello-kernel trong /sys/class*
Sau khi khởi tạo module xong, chúng ta cần tạo tiếp một class, và class này mặc định nằm trong thư mục sys trên BBB
```c
#include <linux/device.h>
```
Thêm thư viên device.h vào hello-kernel.c
Trong file *device.h* có include *class.h*
*class.h* là thư viện cung cấp hàm để tạo class
<!-- © 2025 — Authored by TIEU CHI. -->  
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
> © 2025_TIEUCHI  
## Tạo một deice ở trong class *hello-kernel*
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
> © 2025_TIEUCHI  
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
> © 2025_TIEUCHI  
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
___
> © 2025_TIEUCHI  
## Quy trình viết một driver cơ bản
Để  tạo một driver module đầu tiên, cần đăng kí module bằng các hàm
```c
module_init(hello_init);
module_exit(hello_exit);

```
Các hàm này ở trong thư viện
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
```
và trong *module.h*, *init.h*, tìm thấy các hàm 
```c
#define module_init(x)	__initcall(x);
#define module_exit(x)	__exitcall(x);
```
- Khi chạy lênh *insmod*, thì sẽ tìm và chạy hàm mà chúng ta đăng kí trong *module init*, khi này module init sẽ gọi tới hàm *__init hello_init(void)* 
- annotation *__init* báo cho kernel biết hàm này chỉ dùng khi load module, dùng xong tự động xóa RAM để tiết kiệm
```c
static int __init hello_init(void)
static void __exit hello_exit(void)
```
trong hàm *init* và *exit* này chỉ làm công việc là in log ra để kiểm tra môi trường build kernel có hoạt động tốt hay không 
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
> © 2025_TIEUCHI  
### Tạo một class
Thêm thư viện
```c
#include <linux/device.h>
```
Trong thư viện *device.h* có include *device/class.h*
Đây là thư viện chứa các hàm
```c
#define class_create(owner, name)		\
({						\
	static struct lock_class_key __key;	\
	__class_create(owner, name, &__key);	\
})
extern void class_destroy(struct class *cls);
```
<!-- © 2025 — Authored by TIEU CHI. -->  
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
```
___
> © 2025_TIEUCHI  
### Tạo một device ở trong class *hello-kernel* 
Trong thư viện *device.h* có hàm khởi tạo device, device này được tạo trong class
```c
device_create(struct class *cls, struct device *parent, dev_t devt,
	      void *drvdata, const char *fmt, ...);
```
- **struct class**: class đã tạo trước đó, deice này sẽ được thêm vào trong class (class là một nhóm các driver có chức năng giống nhau)
- **struct device**: luôn để NULL với driver ký tự thông thường
- **dev_t**: số major + minor, được tạo bằng:
    - **alloc_chrdev_region()** (kernel cấp phát)
    - hoặc **MKDEV(major, minor)**
      - Major = số đại diện cho driver.
      - Minor = số đại diện cho thiết bị con của driver.
- **void drvdata** : Con trỏ lưu dữ liệu riêng cho thiết bị.
- **const char fmt** : tên của node tạo trong /dev
<!-- © 2025 — Authored by TIEU CHI. -->  
***Trong hướng dẫn này chỉ tạo driver cơ bản ban đầu, không hỗ trợ điều khiển bất cứ thiết bị nào, cho nên:***
  - Không tạo số major + minor hợp lệ (kernel sẽ không có thiết bị để điều khiển).
  - Con trỏ lưu dữ liệu riêng của thiết bị sẽ đẻ NULL
```c
test_device = device_create(hello_kernel_class,NULL, MKDEV(0,0), NULL, "test");
```
___
Để sủ dụng **MKDEV** (để ghép 2 số major + minor lại thành một mã số) cần thêm thư viện
```c
#include <linux/kdev_t.h>
```
- Thư viện này trong Linux kernel được dùng để làm việc với device number (số major/minor) của thiết bị khi viết driver — đặc biệt là *character driver* và *block driver*.
- Trong thư viện này có các macro
```c
#define MAJOR(dev)	((unsigned int) ((dev) >> MINORBITS))
#define MINOR(dev)	((unsigned int) ((dev) & MINORMASK))
#define MKDEV(ma,mi)	(((ma) << MINORBITS) | (mi))
```
<!-- © 2025 — Authored by TIEU CHI. -->  
**Đoạn code để tạo deivce trong class**
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
> © 2025_TIEUCHI  
### Tạo device0 trong class và attribute file
- Trong một driver có thể điều khiển nhiều thiết bị con giống nhau, ví dụ trong device led thì có led0, led1, led2...
- Tạo một attribute file, một file đặc biệt được tạo bởi driver kernel để giao tiếp với phần cứng, giúp điều khiển phần cứng đơn giản bằng hàm store (ghi) hoặc show (ghi)
___
<!-- © 2025 — Authored by TIEU CHI. -->  
**Tạo device0 (device con duy nhất trong class này, nếu có nhiều device con thì sẽ tạo thêm một thư mục con device/device0)**
```c
test_device = device_create(hello_kernel_class,NULL, MKDEV(0,0), NULL, "device0");
```
**Trong *device.h* có các hàm để tạo file attribute**
```c
int device_create_file(struct device *device,
		       const struct device_attribute *entry);
void device_remove_file(struct device *dev,
			const struct device_attribute *attr);
```
- **test_device** : con trỏ tới device được tạo bằng **device_create()**
- **dev_attr_name** : struct attribute được tạo từ macro **DEVICE_ATTR_RW(_name)**
*Lưu ý: device_attribute phải có đúng cú pháp **dev_attr_name** (name là tên của file attribute)*
___
<!-- © 2025 — Authored by TIEU CHI. -->  
**DEVICE_ATTR_RW(_name)** :
Trong *device.h* có macro 
```c
#define DEVICE_ATTR_RW(_name) \
	struct device_attribute dev_attr_##_name = __ATTR_RW(_name)
```
Khi gọi DEVICE_ATTR_RW(_name), trong kernel sẽ tạo một struct
```c
struct device_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct device *dev, struct device_attribute *attr,
			char *buf);
	ssize_t (*store)(struct device *dev, struct device_attribute *attr,
			 const char *buf, size_t count);
};
```
- **struct attribute attr**: Đây là phần chung của sysfs attribute, chứa tên file (ví dụ "test") và quyền truy cập (0644, 0444, v.v.).
- **ssize_t (*show)(...)**: Con trỏ hàm dùng để đọc giá trị của attribute từ kernel ra buffer. Khi user chạy cat /sys/class/.../device0/test, kernel sẽ gọi hàm show.
- **ssize_t (*store)(...)**: Con trỏ hàm dùng để ghi giá trị từ user space vào kernel. Khi user chạy echo "on" > /sys/class/.../device0/test, kernel sẽ gọi hàm store.

Ví dụ:
```c
//Khi viết 
DEVICE_ATTR_RW(test);

//Kernel sẽ tạo
struct device_attribute dev_attr_test = {
    .attr = { .name = "test", .mode = 0664 },
    .show  = test_show,
    .store = test_store,
};
```

**Các hàm *show* và *store***
```c
ssize_t test_show(struct device *dev, struct device_attribute *attr, char *buf);
ssize_t test_store(struct device *dev, struct device_attribute *attr, const char *buf, size_t count);
```
- **struct device dev** : Con trỏ đến device chứa file attribute.
- **struct device_attribute attr** : Con trỏ đến attribute. Nếu bạn định nghĩa nhiều attribute, đây giúp xác định đang thao tác attribute nào.
- **char *buf (show)**: Buffer để viết dữ liệu ra cho user space. Bạn phải return số byte ghi vào buffer.
- **const char *buf (store)**: Buffer chứa dữ liệu user space ghi vào.
- **size_t count (store)**: Kích thước dữ liệu user ghi.
Khi viết driver này, các tham số của hàm *store* và *show* được truyền tự động nên có thể để nguyên như hàm mãu luôn (*dev* là device hiện tại, *attr* là attribute hiện tại, *buf* được tạo ra để lưu dữ liệu)
*Lưu ý: hàm store và show cũng phải ghi đúng cú pháp là name_store và name_show (name là tên file attribute)*
___
> © 2025_TIEUCHI  

**Code tạo đầy đủ class, device, attribute file**
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
___
> © 2025_TIEUCHI  
## Viết Makefile
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
- **obj-m** : chỉ ra danh sách các module kernel cần build. Ở đây, hello-kernel.o là module của chúng ta.
- **KERNELDIR** : Thư mục source kernel của board. Cần chỉ định để Makefile biết dùng header và makefile của kernel đúng phiên bản.
- **CURRENTDIR=$(shell pwd)**: chạy lệnh để có được vị trí hiện tại của file cần build
- **CROSS=/home/tieuchi/.../bin/arm-none-linux-gnueabihf-** : vị trí của cross compiler cho ARM

- **\$(MAKE) -C $(KERNELDIR)** :  Makefile của module gọi make trong thư mục kernel, có các biến môi trường đúng để build
  - **\$(MAKE)**: gọi make ở một thư mục khác
  - **-C** : change directory
  - **\$(KERNELDIR)** : vị trí của source linux kernel
- **M=$(CURRENTDIR)** : thư mục module ngoài kernel. Kernel sẽ build các module nằm trong thư mục này .c thành .ko
- **ARCH=arm** : bảo kernel build module cho ARM
- **CROSS_COMPILE=$(CROSS)** : Khi host là x86 nhưng target là ARM, phải dùng cross compiler. Kernel makefile sẽ dùng prefix này để gọi gcc: ví dụ arm-none-linux-gnueabihf-gcc.
- **modules** : target trong kernel makefile, báo kernel build tất cả module đã liệt kê trong obj-m.


> © 2025_TIEUCHI  
<!--
© 2025 — Authored by TIEU CHI  
Giữ attribution để tôn trọng quyền tác giả nhé.  
-->