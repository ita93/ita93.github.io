---
layout: post

category: Linux device driver

comments: true
---

Việc điều khiển các thiết bị đọc ghi ngoại vi thông thường sẽ bao gồm việc đọc và ghi vào các thanh ghi của các thiết bị đó. Các thanh ghi này thường được gọi là <code>I/O ports</code>. 

## 1 Memory barriers.
Việc đọc ghi các bộ nhớ thông thường đều khá đơn giản: Ghi có nghĩa là lưu một giá trị vào một địa chỉ trong bộ nhớ, còn đọc là lấy ra giá trị gần đây nhất được ghi vào địa chỉ đó, nó không tạo ra side efect náo. Tuy nhiên, đối với I/O Register thì mọi việc lại khác, bởi vì việc đọc/ghi một I/O register có thể sẽ khiến device thực hiện một số hành động khác nhau nào đó. Do vậy việc caching hay reordering các instruction (thường được thực hiện bởi quá trình tối ưu hóa của Compiler) là không thể sử dụng được.

Kernel cung cấp giải pháp sử dụng các <code>memory barrier</code> để tránh các rủi ro này bằng các hàm sau:
- <code>barrier</code>: Hàm này sẽ khiến cho compiler lưu các giá trị đang được thay đổi vào một thanh ghi CPU, nhưng không tạo ra bất kỳ ảnh hưởng nào đến hardware.
- <code> rmb() </code>: Chèn một barrier vào mem và đảm bảo rằng các lệnh đọc trước barrier phải được hoàn thành trước khi bất kỳ lệnh đọc nào sau barrier được thực thi.
- <code> wmb() </code>: Tương tự <code>rmb()</code>, nhưng là đối với lệnh ghi
- <code> mb() </code>: Tương tự <code>rmb()</code>, nhưng áp dụng cho cả đọc và ghi.

Ngoài ra còn có các hàm <code>smp_rmb()</code>, <code>smp_read_barrier_depends()</code>, <code>smp_wmb()</code>, <code>smp_mb()</code> được sử dụng cho các hệ thống SMP.
Bởi vì các barrier sẽ ảnh hưởng đến hiệu năng chung, nên chỉ sử dụng chúng khi thực sự cần thiết.

## 2. Đăng ký I/O ports.
Trước khi sử dụng I/O ports, chúng ta cần đăng ký với kernel việc sử dụng các port muốn dùng thông qua hàm <code>request_region()</code> và giải phóng vùng nhớ đã đăng ký bằng <code>release_region</code> sau khi sử dụng xong.
{% highlight c %}
#include <linux/ioports.h>
struct resource *request_region(unsinged long from, unsigned long extent, const char *name);
void release_region(unsigned long from, unsigned long extent);
{% endhighlight %}

Trong các hàm này thì <i>from</i> chính là base address của I/O region, còn <i>extent</i> là số lượng port mà chúng ta muốn đăng ký/ giải phóng. Ngoài ra, chúng ta có thể kiểm tra xem các I/O port region nào đã được sử dụng bằng command <code>cat /proc/ioports</code>, nhớ phải chạy lệnh này bằng quyền sudo, ngược lại thì nó in ra toàn số 0.

Lưu ý là việc trên là không bắt buộc, bạn thậm chí có thể truy cập vào một I/O region kể cả khi việc đăng ký thất bại, tuy nhiên nguy cơ tiềm ẩn các BUG không đoán trước được và khó debug có thể xảy ra.


Một ví dụ đơn giản, module <i>sample_ioport</i> sẽ đăng ký một số I/O port sau đó sẽ giải phóng nó khi module exit.
{% highlight c %}
#include <linux/ioport.h>
#include <linux/kernel.h>
#include <linux/module.h>

MODULE_LICENSE("GPL");

int __init oni_init(void) {
  struct resource *res;
  //request region
  res = (struct resource *)request_region(0x380, 5, "oni_dev");
  if (res != NULL) {
    printk(KERN_INFO "Oni: Register region from 0x%llx to 0x%llx \n", res->start, res->end);
  } else {
    printk(KERN_ERR "Oni: Region has been claimed by other device\n");
    return -EBUSY;
  } 
	return 0;
}

void __exit oni_exit(void) {
  //release region
  release_region(0x380, 5);
  printk(KERN_INFO "Region freed\n");
}

module_init(oni_init);
module_exit(oni_exit);
{% endhighlight %}

Ở đây mình request 5 port bắt đầu từ vị trí <b>0x380</b>, đây là một region mà mình thấy còn trống khi kiểm tra bằng cách cat nội dung của file <i>proc/ioports</i>.
Sau khi insert module vừa viết vào hệ thống, thì dmesg sẽ có log như sau:
{% highlight shell %}
[ 7504.265791] Oni: Register region
[ 7764.511573] Oni: Register region
[ 8072.719936] Oni: Register region from 0x380 to 0x384 
{% endhighlight %}

Hơn nữa, trong file <i>ioports</i> cũng có một entry mới được thêm vào:
{% highlight shell %}
$cat /proc/ioports | grep oni_dev
0380-0384 : oni_dev
{% endhighlight %}
Hàm <code>request_region</code> sẽ trả về một con trỏ tới cấu trúc <code>struct resource</code>, cấu trúc này đã được đề cập trong bài <a href="{{ site.url }}/linux device driver/Platform-Device">Platform device</a>. Thực tế thì các hàm <code>*_region()</code> thật ra là wrapper của các hàm <code>request_resource</code> và <code>release_resource</code>, do đó bạn cũng có thể sử dụng các hàm <code>*_resource</code> để quản lý các I/O port.

## 3. Đọc và Ghi Dữ liệu với các I/O register.
Trong file asm/io.h, kernel định nghĩa các hàm đọc và ghi cho 8-bit, 16-bit và 32-bit ports.
### Các hàm Đọc I/O Register
{% highlight c %}
//Đọc một lần.
unsinged char inb(unsigned long port_address);
unsinged short inw(unsigned long port_address);
unsinged long inl(unsigned long port_address);
//Đọc nhiều lần : Đọc count bytes từ I/O port và ghi vào addr.
void insb(unsigned long port_address, void * addr, unsigned long count);
void insw(unsigned long port_address, void * addr, unsigned long count);
void insl(unsigned long port_address, void * addr, unsigned long count);
{% endhighlight %}

### Các hàm Ghi I/O Register
{% highlight c %}
//Đọc một lần.
unsinged char outb(unsigned char value, unsigned long port_address);
unsinged char outw(unsigned short value, unsigned long port_address);
unsinged char outl(unsigned long value, unsigned long port_address);
//Ghi nhiều lần : Ghi count bytes bắt đầu từ addr vào I/O port.
void insb(unsigned long port_address, void * addr, unsigned long count);
void insw(unsigned long port_address, void * addr, unsigned long count);
void insl(unsigned long port_address, void * addr, unsigned long count);
{% endhighlight %}

Nếu muốn test các hàm đọc ghi và bạn đang sử dụng một con PC intel, thì bạn có thể thử dùng I/O port từ <i>0x378</i> đến <i>0x37a</i> của parallel port, nhưng đừng request_region mà cứ dùng thẳng các hàm <code>outb()</code> và <code>inb()</code>.

## 4. Cấp phát, Mapping, và sử dụng I/O Memory.
Mặc dù phổ biến trong các thiết bị intel x86, nhưng I/O port không phải là kỹ thuật chính được sử dụng để Processor kết nối với các thiết bị ngoại vi, mà kỹ thuật đó chính là I/O Memory. 
I/O memory đơn giản là một vùng nhớ của thiết bị ngoại vi khả dụng để Processor có thể truy cập thông qua Bus. Vùng nhớ này có thể được sử dụng cho nhiều mục đích, chẳng hạn như để giữ các Gói dữ liệu, hoặc được sử dụng như các register tương tự I/O ports. Cách truy cập tới I/O memory phụ thuộc vào platform, nhưng việc này được implement bởi Kernel và nó là Transperent với Device driver.

Trước khi sử dụng I/O memory chúng ta cần cấp phát một vùng nhớ để sử dụng, và tương tự như đối với I/O ports, thì sau khi sử dụng chúng ta cần giải phsong nó:
{% highlight c %}
struct resource *request_memory_region(unsigned long start, unsigned long len, char *name);
void release_mem_region(unsigned long start, unsigned long len);
int check_mem_region(unsigned long start, unsigned long len);
{% endhighlight %}

Tuy nhiên, với I/O memory thì việc cấp phát bộ nhớ là chưa đủ, bạn phải đảm bảo rằng kernel có thể truy cập vùng nhớ I/O memory của device, thông qua việc sử dụng <code>ioremap()</code>, hàm này sẽ map địa chỉ bộ nhớ ảo tới vùng nhớ I/O memory.

Việc đọc/ghi từ I/O memory được thực hiện bằng các hàm sau:
{% highlight c %}
unsinged int ioread8(void *addr);
unsinged int ioread16(void *addr);
unsinged int ioread32(void *addr);

usinged int iowrite8(u8 val, void *addr);
usinged int iowrite16(u16 val, void *addr);
usinged int iowrite32(u32 val, void *addr);
{% endhighlight %}
Địa chỉ sử dụng trong các hàm này là địa chỉ được trả về bởi <code>ioremap()</code> + offset 

Nếu bạn muốn đọc hay ghi nhiều giá trị thì có thể dùng các hàm thuộc họ <code>io*_rep</code>, tương tự như I/O port.
{% highlight c %}
void ioread8_rep(void *addr, void *buf, unsigned long count);
void ioread16_rep(void *addr, void *buf, unsigned long count);
void ioread32_rep(void *addr, void *buf, unsigned long count);
void iowrite8_rep(void *addr, void *buf, unsigned long count);
void iowrite16_rep(void *addr, void *buf, unsigned long count);
void iowrite32_rep(void *addr, void *buf, unsigned long count);
{% endhighlight %}

Các thao tác đối với một block I/O Memory được thực hiện bởi các hàm:
{% highlight c %}
void memset_io(void *addr, u8 value, unsigned int count);
void memcpy_fromio(void *dest, void *source, unsigned int count);
void memcpy_toio(void *dest, void *source, unsigned int count);
{% endhighlight %}

## 5. Ví dụ I/O memory.
Sau khi đã trình bày đầy đủ các lý thuyết rườm rà buồn ngủ, thì việc đưa ra một ví dụ về việc sử dụng các API trên để hiểu rõ hơn về chúng là điều cần thiết. Tốt nhất nếu có device thật thì nên thử viết các module tương tác với I/O mem của device đó, để xe các side effect của nó, tuy nhiên nếu không có device thật thì cũng đừng lo, vẫn có thể làm quen với việc sử dụng các API này bình thường. 
Sau đây mình sẽ tạo một module tương tác với một vùng nhớ ảo của hệ thống, sử dụng các API về I/O memory. Chương trình này sẽ là một character device driver, trong đó các hàm đọc ghi thay vì truy cập vào một vùng nhớ được tạo ra bằng <code>kmalloc</code> thì mình sẽ request và map một I/O memory region vào device này, và các hàm đọc ghi sẽ sử dụng các hàm <code>ioread*</code> và <code>iowrite*</code> để đọc và ghi dữ liệu vào vùng nhớ.

Chương trình này sẽ tương đối giống với chương trình ví dụ trong bài <a href="{{ site.url }}/linux device driver/character-device">Character device</a>, chỉ thêm bớt một số điểm nho nhỏ :v.

Mình sẽ chọn ghi vào I/O Memory region của VRAM, đầu tiên cần xác định xem region của nó là từ đâu đến đâu đã. Đầu tiên là dùng <code>lspci</code> để in ra danh sách các PCI device, và trên máy của mình thì entry của VGA là:
{% highlight shell %}
00:02.0 VGA compatible controller: Intel Corporation Xeon E3-1200 v3/4th Gen Core Processor Integrated Graphics Controller (rev 06)
{% endhighlight %}
Tức là chúng ta tìm device có id là 00:02.0 trong proc/iomem.

{% highlight shell %}
e0000000-efffffff : 0000:00:02.0
    f0000000-f0003fff : 0000:02:00.0
  f7800000-f7bfffff : 0000:00:02.0
    f7d00000-f7d00fff : 0000:02:00.0
{% endhighlight %}

Tức là base address của nó là 0xe0000000. Bây giờ define hai biến sau:
{% highlight c %}
  #define VRAM_BASE 0xe0000000
  #define VRAM_SIZE 0x00020000
{% endhighlight %}

Lưu ý là cái địa chỉ này nó phụ thuộc vào máy của bạn, và các side effect xảy ra khi bạn đọc ghi trên vùng nhớ đó cũng phụ thuộc vào máy của bạn.

Trong hàm init của module, chúng ta sẽ request và remap cho memory region này:
{% highlight c %}
oni_res = request_mem_region(VRAM_BASE, VRAM_SIZE, DEV_NAME);
if (oni_res == NULL) {
  printk(KERN_ERR "Memory region is busy\n");
  return EBUSY;
}
{% endhighlight %}
Ở đây biến oni_res là biến có kiểu <i>strut resource *</i> (code full sẽ có cuối bài).

Tiếp theo, thêm các hàm release và unmap vào hàm exit của module, để trả lại các địa chỉ này khi chúng ta không cần đến nó nữa.
{% highlight c %}
iounmap(oni_vram);
release_mem_region(VRAM_BASE, VRAM_SIZE);
{% endhighlight %}

Tiếp theo, trong hàm <i>read</i> và <i>write</i>của <i>file_operations</i> mình sẽ dụng các hàm <code>ioread8()</code> và <code>iowrite8</code> để đọc và ghi các giá trị vào vùng nhớ I/O memory đã yêu cầu.
Sau đây là source code đầy đủ của chương trình.

{% highlight c %}
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/version.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/err.h>
#include <linux/uaccess.h>
#include <linux/ioport.h>
#include <asm/io.h>

#define MINOR_BASE 0
#define MINOR_COUNT 1
#define DEV_NAME "oni_map"
#define VRAM_BASE 0xe0000000
#define VRAM_SIZE 0x00020000

static void __iomem *oni_vram;
static struct cdev oni_mapdev;
static struct class *oni_class;
static struct device *oni_device;
static dev_t dev_num;
static struct resource *oni_res;

static int oni_open(struct inode* node, struct file* filp){
  printk(KERN_INFO "Device file just open\n");
  return 0;
}

static int oni_close(struct inode* node, struct file* filp){
  printk(KERN_INFO "Device file just close\n");
  return 0;
}

static ssize_t oni_read(struct file* filp, char __user *buffer, size_t count, loff_t* offset) {
  u8 byte;
  int i;
  if (*offset > VRAM_SIZE) {
    return -1;
  } 

  if (*offset + count > VRAM_SIZE) {
    count = VRAM_SIZE - *offset;
  }

  for (i = 0; i< count; i++){
    byte = ioread8(oni_vram + *offset + i);
    if (copy_to_user(buffer + i, &byte, 1)) {
      return -EFAULT;
    }
  }

  *offset += count;
  return count;
}

static ssize_t oni_write(struct file* filp, const char __user *buffer, size_t count, loff_t* offset) {
  u8 byte;
  int i;
  if (*offset > VRAM_SIZE){
    return -1;
  }
  
  if (*offset + count > VRAM_SIZE) {
    count = VRAM_SIZE - *offset;
  }

  for (i = 0; i < count; i++) {
    if (copy_from_user(&byte, buffer + i, 1)){
      return -EFAULT;
    }
    iowrite8(byte, oni_vram + *offset + i);
  }
  *offset += count;
  return count;
}

struct file_operations oni_fops = {
  .owner = THIS_MODULE,
  .open = oni_open,
  .release = oni_close,
  .write = oni_write,
  .read = oni_read
};

int __init oni_init(void) {
  int ret;

  //request memory region
  oni_res = request_mem_region(VRAM_BASE, VRAM_SIZE, DEV_NAME);
  if (oni_res == NULL) {
    printk(KERN_ERR "Memory region is busy\n");
    return EBUSY;
  }
  
  oni_vram = ioremap(VRAM_BASE, VRAM_SIZE);
  if (oni_vram == NULL) {
    printk(KERN_ERR "Memory remapping failed\n");
    release_mem_region(VRAM_BASE, VRAM_SIZE);
    return EBUSY;
  }

  ret = alloc_chrdev_region(&dev_num, MINOR_BASE, MINOR_COUNT, DEV_NAME);
  if (ret != 0)
    goto done;

  cdev_init(&oni_mapdev, &oni_fops); 
  ret = cdev_add(&oni_mapdev, dev_num, MINOR_COUNT);
  if (ret != 0){
    goto un_region;
  }

  oni_class = class_create(THIS_MODULE, DEV_NAME);
  if (IS_ERR(oni_class)) {
    goto del_dev;
  }

  oni_device = device_create(oni_class, NULL, dev_num, NULL, DEV_NAME);
  if (IS_ERR(oni_device)) {
    goto destroy_all;
  }

  return 0;

destroy_all:
  class_destroy(oni_class);
del_dev:
  cdev_del(&oni_mapdev);
  printk(KERN_ERR "Cannot create class\n");
un_region:
  unregister_chrdev_region(dev_num, MINOR_COUNT);
  printk(KERN_ERR "Cannot add device to kernel\n");
done:
  printk(KERN_ERR "Cannot allocate a device number\n");
  return -1;
}

void __exit oni_exit(void){
  device_destroy(oni_class, dev_num);
  class_destroy(oni_class);
  cdev_del(&oni_mapdev);
  unregister_chrdev_region(dev_num, MINOR_COUNT);
  iounmap(oni_vram);
  release_mem_region(VRAM_BASE, VRAM_SIZE);
}

module_init(oni_init);
module_exit(oni_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Phi Nguyen");

{% endhighlight %}

Sau khi đã hoàn thành source code, mình sẽ compile nó, và insert nó vào hệ thống 
{% highlight c %}
$sudo insmod onimap.ko
{% endhighlight %}

Sau đấy mình sẽ ghi nội dung linh tinh vào device file (tức là sẽ gọi đến hàm write), lệnh này phải chạy bằng quyền root.
{% highlight c %}
$echo -n "132415645646598741346546467985413215676" > /dev/oni_map
{% endhighlight %}

Bây giờ đọc giá trị đã ghi vào bằng lệnh sau:
{% highlight c %}
$sudo cat /dev/onimap
{% endhighlight %}

Kết quả là trên màn hình sẽ hiện ra giá trị mà mình đã ghi vào cộng với một lô một lốc các ký tự đặc biệt theo sau vì cái vùng nhớ này giá trị mặc định của các ô nhớ là cái gì đấy không thể hiện lên dưới dạng ascii.
