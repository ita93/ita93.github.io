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

# I. ARMv8 assembly

{% highlight c %}
/File: maintest.c
int main()
{
    return 0;
}
{% endhighlight %}
Chương trình viết bằng C ở trên đây nó không làm gì ngoài việc return giá trị 0 cả. Khi ta sử dụng <code>gcc</code> đề build ra một file thực thi, nó thực hiện các bước riêng biệt. Giả sử đối với file <code>maintest.c</code> ở trên, để tạo ra file thực thi, gcc sẽ thực hiện các bước sau:
    <code>Tiền xử lý</code> (Preprocessing) Bước này khá ư là đơn giản, nó chỉ là nó sẽ tạo ra file code C cuối cùng dựa trên các directives( mấy cái có dấu # bên trước,vd: #include, #define)...
    <code>Biên dịch</code> (Compilation) Chuyển đổi source code C thành ngôn ngữ assembly. Nếu muốn xem file code assembly được tạo ra bởi gcc, có thể sử dụng tùy chọn -S của nó.
    <code>Assembly</code> gcc gọi đến chương trình tên <b>as</b> (assembler) để chuyển đổi từ mã assembly sang mã máy, việc này khác đơn giản vì nó chỉ là ánh xạ một-một mà thôi.
    <code>Liên kết</code> (Linking) Quá trình kết hợp mã máy đã tạo ra ở pha trước với mã máy của các thư viện chuẩn hoặc các modules khác, cũng như chỉ ra địa chỉ của các đoạn mã này trong file thực thi. VIệc này được thực hiện bởi chương trình <b>ld</d> và các file link script (nếu cần). Sau bước này, một file thực thi sẽ được tạo ra có tên mặc định là *.out, thường ta sẽ dùng tùy chọn <i>-o</i> của gcc để tạo ra file có tên theo ý muốn.

Bây giờ viết lại chương trình C trên bằng gcc-assembly, thì ta sẽ có file sau:
{% highlight c %}
@File name: testasm.S
@Define machine
    .cpu    cortex-a53
    .fpu    neon-fpu-armv8
    .syntax unified
@Define text section
    .text
    .align  2
@Define main function
    .global main
    .type   main, %function
main:
    str fp, [sp, -4]! @ Save fp to stack
    sub fp, sp, 0

    mov r0, 0
    
    add sp, fp, 0   @Restore sp and fp
    ldr fp, [sp], 4
{% endhighlight %}
Code trên được sử dụng bởi gcc-assembler, nếu sử dụng assembler khác thì tên của instruction có thể khác chút xíu, nhưng các nguyên tắc cơ bản hầu như là giống nhau.
Về cơ bản assembly là một ngôn ngữ line-oriented, tức là mỗi instruction chỉ được viết trên một dòng, và mỗi dòng chỉ có một instruction duy nhất, các dòng này được gọi là <b>Mnemonic</b>. Mỗi <b>mnemonic</b> sẽ được dịch 1-1 thành một dòng mã máy. Ví dụ <code>mov r0, 0</code> ở trên sẽ được chuyển thành mã máy là <b>0xe3a00000</b>. Các assembler khác nhau thì sẽ cố các <code>mnemonic</code> khác nhau, nhưng rất ít, và chúng đều thường tuân theo quy định trong manual của công ty cung cấp CPU.
Nếu một người đã quen với việc lập trình một ngôn ngữ bất kỳ nào đấy, thì khi nhìn vào đoạn code trên đều có thể dễ dàng nhận thấy ký hiệu <i>@</i> là bắt đầu của một đoạn comment, nó giống với // trong C.
Các dòng của assembly có cấu trúc như sau:
<code>label: operation operand(s) @comment</code>
Trong đó, label và comment là không bắt buộc, có thể có hoặc không.<br/>
- Trường <code>label</code> cho phép chúng ta đặt ký hiệu cho bất kỳ dòng nào trong chương trình. Vì mỗi dòng trong chương trình sẽ tương đương với một địa chỉ bộ nhớ nào đó, nên với <code>label</code> ta có thể nhảy về dòng lệnh mong muốn bất kể PC đang ở đâu. Rõ ràng là không phải dòng nào cũng cần label rồi.<br/>
Trường <code>operation</code>: Thực hiện mục đích chính của dòng lệnh. Nó có thể rơi vào 2 trường hợp, trường hợp 1: đây là một <code>mnemonic</code> nó sẽ dịch sang mã máy, sau đó mã máy được copy vào bộ nhớ để sử dụng. Trường hợp 2: Nó là một <code>assembler directive</code> hoặc <code>pseudo op</code> bắt đầu bằng dấu chấm <code>.</code> , những operation dạng này sẽ không được dịch thành mã máy, mà thường nó sẽ thay đổi cách <b>as</b> dịch chương trình.<br/>
- Trường <code>Operand</code> Đây là các toán hạng được sử dụng bởi các <code>operation</code>, chúng có thể là tên của một register, một tên biến được tạo ra bởi lập trình viên, hoặc là một giá trị chuỗi ký tự (số hoặc string). Một ARM instruction thường có từ 0 đến 3 toán hạng, đa số có cấu trúc như sau:
<code>operation [destination[, source1[, source2[,source3]]]</code>.
<br/><br/>
Quay lại chương trình <i>testasm.S</i> ở trên, ta có thể thấy nó bắt đầu bằng 2 dòng comment, tiếp theo là 3 <code>assembler directive</code>, 3 dòng này chỉ ra các đặc tính của cái cpu đang dùng, mấy cái trên đây là dành cho Pi 3 B+, các model khác thì tìm trong manual nha, ahihi.<br/>
Sau khi file nguồn được dịch thành ngôn ngữ máy, nó sẽ tạo ra một object file, có format cụ thể, gọi là ELF file. File ELF được chia ra làm các sections, directive <code>.text</code> ở trong file assembly trên được dùng để chỉ rằng, các dòng mã bên sau dòng này sẽ được lưu vào .text section.
GNU/Linux chia memory vào các segments khác nhau tùy theo mục đích sử dụng cụ thể khi nó load một chương trình. Có 4 nhóm cơ bản như sau:
- <b>Text</b> Lưu các instructions và dữ liệu hằng (const). Dữ liệu trong text segment không thể thay đổi được, nó là read-only.
- <b>Data</b> Lưu các biến toàn cục và biến static. Bao gồm có biến đọc ghi và các biến chỉ đọc.
- <b>Stack</b> Các biến local được tạo tự động sẽ được lưu ở đâu. Vùng này là bộ nhớ đọc-ghi, có thể thay đổi trong run time.
- <b>Heap</b> Lưu mấy cái cấp phát động.
Các ELF sections sẽ được gom vào các segment trên, một segment có thể chứa một hoặc nhiều ELF section, việc này thường được thực hiện bằng link script và ld.
Phân biệt segment và section: Segments chứa thông tin cần thiết ở runtime, trong khi section chưa thông tin cần thiết trong bước linking
<br/><br/>
Tiếp theo đó là directive <code>.global main</code>, có tác dụng làm cho label <code>main</code> được biết đến một cách toàn cục, code ở các file khác có thể tham chiếu đến label này. Theo sau nó là directive <code>.type</code>, khai báo rằng main là tên của một function.