# Tổng hợp về gem Sidekiq và background jobs

## Overview
1. Background jobs
2. Sidekiq
	* Cài đặt và sử dụng cơ bản
	* Cơ chế cơ bản
	* Các cài đặt nâng cao
	* Scheduled jobs
	* Đi sâu vào cơ chế lấy jobs trong Redis của Sidekiq
	* Về Signals hay vấn đề tắt Sidekiq
	* Delayed extensions
	* Sử dụng cùng ActiveJobs
	* Xử lý lỗi-Error Handling
	* Deploy trên Heroku
3. Tài liệu tham khảo

### Background jobs
- Là những công việc được xử lý ngoài luồng request-response trong các ứng dụng web
- Sử dụng cho những task mất nhiều thời gian xử lý hoặc việc hiển thị kết quả xử lý ra màn hình không quan trọng => nếu một câu query mất nhiều hơn vài giây để xử lý thì nên coi nó như một background job để web có thể phản hồi nhanh hơn. Nếu cần thiết thì sau khi complete background job đó có thể được gọi lại trong web page.
- Những jobs nên được xử lý trong background: xử lý ảnh, gửi email, đăng bài tới các trang social, các jobs được lập lịch (scheduled jobs)
=> Các background jobs thường được xử lý với cấu trúc hàng đợi.

Không sử dụng background jobs <br>
![alt text](http://kamisama.me/wp-content/uploads/2012/10/workflow1.jpg)

Và trường hợp sử dụng background jobs:
![alt text](http://kamisama.me/wp-content/uploads/2012/10/workflow2.jpg)

Tham khảo thêm bài viết về background jobs trong PHP để hiểu rõ hơn: http://kamisama.me/2012/10/09/background-jobs-with-php-and-resque-part-1-introduction/

### Gem Sidekiq
- Sử dụng đa luồng và Redis để xử lý nhiều background jobs đồng thời.
- Redis được sử dụng để lưu trữ các jobs. Mặc định sidekiq kết nối với Redis server bằng địa chỉ localhost:6379 trong môi trường development
#### Cài đặt và sử dụng cơ bản:
- Add gem vào Gemfile: gem "sidekiq"
- Tạo một worker/luồng trong folder `app/worker.rb`
- Tạo job để xử lý bất đồng bộ với method `perform_async`
- Chạy sidekiq server: `bundle exec sidekiq` => Các jobs sẽ được xử lý
#### Cơ chế:
Sidekiq gồm 3 phần chính là **Sidekiq Client**, **Sidekiq Server** và **Redis**.

**- Sidekiq Client**: 
+ Chịu trách nhiệm đẩy jobs vào hàng đợi.
+ Sử dụng hàm JSON.dump để convert các dữ liệu của một jobs thành hash (đọc thêm về job format tại https://github.com/mperham/sidekiq/wiki/Job-Format) 
+ Sử dụng các command của Redis như LPUSH để đẩy job vào hàng đợi trong Redis.
+ Lưu ý: các tham số của worker phải là kiểu dữ liệu JSON đơn giản như numbers, string, boolean, array, hash. Các object phức tạp của Ruby như Date, Time, các model trong ActiveRecord sẽ không được biến đổi

**- Sidekiq Server**:
+ Chịu trách nhiệm lấy jobs từ hàng đợi trong Redis để xử lý
+ Sử dụng lệnh BRPOP của Redis để lấy jobs: khi có job trong queue thì lấy, queue rỗng sẽ đợi đến khi có job thì lấy tiếp. (https://redis.io/commands/brpop) 
+ Server sẽ khởi động worker và gọi tới method `perform` với các tham số được truyền tới. 

**- Redis**:
+ Nơi lưu trữ các jobs
+ Là một in-memory database và cũng có thể coi như một NoSQL database
+ Lưu trữ dữ liệu trong RAM nên có tốc độ nhanh. 
+ Lưu ý khi sử dụng Sidekiq cùng với Redis thì nên coi Redis là một bộ lưu trữ dữ liệu cố định chứ không phải một cache. => Cấu hình lại maxmemory-policy noeviction để Redis không xóa các dữ liệu của Sidekiq
 
#### Các cài đặt nâng cao:
File `config/sidekiq.yml`
- Queue: mặc định Sidekiq sử dụng một hàng đợi duy nhất cho việc xử lý các jobs tên là default => có thể tùy chỉnh để dùng nhiều hàng đợi với các mức ưu tiên khác nhau. 
	+ Một số hàng đợi có sẵn của Sidekiq:
		+ Scheduled: Hàng đợi này gồm các jobs lập lịch được xếp theo thứ tự thời gian.
		+ Retry: Khi một job gặp lỗi, Sidekiq sẽ chuyển job này về hàng đợi Retry.
		+ Dead: Hàng đợi này gồm các jobs được coi như đã "chết". Sidekiq sẽ không retry job nào trong hàng đợi này.
- Workers: có thể coi worker là một luồng đang xử lý các jobs. Có thể tùy chỉnh các option về queue, retry, backtrace cho một worker.
- Xử lý đồng thời: mặc định một tiến trình của Sidekiq tạo 25 luồng. 
#### Xử lý scheduled jobs:
- Cơ chế tổng quát
	+ Sử dụng method `perform_in(interval, *args)` hoặc `perform_at(timestamp, *args)`
	+ Mặc định sidekiq scheduler sẽ kiểm tra các scheduled jobs cách 15s một lần => có thể tùy chỉnh.
- Cơ chế chi tiết:
	+ Có một hàng đợi scheduled queue gồm các scheduled job trong Redis
	+ Job tiếp theo được lấy ra là job có thời điểm thực thi <= thời gian hiện tại
	+ Lấy job khỏi scheduled queue và đẩy vào queue đang làm việc hiện tại
	+ Khi khởi động một Sidekiq process sẽ tạo ra một luồng có Poller thực hiện việc kiểm tra scheduled queue của Redis sau khoảng 15s để lấy ra scheduled job cần thực thi (scheduled job này sẽ được đẩy vào working queue hiện tại) 

#### Đi sâu vào cơ chế lấy jobs từ hàng đợi trong Redis của Sidekiq
Cơ chế đẩy và lấy job khỏi queue của Sidekiq là sử dụng các lệnh của Redis.

- Lấy job từ hàng đợi: 
	+ **Mechanism**: Sidekiq sử dụng lệnh BRPOP của Redis - khi có job trong queue thì lấy, queue rỗng sẽ đợi đến khi có job thì lấy tiếp. (https://redis.io/commands/brpop) 

	=> Một job sau khi lấy ra theo cách này sẽ không còn xuất hiện trong Redis nữa. 

	=> Nếu job này đang được xử lý mà worker bị crash, job đang in-process này cũng sẽ bị crash và mất vĩnh viễn.
	+ **Solution**: không xóa job khỏi Redis cho tới khi nó được thực thi trọn vẹn.

	=> Khi Sidekiq được khởi động lại nó sẽ đẩy lại các job chưa finish vào Redis

	=> Giải pháp này không đảm bảo nếu có lỗi kết nối xảy ra giữa redis server 
	+ Việc lấy jobs được một luồng gọi là *Processor* (`lib/sidekiq/processor.rb`) chịu trách nhiệm: lấy job khỏi redis và chạy job đó (khởi tạo worker, chạy method `perform`)
	+ Nếu có lỗi xảy ra với Processor, nó sẽ gọi tới *Manager* (`lib/sidekiq/manager.rb`). Manager quản lý vòng đời/trạng thái của Processor)
	+ Về Manager: 
		+ Quản lý vòng đời của processor. Chịu trách nhiệm:
			+ Khởi động các processor
			+ Processor crash => Manager sẽ chịu trách nhiệm với các job gặp lỗi, thay thế processor bị crash bằng một processor mới. 
			+ Tắt các processor không hoạt động trong thời gian dài
			+ Tắt các processor khi processor đó hết thời gian hoạt động.
		+ Hard shutdown của Manager: 
			+ terminate các workers khi timeout (kể cả khi worker đó vẫn đang chạy)
			+ Đẩy lại job vào queue trong Redis. (sử dụng bulk_requeue của `fetch.rb`)
	+ **Alternative solution cho vấn đề đẩy lại jobs vào Redis khi Sidekiq crash**: Sử dụng Super Fetch trong phiên bản Sidekiq Pro
		+ Cơ chế này sử dụng command RPOPLPUSH của Redis
		+ RPOPLPUSH sẽ thực hiện việc pop và push các phần tử trong hàng đợi một cách nguyên tố (thao tác phải được thực hiện trọn vẹn hoặc không thực hiện gì - đọc thêm về command RPOPLPUSH ở https://redis.io/commands/rpoplpush)
		+ Khi sidekiq crash thì các jobs đang thực thi sẽ bị đẩy vào một hàng đợi là orphaned working queue 
		+ Khi Sidekiq khởi động lại sẽ thực hiện việc tìm kiếm các orphan jobs này bằng cách quét cả Redis database và đẩy lại các jobs đó vào working queue. 

#### Về signals trong Sidekiq
- Sidekiq có phản hồi với những unix signals - các tín hiệu không đồng bộ được gửi đến tiến trình hoặc luồng để ngắt/thông báo một sự kiện gì đó
- Các signals trong unix/linux: https://en.wikibooks.org/wiki/Guide_to_Unix/Commands/Process_Management/Kill
- Sidekiq phản hồi với các signal: TTIN, TSTP, USR2, TERM
- Các signals sidekiq phản hồi:
	+ TTIN: hiển thị lịch sử của tất cả luồng đang hoạt động trong logfile => debug
	+ TSTP (ở các phiên bản sidekiq cũ hơn 5.0 là USR1): Khi nhận tín hiệu Sidekiq sẽ dừng lấy job mới nhưng vẫn tiếp tục xử lý các jobs hiện tại. (lưu ý là Sidekiq sẽ không tắt, chỉ dừng việc lấy job mới thôi)
	+ USR2: mở lại các file logs đã bị xử lý sau quá trình xoay vòng (logrotate)
	+ TERM: Sidekiq ngừng nhận jobs và tự tắt sau một khoảng thời gian. Worker nào chưa hoàn thành jobs cũng sẽ bị tắt và các jobs được đẩy lại vào Redis.
VD: kill -TERM: gửi một TERM signal để ngừng xử lý jobs mới và sẽ tắt sidekiq sau một khoảng thời gian. trong đó kill là một command có sẵn của unix dùng để gửi signals tới các tiến trình
- Sidekiqctl: hỗ trợ việc tắt Sidekiq server bằng cách gửi đi signal TERM hoặc TSTP để shutdown Sidekiq và sau đó kiểm tra xem server đã được tắt hẳn chưa bằng lệnh kill -9 (lệnh này generate ra signal SIGKILL)
 
#### Về Delayed extensions
- Giúp gọi các method không đồng bộ dễ dàng hơn. 
- Mặc định tính năng này bị vô hiệu hóa từ Sidekiq 5 trở đi => khởi động lại bằng Sidekiq::Extensions.enable_delay!
- Sử dụng với ActionMailer: `delay,delay_for(interval)`, `delay_until(time)`
	+ Tránh việc truyền cả instance objects trong trường hợp này mà chỉ truyền object id
- Sử dụng với các model của ActiveRecord: `delay,delay_for(interval)`, `delay_until(time)`
	+ Không dùng các method này với các object instance vì nó sẽ lưu lại trạng thái object trong Redis.
- Sử dụng với methods của các Class: `MyClass.delay.some_method(1, 'bob', true)`
- Nhược điểm của tính năng này: 
	+ sử dụng YAML để serialize các biến => dữ liệu của các jobs có thể trở nên rất lớn nếu truyền vào các object phức tạp của Ruby (date, time)
	+ Tạo thêm methods trong Class. 
#### Sử dụng cùng ActiveJob:
- ActiveJob là một interface để tương tác với các jobs runner (Delayed Job hoặc Resque hoặc Sidekiq). 
- Các options tùy chỉnh của Sidekiq không thể tùy biến thông qua ActiveJob.
- Cấu hình ActiveJob với Sidekiq: `config.active_job.queue_adapter = :sidekiq`
- Sử dụng rails g tạo job: `rails generate job Example`

	=> tạo một job mới trong folder app/jobs/example_job.rb
- Thêm job vào hàng đợi: `ExampleJob.perform_later args`
- ActiveJob không hỗ trợ đầy đủ các tính năng retry của Sidekiq
- ActionMailer: có thêm method `deliver_later`  để gửi email không đồng bộ (coi việc gửi email là một background job) khi sử dụng ActiveJob cùng Sidekiq.
	+ Các Mailers sẽ được đẩy vào hàng đợi mailers
	+ ActiveJob có thể serialize các instance của activerecord với Global ID sau đó các instance sẽ được deserialize lại: `UserMailer.welcome_email(@user).deliver_later`
- GlobalID: cho phép serialize đầy đủ các object của ActiveRecord như một tham số trong method `perform`. (chú ý giữa việc Sidekiq phải sử dụng object ID) 
=> Nếu một record của ActiveRecord object bị xóa sau khi đã được đưa vào hàng đợi nhưng trước khi được xử lý bằng method `perform` thì sao?
=> ActiveJob sẽ raise một exception trong quá trình deserialize instance object đó.
- ActiveJob sử dụng Job ID riêng không liên quan gì tới Sidekiq. Nếu muốn sử dụng Job ID của Sidekiq thì có thể sử dụng method `provider_job_id`
#### Error handling:
- Nếu job có lỗi sau khoảng 25 lần retry (có thể tùy chỉnh) thì Sidekiq sẽ dừng và đẩy job đó vào hàng đợi Dead Job. Job này sẽ không được retry thêm lần nào nữa.
- Nếu muốn retry lại job này phải thực hiện thủ công qua Web UI của Sidekiq.
- Sau 6 tháng nếu job này không được xử lý thì Sidekiq sẽ xóa job. 
#### Deploy trên Heroku
- Tạo một Procfile để khởi động Sidekiq server cùng Rails
- Giám sát trong môi trường production: đảm bảo các tiến trình Sidekiq hoạt động ổn định và không sử dụng quá nhiều CPU.
	+ Sử dụng Web UI hỗ trợ sẵn.
	+ Các vấn đề với webapp của Sidekiq: mất session, authentication, Rack session,
	+ Sử dụng Scout. 

### Tài liệu tham khảo:
Tóm tắt và tổng hợp từ các nguồn.
- Về background jobs: 
	+ https://blog.iron.io/every-web-application-needs-background/
	+ http://kamisama.me/2012/10/09/background-jobs-with-php-and-resque-part-1-introduction/
- Về sidekiq: 
	+ https://github.com/mperham/sidekiq/wiki
- Về redis: 
	+ https://redis.io/documentation
	+ https://blog.panoply.io/redis-vs-mongodb






