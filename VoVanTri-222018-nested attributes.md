Nested attributes cho phép bạn lưu các thuộc tính trên hồ sơ liên quan thông qua cha mẹ. Mặc định trong rails thì nested atrributes updating được tắt và bạn có thể kích hoạt nó bằng cách sư dụng phương thức accepts_nested_attributes_for trong model tương ứng.
Nếu muốn xác nhận rằng một bản ghi con được liên kết với một bản ghi cha, có thể sử dụng phương pháp validates_presence_of và khoá: inverse_of như sau:

class Member < ActiveRecord::Base
  has_many :posts
  accepts_nested_attributes_for :posts
end

class Post < ActiveRecord::Base
  belongs_to :member, inverse_of: :posts
  validates_presence_of :member
end

Lưu ý rằng nếu bạn không chỉ định tùy chọn: inverse_of, thì Active Record sẽ cố gắng tự động đoán mối liên kết ngược dựa trên heuristic.

Khi bạn sử dụng accepts_nested_attributes_for :posts trong model Member thì khi create hoặc update cho đối tượng Member bạn có thể create(update) luôn cho posts bằng cách truyền thuộc tính của posts vào member_params trong controller job

def member_params
    params.require(:member).permit Member::ATTRIBUTES
end

Fields_for tạo ra một scope xung quanh một đối tượng cụ thể như form_for, nhưng không tạo các thẻ form. Điều này làm cho fields_for phù hợp để xác định các đối tượng mô hình bổ sung trong cùng một form của chính nó . Nói một cách đơn giản thì bạn dùng fields_for để xác định đối tượng mà bạn muốn khởi tạo hoặc cập nhật trong nested atttribute .

  ```ruby
  <%= form_for @member do |f| %>
    <%= f.fields_for :posts do |ff| %>
          <%= ff.label :title %>
          <%= ff.file_field :title %>
    <% end %>
    <% end %>
    <%= f.submit "Save", method: :update,
      data: {confirm: "Are you sure"} %>
  <% end %>
  ```

Các tùy chọn được hỗ trợ:

:allow_destroy 
Theo mặc định, bạn sẽ chỉ có thể thiết lập và cập nhật các thuộc tính trên mô hình liên quan. Nếu bạn muốn hủy mô hình liên kết thông qua các thuộc tính băm, bạn phải kích hoạt nó trước bằng cách sử dụng tùy chọn: allow_destroy.
Cập nhật bản ghi bằng các thuộc tính hoặc đánh dấu nó để hủy nếu allow_destroy là true và has_destroy_flag? trả về true.
```ruby  
  def assign_to_or_mark_for_destruction(record, attributes, allow_destroy)
    record.assign_attributes(attributes.except(*UNASSIGNABLE_KEYS))
    record.mark_for_destruction if has_destroy_flag?(attributes) && allow_destroy
  end

  def has_destroy_flag?(hash)
    Type::Boolean.new.cast(hash["_destroy"])
  end

  def allow_destroy?(association_name)
    nested_attributes_options[association_name][:allow_destroy]
  end
```
:reject_if
Bạn cũng có thể thiết lập một: reject_if proc để âm thầm bỏ qua bất kỳ bản ghi mới nếu nó không vượt qua được tiêu chí của bạn.
```ruby
 options[:reject_if] = REJECT_ALL_BLANK_PROC if options[:reject_if] == :all_blank

 REJECT_ALL_BLANK_PROC = proc { |attributes| attributes.all? { |key, value| key ==    "_destroy" || value.blank? } }
```

:limit
Cho phép bạn chỉ định số lượng tối đa các bản ghi liên quan có thể được xử lý với nested attributes.
```ruby
  def check_record_limit!(limit, attributes_collection)
    if limit
      limit = \
        case limit
        when Symbol
          send(limit)
        when Proc
          limit.call
        else
          limit
        end

      if limit && attributes_collection.size > limit
        raise TooManyRecords, "Maximum #{limit} records are allowed. Got #{attributes_collection.size} records instead."
      end
    end
  end
```

:update_only
Đối với mối liên hệ một-một, tùy chọn này cho phép bạn chỉ định cách nested attributes sẽ được sử dụng khi một bản ghi liên quan đã tồn tại. Nói chung, một bản ghi hiện có có thể được cập nhật với tập các giá trị thuộc tính mới hoặc được thay thế bằng một bản ghi hoàn toàn mới có chứa các giá trị đó. Mặc định tùy chọn update_only là false và các nested attributes được sử dụng để cập nhật bản ghi hiện có chỉ khi chúng bao gồm giá trị id của bản ghi. Nếu không một bản ghi mới sẽ được instantiated và được sử dụng để thay thế một hiện có. Tuy nhiên nếu tùy chọn update_only là true, các nested attributes được sử dụng để cập nhật các thuộc tính của bản ghi luôn luôn, bất kể id là có. Tùy chọn này bị bỏ qua cho các hiệp hội thu thập.
```ruby
  def assign_to_or_mark_for_destruction(record, attributes, allow_destroy)
    record.assign_attributes(attributes.except(*UNASSIGNABLE_KEYS))
    record.mark_for_destruction if has_destroy_flag?(attributes) && allow_destroy
  end

  def assign_attributes(new_attributes)
    if !new_attributes.respond_to?(:stringify_keys)
      raise ArgumentError, "When assigning attributes, you must pass a hash as an argument."
    end
    return if new_attributes.blank?

    attributes                  = new_attributes.stringify_keys
    multi_parameter_attributes  = []
    nested_parameter_attributes = []

    attributes = sanitize_for_mass_assignment(attributes)

    attributes.each do |k, v|
      if k.include?("(")
        multi_parameter_attributes << [ k, v ]
      elsif v.is_a?(Hash)
        nested_parameter_attributes << [ k, v ]
      else
        _assign_attribute(k, v)
      end
    end

    assign_nested_parameter_attributes(nested_parameter_attributes) unless 			nested_parameter_attributes.empty?
    assign_multiparameter_attributes(multi_parameter_attributes) unless multi_parameter_attributes.empty?
  end
```
