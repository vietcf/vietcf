---
layout: post
title: "Active Directory Security - Quay lại các khái niệm cơ bản, P1"
author: "vietcf"
categories: job
tags: ['Active Directory']
image: assets/img/2024/04/10/intro.png
---

Như đã giới thiệu, để bắt đầu tôi sẽ nhắc qua một số khái niệm "theo tôi nghĩ" là quan trọng trước khi nói về AD Security. Các khái niệm này có thể đã quen thuộc với mọi người, đặc biệt là những ai từng làm việc với AD. Tuy nhiên tôi vẫn muốn nhắc lại vì đôi khi có nhiều cái quen quá đâm ra chúng ta lại không để ý hoặc nhầm lẫn, vậy thôi cứ làm rõ ở đây với nhau đi.

## MS sinh ra Active Directory làm gì?

Để "hình dung" AD là gì xin lấy cái hình mô tả sơ qua về nó như sau:

![What is active Directory]( {{site.url}}/assets/img/2024/04/10/0-active-directory.png)

Qua cái hình ta có thể thấy AD nó quá quan trọng phải không. Là nơi quản lý hầu hết các tài sản IT của tổ chức. Từ Personal Computer/Workstation/Server sau đó là User. Chỉ nói riêng về AD User hay được sử dụng là nguồn xác thực cho hầu hết các hệ thống trong một tổ chức. Các hệ thống này rất đa dạng: từ Network (Switch, Router,...); Hạ tầng (Email, Monitor, Chat, Ảo hóa, ...); Bảo mật (Firewall,IPS/IDS, VPN...) cho đến các hệ thống nghiệp vụ (ERP/Core, ...). Do vậy chén được AD thì có kha khá thứ hay ho để làm đó.

Nói về AD là gì thì dài dòng vậy, nhưng theo ý kiến cá nhân tôi MS làm ra AD để giải quyết 2 vấn đề chính nhất dưới đây:

+ Quản lý định danh tập trung trong tổ chức. Định danh này là các đối tượng cụ thể như User, Group, Computer, Máy in, ...

+ Quản lý chính sách tập trung áp dụng lên định danh. Ví dụ như Chính sách về mật khẩu, truy cập, chính sách log audit, ...

## Một chút về các đối tượng trên Active Directory

Đầu tiên lấy cái hình về các đối tượng cơ bản trên AD.

![Full object in AD]( {{site.url}}/assets/img/2024/04/10/01_full_object_in_ad.png)

Chắc hẳn ta đã quá quen thuộc với các khái niệm trên hình như: Domain, User, Group, OU, Computer, Printer, ... Các đối tượng này trên AD được gọi chung là  **Security principals**. Thực tế AD lưu trữ các đối tượng này sử dụng cấu trúc theo chuẩn LDAP (Hầu như tất cả lưu theo cấu trúc này). Do vậy mỗi đối tượng này đều có tập các **Thuộc tính - Giá trị** (key-value) của riêng nó và được lưu trong một Ldap Object.

![Ldap Object attributes]( {{site.url}}/assets/img/2024/04/10/01_ldap_object.png)

### Có một vài khái niệm liên quan tới AD Object ít người để ý xin mô tả lại ở đây

#### Về group
- Group có 2 kiểu là Security group và Distribution groups
    + Security group: được sử dụng để cấp quyền cho người dùng truy cập vào các tài nguyên được chia sẻ. 
    + Distribution groups: Được sử dụng để gửi email đến một tập hợp người dùng cụ thể thông qua các ứng dụng email như Exchange Server
- Group cũng có một số Scope để xác định nó giới hạn tren 1 domain hay mở rộng ra nhiều domain trên forest. Có 3 loại chính 
    + Global groups: Chứa user account và computer account trong domain, đc sử dụng đẻ gán quyền cho các objects bên trong bất cứ domain nào của tree hay forest.
    + Universal group: cho phép truy cập đến tất cả các trusted domains. Chỉ đc sử dụng cho một security group. Có thể bao gồm các thành viên từ bất kỳ domain nào trong forest. Univ group giúp củng cố và quản lý các groups dàn trải trên nhiều domains và thực hiện chung các tác vụ của groups.
    + Domain local groups: bao gồm các groups và user/computer khác. đc định nghĩa và quản lý truy cập đến các tài nguyên bên trong một domain.Các thành viên trong domain local group có thể đc gán quyền chỉ trong một domain.
Thông thường MS khuyến cáo sử dụng global groups hoặc universal groups thay vì domain local groups khi xác định quyền trong domain.

#### Trust in AD
Với hệ thống AD nhỏ có khi bạn không sử dụng cái này. Tuy nhiên với một Enterprise đôi khi vì nhu cầu thực tế họ lại phải sử dụng. Để hiểu về trust bạn hãy hình dung thế này đi bạn có hệ thống AD hoàn toàn độc lập domainA.local và domainB.local. Giờ làm sao để user đã logon trên domainA.local có thể login vào sử dụng các file Server trên domainB.local => Đây là lúc bạn cần thiết lập mối quan hệ trust (~tin tưởng) giữa các domain. 

Mối quan hệ Trust có thể là Two-way hoặc One-way. Với Two-way: Khi đã xác thực (authen) trên một domain thì có khả năng sử dụng tài nguyên trên domain còn lại (Có thể xác thực ở cả 2 phía). Còn One-way thì chỉ cho phép xác thực từ 1 phía (domain) nhất định để sử dụng tài nguyên trên domain còn lại mà thôi. 

Trong One-way lại chia ra outgoing trust và incoming trust. Giả sử miền hiện tại là Example.local và miền tin cậy là Example.chom. Thì với outgoing trust sẽ cho phép users từ Example.com thực hiện xác thực (authen) trên domain hiện tại Example.local. Ngược lại với incoming trust cho phép người dùng từ miền hiện tại Example.local xác thực trong miền được tin cậy (Example.com).


![Oneway trust]( {{site.url}}/assets/img/2024/04/10/01_oneway_trust.png)



### Nói thì nghe hài hước nhưng thỉnh thoảng vẫn có một số khái niệm lẫn lộn cần nhắc lại ở đây

Computer/PC/MemberServer có sự khác biệt nào ở đây??? Thật ra tôi thấy chả có sự khác biệt nào cả, bản chất chúng đều là một dạng đối tượng Computer trên AD mà thôi. Nghĩa là cách thức thực hiện/thiết lập và kết quả mang lại cùng một Policy là giống hệt nhau không có sự khác biệt. Tôi nói ở đây là vì có nhiều Sysadmin khi động vào Member Server cứ kêu oai oái kêu là nó khác với PC :) khóc thét.

Domain vs Domain Controller: Một Domain có các Domain Controllers là các các máy chủ (Đối tượng Computer) có vai trò đặc biệt để vận hành hoạt động của Domain. Các máy chủ này mặc định được bố trí vào một OU riêng tách biệt không lẫn lộn với các các đối tượng khác. Cứ join thêm một Domain Controller (Bao gồm cả Read Only Domain Controller - RODC) thì nó auto được nhét vào đây.

![Domain Controller OU]( {{site.url}}/assets/img/2024/04/10/02-domain-controllers.png)

Ngoài ra có 2 đối tượng Group và OU (Organizational Unit) ai mới làm AD cũng hay bị nhẫm lần giữa chúng. Hãy nhớ rằng Group sinh ra để nhóm các đối tượng quản lý và phân quyền truy cập tới tài nguyên (File/Thư mục) hoặc chính sách gửi mail trong khi OU tập trung vào việc nhóm các đối tượng lại với nhau để thiết lập các chính sách (VD: Chính sách mật khẩu, Audit log, ...). 

Trong AD cũng định nghĩa sẵn một số Group/User có sẵn (built-in User/Group). Các User/Group này một số là các đối tượng với *Đặc quyền** đặc biệt trên hệ thống ~ Privilege User/Group trên hệ thống Active Directory. Nhóm này cần được tập trung tối đa trong việc quản lý/sử dụng vì mất một user trong nhóm này là vô cùng nghiêm trọng. Tôi chia ra làm 2 nhóm với Level khác nhau.

Nhóm "Đặc biệt" quan trọng (Nhóm này ở dạng động vật quý hiếm cần bảo vệ tối đa) bao gồm:

![Privileage Group]( {{site.url}}/assets/img/2024/04/10/03-built-in-privilege-user-group.png)

Nhóm "Quan trọng" (Nhóm cần bảo vệ) bao gồm:

Đọc nhiều hơn bạn có thể đọc ở [Link này](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-b--privileged-accounts-and-groups-in-active-directory#domain-admins). Nhưng tôi nghĩ là không nên vì đặc điểm của MS là viết dai viết dài đâm ra thành viết dại chả ai đọc, các nội dung chính tôi đã tóm tắt ở trên rồi.

Group Policy: Một Group Policy là một tập các chính sách (Policy) áp dụng lên các đối tượng trên AD. Nói lý thuyết là vậy nhưng tôi thấy chủ yếu các Policy được "áp" lên 2 nhóm đối tượng chính và quan trọng nhất là Computer và User. Vì rõ là khi ta show ra chi tiết sẽ nhìn thấy 2 phần riêng biệt rõ ràng Computer và User.

![Computer with User]( {{site.url}}/assets/img/2024/04/10/04-computer-with-user.png)

Tôi nhấn mạnh chỗ này để bạn hiểu rằng nếu mà muốn cấu hình lên các Workstation/Member server một Policy nào đó ở phần Computer thì phải nhét chúng vào OU có Group policy thiết lập Policy đó. Tương tự nếu muốn cấu hình Policy ốp ở phần User lên một User thì phải nhét User vào đúng OU có chứa Group Policy được thiết lập. Có nhiều bạn thắc mắc sao tôi cấu hình Policy mà không ăn thì hóa ra là cấu hình ở phần Computer nhưng lại không nhét Workstation/Member server vào OU thiết lập Policy lại đi nhét User vào đó - cái này không ăn là phải.


## Các cơ chế xác thực trên AD điều cần phải biết,

### Kerberos Authentication

### NetNTLM 

## Sự kết nối giữa nhiều Domain trên AD




