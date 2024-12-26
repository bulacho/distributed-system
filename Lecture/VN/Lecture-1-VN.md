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
> - Kết quả thu được: sử dụng và thực hiện các kỹ thuật, kinh nghiệm quan trọng với chương trình phân tán
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

Kết quả: Che dấu độ phức tạp của hệ thống phân tán khỏi ứng dụng

#### Chủ đề: Chịu lỗi
> - 1000 server, mạng lớn -> luôn luôn xảy ra thứ gì đó lỗi. Chúng ta muốn che dấu lỗi khỏi ứng dụng
> - "Tính sẵn sàng cao": dịch vụ vẫn tiếp tục chạy mặc dù có lỗi
> - Ý tưởng: sử dụng những bản sao server. Nếu 1 server gặp sự cố, có thể tiếp tục sử dụng những server khác

#### Chủ đề: Tính nhất quán