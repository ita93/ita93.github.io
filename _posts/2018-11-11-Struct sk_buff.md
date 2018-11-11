---
layout: post

category: Linux Kernel.

comments: true
---

# Structure SK_BUFF

Cùng với <code>net_dev</code> thì <code>struct sk_buff</code> là cấu trúc dữ liệu cực kỳ quan trọng trong Network subsystem của Linux. Các network packet được lược lưu trữ bằng cấu trúc này. Cấu trúc này đuọc sử dụng bởi tất cả các tầng của mạng để lưu trữ header của chúng, các thông tin về payload (data được tạo ra bởi người dùng từ user-space), và các thông tin khác cần thiết cho việc giao tiếp mạng.
sk_buff được định nghĩa trong file header <code>include/linux/sk_buff.h</code>

Trong mạng 7 tầng, thì các tầng mà network subsystem cần phải xử lý(từ dưỡi lên trên là):
    Layer 2: Link Layer
    Layer 3: Network Layer (IP)
    Layer 4: Transport Layer (UDP/TCP)

<code>sk_buff</code> được sử dụng ở 3 layer này, và khi chuyển từ layer này sang layer khác thì một số trường của cấu trúc này bị thay đổi, do các layer sẽ thêm/cắt bỏ header của layer đó khi gửi/nhận packet.
<code>sk_buff<code> chứa rất nhiều trường khác nhau, một số lương khổng lồ đủ để người ta biết được tất cả các thông tin cần thiết khi nhìn vào đấy. Các trường này có thể được chia vào 4 loại: Layout, General, Feature-generic, Management function.

## 1.  Layout Fields.
Những trường này có tác dụng giúp việc tìm kiếm một <code>sk_buff</code> variable trở nên dễ dàng hơn và tổ chức cấu trúc dữ liệu một cách hiệu quả là chính.

Các <code>sk_buff</code> trong Linux kernel được tổ chức thành các danh sách liên kết đôi, khi nhìn vào source code, có thể dễ dàng nhận thấy hai trường <code>next</code> và <code>prev</code> là hai con trỏ trỏ đến <code>sk_buff</code> bên trước và bên sau <code>sk_buff</code> hiện tại trong danh sách liên kết này. 
Ngoài ra có một cấu trúc dữ liệu <code>struct sk_buff_head</code> cũng được khai báo, nó cũng chứa 2 con trỏ <code>sk_buff*<code> tương tự như cấu trúc sk_buff, ngoài ra nó còn chứa <code>qlen</code> là độ dài của danh sách liên kết (các sk_buff). Cấu trúc này giúp cho việc tìm ra head của toàn bộ danh sách liên kết một cách dễ dàng hơn.
{% highlight c %}
struct sk_buff_head{
    struct sk_buff *next;
    struct sk_buff *prev;
    __u32   qlen;
    spinlock_t  lock;
};
{% endhighlight %}

Nếu để ý, ta có thể thấy ngay là struct <code>sk_buff_head</code> này sẽ làm cho danh sách liên kết lưu các sk_buff biến thành một danh sách liên kết vòng (<code>sk_buff_head</code> vừa đóng vai trò là head, vừa đóng vai trò là tail.). Việc cấu trúc <code>sk_buff_head</code> có hai trường đầu tiên (next, prev) tương tự với <code>sk_buff</code> còn có một ưu điểm nữa, đó là chỉ cần implement một function để sử dụng cho cả 2 cấu trúc này.

<a href="{{ site.url }}/images/skbuff/list_of_skbuff.PNG"><img src="{{ site.url }}/images/skbuff/list_of_skbuff.PNG"></a>