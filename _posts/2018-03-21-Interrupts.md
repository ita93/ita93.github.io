---
layout: post

category: Linux device driver

comments: true
---

# Interrupts

Trong máy tính luôn có các thiết bị ngoại vi, các thiết bị này có tốc độ hoạt động chậm hơn processor rất nhiều, vì thế hầu như luôn luôn processor cần chờ đợi các sự kiện ngoại vi, bởi thế cần có một cách nào đó để các thiết bị này thông báo cho processor mỗi khi có một sự kiện nào đấy xảy ra với nó. <br/>

Cách liên lạc đấy chính là <b>Interrupt</b>. Một <i>interrupt</i> đơn giản là một tín hiệu (signal) mà các thiết bị phần cứng (hoặc có thể là phần mềm) gửi khi nó muốn gây sự chú ý với processor (thả thính). Linux xử lý các interrupt giống như cách nó xử lý các singal trong user space. Một driver chỉ cần đăng ký một handler cho các interrupt của nó, và xử lý chúng một cách hợp lý khi xảy ra.<br/>

<span style="color:blue">Lưu ý</span>: Interrupt handler chạy trong interrupt context (song song với context hiện tại, chạy một cách đồng thời).<br/><br/>

## 1. Cài đặt một interrupt handler.<br/>

<b>Interrupt handler</b> là một software handler được cấu hình để xử lý các interrupt. Tất nhiên, cũng như driver, chúng ta cần đăng ký interrupt với Linux kernel, nếu không nó sẽ không biết và bỏ qua.<br/>
Trong kernel chỉ một có một số lượng hữ hạn resource cho các interrupt line (theo ldd3 là 15-16 line, nhưng sách này cũ rồi, không biết giờ đã thay đổi gì chưa). Kernel giữ một sổ đăng ký (giống reg trong windows) các interrupt line. Một module muốn yêu cầu một interrupt channel( hoặc IRQ) trước khi sử dụng nó và sẽ giải phóng nó khi hoàn thành (Vì số lượng có hạn nên phải giải phóng để đứa khác còn dùng), trong nhiều trường hợp, các module cũng có thể chia sẻ interrupt line với các driver khác. <br/>
Để đăng ký interrupt, chúng ta sử dụng hàm sau:<br/>
{% highlight c %}
int request_irq(unsigned int irq, irqreturn_t (*handler)(int, void *, struct pt_regs *),
				unsigned long flags, const char *dev_name, void *dev_id);
{% endhighlight %}
Hàm này trả về giá trị 0 nếu việc request thành công, ngược lại nó sẽ trả về giá trị âm nếu thất bại. Hàm này sử dụng các tham số với ý nghĩa cụ thể như sau:<br/>
<code>unsigned int irq</code> Interrupt number đang được request.<br/>
<code>irqreturn_t (*handler)(int, void*, struct pt_regs *)</code> Đây là con trỏ tới handler function sẽ được cài đặt.<br/>
<code>unsiged long flags</code> Trong các Moderm kernel thì có thể k cần quan tâm nó nữa <br/>
<code>const char *dev_name</code> Đây là tên sẽ được show ra trong file <i>/proc/interrupts</i><br/>
<code>void *dev_id</code> Con trỏ được sử dụng cho các shared interrupt line. Tốt nhất là kể cả khi không có shared interrupt thì vẫn nên trỏ con trỏ này đến device structure.<br/>
<br/>
Sau đó sử dụng hàm sau để giải phóng nó <br/>
{% highlight c %}
void free_irq(unsigned int irq, void *dev_id);
{% endhighlight %}
Cả hai hàm này đều được khai báo trong file <code>linux/interrupt.h</code><br/><br/>

Interrupt handler có thể được cài đặt tại thời điểm khởi tạo driver hoặc thời điểm device được open lần đầu tiên (hàm open của file_operations). Do số interrupt line là có hạn nên việc cài đặt interrupt handler ở thời điểm khởi tạo driver là không nên, bởi vì khả năng cao là số lượng device trong máy tính sẽ nhiều hơn số interrupt line, các device đến sau sẽ không thể lấy được interrupt line, kể cả khi các device đến trước chỉ giữ interrupt line mà để nó mốc meo không động gì dến nó. Vì thế recommend là request interrupt line ở thời điểm device open lần đầu tiên và sau đó giải phóng nó tại thời điểm device được đóng, sau khi hardware inform rằng sẽ không interrupt nữa. Hạn chế của kỹ thuật này đó là chúng ta cần phải giữ một biến đếm (cho mỗi driver) để biết khi nào thì có thể giải phóng interrupt.<br/><br/>

Mỗi khi một hardware interrupt đến được processor, một counter sẽ được tăng lên, cung cấp một công cụ hữu ích để kiểm tra xem device có đang hoạt động như mong muốn hay không. Các interrupt được show trong file <i>/proc/interrupts</i>. Ví dụ<br/>
{% highlight shell %}
           CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7
  0:       8115          0          0          0          0          0          0          0  IR-IO-APIC-edge      timer
  1:          2          0          0          0          0          0          0          0  IR-IO-APIC-edge      i8042
  8:          1          0          0          0          0          0          0          0  IR-IO-APIC-edge      rtc0
  9:          0          0          0          0          0          0          0          0  IR-IO-APIC-fasteoi   acpi
 12:          4          0          0          0          0          0          0          0  IR-IO-APIC-edge      i8042
 16:         25          0          0          0          0          0          0          0  IR-IO-APIC-fasteoi   ehci_hcd:usb1, snd_hda_intel
 23:         28          0          0          0          0          0          0          0  IR-IO-APIC-fasteoi   ehci_hcd:usb2
 40:          0          0          0          0          0          0          0          0  DMAR_MSI-edge      dmar0
 41:          0          0          0          0          0          0          0          0  DMAR_MSI-edge      dmar1
 42:          0          0          0          0          0          0          0          0  IR-PCI-MSI-edge      PCIe PME, pciehp
 43:          0          0          0          0          0          0          0          0  IR-PCI-MSI-edge      PCIe PME
 44:          0          0          0          0          0          0          0          0  IR-PCI-MSI-edge      PCIe PME
 45:   19112759          0          0          0          0          0          0          0  IR-PCI-MSI-edge      eth0
 46:        113          0          0          0          0          0          0          0  IR-PCI-MSI-edge      xhci_hcd
 47:   48445205          0          0          0          0          0          0          0  IR-PCI-MSI-edge      ahci
 48:        290          0          0          0          0          0          0          0  IR-PCI-MSI-edge      snd_hda_intel
NMI:      47262      34998      32245      34010      15699      15260      15118      15351   Non-maskable interrupts
LOC:   37630561   27204803   26168237   25823765   14453398   14292455   15021111   15100351   Local timer interrupts
SPU:          0          0          0          0          0          0          0          0   Spurious interrupts
PMI:      47262      34998      32245      34010      15699      15260      15118      15351   Performance monitoring interrupts
IWI:          0          0          0          0          0          0          0          0   IRQ work interrupts
RES:   13920936   18893192   16083423   14075673    1484409    1376323    1211626    1590645   Rescheduling interrupts
CAL:       1707       2180       2155       2230       2342       2344       2326       2231   Function call interrupts
TLB:    2073986    1829542    1841565    1829222    3144535    3116096    3104140    3125559   TLB shootdowns
TRM:          0          0          0          0          0          0          0          0   Thermal event interrupts
THR:          0          0          0          0          0          0          0          0   Threshold APIC interrupts
MCE:          0          0          0          0          0          0          0          0   Machine check exceptions
MCP:       8075       8075       8075       8075       8075       8075       8075       8075   Machine check polls
ERR:          0
MIS:          0
{% endhighlight %}

Cột đầu tiên là IRQ number, nó chỉ show ra các IRQ của các handlers đã được cài đặt (tại thời điểm in file này ra). Các cột tiếp theo là số interrupt đã tiếp nhận của mỗi CPU. Cột cuối cùng device name (lúc request interrupt có ghi vào).<br/><br/>

Một trong những vấn đề thách thức nhất khi khởi tạo driver là làm thế nào để xác định xem IRQ line nào sẽ được sử dụng bởi device. Driver cần thông tin này để đăng ký handler một cách chính xác. Trong trường hợp này, kernel cung cấp 2 funciton để probing irq là:
{% highlight c %}
unsigned long probe_irq_on(void);
int probe_irq_off(void);
{% endhighlight %}
Kỹ thuật sử dụng: Đầu tiên lấy mask bằng cách gọi <code>probe_irq_on()</code>, sau đó generate một số interrupt, rồi kiểm tra bằng <code>probe_irq_off</code> xem có thành công hay không.

## 2. Viết một interrupt handler trong driver.<br/>
Về cơ bản, một interrupt handler chỉ là một hàm C thông thường. Điểm khác biệt ở đây là hàm này sẽ chạy ở interrupt context, do đó nó có một số hạn chế đối với các hàm chạy trong process context: Một handler không thể truyền dữ liệu hoặc nhận dữ liệu từ user space, không thể sleep, không thể lock một semaphore và chỉ có thể cấp phát bộ nhớ với cờ <b>GFP_ATOMIC</b>.<br/>

Nhiệm vụ của interrupt handler là feedback cho device về việc nhận được interrupt và đọc hoặc ghi dữ liệu dựa vào ý nghĩa của interrupt (IRQ) nhận được. Thông thường, các hardware device có một "interrupt-pending" bit, khi bit này bị clear thì device sẽ không tạo thêm interrupt khác, do đó bước đầu tiên cần thực hiện với interrupt handler là phải xóa bit này. (Không phải tất cả các device đều thế, cái này là hardware depend).<br/>
Một task phổ biến đối với một interrupt handler là đánh thức các process đang ngủ trên device nếu interrupt thông báo tín hiệu về một sự kiện mà chúng đang chờ đợi. Ví dụ việc có data mới đến trong các network driver.<br/>
Một interrupt handler sẽ có cấu trúc như sau:<br/>
{% highlight c %}
static irqreturn_t my_irq_handler(int irq, void *dev_id) 
{% endhighlight %}
Trong đó, có 2 tham số: <br/>
<code>int irq</code> là giá trị của interrupt line number đang phục vụ. Giá trị chỉ có tác dụng trong việc in log.<br/>
<code>void *dev_id</code> là một con trỏ giống với *dev_id trong hàm request_irq. Nó cũng có thể là một con trỏ trỏ đến một giá trị hoạt động như một cookie để phân biệt giữa các device khác nhau có khả năng sử dụng chung một interrupt handler. <br/>

## 3. Enabling và Disabling interrupts.

Mặc dù interrupt giúp cho device giao tiếp với processor, tuy nhiên đôi lúc các device driver phỉa block việc nhận các interrupt trong một khoảng thời gian ngắn (Thường liên quan đến sleep, wait các thứ). Thông thường, các interrupt phải bị block khi device driver đang giữ một spinlock( để tránh deadlock). Có các cách để disable interrupts mà không gọi đến spinlocks. Tuy nhiên, lưu ý là việc disable interrupt là không nên, chỉ sử dụng khi thực sự không còn cách khác.<br/>

Disable một interrupt line:
{% highlight c %}
void disable_irq(int irq);
void disable_irq_nosync(int irq);
void enable_irq(int irq);
{% endhighlight %}
<br/>
Disable một Toàn bộ các interrupt
(Cái này không tốt gì cả, không nên biết).

## 4. Top và Bottom Halves