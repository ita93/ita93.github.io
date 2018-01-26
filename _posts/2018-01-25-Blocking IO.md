# BLOCKING I/O

Trong character driver, có implement hai hàm: read() và write(). Thử tưởng tượng, nếu như driver không thể thỏa mãn được request một cách tức thời, thì nó sẽ phản ứng như thế nào? Điều này tức là driver sẽ làm gì nếu như hàm <code>read()</code> được gọi khi dữ liệu chưa sẵn có, nhưng có thể nó sẽ có trong tương lai gần. Hoặc một process cố gắng thực hiện lệnh <code>write()</code> nhưng device chưa sẵn sàng nhận dữ liệu bởi vì buffer đang full. Trong trường hợp này, driver nên block process (user-space) lại, đưa nó vào tình trạng sleep cho đến khi các request có thể được thực thi.<br/>

## 1 Sleeping trong kernel
-Khi một process ở trạng thái sleep, nó sẽ bị remove khỏi scheduler's run queue cho đến khi statuc của nó được thay đổi bởi một sự kiện nào đó. <br/>
-Một process ở sleep status sẽ không được lập lịch trong CPU.<br/><br/>
Để một đoạn code có thể được đưa vào trạng thái sleep thì nó cần thỏa mãn các điều kiện sau đây:<br/>
-Rule 1: Không sleep khi đang trong một atomic context. Tức là driver không được sleep khi đang giữ spinlock, seqlock hoặc RCU lock.<br/>
-Rule 2: Không thể sleep khi đang có các disabled interrupt.<br/>
-Rule 3: Sleep trong semaphore là được phpes, nhưng cần phải cẩn thận. Nếu một đoạn code sleep trong khi nó đang giữ semaphore thì thread đang đợi semaphore đó cũng sẽ bị đưa vào trạng thái sleep, do đó cần đảm bảo rằng bạn không block luôn kthread sẽ đánh thức sleeping process.<br/>
-Rule 4: Khi một sleeping kthread được wake up thì nó không thể biết là nó đã bị loại ra khỏi CPU được bao lâu, và trong thời gian đó, đã xảy ra những thay đổi nào. Do đó, chúng ta không thể tạo ra một giả thuyết nào về trạng thái của hệ thống sau khi wake up kthread, mà phải kiểm tra xem những điều kiện cần thiết có đảm bảo không.<br/>
-Rule 5: Chỉ sleep khi chắc chắn rằng có một ai đó sẽ đánh thức bạn trong một thời điểm nào đó. Ngược lại hệ thống sẽ bị hang. Để làm được điều này thì có một yêu cầu nữa là awaker cần phải tìm được bạn để đánh thức.<br/><br/>

Chúng ta sẽ dùng một structure tên là <code>wait_queue_head_t</code> để thực hiện việc sleep của device driver. một queue head có thể được khởi tạo bằng các cách sau:<br/>
-Initialize statically: <code>DECLARE_WAIT_QUEUE_HEAD(name);</code>
-Initialize dynamicly: <br/>
<pre>
	wait_queue_head_t my_queue;
	init_waitqueue_head(&my_queue);
</pre>
<br/>
## 2.Simple Sleep
-Khi một process sleep, nó rơi vào trạng thái chờ đợi, chờ đợi một điều kiện nào đó sẽ đúng trong tương lại. Như đã đề cập, bất kỳ process nào sleep đều phải kiểm tra để chăc chắn rằng điều kiện nó chờ đợi thực sự được thỏa mãn khi nó tỉnh giấc. <br/>
-Linux cung cấp function family: <code>wait_event</code> để thực hiện sleeping:<br>
<code>wait_event(queue, condition)</code><br/>
<code>wait_event_interruptible(queue, condition)</code><br/>
<code>wait_event_timeout(queue,condition,timeout)</code><br/>
<code>wait_event_interruptible_timeout(queue, condition, timeout)</code><br/><br/>
-Sau khi đã sleep, linux cung cấp các hàm sau để đánh thức sleeping process:<br/>
<code>wake_event(wait_queue_head_t *queue)</code><br/>
<code>wake_event_interruptible(wait_queue_head_t *queue)</code><br/><br/>
Hàm <code>wake_up</code> đánh thức tất cả các process trong hàng đợi, phiên bản interruptible đánh thức các process thực hiện một interruptible sleep.<br/>
Lưu ý: interruptible sleep tức là các sleep có thể bị interrupt bởi một tác nhân bên ngoài, thông thường đây là cái chúng ta cần dùng.<br/><br/>

## 3.Blocking and Nonblocking Operations
Phần này sẽ nói về việc xác định xem khi nào chúng ta sẽ đưa process vào trạng thái sleep?<br/>

