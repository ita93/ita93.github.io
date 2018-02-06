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
