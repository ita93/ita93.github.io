---
layout: post

category: Linux Kernel.

comments: true
---


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

# NOTE
# 1. __netdev_alloc_skb 
Cần biết Page fragment là gì trước
Một page fragment là một vùng nhớ có kích thước và vị trí bất kỳ nằm trong 0 hoặc trong một tổ hợp các page? Nếu một page có chứa nhiều fragment page thì Ref count của page sẽ được tính bằng tổng số ref count của các fragment.
Đề cấp phát một page fragment, Linux cung cấp các hàm: <code> page_frag_alloc</code> và <code>page_frag_free</code>. Những hàm này được sử dụng bởi network stack và network device driver để cung cấp một vùng memory sử dụng cho như sk_buff->head hoặc sử dụng cho frags của <code>skb_shared_info</code>
{% highlight c %}
/*
Cấp phát 1 skbuff instance cho rx (receive) cho device *dev.
Hàm này sẽ cấp phát một &sk_buff mới. Buffer sẽ có NET_SKB_PAD
headroom. Người dùng sẽ allocate một headroom với kích thước mà
họ cần (không được động tới built-in headroom)
*/
struct sk_buff *__netdev_alloc_skb(struct net_device *dev, unsigned int len,
				   gfp_t gfp_mask)
{
	struct page_frag_cache *nc;
	unsigned long flags;
	struct sk_buff *skb;	
	bool pfmemalloc;
	void *data;

	//Do mặc định sk_buff sẽ được thêm một headroom có kích thước
	//NET_SKB_PAD nên kích thước của sk_buff->data sẽ phải là len+NET_SKB_PAD
	len += NET_SKB_PAD; 
	
	//Nếu kích thước của data lớn hơn page size (aligned) thì không thể dùng cache được,
	//phải sử dụng alloc_skb để cấp phát một sk_buff hoàn tới mới (fresh new)
	if ((len > SKB_WITH_OVERHEAD(PAGE_SIZE)) ||
	    (gfp_mask & (__GFP_DIRECT_RECLAIM | GFP_DMA))) {
		skb = __alloc_skb(len, gfp_mask, SKB_ALLOC_RX, NUMA_NO_NODE);
		if (!skb)
			goto skb_fail;
		goto skb_success;
	}

	/*Trường hợp sử dụng page_frag_cache*/
	len += SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
	len = SKB_DATA_ALIGN(len);
	// Hiện tại len đã được align

	if (sk_memalloc_socks())	//Cái này không hiểu :(
		gfp_mask |= __GFP_MEMALLOC;

	//Disable Interrupt trên local CPU và lưu trạng thái lại
	local_irq_save(flags);

	//Dòng này có tác dụng tìm địa chỉ của biến netdev_alloc_cache ở cpu này
	//đây là một biến per cpu, tức là mỗi cpu sẽ có 1 instance có giá trị khách nhau.
	nc = this_cpu_ptr(&netdev_alloc_cache);
	//Dòng này như đã nói ở trên, dùng để cấp phát 1 page_fragment kích thước len.
	data = __alloc_page_frag(nc, len, gfp_mask);
	pfmemalloc = nc->pfmemalloc;

	//Re-enable Interrupt và restore trạng thái.
	local_irq_restore(flags);

	if (unlikely(!data))
		return NULL;

	//Khởi tạo các thông số cần thiết cho skb mới.
	skb = __build_skb(data, len);
	if (unlikely(!skb)) {
		skb_free_frag(data);
		return NULL;
	}

	/* use OR instead of assignment to avoid clearing of bits in mask */
	if (pfmemalloc)
		skb->pfmemalloc = 1;
	skb->head_frag = 1;

skb_success:
	skb_reserve(skb, NET_SKB_PAD);	//Không gian dành cho built-in headroom.
	skb->dev = dev;

skb_fail:
	return skb;
}
{% endhighlight %}

# 2. __alloc_skb()
{% highlight c %}
/*
Cấp phát một &sk_buff mới, không có head room hay tail room.
Kích thước data của sk_buff mới sẽ lớn hơn hoặc hằng size.
Reference count sẽ được set là 1.
*/
struct sk_buff *__alloc_skb(unsigned int size, gfp_t gfp_mask,
			    int flags, int node)
{
	struct kmem_cache *cache;
	struct skb_shared_info *shinfo;
	struct sk_buff *skb;
	u8 *data;
	bool pfmemalloc;
	
	//nếu cơ SKB_ALLOC_FCLONE được set thì sử dụng fclone cache thay vì head cache.
	cache = (flags & SKB_ALLOC_FCLONE)
		? skbuff_fclone_cache : skbuff_head_cache;
	//Sử dụng __GFP_MEMALLOC trong trường hợp SKB_ALLOC_RX được set và data được dùng cho writeback
	if (sk_memalloc_socks() && (flags & SKB_ALLOC_RX))
		gfp_mask |= __GFP_MEMALLOC;

	/* Get the HEAD */
	skb = kmem_cache_alloc_node(cache, gfp_mask & ~__GFP_DMA, node);
	if (!skb)
		goto out;
	prefetchw(skb);

	/* We do our best to align skb_shared_info on a separate cache
	 * line. It usually works because kmalloc(X > SMP_CACHE_BYTES) gives
	 * aligned memory blocks, unless SLUB/SLAB debug is enabled.
	 * Both skb->head and skb_shared_info are cache line aligned.
	 */
	size = SKB_DATA_ALIGN(size);
	size += SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
	data = kmalloc_reserve(size, gfp_mask, node, &pfmemalloc);
	if (!data)
		goto nodata;
	/* kmalloc(size) might give us more room than requested.
	 * Put skb_shared_info exactly at the end of allocated zone,
	 * to allow max possible filling before reallocation.
	 */
	size = SKB_WITH_OVERHEAD(ksize(data));
	prefetchw(data + size);

	/*
	 * Only clear those fields we need to clear, not those that we will
	 * actually initialise below. Hence, don't put any more fields after
	 * the tail pointer in struct sk_buff!
	 */
	memset(skb, 0, offsetof(struct sk_buff, tail));
	/* Account for allocated memory : skb + skb->head */
	skb->truesize = SKB_TRUESIZE(size);
	skb->pfmemalloc = pfmemalloc;
	atomic_set(&skb->users, 1);
	skb->head = data;
	skb->data = data;
	skb_reset_tail_pointer(skb);
	skb->end = skb->tail + size;
	skb->mac_header = (typeof(skb->mac_header))~0U;
	skb->transport_header = (typeof(skb->transport_header))~0U;

	/* make sure we initialize shinfo sequentially */
	shinfo = skb_shinfo(skb);
	memset(shinfo, 0, offsetof(struct skb_shared_info, dataref));
	atomic_set(&shinfo->dataref, 1);
	kmemcheck_annotate_variable(shinfo->destructor_arg);

	if (flags & SKB_ALLOC_FCLONE) {
		struct sk_buff_fclones *fclones;

		fclones = container_of(skb, struct sk_buff_fclones, skb1);

		kmemcheck_annotate_bitfield(&fclones->skb2, flags1);
		skb->fclone = SKB_FCLONE_ORIG;
		atomic_set(&fclones->fclone_ref, 1);

		fclones->skb2.fclone = SKB_FCLONE_CLONE;
		fclones->skb2.pfmemalloc = pfmemalloc;
	}
out:
	return skb;
nodata:
	kmem_cache_free(cache, skb);
	skb = NULL;
	goto out;
}
{% endhighlight %}