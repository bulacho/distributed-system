# Bài 1: Giới thiệu

## 1. Hệ thống phân tán

Một nhóm máy tính được tổ chức để cung cấc 1 dịch vụ

Ví dụ:
> - Các hệ thống phổ biến như: hệ thống tin nhắn, các website lớn, DNS, hệ thống điện thoại
> - Các hệ thống như distributed database, transaction systems, hệ thống xác thực, các hệ thống xử lý dữ liệu lớn
> - Các hệ thống phân tán đượ cung cấp như 1 dịch vụ như AWS

Trong repo này, sẽ nói về các dịch vụ cơ sở hạ tầng như lưu trữ cho hệ thống website lớn, MapReduce, peer-to-peer sharing

#### Không dễ để xây dựng những hệ thống phân tán, lý do:
> - Xử lý đồng thời
> - Tương tác phức tạp
> - Khó để đạt được hiệu suất cao
> - Dễ gặp lỗi

#### Tại sao lại sử dụng hệ thống phân tán
> - Tăng công suất thông qua xử lý song song
> - Có khả năng chịu lỗi thông qua các bản sao
> - Phù hợp với các thiết bị vật lý phân tán như sensor
> - Tăng cường bảo mật thông qua isolation

#### Labs
> - Mục tiêu: sử dụng và thực hiện các kỹ thuật, kinh nghiệm quan trọng với chương trình phân tán
> - Gồm 5 bài lab:
>> - Lab 1: Framwork dữ liệu lớn phân tán (MapReduce)
>> - Lab 2: Client/Server vs mạng không tin cậy
>> - Lab 3: Khả năng chịu lỗi khi sử dụng replication(Raft)
>> - Lab 4: Cơ sở dữ liệu có khả năng chịu lỗi
>> - Lab 5: Mở rộng hiệu năng của cơ sở dữ liệu thông qua sharding

## 2. Chủ đề chính

Đây là 1 repo về cơ sở hạ tầng cho ứng dụng
> - Lưu trữ
> - Giao tiếp
> - Tính toán

Mục tiêu lớn: Che dấu độ phức tạp của hệ thống phân tán khỏi ứng dụng

#### Chủ đề: Chịu lỗi
> - 1000 server, mạng lớn -> luôn luôn xảy ra thứ gì đó lỗi. Chúng ta muốn che dấu lỗi khỏi ứng dụng
> - "Tính sẵn sàng cao": dịch vụ vẫn tiếp tục chạy mặc dù có lỗi
> - Ý tưởng: sử dụng những bản sao server. Nếu 1 server gặp sự cố, có thể tiếp tục sử dụng những server khác

#### Chủ đề: Tính nhất quán
> - Các cơ sở hạ tầng có mục đích chung cần được xác định hành vi rõ ràng như: read(x) phải cho giá trị từ lần write(x) gần nhất
> - Đạt được hành vi tốt là rất khó. Ví dụ: "replica" server khó giữ được data giống hệt nhau

#### Chủ đề: Hiệu năng
> - Mục tiêu: Mở rộng thông lượng. Nx server -> Nx tổng thông lượng qua xử lý song song CPU, RAM, disk, net.
> - Mở rộng khó đạt được khi N server tăng lên:
>> - Mất cân bằng tải
>> - Độ chậm chễ.
>> - Một vài thứ không thể tăng lên với N server: khởi tạo, tương tác

#### Chủ đề: Tradeoffs
> - Khả năng chịu lỗi, tính nhất quán và hiệu năng là kẻ thù 
> - Khả năng chịu lỗi và tính nhất quán yêu cầu giao tiếp qua lại của các server
>> - Ví dụ: Gửi data đến các server backup
>> - Ví dụ: Kiểm tra nếu có cached data được cập nhật hay không, giao tiếp giữa các server thường chậm và không mở rộng được.
> - Nhiều thiết kế hy sinh tính nhất quán để đạt được tốc độ.
>> - Ví dụ: Read(x) có thể *không* lấy được những trường mới nhất của write(x)! 
> Chúng ta sẽ thấy nhiều điểm thiết kế trong phạm vi tính nhất quán/hiệu năng 

#### Chủ đề: Implementation
> - RPC, thread, concurrency control, configuration

## 3. Case study: MapReduce

#### Tổng quan
> - Bối cảnh: Hàng giờ tính toán trên hàng terabyte dữ liệu. Ví dụ: Xây dựng search index, hoặc sort, hoặc phân tích cấu trúc của web.
> - Mục tiêu: Sử dụng MapReduce để thực hiện
> - MapReduce Coodinator quản lý, và ẩn dấu tất cả khía cạnh của hệ thống phân tán

#### MapReduce - wordcount
![alt text](/image/lecture-1-map-reduce-overview.png)

1. Đầu vào được chia thành M phần.
2. MR Coodinator gọi hàm Map() với từng đầu vào đã được chia, tạo danh. sách các cặp k,v dữ liệu "trung gian". Mỗi lệnh gọi hàm Map() là 1 "task".
3. Khi hàm Map() thực hiện xong, MR Coodinator tập hợp tất cả các v trung gian cho mỗi k và chuyển từng cặp key, value cho lần gọi hàm Reduce().
4. Đầu ra là tập hợp các cặp từ hàm Reduce()

#### Word-count code
``` 
Map(d)
chop d into words
  for each word w
    emit(w, "1")
```
```
Reduce(k, v[])
  emit(len(v[]))
```
#### MapReduce scale

> - N "worker" (có thể) mang lại Nx throughput. Hàm Map(), Reduce() có thể chạy với xử lý song song.
> - Càng nhiều máy chủ -> càng nhiều throughput

#### MapReduce che dấu nhiều chi tiết:

> - Gửi hàm map+reduce đến các máy chủ
> - Kiểm tra các task đã hoàn thành "shuffling" dữ liệu trung gian từ Map() tới Reduce() để cân bằng tải trên các máy chủ, khôi phục từ máy chủ bị lỗi

#### Để đạt được các lại ích này, MapReduce áp đặt 1 vài rule như sau:

> - Không tương tác hoặc trạng thái() (trừ thông qua đầu ra trung gian)
> - Chỉ sử dụng 1 parttern Map/Reduce cho data flow
> - Không real-time hoặc streaming processing


## REFERENCES
### https://pdos.csail.mit.edu/6.824/notes/l01.txt