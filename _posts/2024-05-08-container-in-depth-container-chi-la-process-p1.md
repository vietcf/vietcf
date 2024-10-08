---
layout: post
title: "Container in depth - Docker cũng chỉ là các process thông thường, P1"
categories: job
tags: ['Container']
image: assets/img/2024/05/08/0-container-intro.png
---


>Công việc hàng ngày có liên quan tới bảo mật, tôi hiểu rằng trước khi đi vào thực thi bảo vệ một thứ gì đó việc cần làm đầu tiên là hiểu cách nó hoạt động như thế nào càng chi tiết càng tốt. Và hôm nay thứ cần hiểu đó là Container. Nội dung này có nhiều người viết rồi nhưng hôm nay vẫn xin viết lại theo cách của mình (thật ra là tổng hợp lại từ những bài viết tâm đắc) coi như là ghi chú lại cho chính bản thân đọc lại khi cần.

Sử dụng dụng công nghệ Container như Docker ở thời điểm hiện tại tương đối là đơn giản. Việc cần làm là tải về, cài đặt nó và ghi nhớ một số command cơ bản là có thể sử dụng cho công việc ngon rồi. Ở tư cách người dùng (End User) thường không cần phải suy nghĩ về những gì đang xảy ra sâu bên dưới các command vì việc này đã có các công cụ như Docker/Kubernetes hỗ trợ làm hết rồi. Phải nói là công cụ như Docker/Kubernetes đã làm công việc tuyệt vời khi che giấu được sự phức tạp khỏi người dùng. Chính tính dễ dùng cũng là một lý do giúp nó dễ dàng được cộng đồng đón nhận và sử dụng đông đảo. 

Tuy nhiên, trong thực tế đối với các công việc đặc thù như khi cần phải thực hiện gỡ lỗi (troubleshoot) hoặc thực thi bảo mật cho môi trường container, việc hiểu cách tương tác với các container ở mức độ **thấp (~sâu)** có thể rất hữu ích. May mắn thay, do hầu hết các công cụ container được xây dựng trên nền tảng Linux, ta có thể sử dụng các công cụ Linux (thường là Free) hiện có để tương tác và debug khám phá chúng một cách dễ dàng hơn.  

Loạt bài viết này sẽ phân tích cách các container hoạt động ở mức thấp và đi qua một số ý tưởng thực tế để thực thi bảo mật và troubleshoot trong môi trường container. Nội dung sẽ chủ yếu tập trung vào các container tiêu chuẩn như Docker. Nhưng các ví dụ đưa ra cũng sẽ áp dụng được cho các công cụ chạy container khác như Podman, containerd và CRI-O vì ở mức thấp nguyên lý hoạt động của chúng là như nhau.

# Container không có gì là xa lạ, Vẫn chỉ là tập các processes quen thuộc,

Điều đầu tiên cần hiểu về các container là **“từ góc độ của hệ điều hành, chúng là các tiến trình (processes) - giống như bất kỳ ứng dụng nào chạy trực tiếp trên máy chủ”**. Để chứng minh điều này, chúng ta có thể tạo một máy ảo Linux và cài đặt Docker trên đó và cùng tôi thực hiện các phân tích bên dưới đây.

Lưu ý: Nếu bạn muốn tự thực hiện lại các ví dụ trong loạt bài viết này, tôi khuyên bạn nên sử dụng một máy ảo Linux thay vì Docker cho Windows/MacOS, vì việc sử dụng container trên các môi trường này sẽ làm cho việc nhìn thấy những gì đang xảy ra thực tế trở nên khó khăn hơn.

Hãy bắt đầu bằng cách kiểm tra xem có bất kỳ tiến trình nào có tên nginx nào đang hoạt động trên máy ảo của chúng ta không:

```jsx
ps -fC nginx
```

Lệnh này nên trả về một danh sách trống, đơn giản vì chúng ta không có bất kỳ tiến trình NGINX nào đang chạy vào thời điểm này.

![ps nginx]({{site.url}}/assets/img/2024/05/08/1-container-nginx.png)

Bây giờ, hãy bắt đầu một container Docker bằng cách sử dụng image là nginx từ Docker Hub.

```jsx
docker run -d nginx:1.23.1
```

Sau khi container được khởi động, chúng ta sẽ chạy lệnh ps của chúng ta một lần nữa:

```jsx
ps -fC nginx
```

Lần này, chúng ta sẽ nhận được kết quả như dưới đây, cho thấy rằng một số process NGINX đang chạy trên máy. 

![ps nginx2]({{site.url}}/assets/img/2024/05/08/2-container-nginx.png)

Khi chúng ta đi sâu vào khái niệm rằng các container là các Processes, một câu hỏi ban đầu có thể là: “Làm thế nào để phân biệt giữa một máy chủ NGINX được khởi động từ một hình ảnh Docker và một máy chủ NGINX chỉ mới được cài đặt trên máy ảo?”. Có một vài cách để làm điều đó, nhưng cách đầu tiên, dễ nhất là kiểm tra các container đang chạy bằng lệnh docker ps:

![docker ps]({{site.url}}/assets/img/2024/05/08/3-container-dockerps.png)

Hoặc có thể sử dụng các công cụ kiểm tra các process Linux để xác định xem máy chủ web có đang chạy dưới dạng container không. Tùy chọn --forest trong lệnh ps (ví dụ: ps -ef --forest) cho phép chúng ta xem process ở dạng tree (Hiển thị cây tiến trình theo quan hệ cha - con). Trong trường hợp này, các process NGINX của chúng ta có một  parent  process  là containerd-shim-runc-v2. Ta sẽ thấy một process shim cho mỗi container đang chạy trên máy chủ. Tiến trình shim này là một phần của containerd và được Docker sử dụng để quản lý các  process trong container. Mục tiêu của tiến trình shim này là cho phép containerd hoặc daemon Docker được khởi động lại mà không cần phải khởi động lại tất cả các container đang chạy trên máy chủ.

![docker pstree]({{site.url}}/assets/img/2024/05/08/4-container-pstree.png)


# Các cách tương tác với Container process

Theo giải thích ở trên, chúng ta biết rằng ở góc độ hệ điều hành các container cũng chỉ là các process thông thường như bao process khác. Hiểu được điều này sẽ giúp gì cho chúng ta rất nhiều khi muốn tương tác với chúng? Việc có thể tương tác với các process này hữu ích cả trong việc khắc phục sự cố vận hành cũng điều tra các thay đổi trong các container đang chạy. Có một vài điểm cần nhớ ở đây, đầu tiên là chúng ta có thể sử dụng hệ thống tệp /proc để có được thông tin chi tiết hơn về các container đang chạy trên hệ thống.

Hệ thống tệp tin (filesystem) /proc trong Linux là một hệ thống tệp ảo (virtual filesystem), nói là ảo vì nó hoàn toàn không chứa các tệp tin thực sự lưu trên ổ đĩa mà thay vào đó nó giúp cung cấp thông tin về hệ thống đang chạy. Một người dùng (user) có các quyền đặc quyền phù hợp trên một máy chủ chạy Docker hoàn toàn có thể sử dụng /proc để truy cập thông tin của bất kỳ container nào đang chạy trên máy chủ.

Bây giờ hãy áp dụng để xem một số thông tin về container NGINX chúng ta đã khởi động trước đó. Trên hệ thống thử nghiệm mà tôi đang sử dụng, có thể thấy rằng ID tiến trình của nginx là **83675**. 

![ngix tree]({{site.url}}/assets/img/2024/05/08/4-pstree-nginx.png)

Nếu chúng ta liệt kê các tệp trong /proc, chúng ta sẽ thấy mỗi thư mục được đánh số tương ứng với mỗi tiến trình trên máy chủ, bao gồm cả tiến trình NGINX của chúng ta. Trong các thư mục này chứa một loạt các tệp và thư mục với thông tin về tiến trình đó, điều này có nghĩa là chúng ta có thể đi vào thư mục **83675** để tìm hiểu thêm về tiến trình của chúng ta.

![proc]({{site.url}}/assets/img/2024/05/08/4-proc.png)

Việc này có thể hữu ích khi sử dụng các công cụ Linux để làm việc với các container đã được hardening ~ loại bỏ các công cụ như file editor hoặc process manager. Việc hardening image của container là một khuyến nghị bảo mật nên thực hiện, nhưng điều này làm cho việc debug/troubleshoot trở nên khó khăn hơn so với debug trên host linux thông thường. Ta có thể chỉnh sửa các tệp bên trong container bằng cách truy cập vào hệ thống tệp gốc của container từ thư mục /proc trên máy chủ. Đi tới /proc/[PID]/root sẽ cho bạn danh sách thư mục của tiến trình chứa có PID đó. Trong trường hợp này, chạy ```sudo ls /proc/83675/root``` sẽ cho ta kết quả là một danh sách trông giống như sau:

![proc root]({{site.url}}/assets/img/2024/05/08/4-pstree-nginx-root.png)

Bây giờ, nếu chúng ta sử dụng lệnh touch để thêm một tệp vào thư mục này, chúng ta có thể xác nhận rằng nó đã được thêm bằng cách sử dụng lệnh docker exec để liệt kê các tệp trong container. Kỹ thuật này có thể được sử dụng để thực hiện các thao tác như chỉnh sửa các tệp cấu hình trong các container từ máy chủ.

![proc root touch]({{site.url}}/assets/img/2024/05/08/4-pstree-nginx-root-touch.png)

Và đây là một lợi ích khác của việc nhìn các container là process thông thường: Chúng ta có thể sử dụng các công cụ ở mức máy chủ để kill các process mà không cần sử dụng các công cụ của container. Điều này có thể không phải là cách hay, nên sử dụng trong các trường hợp thông thường, vì nó có thể ảnh hưởng đến các chính sách như Docker restart policy, nhưng có thể có những lúc cần thiết sử dụng.

Ví dụ: Hãy kill container NGINX của ta bằng cách sử dụng lệnh ```sudo kill 83675```. Sau đó, chúng ta có thể chạy docker ps để xác nhận rằng container của chúng ta không còn tồn tại nữa.


![proc root kill]({{site.url}}/assets/img/2024/05/08/4-pstree-nginx-root-kill.png)


# Những thông tin trên có ích gì cho security cho docker?

Trong suốt bài viết nhai đi nhai lại câu "Container cũng chỉ là các process thông thường", điều này cũng mang lại nhiều hệ quả liên quan tới vấn đề security.

Trước tiên, ta cần phải tính đến việc bất kỳ ai có quyền truy cập vào máy chủ cơ bản đều có thể sử dụng danh sách tiến trình chạy trên máy chủ (process lists) để xem thông tin về các container đang chạy — ngay cả khi họ không thể trực tiếp truy cập vào các công cụ như Docker.

Một ví dụ điển hình về điều này là sử dụng công cụ Linux để truy cập vào đọc các file của container, đặc biệt là các file cấu hình lưu thông tin bí mật (secret) như token, secret key, password, ... Người dùng có quyền truy cập vào máy chủ host nếu có quyền cơ bản có thể sử dụng thư mục /proc để xem nội dung các file này. 

![proc file]({{site.url}}/assets/img/2024/05/08/5-read-file-secret.png)

Một hệ quả khác là chúng ta có thể sử dụng các công cụ bảo mật Linux hiện có để tương tác với các container. Việc này sẽ phân tích chi tiết trong các bài sau của loạt bài này. Tuy nhiên đến thời điểm hiện tại chúng ta đã chỉ ra rằng có thể kiểm tra hệ thống tệp tin bên trong của một container thông qua thư mục /proc. Điều này có thể rất hữu ích để điều tra các hành động đã được thực hiện, chẳng hạn như xem các tệp mà một kẻ tấn công đã thêm vào một container.

# Tham khảo

[https://securitylabs.datadoghq.com/articles/container-security-fundamentals-part-1/](https://securitylabs.datadoghq.com/articles/container-security-fundamentals-part-1/)