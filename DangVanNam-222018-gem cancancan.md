# Gem CanCanCan
## Date: 2/2/2018
Cancancan là một thư viện phân quyền cho ruby on rails, nó hạn chế các tài nguyên mà một user được phép truy cập. Tất cả các quyền hạn được quy định ở một nơi duy nhất (là class Ability) và riêng biệt với controllers, views và database queries
- Nó bao gồm 2 phần chính:
+ The authorizations definition library: Thư viện xác nhận phân quyền cho phép xác nhận rules cho người dùng để truy cập đến các đối tượng khác nhau và nó cung cấp các helpers để check các quyền đó.
+ Controller helpers:  giúp đơn giản hóa code trong rails controller bằng các thực hiện loading và checking các phân quyền của các model trong controllers.

## I. Cài đặt cơ bản
- Thêm vào gemfile: gem "cancancan",  "~> 2.0" và chạy bundle install.
( đối với Rails < 4.2 : gem "cancancan",  "~> 1.10").     

## II. Bắt đầu sử dụng:
1. Định nghĩa Abilities
- Ability class là nơi mà tất cả các phân quyền user được định nghĩa.
- Phân quyền cho User được định nghĩa trong class Ability:
                                   ```ruby rails g cancan:ability ```
```ruby
class Ability
  include CanCan::Ability

  def initialize(user)
    can :read, :all .  
    if user.present? 
      can :manage, Post, user_id: user.id 
      if user.admin?
        can :manage, :all
      end
    end
  end
end
```
- Các model người dùng hiện thời được cho vào phương thức initialize, vì vậy việc phân quyền có thể được thay đổi dựa trên bất kỳ các thuộc tính của user.
1.1 Can method
- can method được tạo nên trong class ability của cancancan.
- dùng để định nghĩa các quyền và yêu cầu 2 đối số.
+ đối số đầu là action cho phép.
+ đối số thứ 2 là lớp của đối tượng đang thiết lập.
                                     ```ruby can :update,  Article ```
- Bạn có thể đặt :manage để đại diện cho bất kỳ hành động nào và :all đại diện cho bất kỳ đối tượng nào.
```ruby
can :manage,  Article 
can :read,  :all     
can :manage,  :all  
```
- có 4 action thường được sử dụng là create, read, update và destroy.
- Không giống như 7 RESTful action trong Rails, CanCanCan tự động thêm một số alias thuận tiện cho việc lập map action của controller.
```ruby
alias_action :index, :show, :to => :read
alias_action :new, :to => :create
alias_action :edit, :to => :update
```
- Ngoài viêc sử dụng trên ra, ta có thể Custom action 
+  ngoài  7 RESTful action, ta có thể tạo riêng cho mình những action mà ta muốn.
- Ta có thể truyền một mảng cho một trong hai tham số này để khớp với mảng nào. Ví dụ: 
```ruby can [:update,  :destroy], [Article,  Comment] ```
ở đây người dùng có thể update và destroy cả bài viết và comment.
1.2 Hash of Condition
- nó có thể được truyền vào để giới hạn những bản ghi nào mà phân quyền này áp dụng
ví dụ: Ở đây user sẽ chỉ được phép đọc các project đang hoạt động mà họ sở hữu.
                      ```ruby can :read, Project, active: true, user_id: user.id ```
- Điều quan trọng là chỉ sử dụng cột cơ sở dữ liệu cho các điều kiện này. Do đó, nó có thể được sử dụng cho Fetching Records.
- ta có thể sử dụng nested hash để định nghĩa điều trên trên association (Association là cách để tạo ra liên kết giữa 2 model với nhau.)


* Fetching Records
-  là việc hạn chế các bản ghi nào được trả lại từ cơ sở dữ liệu dựa trên những gì người dùng có thể truy cập.
- Điều này có thể được thực hiện bằng phương pháp reach_by trên bất kỳ Active Record model nào. Đơn giản chỉ cần truyền vào nó current ability để chỉ tìm các hồ sơ mà người dùng có thể.
(current_ability là một phương thức do CanCan cung cấp cho controller extending ActionController :: Base).
vd:
```ruby @articles = Article.accessible_by(current_ability) ```
+ Lưu ý: Từ ver 1.4, điều này được thực hiện tự động bởi load_resource method cho action index.
- Bạn có thể thay đổi action bằng cách truyền nó như là đối số thứ hai.
```ruby @articles = Article.accessible_by(current_ability, :update) ```
 Ở đây chỉ tìm thấy các record mà user có quyền để cập nhật.
- Nếu bạn muốn sử dụng action của controller hiện tại, thì thêm to_sym vào nó:
```ruby @articles = Article.accessible_by(current_ability, params[:action].to_sym) ```
- Đây là một Active Record scope. Do đó, các scope và pagination khác có thể bị ràng buộc vào nó .
- Nếu bạn đang sử dụng một cái gì đó khác với Active Record bạn có thể lấy các điều kiện Hash trực tiếp từ ability hiện tại.
```ruby current_ability.model_adapter(TargetClass, :target_action).conditions ```
- Ngoài ra, bạn có thể định nghĩa đoạn SQL bên cạnh các block ( sử dụng  Defining Abilities with Blocks).
* Defining Abilities with Blocks
- Nếu bạn muốn lồng điều kiện phức tạp vào 1 hash, ta có thể sử dụng block trong Ruby để định nghĩa cho nó.
- vd:
```ruby
can :update, Project do |project|
  project.priority < 3
end
```
- Nếu block trả về true thì người dùng có quyền truy cập, nếu không thì anh ta sẽ bị từ chối truy cập.
-  Không viết việc checking quyền truy cập vào trong block. Điều này có nghĩa là bất kỳ điều kiện nào không phụ thuộc vào các thuộc tính đối tượng phải được di chuyển bên ngoài khối.


ví dụ:
```ruby
don't do this:
can :update, Project do |project|
  user.admin? # this won't always get called
end

do this:
can :update, Project if user.admin?
```
- Nếu định nghĩa can hoặc cannot với một khối và một đối tượng không được truyền vào thì việc check sẽ được thông qua.
```ruby
can :update, Project do |project|
  false
end
```
 - các điều kiện của block chỉ được thực thi thông qua Ruby. 
- Để lấy các bản ghi từ cơ sở dữ liệu bạn cần phải thêm vào một chuỗi SQL diễn tả điều kiện cần tới.
can :update, Project, ["priority < ?", 3] do |project|
  project.priority < 3
end
- CanCan 1.6, có thể truyền vào một scope thay vì một chuỗi SQL khi sử dụng một block trong một ability.
ví dụ:
```ruby
can :read, Article, Article.published do |article|
  article.published_at <= Time.now
end
```
có thể hiểu được cú pháp viết nó như sau:
```ruby
can [:ability], Model, Model.scope_to_select_on_index_action do |model_instance|
  model_instance.condition_to_evaluate_for_new_create_edit_update_destroy
end
```
* Overriding All Behavior (ghi đè mọi behavior)
- Bạn có thể ghi đè lên tất cả các hành vi có thể bằng cách không đưa ra đối số, điều này rất hữu ích khi các điều khoản được định nghĩa bên ngoài ruby .
```ruby
can do |action, subject_class, subject|
   ...
end
```
2. Kiểm tra Abilities
- Checking Abilities
- Sau khi ability được định nghĩa, ta có thể sử dụng can? method trong controller hay view để kiểm tra phân quyền user cho một action và object.
```ruby
can? :destroy, @project
```
- cannot? method nghịch chiều với can?
```ruby
cannot? :destroy, @project
```
- Checking với class
+ Ta có thể truyền vào 1 class thay vì instance
```ruby
<% if can? :create, Project %>
  <%= link_to "New Project", new_project_path %>
<% end %>
```
+ Lưu ý: nếu 1 điều kiện của block hay hash tồn tại chúng sẽ bị bỏ qua khi kiểm tra trên lớp và nó sẽ trả về true.
ví dụ:
```ruby
can :read, Project, :priority => 3
can? :read, Project # returns true
```
+ Nó không thể trả lời can? hoàn toàn được, vì không đủ chi tiết được đưa ra. Ở đây, class không có attri priority để check.
3. Controller helpers
3.1 Authorizations
- Phương thức authorize! trong controller sẽ đưa ra một ngoại lệ nếu user không có khả năng để thực hiện hành động đó.
```ruby
def show
  @article = Article.find(params[:id])
  authorize! :read, @article
end
```

3.2 Loaders
- Sử dụng phương thức load_and_authorize_resource để cung cấp ủy quyền tự động cho tất cả các hành động trong RESTfull. Nó sẽ sử dụng bộ lọc trước để load tài nguyên vào một biến instance và cho phép nó mọi hành động.
```ruby
class ArticlesController < ApplicationController
  load_and_authorize_resource

  def show
    # @article is already loaded and authorized
  end
end
```
3.3 Strong Parameters
- bạn có làm sạch đầu ra trước khi lưu bản ghi trong các action như :create và :update






