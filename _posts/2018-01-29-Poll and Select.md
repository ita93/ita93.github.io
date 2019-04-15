---
layout: post

category: Linux device driver

comments: true
---


## I. Poll and Select
Các ứng dụng(user-app) sử dụng nonblocking I/O sẽ thường sử dụng thêm <code>poll, select và epull </code>system call. <i>poll, select, epoll</i> có cùng 1 chức năng: cho phép một process xác định xem nó có thể đọc hoặc ghi từ một hoặc nhiều open file mà không bị block hay không. Những lời gọi hàm này cũng có thể block một process đến khi có một tập file descriptors(fd) trở nên khả dụng cho việc đọc hoặc ghi. Bởi thế, chúng thường được sử dụng trong các user-app cần phải sử dụng nhiều luồng đọc/ghi cùng lúc mà không có bất kỳ luồng nào bị stuck(V/d: network). <br/><br/>
Những system call kể trên được hỗ trợ bởi một driver method duy nhất là <code>poll</code>:<br/>
	<code>unsigned int (*poll) (struct file *filp, poll_table *wait);</code><br/>

Driver method được gọi mỗi khi một user-app thực hiện lời gọi đến một trong các system call <code>poll, select, epoll</code> trên một file descriptor được liên kết với driver. Nhiệm vụ của driver method có 2 bước:<br/>
1. Gọi hàm <code>poll_wait</code> trên một hoặc nhiều hàng đợi, hàng đợi này được yêu cầu là phải có thể chỉ ra được sự thay đổi của poll status. Nếu không có file descriptor nào khả dụng cho I/O ở thời điểm hiện tại, thì kernel sẽ đẩy process vào hàng đợi để đợi đến khi tất cả các file descriptors được truyền đến system call.<br/>
2. Trả về một bit mask miêu tả operation nào (nếu có) có thể được thực hiện ngay lập tức mà không bị block.<br/><br/>
Driver có thể thêm một wait queue vào <code>poll_table</code> bằng cách sử dụng lời gọi hàm: <br/>
<code>void poll_wait(struct file *, wait_queue_head_t *, poll_table *);</code><br/><br/>
Các bitmask có thể được trả về ở bước 2 gồm có:<br/>
<code>POLLIN</code> Bit này được set nếu như device có thể đọc mà không bị block<br/>
<code>POLLRDNORM</code>Bit này được set nếu như "normal" data là khả dụng cho việc đọc. Một readable device sẽ trả về (POLLIN|POLLRDNORM)<br/>
<code>POLLRDBAND</code>out-of-band data is available for reading from the device<br/>
<code>POLLPRI</code> High-priority data(out-of-band) có thể được đọc mà không bị blocking. Nếu bit này được set, lời gọi <code>select</code> sẽ thông báo rằng một exception condition đã diễn ra trong file, bời vì <code>select</code> report out-of-band data như một exception condition<br/>
<code>POLLHUP</code> Khi một process đọc đến EOF, driver phải set bit POLLHUP(hang-up). Một process gọi <code>select</code> sẽ được nói rằng device là có thể đọc, vì được yêu cầu bởi chức năng của hàm này.<br/>
<code>POLLERR</code> Một error condition đã xảy ra trong thiết bị. Khi <code>poll</code> được gọi, thiết bị được báo cáo là có thể đọc và cũng có thể ghi được, vì cả read và write đều trả về 1 error code without blocking<br/>
<code>POLLOUT</code>Device cÓ thể được ghi mà không bị block<br/>
<code>POLLWRNORM</code>Giống với POLLOUT<br/>
<code>POLLWRBAND</code>Nonzero-priority data có thể được ghi vào device<br/><br/>

Out of band data là gì? Data truyền qua socket.
<dev>
Peding for an example
</dev>

## II. Interaction with read and write.<br/>
Mục đích của việc gọi <code>poll</code> và <code>select</code> là để xác định trước xem một I/O operation sẽ bị block hay không. Poll và select rất hữu dụng, bởi vì chúng cho phép user-app chờ đợi nhiều data stream một cách đồng thời. (Sẽ tìm ví dụ sau).<br/><br/>
Để 3 system call (poll, select, epoll) hoạt động đúng, chúng ta cần lưu ý đến một số luật sau.<br/>
### 1. Đọc dữ liệu từ device.
a,Nếu có dữ liệu trong input buffer, <code>read</code> call sẽ return ngay tức thì, với độ trễ không đáng kể, kể cả khi như lượng dữ liệu trong input buffer ít hơn lượng dữ liệu được yêu cầu bởi user-app nhưng driver đảm bảo là lượng dữ liệu còn lại sẽ được chuyển đến input buffer sớm. Trong trường hợp này, <code>poll</code> nên trả về POLLIN|POLLRDNORM.<br/>
b,Nếu không có data trong input buffer, mặc định, <code>read</code> phải block đến khi có ít nhất 1 byte data ở trong input buffer. Ngược lại, nếu như O_NONBLOCK được set, <code>read</code> sẽ return ngay lập tức với giá trị trả về là -EAGAIN. Trong trường hợp này, <code>poll</code> phải report rằng device hiện tại là unreadable cho đến khi có ít nhất 1 byte data đc truyền đến. Ngay khi data đc truyền đến buffer, chúng ta sẽ quay lại case a.<br/>
c, Nếu đang ở EOF, <code>read</code> sẽ return ngay lập tức với giá trị trả về là 0. <code>poll</code> trả về POLLUP.<br/><br/>
### 2. Ghi vào device.
a,Nếu output buffer không đủ không gian trống, <code>write</code> sẽ return ngay lập tức. Nó có thể cấp nhận ít data hơn request, nhưng nó không chấp nhận việc không có bất cứ một byte dữ liệu nào. Trong trường hợp này, <code>poll</code> sẽ trả về POLLOUT|POLLWRNORM.<br/>
b,Nếu output buffer đầy, mặc định, <code>write</code> sẽ block đến khi có không gian trống. Nếu O_NONBLOCK được set, <code>write</code> sẽ ngay lập tức trả về giá trị -EAGAIN. Trong trường hợp này, <code>poll</code> sẽ thông báo là không thể ghi vào file được. Ngược lại, nếu device không thể tiếp nhận them dữ liệu, <code>write</code> sẽ trả về -ENOSPC (always)<br/>
c, Kể cả O_NONBLOCK không được set, thì <code>write</code> cũng không bao giờ đợi data transsmission trước khi return. Điều này là bởi vì nhiều ứng dụng sử dụng <code>select</code> để tìm xem <code>write</code> có bị block không? Nếu device là có thể ghi, system call phải bị block. Nếu như user-app đang sử dụng device muốn đảm bảo rằng data trong hàng đợi để vào output buffer đã được truyền đi thực sự, driver phải cung cấp <code>fsync</code> method. <br/><br/>

### 3. Flushing pending output
Như đã đề cập ở trên, <code>write</code> không có phương pháp nào để tính toán lượng output data. Để làm điều này driver phải cung cấp <i>fsync</i>:<br/>
<code>int(*fsync)(struct file *filp, struct detry *dentry, int datasync);</code><br/>
Khi user-app gọi đến <i>fsync</i>, lời gọi này chỉ nên được return khi device đã hoàn thành việc flush data (đẩy hết data trong output buffer sang device), bất kể việc làm này mất bao nhiêu thời gian đi chăng nữa thì nó cũng k nên đợi.<br/>

## III. The underlying data structure



