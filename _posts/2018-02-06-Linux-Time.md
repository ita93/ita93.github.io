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

{% highlight c %}

struct timer_list {
	/*
	 * All fields that change during normal runtime grouped to the
	 * same cacheline
	 */
	struct hlist_node	entry;
	unsigned long		expires;
	void			(*function)(struct timer_list *);
	u32			flags;

#ifdef CONFIG_LOCKDEP
	struct lockdep_map	lockdep_map;
#endif
};
{% endhighlight %}

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

/*Đây là hàm callback. Như đã nói ở trên
kể tử kernel 4.15 timer api đã có sự thay đổi. Và hàm callback nhận tham
số kiểu <i>struct timer_list *</i> thay vì <i>unsigned long</i>
*/
void oni_callback(struct timer_list* timer) {
	//jiffies ở đây chính là thời điểm timer timeout + delta nhỏ.
  printk("Oni callback function (%ld).\n", jiffies); 
}

int init_module(void) {
  int ret;
  printk("Oni Timer module installing\n");
  
  timer_setup(&my_timer, oni_callback, 0);						//Khởi tạo timer_list và gán callback function.

  //Ở đây mình sẽ in ra thời điểm timer bắt đầu chạy để đối chiếu.
  printk("Starting timer, current time (%ld)\n", jiffies);	
  ret = mod_timer(&my_timer, jiffies + msecs_to_jiffies(200));	//Đặt timeout cho timer (200ms).
  
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
Theo log trên ta có fired_time - started_time = 4297990134 - 4297990083 = 51 jiffies * 4ms = 204 ms. (4ms là delta nhỏ ở trên, do printk tạo ra iterrupt). Tức là timer của mình chạy đủ 200ms. 

## 4. High-resolution timer API
Như đã nói ở phần đầu thì kernel còn một loại timer nữa là <b>hrtimer - high resolution timer</b>, tạm dịch là timer độ chính xác cao: hrtimer cho độ chính xác tới nano secs. Khác với standard timer được xây dụng dựa trên giá trị jiffer, thì <code>hrtimer</code> lại sử dụng giá trị <i>ktime</i> làm đơn vị đo, kernel cung cấp hàm ktime_set để chuyển đổi từ thời gian thông thường thành ktime. Hrtimer cũng được biểu diễn bằng một struct tên là <code>hrtimer</code>.
Hrtimer sẽ dùng một cấu trúc cây (red-black tree) - <code>struct rbtree_node </code> của kernel để quản lý các timer, các timer được thêm vào cây theo thứ tự thời gian trước-sau để giảm thiệu chi phí xử lý. 
Để sử dụng được HRT thì kernl cần phải enable CONFIG_HIGH_RES_TIMERS. Để biết HRT có được enable trên thiết bị mà đang sử dụng hay không, ta có thể dùng cmd sau:
{% highlight shell %}
cat /proc/timer_list |grep resolution
{% endhighlight %}

Nếu kết quả trả về có ít nhất 1 entry ghi ".resolution: 1 nsecs" thì có nghĩa là HRT đã được enable. Vậy câu hỏi là: khi nào thì dùng HRT? tất nhiên là khi cần độ chính xác cao - như cái tên rồi :gach:. Thật ra nó thường được dùng trong multimedia là chính thì phải, có lần đọc được thế ở đâu quên rồi, ahihi.

{% highlight c %}
//Kernel 4.19
//file: linux/hrtimer.h 
/**
 * struct hrtimer - the basic hrtimer structure
 * @node:	timerqueue node, which also manages node.expires,
 *		the absolute expiry time in the hrtimers internal
 *		representation. The time is related to the clock on
 *		which the timer is based. Is setup by adding
 *		slack to the _softexpires value. For non range timers
 *		identical to _softexpires.
 * @_softexpires: the absolute earliest expiry time of the hrtimer.
 *		The time which was given as expiry time when the timer
 *		was armed.
 * @function:	timer expiry callback function
 * @base:	pointer to the timer base (per cpu and per clock)
 * @state:	state information (See bit values above)
 * @is_rel:	Set if the timer was armed relative
 *
 * The hrtimer structure must be initialized by hrtimer_init()
 */
struct hrtimer {
	struct timerqueue_node		node;
	ktime_t				_softexpires;
	enum hrtimer_restart		(*function)(struct hrtimer *);
	struct hrtimer_clock_base	*base;
	u8				state;
	u8				is_rel;
};
{% endhighlight %}

Một HRT được khởi tạo thông qua hàm <code>hrtimer_init</code>, hàm này nhận 3 tham số: timer struct, clock được sử dụng và timer mode. Clock được định nghĩa trong file <i>timer.h</i>, đại diện cho các loại clock mà hệ thống hỗ trợ (ví dụ như CLOCK_MONOTONIC hay CLOCK_REALTIME), còn timer mode có hai dạng là thời gian tương đối (HRTIMER_MODE_RLL) và thời gian tuyệt đối (HRTIMER_MODE_ABS).

{% highlight c%}
void hrtimer_init( struct hrtimer *time, clockid_t which_clock, 
            enum hrtimer_mode mode );
int hrtimer_start(struct hrtimer *timer, ktime_t time, const 
            enum hrtimer_mode mode);
{% endhighlight %}

Một khi đã khởi tạo timer, thì ta có thể bắt đầu nó bằng cách gọi hàm <code>hrtimer_start</code>, trong hàm này sẽ có thời gian timeout được truyền vào. Và tương tự như standard timer thì HRT cũng cung cấp khả năng hủy bọ timer bằng các hàm <code>hrtimer_cancel</code> hoặc <code>hrtimer_try_to_cancal</code>. Điểm khách nhau giữa hai hàm cancel này là nếu như chúng được gọi khi timer đã gọi hàm callback, thì hàm đầu tiên sẽ đợi hàm callback hoàn thành rồi, còn hàm thứ hai sẽ kệ nó và báo lỗi là không cancel được. 
Ngoài ra ta cũng có thể kiểm tra xem timer đã gọi hàm callback của nó chưa bằng cách sử dụng <code>hrtimer_callback_running</code> function. Hai hàm khác cũng có thể được dùng để kiểm tra trạng thái timer là : <code>hrtimer_get_remaining</code> và <code>hrtimer_cb_get_time</code>, một hàm dùng để kiểm tra thời gian còn lại của timer, một hàm dùng để kiểm tra thời gian đã chạy của timer.

Kernel cũng cung cấp hai hàm <code>hrtimer_forward</code> và <code>hrtimer_forward_now</code> để thay đổi thời gian timeout của timer sau khi nó đã start.
{% highlight c %}
u64 hrtimer_forward (struct hrtimer * timer, ktime_t now, ktime_t interval);
u64 hrtimer_forward_now(struct hrtimer *timer, ktime_t interval);
{% endhighlight %}

## 5. HRT example.