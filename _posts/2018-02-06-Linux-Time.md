---
layout: post

category: Linux device driver

comments: true
---
# Linux Kernel Time.
## 1. Linux Time .
Trong Linux kernel, thời gian được đo bằng một biến golbal tên là <code> jiffies </code>. Jiffies là biến kiểu <code>usigned long</code> và có giá trị bằng số lần xảy ra ngát của iterrrupt timer kể từ khi hệ thống được khởi động. Tần số của ngắt này được khai báo trong biến <code>HZ</code> và biến này có thể được cấu hình ở thời điểm biên dịch (CONFIG_HZ), tuy nhiên nếu như không có nhu cầu gì đặc biệt thì tốt nhất nên để nó ở giá trị mặc định (250HZ trên x86_64).

## 2. Linux Standard timer.
Có nhiều mục đích cần đến <code> jiffies</code>, tuy nhiên phổ biến nhất có lẽ là dùng để tính timeout cho timer.
Trong Linux có 2 bộ timer là: Standard timer, hay còn gọi là timer wheel và một bộ timer có độ chính xác cao (đến nanosecond) gọi là high resolution timers (hrtimer). Mặc dụ timer wheel ít chính xác hơn so với hrtimer tuy nhiên nó là giải pháp hiệu quả trong trường hợp timer có thể bị hủy bỏ trước khi nó kịp timeout (thường dùng để kiểm tra timeout của peer trong networking).

Standard Timer cung cấp các hàm để tạo, hủy bỏ và quản lý timer:
Trong Linux kernel, mỗi Timer là một biến với kiểu <code>struct timer_list</code>, cấu trúc này có chứa một callback function được gọi khi timer hết hạn, cấu trúc này và các hàm liên quan được khai báo trong <code>linux/timer.h</code>. Dĩ nhiên là để sử dụng Timer chúng ta cần khởi tạo nó, bằng cách sử dụng <code>init_timer</code> hoặc <code>setup_timer</code>.
(ngoài ra còn có 1 cái macro tên DEFINE_TIMER nữa).

{% highlight c %}
void init_timer( struct timer_list *timer );
void setup_timer( struct timer_list *timer, 
                     void (*function)(unsigned long), unsigned long data );
{% endhighlight %}
Hàm <code>setup_timer</code> sẽ khởi tạo timer_list và gán hàm truyền vào (tham số thứ 2) cho callback function của timer_líst vừa được khởi tạo. Trong khi hàm <code>init_timer</code> thì chúng ta phải tự gán các giá trị cần thiết của timer sau đó gọi <code>init_timer</code> để đăng ký với kernel. (THật ra thì macro setup_timer sẽ gọi đến macro init_timer).
<b>Note</b> Trong các version kernel mới(từ 4.15) thì <code>setup_timer</code> được đổi thành <code>timer_setup</code>

Sau khi đã khởi tạo timer, ta có thể đặt thời gian timeout cho nó thông qua hàm <code>mod_timer</code> hay xóa timer với <code>del_timer</code>.

## 3. Timer example.

Sau đây mình sẽ thử viết một module đơn giản để demo khả năng của timer api. Trong module này mình sẽ khai báo một timer trong hàm init_module và đặt timeout cho nó là 200ms.
Và gán hàm <code>oni_callback</code> làm callback function cho nó. Hàm này sẽ được gọi sau 200ms. Trong code có giải thích cụ thể bằng comment.

{% highlight c %}
//Kernel 4.15.0
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/timer.h>

MODULE_LICENSE("GPL");

static struct timer_list my_timer;					//Khai báo timer list.

//Đây là hàm callback. Như đã nói ở trên, kể tử kernel 4.15 timer api đã có sự thay đổi. Và hàm callback sẽ nhâm tham
//số kiểu <i>struct timer_list *</i> thay vì <i>unsigned long</i>
void oni_callback(struct timer_list* timer) {
  printk("Oni callback function (%ld).\n", jiffies); //jiffies ở đây chính là thời điểm timer timeout + delta nhỏ.
}

int init_module(void) {
  int ret;
  printk("Oni Timer module installing\n");
  
  timer_setup(&my_timer, oni_callback, 0);						//Khởi tạo timer_list và gán callback function.

  //Ở đây mình sẽ in ra thời điểm timer bắt đầu chạy để đối chiếu.
  printk("Starting timer, current time (%ld)\n", jiffies);	
  ret = mod_timer(&my_timer, jiffies + msecs_to_jiffies(200));	//Đặt timeout cho timer.
  
  if (ret) {
    printk("Error in mod_timer\n");

    return 0;
  }
}

void cleanup_module(void) {
  int ret;
  //Xóa bỏ timer khi không sử dụng nữa
  ret = del_timer(&my_timer);
  if (ret) {
    printk("Oni timer could not be removed...\n");
  }
  printk("Bye bye timer\n");
  return;
}

{% endhighlight %}

Khi insert module này vào hệ thống, dmesg sẽ hiển thị log sau:
{% highlight shell %}
[12390.892122] Oni Timer module installing
[12390.892124] Starting timer, current time (4297990083)
[12391.092317] Oni callback function (4297990134)
{% endhighlight %}

Mình đang chạy thử nó trên x86_64, và như nói ở trên thì HZ của platform này là 250, tức là 1 jiffies sẽ tương đương với 4ms.
Theo log trên ta có fired_time - started_time = 4297990134 - 4297990083 = 51 jiffies * 4ms = 204 ms. (4ms là delta nhỏ ở trên, do printk tạo ra iterrupt).