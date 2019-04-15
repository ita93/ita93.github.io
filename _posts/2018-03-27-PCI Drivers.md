---
layout: post

category: Linux device driver

comments: true
---

## 1 PCI Interface.
Cái PCI này có thể nghe rất quen, rất quen thuộc với nhiều người, nếu ai từng cài win(hộ) và lúc cài driver(Windows8-10 thì chắc ít gặp vì nó tự cài từ hồi nào rồi) thì chắc chắn sẽ phải cài một số driver có tên là PCI gì gì đấy, hoặc giả bạn không cài PCI driver cho windows thì sound, mạng mẽo các thứ sẽ không dùng được kể cả bạn có cài driver mạng hay âm thanh rồi đi chăng nữa. Nếu bạn từng tìm cách sửa lỗi bằng cách scan new hardware trong Device Manager của Windows thì nếu để ý bạn cũng có thể thấy một số device PCI gì gì đấy ở đây. Vậy PCI là cái gì?
PCI là bất kỳ thành phần phần cứng nào của máy tính được kết nối trực tiếp vào máy tính bằng cách cắm thẳng vào công PCI có trên bo mạch chủ (motherboard - bỏ mẹ). PCI, viết tắt của Peripheral Component Interconnect. Đây là một tập Specification được Intel giới thiệu vào năm 1993( ĐÚng năm mình được sinh ra, thật là ý trời), tuy nhiên phải đến năm 1995 thì công nghệ này mới được các công ty máy tính tích hợp lên bo mạch chủ của các máy tính cá nhân. Về mặt vật lý, PCI slot trên motherboard có hình dạng như sau:<br/>
<img src="https://cmeimg-a.akamaihd.net/640/photos.demandstudios.com/getty/article/228/192/493830023.jpg">
<br/>
Về cơ bản, mọi máy tính cá nhân có hai bus chính được liên kết với Central Processing Unit(CPU). System bus, nhanh nhất, kết nối đến RAM, CPU. Cái thứ 2 chính là PCI bus, không nhanh bằng System bus, có mục đích chính là cung cấp khả năng giao tiếp với các thiết bị phần cứng của nhưng loại như Audio, Video, Network ... (những cái này có microprocessor riêng). 
<br/>
Ví dụ, Network Card là một ví dụ về PCI device, nó có chân để cắm vào pci slot trên motherboard( PCI bus). Trên các laptop thì nó sử dụng miniPCI, cũng là PCI nhưng có ít pin hơn, do kích thước nhỏ hơn.
Cái chúng ta quan tâm ở PCI driver là cách nó tìm ra phần cứng của nó và giành quyền truy cập. Cũng giống như đối với interrupt line number, PCI driver cần một phương pháp để tìm ra phần cứng đó.
PCI là platform independence, một network card hoạt động được với X86PC thì nó cũng có thể hoạt động với ARM board (nếu có khe cắm đúng kích thước). PCI devices tự động config ở boot time (PCI là coldplug), do đó, device driver phải truy cập dược thông tin trong device để có thể hoàn thành quá trình khởi tạo.

## 1.1 PCI Addressing.
Mỗi PCI peripheral được định danh bởi một <i>bus number</i>, một <i>device number</i> và một <i>function number</i>. Trong PCI specification thì một system có 256 buses, nhưng đối với một số hệ thống lớn thì 256 là con số không đủ, do đó Linux có tính năng PCI domains. Mỗi PCI domain có thể host cho 256 buses. Mỗi bus lại có thể host cho 32 devices, và mỗi device lại có thể là 1 cái board đa năng(tối đa là 8 tính năng). Bởi thế tính ra có thể có 256*32*8=65536 functions. Cái này là lý thuyết thôi, chứ thực tế chắc không thấy. Nói ra để thấy sự nguy hiểm của nó thôi, chứ thực tế We don't need to care about it.

Các PCI peripheral có 16-bit hardware address liên kết. Mặc dù hầu như đối với driver writer thì những thông tin này được ẩn đằng sau <code>struct pci_dev</code>, tuy nhiên, đôi khi nó chúng ta vẫn có thể thấy nó trong output của <code>lspci</code> và thông qua các file thông tin lưu trong filesystem <i>/proc/pci</i> và <i>proc/bus/pci</i>. 
Ví dụ, sau đây là output của lspci:
{% highlight shell %}
0:00.0 Host bridge: Intel Corporation 4th Gen Core Processor DRAM Controller (rev 06)
00:02.0 VGA compatible controller: Intel Corporation Xeon E3-1200 v3/4th Gen Core Processor Integrated Graphics Controller (rev 06)
00:03.0 Audio device: Intel Corporation Xeon E3-1200 v3/4th Gen Core Processor HD Audio Controller (rev 06)
00:14.0 USB controller: Intel Corporation 8 Series/C220 Series Chipset Family USB xHCI (rev 05)
00:16.0 Communication controller: Intel Corporation 8 Series/C220 Series Chipset Family MEI Controller #1 (rev 04)
00:1a.0 USB controller: Intel Corporation 8 Series/C220 Series Chipset Family USB EHCI #2 (rev 05)
00:1b.0 Audio device: Intel Corporation 8 Series/C220 Series Chipset High Definition Audio Controller (rev 05)
00:1c.0 PCI bridge: Intel Corporation 8 Series/C220 Series Chipset Family PCI Express Root Port #1 (rev d5)
00:1c.3 PCI bridge: Intel Corporation 8 Series/C220 Series Chipset Family PCI Express Root Port #4 (rev d5)
00:1c.4 PCI bridge: Intel Corporation 8 Series/C220 Series Chipset Family PCI Express Root Port #5 (rev d5)
00:1d.0 USB controller: Intel Corporation 8 Series/C220 Series Chipset Family USB EHCI #1 (rev 05)
00:1f.0 ISA bridge: Intel Corporation C220 Series Chipset Family H81 Express LPC Controller (rev 05)
00:1f.2 SATA controller: Intel Corporation 8 Series/C220 Series Chipset Family 6-port SATA Controller 1 [AHCI mode] (rev 05)
00:1f.3 SMBus: Intel Corporation 8 Series/C220 Series Chipset Family SMBus Controller (rev 05)
02:00.0 Ethernet controller: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller (rev 0c)
03:00.0 Network controller: Qualcomm Atheros QCA9565 / AR9565 Wireless Network Adapter (rev 01)
{% endhighlight %}
Cột đầu tiên chính là 16-bit hardware address của PCI peripheral, theo cầu trúc <i>domain:host:device.function</i>, ví dụ đối với VGA compatible controller ở trên thì host là 0x00, device là 0x02 và function là 0x0.

Mạch điện phần cứng của mỗi peripheral board trả lời các câu hỏi truy vấn liên quan đến các không gian địa chỉ: Mem loc, I/O ports và Configuration register khi được yêu cầu. Hai địa chỉ đầu tiên được chia sẻ bởi tất cả các device trong cùng một PCI bus, tức là khi bạn truy cập một địa chỉ bộ nhớ, tất cả các device trong PCI bus đó sẽ thấy được bus cycle ở cùng một thời điểm. Trong khi đó, Configuration space, khai thác <i>geographical addressing</i>. Configuration chỉ truy vấn địa chỉ của một slot tại một thời điểm, nên không bao giờ chúng va chạm lẫn nhau. Memory và I/O address space được truy cập theo cách thông thường thông qua <i>inb</i>, <i>readb</i>... Configuration transactions(not access) được thực hiện bằng cách gọi một số kernel function cụ thể để truy cập vào các configuration register. Mỗi PCI slot có 4 interrupt pins, và mỗi device function có thể sử dụng một trong số chúng mà không cần quan tâm đến những pin này sẽ được đính tuyền đến CPU như thế nào.

I/O space trong một CPU bus sử dụng 32-bit address bus(upto 4GB I/O ports), trong khi Memory space có thể được truy cập thông qua địa chỉ 32-bit hoặc 64-bit. Về mặt lý thuyết, các địa chỉ này là đơn nhất trên một thiết bị, tuy nhiên, software có thể tiềm ẩn lỗi cấu hình 2 thiết bị vào cùng một địa chỉ, khiến cho việc truy cập một trong số chúng là không thể. Nhưng đáng an tâm là vấn đề này sẽ không bao giờ xảy ra nếu driver không động chạm gì đến những thanh ghi mà nó không nên động vào. 

Tất cả các I/O và Memory address region được cung cấp bởi interface board có thể được remapping bởi configuration transactions. Tức là, firmware khởi tạo PCI hardware ở system boot, mapping mỗi region với một địa chỉ khác nhau để tránh xung đột. Địa chỉ mà những region nào đang được map vào có thể đọc được từ configuration space, do đó Linux driver có thể truy cập các device của nó mà không cần probing. Sau khi đọc các configuration register, driver có thể truy cập phần cứng của nó một cách an toàn.

PCI configuration space chứa 256bytes cho mỗi device function. 4bytes của configuration space giữ một unique funciton ID, do đó driver có thể định danh được device của nó bằng cách tìm kiếm một ID cụ thể (của peripheral). Tóm lại, mỗi device board được đánh địa chỉ một cách địa lý () để truy vấn các configuration register của nó; thông tin trong các thanh ghi nào có thể được sử dụng để thực hiện các truy cập I/O đơn giản.
<b><i>
Later 
Device = Device function
Device = (Domain number, bus number, device number, function number).
</i></b>

(Pending - Need to finish Computer bus first).