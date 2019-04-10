---
layout: post

category: Linux device driver

comments: true
---
# Mutex - Complitions - Spinlock.

## 1. Mutex trong Linux.
Mutex là gì ? Theo 1 góc nào đó trên wiki thì:  khái niệm "mutex" thường được sử dụng để mô tả một cấu trúc ngăn cản hai tiến trình cũng thực hiện một đoạn mã hoặc truy cập một dữ liệu cùng lúc. Trên góc nhìn thực tế đầy tiền mặt, thì chẳng hạn như có 1 cột ATM VCB đang dựng đấy, người nào muốn rút tiền thì phải nhét thẻ(card) và gõ pass mang tiền về. Rõ ràng là để đưa thẻ cho cây ATM thì cần tiếp cận được cái lỗ (nhận thẻ) của nó, có thể coi cãi lỗ đấy là key, ai sở hữu key đó thì mới có thể rút tiền được, ngược lại sẽ phải đứng đợi đến khi cái lỗ đấy trống để đút vào. 
<p>Còn trong thế giới phần mềm thì bạn có thể tưởng tượng ra việc bạn có một dãy đèn LED và có 2 thread (hoặc process) đều cố gắng ghi các giá trị khách nhau vào vị trí đèn, điều gì sẽ xảy ra ? Đây chính là trường hợp Race condition (sách tiếng việt gọi là miền găng). Bởi vì việc thread nào chạy trước, thread nào chạy sau hoàn toàn phụ thuộc vào giải thuật lập lịch và về cơ bản là chúng ta không đoán trước được, nên kết quả của viêc này chúng ta không đoán trước được.</p>
Để tránh race condition, các nhà khoa học tiền bối đã phát minh ra một số phương pháp, trong đáy có mutex. (viết y như môn tập làm văn =)) ).
Vậy về mặt khoa học, Mutex là gì? Mutex là một khóa "loại trừ lẫn nhau". Tại mỗi thời điểm chỉ có 1 thread có thể giữ khóa. Mutex có thể được sử dụng để tránh race condition. Một khi mutex bị lock thì chỉ có thread đã lock nó mới có thể unlock nó. Do đó, mỗi khi bạn muốn truy cập 1 tài nguyên dùng chung thì đầu tiên bạn phải khóa mutex (trong trường hợp bạn dành được khóa) rồi mới sử dụng tài nguyên, và nhớ phải unlock nó cho người khác dùng sau khi bạn đã hoàn thành công việc.

Linux là một hệ điều hành đa nhiệm nên việc xử lý race condition là bắt buốc phải có. Và linux kernel đương nhiên là sẽ cung cấp mutex cho chúng ta sử dụng.

{% highlight c %} 
struct mutex {
    atomic_t        count;
    spinlock_t      wait_lock;
    struct list_head    wait_list;
};
{% endhighlight %}

Một mutex có thể được khởi tạo bằng <code>DEFINE_MUTEX(name)</code> - trong trường hợp mutex là một biến global khai báo ở compile time. Hoặc <code>mutex_init(struct mutex *lock)</code>- trong trường hợp mutex mutex được khai báo ở run-time.

Khi mutex đã được khởi tạo thì chúng ta có thể lock hay unlock nó. Kernel API cung cấp 5 hàm cho các tác vụ này, bao gồm 3 hàm được sử dụng cho việc lock, một hàm cho việc unlock, hàm còn lại dùng cho việc kiểm tra tình trạng mutex.
   - <code>mutex_lock</code> Sử dụng để lock/acquire mutex. Nếu mutex không khả dụng, thì task hiện tại sẽ được cho vào trạng thái sleep cho đến khi nó giành được mutex.
   - <code>mutex_lock_interruptible</code> Tương tự như hàm mutex_lock, tuy nhiên trong thời gian đợi mutex, ta có thể interrupt task, chẳng hạn với CTRL+C.
   - <code>mutex_trylock</code>  Giống như cái tên của nó, nếu không dành được lock nó sẽ return chứ không đợi chờ gì cả.
   - <code>mutex_unlock</code> Hàm này được sử dụng để giải phóng một mutex mà nó đã khóa trước đó (cùng 1 thread - task). Ai tắt nút thì người đó phải mở nút.
   - <code>mutex_is_locked</code> kiểm tra xem mutex có đang bị lock không (có thể dùng chúng với mutex_trylock).

## 2. Completions
Một trong các pattern phổ biến trong kernel programming là khởi tạo một số activity bên ngoài current thread, sau đó đợi đến khi activity đó hoànthành. Activity này có thể là tạo một kernel thread hoặc một user-space process mới, một request đến một process đã tồn tại, hoặc một sốhardware-based action.
Ví dụ:
<code>
	struct semaphore sem;<br/>
	init_MUTEX_LOCKED(&sem);<br/>
	start_external_task(&sem);<br/>
	down(&sem);
</code>
(Code trên sẽ làm giảm performance)
external_task sau đó có thể gọi up(&sem) khi công việc của nó hoàn thành. 
completion interface được dùng trong trường hợp này. Nó cho phép một thread có thể thông báo với một thread khác rằng nó đã hoàn thành công việc.
file header: <code>linix/completion.h</code><br/>
Tạo một completion bằng macro: <code>DECLARE_COMPLETION(my_completion);</code><br/>
Trường hợp cần khởi tạo ở runtime:
<code>
struct comletion my_completion;<br/>
struct init_completion(&my_completion);<br/>
</code>
Sau đấy chúng ta cần báo cho thread biết nó cần đợi completion.<br/>
<code>wait_for_completion(struct completion *c);</code><br/>
function này tạo ra một uninterruptible wait(tức là chúng ta không thể kill được process cho đến khi cái completion được set là đã hoàn thành).<br/>
Thread đang được đợi, sẽ thông báo cho calling thead rằng nó đã hoàn thành công việc bằng cách gọi một trong hai hàm sau:<br/>
<code>complete(struct completion *c);</code> Chỉ wake up một thread duy nhất<br/>
<code>compelete_all(struct completion *c);</code> Weke up tất cả các thread đang đợi<br/>
Một completion thường là one-shot device, tức là nó chỉ được dùng 1 lần sau đó sẽ bị discard. Tuy nhiên, việc sử dụng lại một completion là khả dĩ.Nếu complete_all không được sử dụng, thì struct completion có thể được sử dụng lại một cách dễ dàng. Nếu complete_all đã dược gọi, thì completionstruct cầ được tái tạo trước khi sử dụng với macro: <br/>
<code>INIT_COMPLETION(struct completion c);</code><br/>
Appendex: <code>void complete_and_exit(struct completion *c, long retval);</code>

## 3. Spinlocks
Mặc dù mutex rất hữu ích, nhưng trong kernel việc xử lý race condition được thực hiện bằng một kỹ thuật tên là spinlock. Không giống như mutex, spinlocks có thẻ được sử dụng được ở trong các đoạn code không thể sleep, ví dụ như các interrupt handlers. Khi được sử dụng đúng cách, spinlock cung cấp hiệu năng cao hơn so với mutex. Tuy nhiên spinlock cũng có một tập các ràng buộc riêng của nó.<br/>

Một spinlock là một mutual exclusion device có thể có hai và chỉ hai trạng thái "locked" và "unlocked". Nó thường được implement như một bit trong một số int. <br/>
Nếu như lock là khả dụng, thì "locked" bit được set và code sẽ tiếp tục thực thi (đi vào critical section). Ngược lại, nếu một ai đó đã set bit "locked" từ trước, thì code sẽ đi vào một vòng lặp nhỏ, và lặp đi lặp lại việc kiểm tra lock cho đến khi bit "locked" được unset. Vòng lặp này được gọi là <code>spin</code>. <br/> Đây cũng là điểm khác biệt giữa Mutex và Spinlock.
Tất nhiên, việc set và unset "locked" bit cần được thực hiện trong ngữ cảnh atomic, điểu này đảm bảo rằng chỉ có một thread duy nhất có thể dành được lock, kể cả nếu như có nhiều spin đang hoạt động.<br/>
Cần cẩn thận với deadlocks trên hyperthreaded processors.<br/>
Spinlock được tạo ra để hướng đến việc sử dụng trên multiprocessor systems.

## 4. Spinlock API
Để sử dụng được spinlock trong kernel thì các config CONFIG_PREEMPT và CONFIG_SMP phải được enable.
Các hàm và cấu trúc liên quan của spinlock được khai báo trong file header <code>linux/spinlock.h</code><br/>
Và cũng giống như mutex, spinlock api cung cấp hai khả năng khởi tạo một spinlock: 
   - Khởi tạo spinlock ở compile time: 	<code>DEFINE_SPINLOCK(lock);</code><br/>
   - Khởi tạo spinlock ở runtime:		<code>void spin_lock_init(spinlock_t *lock);</code><br/>

Trước khi vào miền găng, cần phải dành được lock bằng lời gọi hàm sau:<br/>
<code>spin_lock(spinlock_t *lock);</code><br/>
Khi code của bạn gọi hàm này, nó sẽ spin đến khi lock khả dụng, lưu ý là tất cả các spin là không thể ngắt.(Cẩn thận deadlock)<br/>
Sau khi thực hiện xong các tác vụ cần thiết, chúng ta giải phóng lock:<br/>
<code>void spin_unlock(spinlock_t *lock);</code><br/>

## 5. Spinlocks and atomic context

Thử tưởng tượng trong lúc driver của bạn đang yêu cầu một spinlock và đang chuẩn bị thực hiện công việc của nó trong miền găng thì ở một nơi nào đó, nó bị mất quyền sử dụng processor.(Có thể nó gọi đến một hàm nào đó khiến pocess sleep). Hoặc trong hệ thống SMP, kernel kick nó ra và higher-priority process gạt code của bạn sang một bên. Vấn đề là, hiện tại code của bạn hiện tại đang giữ lock, và nó sẽ không được giải phóng tại bất kỳ thời điểm có thể dự báo trong tương lai. Nếu một số thread khác cố gắng lấy cùng lock đó, nó sẽ đợi thời gian rất dài. Trong trường hợp tệ nhất, toàn hệ thống có thể rơi vào deadlock. <br/>

May mắn là, spinlock có thể xử lý trường hợp kernel preemption bởi chính nó. Mỗi khi code holds một spinlock. preemption sẽ bị vô hiệu hóa trên processor liên quan. Kể cả đối với hệ thống đơn tiến trình, preemption cũng cần được vô hiệu hóa theo cách này để tránh vi phạm các nguyên tắc của miền găng. (Do đó spinlock cần sử dụng một cách chính xác nếu không hiệu năng hệ thống sẽ bị giảm kinh khủng).<br/>

Tuy nhiên, việc tránh cho process sleep trong khi đang giữ lock là một điều khó khăn hơn nhiều; nhiều kernel functions có thể sleep, và điều này không phải lúc nào cũng được ghi chép một cách rõ ràng trong tài liệu. Do đó khi sử dụng spinlock, cần phải quan tâm đến tất cả function liên quan trong code của bạn.<br/>

Một kịch bản deadlock khác. Giả như driver của bạn đang excute và vừa mới lấy được lock để truy cập vào device của nó. Trong khi lock được giữ của driver, thì device thực hiện một interrupt, điều này khiến cho interrupt handler bắt đầu được thực thi. Interrupt handler, trước khi truy cập device, phải lấy được lock. Việc lấy spinlock từ một interrupt handler là điều không nên làm(không được phép làm); đây cũng là 1 trong những lý do mà spinlock operations không sleep. Nhưng điều gì sẽ xảy ra nếu interrupt routine thực thi trên cùng một processor với driver code đang giữ lock? Trong khi interrupt handler đang spinning, thì noninterrupt code sẽ không thể thực thi cũng như giải phóng lock được. Điều này khiến cho processor này sẽ spin mãi mãi. Để giải quyết nghịch cảnh này, yêu cầu cấp thiết là phải disable interrupt trên locl CPU khi spinlock đang được giữ. May mắn là có nhiều spinlock functions có thể làm được điều này. <br/>

Luật lệ quan trọng cuối cùng trong việc sử dụng spinlock là spinlocks phải luôn luôn được giữ trong khoảng thời gian ngắn nhất có thể. Spinlock bị giữ càng lâu, thì processor đợi spinlock sẽ bị block càng lâu, và nguy cơ các processor này rơi vào spinning khác càng nhiều hơn. Thời gian giữ lock dài cũng khiến processor đứng trong scheduler lâu hơn, điều này khiến cho process khác có priority cao hơn phải đợi lâu hơn. <br/>

Một driver được viết tệ hại có thể khiến tất cả các process phải ngồi đợi lock quá lâu => Performance giảm tụt quần

## 6. Spinlock Functions.
a. Các function có thể lock một spinlock
Spinock api cung cấp khá nhiều hàm lock để sử dụng trong các ngữ cảnh khách nhau
   - <code>void spin_lock(spinlock_t *lock);</code> //Đã nói ở phần trên <br/>
   - <code>void spin_lock_irqsave(spinlock_t *lock, unsigned long flags);</code>Hàm này sử dụng khi bạn cần lock giữa HARD IRQ và bottom half<br/>
   - <code>void spin_lock_irq(spinlock_t *lock)</code><br/> Giống hàm trên nhưng nó không lưu lại trạng thái của interrupt mà sẽ luôn enable interrupts khi code giải phón spinlock.<br/>
   - <code>void spin_lock_bh(spinlock_t *lock);</code><br/>Sử dụng khi cần tạo lock giữa Process context và Bottom half interrupt. Hàm này vô hiệu hóa software interrupts trước khi obtain lock, nhưng vẫn để hardware interrupts hoạt động.<br/>
b. Tương ứng với các hàm trên chúng ta có 4 hàm để unlock một spinlock
   - <code>void spin_unlock(spinlock_t *lock);</code><br/>
   - <code>void spin_unlock_restore(spinlock_t *lock, unsigned log flags);</code><br/>
   - <code>void spin_unlock_irq(spinlock_t *lock);</code><br/>
   - <code>void spin_unlock_bh(spinlock_t *lock);</code><br/>

<br/><br/>no blocking version<br/>
   - <code>int spin_trylock(spinlock_t *lock);</code><br/>
   - <code>int spin_trylock_bh(spinlock_t *lock);</code><br/>


## 7. Practice make perfect
Bây giờ đến phần ví dụ, phần này sẽ dùng lại device driver đã viết trong bài  <a href="{{ site.url }}/linux device driver/character-device">Character device</a>, và thêm phần xử lý miền găng vào. Nhưng trước hết, hãy viết một user-program để việc test được dễ dàng và rõ ràng hơn. Tạo một file code mới có tên là: <code>oni_test_app.c</code>
{% highlight c %}
#include<stdio.h>
#include<stdlib.h>
#include<errno.h>
#include<fcntl.h>
#include<string.h>
#include<unistd.h>
 
#define BUFFER_LENGTH 256               ///< The buffer length (crude but fine)
static char receive[BUFFER_LENGTH];     ///< The receive buffer from the LKM
 
int main(){
   int ret, fd;
   char stringToSend[BUFFER_LENGTH];
   printf("Starting device test code example...\n");
   fd = open("/dev/oni_chrdev", O_RDWR);             // Open the device with read/write access
   if (fd < 0){
      perror("Failed to open the device...");
      return errno;
   }
   printf("Type in a short string to send to the kernel module:\n");
   scanf("%[^\n]%*c", stringToSend);                // Read in a string (with spaces)
   printf("Writing message to the device [%s].\n", stringToSend);
   ret = write(fd, stringToSend, strlen(stringToSend)); // Send the string to the LKM
   if (ret < 0){
      perror("Failed to write the message to the device.");
      return errno;
   }
 
   printf("Press ENTER to read back from the device...\n");
   getchar();
 
   printf("Reading from the device...\n");
   ret = read(fd, receive, BUFFER_LENGTH);        // Read the response from the LKM
   if (ret < 0){
      perror("Failed to read the message from the device.");
      return errno;
   }
   printf("The received message is: [%s]\n", receive);
   printf("End of the program\n");
   return 0;
}
{% endhighlight %}

Compile file test vừa tạo bằng command sau:<br/>
{% highlight shell %}
$gcc oni_chrdev_test.c -o test
{% endhighlight %}

Bây giờ insert module vào kernel và chạy thử file test bằng quyền sudo, thực hiện các bước theo chỉ dẫn in ra, nhận được kết quả như sau:<br/>
{% highlight shell %}
$ sudo ./test
$ Starting device test code example...
$ Type in a short string to send to the kernel module"
$ Linux kernel
$ Writing message to the device [Linux kernel]
$ Press ENTER to read back from the device...
$ Reading from the device...
$ The received message is: [nguyen phi]
$ End of the program
{% endhighlight %}

Giải thích: khi chạy file test, nó sẽ yêu cầu nhập vào một string ngắn từ bàn phím, string này sẽ được lưu copy vào kernel space và lưu ở biến <code>msg</code> của module. Sau đó, khi người dùng ấn Enter, giá trị của msg sẽ được copy ngược lại ra user-app và in ra màn hình.<br/>
Bây giờ hãy thử một bài test để chứng tỏ tính không đồng bộ của device driver hiện tại: <br/>
-Mở 2 tab terminal và chạy <code>sudo ./test</code> ở cả 2 tab.<br>
-Ở tab đầu tiên, nhập vào chuỗi "Temp1", rồi để nó ở đấy, tức là chương trình đã ghi chuỗi "Temp1" vào device.<br/>
-Ở tab 2, nhập vào chuỗi "Temp2", sau đó gõ enter để đọc, thì chương trình sẽ in ra <code>The received message is: Temp2</code><br/>
-Quay lại tab đầu tiên, bây giờ gõ Enter, thì màn hình sẽ in ra <code>The received message is: Temp2</code>. Tức là thay vì in ra chuỗi "Temp1" thì nó lại in ra "Temp2"<br/>. Lý do là vì cả 2 chương trình test đều dùng chung resource là device file <i>oni_chrdev</i>. Chương trình 2 chạy sau, nên hàm write của nó đã overwrite giá trị biến <code>msg</code>. Sau đó khi p2 thực hiện hàm read, nó cũng xóa luôn msg, nên p1 thực hiện hàm read sau sẽ chỉ đọc được một chuỗi rỗng, ngược lại nếu thực hiện <code>read</code> của p1 trước p2, thì giá trị in ra cũng là [Temp2] chứ không phải [Temp1] như mong đợi.<br/><br/>

Để giải quyết vấn đề này, chúng ta sẽ chỉ cho phép một instance của device file (fd) được mở tại cùng 1 thời điểm. Mình sẽ khai báo một mutex và lock nó ở hàm open và unlock ở hàm release. Đầu tiên phải thêm header chứa mutex vào và định nghĩa 1 mutex để sử dụng:<br/>

{% highlight c %}
#include <linux/mutex.h>
........
static DEFINE_MUTEX(oni_mutex);
{% endhighlight %}

<br/>Bây giờ cần sửa hàm <code>oni_open</code>, hiện tại hàm này đang không làm gì cả. Hàm này sẽ kiểm tra xem mutex có đang vô chủ không, nếu có thì không cần làm gì cả, ngược lại, nó sẽ return lỗi device file đang bận và không cho phép user-app mở device file.<br/>

{% highlight c %}
static int oni_open(struct inode* node, struct file *filp)
{
	if(!mutex_trylock(&oni_mutex))
	{
		printk(KERN_ALERT "Oni chardev: Device in use by another process");
		return -EBUSY;
	}
	return 0;
}
{% endhighlight %}

<br/>Đến đây, nếu một process "P1" dành được mutex, nó sẽ mở được device file, nhưng các process khác sẽ không bao giờ động đến device file được, kể cả khi P1 đã đóng device file, vì hiện tại, mình chưa tạo đoạn code giải phóng mutex. Để làm điều này, mình sửa hàm <code>oni_release</code> như sau:<br/>

{% highlight c %}
static int oni_open(struct inode* node, struct file *filp)
{
	mutex_unlock(&oni_mutex);
	return 0;
}
{% endhighlight %}

Compile lại device driver và insert nó vào kernel. Mở 2 tab terminal. Ở tab đầu tiên chạy 1 process test:<br/>
<code>sudo ./test</code>
Kết quả hiện ra như sau:
{% highlight shell %}
Starting device test code example...
Type in a short string to send to the kernel module:
{% endhighlight %}
<br/>
Mở tiếp 1 process khác ở tab thứ 2: <code>sudo ./test</code>. Kết quả hiện ra như sau:<br/>
{% highlight shell %}
Starting device test code example...
Failed to open the device ...: Device or resource busy.
{% endhighlight %}

Như vậy, với việc thêm Mutex vào hàm open và release, bây giờ chỉ có 1 fd có thể được mở tại cùng một thời điểm.<br/><br/>
Tiếp theo, thay vì dùng Mutex, hãy thử dùng spinlock. Thay header <code>linux/mutex.h</code> bằng <code>linux/spinlock.h>.<br/>
Cũng giống như Mutex, để dùng được spinlock, ta cần định nghĩa nó trước thay dòng <code>DEFINE_MUTEX(oni_mutex></code> bằng <code>DEFINE_SPINLOCK(my_lock);</code><br/>
Sửa nội dung hàm open và release thành như sau:

{% highlight c %}
static int oni_open(struct inode* node, struct file *filp)
{
	spin_lock(&my_lock);
	return 0;
}
{% endhighlight %}
<br/>
{% highlight c %}
static int oni_open(struct inode* node, struct file *filp)
{
	spin_lock(&my_lock);
	return 0;
}
{% endhighlight %}

Với code này, khi mở một process test thứ 2, nó sẽ phải đợi (spin) đến khi process đầu tiên đóng device file thì nó mới được mở device file để thực hiện các tác vụ.