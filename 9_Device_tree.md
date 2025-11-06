Devicd tree is hardware map, instruction for kernel to know what value is write to which register, so kernel driver don't have to write many driver for many board

IN this instruction, we will find out how to config and build and flash device tree to zImage

to find device tree in ti-sdk

-> cd ti-sdk -> boardsupport -> ti-linux-kernel -> arch -> arm -> boot -> dts -> am335x-boneblack.dts

file .dts: device tree source - describe board hardware for linux kernel
file .dtc: device tree compiler
file .dtb: device tree blob
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
    // int rets = gpio_request(28, "mygppio");
    // gpio_direction_output(28,0);
    // gpio_set_value(GPIO_NUM,0);
    // if (!rets) {
    // printk("[HELLO KERNEL DRIVER]: gpio %d requested successfully\n", GPIO_NUM);
    // }
    // else if (rets) {
    //     printk("*[HELLO KERNEL DRIVER]: can not request gpio1 - 28 ");
    //     return -1;
    // }

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


cd ti-sdk -> makerules -> Makefile_linux-dtbs

-> cd ti-sdk 
make linux-dtbs

this action will build all  device tree, we will choose one of them and import to board

make linux-dtbs_stage
the built device tree will be in /arch/arm/boot/dts/ and copied to /board-support/built-images/dtb/ti


cd bin -> setupBoard.minicom
```
end setenv rootpath '/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/targetNFS'

expect {
    "=>"
}
send "setenv bootfile zImage"

expect {
    "=>"
}
send "setenv fdtfile am335x-boneblack.dtb"

expect {
    "=>"
}
send "setenv bootargs 'console=ttyO0,115200n8 root=/dev/nfs nfsroot=192.168.8.9:/home/tieuchi/ti-processor-sdk-linux-am335x-evm-09.03.05.02/targetNFS,nolock,v3,tcp,rsize=4096,wsize=4096 rw ip=192.168.8.8:192.168.8.9:192.168.8.1:255.255.255.0::eth0'"
```

copy am335x-boneblack.dtb in boardsupport/dtb -> tftp server
cp board-support/built-images/dtb/ti/am335x-boneblack.dtb /tftpboot/

--------------------------------------------------

You have already known how to build kernel and your customized device tree, and insert your driver module, but you will have to insert your driver module everytime  you reboot, now we will learn to insert your driver module into kernel, so your driver will be boot with kernel

cd ti-sdk -> board-support -> ti-linux -> drivers
In "driver" -> create directory "hello-driver" 
->Copy and paste file "hello-kernel.c" to "hello-driver" 
In "driver" directory -> Makefile 
	add: 
		obj-y += hello-driver/
	at the end of Makefile
In "hello-driver" create Makefile
in Makefile -> obj-y += hello-kernel.o

make linux 
-> make linux_stage
(*this will copy zImage to /built-image/)
-> copy zImage to /tftpboot/
boot your board again
**But this may not work, because your device tree and kernel don't boot at the same time, so node may no find the gpio


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
// module_init(hello_init);
// module_exit(hello_exit);
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
// module_init(hello_init);
// module_exit(hello_exit);
```












