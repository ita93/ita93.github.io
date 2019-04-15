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
{% highlgiht c %}
#include <linux/ioports.h>
struct resource *request_region(unsinged long from, unsigned long extent, const char *name);
void release_region(unsigned long from, unsigned long extent);
{% endhighlight %}

Trong các hàm này thì <i>from</i> chính là base address của I/O region, còn <i>extent</i> là số lượng port mà chúng ta muốn đăng ký/ giải phóng.

Lưu ý là việc trên là không bắt buộc, bạn thậm chí có thể truy cập vào một I/O region kể cả khi việc đăng ký thất bại, tuy nhiên nguy cơ tiềm ẩn các BUG không đoán trước được và khó debug có thể xảy ra.

Hàm <code>request_region</code> sẽ trả về một con trỏ tới cấu trúc <code>struct resource</code>, cấu trúc này đã được đề cập trong bài <a href="{{ site.url }}/linux device driver/platform-device">Platform device</a>. Thực tế thì các hàm <code>*_region()</code> thật ra là wrapper của các hàm <code>request_resource</code> và <code>release_resource</code>, do đó bạn cũng có thể sử dụng các hàm <code>*_resource</code> để quản lý các I/O port.

## 3. Đọc và Ghi Dữ liệu với các I/O register.
Trong file asm/io.h, kernel định nghĩa các hàm đọc và ghi cho 8-bit, 16-bit và 32-bit ports.
### Các hàm Đọc I/O Register
{% highlight c %}
//Đọc một lần.
unsinged char inb(unsigned long port_address);
unsinged char inw(unsigned long port_address);
unsinged char inl(unsigned long port_address);
//Đọc nhiều lần : Đọc count bytes từ I/O port và ghi vào addr.
void insb(unsigned long port_address, void * addr, unsigned long count);
void insw(unsigned long port_address, void * addr, unsigned long count);
void insl(unsigned long port_address, void * addr, unsigned long count);
{% endhighlight %}

### Các hàm Ghi I/O Register
{% highlight c %}
//Đọc một lần.
unsinged char outb(unsigned long port_address);
unsinged char outw(unsigned long port_address);
unsinged char outl(unsigned long port_address);
//Ghi nhiều lần : Ghi count bytes bắt đầu từ addr vào I/O port.
void insb(unsigned long port_address, void * addr, unsigned long count);
void insw(unsigned long port_address, void * addr, unsigned long count);
void insl(unsigned long port_address, void * addr, unsigned long count);
{% endhighlight %}

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
