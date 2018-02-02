---
layout: post

category: Linux device driver

comments: true
---
# Concurrency and Race conditions(DRAFT Ver).

## 1. Semaphore trong Linux.

Để sử dụng semaphores, module của chúng ta phải include header asm/semphore.h. 
Kiểu dữ liệu cần dùng ở đây là struct semaphore;

a. Những cách khai báo và khởi tạo semaphores.
Cách 1: Tạo semaphore một cách trực tiếp, sau đó setup với sema_init:
<code>sema_init(struct semaphore *sem, int val);</code>
<code>val</code> là giá trị khởi tạo được gán cho semaphore.

Tuy nhiên, thông thường semaphore được sử dụng ở mutex mode. Kernel cung cấp một số hàm và macro, để thực hiện việc này dễ dàng hơn.
<code>DECLARE_MUTEX(name)</code>
<code>DECLARE_MUTEX_LOCKED(name)</code>
<p>Hai macro trên giúp chúng ta khai báo và khởi tạo semaphore ở mutex mode. Kết quả trả về ở đây là một biến semaphore (name) có giá trị khởi tạo (val) là 1(Với macro DECLARE_MUTEX) hoặc 0(với macro DECLARE_MUTEX_LOCKED). Đối với mutex bị lock thì nó cần được unlock trước khi bất kỳ một thread nào đó có thể truy cập đến nó.</p>

<p>Để giảm giá trị của val trong semaphore, ta dùng các version của down function:</p>
<code>void down(struct semaphore *sem);</code><br/>
<code>int down_interruptible(struct semaphore *sem);</code><br/>
<code>int down_trylock(struct semaphore *sem);</code><br/>
Sự khác biệt giữa các hàm này là như thế nào? <i>down</i> giảm giá trị của sem->val và wait. <i>down_interruptible</i> thực hiện cùng một tác vụ nhưng việc đợi có thể bị ngắt. Thông thường chúng ta muốn cho phép user-space process có thể ngừng (killed) bởi người dùng nên <i>down_interruptible</i> thông dụng hơn.<br/> 
<i>down</i> version là một cách tốt để tạo ra các process không thể kill. (D state in ps command).
Nếu tác vụ bị ngắt, thì <code>down_interruptible</code> trả về một giá tị khác không và caller không giữ semaphore. Việc sử dụng down_interruptible yêu cầu chúng ta phải luôn luôn check giá trị trá về và phản ứng một cách tương ứng.<br/>
<i>down_trylock</i> không bao giờ sleep, nếu semaphore không khả dụng ở thời điểm gọi đến, nó sẽ return ngay lập tức một giá trị khác 0.<br/>

Một khi thread đã gọi thành công một trong các version của <code>down</code> thì nó được gọi là holding thread của semaphore đó. Bây giờ thread này có toàn quyền với semaphore. Do đó nó cần giải phóng semaphore cho các thread khác sau khi hoàn thành công việc trong miền găng.<br/>
<code>up(struct semaphore *sem);</code><br/>
Sau khi up() được gọi thì caller thread không còn hold semaphore nữa.<br/>

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
Mặc dù semaphore rất hữu ích, nhưng hầu hết các locking trong kernel được thực hiện bằng một kỹ thuật tên là spinlock. Không giống như semaphores, spinlocks có thẻ được sử dụng được ở trong các đoạn code không thể sleep, ví dụ như các interrupt handlers. Khi được sử dụng đúng cách, spinlock cung cấp hiệu năng cao hơn so với semaphore. Tuy nhiên spinlock cũng có một tập các ràng buộc riêng của nó.<br/>

Một spinlock là một mutual exclusion device có thể có hai và chỉ hai trạng thái "locked" và "unlocked". Nó thường được implement như một bit trong một số int. <br/>
Nếu như lock là khả dụng, thì "locked" bit được set và code sẽ tiếp tục thực thi (đi vào critical section). Ngược lại, nếu một ai đó đã set bit "locked" từ trước, thì code sẽ đi vào một vòng lặp nhỏ, và lặp đi lặp lại việc kiểm tra lock cho đến khi bit "locked" được unset. Vòng lặp này được gọi là <code>spin</code>. <br/>
Tất nhiên, việc set và unset "locked" bit cần được thực hiện trong ngữ cảnh atomic, điểu này đảm bảo rằng chỉ có một thread duy nhất có thể dành được lock, kể cả nếu như có nhiều spin đang hoạt động.<br/>
Cần cẩn thận với deadlocks trên hyperthreaded processors.<br/>
Spinlock được tạo ra để hướng đến việc sử dụng trên multiprocessor systems.

## 4. Spinlock API
file header <code>linux/spinlock.h</code><br/>
Khởi tạo spinlock ở compile time: 	<code>spinlock_t my_lock = SPIN_LOCK_UNLOCKED;</code><br/>
Khởi tạo spinlock ở runtime:		<code>void spin_lock_init(spinlock_t *lock);</code><br/>
Trước khi vào miền găng, cần phải dành được lock bằng lời gọi hàm sau:<br/>
<code>spin_lock(spinlock_t *lock);</code><br/>
Khi code của bạn gọi hàm này, nó sẽ spin đến khi lock khả dụng, lưu ý là tất cả các spin là không thể ngắt.(Cẩn thận deadlock)<br/>
Sau khi thực hiện xong các tác vụ cần thiết, chúng ta giải phóng lock:<br/>
<code>void spin_unlock(spinlock_t *lock);</code><br/>

## 5. Spinlocks and atomic context

Thử tưởng tượng trong lúc driver của bạn đang yêu cầu một spinlock và đang chuẩn bị thực hiện công việc của nó trong miền găng thì ở một nơi nào đó, nó bị mất quyền sử dụng processor.(Có thể nó gọi đến một hàm nào đó khiến pocess sleep). Hoặc trong hệ thống SMP, kernel kick nó ra và higher-priority process gạt code của bạn sang một bên. Vấn đề là, hiện tại code của bạn heienj tại đang giữ lock, và nó sẽ không được giải phóng tại bất kỳ thời điểm có thể dự báo trong tương lai. Nếu một số thread khác cố gắng lấy cùng lock đó, nó sẽ đợi thời gian rất dài. Trong trường hợp tệ nhất, toàn hệ thống có thể rơi vào deadlock. <br/>

May mắn là, spinlock có thể xử lý trường hợp kernel preemption bởi chính nó. Mỗi khi code holds một spinlock. preemption sẽ bị vô hiệu hóa trên processor liên quan. Kể cả đối với hệ thống đơn tiến trình, preemption cũng cần được vô hiệu hóa theo cách này để tránh vi phạm các nguyên tắc của miền găng. (Do đó spinlock cần sử dụng một cách chính xác nếu không hiệu năng hệ thống sẽ bị giảm kinh khủng).<br/>

Tuy nhiên, việc tránh cho process sleep trong khi đang giữ lock là một điều khó khăn hơn nhiều; nhiều kernel functions có thể sleep, và điều này không phải lúc nào cũng được ghi chép một cách rõ ràng trong tài liệu. Do đó khi sử dụng spinlock, cần phải quan tâm đến tất cả function liên quan trong code của bạn.<br/>

Một kịch bản deadlock khác. Giả như driver của bạn đang excute và vừa mới lấy được lock để truy cập vào device của nó. Trong khi lock được giữ của driver, thì device thực hiện một interrupt, điều này khiến cho interrupt handler bắt đầu run. Interrupt handler, trước khi truy cập device, phải lấy được lock. Việc lấy spinlock từ một interrupt handler là điều không nên làm(không được phép làm); đây cũng là 1 trong những lý do mà spinlock operations không sleep. Nhưng điều gì sẽ xảy ra nếu interrupt routine thực thi trên cùng một processor với driver code đang giữ lock? Trong khi interrupt handler đang spinning, thì noninterrupt code sẽ không thể thực thi cũng như giải phóng lock được. Điều này khiến cho processor này sẽ spin mãi mãi. Để giải quyết nghịch cảnh này, yêu cầu cấp thiết là phải disable interrupt trên locl CPU khi spinlock đang được giữ. May mắn là có nhiều spinlock functions có thể làm được điều này. <br/>

Luật lệ quan trọng cuối cùng trong việc sử dụng spinlock là spinlocks phải luôn luôn được giữ trong khoảng thời gian ngắn nhất có thể. Spinlock bị giữ càng lâu, thì processor đợi spinlock sẽ bị block càng lâu, và nguy cơ các processor này rơi vào spinning khác càng nhiều hơn. Thời gian giữ lock dài cũng khiến processor đứng trong scheduler lâu hơn, điều này khiến cho process khác có priority cao hơn phải đợi lâu hơn. <br/>

Một driver được viết tệ hại có thể khiến tất cả các process phải ngồi đợi lock quá lâu => Performance giảm tụt quần

## 6. Spinlock Functions.
a. Các function có thể lock một spinlock
<code>void spin_lock(spinlock_t *lock);</code> //Đã nói ở phần trên <br/>
<code>void spin_lock_irqsave(spinlock_t *lock, unsigned long flags);</code><br/>
Hàm này sẽ disable interrupts(local processor only) trước khi obtain spinlock.<br/>
<code>
void spin_lock_irq(spinlock_t *lock);
</code><br/>
Giống hàm trên nhưng nó không lưu lại trạng thái của interrupt mà sẽ luôn enable interrupts khi code giải phóng spinlock.<br/>
<code>void spin_lock_bh(spinlock_t *lock);</code><br/>
Hàm này vô hiệu hóa software interrupts trước khi obtain lock, nhưng vẫn để hardware interrupts hoạt động.<br/>
b. Tương ứng với các hàm trên chúng ta có 4 hàm để unlock một spinlock
<code>void spin_unlock(spinlock_t *lock);</code><br/>
<code>void spin_unlock_restore(spinlock_t *lock, unsigned log flags);</code><br/>
<code>void spin_unlock_irq(spinlock_t *lock);</code><br/>
<code>void spin_unlock_bh(spinlock_t *lock);</code><br/>

<br/><br/>no blocking version<br/>
<code>int spin_trylock(spinlock_t *lock);</code><br/>
<code>int spin_trylock_bh(spinlock_t *lock);</code><br/>

## 7. Practice make perfect
Bây giờ đến phần ví dụ, phần này sẽ dùng lại device driver đã viết trong bài [Character device], và thêm phần xử lý miền găng vào. Nhưng trước hết, hãy viết một user-program để việc test được dễ dàng và rõ ràng hơn. Tạo một file code mới có tên là: <code>oni_test_app.c</code>
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

Giải thích: khi chạy file test, nó sẽ yêu cầu nhập vào một string ngắn từ bàn phím, string này sẽ được lưu copy vào kernel space và lưu ở biến <code>msg</code> của module. Sau đó, khi người dùng ấn Enter, giá trị của msg sẽ được copy ngược lại ra user-app và in ra màn hình.
