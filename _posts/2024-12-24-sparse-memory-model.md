---
layout: post

category: Linux device driver

comments: true
---
# Quản lý bộ nhớ vật lý trong Linux kernel (physical memory management)

Ai đã từng tìm hiểu về hệ điều hành đều biết rằng bộ nhớ vật lý được quản lý bằng cách chia nhỏ ra thành các vùng nhớ có kích thước bằng nhau, linux kernel gọi chúng là các pages, kích thước của page phụ thuộc vào kiến trúc của hệ thống, thông thường là 4KB, ví dụ ARM và X86. Thực tế thì tên gọi đúng của các vùng nhớ này là Physical frames, và mỗi frame được đại diện bởi một biến có kiểu `struct page` trong linux kernel. 
Có lẽ mọi người đều từng nghe tới cách kernel sử dụng buddy system để cấp phát các page khi chúng ta yêu cầu thông qua các api như `alloc_pages(mask, order)` hay `__get_free_page*`, tuy nhiên liệu bạn đã từng thắc mắc rằng liệu kernel theo dõi và lựa chọn page như thế nào để trả về thông qua các api này?
## Sparse memory model - cách kernel lưu physical frame trong hệ thống.
Nói nôm na thì physical memory model là cách đánh địa chỉ bộ nhớ của hệ thống, trong trường hợp đơn giản nhất, bộ nhớ vật lý của hệ thống có thể được đánh địa chỉ liên tục từ 0 đến maximum address. Tuy nhiên trong thực tế, dãy địa chỉ của bộ nhớ vật lý của thể không phải là liên tục, chẳng hạn như trong các NUMA system.
Đối với các hệ thống đơn giản, FLATMEM model có thể được sử dụng để quản lý không gian địa chỉ này. Trong FLATMEM model, một mảng global tên là `mem_map` sẽ được sử dụng để theo dõi toàn bộ không gian bộ nhớ vật lý, với mỗi phần tử của mảng là một `struct page` instance - mô tả thông tin của physical frame tương ứng. Trong model này, `page` tương ứng của mỗi PFN - physical frame number có thể dễ dàng được tìm ra vì chúng chỉ là linear mapping với phần tử đầu tiên của mảng mem_map sẽ có PFN là ARCH_PFN_OFFSET.
`pfn_to_page(u64 x) -> {return mem_map[x - ARCH_PFN_OFFSET]}`
Mặc dù vậy, model này lãng phí rất nhiều không gian bộ nhớ, vì nó tạo ra một `struct page` instance cho mỗi physical frame number có thể được có của Architecture, bất kể frame đó có thực sự tồn tại trong hệ thống đang chạy hay không.

Để giải quyết vấn đề này, SPARSEMEM model đã được ra: [https://lwn.net/Articles/134804/](sparse memory model patch)
Trong model này, bộ nhớ vật lý của hệ thống được chia ra thành các phần nhỏ hơn gọi là memory section, có kích thước phụ thuộc vào architecture, và được đại diện bởi một biến kiểu `struct mem_section`, điều đặc biệt của model này là nó sẽ chỉ tạo ra `page` instance trong hệ thống nếu physical frame tương ứng thực sự tồn tại, ngoài ra đây cũng là model duy nhất trong linux cho phép memory hotplug.
Model này có một sub setting là SPARSEMEM_VMEMMAP, tuy nhiên chúng ta sẽ tạm thời bỏ qua nó.
### memory section 
memory section là một đơn vị quản lý bộ nhớ vật lý được đưa vào linux kernel để phục vụ cho SPARSEMEM model, đây là một đơn vị đánh dấu vùng nhớ có kích thước lớn hơn page nhưng nhỏ hơn node, với x86-64, kích thước của nó là 128MB.
{% highlight c %} 
struct mem_section {
    unsigned long section_mem_map;
    struct mem_section_usage *usage;
};
{% endhighlight %}

Các `mem_section` của hệ thống được kernel lưu trữ trong một mảng hai chiều có tên là [`mem_section`]("https://elixir.bootlin.com/linux/v6.5-rc7/source/mm/sparse.c#L27"), mỗi element trong mảng này quản lý một vùng địa chỉ có kích thước 128MB (focus on x86_64), 
{% highlight c %}
struct mem_section mem_section[NR_SECTION_ROOTS][SECTIONS_PER_ROOT];
// put macros here for easier reference
#define SECTIONS_PER_ROOT   (PAGE_SIZE/sizeof(struct mem_section))
#define NR_SECTION_ROOTS DIV_ROUND(NR_MEM_SECTIONS, SECTION_PER_ROOT)
{% endhighlight %}

Có thể thấy rằng mỗi section root sẽ cần không gian nhớ 1 page để lưu trữ thông tin, tức là toàn bộ hệ thống của NR_SECTION_ROOTS để lưu trữ thông tin về bộ nhớ vật lý.
Với hệ thống X86_64 sử dụng 5 level page table (52 bits physical memory) thì cần tất cả 2^19 (`1 << (52 - 27)`)sections để theo dõi thông tin bộ nhớ vật lý, tương tự thì đối với hệ thống sử dụng 4 level page table sẽ là 2^11 sections.

### Truy cập pages trong SPARSEMEMORY model
Trường `section_mem_map`(unsigned long, nhưng thưc tế nó là một con trỏ tới một mảng 1 chiều) của `struct mem_section` chính là một con trỏ tới phần tử đầu tiên của một mảng có chứa PAGES_PER_SECTION phần tử, mỗi phần tử là một instance của `struct page` - đây chính là nơi kernel lưu trữ các page trong hệ thống.
```
    mem_section
    +-----------------------------+
    |usage                        |
    |   (mem_section_usage *)     |
    |                             |
    |                             |
    +-----------------------------+         mem_map[PAGES_PER_SECTION]
    |section_mem_map              |  ---->  +------------------------+
    |   (unsigned long)           |    [0]  |struct page             |
    |                             |         |                        |
    |                             |         +------------------------+
    +-----------------------------+    [1]  |struct page             |
                                            |                        |
                                            +------------------------+
                                       [2]  |struct page             |
                                            |                        |
                                            +------------------------+
                                            |                        |
                                            .                        .
                                            .                        .
                                            .                        .
                                            |                        |
                                            +------------------------+
                                            |struct page             |
                                            |                        |
                                            +------------------------+
                   [PAGES_PER_SECTION - 1]  |struct page             |
                                            |                        |
                                            +------------------------+
```
Lưu ý rằng, mem_section ứng với mỗi section index sẽ luôn tồn tại trong hệ thống, tuy nhiên mảng section_mem_map chỉ được allocate nếu như section này là hợp lệ - tức là có physical frame number nằm trong section này, do đó nó tiết kiệm được rất nhiều không gian bộ nhớ trong các hệ thống mà địa chỉ vật lý là không liên tục.
Điểm trừ của model này là các hàm như `__page_to_pfn` và `__pfn_to_page` có phần phức tạp hơn, do chúng phải tìm ra section trước khi có thể tìm ra page tương ứng.

### Quá trình khởi tạo sparse model trên x86_64
Qúa trình này bắt đầu tại bằng hàm `sparse_init()`, hàm này thường được gọi đến thống qua các lời gọi hàm trong `setup_arch()`
```
setup_arch() -> x86_init.paging.pagetable_init() -> paging_init() -> sparse_init()
```
![sparse_init](/images/mm/sparse_model.drawio.png)

Hàm memblocks_present có nhiệm vụ đánh dấu các mem section tồn tại trong hệ thống bằng cách set bit ```SECTION_MARKED_PRESENT``` của ```ms->section_mem_map```

Hàm ```sparse_init_nid(int nid, unsigned long pnum_begin, unsigned long pnum_end, unsigned long map_count)``` có nhiệm vụ khởi tạo sparse model cho node ```nid```, node này chứa các mem section nằm trong khoảng từ pnum_begin đến trước pnum_end. 
sparse_init_nid sẽ sử dụng các alloc api của memblock để allocate một mảng kiểu page[] gồm PAGES_PER_SECTION phần tử và encode địa chỉ của mảng này vào ms->section_mem_map.

```
map = __populate_section_memmap(pfn, PAGES_PER_SECTION,
				nid, NULL, NULL); // allocate a page array
		....
		check_usemap_section_nr(nid, usage);
		sparse_init_one_section(__nr_to_section(pnum), pnum, map, usage,
				SECTION_IS_EARLY); // encode PFN to ms->section_mem_map
```

Lưu ý là ms->section_mem_map sẽ được encode như sau:
```ms->section_mem_map = mem_map_addr - section_to_pfn(sec);
```
Trong đó mem_map_addr là địa chỉa của phần tử đầu tiên trong mảng []page, còn section_to_pfn(sec) sẽ trả về PFN của page đầu tiên trong section này.

Một số lower bit của ms->section_mem_map sẽ được sử dụng để lưu các flags 

### Truy cập thông tin từ Sparse model

```
#define __page_to_pfn(pg)					\
({	const struct page *__pg = (pg);				\
	int __sec = page_to_section(__pg);			\
	(unsigned long)(__pg - __section_mem_map_addr(__nr_to_section(__sec)));	\
})

#define __pfn_to_page(pfn)				\
({	unsigned long __pfn = (pfn);			\
	struct mem_section *__sec = __pfn_to_section(__pfn);	\
	__section_mem_map_addr(__sec) + __pfn;		\
})
```
Do ms->section_mem_map = start_page - start_pfn nên pg - ms->section_mem_map sẽ là ```pg - start_page + start_pfn ``` = PFN của pg. section_mem_map đã chứa luôn start_pfn của section mà không cần phải tạo thêm một trường khác để lưu nó.