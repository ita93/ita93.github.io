---
layout: post

category: Linux device driver

comments: true
---

# I. Platform Devices và drivers.

Platform device là gì ? Một số người cho rằng platform device là các device được tích hợp sẵn trong các SoC. Tuy nhiên thực tế điều này không hoàn toàn đúng. Các platform device là các thiết bị không thể liệt kê được bởi hệ thống (tức là not-hotplug, khi bạn cắm một thiết bị mới vào thì OS không hề biết rằng bạn vừa thêm vào 1 cái gì đấy, khác với hotplug như USB hay PCI), do đó kernel cần phải biết về những thiết bị này từ trước lúc khởi động (tức là chúng ta phải đăng ký những thiết bị này với kernel lúc build kernel). Một số ví dụ về các thiết bị này gồm có: I2C, UART, SPI, ...

Nhìn từ góc độc kernel, những thiết bị này là các thiết bị gốc, và không kết nối đến bất kỳ cái gì cả (tức là nó là 1 phần của kernel). Trong kernel, nó sử dụng các pseudo platform bus, còn được gọi là các platform bus như là một bus ảo của kernel cho các thiết bị không nằm trên các Bus vật lý mà kernel có thể biết đến.

Trong file header <code>linux/platform_device.h</code> có định nghĩa 2 driver model interface cho platform bus nêu ở trên, gồm có:
{% highlight c %}
struct platform_device;
struct platform_driver;
{% endhighlight %}


## 1. Platform drivers
Platform driver tuân theo quy ước chuẩn của driver model, ở đó việc phát hiện/liệt kê các thiết bị được thực hiện bên ngoài các driver, và các driver cung cấp hàm <code>probe()</code> và <code>remove</code>. Chúng cung cấp các công cụ quản lý năng lượng theo tiêu chuẩn cơ bản.
{% highlight c %}
    struct platform_driver{
        int (*probe)(struct platform_device *);
        int (*probe)(struct platform_device *);
        void (*shutdown)(struct platform_device *);
        int (*suspend)(struct platform_device *, pm_message_t state);
        int (*resume_early)(struct platform_device *);
        int (*resume)(struct platform_device *);
        struct device_driver driver;
    };
{% endhighlight %}

lưu ý rằng hàm probe() nói chung nên xác thực xem phần cứng của thiết bị đã được chỉ ra thực sự tồn tại. Probing có thể sử dụng device resource bao gồm clock, và device platform_data.

Platform drivers đăng ký nó với hệ thống bằng cách sau:
{% highlight c%}
int platform_driver_register(struct platform_driver *drv);
{% endhighlight %}

Hoặc sử dụng hàm sau trong init section:
{% highlight c%}
int platform_driver_probe(struct platform_driver *drv, int(*probe)(struct platform_device*))
{% endhighlight %}

Các kernel module có thể được bao gồm nhiều platform drivers. Platform core cung cấp các hàm trợ giúp để đăng ký và hủy đăng ký một mảng các driver:
{% highlight c%}
	int __platform_register_drivers(struct platform_driver * const *drivers,
				      unsigned int count, struct module *owner);
	void platform_unregister_drivers(struct platform_driver * const *drivers,
					 unsigned int count)
{% endhighlight %}

Nếu một trong các driver đăng ký thất bại thì tất cả các drivers trong batch đều bị unregister. (Transaction)

Lưu ý là không phải tất cả platform device được được handle bằng platform driver.

Platform driver phải hiện thực một hàm <code>probe()</code>, được gọi bởi kernel khi module được inserted hoặc khi một device yêu cầu driver đó. Khi phát triển một platform driver, cấu trúc chính mà chúng ta cần sử dụng là <code>struct platform_driver</code>, bạn cần định nghĩa các trường (functions) cơ bản của cấu trúc này trước khi đăng ký nó với platform bus core.

Ví dụ
{% highlight c %}
static struct platform_driver mydrv = {
    .probe  = my_pdrv_probe,
    .remove = my_pdrv_remove,
    .driver = {
        .name = "my_platform_driver",
        .owner = THIS_MODULE,
    },
};
{% endhighlight %}


    - probe(): Đây là hàm được gọi khi thiết bị đòi hỏi driver ở lần đầu tiên, nó được khai báo như sau:
            {% highlight c %}
            static int my_pdrv_probe(struct platform_device *pdev);
            {% endhighlight %}

    - remove(): hàm này được gọi khi device không còn sử dụng driver nữa, khai báo như sau:
            {% highlight c %}
            static int my_pdrv_remove(struct platform_device *pdev);
            {% endhighlight %}

Đăng ký platform driver với kernel rất bằng cách gọi hàm <code>platform_driver_register()</code> hoặc <code>platform_driver_probe()</code> trong hàm init. Hai hàm này khác nhau ở điểm nào?
    - <code>platform_driver_register()</code> đăng ký và đặt driver vào danh sách các driver được bảo trì bởi kernel, do đó hàm probe() của driver có thể được gọi theo yêu cầu (tức là gọi khi có một device mới xuất hiện và match với signature).
    - <code>palatform_driver_probe()</code>, kernel ngay lập tức chạy một vòng lặp, kiểm tra xem nếu có một platform device match với driver (bằng trường .name) và sau đó gọi hàm <code>probe()</code> nếu nó thấy device match. Điều này có nghĩa là device phải đang kết nối với bus slot khi đăng ký, ngược lại nó sẽ bị bỏ qua. Hàm này nên đặt trong __init() section để đảm bảo là nó sẽ bị giải phóng sau khi quá trình khởi động kết thúc (ngược lại vòng lặp tiếp tục chạy sẽ rất tốn tài nguyên)

Tiếp theo chúng ta sẽ thử viết một platform driver đơn giản, nhiệm vụ của nó là tự đăng ký nó với kernel.

{% highlight c %}
/*File oni_pdriver.c*/
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/platform_device.h>

static int my_pdrv_probe(struct platform_device *pdev){
    pr_info("Hello! device probed! \n");
    return 0;
}

static void my_pdrv_remove(struct platform_device *pdev){
    pr_info("Good bye!!!!!!!!!");
}

static struct platform_driver mydrv = {
    .probe      = my_pdrv_probe,
    .remove     = my_pdrv_remove,
    .driver     = {
        .name = "oni_pdev",
        .owner = THIS_MODULE,
    },
};

static init __init my_drv_init(void){
    pr_info("Hello Guy\n");

    /*Registering with Kernel */
    platform_driver_register(&mypdrv);
    return 0;
}

static void __exit my_pdrv_remove(void){
    pr_info("Good bye Guy\n");
    /*Unregistering from Kernel*/
    platform_driver_unregister(&my_drv);
}

module_init(my_pdrv_init);
module_exit(my_pdrv_remove);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Phi Nguyen");
MODULE_DESCRIPTION("HIHI");

{% endhighlight %}

Hãy compile vào insert nó vào kernel xem có chuyện gì xảy ra.

## 2. Platform device
Bao gồm: các device sử dụng I/O port truyền thống, các host bridges đến các peripheral bus, và hầu hết các controller được tích hợp vào SoC.
Platform device không kết nối đến các bus vật lý mà nó chỉ kết nối đến psuedo bus.
Điểm chung của các thiết bị này là: đánh địa chỉ trực tiếp từ một CPU bus. Đôi khi (rất hiếm), một <code>platform_device</code> sẽ được kết nối thông qua một hoặc một số loại bus khác; nhưng các thanh ghi cảu nó vẫn được đánh địa chỉ một cách trực tiếp.

Các platform device có một tên (name field), được sử dụng trong driver binding, và một danh sách các tài nguyên chẳng hạn như addresses và IRQs. Sau khi bạn đã thực hiện xong driver, bạn sẽ phải đưa cho kernel một hoặc nhiều device sẽ cần đến driver đó. Một platform driver được biểu diễn trong kernel như một thực thể của cấu trúc sau:

{% highlight c %}
struct platform_device{
    const char      *name;
    u32             id;  
    struct device   dev;
    u32             num_resources;
    struct resource *resource;
};
{% endhighlight %}

Trường <code>name</code> cần phải trùng với trường <code>name</code> của platform_driver ở phần 1.

### 2.1 Tài nguyên và thông tin dữ liệu của platform device

Đối với các hot-pluggable devices, kernel biết rằng device đó là gì, có khả năng gì và cần gì để có thể hoạt động tốt, ngược lại, kernel không hề biết về các platform device, do đó, việc cung cấp thêm các thông tin cho kernel là cần thiết. Có hai phương thức để thông tin cho kernel về các tài nguyên (irq, dma, mem, region, bus, I/O ports) và dữ liệu (những dữ liệu riêng tư mà driver có thể cần đến) là device provisioning và [?]
### 2.1.2. Device provisioning (cũ và không khuyên dùng)
Khai báo trong mach files
#### a. Tài nguyên (resource).
Nhìn từ góc độ phần cứng, tài nguyên hay resource là tất cả các thành phần đặc tả cho đevice, và đấy là những gì device cần cho mục địch cài đặt và hoạt động chính xác. Có 6 loại resource trong kernel, được liệt kê trong header <code>include/linux/ioport.h</code>, các loại resource này được define và sử dụng như các cờ để định loài cho resouce.

{% highlight c%}
#define IORESOURCE_IO  0x00000100  /* PCI/ISA I/O ports */ 
#define IORESOURCE_MEM 0x00000200  /* Memory regions */ 
#define IORESOURCE_REG 0x00000300  /* Register offsets */ 
#define IORESOURCE_IRQ 0x00000400  /* IRQ line */ 
#define IORESOURCE_DMA 0x00000800  /* DMA channels */ 
#define IORESOURCE_BUS 0x00001000  /* Bus */
{% endhighlight %}

Trong kernel, một resource được biểu diễn như là một thực thể của cấu trúc sau:
{% highlight c %}
struct resource {
    resource_size_t start;
    resource_size_t end;
    const char *name;
    unsigned long flags;
};
{% endhighlight %}
Ở đây: start/end : là vị trí bắt đầu và kết thúc của resource. ĐỐi với I/O hoặc memory regions, thì đây chính là nơi chúng bắt đầu/kết thúc (address). ĐỐi với các kiểu resource IRQ lines, buses hay DMA channels thì start và end phải có cùng giá trị.
Flags là một mặt nạ để định loại cho resource (các IORESOURCE_* đã nêu ở trên). Tên là định danh của resource.

Khi chúng ta khai báo resource cho platform device thì chúng ta cần lấy nó ra trong driver code để làm việc với nó. Hàm <code>probe()</code> là một vị trí thích hợp cho việc này. Tham số <code>*pdev</code> trong hàm <code>probe()</code> được điền một cách tự động bởi kernel với đầy đủ data và resource đã khai báo ở trên. 
Để lấy resource ra từ pdev chúng ta sử dụng hàm sau:
{% highlight c %}
struct resource *platform_get_resource(struct platform_device *pdev, unsigned int type, unsigned int num);
{% endhighlight %}

Nếu resource là một IRQ, thì chúng ta có thể dùng cách đơn giản hơn, là sử dụng hàm sau:
{% highlight c %}
int platform_get_irq(struct platform_device *pdev, unsigned int num);
{% endhighlight %}

#### b. Data (Dữ liệu).
Bao gồm các dữ liệu không nằm trong các Resource type ở trên, ví dụ như GPIO. 
Như chúng ta đã biết struct <code>platform_device</code> có chứa trường dev có kiểu <code>struct device</code>, cấu trúc này là chứa một trường có kiểu <code>struct platform_data</code>, đây là nơi chúng ta lưu platform data.
THử tìm hiểu một ví dụ sau (code này mình tóm từ trên mạng về, hí hí):
{% highlight c %}
struct my_gpios{
    int reset_gpio;
    int led_gpio;
};

static struct my_gpios needed_gpios={
    .reset_gpio = 47,
    .led_gpio = 41
};

static struct resource needed_resources[] = {
    [0] = {/*The first memory region*/
        .start = JZ4740_UDC_BASE_ADDR,
        .end = JZ4740_UDC_BASE_ADDR + 0x10000 - 1,
        .flags = IORESOURCE_MEM,
        .name = "mem1",
    },
    [1] = {
        .start  = JZ4740_UDC_BASE_ADDR2,
        .end    = JZ4740_UDC_BASE_ADDR2 + 0x10000 - ,
        .flags  = IORESOURCE_MEM,
        .name   = "mem2",
    },
    [2] = {
        .start  = JZ4740_IRQ_UDC,
        .end    = JZ4740_IRQ_UDC,
        .flags  = IORESOURCE_IRQ,
        .name   = "mc",
    }
};

static struct platform_device my_device = {
    .name   = "my-platform-device",
    .id     = 0,
    .dev    = {
        .platform_data  = &needed_gpios,
    },
    .resource   =  needed_resources,
    .num_resources = ARRY_SIZE(needed_resources),
},

platform_device_register(&my_device);

{% endhighlight %}

Để lấy platform_data đã khai báo, chúng ta làm như sau:
{% highlight c %}
void *dev_get_platdata(const struct device *dev);
struct my_gpios *picked_gpios = dev_get_platdata(&pdev->dev);
{% endhighlight %}

# II. Kernel Kết hợp platform device với platform driver như thế nào?

Làm thế nào kernel biết được device nào được điều khiển bởi driver nào? Câu trả lời là <code>MODULE_DEVICE_TABLE</code>. Macro này sẽ giúp driver đưa ra một bảng ID của nó, bảng này mô tả những device mà nó có thể hỗ trợ. Trong khi chờ đợi, nếu như driver có thể được biên dịch như là một module (thay vì built-in), trường <code>driver.name</code> nên trùng với module name. Nếu không, module sẽ không thể load một cách tự động, trừ khi chúng ta đã sử dụng <code>MODULE_ALIAS</code> macro để thêm một tên khác cho module. Ở thời điểm biên dịch, những thông tin này sẽ được đọc từ tất cả các driver để tạo ra một bảng device table. Khi kernel phải tìm kiếm driver cho một device, kernel sẽ duyệt qua device table. Nếu kernel tìm thấy một driver (table row) phù hợp với <code>compatible</code>, hoặc device/vendor id, hoặc name của device được thêm vào, thì module đã cung cấp thông tin để tạo nên row đó sẽ được load, và <code>probe</code> function sẽ được gọi với tham số truyền vào là device tương ứng.

{% highlight c %}
#include <linux/module.h>
#define MODULE_DEVICE_TABLE(type, name)
{% endhighlight %}

Ở đây trường <code>type</code> có thể là <code>i2c, spi, of, platform, usb, pci</code> hoặc bất kỳ bus nào khác đã được định nghĩa bởi kernel trong file <code>mod_devicetable.h</code>. Nó phụ thuộc vào bus mà device ngồi, và phục thuộc vào kỹ thuật matching bạn muốn dùng. CÒn trường <code>name</code> là một con trỏ đến một mảng xxx_device_id, được sử dụng cho mục đích matching device. xxx chính là tên của bus bạn muốn dùng, hoặc of_device_id nếu bạn dùng device tree.

Kernel thực hiện việc match platform device với driver thông qua hàm <code>platform_match</code> được định nghĩa trong file /drivers/base/platform.c:
{% highlight c %}
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdev = to_platform_driver(dev);

    /*Nếu driver_override được định nghĩa, thì chỉ bind với matching driver */
    if(pdev->driver_override)
        return !strcmp(pdev->driver_override, drv->name);
    /*Thực hiện match theo style OF trước */
    if(of_driver_match_device(dev, drv))
        return 1;
    /* Thực hiện ACPI match */
    if(acpi_driver_match_device(dev,drv))
        return 1;
    /*Id table */
    if(pdrv->id_table)
        return platform_match_id(pdrv->id_table, pdev) != NULL;
    
    /*Fall-back to driver name match */
    return (strcmp(pdev->name, drv->name) == 0);
}
{% endhighlight %}

Thật ra mấy cái hàm matching trên chỉ là so sánh xâu mà thôi.

Bây giờ, mình sẽ viết một platform device để phối kết hợp với cái platform driver ở trên.
Mình sẽ viết platform device trong một file mới đặt tên là <code>oni_pdev</code>. 
Trước hết chúng ta sẽ định nghĩa ra một struct để  lưu giữ platform data. Cấu trúc này mình sẽ định nghĩa ra 2 trường là tên và tuổi (y như bài tập hồi sing viên :v). Tuy nhiên thay vì định nghĩa vào file oni_pdev.c, mình thực hiện điều này trong một file header mới tên là <code>oni_pp.h<code> vì chúng ta cần sử dụng struct này trong file platform driver.
{% highlight c %}
/*file oni_pp.h*/
struct oni_pdata{
    char* name;
    int age;
};
{% endhighlight %}

Quay lại file <code>oni_pdev.c</code>, điều đầu tiên tất nhiên là include file header ở trên vào. Tiếp theo chúng ta khai báo platform data instance và resource cho device như sau:
{% highlight c %}
/*resource là mảng các struct, chúng ta chỉ cần 1 element thôi*/
struct resource oni_rs[]={
    {
        .name = "oni_rs",       //Tên !important
        .flags = IORESOURCE_MEM,
        .start = 0x00,
        .end = 0x10,
    },
};

/*platform data instance*/
struct oni_pdata oni_pdev={
    .name = "Phi Nguyen",
    .age = 25,
};
{% endhighlight %}

Sau khi đã có data và resource, mình sẽ khai báo platform_device của mình:
{% highlight c %}
struct platform_device oni_device = {
    .name = "oni_device",   //Tên ở đây phải trùng với tên ở struct platform_driver đã viết ở phần trước*/
    .id = 0,
    .resource = oni_rs,
    .num_resources = 1,
    .dev = {
        .platform_data = &oni_pdev,
    }
};
{% endhighlight %}
Tiếp theo mình sẽ đăng ký platform device này ở trong hàm init của module và insmod module vào hệ thống.

Bây giờ mình sửa lại file oni_pdriver.c ở phần I một chút để có thể hiển thị rõ hơn việc pdriver và pdevice đã được matching với nhau.
Trong hàm <code>my_pdriver_probe()</code> mình sẽ thay dòng code
{% highlight c %}
    pr_info("Hello! device probed! \n");
{% endhighlight %}
bằng đoạn code sau:
{% highlight c %}
    struct oni_pdata* odp = dev_get_platdata(&pdev->dev);
    pr_info("Our device has name %s and age %d\n", odp->name, odp->age);
{% endhighlight %}

Quên mất là phải include file header đã tạo vào nữa.
Sau đấy rebuild nó và insmod vào kernel. Kiểm tra dmesg, bạn sẽ thấy dòng log sau:
<code>Our device has name Phi Nguyen and age 25</code>
Đây là data chúng ta đã khai báo cho platform device.

