---
layout: post

category: Computer Architecture.

comments: true
---
Theo như Wikipedia:
Một tập lệnh, hoặc kiến trúc tập lệnh (tiếng Anh: Instruction Set Architecture, viết tắt ISA), là một phần của kiến trúc máy tính liên quan đến lập trình, bao gồm các bản địa các loại dữ liệu, hướng dẫn, đăng ký, giải quyết chế độ, kiến trúc bộ nhớ, làm gián đoạn và xử lý ngoại lệ, và bên ngoài I / O. An ISA bao gồm một đặc điểm kỹ thuật của các thiết lập của opcode (ngôn ngữ máy), và các lệnh bản địa thực hiện bởi một bộ xử lý cụ thể.

Tóm lại thì Instruction Set là mọt tập các câu lệnh (instruction) để ra lệnh cho máy tính. Các kiến trúc khác nhau sẽ có các Instruciton set khác nhau. Các máy tính đời đầu thường có Instruction set rất đơn giản, và thật bất ngờ, các máy tính hiện đại thời nay cũng thế, chả có thay đổi gì mấy :v. Vì "Đơn giản là Hiệu quả".

Ví dụ như, trong ARMv8, phép cộng <code>a=b+c</code> được thực hiện như sau:
{% highlight c %}
ADD a, b, c
{% endhighlight %}

Phép trừ:
{% highlight c %}
SUB a, b, c
{% endhighlight %}

Các phép cộng số học cơ bản như trừ, nhân, chia được đều có chung một cấu trúc như này. Như vậy, các phép tính đơn giản đều có tính đồng bộ (cấu tạo giống nhau), đều này giúp cho việc implement đơn giản hơn và hiệu năng cao hơn.
