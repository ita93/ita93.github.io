---
layout: post

category: Linux device driver

comments: true
---
# Access Control với device file (quyền truy cập).

Việc cung cấp AC đôi khi là quan trọng đối với tính tin cậy của một device node. Không chỉ việc cho phép người không có thẩm quyền được phép sử dụng device, mà còn để hạn chế việc sử dụng device cho một người có thẩm quyền duy nhất (bỏ qua những người có thẩm quyền khác) tại một thời điểm nào đó. 
Xét một ví dụ, đấy là tty. Trong trường hợp này, tiến trình <i>login</i> tiến hành thay đổi quyền sở hữu của device node mỗi khi có một người dùng đăng nhập vào hệ thống, mục đích là để loại bỏ khả năng người dùng khác can thiệt hoặc giám sát luồng dữ liệu của tty. Tuy nhiên, việc privileged program để thay đổi ownership của device mỗi lần nó được mở là không thực tế.

## I. Single-Open Devices.
Phương thức này có nghĩa là chỉ cho phép một process duy nhất được mở device file tại một thời điểm nhất định. Kỹ thuật này tốt nhất là nên tránh bởi vì nó hạn chế tính linh hoạt của người dùng. Một người dùng có thể muốn chạy các processes khác nhau trên cùng một device, một process đọc thông tin trong khi cái khác sẽ ghi dữ liệu. Trong một số hoàn cảnh, người dùng có thể làm rất nhiều thứ bằng cách chạy một shell script, miễn sao họ có thể truy cập vào device một cách đồng thời (tức là mở nhiều process của cùng 1 user mở cùng 1 device file). Tuy nhiên đây là cách đơn giản nhất để thực hiện AC. 
Phương thức này đã được mình dùng trong các ví dụ về Concurrency access.

## II. Single-User devices.
Một level cao hơn việc single-open là sử dụng single-user.
Giải pháp này giúp việc test device dễ dàng hơn, vì người dùng có thể đọc và ghi từ nhiều process tại cùng một thời điểm (tất nhiên việc bảo đảm interity là của user). Việc này được thực hiện bằng cách thêm checking ở open(); những checking này được thực hiện sau các permission checking thông thường. 