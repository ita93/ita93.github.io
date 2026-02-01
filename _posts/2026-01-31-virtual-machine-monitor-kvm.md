---
layout: post

category: Virtualization

comments: true
---
# Virtual Machine Monitor với KVM và Rust
Thử viết một VMM dựa trên KVM API bằng Rust trong đôi ngày cuối tuần.

## I. Vài khái niệm cơ bản
### Virtualization - ảo hóa
Về cơ bản thì ảo hóa là kỹ thuật mô phỏng lại môi trường phần cứng của hệ thống (trong các máy ảo - virtual machine). Kỹ thuật này tạo ra một lớp trừu tượng tương tự nhưng không đồng nhất với môi trường phần cứng đang host nó.
Mỗi virtual machine (VM) có một môi trường tách biệt và có các "phần cứng" mong muốn của nó, mặc dù thực tế thì tất các các VM trên cùng một máy vật lý sẽ chia sẻ cùng một tài nguyên, và tài nguyên này được kiểm soát chặt chẽ bởi một phần mềm đặc biệt của hệ thống gọi là hypervisor.
Ảo hóa giải quyết được nhiều vấn đề của hệ thống tính toán:
- Sử dụng tài nguyên hiệu quả: Thay vì mỗi server chỉ chạy 1 OS và sử dụng 1 phần nhỏ của CPU, chúng ta có thể chạy hàng chục VM trên đó và sử dụng phần lớn tài nguyên của hệ thống.
- Tạo tra một môi trường cô lập với phần còn lại của hệ thống. Mỗi máy ảo là một bản giả lập bằng phần mêm của hệ thống máy tính, nếu OS của một VM bị crash thì nó chỉ ảnh hưởng đến chính VM đó. 
- Linh hoạt: Việc khởi tạo một máy ảo mới diễn ra nhanh chóng và các VMM hiện nay đều cho phép migrate máy ảo giữa các host mà không có downtime.
- Có khả năng chạy các phần mềm/hệ điều hành cũ.

### Virtual machine monitor (VMM) và hypervisor
VMM là một phần mềm tạo và quản lý máy ảo, chằng hạn như Virtual Box, Qemu, HyperV (desktop app)
Thông thường VMM và hypervisor thường được dùng lẫn lộn, mặc dù vậy, theo định nghĩa nguyên thủy thì:
- VMM tập trung vào khía cạnh quản lý và giám sát.
- Hypervisor nhấn mạnh rằng "thứ này" có phân quyền cao hơn so với Kernel: Hypervisor vs supervisor.
Còn theo như cái sự hiểu biết của bản thân tui, thì hypervisor là component thực hiện việc ảo hóa VCPU.
### Hypervisor Type 1 và Type 2
Các hypervisor được chia làm hai loại
#### Type 1: Bare-metal Hypervisor
Chạy trực tiếp trên phần cứng, không thông qua việc sử dụng bất kỳ hệ điều hành nào, nó thực hiện đầy đủ các chức năng của hệ điều hành.
```
┌─────────┐ ┌─────────┐ ┌─────────┐
│  VM 1   │ │  VM 2   │ │  VM 3   │
│ (Guest) │ │ (Guest) │ │ (Guest) │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     └───────────┴───────────┘
                 │
     ┌───────────▼───────────┐
     │   Type 1 Hypervisor   │
     │ (runs on bare metal)  │
     └───────────┬───────────┘
                 │
     ┌───────────▼───────────┐
     │       Hardware        │
     └───────────────────────┘
```
Ví dụ: VMware ESXi, Microsoft Hyper-V, Xen, KVM (with Linux as the host)
Do hypervisor loại 1 chạy trực tiếp trên phần cứng nên nó mang lại hiệu năng cao hơn và an toàn hơn (do nó không chứa các component thừa thãi của một OS thông thường)
#### Type 2: 
Là một phần mềm sử dụng lại sức mạnh của một OS thông thường
```
┌─────────┐ ┌─────────┐
│  VM 1   │ │  VM 2   │
│ (Guest) │ │ (Guest) │
└────┬────┘ └────┬────┘
     │           │
     └─────┬─────┘
           │
┌──────────▼──────────┐
│  Type 2 Hypervisor  │  ◄── Runs as an application
│  (VirtualBox, etc.) │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│   Host OS (Windows, │  ◄── Regular operating system
│   macOS, Linux)     │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│      Hardware       │
└─────────────────────┘
```
Ví dụ: VirtualBox (TCG), VMware Workstation, Parallels Desktop, QEMU (TCG no KVM)
Mặc dù dễ dàng cài đặt và có khả năng tận dụng các phần mềm khác chạy trên cùng hệ điều hành tuy nhiên hypervisor loại này mang lại overhead và ít an toàn hơn.

## KVM
KVM hoạt động như một kernel module của Linux kernel, được merge vào kernel từ năm 2007. Nó tận dụng khả năng ảo hóa hệ thống của intel CPU - Vtx (sau này nó có support thêm AMD-V, S390, ARM nữa) để tạo ra các VCPU với overhead gần như bằng 0.

### Intel Vtx
là một extension của intel cpu:
- CPU có một privilege level mới gọi là "Guest mode"
- Guest mode chạy trực tiếp trên CPU ở tốc độ gần như native.
- Đối với một số priviledge instruction mà Guest mode không được phép thực hiện thì CPU sẽ tự động chuyển về host mode và tạo ra exception để hypervisor xử lý.
### KVM API
KVM sử dụng ioctl thông qua /dev/kvm để cho phép VMM tạo và quản lý máy ảo. KVM chỉ emulate một vài peripheral quan trọng, còn lại các thiết bị ngoại vi khác phải được emulate bởi VMM, ví dụ như block, network, etc...

Theo như định nghĩa ở trên thì KVM là một đứa nhập nhằng không biết xếp vô loại 1 hay 2, nhưng quan trọng là nó chạy được :v
```
┌─────────┐ ┌─────────┐   ┌────────────────┐
│  VM 1   │ │  VM 2   │   │ Regular Apps   │
│ (Guest) │ │ (Guest) │   │ (Firefox, etc) │
└────┬────┘ └────┬────┘   └───────┬────────┘
     │           │                │
     └─────┬─────┘                │
           │                      │
┌──────────▼──────────────────────▼────────┐
│                Linux Kernel              │
│  ┌─────────────┐                         │
│  │ KVM Module  │  ◄── Turns Linux into   │
│  │             │      a hypervisor       │
│  └─────────────┘                         │
└──────────────────────┬───────────────────┘
                       │
┌──────────────────────▼───────────────────┐
│   Hardware (Intel VT-x / AMD-V)          │
└──────────────────────────────────────────┘
```

## II. Tự viết VMM bằng Rust.
[`GITHUB`]("https://github.com/ita93/rust-kvm-tool/tree/main")

Thực tế thì việc viết một VMM sử dụng Rust khá là đơn giản bằng cách sử dụng các crate có sẵn của rust-vmm, tuy nhiên do mục tiêu học tập và tìm hiểu, mình sẽ không sử dụng các wrapper/helper của rust-vmm, thay vào đó sẽ sử dụng raw libc function để tương tác với kvm. Tuy vậy, việc viết/gen lại các hằng số cơ bản khác mất thời gian, nên đối với các hằng số, mình sẽ sử dụng lại định nghĩa sẵn của rust-vmm.
Về cách sử dụng của KVM API, trên LWN đã có một bài hướng dẫn khá đầy đủ [`Using KVM API`]("https://lwn.net/Articles/658511/") . Cơ bản thì nó có mấy bước sau:
- S1: Mở KVM handle (/dev/kvm) để lấy descriptor, thông qua fd này, vmm có thể tương tác với KVM.
- S2: Tạo VM bằng cách sử dụng KVM_CREATE_VM ioctl trên kvm fd vừa mở.
- S3: Cung cấp memory region cho VM để sử dụng.
- S4: Khởi tạo và config các VCPU. (mình chỉ có 1 thôi)
- S5: Load guest code vào memory (vùng memory đã cấp phát ở S3).
- S6: Handle các VM exit event.
Bây giờ sẽ đi vào các bước cụ thể
### Bước 1: Lấy file descriptor của KVM handle và khởi tạo VM
Việc lấy File descriptor có thể được thực hiện thông qua hàm open() của libc crate:
{% highlight rust %} 
     let kvm_path = CString::new("/dev/kvm")?;
     let kvm_fd = unsafe { libc::open(kvm_path.as_ptr(), O_RDWR) };
{% endhighlight %}

Mình cũng tạo ra một struct mới để giữ file descriptor này nhằm tiện lợi hơn trong việc sử dụng:
{% highlight rust %} 
struct Kvm {
    fd: c_int,
}

impl Kvm {
    fn new() -> Result<Self> {
        let kvm_path = CString::new("/dev/kvm")?;
        let kvm_fd = unsafe { libc::open(kvm_path.as_ptr(), O_RDWR) };

        if kvm_fd >= 0 {
            Ok(Self { fd: kvm_fd })
        } else {
            Err(io::Error::last_os_error()).context("failed to open /dev/kvm")
        }
    }
}
{% endhighlight %}
Ở đây, mình đã lưu fd vào một trường fd của struct Kvm, tuy nhiên, do đây là raw libc code nên nó không tận dụng được các tính năng safety của Rust để tự động gọi close() khi object đi hết vòng đời. Một cách để khắc phục điều này là implement Drop cho struct KVM
{% highlight rust %} 
impl Drop for Kvm {
    fn drop(&mut self) {
        unsafe {
            libc::close(self.fd);
        }
    }
}
{% endhighlight %}

Dù vậy, cách này trông không ngầu, không thông dụng cho lắm, thay vào đó, thường ta sẽ sử dụng [`OwnedFd`]("https://doc.rust-lang.org/beta/std/os/fd/struct.OwnedFd.html") để wrap raw file descriptor, OwnedFd đảm bảo rằng file sẽ được close() bởi nó ở cuối vòng đời và không một thực thể nào khác có thể close() file descritor nó đang sở hữu ngoài nó. 
Cấu trúc Kvm được update thành:
{% highlight rust %} 
struct Kvm {
    fd: OwnedFd,
}

impl Kvm {
    fn new() -> Result<Self> {
        let kvm_path = CString::new("/dev/kvm")?;
        let kvm_fd = unsafe { libc::open(kvm_path.as_ptr(), O_RDWR) };

        if kvm_fd >= 0 {
            Ok(Self {
                fd: unsafe { OwnedFd::from_raw_fd(kvm_fd) },
            })
        } else {
            Err(io::Error::last_os_error()).context("failed to open /dev/kvm")
        }
    }
}
{% endhighlight %}
Ngoài ra `struct Kvm` còn một hàm setup() dùng để kiểm tra và xác thực KVM API version thông qua KVM_GET_API_VERSION (xem trên github)
Sau khi đã có được KVM fd, chúng ta có thể khởi tạo một VM thông qua ioctl KVM_CREATE_VM:
{% highlight rust %} 
let fd = unsafe { ioctl(kvm.fd.as_raw_fd(), KVM_CREATE_VM, 0) };
{% endhighlight %}
Để thuận tiện cho việc sử dụng, một struct cũng được định nghĩa để giữ các biến liên quan đến một VM:
{% highlight rust %} 
pub struct VM {
    kvm: Kvm,
    pub fd: OwnedFd,
}
impl VM {
     pub fn new() -> Result<Self> {
          let kvm = Kvm::new()?;
          kvm.setup()?;
          let fd = unsafe { ioctl(kvm.fd.as_raw_fd(), KVM_CREATE_VM, 0) };
          // ... create VM instance
     }
}
{% endhighlight %}
tương tự như đối với KVM handle, nếu KVM_CREATE_VM thành công, file descriptor trả về sẽ được lưu vào trường fd của `struct VM`

### Bước 2: Memory cho VM
Một máy ảo cần RAM. Nhưng khác với máy vật lý, nơi RAM là phần cứng, “RAM” của máy ảo thực chất chỉ là một vùng bộ nhớ của máy host mà chúng ta dành riêng cho máy guest sử dụng.

Điểm mấu chốt là: địa chỉ vật lý của guest không phải là địa chỉ vật lý của host. Khi guest truy cập địa chỉ 0x1000, nó không được phép truy cập trực tiếp vào bộ nhớ của host tại địa chỉ đó. Thay vào đó, KVM sẽ dịch các địa chỉ vật lý của guest sang các địa chỉ ảo của host thông qua một cơ chế ánh xạ (mapping) do chúng ta cung cấp.
```
Guest Virtual Address (GVA)
        ↓  (guest page table)
Guest Physical Address (GPA)
        ↓  (EPT / NPT)
Host Physical Address (HPA)
```

Chúng ta cần cung cấp địa chỉ vùng nhớ dự định sử dụng cho VM với KVM handle, thông qua ioctl, nhưng trước hết mình sẽ define một struct để việc quản lý và giải phóng vùng nhớ này được dễ dàng hơn [`memory.rs`]("https://github.com/ita93/rust-kvm-tool/blob/main/src/memory.rs")
{% highlight rust %} 
pub(crate) struct MemoryWrapper {
    // NonNull verifies the pointer isn't null and is useful for optimization
    ptr: NonNull<c_void>,
    size: usize,
}
{% endhighlight %}
Mình viết ra vài method cho cấu trúc này, tuy nhiên có 2 method quan trọng nhất là new() dùng để tạo ra một vùng nhớ với kích thước yêu cầu thông qua hàm mmap của libc, và hàm drop() có tác dụng unmap vùng nhớ này ở cuối vòng đời
mmap() với MAP_ANON|MAP_SHARED sẽ tạo ra vùng nhớ anonymous và cho phép share với thread khác.
{% highlight rust %} 
 pub fn new(mem_size: u64) -> Result<Self> {
        let addr = unsafe {
            mmap(
                core::ptr::null_mut(),
                mem_size as usize,
                PROT_READ | PROT_WRITE,
                MAP_ANON | MAP_SHARED,
                -1,
                0,
            )
        };
    /*...*/
 }

// Release the memory automatically 
 impl Drop for MemoryWrapper {
    fn drop(&mut self) {
        unsafe {
            if munmap(self.ptr.as_ptr(), self.size) != 0 {
                eprintln!(
                    "Warning: munmap failed: {}",
                    std::io::Error::last_os_error()
                );
            } else {
                println!("Guest memory unmapped successfully.");
            }
        }
    }
}
{% endhighlight %}

Như đã đề cập ở trên, vmm cần thông báo cho kvm về vùng nhớ này thông qua một ioctl: KVM_SET_USER_MEMORY_REGION
{% highlight rust %} 
impl VM {
// ....
    pub fn insert_memory_region(&mut self, slot: u32, flags: u32, 
                                mem_size: u64, guest_pa: u64) -> Result<()> {
        let host_memory = MemoryWrapper::new(mem_size)?;
        let mem_region = kvm_userspace_memory_region {
            slot,                           // Memory slot index
            flags,                          // KVM_MEM_LOG_DIRTY_PAGES, etc.
            userspace_addr: host_memory,    // Host virtual address
            memory_size: mem_size,          // Size of the region
            guest_phys_addr: guest_pa,      // Where guest sees this memory
        };

        let ret = unsafe {
            ioctl(self.fd.as_raw_fd(), KVM_SET_USER_MEMORY_REGION, &mem_region, 0)
        };
        // ...
    }
//....
}
{% endhighlight %}
`kvm_userspace_memory_region` là một struct được định nghĩa tuân theo quy định của KVM API:
{% highlight rust %} 
#[repr(C)]
#[derive(Default)]
struct kvm_userspace_memory_region {
    pub slot: u32,                     // An index for this memory slot
    pub flags: u32,                    // Flags
    pub guest_phys_addr: u64,          // Where the guest thinks its RAM starts
    pub memory_size: u64,              // How many bytes of RAM
    pub userspace_addr: MemoryWrapper, // userspace_addr
}
{% endhighlight %}
Ở đây  `repr(C)` có tác dụng yêu cầu rustc biểu diễn layout của struct này theo tiêu chuẩn của ngôn ngữ C, MemoryWrapper là struct ta đã định nghĩa ở trên.

### Bước 3: VCPU cho VM
CPU là bộ phận tối quan trọng đối với một máy tính, và máy ảo cũng không phải ngoại lệ.
Một trong những function quan trọng nhất trong quá trình khởi tạo VM là `create_vcpu`. Đây không chỉ đơn thuần là tạo ra một File Descriptor (FD), mà là thiết lập một vùng nhớ chia sẻ (shared memory) để chương trình Rust của chúng ta (ở userspace) và KVM (ở kernelspace) có thể "nói chuyện" với nhau mà không tốn quá nhiều chi phí copy dữ liệu.
VCPU sẽ được định nghĩa như sau
{% highlight rust %} 
pub struct VCpu {
    pub fd: OwnedFd,
    pub run_ptr: NonNull<kvm_run>,
    pub mmap_size: usize,
}
{% endhighlight %}
Khai báo cấu trúc `kvm_run` - đây là cấu tạo của message sẽ được trao đổi giữa kvm và vmm khi có exit event. Về cơ bản thì cấu trúc này được gen lại từ linux-header, thông tin cụ thể về các field và type của các field có thể xem ở [`kvm_run`]("https://docs.rs/kvm-bindings/latest/kvm_bindings/struct.kvm_run.html")
{% highlight rust %} 
#[repr(C)]
#[derive(Debug)]
struct kvm_run {
    pub request_interrupt_window: u8,
    pub immediate_exit: u8,
    pub padding1: [u8; 6],
    pub exit_reason: u32,
    pub ready_for_interrupt_injection: u8,
    pub if_flag: u8,
    pub flags: u16,
    pub cr8: u64,
    pub apic_base: u64,
    pub __bindgen_anon_1: kvm_run__bindgen_ty_1,
    pub kvm_valid_regs: __u64,
    pub kvm_dirty_regs: __u64,
    pub s: kvm_run__bindgen_ty_2,
}
{% endhighlight %}
Trước khi tạo VCPU, chúng ta cần biết cấu trúc giao tiếp kvm_run sẽ chiếm bao nhiêu bộ nhớ. Kích thước này không cố định mà phụ thuộc vào kiến trúc CPU.

{% highlight rust %} 
let map_size = unsafe { ioctl(self.kvm.fd.as_raw_fd(), KVM_GET_VCPU_MMAP_SIZE, 0) };
{% endhighlight %}

Nếu bước này thất bại (map_size < 0), chúng ta không thể tiếp tục vì không biết phải cấp phát bao nhiêu RAM để giao tiếp với kernel.
Tiếp theo, tạo VCPU bằng KVM_CREATE_VCPU ioctl, tương tự như với VM và KVM handle, kết quả trả về là một fd
{% highlight rust %} 
let vcpu_fd = unsafe { ioctl(self.fd.as_raw_fd(), KVM_CREATE_VCPU, 0) };
{% endhighlight %}

Đây là đoạn code quan trọng nhất. Chúng ta sử dụng mmap để ánh xạ vùng nhớ của kernel (nơi chứa trạng thái VCPU) vào không gian địa chỉ của tiến trình Rust hiện tại.
Tại sao phải làm vậy? Cấu trúc kvm_run chứa thông tin cực kỳ quan trọng như: Tại sao VM dừng lại? (Exit Reason), dữ liệu I/O, trạng thái thanh ghi,... Thay vì dùng syscall để copy dữ liệu này ra vào liên tục (rất chậm), mmap cho phép cả User và Kernel cùng đọc/ghi trực tiếp vào một vùng RAM.
{% highlight rust %} 
let addr = unsafe {
    mmap(
        core::ptr::null_mut(),
        map_size as usize,
        PROT_READ | PROT_WRITE, // Cho phép đọc và ghi
        MAP_SHARED,             // Quan trọng: Thay đổi sẽ được thấy bởi cả 2 phía
        vcpu_fd,
        0,
    )
};
{% endhighlight %}


Cuối cùng, vẫn như cũ, ta đóng gói kết quả vào VCpu struct và lưu và `VM` để tận dụng Drop của rust cho RAII
{% highlight rust %} 
self.vcpus_fd.push(VCpu {
    fd: unsafe { OwnedFd::from_raw_fd(vcpu_fd) }, // Quản lý vòng đời FD tự động
    run_ptr: kvm_run_mmap,                        // Con trỏ tới vùng nhớ chia sẻ
    mmap_size: map_size as usize,
});
{% endhighlight %}

Đến lúc này, nếu bạn load code vào vùng nhớ đã cấp phát (guest memory) và gọi KVM_RUN ioctl trên VCPU vừa tạo, thì VMM đã có thể thực thi code, tuy vậy, chỉ có thể là code 16bit. Cơ bản thì mình không biết làm gì với code 16bit ngoài in hello world, nên nó không được Kun, do đó mục tiêu của cái VMM này là load code 64bit. Nên ta cần chuyển đổi mode của VCPU từ 16 bit thành 64bit trước.

### Bước 4: Bước vào thế giới 64bit.
Khi một CPU thuộc kiến trúc x86 được bật lên, nó luôn bắt đầu ở real mode. Đây là chế độ 16‑bit rất cũ, được giữ lại để tương thích với bộ vi xử lý Intel 8086.
Tuy nhiên, các hệ điều hành hiện đại — như Linux 64‑bit — cần chạy trong long mode, tức chế độ 64‑bit.
Vì vậy, trong quá trình khởi động, hệ thống phải thực hiện một loạt bước cấu hình để chuyển CPU từ real mode → protected mode → long mode. Chỉ khi vào long mode, CPU mới có thể chạy mã 64‑bit, truy cập không gian địa chỉ lớn, và sử dụng các tính năng hiện đại của kiến trúc x86‑64. Việc này yêu cầu: 
- Setup Page tables: 64bit mode yêu cầu page table phải được setup từ trước.
- Cung cấp Segment descriptors.
#### 4.1 Thiết lập Page Table (Identity Mapping)
Tại sao sử dụng Identity mapping?
Trên x86, tất cả các OS (bao gồm cả Linux, mac, windows) đều phải sử dụng identity mapping ở early boot, bởi vì code ở early boot của OS được hardcode bằng địa chỉ vật lý. Tuy nhiên khi paging được enabled địa chỉ CPU nhìn thấy là địa chỉ ảo, do đó nếu ánh xạ này không phải là đồng nhất thì CPU sẽ không thể tìm được địa chỉ của câu lệnh tiếp theo trong code => Tripple fault -> Crash.
Sau khi khởi động thành công, OS (có thể) sẽ thay thế page table ban đầu bằng page table của chính nó.
{% highlight rust %} 
fn enter_long_mode(&self, mem: &mut [u8]) -> Result<()> {
    // Page table locations in guest physical memory
    let pml4_addr: u64 = 0x2000;  // Page Map Level 4
    let pdpt_addr: u64 = 0x3000;  // Page Directory Pointer Table
    let pd_addr: u64 = 0x4000;    // Page Directory

    // Page table entry flags
    let flags: u64 = (1 << 0)     // Present - page is valid
                   | (1 << 1);    // Read/Write - page is writable

    // PML4[0] points to PDPT
    // This single entry covers the first 512GB of address space
    let entry = pdpt_addr | flags;
    mem[0x2000..0x2008].copy_from_slice(&entry.to_le_bytes());
    
    // PDPT[0] points to PD
    // This single entry covers the first 1GB of address space
    let entry = pd_addr | flags;
    mem[0x3000..0x3008].copy_from_slice(&entry.to_le_bytes());
    
    // PD entries: each covers 2MB (using huge pages)
    // 512 entries × 2MB = 1GB of identity-mapped memory
    let huge_page_flags = flags | 0x80;  // PS bit = Page Size (2MB pages)
    for i in 0..512 {
        let phys_addr = i * 0x200000;    // 2MB aligned physical address
        let entry = phys_addr | huge_page_flags;
        let offset = 0x4000 + (i as usize * 8);
        mem[offset..offset + 8].copy_from_slice(&entry.to_le_bytes());
    }
    // ...
}
{% endhighlight %}
Chức năng của từng Bit
CR0 (Control Register 0):
PE (bit 0): Chuyển từ chế độ thực (real mode) sang chế độ bảo vệ (protected mode).
PG (bit 31): Enable paging. Sau khi bật, VA sẽ được thông dịch thông qua page tables.

CR3 (Control Register 3):
Chứa địa chỉ vật lý của PML4 
Khi CPU cần dịch một địa chỉ, nó sẽ bắt đầu từ vị trí được chỉ định ở đây.

CR4 (Control Register 4):
PAE (bit 5): Kích hoạt mở rộng địa chỉ vật lý (Physical Address Extension). Đây là điều kiện bắt buộc cho Long Mode vì phân trang 64-bit sử dụng định dạng PAE.
PSE (bit 4): Cho phép sử dụng các trang lớn (huge pages) kích thước 2MB và 4MB.

EFER (Extended Feature Enable Register - Thanh ghi kích hoạt tính năng mở rộng):
LME (bit 8): Long Mode Enable – Thông báo cho CPU rằng chúng ta muốn sử dụng chế độ 64-bit.
LMA (bit 10): Long Mode Active – Bit này được CPU tự động thiết lập khi chế độ 64-bit thực sự được kích hoạt thành công.

Đoạn code trên tạo ra một page table với 4 levels (CPU cùi bắp không cần suport L5) như sau:
![page_tables](/images/vmm/page_tables.jpeg)
##### Cấu trúc Page Table Entry - PTE
Mỗi địa chỉ ảo sẽ có kích thước 8 bytes
Bit 0: Present  - Nếu bằng 0, việc truy cập trang này sẽ gây ra lỗi (fault).
Bit 1: Read/Write (Đọc/Ghi) - Nếu bằng 0, việc ghi dữ liệu sẽ gây ra lỗi.
Bit 7 (PS): Page Size (Kích thước trang) - Do chúng ta sử dụng 2MB page size nên bit này bằng 1 ở PD (tui nhớ không nhầm thì linux kernel setup identity mapping đến page size 1G - nếu khả dĩ)
Bits 12-51: Địa chỉ vật lý của cấp tiếp theo (hoặc của trang cuối cùng).
##### Kết quả
Sau khi thiết lập các bảng trang này:
Địa chỉ ảo 0x0 ánh xạ tới địa chỉ vật lý 0x0.
Địa chỉ ảo 0x1000 ánh xạ tới địa chỉ vật lý 0x1000.
Điều này tiếp diễn cho đến 1GB (tương đương 512 trang × 2MB mỗi trang).
Kernel (nhân hệ điều hành) có thể thực thi mà không cần lo lắng về việc chuyển đổi địa chỉ (do địa chỉ ảo và vật lý trùng khớp nhau).

Nếu setup của bạn là đúng thì PTE đầu tiên sẽ có giá trị là `0x0000000000000083` (0x->0x, PS, R/W, P)
#### 4.2 Segment descriptor và GDT
Về Segment descritpr và GDT trên intel, có thể tìm hiểu trên wiki. Cơ bản thì đối với 64-bit mode, những descriptor này chỉ có tác dụng giúp CPU kiểm tra privilege:
- CPU cần kiểm tra xem segment có hợp lệ và present không.
- Kiểm tra CS register để xem nó đang chạy ở mode nào (64-bit hay 32-bit compatibilty mode)
Để VMM có thể chuyển VCPU vào Guest mode, chúng ta cần cung cấp các segment hợp lệ.
Helper để tạo segment:
{% highlight rust %} 
#[repr(C)]
#[derive(Debug, Copy, Clone, Default)]
pub struct kvm_segment {
    pub base: u64,
    pub limit: u32,
    pub selector: u16,
    pub type_: u8,
    pub present: u8,
    pub dpl: u8,
    pub db: u8,
    pub s: u8,
    pub l: u8,
    pub g: u8,
    pub avl: u8,
    pub padding: u8,
}
/* Đoạn code này mình copy ở đâu về mà quên rồi */
const fn seg_with_st(selector_index: u16, type_: u8) -> kvm_segment {
    kvm_segment {
        base: 0,              // Base address (ignored in long mode for most segments)
        limit: 0xffffffff,    // Segment limit (ignored in long mode)
        selector: selector_index << 3,  // Selector = index × 8 (each GDT entry is 8 bytes)
        type_,                // Segment type (code/data, permissions)
        present: 1,           // Segment is valid
        dpl: 0,               // Privilege level 0 (kernel)
        db: if type_ == 0b1011 { 0 } else { 1 },  // D/B flag
        s: 1,                 // Not a system segment
        l: if type_ == 0b1011 { 1 } else { 0 },   // Long mode bit (code only)
        g: 1,                 // Granularity (limit × 4KB)
        avl: 0,               // Available for OS use
        padding: 0,
    }
}

// Code segment: selector index 1, type 0b1011 (Execute/Read, accessed)
const CODE_SEG: kvm_segment = seg_with_st(1, 0b1011);

// Data segment: selector index 2, type 0b0011 (Read/Write, accessed)  
const DATA_SEG: kvm_segment = seg_with_st(2, 0b0011);
{% endhighlight %}

Lưu ý rằng, đối với  64-bit code, thì D bắt buộc phải bằng 0 khi L=1
Sau khi đã có CODE và Data segment, ta cần ghi chúng vào memory của VM, mình sẽ chọn ghi vào ở 0x500
{% highlight rust %} 
fn enter_long_mode(&self, mem: &mut [u8]) -> Result<()> {
// ...
    let gdt_table: [u64; 3] = [
        0,                       // Entry 0: NULL descriptor (required)
        to_gdt_entry(&CODE_SEG), // Entry 1: Code segment
        to_gdt_entry(&DATA_SEG), // Entry 2: Data segment
    ];

    // Copy GDT to guest memory at address 0x500
    mem[0x500..0x500 + gdt_bytes.len()].copy_from_slice(gdt_bytes);

    // Tell the CPU where the GDT is
    sregs.gdt.base = 0x500;
    sregs.gdt.limit = std::mem::size_of_val(&gdt_table) as u16 - 1;
//...
}
{% endhighlight %}
Cơ bản thì code trên sẽ copy toàn bộ slice gdt_table vào `mem` - đây chính là vùng memory chúng ta đã mmap (dành cho VM memory, không phải kvm_run). 
Ở đây, sregs là biến kiểu `kvm_sregs` lưu trữ các thông tin về các special register của kvm
Lúc này memory sẽ trông như này:
```
Address 0x500:
┌────────────────────────┐
│ 0x0000000000000000     │ NULL descriptor (index 0)
├────────────────────────┤
│ 0x00AF9A000000FFFF     │ Code segment (index 1, selector 0x08)
├────────────────────────┤
│ 0x00CF92000000FFFF     │ Data segment (index 2, selector 0x10)
└────────────────────────┘
```

Tuy nhiên KVM vẫn chưa có ý thức gì về các register và GDT này, chúng ta cần phải nói cho nó biết thông qua ioctl `KVM_SET_SREGS`
Đối với mỗi registers này, chúng ta đều cần lấy giá trị hiện tại của nó thông qua ioctl `KVM_GET_SREGS` để đảm bào không ghi đè giá trị không mong muốn lên các sreg mà chúng ta không muốn động tới.
{% highlight rust %} 
// Get current special registers
let mut sregs = kvm_sregs::default();
unsafe { ioctl(self.fd.as_raw_fd(), KVM_GET_SREGS, &mut sregs, 0) };

// CR3: Page table base address
sregs.cr3 = pml4_addr;  // 0x2000 - tells CPU where page tables are

// CR0: System control flags
sregs.cr0 |= X86_CR0_PE;  // 0x1 - Protection Enable (enter protected mode)
sregs.cr0 |= X86_CR0_PG;  // 0x80000000 - Paging Enable

// CR4: Extended features
sregs.cr4 |= X86_CR4_PAE;  // 0x20 - Physical Address Extension (required for long mode)
sregs.cr4 |= X86_CR4_PSE;  // 0x10 - Page Size Extension (allow 2MB/4MB pages)

// EFER: Extended Feature Enable Register (MSR)
sregs.efer |= EFER_LME;   // 0x100 - Long Mode Enable
sregs.efer |= EFER_LMA;   // 0x400 - Long Mode Active

unsafe { ioctl(self.fd.as_raw_fd(), KVM_SET_SREGS, &sregs, 0) };
{% endhighlight %}

### Bước 5: Runnnnn the Guest
```
VMM calls KVM_RUN
        │
        ▼
┌───────────────────┐
│ KVM prepares VMCS │ (Virtual Machine Control Structure)
│ loads guest state │
└─────────┬─────────┘
          │
          ▼
    ┌───────────┐
    │ VM Entry  │ ── CPU switches to guest mode
    └─────┬─────┘
          │
          ▼
   Guest code runs directly on CPU (near-native speed)
          │
          │ (I/O instruction, interrupt, etc.)
          ▼
    ┌───────────┐
    │ VM Exit   │ ── CPU switches back to host mode
    └─────┬─────┘
          │
          ▼
┌───────────────────┐
│ KVM saves guest   │
│ state, returns    │
│ to VMM            │
└─────────┬─────────┘
          │
          ▼
VMM examines exit_reason and handles it
```
Ở bước này mình sẽ load một đoạn code đơn giản vào bộ nhớ của máy ảo để kiểm thử :v, Guest code sẽ được load vào địa chỉ vật lý 0x100000 của máy ảo.
{% highlight rust %} 
impl VM {
// ....
    pub fn load_code(&self, code: &Vec<u8>) -> Result<()> {
        unsafe {
            std::ptr::copy_nonoverlapping(
                code.as_ptr(),
                self.memory_regions.userspace_addr.as_mut_ptr().add(0x100000)
                code.len(),
            );
        };
        Ok(())
    }
// ....
}

// in main.rs
vm.load_code(&code)?;
{% endhighlight %}

Đoạn code 64bit sẽ sử dụng là : [`hello.S`]("https://github.com/ita93/rust-kvm-tool/blob/main/64bits-baremetal/hello.S")
Lưu ý rằng chúng ta sẽ load compiled binary file vào bộ nhớ chứ không phải text file này. 
compile: `nasm -f elf64 -o hello.o hello.asm`
Sau khi load code vào memory, KVM vẫn chưa biết rằng nó phải bắt đầu chạy code từ đâu vì chúng ta chưa setup RIP cho nó.
Mình sẽ viết một function mới để phục vụ việc này
{% highlight rust %} 
impl VCpu {
    //....
    fn init_registers(&self, rip: __u64, rsi: __u64) -> Result<()> {
        let mut regs = kvm_regs::default();
        unsafe { ioctl(self.fd.as_raw_fd(), KVM_GET_REGS, &mut regs, 0) };

        regs.rflags = 2;   // Bit 1 must always be set (reserved)
        regs.rip = rip;    // Instruction pointer = kernel entry point
        regs.rsi = rsi;    // RSI = pointer to boot_params (Linux boot protocol)

        unsafe { ioctl(self.fd.as_raw_fd(), KVM_SET_REGS, &mut regs, 0) };
        Ok(())
    }
//....
}
{% endhighlight %}

Cuối cùng, cần một hàm để yêu cầu VCPU bắt đầu thực thi Guest code
{% highlight rust %} 
impl VCpu {
    //....
    fn run(&self) -> Result<()> {
        loop {
            let ret = unsafe { ioctl(self.fd, KVM_RUN, 0) };
            if ret < 0 {
                return Err(io::Error::last_os_error())
                    .context("failed to get start kvm run");
            }
            let kvm_run = unsafe { self.run_ptr.as_ref() };
            println!("KVM EXIT {:?}\n", kvm_run);
        }
    }
//....
}
{% endhighlight %}

Kết hợp các hàm lại với nhau để bắt đầu quá trình thực thi của máy ảo
{% highlight rust %} 
impl VM{
    // ...
    pub fn run(&mut self) -> Result<()> {
        // actually we only have one cpu
        for vcpu in &self.vcpus_fd {
            vcpu.init_registers(0x100000, 0)?;
            vcpu.run()?;
        }
        Ok(())
    }
    // ...
}

// In main.rs
let mut vm = VM::new().expect("unable to open VM fd");
vm.insert_memory_region(0, 1u32, 250 * 1024 * 1024, 0)?;
vm.create_vcpu()?;
vm.enter_long_mode()?;
vm.load_code(&code)?;
vm.run()?;
{% endhighlight %}
Lúc này ta có thể build và chạy code với:
```  cargo run -- 64bits-baremetal/guest.bin ```
và hy vọng nó sẽ in ra hello world.
Nhưng..., nó chỉ in ra một đống "KVM_EXIT" gì đó liên tục, lý do là KVM sẽ trả về output tại VM_EXIT bằng cách ghi dữ liệu vào vùng địa chỉ đã mapping của `kvm_run` instance, tuy nhiên chúng ta chưa hề handle nó, do đó chúng ta cần quay lại hàm `VCpu.run()` để xử lý việc này
{% highlight rust %} 
impl VCpu {
    //....
    fn run(&self) -> Result<()> {
        // ..... Giữ lại đoạn code ở trên (có thể xóa phần println đi)
        match kvm_run.exit_reason {
            KVM_EXIT_IO => {
                    let io = unsafe { kvm_run.__bindgen_anon_1.io };
                    let port = io.port;
                    let dir = io.direction as u32;
                    if dir == KVM_EXIT_IO_OUT && port == 0x3f8u16 {
                    let offset = unsafe { kvm_run.__bindgen_anon_1.io.data_offset } as usize;
                    let character =
                        unsafe { *((self.run_ptr.as_ptr() as *mut u8).add(offset)) } as char;
                    print!("{}", character);
                    _ = std::io::stdout().flush();
                }
            }
            KVM_EXIT_HLT => {
                println!("Guest executed HLT. Stopping.");
                break Ok(());
            }
            _ => {}
        };
    }
    // .....
}
{% endhighlight %}
Để kiểm tra xem VM Exit có phải xảy ra do có dữ liệu I/O không, ta kiểm tra exit reason, nếu exit reason là KVM_EXIT_IO thì VCPU đã quay lại host mode do I/O. Lúc này nếu dữ liệu là OUT và port là `0x3f8u16` thì ta có thể đọc dữ liệu ra bằng cách copy data ở địa chi `offset` tính từ `kvm_run` pointer.

Thành quả là nó sẽ in ra:
```Hello, KVM!Guest executed HLT. Stopping.```

Như vậy ta đã có một VMM cơ bản có thể thực thi được Guest code 64bit.

## III. Boot vmlinux
Tuy nhiên nếu chỉ có in ra mỗi hello world thì nó cũng không có gì thú vị, và việc setup một đống code chỉ để in Hello world nó khá thừa thãi và cũng không có chỗ nào nhìn giống một cái VM.
Dể demo rõ ràng hơn về VMM, chúng ta sẽ nâng cấp nó để nó có thể boot được một vmlinux cơ bản.
Đầu tiên cần chuẩn bị một file `vmlinux`, có thể tự compile, hoặc download nó về từ 
```
https://github.com/ita93/rust-kvm-tool/blob/main/64bits-baremetal/vmlinux.bin
```
Đây là một file ELF
```
phinguyendp@phi:~/Rust/rust-kvm-tool$ file 64bits-baremetal/vmlinux.bin
64bits-baremetal/vmlinux.bin: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=7c8bd33f36cb5eed93f1c724ab40b407198bbe6c, not stripped
```

Thử chạy luôn với `cargo run -- vmlinux.bin` xem nó hoạt động như thế nào
```
phinguyendp@phi:~/Rust/rust-kvm-tool$ cargo run -- ../mini-vmm/vmlinux.bin
warning: `rust-kvm-tool` (bin "rust-kvm-tool") generated 12 warnings (run `cargo fix --bin "rust-kvm-tool" -p rust-kvm-tool` to apply 3 suggestions)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.68s
     Running `target/debug/rust-kvm-tool ../mini-vmm/vmlinux.bin`
Succeffully loading guest code to memory (size: 21441304)

```
Ok, nó chỉ in ra một dòng log về kích thước file đã load vào, và không còn gì khác cả, lý do là để boot được linux, ta cần tuân thủ bootprotocol của nó.
### Linux boot protocol cho x86
Trước hết cần đọc một tý về boot protocol ở doc của linux [`https://www.kernel.org/doc/html/v6.1/x86/boot.html`]("https://www.kernel.org/doc/html/v6.1/x86/boot.html"), hoặc hỏi thầy Gemini cho nhanh. Do đó trước khi cho VCPU thực thi code, ta cần phải cung cấp boot parameter hợp lệ và lưu nó vào địa chỉ nhớ yêu cầu.
Để tiết kiệm thời gian cho việc code, thì mình đã copy file định nghĩa các cấu trúc và hằng số liên quan đến boot parameter của linux từ [`linux-loader`]() repository (cơ bản thì file này được gen ra từ linux-header bằng cách sử dụng rust-bindgen) và lưu nội dung vào `src/bootparams.rs` . Trong đây có `struct boot_params` là đối tượng chính mà chúng ta cần quan tâm.
{% highlight rust %} 
#[repr(C, packed)]
#[derive(Copy, Clone)]
pub struct boot_params {
    pub screen_info: screen_info,
    pub apm_bios_info: apm_bios_info,
    pub _pad2: [__u8; 4usize],
    pub tboot_addr: __u64,
    pub ist_info: ist_info,
    pub acpi_rsdp_addr: __u64,
    pub _pad3: [__u8; 8usize],
    pub hd0_info: [__u8; 16usize],
    pub hd1_info: [__u8; 16usize],
    pub sys_desc_table: sys_desc_table,
    pub olpc_ofw_header: olpc_ofw_header,
    pub ext_ramdisk_image: __u32,
    pub ext_ramdisk_size: __u32,
    pub ext_cmd_line_ptr: __u32,
    pub _pad4: [__u8; 112usize],
    pub cc_blob_address: __u32,
    pub edid_info: edid_info,
    pub efi_info: efi_info,
    pub alt_mem_k: __u32,
    pub scratch: __u32,
    pub e820_entries: __u8,
    pub eddbuf_entries: __u8,
    pub edd_mbr_sig_buf_entries: __u8,
    pub kbd_status: __u8,
    pub secure_boot: __u8,
    pub _pad5: [__u8; 2usize],
    pub sentinel: __u8,
    pub _pad6: [__u8; 1usize],
    pub hdr: setup_header,
    pub _pad7: [__u8; 36usize],
    pub edd_mbr_sig_buffer: [__u32; 16usize],
    pub e820_table: [boot_e820_entry; 128usize],
    pub _pad8: [__u8; 48usize],
    pub eddbuf: [edd_info; 6usize],
    pub _pad9: [__u8; 276usize],
}
{% endhighlight %}

Có hai việc cần làm đối với `boot_params`, một là thiết lập các trường của header, và 2 là cấu hình E820 memory map.
Đầu tiên, đối với header, nó được lưu trữ ở `boot_params.hdr` của biến, chúng ta cần thiết lập một số thông số sau:
- `boot_params.hdr.type_of_loader`: Loại boot loader chúng ta dùng để boot linux, do chúng ta không dùng boot loader nào trong danh sách được cung cấp bởi linux kernel nên ta set trường này thành `0xFF`
- `boot_params.hdr.boot_flags`: Bắt buộc phải là 0xAA55, đây là magic number quy định linux kernel.
- `boot_params.hdr.header`: Đây cũng là một magic number, có giá trị là:  `“HdrS” (0x53726448)`
- `boot_params.hdr.cmd_line_ptr`: Địa chỉ của kernel command line. (lưu ý là buộc phải nằm trong vùng 32bit)
- `boot_params.hdr.cmdline_size`: Kích thước của command line, tối đa là 255
- `boot_params.hdr.loadfags`: bitmask về các option load kernel. Ở đây mình dùng: CAN_USE_HEAP \| 0x01 \| KEEP_SEGMENTS, nghĩa là: load protected-mode code ở 0x100000 (bit 0x1), heap_end_ptr là valid
- `boot_params.hdr.heap_end_ptr`: The end of setup stack/heap tính từ real-mode code
{% highlight rust %} 
fn setup_boot_params(boot_params: &mut boot_params, cmdline_addr: u32, total_mem: u64) {
    // 1. Bootloader identification
    // 0xFF = "Unregistered bootloader" - Linux accepts any value here
    boot_params.hdr.type_of_loader = 0xFF;
    
    // Magic numbers Linux checks to verify boot_params is valid
    boot_params.hdr.boot_flag = 0xAA55;        // Same as MBR signature
    boot_params.hdr.header = 0x53726448;       // "HdrS" in ASCII
    
    // 2. Command line setup
    // This is how we pass "console=ttyS0" to the kernel
    boot_params.hdr.cmd_line_ptr = cmdline_addr;  // 0x20000 in our case
    boot_params.hdr.cmdline_size = CMDLINE.len() as u32;
    
    // 3. Heap configuration
    // The kernel uses this for early allocations before proper memory management
    boot_params.hdr.loadflags = CAN_USE_HEAP | 0x01 | KEEP_SEGMENTS;
    boot_params.hdr.heap_end_ptr = 0xFE00;
// ...... bellow is e820 setup
{% endhighlight %}

CMDLINE được định nghĩa như sau:
{% highlight rust %} 
const CMDLINE: &[u8] = b"console=ttyS0 earlyprintk=ttyS0";
{% endhighlight %}

Tiếp theo chúng ta cần setup E820 table.
`E820 là cấy chi?` E820 memory map là một bảng mô tả layout bộ nhớ vật lý, được firmware (BIOS/UEFI) cung cấp cho hệ điều hành trong giai đoạn boot. Nó trả lời câu hỏi cốt lõi:
“Vùng địa chỉ vật lý nào là RAM dùng được, vùng nào phải tránh?”
Một entry E820 thường có dạng:
```
(start_address, size, type)
```
Trong đó type có thể là:
E820_TYPE_RAM: RAM dùng được
E820_TYPE_RESERVED: không được đụng tới
E820_TYPE_ACPI, NVS: vùng ACPI, firmware

Ví dụ:
```
Ví dụ:
0x00000 ┌─────────────────┐
        │   Low Memory    │ ◄── Usable RAM (640KB)
        │   (E820_RAM)    │
0x9FC00 ├─────────────────┤
        │   Legacy Hole   │ ◄── Reserved for VGA, ROMs, etc.
        │   (not mapped)  │
0x100000├─────────────────┤
        │   High Memory   │ ◄── Usable RAM (rest of memory)
        │   (E820_RAM)    │
        │                 │
        └─────────────────┘
```
Trong môi trường ảo hóa, E820 trong VM không đến từ BIOS thật, mà do VMM / hypervisor tạo ra.
Linux kernel trong guest không phân biệt được là đang chạy trên phần cứng vật lý thật hay máy ảo
Nó chỉ biết tin vào E820 map mà nó nhận được lúc boot.
Mình thiết lập e820 table trong boot_params như sau:
{% highlight rust %} 
fn setup_boot_params(boot_params: &mut boot_params, cmdline_addr: u32, total_mem: u64) {
//......
    // 4. E820 Memory Map - THIS IS CRITICAL
    // Linux uses E820 to know what memory is available
    boot_params.e820_entries = 2;
    
    // Low memory: 0 to 640KB (conventional memory)
    // The "hole" from 640KB to 1MB is reserved for legacy hardware
    boot_params.e820_table[0].addr = 0x0;
    boot_params.e820_table[0].size = 0x9FC00;  // 639KB
    boot_params.e820_table[0].type_ = E820_RAM;
    
    // High memory: 1MB to end of RAM
    // This is where the kernel and most data lives
    boot_params.e820_table[1].addr = 0x100000;  // 1MB
    boot_params.e820_table[1].size = total_mem - 0x100000;
    boot_params.e820_table[1].type_ = E820_RAM;
}
{% endhighlight %}

Nếu bạn sử dụng `dmesg` trên máy linux của bạn thì bạn sẽ thấy có nhiều entry hơn nữa (tầm chục cái), tuy nhiên để tiết kiệm thời gian và vì lười nên mình chỉ tạo ra một table đơn giản đủ dùng.
Sau khi đã thiết lập xong `boot_params`, chúng ta copy nó vào memory của VM ở địa chỉ 0x10000
{% highlight rust %} 
pub fn load_code(&self, vmlinux: &Vec<u8>) -> Result<__u64> {
    //... declare variables ...
    Self::setup_boot_params(
        &mut boot_params,
        ADDR_CMDLINE as u32, // ADDR_CMDLINE is defined as and const (0x20000)
        self.memory_regions.memory_size,
    );
    // ... load kernel ...
    
    // Copy boot_params to 0x10000 (64KB)
    unsafe {
        std::ptr::copy_nonoverlapping(
            &boot_params as *const boot_params as *const u8,
            self.memory_regions.userspace_addr.as_mut_ptr().add(0x10000),
            size_of_val(&boot_params),
        );
        
        // Copy command line to 0x20000 (128KB)
        std::ptr::copy_nonoverlapping(
            CMDLINE.as_ptr(),  // "console=ttyS0 earlyprintk=ttyS0"
            self.memory_regions.userspace_addr.as_mut_ptr().add(0x20000),
            CMDLINE.len(),
        );
    }
}
{% endhighlight %}

Vậy là đã có một function đơn  giản để thiết lập boot protocol, tiếp theo ta cần update hàm `load_code()` để có thể load được file vmlinux.bin vào bộ nhớ
### Load vmlinux.bin
Trước hết cần hiểu rằng vmlinux.bin là một raw ELF không nén.
ELF (Executable and Linkable Format) là một định dạng chuẩn, dùng để mô tả:
- Mỗi section/segment phải được nạp vào vị trí nào trong bộ nhớ
- Mỗi section cần quyền gì (đọc / ghi / thực thi)
- Địa chỉ entry point (nơi CPU bắt đầu chạy)
Vì vậy, không thể chỉ copy nguyên file vào RAM – ta phải parse ELF và load từng segment vào đúng địa chỉ của nó.
Đây là program headers của file vmlinux.bin này, thực tế chúng ta chỉ cần load các vùng nhớ được chỉ ra bởi header thuộc type LOAD (PT_LOAD) vào memory, theo đúng địa chỉ (Offset) được nêu ra trong các headers (lưu ý là chúng ta đã setup 1 vùng identity mapping từ trước)
```
phinguyendp@phi:~/Rust/rust-kvm-tool/64bits-baremetal$ readelf --program-headers vmlinux.bin
Elf file type is EXEC (Executable file)
Entry point 0x1000000
There are 5 program headers, starting at offset 64
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x0000000000b72000 0x0000000000b72000  R E    0x200000
  LOAD           0x0000000000e00000 0xffffffff81c00000 0x0000000001c00000
                 0x00000000000b0000 0x00000000000b0000  RW     0x200000
  LOAD           0x0000000001000000 0x0000000000000000 0x0000000001cb0000
                 0x000000000001f658 0x000000000001f658  RW     0x200000
  LOAD           0x00000000010d0000 0xffffffff81cd0000 0x0000000001cd0000
                 0x0000000000133000 0x0000000000413000  RWE    0x200000
  NOTE           0x0000000000a031d4 0xffffffff818031d4 0x00000000018031d4
                 0x0000000000000024 0x0000000000000024         0x4
                 ....
```
Việc copy dữ liệu từ vmlinux.bin vào guest memory được thực hiện trong `load_code()` như sau:
{% highlight rust %} 
pub fn load_code(&self, vmlinux: &Vec<u8>) -> Result<__u64> {
    // Parse ELF headers
    let elf = ElfFile::new(vmlinux).map_err(|e| anyhow::anyhow!(e))?;
    
    // Iterate through program headers (segments)
    for ph in elf.program_iter() {
        // Only load PT_LOAD segments (actual code/data)
        if ph.get_type().unwrap_or(xmas_elf::program::Type::Null)
            == xmas_elf::program::Type::Load
        {
            let offset = ph.offset() as usize;      // Offset in ELF file
            let file_size = ph.file_size() as usize; // Bytes in file
            let mem_size = ph.mem_size() as usize;   // Bytes in memory
            let paddr = ph.physical_addr() as usize; // Guest physical address

            unsafe {
                let dest = self.memory_regions.userspace_addr.as_mut_ptr();
                let src = vmlinux[offset..(offset + file_size)].as_ptr();

                // Copy segment from ELF to guest memory
                std::ptr::copy_nonoverlapping(src, dest.add(paddr), file_size);

                // Zero BSS (mem_size > file_size means uninitialized data)
                if mem_size > file_size {
                    std::ptr::write_bytes(
                        dest.add(paddr + file_size), 
                        0, 
                        mem_size - file_size
                    );
                }
            };
        }
    }
    
    // Return the entry point - where execution starts
    Ok(elf.header.pt2.entry_point())
}
{% endhighlight %}

Ở đây `elf.header.pt2.entry_point()` chính là entry point của kernel, ta return nó như kết quả của load_code(), cho phép caller của function này sử dụng nó để thiết lập `RIP`
Tại thời điểm này:
- Kernel code được được đặt ở vị trí mong muốn trong memory.
- Các biến toàn cục đã được khởi tạo (zeros for BSS)
Tại thời điểm này, việc load dữ liệu vào memory coi như hoàn thành, Cở bản layout của Guest memory sẽ như sau (Cursor vẽ :v):
```
┌────────────────────────────────────────┐ 0x0
│       Interrupt Vector Table           │ (not used in our minimal setup)
├────────────────────────────────────────┤ 0x500
│       GDT (Global Descriptor Table)    │ 24 bytes (3 entries × 8 bytes)
├────────────────────────────────────────┤ 0x2000
│       PML4 (Page Map Level 4)          │ 4KB, 1 entry used
├────────────────────────────────────────┤ 0x3000
│       PDPT (Page Directory Ptr Table)  │ 4KB, 1 entry used
├────────────────────────────────────────┤ 0x4000
│       PD (Page Directory)              │ 4KB, 512 entries (1GB mapped)
├────────────────────────────────────────┤ 0x10000 (64KB)
│       boot_params structure            │ ~4KB
├────────────────────────────────────────┤ 0x20000 (128KB)
│       Kernel command line              │ "console=ttyS0..."
├────────────────────────────────────────┤ 0x100000 (1MB)
│                                        │
│       Linux Kernel Image               │
│       (loaded from ELF segments)       │
│       - .text (code)                   │
│       - .rodata (constants)            │
│       - .data (initialized globals)    │
│       - .bss (zeroed globals)          │
│                                        │
├────────────────────────────────────────┤
│                                        │
│       Free RAM                         │
│       (kernel allocates from here)     │
│                                        │
└────────────────────────────────────────┘ ?GB
```
Tiếp theo chúng ta cần cập nhật ví trị thực thi code cho KVM là có thể khởi động máy ảo được, tuy vậy còn một bước khác ta cần làm trước, đó là cập nhật IO handler, nếu không chúng ta không thể biết được code có chạy OK hay không 

### Xử lý Serial console I/O
Khi kernel khởi động, nó sẽ in ra các thông báo. Nếu không có cơ chế giả lập console, chúng ta sẽ không bao giờ thấy được các thông báo này — vì máy ảo (guest) sẽ ghi dữ liệu ra “phần cứng” vốn không hề tồn tại.
Tham số dòng lệnh kernel console=ttyS0 yêu cầu Linux xuất log ra cổng serial đầu tiên (COM1). Cổng serial rất đơn giản: chỉ cần ghi một byte vào cổng I/O 0x3F8 là dữ liệu sẽ được gửi ra ngoài. Chúng ta chặn (intercept) các thao tác ghi này và in nội dung đó ra terminal của mình.

Cách KVM I/O hoạt động
- Khi guest thực thi một lệnh out (ghi dữ liệu ra cổng I/O):
- CPU trap (chuyển quyền điều khiển) sang KVM
- KVM lưu thông tin về thao tác I/O vào cấu trúc kvm_run
- KVM trả quyền điều khiển về cho VMM của chúng ta với exit_reason là KVM_EXIT_IO
- VMM xử lý I/O này và gọi KVM_RUN lần nữa để tiếp tục chạy guest
Ta tách phần xử lý KVM_EXIT_IO ra một hàm riêng để dễ quản lý:
{% highlight rust %} 
fn handle_io(&self, kvm_run: &kvm_run) -> Result<()> {
    // Get I/O details from the kvm_run structure
    let io = unsafe { kvm_run.__bindgen_anon_1.io };
    let port = io.port;                                    // Which port?
    let is_out = io.direction as u32 == KVM_EXIT_IO_OUT;  // Read or write?
    
    // Data is at an offset within the kvm_run mmap'd region
    let data_ptr = unsafe { 
        (self.run_ptr.as_ptr() as *mut u8).add(io.data_offset as usize) 
    };

    match (port, is_out) {
        // Guest writing to COM1 data register - this is a character!
        (COM1_DATA, true) => {
            let ch = unsafe { *data_ptr } as char;
            print!("{}", ch);
            _ = std::io::stdout().flush();  // Show immediately
        }
        
        // Guest reading Line Status Register
        // Kernel checks "is the transmitter ready?" before sending
        (COM1_LSR, false) => {
            // Return: Transmitter empty (0x20) | Holding register empty (0x40)
            // This tells the kernel "yes, you can send"
            unsafe { *data_ptr = 0x60; }
        }
        
        // Other UART registers - ignore writes, return 0 for reads
        (COM1_IER..=COM1_MSR, _) => {
            if !is_out {
                unsafe { *data_ptr = 0; }
            }
        }
        _ => {}
    }
    Ok(())
}
{% endhighlight %}
Ở đây cần lưu ý rằng, do 8250/16550 serial driver sẽ đợi ở `COM1_LSR` cho đến khi có tín hiệu thông báo với nó là nó có thể truyền dữ liệu vào iout, ta cần thêm đoạn xử lý để ghí dữ liệu vào  `COM1_LSR`.
### Thiết lập các interrupt cần thiết
Trước khi cho CPU guest bắt đầu chạy (KVM_RUN), VMM bắt buộc phải thiết lập hệ thống ngắt (interrupt subsystem). Trên x86, Linux không thể boot đúng nếu thiếu các thành phần này, ngay cả khi chưa có interrupt thực sự được phát ra. Trước khi chạy guest, VMM phải thiết lập đầy đủ hạ tầng ngắt: TSS để xử lý ring transition, identity map để đảm bảo interrupt an toàn khi paging chưa ổn định, irqchip để cung cấp APIC, và PIT để hỗ trợ timing trong early boot. Thiếu bất kỳ thành phần nào cũng có thể khiến Linux treo ngay từ những dòng printk đầu tiên.

Đoạn code sau thực hiện toàn bộ phần thiết lập đó:
{% highlight rust %} 
pub fn setup_irqchip(&self) -> Result<()> {
    // TSS ADDR
    let ret = unsafe { ioctl(self.fd.as_raw_fd(), KVM_SET_TSS_ADDR, ADDR_TSS, 0) };
    if ret < 0 {
        return Err(io::Error::last_os_error()).context("KVM_SET_TSS_ADDR failed");
    }

    // IDENTITY mapping
    let ret = unsafe {
        ioctl(
            self.fd.as_raw_fd(),
            KVM_SET_IDENTITY_MAP_ADDR,
            &ADDR_IDENTITY_MAP,
            0,
        )
    };
    if ret < 0 {
        return Err(io::Error::last_os_error()).context("KVM_SET_IDENTITY_MAP_ADDR failed");
    }

    // Create in-kernel APIC
    let ret = unsafe { ioctl(self.fd.as_raw_fd(), KVM_CREATE_IRQCHIP, 0) };
    if ret < 0 {
        return Err(io::Error::last_os_error()).context("KVM_CREATE_IRQCHIP failed");
    }

    // Create PIT (timer)
    let pit_config = kvm_pit_config::default();
    let ret = unsafe { ioctl(self.fd.as_raw_fd(), KVM_CREATE_PIT2, &pit_config) };
    if ret < 0 {
        return Err(io::Error::last_os_error()).context("KVM_CREATE_PIT2 failed");
    }
    Ok(())
}
{% endhighlight %}

#### KVM_SET_TSS_ADDR – Vì sao cần TSS dù đang ở long mode? 
TSS là gì?

TSS (Task State Segment) là cấu trúc x86 dùng để:
- Chuyển stack khi vào interrupt / exception
- Lưu RSP0 (kernel stack) cho ring transition
- Hỗ trợ cơ chế interrupt an toàn. CPU thật yêu cầu TSS hợp lệ khi: Có interrupt hoặc có exception, nếu TSS không tồn tại thì sẽ xảy ra triple fault.
Dù Linux 64-bit không dùng task switching cổ điển, TSS vẫn bắt buộc tồn tại. KVM không tự tạo TSS cho bạn. VMM phải nói rõ: "TSS của guest nằm ở đâu trong guest physical memory."
Chúng ta cần đưa địa chỉ `ADDR_TSS` cho KVM, địa chỉ này là địa chỉ vật lý của VM, và bắt buộc phải là vùng ram trống, không overlap với kernel, thông thường sẽ đặt ở vị trí thấp (dưới 1 MB) hoặc 1 page riêng, ở đây ta chọn `0xffff_d000`. Và dùng ioctl để thông báo cho KVM về sự lựa chọn này:
```ioctl(fd, KVM_SET_TSS_ADDR, ADDR_TSS)```

#### KVM_SET_IDENTITY_MAP_ADDR – Cái này để làm gì?
Identity map là gì?
Đây là một page table đặc biệt, dùng khi:
- CPU vào interrupt
- Nhưng paging / CR3 chưa hoàn chỉnh
- Hoặc trong các trường hợp chuyển mode

#### KVM_CREATE_IRQCHIP – Tạo APIC trong kernel
Linux luôn giả định có APIC, ngay cả trong VM.
Nếu không tạo IRQ chip thì sao?
- Không có Local APIC
- Không có timer interrupt
- Không có IPI
- SMP không hoạt động
- Scheduler không chạy
- Kernel có thể treo hoặc panic rất sớm
Thật ra thì IRQCHIP có thể được emulate ở userspace, tuy nhiên nó chậm và quá phức tạp so với nội dung bài viết, nên ta dùng của kernel:
```ioctl(fd, KVM_CREATE_IRQCHIP)```

#### KVM_CREATE_PIT2 –
Programmable Interval Timer (8254) – bộ đếm thời gian cổ điển, được linux dùng trong earlyboot. Linux dùng PIT để làm gì?
- Delay calibration
- Busy-wait loops
- Đo tần số TSC
Fallback timer trước khi APIC timer hoạt động.
Lát nữa chúng ta sẽ thấy lỗi liên quan đến cái này.

### Thiết lập registers và bắt đầu chạy máy ảo
Trước khi chuyển quyền điều khiển vào máy ảo (enter the VM), chúng ta cần thiết lập trạng thái ban đầu của CPU:
{% highlight rust %} 
fn init_registers(&self, rip: __u64, rsi: __u64) -> Result<()> {
    let mut regs = kvm_regs::default();
    unsafe { ioctl(self.fd.as_raw_fd(), KVM_GET_REGS, &mut regs, 0) };

    regs.rflags = 2;   // Bit 1 luôn phải được set (bit dự trữ)
    regs.rip = rip;    // Instruction Pointer = điểm vào (entry point) của kernel
    regs.rsi = rsi;    // RSI = con trỏ tới cấu trúc boot_params (Linux boot protocol)

    unsafe { ioctl(self.fd.as_raw_fd(), KVM_SET_REGS, &mut regs, 0) };
    Ok(())
}
{% endhighlight %}
Theo Linux 64-bit boot protocol, các thanh ghi khi kernel bắt đầu thực thi được quy ước như sau:
- `RIP`: địa chỉ entry point của kernel
- `RSI`: con trỏ tới cấu trúc boot_params
Các thanh ghi khác: không được định nghĩa (kernel không phụ thuộc vào giá trị của chúng)

Nói cách khác, khi kernel bắt đầu chạy, nó không tự dò tìm môi trường boot mà dựa trực tiếp vào con trỏ `boot_params` được truyền qua thanh ghi RSI. Cấu trúc này chứa toàn bộ thông tin quan trọng như:
- E820 memory map
- Tham số dòng lệnh kernel
- Thông tin về initrd
- Các feature mà bootloader / VMM hỗ trợ
- Việc thiết lập đúng `RIP` và `RSI` là điều kiện bắt buộc để kernel Linux có thể khởi động thành công trong môi trường KVM.


Tiếp theo cập nhật đoạn code xử lý IO trong hàm `run()` của CPU thành:
{% highlight rust %} 
fn run(&self) -> Result<()> {
    loop {
        // Enter guest mode - CPU executes guest code until something happens
        let ret = unsafe { ioctl(self.fd.as_raw_fd(), KVM_RUN, 0) };
        if ret < 0 {
            return Err(io::Error::last_os_error())
                .context("KVM_RUN failed");
        }
        
        // Examine why we exited
        let kvm_run = unsafe { self.run_ptr.as_ref() };

        match kvm_run.exit_reason {
            KVM_EXIT_IO => {
                // Guest did port I/O - handle it
                self.handle_io(kvm_run)?;
            }
            KVM_EXIT_HLT => {
                // Guest executed HLT instruction
                // This means the kernel is done (or idle)
                println!("Guest executed HLT. Stopping.");
                break Ok(());
            }
            other => {
                // Unknown exit - for debugging, we could dump state here
            }
        };
    }
}
{% endhighlight %}

Thiết lập lại hàm `main()` để input entry point đúng vào cho VM
{% highlight rust %} 
fn main() -> Result<()> {
    //... Declarations 
    let mut vm = VM::new().expect("unable to open VM fd");
    // 2GB ram
    vm.insert_memory_region(0, 1u32, 2 * 1024 * 1024 * 1024, 0)?;
    vm.setup_irqchip()?;
    vm.create_vcpu()?;
    vm.enter_long_mode()?;
    let entry = vm.load_code(&code)?;
    println!("Code will start at {:#x}", entry);
    vm.run(entry)?;
}
{% endhighlight %}

### Testtt
Thực hiện câu lệnh sau để bắt đầu chạy VM:
```
cargo run -- [path to vmlinux.bin]
```
Ngay khi thực hiện cargo run, linux sẽ được boot lên và in một số thông tin ra màn hình
```
warning: `rust-kvm-tool` (bin "rust-kvm-tool") generated 9 warnings (run `cargo fix --bin "rust-kvm-tool" -p rust-kvm-tool` to apply 2 suggestions)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.30s
     Running `target/debug/rust-kvm-tool ../mini-vmm/vmlinux.bin`
Succeffully loading vmlinux to memory (size: 21441304)
Warning: munmap failed: Invalid argument (os error 22)
Code will start at 0x1000000

[    0.000000] Linux version 4.14.174 (@57edebb99db7) (gcc version 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)) #2 SMP Wed Jul 14 11:47:24 UTC 2021
[    0.000000] Command line: console=ttyS0 earlyprintk=ttyS0
[    0.000000] Disabled fast string operations
[    0.000000] x86/fpu: Supporting XSAVE feature 0x001: 'x87 floating point registers'
[    0.000000] x86/fpu: Supporting XSAVE feature 0x002: 'SSE registers'
[    0.000000] x86/fpu: Supporting XSAVE feature 0x004: 'AVX registers'
[    0.000000] x86/fpu: xstate_offset[2]:  576, xstate_sizes[2]:  256
[    0.000000] x86/fpu: Enabled xstate features 0x7, context size is 832 bytes, using 'compacted' format.
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000007fffffff] usable
[    0.000000] bootconsole [earlyser0] enabled
[    0.000000] NX (Execute Disable) protection: active
[    0.000000] DMI not present or invalid.
[    0.000000] Hypervisor detected: KVM
```
Tuy nhiên, có điều gì đó không ổn, vì nó stuck luôn ở đây, trong khi mình expect rằng nó sẽ bị panic ở bược load rootfs (theo kinh nghiệm đọc khá nhiều bài về kvm toys)

Sau một lúc tìm hiểu, thì điều này là do kernel đang đợi dữ liệu ở port `0x61`. Cổng 0x61 là cổng điều khiển bàn phím / PIT đời cũ.
Linux poll cổng này từ rất sớm để hiệu chuẩn độ trễ và kiểm tra tính hợp lý của thời gian. Nếu VMM không trả về bits như mong đợi, kernel sẽ rơi vào một vòng lặp vô tận (spin forever).

Theo như đoạn code ở trong file `arch/x86/kernel/tsc.c` thì chúng ta cần trả về giá trị `0x20` nếu như có yêu cầu input từ port này:
{% highlight c %} 
    // linux kernel: arch/x86/kernel/tsc.c
	while ((inb(0x61) & 0x20) == 0) {
        //...
    }
{% endhighlight %}

Ta thêm 1 match arm vào handle_io để ghi 0x20 vào 0x61
{% highlight rust %} 
        (0x61, false) => unsafe {
                *data_ptr = 0x20;
            },
{% endhighlight %}

Thử chạy lại VMM:
[![asciicast](https://asciinema.org/a/7NbtVpBDVH8Jyn0k.svg)](https://asciinema.org/a/7NbtVpBDVH8Jyn0k)

Ok, lần này đúng như mong đợi, nó đã panic ở bước load rootfs vì ta không cung cấp cho nó file rootfs nào cả. Nếu có thời gian sẽ cập nhật, không thì thôi.

Như vậy, ta đã có một VMM cơ bản có thể boot được linux kernel viết bằng Rust.

# References
- [KVM API Documentation](https://www.kernel.org/doc/html/latest/virt/kvm/api.html)
- [Linux x86 Boot Protocol](https://www.kernel.org/doc/html/latest/arch/x86/boot.html)
- [Intel Software Developer's Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [OSDev Wiki - Setting Up Long Mode](https://wiki.osdev.org/Setting_Up_Long_Mode)
- [rust-vmm Project](https://github.com/rust-vmm) - Production-quality Rust VMM components
- [yeet.cx are you bios now](https://yeet.cx/blog/you-are-the-bios-now)
- [KVM host in a few lines of code](https://zserge.com/posts/kvm/)


*Built with Rust, curiosity, and lots of debugging.*