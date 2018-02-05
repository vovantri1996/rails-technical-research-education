
# I. Action cable - Tích hợp Websocket cho Rails
Websocket là phương thức giao tiếp 2 chiều giữa client và server. Dữ liệu được truyền tải qua giao thức HTTP(Ajax).
- Nó cho phép viết các tính năng thời gian thực
- Nó cung cấp đầy đủ các tính năng để hỗ trợ liên kết giữa client-side javascrip framework and sever-side Rail framework.
# II. Detail
Websocket có thể chạy trên một server riêng hoặc là thiết lập để chạy trực tiếp trên Rails server. Nó có thể thực hiện được nhiều kết nối cùng một lúc multi thread. Và mỗi kết nối như thế sẽ có một kết nối đến websocket và được gọi là "consumer"(client)
Action cable thông qua Rack socket để quản lý các connection đến các server, đa luồn để tạo nên các connection channel. Với mỗi connection channel như vậy thì sẽ connect tới sub-URI của app để truyển tải dữ liệu đến các vùng nhất định. Nó cung caaso server-side để truyển nội dung như new messages or notification.
Mỗi consumer có thể subscribe nhiều chanel. Vi dụ như 1 consumer có thể subscribe nhiều phòng chat trong cùng 1 thời gian.
Action cable sử dụng redis dể lưu dữ liệu, đồng bộ nọi dung thông qua các instances của ứng dụng.
## 1. Connection
Khi server chấp nhận websocket thì 1 connection object đươc khởi tạo. Object này là cha của tất cả các chanel. Connection sẽ không thực hiện thao tác logic nào ngoài xác thực, và ủy quyền.
Connection là instance của ApplicationCable::Connection : xử lý xác nhận và thiết lập connect
## 2. Channel
1 channel được coi như là 1 controller trong mô hình MVC còn mặc định rails sẽ tạo ra một class cha để đóng gói chia sẻ logic giữa các channel ApplicationCable::Channel::Base
## 3. Stream
Stream cung cấp cơ chế dịnh tuyến channel để publisher nội dung đến subcribers
## 4. Pub/Sub
Là mẫu gửi thông điệp mà người gửi (publishers), không lập trình thông điệp gửi trực tiếp tới người nhận cụ thể (subscribers). Thay vào đó, lập trình viên “gửi” các thông điệp (sự kiện), mà không hề biết gì về người nhận. Tương tự như vậy, người nhận thể hiện sở thích vào một hay nhiều sự kiện (events), và chỉ nhận thông điệp họ mong muốn, mà không hề biết về thông tin người gửi.
## 5. Thiết lâp
- Chúng ta cần tạo một địa chỉ để lắng nghe request lên websocket tại địa chỉ localhost:3000/cable
- THiết lập kết nối từ client lên websocket(consumer)
- Định nghĩa kênh truyền action cable. Tạo một file kees thừa từ ApplicationCable::Channel
- Và khi mà có một tin nhắn được tạo thì chúng ta cần truyền nó đến kênh của customer. Tại đây thì thông tin được truyền tới kênh dưới dạng json
- Action Cable sử dụng redis để gửi và nhận tin nhắn thông qua kênh đã tạo. Vì vậy redis đóng vai trò như là vùng dữ liệu lưu trữ và đảm bảo tính bảo mật để cập nhật qua các vùng của 2 dự án.
# III. Về connection
Khi tạo mới rails app thì mặc định rails đã tạo ra 2 file là channel.rb được kế thừa từ Application::Channel::Base và connection.rb được kế thừa từ Application::Connection::Base.
Class Connection với mục đích chính là để xác thực kết nối mà thực tế là xác thực người dùng khi có yêu cầu kết nối đến một channel.
- Khi Websocket kết nối đến Action cable được chấp nhận thì 1 object connection được khởi tạo và nó là cha của tất cả các kênh sub từ trên đó. Và tin nhắn được gửi đến các sub dựa trên định danh của consumer nào. ví dụ:
```ruby
module ApplicationCable
 class Connection < ActionCable::Connection::Base
   identified_by :current_user

   def connect
     self.current_user = find_verified_user
     logger.add_tags current_user.name
   end

   def disconnect
     # Any cleanup work needed when the cable connection is cut.
   end

   private
     def find_verified_user
       User.find_by_identity(cookies.encrypted[:identity_id]) ||
         reject_unauthorized_connection
     end
 end
end
```
#### indentified_by :current_user:
Khai báo connection này được định danh bởi chính người dùng là current_user. Để từ đó ta có thể tìm được tất cả các connection đến current_user và có thể xóa connection đến các current_user.
indentified_by được định nghĩa ở indentification.rb (20 - 24 line)
```ruby
def identified_by(*identifiers)
  Array(identifiers).each { |identifier| attr_accessor identifier }
  self.identifiers += identifiers
end
```
Như vậy khi khai báo indentify_by thì đồng nghĩa với việc chúng ta khai báo attr_accessor.
Khi connect thì phải xác nhận có phải là current_user được lưu trên cookies hay không và add một tag mới là name của current_use để phân biệt giữa các messages.
- Khi kế thừa từ Base của connection thì nó sẽ khởi tạo websocket, subscription, messages buffer, created_at
#### process method:
```ruby
def process #:nodoc:
  logger.info started_request_message

  if websocket.possible? && allow_request_origin?
    respond_to_successful_request
  else
    respond_to_invalid_request
  end
end
```
được call bởi rails server khi có connection tới websocket mới được thiết lập.
#### close method:
đóng kết nối tới websocket của tab hiện tại
#### recieve:
Dữ liệu được nhận qua handle của method này thông qua cable. Dữ liệu là đoạn json được decode và gửi đến channel phù hợp
#### statistics:
return hash: started_at, indentifier, subscriptions.

# IV. Vấn đề bảo mật
Khi khởi tạo 1 connection đến websocket thì nó đã tạo 1 kết nối đên url <code>"ws://localhost:3000/cable"</code> Vậy ws là gì? <br>
Ws và wss là một scheme(một properti của websocket). ws được dùng cho web có giao thức http còn wss thì chỉ được dùng trên giao thức https.
Khi gửi một đoạn tin nhắn thì nội dung tin nhắn đó đã được mã hóa và gửi với dạng json, sau khi định tuyền đúng đến subcriber thì sẽ decode chuỗi json đó và trả lại cho subcriber.
