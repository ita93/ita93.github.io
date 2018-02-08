---
layout: post

category: Linux device driver

comments: true
---

<code>kmalloc</code> rất hữu ích và tương đối dễ sử dụng, nó nhanh. Một đặc điểm của kmalloc là sau khi nó cấp pháp một vùng nhớ mới cho người yêu cầu, thì nội dung hiện tại của vùng nhớ này vẫn nằm nguyên, không hề bị xóa đi. Do Kernel là DMA nên allocated region là liên tục trên bộ nhớ vật lý. Bài này sẽ tìm hiểu về công cụ này.

## 1 Flag Argument.
Mình đã từng dùng malloc trong bài ioctl, tuy nhiên bây giờ mới đến đoạn tìm hiểu về nó (hí hí). Prototype của hàm này như sau:
{% highlight c %}
#include <linux/slab.h>
void *kmalloc(size_t size, int flags);
{% endhighlight %}
Hàm <code>kmalloc</code> nhận 2 tham số, tham số đầu tiên là kích thước của vùng nhớ sẽ được cấp phát. Tham số thứ hai, được gọi là <i>allocation flags</i>, là một cờ dùng để điều khiển hoạt động của kmalloc.<br/>
Flag phổ biến nhất là GFP_KERNEL, khi dùng flag này, việc cấp phát được thực hiện bằng cách gọi đến hàm <code>__get_free_pages</code>, GFP là tên viết tắt của hàm này. Cờ này dùng trong các process context. Sử dụng GFP_KERNEL có nghĩa là <code>kmalloc</code> có thể đưa process hiện tại vào hàng đợi để chờ đến khi có một page khả dụng (nếu bộ nhớ không còn đủ). Do có thể gây ra sleep nên hàm gọi đến <code>kmalloc</code> sử dụng cờ <code>GFP_KERNEL</code> không được phép chạy trong các atomic context (interrupts,...). Trong lúc process đi ngủ thì kernel sẽ thực hiện các hành động hợp lý để tìm được một vùng nhớ khả dụng, hoặc bằng cách đẩy hết buffer vào đĩa cứng, hoặc swapping mem từ một user-process.<br/><br/>
Đôi khi <code>kmalloc</code> cũng có thể được gọi từ bên ngoài process context, điều này có thể xảy ra trong các interrupt handlers, tasklets hay kernel timers. Trong trường hợp này, process hiện thời sẽ đi ngủ và driver code không thể sử dụng <code>GFP_KERNEL</code> được, thay vào đó, <code>GFP_ATOMIC</code> sẽ được dùng. Thông thường kernel sẽ cố gắng giữ một vài free pages nhằm mục đích xử lý các atomic allocation. Khi <code>GFP_ATOMIC</code> được sử dụng, <code>kmalloc</code> có thể sử dụng đến những page này, trong trường hợp không còn page nào khả dụng, thì việc cấp phát bộ nhớ sẽ thất bại.<br/><br/>
<code>GFP_KERNEL</code> và <code>GFP_ATOMIC</code> được gọi là các combinations of flags, hoặc các <i>allocation priorities</i>. Chúng được khai báo trong header <code>linux/gfp</code> bên cạnh các flag thông thường, các flag này thường có prefix là <b>__</b>. Kernel còn có một số <i>allocation priorities</i> cụ thể như sau:<br/>
<code>GFP_ATOMIC</code> : ĐƯợc sử dụng để cấp phát mem từ interrupt handler và các code nằm ngoài process context. Không bao giờ sleep, có thể thành công hoặc không.<br/>
<code>GFP_KERNEL</code> : Cấp phát bộ nhớ thông thường, có thể ngủ.<br/>
<code>GFP_USER</code> : Cấp phát bộ nhớ từ user-space pages; có thể ngủ.<br/>
<code>GFP_HIGHUSER</code> : Cấp phát bộ nhớ từ user-spaces pages, sử  dụng high memory.
<code>GFP_NOIO</code>, <code>GFP_NOFS</code>: Hoạt động giống như <code>GFP_KERNEL</code> nhưng chúng thêm các ràng buộc mà kernel cần làm để thỏa mãn yêu cầu. <code>GFP_NOFS</code> không cho phép thực hiện bất kỳ filesystem call nào, trong khi <code>GFP_NOIO</code> không cho phép khởi tạo bất kỳ I/O nào. Mục đích chính của các cờ này là để sử dụng trong các file system và virtual mem code, nơi mà việc allocation được cho phép ngủ, nhưng việc thực hiện các lời gọi filesystem không phải là ý kiến hay.<br/><br/>
Các flag nêu trên có thể được kết hợp với một số flag sau đây bằng phép bitwise OR:<br/>
<code>__GFP_DMA</code> Yêu cầu việc cấp phát phải diễn ra trong DMA-capable mem zone.<br/>
<code>__GFP_HIGHMEM</code> Chỉ ra rằng bộ nhớ cấp phát có thể được đặt ở high mem.<br/>
<code>__GFP_COLD</code> Thông thường, memory allocator cố gắng trả về các page được tìm thấy trong processor cache - <i>cache warm page</i>. Cờ này sẽ yêu cầu các <i>cold page</i>. Cờ này hữu ích trong trường hợp cấp phát page để đọc DMA.<br/>
<code>__GFP_NOWARN</code> Đây là cờ hiếm khi được sử dụng. no warning.</br>
<code>__GFP_HIGH</code> Đánh dấu request có độ ưu tiên cao, khẩn cấp.
<code>__GFP_REPEAT</code> Trong trường hợp cấp phát không thành công, nó sẽ thử lại (nhưng có thể việc cấp phát vẫn lỗi).<br/>
<code>__GFP_NOFAIL</code> Việc cấp phát không bao giờ được thất bại, nó sẽ thử lại đến khi nào thành công thì thôi.<br/>
<code>__GFP_NORETRY</code> Bỏ cuộc ngay khi vùng nhớ yêu cầu không khả dĩ.<br/><br/>

Trong phần giải thích các flag ở trên, có đề cập đến Memory zone, vậy mem zone là gì?<br/>
Trong linux kernel có ít nhất 3 mem zone: DMA, normal và high memory. Thông thường việc cấp phát diễn ra ở <i>normal</i> zone, tuy nhiên có thể được setup bằng cách sử dụng một số cờ nhất định.
DMA mem là bộ nhớ cho phép thực hiện DMA access. <i>High mem</i> là kỹ thuật sử dụng để cho phép truy cập đến một lượng memory tương đối lớn trên 32-bit platforms.<br/>

## 2. Size argument
Kernel quản lý system's physical mem, nó chỉ khả dụng trong các page-sized chunk. Kernel sử dụng page-oriented allocation technique để cấp phát bộ nhớ (thay vì head-oriented như malloc).<br/>
Linux xử lý các yêu cầu cấp phát bộ nhơ bằng cách tạo ra một tập các pool của các mem objects có kích thước cố định. Các yêu cầu cấp phát được xử lý bằng cách tìm đến một pool đang giữ số lượng object đủ lớn và trao toàn bộ một mem chunk cho người đã request.<br/>
Kernel chỉ có thể cấp phát một mảng cố định về kích thước (đã được định nghĩa từ trước) các bytes. Nếu bạn yêu cầu một lượng bộ nhớ bất kỳ, thông thường bạn sẽ nhận được nhiều hơn một ít những gì bạn yêu cầu (fragmentation), có thể lên đến gấp đôi.<br/>
memory chunk được cấp phát bởi <i>kmalloc</i> có giới hạn tùy thuộc vào config và arch, lớn nhất là 128KB thì phải. Trong trường hợp cần nhiều bộ nhớ hơn, thì <code>kmalloc</code> không phải là lựa chọn tốt nhất.<br/>

## 3. Lookaside caches.