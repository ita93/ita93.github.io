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
	Một trong các pattern phổ biến trong kernel programming là khởi tạo một số activity bên ngoài current thread, sau đó đợi đến khi activity đó hoàn thành. Activity này có thể là tạo một kernel thread hoặc một user-space process mới, một request đến một process đã tồn tại, hoặc một số hardware-based action.
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

	Một completion thường là one-shot device, tức là nó chỉ được dùng 1 lần sau đó sẽ bị discard. Tuy nhiên, việc sử dụng lại một completion là khả dĩ. Nếu complete_all không được sử dụng, thì struct completion có thể được sử dụng lại một cách dễ dàng. Nếu complete_all đã dược gọi, thì completion struct cầ được tái tạo trước khi sử dụng với macro: <br/>
	<code>INIT_COMPLETION(struct completion c);</code><br/>
	Appendex: <code>void complete_and_exit(struct completion *c, long retval);</code>
