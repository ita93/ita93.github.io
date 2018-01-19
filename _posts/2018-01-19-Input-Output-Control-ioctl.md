# Input/Output Control

Hầu hết các driver cần có khả năng đọc và ghi device - tức là thực hiện các hardware control khác nhau thông quan device driver. Đa phần các device có thể thực hiện các thao tác phức tạp hơn việc chỉ truyền dữ liệu, chẳng hạn như report lỗi, thay đổi baudrate, tự hủy, etc... Những thao tác này thông thường được support bằng command <code>ioctl</code>.

Ở user space, linux cung cấp ioctl system call có prototype như sau:<br/>
<code>int ioctl(int fd, unsigned long cmd, ...);</code></br>
Trong C, <code>...</code> là va_arg (một lượng tham số truyền vào không biết trước), tuy nhiên, một system call không thể nào có va_arg được(Tại sao?). Bởi thế, <code>...</code> ở đây là một argument tùy chọn (có thể có hoặc không). argument này có thể là bất kỳ type nào.<br/>

-Prototype của ioctl trong ldd:<br/>
<code>int (*ioctl)(struct inode *inode, struct file *filp, unsigned int cmd, unsigned long arg);</code><br/>
<i>[kernel]inode+filp = [user-space]fd</i></br>
(need a image here)<br/>
-Nếu ioctl có additional arg thì trong kernel-space, nó luôn được truyền như là một biến unsigned long, bất kể biến truyền vào ở user-space là integer hay pointer. Nếu user-space program không pass optional arg thì biến <code>arg</code> trong ldd sẽ là <b>undifined</b><br/>
<i>type checking for ioctl is disabled</i><br/>
- Đối với từng <code>cmd</code> thì có một tác vụ tương ứng được thực hiện<br/>
- Thông thường ioctl sẽ sử dụng <code>switch(cmd)</code> để thực hiện nhiệm vụ của nó.<br/>
<br/><br/>
## 1. Command number (cmd arg) và cách chọn cmd arg.
-Không nên sử dụng cách chọn một set các số bắt đầu từ 0(hoặc 1) để dụng cho cmd arg. Lý do là các <code>ioctl</code> của các driver khác nhau nên sử dụng các cmd arg khác nhau, hay nói cách khác cmd arg là unique value trong toàn hệ thống.<br/>
-Tại sao lại dùng unique value cho toàn bộ cmd arg? Vì khi đấy nếu user-space prog truyền cmd của driver A và ioctl của driver B thì prog sẽ nhận được giá trị trả về là EINVAL thay vì một thao tác không đoán trước được và không đúng mong muốn của người dùng.<br/><br/><br/>
Vậy chọn giá trị nào cho <code>cmd</code>? <br/>
Đầu tiên cần kiểm tra <code>include/asm/ioctl.h</code> và <i>Documentation/ioctl-number.txt</i> để xem những giá trị đã được xí trước để tránh dùng những giá trị này.<br/>

Command numbers (<code>cmd</code>) được định nghĩa bằng cách sử dụng 4 bitfields: type, number, direction, type. Mỗi bitfield có ý nghĩa như sau:<br/>
<code>type</code>: Magic number, ứng với một device driver có một magic number duy nhất. Bitfield này có độ rộng là 8 bits(_IOC_TYPEBITS).<br/>
<code>number</code>: Đây là số thứ tự. Có kích thước 8 bít (_IOC_NRBITS).<br/>
<code>direction</code>: Chiều transfer của dữ liệu (đọc/ghi/nothing): _IOC_NON(không truyền data), _IOC_WRITE, _IOC_READ, nếu vừa đọc vừa ghi data thì có thể dùng _IOC_WRITE|_IOC_READ. view point ở đây là từ user-space application. tức là ĐỌC từ device, GHI vào device.<br/>
<code>size</code> Kích thước của data. Bitfield này có độ rộng phụ thuộc và arch, nhưng thường là 13/14 bits. Kích thước cho từng arch cụ thể có thể được tìm thấy bằng macro _IOC_SIZEBITS. Thật ra field này là không bắt buộc, nhưng việc sử dụng nó giúp cho driver có thể phát hiện errors. Vì chỉ có 13/14bits thì nó sẽ không đủ để ghi kích thước data khi data lớn.<br/><br/><br/>

## 2. Setup command number với asm/ioctl.h
header này cung cấp các macro giúp chúng ta dễ dàng hơn trong việc setup các command number:
-<code>_IO(type, nr)</code> : Command không nhận argument<br/>
-<code>_IOR(type, nr, datatype)</code>: Command đọc dữ liệu từ device<br/>
-<code>_IOW(type, nr, datatype)</code>: Command ghi dữ liệu vào device<br/>
-<code>_IOWR(type, nr, datatype)</code>: Dữ liệu được truyền ở cả hai chiều.<br/>
Đối với các command này: type và number được truyền như các args, ở đây chúng ta không thấy field size, thật ra, nó sẽ được macro tạo ra bằng cách <code>sizeof(datatype)</code><br/><br/>

Cũng trong header này, các macros dùng decode numbers được định nghĩa như sau:<code>_IOC_DIR(nr), _IOC_TYPE(nr), _IOC_NR(nr), _IOC_SIZE(nr)</code> tương ứng (direction, type, sequence number, data size).<br/>
Ví dụ về ioctl command number definations:<br/>
<code>#define ONI_IOCREAD _IOR(ONI_MAGIC_NR,FIRST_SEQ);</code><br/><br/>

#3. Giá trị trả về của ioctl
- Nếu command number truyền vào không đúng thì giá trị trả về nên là -EINVAL/-ENOTTY<br/>

#4. Sử dụng argument trong ioctl
-Argument ở đây có thể là integer number hoặc pointer<br/>
-Nếu argument truyền vào là một pointer thì cần đảm bảo rằng user space address là hợp lệ, nếu không nó có thể gây ra kernel ops... Driver cần kiểm tra tất cả các pointer được truyền vào <br/>








