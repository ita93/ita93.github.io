---
layout: post

category: Linux device driver

comments: true
---
# Linux Time.
## 1. Time Lapes.
Thời gian trong kernel được track bởi timer interrupts.<br/>
Timer interrupt được tạo bởi hardware vào các thời điểm các nhau một khoảng thời gian nhất định( gọi là regular intervals). Các interval này được lập trình ở boot time bởi kernel dựa vào giá trị của <code>HZ</code>, đây là một giá trị phụ thuộc vào arch và được định nghĩa trong file header <code>linux/param.h</code>. Giá trị mặc định của <code>HZ</code> trong kernel source là từ 50-1200 ticks per second trên máy thật, và 24 ticks per second trên máy ảo. Hầu hết các platforms chạy ở 100 hoặc 1000 interrupts một giây; các máy x86 PC mặc định là 1000. Kể cả khi bạn biết giá trị của HZ, bạn cũng không nên tính toán dựa vào giá trị này khi trong code của bạn. <br/>
Việc thay đổi giá trị của <code>HZ</code> được thực hiện bằng cách sửa đổi header file và re-compile kernel và tất cả các modules sử dụng giá trị này. Tuy nhiên, giải pháp là nên để nguyên giá trị mặc định.<br/>

Kernel có lưu một counter, tại thời điểm boot up, nó được set giá trị là 0, và sẽ được tăng lên mỗi khi xảy ra timer interrupt. Counter là biến 64-bit đối với bất kỳ arch nào, được gọi là <code>jiffies_64</code>. Tuy nhiên thông thường chúng ta sử dụng <code>jiffies</code> vì nó nhanh hơn và atomic.<br/>

## 2. Sử dụng <code>jiffies</code> Counter
<code>jiffies</code> và các hàm tiện ích của nó được khai báo trong header <code>linux/jiffies.h</code>. file header này đã được include trong file <code>sched.h</code>. Cả <code>jiffies</code> và <code>jiffies_64</code> là read-only.<br/>
Để lưu giá trị của <code>jiffies</code> trong code thì dùng một biến <code>unsigned long</code>.<br/>
P/S: volatile => tell the compiler not to optimize memory reads.<br/>
Ví dụ
{% highlight c %}
#include <linux/jiffies.h>
unsigned long j, stamp_1, stamp_half, stamp_n;

j=jiffies; /* Đọc giá trị jiffes hiện tại*/
stamp_1 = j + HZ; /* 1 second in the future */
stamp_half = j + HZ/2; /*Nửa giây nữa*/
stamp_b = j+n*HZ/1000; /* n mili giây */
{% endhighlight %}

Trong quá trình code, đôi khi cần phải so sánh các giá trị jiffies với nhau, linus cung cấp các hàm sau để thực hiện việc so sánh đó:
{% highlight c %}
int time_after(unsigned long a, unsigned long b); /*Kiểm tra xem thời điểm a có nằm sau thời điểm b không */
int time_before(unsigned long a, unsigned long b); /*Kiểm tra xem thời điểm a có nằm trước thời điểm b không */
int time_after_eq(unsigned long a, unsigned long b);/*Kiểm tra xem thời điểm a có nằm sau hoặc bằng thời điểm b không */
int time_before_eq(unsigned long a, unsigned long b);/*Kiểm tra xem thời điểm a có nằm trước hoặc bằng thời điểm b không */
{% endhighlight %}

Nếu check kernel source code, ta sẽ thấy, các hàm trên sẽ convert jiffy thành các giá trị signed long, (xử lý phần bù) và so sánh.<br/>
Vì user space biểu diễn thời gian bằng các cấu trúc <code>timeval</code> và <code>timespec</code>, nên kernel export 4 helper functions để chuyển đổi giá trị thời gian từ jiffies thành hai cấu trúc này và ngược lại:
{% highlight c %}
#include <linux/time.h>

unsigned long timespec_to_jiffies(struct timespec *value);
void jiffies_to_timespec(unsigned long jiffies, struct timespec *value);
unsigned long timeval_to_jiffies(struct timeval *value);
void jiffies_to_timeval(unsigned long jiffies, struct timeval *value);
{% endhighlight %}

Việc tryt cập <code>jiffies_64</code> không đơn giản như việc truy cập <code>jiffies</code>. Trong khi trên các máy tính cấu trúc 64-bit, hai biến này thực sự là một, thì trên máy 32-bit giá trị của <code>jiffies_64</code> không còn atomic nữa. Điều này có nghĩa là bạn có thể đọc sai giá trị nếu cả hai nửa của giá biến được update trong quá trình đọc. Thông thường thì rất rất hiếm khi cần đọc giá trị <code>jiffies_64</code>, tuy nhiên trong trường hợp cần thiết, kernel đã cung cấp giải pháp locking để đọc giá trị này như sau:
{% highlight c %}
#include <linux/jiffies.h>
u64 get_jiffies_64(void);
{% endhighlight %}

<code>u64</code> type được định nghĩa trong header <code>linux/types.h</code>.

## 3. Register "của" processor.
[Lý thuyết suông] Trong các bộ xử lý hiện đại, theo kinh nghiệm, hiệu năng của processor bị giới hạn bởi việc thời gian thực hiện của các câu lệnh về cơ bản là không thể đón trước được trong hầu hết các CPU do các cache mem, scheduling và branch prediction. Như một giải pháp, các nhà sản xuất CPU giới thiệu một clock cycles như một cách đơn giản và tin cậy để đo các time laps. Bởi vậy, hầu hết các bộ xử lý hiện đại có chứa một counter register, register này tăng lên một cách đều đặn mỗi clock cycle. Ngày nay, những clock counter này là cách đáng tin cậy duy nhất để thực hiện các high-resolution timekeeping task. <br/>
Tùy thuộc vào platform mà register có thể đọc được hoặc không đọc được từ user space, tương tự đối với việc ghi, và kích thước của nó cũng khác nhau, có thể là 64 bit hoặc 32 bit. Đối với một số platform, register này thậm chí còn không tồn tại. Mặc nhiên là, đừng bao giờ reset giá trị của register này.<br/>
Phần tiếp theo sẽ giới thiệu về Timestamp counter (TSC), đây là counter register của linux kernel. Các hàm để có thể đọc được giá trị tsc được giới thiệu ở phần highlight bên dưới:
{% highlight c %}
#include <asm/msr.h>
/*Do kernel source đổi rồi, chưa kịp tìm hiểu nên từ từ viết sau ahihi*/
{% endhighlight %}

Cơ bản thì thông thường chỉ cần dùng jiffies là đủ, tuy nhiên nếu cần việc đo lường chính xác cực cao cho những time lapses ngắn thì TSC mới được dùng.

## 4. Hiểu về thời gian hiện tại của kernel
Kernel-space (modules, driver) sẽ không bao giờ cần quan tâm đến wall-clock time (cái giờ phút giây ...), những thông tin này thường chỉ được sử dụng ở user-space program (cron và syslogd ...). Hơn nữa thì C lib cũng hỗ trợ real-world time tốt hơn ở tầng user-space. Kernel cung cấp một hàm để chuyển đổi wall-clock time thành jiffies:
{% highlight c %}
#include <linux/time.h>

unsigned long mktime(unsigned int year, unsigned int mon, unsigned int day, unsigned int hour, unsigned int min, unsigned int sec);
{% endhighlight %}

Mặc dù driver không bao giờ với xử lý với dạng biểu diễn human-readable của thời gian, đôi lúc, nó cần thực hiện một số việc với timestamp, do đó <linux/time> cung cấp hàm <code>do_gettimeofday</code>
{% highlight c %}
#include <linux/time.h>
void do_gettimeofday(struct timeval *tv);
{% endhighlight %}

Trong kernel-space, chúng ta cũng có thể lấy được current time (dưới dạng jiffies), giá trị này được lưu trong <code>struct timespec xtime</code>. Hiển nhiên là việc truy cập trực tiếp biến này là không được khuyến khích, (vì tính atomic khó đảm bảo). Bởi thế kernel cung cấp hàm sau:
{% highlight c %}
#include <linux/time.h>
struct timespec current_kernel_time(void);
{% endhighlight %}

## 4.1. Viết module để nghịch cái Time này (pending)

## 5. Delaying Execution.
Cũng giống như đối với các user-app thông thường, device driver đôi khi cũng cần delay việc thực hiện một đoạn code trong một khoảng thời gian vì một số lý do nào đó, tuy nhiên, cơ bản nhất vẫn là chờ đợi hardware thực thi một số task. Trong kernel có một số kỹ thuật khác nhau để thực hiện việc delay này, phần này sẽ tìm hiểu về các kỹ thuật đó.<br/>
## 5.1 Long Delays.
Long delay là khoảng delay nhiều hơn một clock tick (muti-jiffies). Sau đây là các kỹ thật để thực hiện long delay
### a. Busy waiting.
-Cách này không được recommend.<br/>
-Phương pháp là thực hiện một vòng lặp, duyệt qua các giá trị của jiffies đến khi thỏa mãn điều kiện.<br/>
-Sai số thời gian có thể cao<br/>
{% highlight c %}
while(time_before(jiffies,j1))
	cpu_relax();
{% endhighlight %}
<code>cpu_relax</code> là một lười gọi phụ thuộc vào arch, mục đích của nó là thông báo với processor là không làm gì cả. Trong các hyperthreaded system, nó có thể nhường core cho thread khác. Chỉ nhìn vào code, cũng thấy là nó ảnh hưởng lớn để performance, tốt nhất là nên bỏ qua nó. Nếu kernel không phải là preemptive opration, thì vòng lặp sẽ hoàn toàn lock processor cho đến khi nó kết thúc.<br/>
Tệ hơn nữa, nếu interrupt bị disable khi processor đang ở trong loop, jiffies sẽ không còn được update nữa, vòng lặp sẽ là vĩnh cửu và không còn cách nào khác là phải reset máy.<br/>

### b. Yielding the processor.
Phương pháp này cũng tương tự busy wait, tuy nhiên thay vì giữ processor và không làm gì cả, thì chúng ta sẽ giải phóng nó, và cho người khác sử dụng. 
{% highlight c %}
while(time_before(jiffies,j1))
	schedule();
{% endhighlight %}
Tuy nhiên, giải pháp này vẫn chưa tối ưu. Mặc dù process chỉ release cpu nhưng nó vẫn nằm trong run queue, tức là nó vẫn chạy.

### c. Timeouts.
-Cách tốt nhất để thực hiện delay là yêu cầu kernel làm việc đó cho bạn. Có hai cách để setup timeouts dựa vào jiffies, tùy thuộc vào việc driver có đang đợi event nào hay không.<br/>
-Nếu driver sử dụng một wait queue để đợi một event nào đấy, nhưng bạn cũng muốn chắc chắn là nó sẽ chỉ chạy trong một khoảng thời gian nhất định, thì có thể dùng <code>wait_event_timeout</code> hoặc <code>wait_event_interruptible_timout</code>. (Đã trình bày trong bài Blocking IO).<br/>. Nếu thời gian quá timeout thì function trả về 0, nếu process được đánh thức bởi condition thì nó trả về giá trị bằng với giá trị jiffies còn chưa chạy đến. Giá trị trả về không bao giờ là âm.
-Nếu driver không chờ đợi sự kiện nào đánh thức nó mà chỉ muốn delay việc thực hiện một khoảng thời gian nhất định, thì driver có thể dùng hàm <code>schedule_timeout</code>
{% highlight c %}
#include <linux/sched.h>
signed long schedule_timout(signed long timeout);
{% endhighlight %}

Giá trị trả về luôn là 0 nếu timeout được reach. <br/>
Để thực hiện được <code>schedule_timeout</code>, việc đầu tiên cần làm là khai báo state của process <code>set_current_state(TASK_INTERRUPTIBLE);</code>

## 5.2 Short delays
Khi device driver cần xử lý latencies trong hardware của nó (hardware của device), delays thường là vài microseconds là cùng, trường hợp này không thẻ dụng clock tick được.
Các function dùng cho short delays:
{% highlight c %}
#include <linux/delay.h>
void ndelay(unsigned long nsecs);
void udelay(unsigned long usecs);
void mdelay(unsigned long msecs);
{% endhighlight %}
Một số arch không định nghĩa <code>ndelay</code> và <code>mdelay</code>, trong trường hợp này, kernel sẽ cung cấp các hàm mặc định dựa vào <code>udelay</code>.<br/>
Vậy <code>udelay</code> được implement như nào? Nó sử dụng software loop dựa vào tốc độc của processor được tính ở boot time, bằng cách sử dụng giá trị <code>int loops_per_jiffy</code>. Để tránh integer overflow trong khi tính ở vòng loop, <code>udelay</code> áp một cận trên cho các giá trị truyền vào, nếu bạn sử dụng giá trị vượt quá cận này thì sẽ bị báo: <code>__bad_udelay</code>.<br/>
Cần phải nhơ là cả 3 delay functions này đều là busy-waiting, các task khác không thể chạy trong time lapse. Do đó, chúng ta chỉ nên sử dụng chúng khi không còn lựa chọn nào khác.<br/>
Vậy nếu muốn delay thì chúng ta làm thế nào? Câu trả lời là linus cho chúng ta 3 quyền năng khác để làm điều này mà không sử dụng đến busy-wait:<br/>
{% highlight c%}
void msleep(unsigned int milsec);
unsigned long msleep_interruptible(unsigned int milsec);
void ssleep(unsigned int secs);
{% endhighlight %}

##6. Kernel Timers.
Nếu muốn lên kế hoạch cho một hành đồng xảy ra sau, mà không block process, kernel time là giải pháp tốt nhất. Kernel timer được sử dụng để lập lịch cho việc thực thi của một function sẽ diễn ra tại một thời điểm cụ thể trong tương lai, dựa vào <b>clock tick</b>, và có thể được sử dụng cho nhiều task khác nhau; ví dụ như tắt motor đĩa, hay nói chung là thực hiện các thao tác phần cứng gì đấy sau thời điểm device driver đã kết thúc.<br/>
Kernel timer là một cấu trúc dữ liệu, nó sẽ ra lệnh cho kernel thực thi một hàm được người dùng định nghĩa, tại thời điểm trong tương lai do người dùng hẹn trước và với tham số mà người dùng truyền vào (lưu ý người dùng ở đây không phải user-space mà là driver-writer). Nó được implement trong các file <code>linux/timer.h</code> và <code>kernel/timer.c</code>.<br/><br/>
Các function được lên lịch để chạy hầu như chắc chắn không chạy( sử dụng processor resource) trong khi process đăng ký nó (process gọi thực hiện việc schedule cho function) đang thực thi. Thay vào đó, chúng chạy một cách không đồng bộ. Tức là khi timer chạy thì tiến trình đã thực hiện hẹn giờ nó có thể đang ngủ, đang thực thi ở một processor khác, hoặc có khi đã ngỏm.<br/><br/>
Kernel timers được chạy như là kết quả của một "software interrrupt".<br/>
Các hàm được truyền vào timer phải là atomic.<br/>
Kernel timer chạy bên ngoài proccess context, nên cần thỏa mãn một số luật sau:<br/>
-Không được truy cập đến user-space. Bởi vì đây không phải process context nên không có cách nào liên kết đến user-space được.<br/>
-<code>current</code> pointer không còn có ý nghĩa nữa và không thể sử dụng vì đây không có cái process nào cả.<br/>
-Không được phép sleep hoặc thực hiện bất kỳ schedule nào.<br/><br/>
Kernel code có thể kiểm tra xem nó có đang chạy trong interrupt context hay không bằng hàm <code>in_interrupt()</code>, hàm này trả về giá trị khác không nếu processor đang chạy trong interrupt context bất kể đó là hardware interrupt hay software interrupt.<br/>
Còn một hàm khác nữa là <code>in_atomic()</code>, nó trả về giá trị khác không nếu scheduling là không được phép.<br/><br/>

Một lưu ý quan trọng về kernel timer là nó cho một một task có thể đăng ký chính nào vào timer.<br/>

##6.1 Timer API.
Linux kernel cung cấp cho driver một số tiện ích để khai báo, đăng ký và xóa bỏ kernel timer. Tất cả được khai báo trong header <code>linux/timer.h</code>. 
{% highlight c %}
struct timer_list{
	/*...*/
	unsigned long expires;
	void(*function)(unsigned long);
	unsigned long data;
}
{% endhighlight %}

Cấu trúc này bao gồm nhiều trường hơn thế này, nhưng ở đây chỉ nêu ra những trường có thể được truy cập từ bên ngoài timer code, ý nghĩa cụ thể như sau:<br/>
<code>expires</code> : giá trị <code>jiffies</code> mà timer sẽ chạy( relative). Giá trị của expires không phải là <code>jiffies_64</code> vì kernel developer không muốn timer phải đợi quá lâu và 64-bit operations chậm trên nền tảng 32 bit.<br/>
<code>function</code> : hàm thực thi của timer, với một tham số kiểu long. Nếu muốn truyền nhiều thông tin đến function thì chúng ta có thể tạo ra một struct rồi truyền con trỏ của nó vào hàm. <br/><br/>
Trước khi sử dụng, struct phải được khởi tạo. Bước này đảm bảo rằng tất cả các trường được setup đúng. Việc khởi tạo được thực hiện bằng một trong hai macro sau.
{% highlight c %}
void init_timer(struct timer_list *timer);
struct timer_list TIMER_INITIALIZER(_function, _expires, _data);
{% endhighlight %}
Sau khi đã khởi tạo, cần phải add timer vào luồng chạy.
{% highlight c %}
void add_timer(struct timer_list *timer);
int del_timer(struct timer_lít *timer);
{% endhighlight %}