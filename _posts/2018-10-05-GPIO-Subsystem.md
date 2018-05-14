---
layout: post

category: Linux device driver

comments: true
---
# A. GPIO Subsystem

GPIO là gì? GPIO là viết tắt của General Port Input output, các port này xử lý cả tín hiệu số cả vào lẫn ra. Khi hoạt động như một cổng vào, nó có thể được sử dụng để kết nối CPU với tín hiệu ON/OFF nhận được từ các công tắc (ví dụ các nút bấm trên router), hoặc đọc các tín hiệu số từ các sensor. Còn nếu hoạt động như một cổng ra, nó có thể truyền các tín hiệu điều khiển đến các thiết bị dựa trên kết quả của các lệnh thực thi trong CPU, chẳng hạn như điều khiển tín hiệu bật tắt của đèn LED (Cái này sẽ trình bày trong ví dụ). 
Các giá trị số vào/ra của GPIO chỉ có thể là 0 (LOW) hoặc 1(HIGH). 
Trong Linux kernel, để sử dụng GPIO, trước hết LKM cần phải giành lấy nó từ kernel, để trở thành chủ nhân của GPIO này, ngăn chặn việc các LKM khác truy cập vào cùng GPIO. Sau khi đã thành chủ sở hữu của GPIO, bạn có thể cài đặt chiều truyền tín hiệu số (Vào hoặc ra), thay đổi giá trị truyền vào đối với chiều ra, đọc giá trị nhận được đối với chiều vào, cấu hình các thứ linh tinh khác...

Có hai cách để làm việc với GPIO trong kernel, bao gồm:
-   Cách truyền thống (đã depreciate): biểu diễn các GPIO bằng các số int.
-   Cách mới: Biểu diễn bằng GPIO description.

## I. Legacy GPIO interface trong Linux.
Mặc dù cách này đã bị đánh dấu là depreciate nhưng nó vẫn là cách được biết đến rộng rãi nhất. Các GPIO port được định danh bằng các số integer. Các function cần thiết được include từ header:
{% highlight c %}
#include <linux/gpio.h>
{% endhighlight %}

### 1. Giành quyền sử dụng GPIO và cấu hình GPIO.
Bạn có thể chỉ định và giành quyền sử dụng của một GPIO bằng hàm <code>gpio_request()</code>:
{% highlight c %}
static int gpio_request(unsigned gpio, const char *label);
{% endhighlight %}

Tham số <code>unsigned gpio</code> chính là GPIO number, được biểu diễn dưới dạng số nguyên và label là tên sẽ được sử dụng bởi kernel trong sysfs. Hàm này sẽ trả về 0 nếu như việc request thành công, ngược lại nó sẽ trả về giá trị âm. Sau khi sử dụng xong GPIO, bạn nên giải phóng nó để những người khác có thể sử dụng:
{% highlight c %}
static int gpio_free(unsigned gpio);
{% endhighlight %}

Ngoài ra, bạn cũng có thể kiểm tra xem GPIO number có hợp lệ hay không trước khi request nó:
{% highlight c %}
static bool gpio_is_valid(int number);
{% endhighlight %}

Sau khi đã giành quyền sử dụng GPIO, bước tiếp theo cần làm là cài đặt chiều truyền tín hiệu số của nó, có 2 chiều: input và output được setup bằng hai hàm sau:
{% highlight c %}
static int gpio_direction_input(unsigned gpio);
static int gpio_direction_output(unsigned gpio, value);
{% endhighlight %}

Tham số <code>value</code> trong trường hợp gpio output sẽ là giá trị mặc định được gán cho GPIO port đấy. Các hàm này nên được dùng trong context có thể ngủ được, thông thường chúng ta sẽ gọi những hàm này ở trong phần probe của driver.

### 2. Sử dụng legacy gpio để đọc ghi tín hiệu.
Việc truy cập GPIO thường liên quan đến các atomic context, đặc biệt là trong một interrupt handler, các đoạn code trong context này phải đảm bảo rằng GPIO controller callback function sẽ không thể ngủ. Một controller driver được thiết kế tốt sẽ có khả năng thông báo với các drivers khác việc lời gọi hàm có thể ngủ hay khôn. Việc này có thể được kiểm tra bằng : <code>gpio_cansleep()</code>.

#### 2.1 Trong atomic context.
Một số GPIO controller, đặc biệt là trên các SoC, có thể truy cập và quản lý thông qua các thao tác đọc ghi bộ nhớ thông thường, do đó việc ngủ nghỉ là không cần thiết. Đối với những GPIO này, <code>gpio_cansleep()</code> luôn luôn trả về giá trị <code>false</code>, và bạn có thể đọc/ghi giá trị ngay trong IRQ Handler với <code>gpio_get_value()</code> và <code>gpio_set_value()</code>.

#### 2.2 Trong non-atomic context.
Các GPIO controllẻ được nối vào buses, chẳng hạn như SPI và I2C thì những function truy cập các bus này là có thể dẫn đến việc sleep, do đó <code>gpio_cansleep()</code> function sẽ luôn luôn trả về true. Trong trường hợp này, bạn không nên truy cập các GPIO này bên trong IRQ handled (thay vào đó, truy cập nó trong bottom half thông qua threaded irq). Hơn nữa, hãy sử dụng các hàm đọc ghi thay thế sau là <code>gpio_get_value_cansleep()</code> và <code>gpio_set_value_cansleep()</code>.

### 3. Ví dụ: sử dụng nút Reset đê bật tắt đèn LED trên Router wifi: Compex WPJ563.
Phần này mình sẽ ví dụ về việc sử dụng GPIO để thao tác với đèn LED và nút bấm, cũng như mapping từ GPIO Number sang IRQ number, ví dụ được thực hiện trên Board WPJ563, chạy firmware OpenWRT/LEDE.
Bạn có thể xem hardware specific của nó ở đây: https://www.compex.com.sg/product/wpj563/.
Dựa vào hardware specific, Board có 4 Đèn LED liên kết với các GPIO number: 
-   GPIO1   :   RSS1/EEPROM CLK
-   GPIO5   :   RSS2/EEPROM data
-   GPIO6   :   RSS3
-   GPIO7   :   RSS4/Diag

Và một button: GPIO2: RESET.
Ở ví dụ này mình sẽ dùng Button RESET trên để bật tắt các LED: RSS3 và RSS4. Module sẽ có tên là: wpj563_led

Trước hết, clone lede source ở đây: https://github.com/lede-project/source.git

Chúng ta sẽ tạo một package trong section kernel module để chứ module wpj563_led
Chuyển đến thư mục chứa section kernel và tạo thư mục: 
{% highlight shell %}
cd packages/kernel
mkdir wpj563-led
{% endhighlight %}

Bây giờ, bạn tạo một <code>Makefile</code> cho package mới, với nội dung như sau:
{% highlight shell %}
#copyright (C) 2018 Phi Nguyen
#
#

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=wpj563-led
PKG_VERSION:=0.1
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk

define KernelPackage/wpj563-led
  SUBMENU:=Oni Modules
  TITLE:=Simple GPIO Button driver by Oni
  FILES:=$(PKG_BUILD_DIR)/gpio_legacy.ko
  AUTOLOAD:=$(call AutoLoad,30,gpio_legacy,1)
  KCONFIG:=
endef

define KernelPackage/wpj563-led/description
	legacy gpio
endef

MAKE_OPTS:= \
        $(KERNEL_MAKE_FLAGS) \
        SUBDIRS="$(PKG_BUILD_DIR)"

define Build/Compile
        $(MAKE) -C "$(LINUX_DIR)" \
                $(MAKE_OPTS) \
                modules
endef

$(eval $(call KernelPackage,wpj563-led))

{% endhighlight %}

Tiếp theo tạo thư mục src để chứa file source:
{% highlight shell %}
mkdir src
cd src
{% endhighlight %}

Chúng ta sẽ viết file C cho module ở đây, đặt tên là wpj563-led.c
Đầu tiên, include các header cần thiết:
{% highlight c %}
#include <linux/init.h> 
#include <linux/module.h> 
#include <linux/kernel.h> 
#include <linux/gpio.h>        /* For Legacy integer based GPIO */ 
#include <linux/interrupt.h>   /* For IRQ */ 
{% endhighlight %}

Khai báo các GPIO number sẽ dùng:
{% highlight c %}
//Declare GPIO number of LEDs and BTNs
static unsigned int GPIO_LED_GREEN2 = 6;
static unsigned int GPIO_BTN1 = 2;
static unsigned int GPIO_LED_GREEN = 7;
static unsigned int irq;
{% endhighlight %}

Cũng như những module khác, chúng ta cần khai báo hàm init cho module

{% highlight c %}
static int __init gpiohello_init(void)
{
    int retval;
}
{% endhighlight %}

Phần tiếp theo sẽ áp dụng những gì đã nói trong phần 1, 2 ahíhí. Trước hết check xem các GPIO number đã khai báo có chính xác không, bằng cách thêm đoạn code sau vào hàm gpiohello_init()
{% highlight c %}
...
if (!gpio_is_valid(GPIO_LED_GREEN)||!gpio_is_valid(GPIO_BTN1)||!gpio_is_valid(GPIO_LED_GREEN2))
    {
        pr_info("Invalid GPIO\n");
        return -ENODEV;
    }
{% endhighlight %}

Tiếp theo chúng ta gửi yêu cầu quyền sử dụng các GPIO trên tới kernel:
{% highlight c %}
...
gpio_request(GPIO_LED_GREEN, "green-led");
gpio_request(GPIO_LED_GREEN2, "green2-led");
gpio_request(GPIO_BTN1, "button-1");
{% endhighlight %}

Sau khi đã dành được quyền sử dụng, bước tiếp theo là set direction. Đối với GPIO tương ứng với button, thì mục đích của nó là đọc tín hiệu từ button truyền đến kernel, nên chúng ta sẽ set nó theo chiều input, còn GPIO của các đèn LED sẽ được setup là output, vì chúng ta muốn ghi giá trị HIgh/low vào nó để tắt/bật LED. Đối với WPJ563 thì High(1) nghĩa là tắt đèn còn Low(0) nghĩa là bật đèn. Chúng ta sẽ truyền giá trị khởi tạo là High(1).
{% highlight c %}
...
gpio_direction_input(GPIO_BTN1);
gpio_direction_output(GPIO_LED_GREEN,1);
gpio_direction_output(GPIO_LED_GREEN2, 1);
{% endhighlight %}

Việc bấm nút sẽ tạo ra interrupt, chúng ta sẽ điều khiển đèn khi có interrupt xảy ra, do đó cần liên kết Button GPIO trên với một Interrupt Line và đăng ký threaded handler cho nó:
{% highlight c %}
    ...
    irq =  gpio_to_irq(GPIO_BTN1);
    retval = request_threaded_irq(irq, NULL, (irq_handler_t)btn1_pushed_irq_handler, IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING | IRQF_ONESHOT, "Oni GPIO", NULL);
    if(retval < 0){
        pr_err("ONI_GPIO: Interrupt number %d isn't free\n",irq);
        gpio_free(GPIO_LED_GREEN2);
        gpio_free(GPIO_LED_GREEN);
        gpio_free(GPIO_BTN1);

        pr_info("ONI_GPIO: The Earth rejected you\n");
        return retval;
    }
{% endhighlight %}

Hàm gpio_to_irq() sẽ trả về interrupt line tương ứng với GPIO number mà chúng ta truyền vào. 
Hàm request_threađe_irq() ở trên sẽ gán hàm btn1_pushed_irq_handler() như là bottom half của interrupt handler cho irq line, và sử dụng hàm top half mặc định. (Cái này mình đã trình bày trong bài về Interrupt).

Rõ ràng chúng ta đang thiếu hàm btn1_pushed_irq_handler(), định nghĩa hàm này ở trước hàm init của module (ngay sau phần khai báo các gpio number):
{% highlight c %}
static irq_handler_t btn1_pushed_irq_handler(unsigned int irq, void *dev_id, struct pt_regs *regs)
{
    int state, oni_led_state;
    state = gpio_get_value(GPIO_BTN1);
    oni_led_state = gpio_get_value(GPIO_LED_GREEN2);
    oni_led_state = 1-oni_led_state;
    gpio_set_value(GPIO_LED_GREEN2, oni_led_state);
    gpio_set_value(GPIO_LED_GREEN, oni_led_state);

    pr_info("GPIO_BTN1 interrupt: Interrupt! GPIO_BTN1 state is %d\n", state);
    return (irq_handler_t)IRQ_HANDLED;
}
{% endhighlight %}

Giải thích về hàm này: Trong hàm này chúng ta sẽ in ra giá trị của button (0 hoặc 1). Cùng với 1 biến oni_led_state để lưu giá trị hiện tại của led, mỗi lần gọi hàm chúng ta sẽ đổi giá trị của biến này và ghi nó vào các LED GPIO để thay đổi trạng thái của các đèn LED.

Phần còn lại là hàm exit module và các Macro linh tinh:
{% highlight c %}
static void __exit gpiohello_exit(void)
{
    free_irq(irq, NULL);
    gpio_free(GPIO_LED_GREEN2);
    gpio_free(GPIO_LED_GREEN);
    gpio_free(GPIO_BTN1);

    pr_info("End of the world\n");
}

module_init(gpiohello_init);
module_exit(gpiohello_exit);

MODULE_AUTHOR("Phi Nguyen");
MODULE_LICENSE("GPL");
{% endhighlight %}

Lưu file này lại, sau đó cũng trong thư mục <code>src</code> này, chúng ta tạo file Makèile cho module:
{% highlight shell %}
echo "obj-m += gpio_legacy.o" > Makefile
{% endhighlight %}

OK. Done, Package đã xong, bây giờ đến đoạn config và build firmware mới.
Đầu tiên phải cd quay lại thư mục source của LEDE:
{% highlight shell %}
./scripts/feeds update -a
./scripts/feeds install -a
make menuconfig
{% endhighlight %}

Tại đây, config các trường cần thiết như sau:
{% highlight shell %}
    Target System (Atheros AR7xxx/AR9xxx)
    Subtarget (Generic)
    Target Profile (Compex WPJ563 (16M flash))

    Kernel modules —> 
        Oni Modules ->
            (*) Simple GPIO Button driver by Oni
{% endhighlight %}

build nó nào: <code>make</code>

Sau khi hoàn thành, file firmware sẽ là: <code>bin/targets/ar71xx/generic/openwrt-ar71xx-generic-wpj563-squashfs-sysupgrade.bin</code>

Flash nó bằng các command sau (chạy trong uboot):
{% highligh shell %}
tftp 0x80060000 openwrt-ar71xx-generic-wpj563-squashfs-sysupgrade.bin
erase 0x9f030000 +$filesize;cp.b $fileaddr 0x9f030000 $filesize;boot
{% endhighligh %}
Lúc mới khởi động xong thì cả 2 đèn LED đều bị tắt.
<img src="{{ site.url }}/images/GPIO/ligt off.JPG">
Sau khi nó khởi động xong, bạn cần vào /etc/rc.button xóa hết các file trong này đi rồi reboot lại, nếu không lúc bấm nút nó sẽ Reboot board =)).

Đây là nút bấm: (push button):
<img src="{{ site.url }}/images/GPIO/button.JPG">
OK, bây giờ bấm nút và tận hưởng thành quả thôi.
Sau khi bấm nút thì đèn sẽ sáng lên như này, bây giờ bấm 1 lần nữa thì đèn sẽ bị tắt.
<img src="{{ site.url }}/images/GPIO/light on.JPG">

## II. GPIO interface dựa trên descriptor.

Đối với descriptor-based GPIO, một GPIO được mô tả bằng cấu trúc:
{% highlight c %}
struct gpio_desc{
    struct gpio_chip *chip;
    unsigned log flags;
    const char *label;
};
{% highlight %}

Cấu trúc này nằm trong header:
{% highlight c %}
#include <linux/gpio/consumer.h>
{% highlight %}

Trước khi cấp phát và giành quyền sử dụng GPIO, những GPIO này phải được ánh xạ tới đâu đó, tức là chúng phải được gắn với device của bạn (đối với integer-based, thì chúng ta chỉ cần request gpio number và dùng là được). Các phương pháp ánh xạ (mapping) gồm có:
-   Platform data mapping: Thực hiện trong board file.
-   Device tree: Thực hiện bằng device tree.
-   ACPI: Sử dụng ACPI mapping, thường được sử dụng trong các system x86.
