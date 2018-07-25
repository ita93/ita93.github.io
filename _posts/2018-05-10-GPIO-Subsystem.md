---
layout: post

category: Linux device driver

comments: true
---
<figure>
  <img src="{{ site.url }}/images/GPIO/gpio-pins.jpg" alt="GPIO Pins">
  <figcaption>GPIO Pins trên máy Pi</figcaption>
</figure>

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
{% highlight shell %}
tftp 0x80060000 openwrt-ar71xx-generic-wpj563-squashfs-sysupgrade.bin
erase 0x9f030000 +$filesize;cp.b $fileaddr 0x9f030000 $filesize;boot
{% endhighlight %}
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
{% endhighlight %}

Cấu trúc này nằm trong header:
{% highlight c %}
	#include <linux/gpio/consumer.h>
{% endhighlight %}

Trước khi cấp phát và giành quyền sử dụng GPIO, những GPIO này phải được ánh xạ tới đâu đó, tức là chúng phải được gắn với device của bạn (đối với integer-based, thì chúng ta chỉ cần request gpio number và dùng là được). Các phương pháp ánh xạ (mapping) gồm có:
-   Platform data mapping: Thực hiện trong board file.
-   Device tree: Thực hiện bằng device tree.
-   ACPI: Sử dụng ACPI mapping, thường được sử dụng trong các system x86.

### 1. Cấp phát và sử dụng descriptor-base GPIO
Bạn có thể sử dụng gpiod_get() hoặc gpio_get_index() để cấp phát một GPIO descriptor:

{% highlight c %}
struct gpio_desc *gpiod_get_index(struct device *dev, const char *const_id, unsigned int idx, enum gpio_flags flags);
struct gpio_desc *gpiod_get(struct device *dev, const char *con_id, enum gpiod_flags flags);
{% endhighlight %}

Hai hàm này trả về cấu trúc GPIO descriptor tương ứng với GPIO đã được truyền vào, điểm khác nhau là hàm đầu tiên thế trả về GPIO theo index trong biến idx, còn hàm thứ hai sẽ trả về GPIO ở index 0, (nếu bạn kiểm tra kernel sourc, thì sẽ thấy hàm thứ 2 thật ra chỉ là 1 lời gọi đến hàm thứ nhất). <code>dev</code> là device mà GPIO descriptor thuộc về.
<code>flag</code> dùng để cấu hình chiều truyền của GPIO, nó là một thể hiện của <code>enum gpio_flags</code>:
{% highlight c %}
enum gpio_flags{
    GPIOD_ASIS = 0,
    GPIOD_IN = GPIOD_FLAGS_BIT_DIR_SET,
    GPIOD_OUT_LOW = GPIOD_FLAGS_BIT_DIR_SET |
                    GPIOD_FLAGS_BIT_DIR_OUT,
    GPIOD_OUT_HIGH = GPIO_FLAGS_BIT_DIR_SET |
                    GPIO_FLAGS_BIT_DIR_OUT |
                    GPIO_FLAGS_BIT_DIR_VAL,
};
{% endhighlight %}

Sau khi đã cấp phát và dành quyền sử dụng GPIO, chúng ta có thể thao tác đọc, ghi thay đổi debounce với các hàm sau:
{% highlight c %}
int gpiod_direction_input(struct gpio_desc *desc); 
int gpiod_direction_output(struct gpio_desc *desc, int value);
int gpiod_set_debounce(struct gpio_desc *desc, unsigned debounce);
struct gpio_desc *gpio_to_desc(unsigned gpio); 
int desc_to_gpio(const struct gpio_desc *desc);
int gpiod_get_value(const struct gpio_desc *desc); 
void gpiod_set_value(struct gpio_desc *desc, int value);
{% endhighlight %}

# B. GPIO Controller Driver.

Driver cho các device này sẽ cung cấp các khả năng sau:
-   Các phương thức để khởi tạo GPIO line direction
-   Các phương thức dùng để truy cập giá trị GPIO Line.
-   Các phương thức dùng để cài đặt cấu hình điện(electrial) cho một GPIO line.
-   Cờ: Các lời gọi có thể ngủ hay không.
-   Tên của Line (một mảng) để định danh các line (Tùy chọn).
-   debugfs dump method (Tùy chọn).
-   Base number (Sẽ được gán tự động nếu bạn bỏ qua). (tùy chọn).

Trong kernel code, một GPIO controller được biểu diễn đầy đủ bằng cấu trúc gpio_chip:
{% highlight c %}
/**
 * struct gpio_chip - abstract a GPIO controller
 * @label: a functional name for the GPIO device, such as a part
 *	number or the name of the SoC IP-block implementing it.
 * @gpiodev: the internal state holder, opaque struct
 * @parent: optional parent device providing the GPIOs
 * @owner: helps prevent removal of modules exporting active GPIOs
 * @request: optional hook for chip-specific activation, such as
 *	enabling module power and clock; may sleep
 * @free: optional hook for chip-specific deactivation, such as
 *	disabling module power and clock; may sleep
 * @get_direction: returns direction for signal "offset", 0=out, 1=in,
 *	(same as GPIOF_DIR_XXX), or negative error
 * @direction_input: configures signal "offset" as input, or returns error
 * @direction_output: configures signal "offset" as output, or returns error
 * @get: returns value for signal "offset", 0=low, 1=high, or negative error
 * @get_multiple: reads values for multiple signals defined by "mask" and
 *	stores them in "bits", returns 0 on success or negative error
 * @set: assigns output value for signal "offset"
 * @set_multiple: assigns output values for multiple signals defined by "mask"
 * @set_config: optional hook for all kinds of settings. Uses the same
 *	packed config format as generic pinconf.
 * @to_irq: optional hook supporting non-static gpio_to_irq() mappings;
 *	implementation may not sleep
 * @dbg_show: optional routine to show contents in debugfs; default code
 *	will be used when this is omitted, but custom code can show extra
 *	state (such as pullup/pulldown configuration).
 * @base: identifies the first GPIO number handled by this chip;
 *	or, if negative during registration, requests dynamic ID allocation.
 *	DEPRECATION: providing anything non-negative and nailing the base
 *	offset of GPIO chips is deprecated. Please pass -1 as base to
 *	let gpiolib select the chip base in all possible cases. We want to
 *	get rid of the static GPIO number space in the long run.
 * @ngpio: the number of GPIOs handled by this controller; the last GPIO
 *	handled is (base + ngpio - 1).
 * @names: if set, must be an array of strings to use as alternative
 *      names for the GPIOs in this chip. Any entry in the array
 *      may be NULL if there is no alias for the GPIO, however the
 *      array must be @ngpio entries long.  A name can include a single printk
 *      format specifier for an unsigned int.  It is substituted by the actual
 *      number of the gpio.
 * @can_sleep: flag must be set iff get()/set() methods sleep, as they
 *	must while accessing GPIO expander chips over I2C or SPI. This
 *	implies that if the chip supports IRQs, these IRQs need to be threaded
 *	as the chip access may sleep when e.g. reading out the IRQ status
 *	registers.
 * @read_reg: reader function for generic GPIO
 * @write_reg: writer function for generic GPIO
 * @be_bits: if the generic GPIO has big endian bit order (bit 31 is representing
 *	line 0, bit 30 is line 1 ... bit 0 is line 31) this is set to true by the
 *	generic GPIO core. It is for internal housekeeping only.
 * @reg_dat: data (in) register for generic GPIO
 * @reg_set: output set register (out=high) for generic GPIO
 * @reg_clr: output clear register (out=low) for generic GPIO
 * @reg_dir: direction setting register for generic GPIO
 * @bgpio_bits: number of register bits used for a generic GPIO i.e.
 *	<register width> * 8
 * @bgpio_lock: used to lock chip->bgpio_data. Also, this is needed to keep
 *	shadowed and real data registers writes together.
 * @bgpio_data:	shadowed data register for generic GPIO to clear/set bits
 *	safely.
 * @bgpio_dir: shadowed direction register for generic GPIO to clear/set
 *	direction safely.
 *
 * A gpio_chip can help platforms abstract various sources of GPIOs so
 * they can all be accessed through a common programing interface.
 * Example sources would be SOC controllers, FPGAs, multifunction
 * chips, dedicated GPIO expanders, and so on.
 *
 * Each chip controls a number of signals, identified in method calls
 * by "offset" values in the range 0..(@ngpio - 1).  When those signals
 * are referenced through calls like gpio_get_value(gpio), the offset
 * is calculated by subtracting @base from the gpio number.
 */
struct gpio_chip {
	const char		*label;
	struct gpio_device	*gpiodev;
	struct device		*parent;
	struct module		*owner;

	int			(*request)(struct gpio_chip *chip,
						unsigned offset);
	void			(*free)(struct gpio_chip *chip,
						unsigned offset);
	int			(*get_direction)(struct gpio_chip *chip,
						unsigned offset);
	int			(*direction_input)(struct gpio_chip *chip,
						unsigned offset);
	int			(*direction_output)(struct gpio_chip *chip,
						unsigned offset, int value);
	int			(*get)(struct gpio_chip *chip,
						unsigned offset);
	int			(*get_multiple)(struct gpio_chip *chip,
						unsigned long *mask,
						unsigned long *bits);
	void			(*set)(struct gpio_chip *chip,
						unsigned offset, int value);
	void			(*set_multiple)(struct gpio_chip *chip,
						unsigned long *mask,
						unsigned long *bits);
	int			(*set_config)(struct gpio_chip *chip,
					      unsigned offset,
					      unsigned long config);
	int			(*to_irq)(struct gpio_chip *chip,
						unsigned offset);

	void			(*dbg_show)(struct seq_file *s,
						struct gpio_chip *chip);
	int			base;
	u16			ngpio;
	const char		*const *names;
	bool			can_sleep;

#if IS_ENABLED(CONFIG_GPIO_GENERIC)
	unsigned long (*read_reg)(void __iomem *reg);
	void (*write_reg)(void __iomem *reg, unsigned long data);
	bool be_bits;
	void __iomem *reg_dat;
	void __iomem *reg_set;
	void __iomem *reg_clr;
	void __iomem *reg_dir;
	int bgpio_bits;
	spinlock_t bgpio_lock;
	unsigned long bgpio_data;
	unsigned long bgpio_dir;
#endif

#ifdef CONFIG_GPIOLIB_IRQCHIP
	/*
	 * With CONFIG_GPIOLIB_IRQCHIP we get an irqchip inside the gpiolib
	 * to handle IRQs for most practical cases.
	 */

	/**
	 * @irq:
	 *
	 * Integrates interrupt chip functionality with the GPIO chip. Can be
	 * used to handle IRQs for most practical cases.
	 */
	struct gpio_irq_chip irq;
#endif

#if defined(CONFIG_OF_GPIO)
	/*
	 * If CONFIG_OF is enabled, then all GPIO controllers described in the
	 * device tree automatically may have an OF translation
	 */

	/**
	 * @of_node:
	 *
	 * Pointer to a device tree node representing this GPIO controller.
	 */
	struct device_node *of_node;

	/**
	 * @of_gpio_n_cells:
	 *
	 * Number of cells used to form the GPIO specifier.
	 */
	unsigned int of_gpio_n_cells;

	/**
	 * @of_xlate:
	 *
	 * Callback to translate a device tree GPIO specifier into a chip-
	 * relative GPIO number and flags.
	 */
	int (*of_xlate)(struct gpio_chip *gc,
			const struct of_phandle_args *gpiospec, u32 *flags);
#endif
};
{% endhighlight %}