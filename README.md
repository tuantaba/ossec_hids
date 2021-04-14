## Cài đặt OSSEC Agent trên Windows
### Mô hình
OSSEC cung cấp chức năng nhận diện nguy cơ từ log hệ thống, phát hiện rootkit, phát hiện
thay đổi file cấu hình hệ thống.  

### Yêu cầu
- Ossec server/manager đã cài đặt
- Windows Server 2008, 2012

### Cài đặt
#### Cấu hình OSSEC
- Các cấu hình của OSSEC Manager/Agent nằm trong file $OSSEC_HOME/etc/ossec.conf
- Cấu trúc config file của OSSEC sử dụng các tag giống như XML hay HTML:
- Bắt đầu với tag <ossec_config>
- Tag <global> cho Manager và <client> cho Agent: cấu hình riêng, ví dụ như cấu hình
- alert mail. Trên Manager host đã cài sẵn postfixd, cấu hình sẵn alert mail nhận trong
- /var/mail/{{ user }}.
- Tag <rule> cho Manager: Thêm các rules (sử dụng tag <include>) cho việc nhận diện
- bằng cách match các nội dung trong các log files. OSSEC đi kèm sẵn với nhiều rules
- cho các loại log như apache, mysql, sshd, pamd.. (file có dạng rule_name.xml, có thể
- chỉnh sửa hoặc tạo mới).
- Tag <syscheck>: cấu hình monitor các folder và file hệ thống hay Windows Registry.
- Tag <rootcheck>: cấu hình phát hiện rootkit
- Tag <localfile>: cấu hình monitor các log hệ thống
- Tag <command> và tag <active-reponse>: Cấu hình các trigger actions cho OSSEC

Xem chi tiết các tag và các tag tại [đây](http://ossec.github.io/docs/syntax/ossec_config.html).
#### Cài đặt agent trên Windows server
Quy trình xác thực kết nối Agent mới đến Manager thông thường sẽ là tạo thông tin cho agent
mới trên manager, sau đó trích xất agent key riêng cho Agent đó, cuối cùng nạp key vừa tạo
vào Agent trước khi thực hiện kết nối.  

Download Windows agent tại [đây](https://bintray.com/artifact/download/ossec/ossec-hids/ossec-agent-win32-2.8.3.exe).  

**Bước 1**: Trên Manager host, thêm Agent và lấy agent key.  

Chạy lệnh $OSSEC_HOME/bin/manage_agents  

Chọn A để thêm thông tin agent mới. Name là bắt buộc, có thể là tên hoặc địa chỉ IP. IP
Address có thể là any.  

Tiếp tục, chọn Q để xuất agent key cho agent mới thiết lập.  

**Bước 2**: Trên Agent host, chạy Programs > Agent Manager để kết nối đến server.  

Điền IP của Manager host và key vừa tạo được ở trên và ấn save. Sau đó chạy **Manage >
Start OSSEC**.  

**Bước 3**: Cấu hình Agent.  
Các cấu hình của Agent, tương tự như với Manager, nằm trong file ossec.conf. Có thể truy
cập từ cửa sổ OSSEC Agent Manager, **View > View Config**. Mỗi khi thay đổi, cần restart lại
OSSEC, **Manage > Restart**.  

Lưu ý: Với agent trên Linux, việc kết nối có thể là tự động theo trình tự: manager mở port
nhận yêu cầu kết nối, agent gửi yêu cầu kết nối kèm theo thông tin Name, manager tự động
gửi lại key cho agent. Nhưng bản agent trên Windows chưa hỗ trợ quy trình này, nên
việc kết nối vẫn bằng cách thủ công. Có thể tham khảo thêm tại [đây](http://philipshramko.blogspot.com/2010/10/deploying-ossec-hids-via-active.html) để đăng kí agent key
cho nhiều server cùng lúc.  
#### Cấu hình tập trung
OSSEC cho phép việc nhiều server có thẻ chia sẻ cùng 1 cấu hình. File cấu hình chia sẻ này
đặt tại Manager host và chứa tất cả cấu hình đã được phân loại cho từng server hay nhómserver (theo tên hay theo OS). Manager sẽ tuần tự quét file cấu hình chia sẻ, sau đó sẽ đẩy
cấu hình mới về các server đã được thiết lập nhận cấu hình.  

Cấu hình tập trung sẽ nằm trong file $OSSEC_HOME/etc/shared/agent.conf (tạo mới nếu
chưa có), có dạng như sau:
```
<agent_config>
  <localfile>
    <location>/var/log/my.log</location>
    <log_format>syslog</log_format>
  </localfile>
</agent_config>

<agent_config name="agent1">
  <localfile>
    <location>/var/log/my.log</location>
    <log_format>syslog</log_format>
  </localfile>
</agent_config>

<agent_config os="Linux">
  <localfile>
    <location>/var/log/my.log2</location>
    <log_format>syslog</log_format>
  </localfile>
</agent_config>

<agent_config os="Windows">
  <localfile>
    <location>C:\myapp\my.log</location>
    <log_format>syslog</log_format>
  </localfile>
</agent_config>
```
Như vậy file cấu hình trên Agent chỉ cần tối thiểu như sau:
```
<ossec_config>
  <client>
    <server-ip>1.2.3.4</server-ip>
  </client>
</ossec_config>
```
Có thể kiểm tra xem cấu hình mới đã được push về đúng Agent chưa bằng cách so sánh mã
MD5 của file agent.conf với client version của Agent bằng cách sử dụng lệnh:
```
$OSSEC_HOME/bin/agent_control -i {{ agent_id }}
```
Sau khi Agent đã nhận cấu hình được push đúng, cần khởi động lại Agent đế thay đổi có hiệu
lực. Có thể restart một agent từ Manager bằng lệnh hoặc có thể thiết lập để Agent tự khởi
động lại mỗi khi file cấu hình chia sẻ được cập nhật (link tham khảo Tips and Hacks ở mục
dưới):
```
$OSSEC_HOME/bin/agent_control -R {{ agent_id }}
```
