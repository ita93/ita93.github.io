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
<code>sk_buff</code> chứa rất nhiều trường khác nhau, một số lương khổng lồ đủ để người ta biết được tất cả các thông tin cần thiết khi nhìn vào đấy. Các trường này có thể được chia vào 4 loại: Layout, General, Feature-generic, Management function.

## 1.  Layout Fields.
Những trường này có tác dụng giúp việc tìm kiếm một <code>sk_buff</code> variable trở nên dễ dàng hơn và tổ chức cấu trúc dữ liệu một cách hiệu quả là chính.

Các <code>sk_buff</code> trong Linux kernel được tổ chức thành các danh sách liên kết đôi, khi nhìn vào source code, có thể dễ dàng nhận thấy hai trường <code>next</code> và <code>prev</code> là hai con trỏ trỏ đến <code>sk_buff</code> bên trước và bên sau <code>sk_buff</code> hiện tại trong danh sách liên kết này. 
Ngoài ra có một cấu trúc dữ liệu <code>struct sk_buff_head</code> cũng được khai báo, nó cũng chứa 2 con trỏ <code>sk_buff*</code> tương tự như cấu trúc sk_buff, ngoài ra nó còn chứa <code>qlen</code> là độ dài của danh sách liên kết (các sk_buff). Cấu trúc này giúp cho việc tìm ra head của toàn bộ danh sách liên kết một cách dễ dàng hơn.
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

Ngoài ra trong loại Layout fields này còn có một số field đáng chú ý nữa:
<code>struct sock *sk</code> Đây là con trỏ trỏ tới cấu trúc dữ liệu <code>sock</code> của socket sở hữu buffer này. Con trỏ này được sử dụng khi dữ liệu được tạo/ hoặc nhận bởi một process, sử dụng ở TCP/UDP layer. <br/>

<code>unsigned int len</code> Đây là kích thước của khối dữ liệu trong buffer. Nó bao gồm cả kích thước của main buffer (trỏ đến bởi *head) và cả data trong fragments. Giá trị của nó thay đổi khi buffer được chuyển từ một network layer này sang một network layer khác, bởi vì khi đó, header sẽ được thêm vào (khi chuẩn bị gửi dữ liệu đi) hoặc cắt bớt (khi nhận dữ liệu).<br/>

<code>unsigned int data_len</code>: Kích thước của dữ liệu nằm trong fragment. <br/>

<code>usigned int mac_len</code>: Kích thước của MAC header. <br/>

<code>atomic_t users</code>: Trường này dùng để đánh dấu số lượng thực thể đang sử dụng sk_buff này, mục đích chính là tránh việc giải phóng <code>sk_buff</code> khi vẫn còn ai đó sử dụng nó. Điều này đòi hỏi mỗi người dùng (không phải người thật đâu :v ) phải tăng giá trị của trường này khi họ sử dụng <code>sk_buff</code> và giảm giá trị khi họ không còn dùng nó nữa. Lưu ý là trường này chỉ có ý nghĩa với cấu trúc <code>sk_buff</code>, nó không có ý nghĩa với dữ liệu thực sự mà <code>sk_buff</code> tham chiếu tới.<br/>

<code>unsigned int truesize</code> Trường này biểu diễn tổng kích thước của buffer, bao gồm cả chính <code>sk_buff</code> ( sizeof(struct sk_buff) ). Nó được khởi tạo bởi hàm <code>alloc_skb</code>: len+sizeof(sk_buff). Trường này sẽ được update khi <code>skb->len</code> thay đổi. <br/>

<code>usigned char *head, *end, *data, *tail</code>: Mấy thằng này rất quan trọng. <br/>

<code>void (*destructor)(...)</code>: Như tên gọi. <br/>

## 2.  General Fields.
Một số lượng lớn các trường của cấu trúc <code>sk_buff</code> là thuộc nhóm này.<br/>
<code>struct timeval stamp</code> Chỉ sử dụng đối với một packet được nhận từ host khác, ý nghĩa của nó là timestamp của packet. Giá trị của trường này được gán bởi hàm <code>netif_rx_internal</code> bằng cách gọi đến <code>net_timestamp_check</code>.
<code>struct net_device *dev</code> Giá trị của trường này phụ thuộc vào việc packet là nhận hay gửi. Về cơ bản thì trường này chính là device mà packet được nhận hay gửi đi.
<code>struct net_device *input_dev</dev> Device nhận packet. giá trị null nếu packet được tạo cục bộ. (trong cùng máy)
<code>struct net_device *real_dev</code> Được sử dụng bởi packet nằm trên virtual device.

<code> union{...} transport_header, network_header, mac_header </code> (trong kernel version cũ chúng lần lượt là h, nh, mac). Đây là các header của các network layer tương ứng.
<code> struct dst_entry dst</code> Được sử dụng bởi routing system.
<code>char cb[40]</code> cb là viết tắt của control buffer. Dùng để lưu thông tin của mỗi layer, và chỉ sử dụng trong layer đó. 40 bytes là đủ lớn để lưu trữ bất kỳ thông tin nào của mỗi layer.
<code>unsigned char ip_summed</code> và <code>unsigned int csum</code>: check sum và flag.
<code>unsigned char cloned</code> Được set nếu đây là clone của 1 <code>sk_buff</code> khác.
<code>unsigned char pkt_type</code> Phân loại frame dựa trên địa chỉ ở Layer 2. Các giá trị bao gồm: PACKET_HOST, PACKET_MULTICAST, PACKET_BROADCAST, PACKET_OTHERHOST, PACKET_OUTGOING, PACKET_LOOPBACK, PACKET_FASTROUTE.
<code>__u32 priority</code> Quality of service.
<code>unsigned short protocol</code> Đây là giao thức được sử dụng ở tầng tiếp theo (cao hơn) từ góc độ của device driver ở L2. Giá trị của trường này thường là: IP, IPv6 và ARP.

## 3.  Feature-Specific Fields.
Các trường này dùng cho các tính năng đặc biệt như QoS, Netfilter, và chúng chỉ tồn tại khi các kernel config tương ứng được enable, (thông qua các macro), mấy cái này kệ đi, khỏi quan tâm :v

## 4. Management Functions 
Các hàm này rất chi là quan trọng :v, dùng để điều chỉnh các thành phần của cấu trúc sk_buff, hoặc là các phần tử của list sk_buff. Thường có 2 phiên bản là <code>__function()</code> và <code>function()</code>, về cơ bản thì cái thứ 2 sẽ là bản đóng gói của cái thứ nhất, sử dụng các default argument, hoặc thực hiện những việc handle lock trước khi gọi đến cái thứ nhất...

### a. Các hàm cấp phát sk_buff.
Core function:
{% highlight c %}
struct sk_buff *__alloc_skb(unsigned int size, gfp_t gfp_mask,
			    int flags, int node)
{% endhighlight %}

Đối với một sk_buff instance, chúng ta cần phải thực hiện 2 việc: cấp phát vùng nhớ cho <code>struct sk_buff</code> và cấp phát vùng nhớ cho <code>skb->data</code>.
<code>sk_buff</code> và cả data buffẻ của nó đều được ưu tiên lấy ra từ cache.
Lưu ý là kích thước thực tế của bộ nhớ động được cấp phát có thể sẽ không phải là <code>size</code> như tham số truyền vào, vì nó sẽ được làm tròn để trở thành bội số của <code>SKB_DATA_ALIGN</code>.
### b. Các hàm điều chỉnh layout của dữ liệu chính.
-   <code>skb_reserve</code>
-   <code>skb_put</code>
-   <code>skb_pull</code>
-   <code>skb_push</code>