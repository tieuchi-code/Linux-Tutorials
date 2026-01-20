# Device tree 

Device tree là hardware map giúp kernel biết cần cấu hình phần cứng như thế nào (ghi giá trị nào vào thanh ghi), device tree còn giúp các driver có thể được sử dụng trên nhiều dòng chip khác nhau (một driver chi cần được enable trong device tree là có thể chạy được trên chip đó)


Chúng ta sẽ học cách viết một **driver** và enable driver đó trong **device tree**

- file **.dts**: device tree source - đây là file source code mô tả phần cứng cho kernel
- file **.dtc**: device tree compiler
- file **.dtb**: device tree blob - đây là file được biên dịch từ .dts

Tìm file **device tree source** trong TI-SDK

-> cd ti-sdk -> boardsupport -> ti-linux-kernel -> arch -> arm -> boot -> dts -> am335x-boneblack.dts
___
Đầu tiên sẽ viết code để enable driver trong file .dts

```c
#include "am33xx.dtsi"
#include "am335x-bone-common.dtsi"
#include "am335x-boneblack-common.dtsi"
#include "am335x-boneblack-hdmi.dtsi"

/ {
	model = "TI AM335x BeagleBone Black";
	compatible = "ti,am335x-bone-black", "ti,am335x-bone", "ti,am33xx";
	hello_led {
		compatible = "hello-led";
		gpios = <&gpio1 28 GPIO_ACTIVE_HIGH>;
	};
};
```

Sau đó là viết driver điều khiển chân GPIO
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/device.h>
#include <linux/kdev_t.h>
#include <linux/mutex.h>
#include <linux/gpio.h>
#include <linux/string.h>

#include <linux/gpio/consumer.h>
#include <linux/of_gpio.h>
#include <linux/of.h>

#define GPIO_NUM 28

struct class* hello_kernel_class;
struct device* test_device;

DEFINE_MUTEX(test_lock);

typedef enum {
    LED_OFF, LED_ON
} led_state_t;

led_state_t led_state = LED_OFF;

ssize_t test_show(struct device *dev, struct device_attribute *attr, char *buf) 
{
    mutex_lock(&test_lock);
    sprintf(buf,"led %s\n",(led_state == LED_ON)? "ON":"OFF");
    printk("[HELLO KERNEL DRIVER]: you just read /sys/class/FrDevTree/device0/test \n");
    mutex_unlock(&test_lock);
    return strlen(buf);
}

ssize_t test_store(struct device *dev, struct device_attribute *attr, const char *buf, size_t count) 
{
    mutex_lock(&test_lock);
    printk("[HELLO KERNEL DRIVER]: you just write [%s] to /sys/class/FrDevTree/device0/test \n", buf);
    if (strstr(buf, "on")) {
        led_state = LED_ON;
    }
    else if (strstr(buf,"off"))
    {
        led_state = LED_OFF;
    }
    gpio_set_value(GPIO_NUM,led_state);
    mutex_unlock(&test_lock);
    return count;
}

DEVICE_ATTR_RW(test);

#define HELLO_LED_DTB_PATH "/hello_led"
struct gpio_desc *g;

static int __init hello_init(void)
{
    struct device_node *np = of_find_node_by_path(HELLO_LED_DTB_PATH);
    if (!np) {
        printk("[HELLO KERNEL DRIVER]: Not found %s \n",HELLO_LED_DTB_PATH);
        return -1;
    }

    g = gpiod_get_from_of_node(np,"gpios",0, GPIOD_OUT_LOW, NULL);
    of_node_put(np);
    if (IS_ERR(g)) {
        printk("[HELLO KERNEL DRIVER]: gpiod_get_from_node failed: \n");
        return -1;
    }

    gpiod_set_value_cansleep(g,1);

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
    gpio_free(GPIO_NUM);
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
**make linux-dtbs** để build device tree
Sau đó copy file **am335x-boneblack.dtb** vào **tftpboot**
___
Sửa file **setupBoard.minicom** để boot đúng file **.dtb** lên board
```
expect {
    "=>"
}
send "setenv bootfile zImage"

expect {
    "=>"
}
send "setenv fdtfile am335x-boneblack.dtb"
```
___

## Module platform driver

Chúng ta đã học viết **driver module** và insmod trên board. Nhưng sẽ phải insmod lại tất cả driver mỗi lần khởi động
Chúng ta sẽ học **build-in driver** mà mình viết vào trong **kernel** luôn, khởi động lại là module có sẵn

Tìm source của **driver** trong **ti-sdk**

ti-sdk -> board-support -> ti-linux -> drivers

Trong **drivers** 
Tạo thư mục **mydriver** 
Trong **mydriver**, tạo **mydriver.c** và **Makefile**
Copy file **mydriver.c** đã viết vào **mydriver.c**
Thêm vào **Makefile** trong **mydriver**
```makefile
obj-y += mydriver.o
```

Trong **Makefile** của thư mục **drivers**, thêm vào cuối:
```makefile
obj-y += mydriver/
```
Build lại kernel & device tree, copy vào tftpboot rồi boot lên board

Nhưng cách này có thể không được, bởi vì device tree và driver không được boot lên cùng lúc nên boot driver trước thì không có device tree để nhận cmpatible


**This file will be build-in to kernel, use module platform driver**
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/device.h>
#include <linux/kdev_t.h>
#include <linux/mutex.h>
#include <linux/gpio.h>
#include <linux/string.h>

#include <linux/gpio/consumer.h>
#include <linux/of_gpio.h>
#include <linux/of.h>

#include <linux/platform_device.h>

#define GPIO_NUM 28

struct class* hello_kernel_class;
struct device* test_device;

DEFINE_MUTEX(test_lock);

typedef enum {
    LED_OFF, LED_ON
} led_state_t;

led_state_t led_state = LED_OFF;

ssize_t test_show(struct device *dev, struct device_attribute *attr, char *buf) 
{
    mutex_lock(&test_lock);
    sprintf(buf,"led %s\n",(led_state == LED_ON)? "ON":"OFF");
    printk("[HELLO KERNEL DRIVER]: you just read /sys/class/FrDevTree/device0/test \n");
    mutex_unlock(&test_lock);
    return strlen(buf);
}

ssize_t test_store(struct device *dev, struct device_attribute *attr, const char *buf, size_t count) 
{
    mutex_lock(&test_lock);
    printk("[HELLO KERNEL DRIVER]: you just write [%s] to /sys/class/FrDevTree/device0/test \n", buf);
    if (strstr(buf, "on")) {
        led_state = LED_ON;
    }
    else if (strstr(buf,"off"))
    {
        led_state = LED_OFF;
    }
    gpio_set_value(GPIO_NUM,led_state);
    mutex_unlock(&test_lock);
    return count;
}

DEVICE_ATTR_RW(test);

#define HELLO_LED_DTB_PATH "/hello_led"
struct gpio_desc *g;

int hello_driver_init(struct platform_device * pd)
{
    struct device_node *np = of_find_node_by_path(HELLO_LED_DTB_PATH);
    if (!np) {
        printk("[HELLO KERNEL DRIVER]: Not found %s \n",HELLO_LED_DTB_PATH);
        return -1;
    }

    g = gpiod_get_from_of_node(np,"gpios",0, GPIOD_OUT_LOW, NULL);
    of_node_put(np);
    if (IS_ERR(g)) {
        printk("[HELLO KERNEL DRIVER]: gpiod_get_from_node failed: \n");
        return -1;
    }

    gpiod_set_value_cansleep(g,1);

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
int hello_drvier_exit(struct platform_device * pd)
{
    printk("[HELLO KERNEL]: module exited\n");
    gpio_free(GPIO_NUM);
    device_remove_file(test_device, &dev_attr_test);
    device_destroy(hello_kernel_class, MKDEV(0,0));
    class_destroy(hello_kernel_class);
    return 0;
}
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Tieu Chi");
MODULE_DESCRIPTION("Hello kernel demo");
struct of_device_id hello_led_of_driver[] = {
    {.compatible = "hello-led"},
    {}
};
static struct platform_driver hello_led_driver = {
    .probe = hello_driver_init,
    .remove = hello_drvier_exit,
    .driver = {
        .name = "hello-kernel",
        .of_match_table = hello_led_of_driver,
    },
};
module_platform_driver(hello_led_driver);
```
--------------------------------------------------

copy this new code to hello-driver.c is hello-driver directory you made before

-> make linux
-> make linux_stage
-> copy zImage to tftpboot
 reboot
 
check: 
	dmesg | grep -i hello
	ls /sys/class
	
-----------------------------------------------------

this update below is for more convenient, you don,t need to remember the path 

//this file will be build-in to kernel, use module platform driver
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/device.h>
#include <linux/kdev_t.h>
#include <linux/mutex.h>
#include <linux/gpio.h>
#include <linux/string.h>

#include <linux/gpio/consumer.h>
#include <linux/of_gpio.h>
#include <linux/of.h>

#include <linux/platform_device.h>

#define GPIO_NUM 28

struct class* hello_kernel_class;
struct device* test_device;

DEFINE_MUTEX(test_lock);

typedef enum {
    LED_OFF, LED_ON
} led_state_t;

led_state_t led_state = LED_OFF;

ssize_t test_show(struct device *dev, struct device_attribute *attr, char *buf) 
{
    mutex_lock(&test_lock);
    sprintf(buf,"led %s\n",(led_state == LED_ON)? "ON":"OFF");
    printk("[HELLO KERNEL DRIVER]: you just read /sys/class/FrDevTree/device0/test \n");
    mutex_unlock(&test_lock);
    return strlen(buf);
}

ssize_t test_store(struct device *dev, struct device_attribute *attr, const char *buf, size_t count) 
{
    mutex_lock(&test_lock);
    printk("[HELLO KERNEL DRIVER]: you just write [%s] to /sys/class/FrDevTree/device0/test \n", buf);
    if (strstr(buf, "on")) {
        led_state = LED_ON;
    }
    else if (strstr(buf,"off"))
    {
        led_state = LED_OFF;
    }
    gpio_set_value(GPIO_NUM,led_state);
    mutex_unlock(&test_lock);
    return count;
}

DEVICE_ATTR_RW(test);

//#define HELLO_LED_DTB_PATH "/hello_led"
struct gpio_desc *g;

int hello_driver_init(struct platform_device * pd)
{
    //struct device_node *np = of_find_node_by_path(HELLO_LED_DTB_PATH);
    struct device_node *np = pd->dev.of_node;
    if (!np) {
        printk("[HELLO KERNEL DRIVER]: Not found  \n");
        return -1;
    }

    g = gpiod_get_from_of_node(np,"gpios",0, GPIOD_OUT_LOW, NULL);
    of_node_put(np);
    if (IS_ERR(g)) {
        printk("[HELLO KERNEL DRIVER]: gpiod_get_from_node failed: \n");
        return -1;
    }

    gpiod_set_value_cansleep(g,1);

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
int hello_drvier_exit(struct platform_device * pd)
{
    printk("[HELLO KERNEL]: module exited\n");
    gpio_free(GPIO_NUM);
    device_remove_file(test_device, &dev_attr_test);
    device_destroy(hello_kernel_class, MKDEV(0,0));
    class_destroy(hello_kernel_class);
    return 0;
}
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Tieu Chi");
MODULE_DESCRIPTION("Hello kernel demo");
struct of_device_id hello_led_of_driver[] = {
    {.compatible = "hello-led"},
    {}
};
static struct platform_driver hello_led_driver = {
    .probe = hello_driver_init,
    .remove = hello_drvier_exit,
    .driver = {
        .name = "hello-kernel",
        .of_match_table = hello_led_of_driver,
    },

};

module_platform_driver(hello_led_driver);
```









