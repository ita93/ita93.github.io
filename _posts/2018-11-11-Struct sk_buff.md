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
<code>sk_buff<code> chứa rất nhiều trường khác nhau, một số lương khổng lồ đủ để người ta biết được tất cả các thông tin cần thiết khi nhìn vào đấy. Các trường này có thể được chia vào 4 loại: Layout, General, Feature-generic, Management function.

## 1.  Layout Fields.
    Những trường này có tác dụng giúp việc tìm kiếm một <code>sk_buff</code> variable trở nên dễ dàng hơn và tổ chức cấu trúc dữ liệu một cách hiệu quả là chính.


