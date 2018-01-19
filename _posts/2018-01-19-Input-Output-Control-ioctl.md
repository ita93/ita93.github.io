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
## 1. Cách chọn giá trị cho cmd arg trong ioctl
-Không nên sử dụng cách chọn một set các số bắt đầu từ 0(hoặc 1) để dụng cho cmd arg. Lý do là các <code>ioctl</code> của các driver khác nhau nên sử dụng các cmd arg khác nhau, hay nói cách khác cmd arg là unique value trong toàn hệ thống.<br/>
-Tại sao lại dùng unique value cho toàn bộ cmd arg? Vì khi đấy nếu user-space prog truyền cmd của driver A và ioctl của driver B thì prog sẽ nhận được giá trị trả về là EINVAL thay vì một thao tác không đoán trước được và không đúng mong muốn của người dùng.<br/>


