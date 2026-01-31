---
layout: post

category: Linux device driver

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
Về cách sử dụng của KVM API, trên LWN đã có một bài hướng dẫn khá đầy đủ https://lwn.net/Articles/658511/ . Cơ bản thì nó có mấy bước sau:
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
                    .context("failed to get supported cpuid features");
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