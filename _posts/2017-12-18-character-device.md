# Character Device Driver.

Character Device (chardev) là các device được truy cập như một luồng nhị phân (byte stream - tương tự như các file trong máy tính), và character device
driver có nhiệm vụ thực hiện những thao tác đọc ghi này. Đối với một character device driver, ít nhất 4 system call: open, close read và write cần được implement để chardev có thể hoạt động một cách bình thường. 

Một số ví dụ về character device là: Serial console và console.

Do Linux về cơ bản là file, file và file nên các character device cũng không phải là ngoại lệ, chúng được truy cập bằng các file system node (trong /dev directory). Tuy nhiên, không giống như các file thông thường khác (có thể truy cập tiến, lùi tùy ý), các file dev này chỉ được truy cập theo một cách tuần tự. 

Khi dùng command ls -l /dev, các device file được liệt kê, các file có chữ <span style="color:blue">c</span> ở đầu là các device file của chardev.

```python
crw-r--r--    1 root     root        5,   1 Dec 27 03:45 console
crw-rw-rw-    1 root     root        1,   7 Jan  1  1970 full
crw-r--r--    1 root     root       10, 229 Dec 27 03:06 fuse
crw-r--r--    1 root     root        1,  11 Jan  1  1970 kmsg
crw-r--r--    1 root     root        1,   1 Jan  1  1970 mem
brw-r--r--    1 root     root       31,   0 Jan  1  1970 mtdblock0
brw-r--r--    1 root     root       31,   1 Jan  1  1970 mtdblock1
brw-r--r--    1 root     root       31,   2 Jan  1  1970 mtdblock2
brw-r--r--    1 root     root       31,   3 Jan  1  1970 mtdblock3
brw-r--r--    1 root     root       31,   4 Jan  1  1970 mtdblock4
brw-r--r--    1 root     root       31,   5 Jan  1  1970 mtdblock5
brw-r--r--    1 root     root       31,   6 Jan  1  1970 mtdblock6
brw-r--r--    1 root     root       31,   7 Jan  1  1970 mtdblock7
crw-rw-rw-    1 root     root        1,   3 Jan  1  1970 null
crw-r--r--    1 root     root        1,   4 Jan  1  1970 port
crw-------    1 root     root      108,   0 Dec 27 03:06 ppp
crw-rw-rw-    1 root     root        5,   2 Dec 27 04:32 ptmx
drwxr-xr-x    2 root     root             0 Jan  1  1970 pts
crw-r--r--    1 root     root        1,   8 Jan  1  1970 random
drwxr-xr-x    2 root     root            40 Jan  1  1970 shm
crw-r--r--    1 root     root       10, 254 Dec 27 03:06 switch_ssdk
crw-rw-rw-    1 root     root        5,   0 Jan  1  1970 tty
crw-------    1 root     root        4,  64 Dec 27 03:45 ttyS0
crw-r--r--    1 root     root        1,   9 Jan  1  1970 urandom
crw-r--r--    1 root     root       10, 130 Dec 27 03:06 watchdog
crw-rw-rw-    1 root     root        1,   5 Jan  1  1970 zero
```

###1. Major number và Minor number - Định danh của device
Như có thể thấy ở trên, hầu hết các device trong Linux là chardev. Ngoài ra, có thể để ý thấy, mỗi device file thường có 2 số được phân tách bởi dấu phẩy đi kèm nhau, ví dụ: với device ttyS0 hai số này là 4 và 64. Hai số này được gọi là Major và Minor, chúng được dùng để định danh device trong một rừng cái device của hệ thống. 
Major number (số trước dấu phẩy): Dùng để xác định driver nào sẽ liên kết với thiết bị, ví dụ ttyS0 sẽ được điều khiển bời driver 4, trong khi các mtd device được điều khiển bởi driver 31.
Minor number (số sau dấu phẩy): Cũng ở ví dụ trên, ta có thể thấy 8 mtd device có chung một major number, nhưng minor number khác nhau, tức là Minor number được sử dụng để phân biệt các device sử dụng chung một driver.

Trong kernel, có sử dụng một kiểu biến là  dev_t (linux/types.h), kiểu này đượuc dùng để lưu dữ các device numbers(major và minor), nói cách khác, các device trong kernel được phân biệt bằng các biến có kiểu này.
dev_t có kích thước là 32-bit, với 12 bit dùng cho Major number và 20 bit còn lại cho minor number, lưu ý là thứ tự của 2 phần này không được chỉ rõ, nên tốt nhất chúng ta không nên tự xác định major và minor bằng tay từ device number, thay vào đó, kernel source cung cấp cho chúng ta 2 macro để làm việc này là: MAJOR(dev_t dev) và MINOR(dev_t dev). Cả hai Macro này đều được định nghĩa trong <linux/kdev_t.h>.
Tương tự, nếu như chúng ta đã biết major và minor của một device, chúng ta có thể tìm ra device number của nó bằng Macro: MKDEV(int major, int minor)