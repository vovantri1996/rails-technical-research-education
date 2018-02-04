# 1. Metaprogramming là gì ?
Metaprogramming là code sinh code, viết chương trình này sinh ra chương trình kia or điều khiển một chương trình khác

# 2. Tại sao lại sử dụng metaprogramming?
Khi sử dụng metaprogramming thì source trở nên đơn giản hơn, giảm thiểu vấn dề duplicate như các method tương tự nhau, giúp cho hẹ thống đươn giản hơn.

# 3. Cách sử dụng
Mỗi object có các method riêng của nó và các biến instance có thể được thêm vào

Ví dụ: Định nghĩa một method trong object để tạo một instance varible 
```ruby
@my_obj = Object.new
# setter
def set_my_obj=(var)
    @my_obj_instance = var
End
# getter
def get_my_obj
    @my_obj_instance
End
```
vậy ví dụ như trong một model user có nhiều attributes name, email, address thì phải tạo mỗi attributes là 2 cái get set thì có n attributes thì có 2 * n method.

### sử dụng meta:
```ruby
class userModel
  features = ["name", "email", "address"]
  features.each do |feature|
    define_method("#{name}=") do |var|
      instance_variable_set(“@#{feature}”, var)
    End
  End
  features.each do |feature|
    define_method("#{name}=") do
      instance_variable_get(“@#{feature}”)
    End
  End
[…]
End
```
->  for each và khi có thêm instance thì chỉ cần thêm vào mảng là nó sẽ tự sinh các method của attr
....Trong ruby hỗ trợ module Module#attr_accessor, khi khai báo trong đây mặc nhiên nó sẽ sinh ra 2 method get và set cho mỗi obj
```ruby
class UserModel
  attr_accessor :name, :email, :address
End
```
và nhiều attributes thì chúng ta nên tạo một class marco(lớp phụ thuộc). 
```ruby
class UserModel
  def self.features(*agrs)
  args.each do |features|
    attributes_accessor "#{feature}"
end
  End
  features :name, :email, :address
End
```
### Metaprogramming trong devise
- Ở trong helper trong /lib/devise/controllers/helpers.rb (112-139 line)
Khi tạo ra một controler của devise thì tên của controller sẽ được mapping vào để tạo ra các method của helper tương ứng. ví dụ users_controller thì nó sẽ sinh ra các method helper: authenticate_user!, user_signed_in?, current_user, user_session.

- Url helper: Ở đây nó sinh ra các method như là 
```ruby
  # new_session_path(:user)      => new_user_session_path
  # session_path(:user)          => user_session_path
  # destroy_session_path(:user)  => destroy_user_session_path
  # new_password_path(:user)     => new_user_password_path
  # password_path(:user)         => user_password_path
  # edit_password_path(:user)    => edit_user_password_path
  # new_confirmation_path(:user) => new_user_confirmation_path
  # confirmation_path(:user)     => user_confirmation_path
def self.generate_helpers!(routes=nil)
  routes ||= begin
    mappings = Devise.mappings.values.map(&:used_helpers).flatten.uniq
    Devise::URL_HELPERS.slice(*mappings)
  end

  routes.each do |module_name, actions|
    [:path, :url].each do |path_or_url|
      actions.each do |action|
        action = action ? "#{action}_" : ""
        method = :"#{action}#{module_name}_#{path_or_url}"

        define_method method do |resource_or_scope, *args|
          scope = Devise::Mapping.find_scope!(resource_or_scope)
          router_name = Devise.mappings[scope].router_name
          context = router_name ? send(router_name) : _devise_route_context
          context.send("#{action}#{scope}_#{module_name}_#{path_or_url}", *args)
        end
      end
    end
  end
end

generate_helpers!(Devise::URL_HELPERS)
```

- lib/mapping.rb:
```ruby
def self.add_module(m)
  class_eval <<-METHOD, __FILE__, __LINE__ + 1
    def #{m}?
      self.modules.include?(:#{m})
    end
  METHOD
end
```
Dùng cho việc sinh ra các module ở bên thứ 3, Khi thêm vào thì ở đây sẽ sinh ra một method tương ứng với tên module
- ở models.rb
```ruby
def self.config(mod, *accessors) #:nodoc:
  class << mod; attr_accessor :available_configs; end
  mod.available_configs = accessors

  accessors.each do |accessor|
    mod.class_eval <<-METHOD, __FILE__, __LINE__ + 1
      def #{accessor}
        if defined?(@#{accessor})
          @#{accessor}
        elsif superclass.respond_to?(:#{accessor})
          superclass.#{accessor}
        else
          Devise.#{accessor}
        end
      end
      def #{accessor}=(value)
        @#{accessor} = value
      end
    METHOD
  end
end
```
