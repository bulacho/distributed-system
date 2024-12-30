# MapReduce: Xử lý dữ liệu trên các cụm lớn

#### Giới thiệu: 

> - Đây là 1 bài tổng quan mình đúc kết từ bài báo MapReduce: Simplified Data Processing on Large Clusters của Jeffrey Dean và Sanjay Ghemawat
> - Link: https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf

#### Tóm tắt

> - MapReduce là một mô hình lập trình và triển khai liên quan để xử lý và tạo các tập dữ liệu lớn.
> - MapReduce là abstraction.
> - Sử dụng *map* function xử lý cặp key/value để tạo 1 bộ trung gian cặp key/value, và 1 *reduce* function để hợp nhất tất cả các giá trị trung gian liên quan với các khóa trung gian.
> - MapReduce có thể được triển khai trên các cụm máy tính lớn, cho phép xử lý nhiều terabyte dữ liệu

## 1. Giới thiệu

> - Trong 5 năm từ trước và trong thời điểm bài viết, nhiều nhà phát triển (author - trong bài viết để là authors) và những người khác làm việc tại Google đã phải thực hiện hàng trăm tính toán đặc biệt để xử lý lượng raw data lớn, ví dụ như crawled doc, web request log,... để tính toán các loại dữ liệu dẫn xuất khác nhau, như inverted indices, các biểu diễn khác nhau của tài liệu web có cấu trúc, tóm tắt số lượng page crawled mỗi máy chủ, tập hợp tru vấn thường xuyên nhất trong một ngày nhất định,... Hầu hết tính toán như vậy đều đơn giản về mặt khái niệm. Tuy nhiên, số lượng data đầu vào thường lớn và việc tính toán phải trải dài trên hàng trăm hoặc hàng nghìn máy chủ phân tán để hoàn thành trong 1 khoảng thời gian hợp lý. Vấn đề là làm sao để thực hiện tính toán song song, phân tán dữ liệu, và xử lý lỗi để làm mờ đi tính toán ban đầu đơn giản với lượng mã phức tạp lớn.
> - Với sự phức tạp này, các nhà phát triển đã thiết kế ra 1 sự trừu tượng mới cho phép họ thể hiện những thứ tính toán đơn giản họ cố gắng thực hiện nhưng ẩn đi các chi tiết phức tạp về xử lý song song, khả năng chịu lỗi và cân bằng tải trong 1 thư viện. Sự trừu tượng được lấy cảm hứng từ các nguyên thủy map và reduce có trong Lisp và nhiều ngôn ngữ hàm khác. Họ nhận ra rằng nhiều trong số tính toán liên quan đến việc áp dụng một thao tác *map* cho mỗi "bản ghi" logic trong đầu vào của để tính toán một tập hợp các cặp khóa/giá trị trung gian, và sau đó áp dụng 1 thao tác *reduce* cho tất cả giá trị được chia sẻ cùng khóa, để tổng hợp dữ liệu. Việc sử dụng mô hình hàm với các thao tác map và reduce do người dùng chỉ định cho phép dễ dàng song song hóa các tính toán lớn và sử dụng tái thực thi như là cơ chế chính để chịu lỗi.
> - Những đóng góp chính của công trình này là một giao diện đơn giản và mạnh mẽ cho phép tự động song song hóa và phân phối các tính toán quy mô lớn, kết hợp với việc triển khai giao diện này để đạt được hiệu suất cao trên các cụm lớn của các máy tính thông thường.
> - Phần 2 mô tả mô hình lập trình cơ bản và đưa ra một vài ví dụ. Phần 3 mô tả cách thực hiện của giao diện MapReduce được điểu chỉnh phù hợp với môi trường tính toán của tác giả. Phần 4 mô tả một số cải tiến của mô hình lập trình mà tác giả thấy hữu ích. Phần 5 bao gồm các phép đo hiệu năng  cho nhiều nhiệu vụ khác nhau. Phần 6 khám phá việc sử dụng MapReduce trong Google bao gồm trải nghiệm của các nhà phá triển trong việc sử dụng nó như cơ sở viết lại 1 sản phẩm hệ thống index. Phần 7 thảo luận liên quan và tương lai làm việc

## 2. Mô hình lập trình

> - Việc tính toán lấy 1 tập hợp đầu vào các cặp key/value, và đưa ra 1 tập hợp đầu ra các cặp key/value. Người dùng thư viện MapReduce thể hiện tính toán qua 2 function: Map và Reduce
> - Map, viết bởi người dùng, lấy 1 cặp đầu vào và đưa ra 1 tập hợp các cặp trung gian key/value. Thư viện MapReduce nhóm tất cả các giá trị trung gian liên hệ với nhau qua 1 khóa trung gian và đưa chúng tới hàm Reduce
> - Reduce, viết bởi người dung, chấp nhận 1 khóa trung gian và tập hợp các giá trị cho khóa đó. Nó hợp nhất lại các giá trị để tạo thành 1 tập hợp giá trị nhỏ hơn. Thông thường, chỉ có không hoặc một giá trị đầu ra được tạo ra cho mỗi lần gọi Reduce. Các giá trị trung gian được cung cấp cho hàm reduce của người dùng thông qua một bộ lặp. Điều này cho phép ta có thể xử lý 1 danh sách giá trị lớn để phù hợp với bộ nhớ.

### 2.1. Ví dụ

> - Hãy xem xét vấn đề đếm số lần xuất hiện của mỗi từ trong một tập hợp lớn các tài liệu. Người dùng sẽ viết mã tương tự như mã giả sau:
```
  map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
      EmitIntermediate(w, "1");
  
  reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
      result += ParseInt(v);
    Emit(AsString(result))
```
> - Hàm map đưa ra mỗi từ cùng với số lần xuất hiện liên quan (chỉ là '1' trong ví dụ đơn giản này). Hàm reduce cộng tất cả các số đếm được đưa ra cho một từ cụ thể.
> - Ngoài ra, người dùng viết mã để điền vào một đối tượng đặc tả MapReduce với tên của các tệp đầu vào và đầu ra, và các tham số điều chỉnh tùy chọn. Sau đó, người dùng gọi hàm MapReduce, truyền cho nó đối tượng đặc tả. Mã của người dùng được liên kết với thư viện MapReduce (được triển khai bằng C++).

### 2.2. Types

> - Mặc dù mã giả trước đó được viết dưới dạng đầu và đầu ra, về khái niệm, các hàm map và reduce do người dùng cung cấp các kiểu liên quan
```
  map(k1,v1) -> list(k2,v2)
  reduce (k2,list(v2)) -> list(v2)
```
> - Nghĩa là, các khóa và giá trị đầu vào được rút ra từ một miền khác với các khóa và giá trị đầu ra. Hơn nữa, các khóa và giá trị trung gian thuộc cùng một miền với các khóa và giá trị đầu ra.

### 2.3. Ví dụ

> - Dưới đây là 1 vài ví dụ thú vị về chương trình có thể dễ dàng thể hiện dưới dạng tính toán MapReduce
>> - **Distributed Grep**: Hàm Map đưa ra 1 dòng nếu nó khớp với 1 khuôn mẫu được cung cấp. Hàm Reduce là 1 hàm nhận diện chỉ sao chép các dữ liệu trung gian được cung cấp ra output
>> - **Đếm tần suất truy cấp URL**: Hàm Map xử lý log truy cập của 1 web và đầu ra (URL,1). Hàm Reduce tổn hợp chúng lại với tất cả giá trị cho cùng 1 URl và đưa ra 1 cặp (URL, total count)
>> - **Reverse Web-Link Graph**: Đầu ra hàm Map là cặp (target, source) cho mỗi đường dẫn tới target URL được tìm thấy trong page có tên là source. Hàm Reduce nối danh sách của tất cả source URL liên quan với 1 URL target và đưa ra output (target, list(source))
>> - **Term-Vector per Host**: Một vector thuật ngữ tóm tắt các từ quan trọng nhất xuất hiện trong một tài liệu hoặc một tập hợp tài liệu dưới dạng danh sách các cặp ⟨từ, tần suất⟩. Hàm map phát ra một cặp ⟨hostname, vector thuật ngữ⟩ cho mỗi tài liệu đầu vào (trong đó hostname được trích xuất từ URL của tài liệu). Hàm reduce nhận tất cả các vector thuật ngữ cho mỗi tài liệu của một máy chủ cụ thể. Nó cộng các vector thuật ngữ này lại, loại bỏ các thuật ngữ không thường xuyên, và sau đó phát ra một cặp ⟨hostname, vector thuật ngữ⟩ cuối cùng.
>> - **Inverted Index**: Hàm Map phân tích mỗi tài liệu và đưa ra 1 chuỗi cặp (word,documentID). Hàm Reduce chấp nhận tất cả các cặp cho 1 từ, sắp xếp các documentID giống nhau và đưa ra 1 cặp (word,list(document ID)). Tập hợp của tất cả cặp đầu ra tạo thành 1 inverted index. Việc bổ sung tính toán này để theo dõi vị trí của các từ cũng rất dễ dàng.
>> - **Distributed Sort**: Hàm Map lấy ra mỗi khóa từ từng bản ghi và đưa ra cặp (key, record). Hàm Reduce đưa ra tất cả cặp không thay đổi. Việc tính toán dựa trên các cơ sở phân vùng được mô ta trong phần 4.1 và thuộc tính sắp xếp mô tả ở phần 4.2

## 3. Implementation
> - Nhiều triển khai khác nhau của giao diện MapReduce là có thể. Lựa chọn đúng phụ thuộc và môi trường. Ví dụ, một triển khai có thể phù hợp cho một máy chia sẻ bộ nhớ nhỏ, một triển khai khác cho một bộ xử lý đa nhân NUMA lớn, và một triển khai khác nữa cho một tập hợp lớn hơn các máy tính kết nối mạng.
> - Phần này mô tả 1 triển khai hướng để tính toán trong môi trường Google đang sử dụng: cụm lớn máy chủ kết nối với nhau qua switched Ethernet. Môi trường của họ bao gồm:
>> - (1) Các máy tính được với bộ xử lý 2 nhân - x86, chậy Linux với 2-4Gb bộ nhớ
>> - (2) Phần cứng mạng thông thường được sử dụng - thường là 100 megabit/giây hoặc 1 gigabit/giây ở mức độ máy, nhưng trung bình ít hơn đáng kể.
>> - (3) 1 cụm bao gồm hàng trăm hoặc ngàn máy chủ, do đó các lỗi máy là điều thường xuyên xảy ra.
>> - (4) Bộ nhớ được cung cấp bởi các đĩa IDE giá rẻ gắn trực tiếp vào từng máy. Một hệ thống tệp phân tán được phát triển nội bộ được sử dụng để quản lý dữ liệu được lưu trữ trên các đĩa này.
>> - (5) Người dùng gửi các công việc vào một hệ thống lập lịch. Mỗi công việc bao gồm một tập hợp các tác vụ và được lập lịch bởi bộ lập lịch để ánh xạ đến một tập hợp các máy có sẵn trong một cụm.

### 3.1. Tổng quan về thực thi
> - Các lời gọi Map được phân phối trên nhiều máy bằng cách tự động phân vùng dữ liệu đầu vào thành một tập hợp các phần M. Các phần đầu vào có thể được xử lý song song bởi các máy khác nhau. Các lời gọi Reduce được phân phối bằng cách phân vùng không gian khóa trung gian thành R phần bằng cách sử dụng một hàm phân vùng (ví dụ: hash(key) mod R). Số lượng phân vùng (R) và hàm phân vùng được chỉ định bởi người dùng.
![alt text](/image/figure-1-MapReduce(2004).png)
Hình 1: Execution overview
> - Hình 1 cho thấy luồng tổng thể của một hoạt động MapReduce trong triển khai. Khi chương trình người dùng gọi hàm MapReduce, chuỗi các hành động sau sẽ xảy ra (các nhãn được đánh số trong Hình 1 tương ứng với các số trong danh sách dưới đây):
>> - 1. Thư viện MapReduce trong chương trình người dùng đầu tiên chia các tệp đầu vào thành M phần, mỗi phần thường có kích thước từ 16 megabyte đến 64 megabyte (MB) (có thể điều chỉnh bởi người dùng thông qua một tham số tùy chọn). Sau đó, nó khởi động nhiều bản sao của chương trình trên một cụm máy.
>> - 2. Một trong các bản sao của chương trình là đặc biệt - master. Các bản sao còn lại là các worker được master chỉ định công việc. Có M tác vụ map và R tác vụ reduce để chỉ định. Master chọn các worker nhàn rỗi và chỉ định cho mỗi worker một tác vụ map hoặc reduce.
>> - 3. Một worker được chỉ định một tác vụ map đọc nội dung của phần đầu vào tương ứng. Nó phân tích các cặp khóa/giá trị từ dữ liệu đầu vào và chuyển mỗi cặp đến hàm Map do người dùng định nghĩa. Các cặp khóa/giá trị trung gian được tạo ra bởi hàm Map được lưu trữ trong bộ nhớ.
>> - 4. Định kỳ, các cặp được lưu trữ trong bộ nhớ được ghi vào đĩa cục bộ, phân vùng thành R vùng bằng hàm phân vùng. Vị trí của các cặp được lưu trữ trên đĩa cục bộ được chuyển lại cho master, người chịu trách nhiệm chuyển tiếp các vị trí này cho các worker reduce.
>> - 5. Khi một worker reduce được master thông báo về các vị trí này, nó sử dụng các cuộc gọi thủ tục từ xa để đọc dữ liệu được lưu trữ từ các đĩa cục bộ của các worker map. Khi một worker reduce đã đọc tất cả dữ liệu trung gian, nó sắp xếp dữ liệu theo các khóa trung gian để tất cả các lần xuất hiện của cùng một khóa được nhóm lại với nhau. Việc sắp xếp là cần thiết vì thường có nhiều khóa khác nhau ánh xạ đến cùng một tác vụ reduce. Nếu lượng dữ liệu trung gian quá lớn để vừa trong bộ nhớ, một sắp xếp bên ngoài được sử dụng.
>> - 6. Worker reduce lặp qua dữ liệu trung gian đã sắp xếp và đối với mỗi khóa trung gian duy nhất gặp phải, nó chuyển khóa và tập hợp các giá trị trung gian tương ứng đến hàm Reduce của người dùng. Đầu ra của hàm Reduce được thêm vào một tệp đầu ra cuối cùng cho phân vùng reduce này.
>> - 7. Khi tất cả các tác vụ map và reduce đã hoàn thành, master đánh thức chương trình người dùng. Tại thời điểm này, lời gọi MapReduce trong chương trình người dùng trả về mã người dùng.

>> - Sau khi hoàn thành thành công, đầu ra của việc thực thi MapReduce có sẵn trong các tệp đầu ra R (một tệp cho mỗi tác vụ reduce, với tên tệp được chỉ định bởi người dùng). Thông thường, người dùng không cần phải kết hợp các tệp đầu ra R này thành một tệp - họ thường chuyển các tệp này làm đầu vào cho một lời gọi MapReduce khác hoặc sử dụng chúng từ một ứng dụng phân tán khác có thể xử lý đầu vào được phân vùng thành nhiều tệp.

### 3.2. Cấu trúc dữ liệu của Master
> - Master giữ lại 1 số cấu trúc dữ liệu. Với từng tác vụ map và reduce, nó lưu trữ trạng thái(idle, in-progress, hoặc completed), và danh tính của từng worker (đối với các tác vụ non-idle).
> - Master là kênh thông qua đó vị trí của các vùng file trung gian được truyền từ map sang reduce. Do đó, đối với từng tác vụ map hoàn thành, master lưu trữ vị trí và kích cơ của các vùng file trung gian R được tạo bởi tác vụ Map. Các bản cập nhật về vị trí và kích thước này được nhận khi tác vụ map hoàn thoành. Thông tin được đẩy dần dần đến các worker có các tác vụ reduce có trạng thái in-progress.

### 3.3. Khả năng chịu lỗi
> - Vì thư viện MapReduce được thiết kể để giúp xử lý 1 lượng lớn data xử dụng hàng trăm hoặc hàng nghìn máy chủ, thư viện phải chịu lỗi 1 cách linh họat

#### Worker Failure
> - Master ping các worker 1 cách định kỳ. Nếu không có phản hồi được gửi lại từ worker trong 1 khoảng thời gian nhất định, master gán woker là lỗi. Các tác vụ map hoàn thành bởi woker sẽ được đặt lại trạng thái idle ban đầu, và do đó trở nên đủ điều kiện để lập lịch trên các worker khác. Tương tự, bất kỳ tác vụ map hoặc reduce đang thực hiện trên worker lỗi sẽ trở lại trạng thái idle và đủ điều kiện để lập lịch trên các worker khác.
> - Các tác vụ đã hoàn thành được thực hiện lại khi có lỗi vì đầu ra của chúng được lưu trên máy chủ có lỗi và do đó không thể truy cập được. Các tác vụ reduce đã hoàn thành không cần phải thực hiện lại vì đầu ra của chúng được lưu trữ trong hệ thống tệp toàn cầu.
> - Khi 1 tác vụ map hoàn thành đầu tiên bởi worker A và sau đó lại được thực thi bởi worker B(vì woker A lỗi), tất cả worker thực hiện tác vụ reduce được thông báo thực hiện lại. Bất kỳ tác vụ reduce nào chưa đọc dữ liệu từ worker A sẽ đọc dữ liệu từ worker B.
> - MapReduce có khả năng chịu lỗi lớn. Ví dụ, trong một hoạt động MapReduce, bảo trì mạng trên một cụm đang chạy đã khiến các nhóm gồm 80 máy cùng lúc không thể truy cập trong vài phút. Master MapReduce chỉ đơn giản là thực hiện lại công việc do các máy worker không thể truy cập thực hiện và tiếp tục tiến hành, cuối cùng hoàn thành hoạt động MapReduce.

#### Master Failure
> - Đơn giản để làm cho master ghi các điểm kiểm tra định kỳ của các cấu trúc dữ liệu master được mô tả ở trên. Nếu tác vụ master chết, một bản sao mới có thể được khởi động từ trạng thái đã kiểm tra cuối cùng. Tuy nhiên, vì chỉ có một master, khả năng thất bại của nó là không cao; do đó, triển khai hiện tại hủy bỏ tính toán MapReduce nếu master thất bại.

#### Ngữ nghĩ trong trường hợp có lỗi
> - Trong trường hợp có lỗi, MapReduce đảm bảo rằng các toán tử map và reduce do người dùng cung cấp là các hàm xác định của các giá trị đầu vào của chúng, triển khai phân tán tạo ra cùng một đầu ra như sẽ được tạo ra bởi một thực thi tuần tự không lỗi của toàn bộ chương trình.
> - Để đạt được điều này, MapReduce dựa vào các cam kết nguyên tử của đầu ra tác vụ map và reduce. Mỗi tác vụ đang tiến hành ghi đầu ra của nó vào các tệp tạm thời riêng tư. Khi một tác vụ map hoàn thành, worker gửi một thông báo đến master và bao gồm tên của các tệp tạm thời trong thông báo. Nếu master nhận được thông báo hoàn thành cho một tác vụ map đã hoàn thành, nó bỏ qua thông báo. Nếu không, nó ghi lại tên của các tệp trong một cấu trúc dữ liệu master.
> - Khi một tác vụ reduce hoàn thành, worker reduce đổi tên tệp đầu ra tạm thời của nó thành tệp đầu ra cuối cùng một cách nguyên tử. Nếu cùng một tác vụ reduce được thực hiện trên nhiều máy, nhiều lệnh đổi tên sẽ được thực hiện cho cùng một tệp đầu ra cuối cùng. MapReduce dựa vào thao tác đổi tên nguyên tử được cung cấp bởi hệ thống tệp cơ bản để đảm bảo rằng trạng thái hệ thống tệp cuối cùng chỉ chứa dữ liệu được tạo ra bởi một lần thực thi của tác vụ reduce.
> - Phần lớn các toán tử map và reduce của MapReduce là xác định, và thực tế rằng ngữ nghĩa của chúng tương đương với một thực thi tuần tự trong trường hợp này làm cho lập trình viên dễ dàng suy luận về hành vi của chương trình của họ. Khi các toán tử map và/hoặc reduce không xác định, MapReduce cung cấp ngữ nghĩa yếu hơn nhưng vẫn hợp lý. Trong trường hợp có các toán tử không xác định, đầu ra của một tác vụ reduce cụ thể tương đương với đầu ra cho tác vụ đó được tạo ra bởi một thực thi tuần tự của chương trình không xác định. Tuy nhiên, đầu ra cho một tác vụ reduce khác có thể tương ứng với đầu ra cho tác vụ đó được tạo ra bởi một thực thi tuần tự khác của chương trình không xác định.
> - Điều này có nghĩa là, trong trường hợp có lỗi, MapReduce vẫn đảm bảo tính nhất quán và độ tin cậy của kết quả đầu ra, ngay cả khi phải thực hiện lại các tác vụ map hoặc reduce

### 3.4. Locality
> - Băng thông mạng là một tài nguyên tương đối khan hiếm trong môi trường tính toán. Các nhà phát triển tiết kiệm băng thông mạng bằng cách tận dụng thực tế rằng dữ liệu đầu vào (được quản lý bởi GFS) được lưu trữ trên các đĩa cục bộ của các máy tạo thành cụm. GFS chia mỗi tệp thành các khối 64 MB và lưu trữ nhiều bản sao của mỗi khối (thường là 3 bản sao) trên các máy khác nhau. Master MapReduce xem xét thông tin vị trí của các tệp đầu vào và cố gắng lập lịch một tác vụ map trên một máy chứa một bản sao của dữ liệu đầu vào tương ứng. Nếu không thành công, nó cố gắng lập lịch một tác vụ map gần một bản sao của dữ liệu đầu vào của tác vụ đó (ví dụ: trên một máy worker nằm trên cùng một công tắc mạng với máy chứa dữ liệu). Khi chạy các hoạt động MapReduce lớn trên một phần đáng kể của các worker trong một cụm, hầu hết dữ liệu đầu vào được đọc cục bộ và không tiêu tốn băng thông mạng.

### 3.5. Task Granularity
> - Chia gia đoạn map thành M phần và giai đoạn reduce thành R phần. Lý tưởng nhất, nên M và R nên nhiều hơn số lượng máy chủ worker. Việc mỗi worker thực hiện các tác vụ khác nhau nâng cao cân bằng tải, và tăng tốc độ phục hồi khi 1 worker lỗi: các tác vụ tast đã hoàn thành có thể được phân phối lại trên các worker khác
> - Có những giới hơn thức tế trên độ lớn của M và R trong triển khai, vì master phải thực hiện O(M+R) quyết định lập lịch và giữ O(O+R) trạng thái trong bộ nhớ.
> - Hơn thế nữa, R thường ràng buộc bởi người dùng vì output với từng tác vụ task kết thúc trong một tệp riêng biệt. Trong thực tế, tác giả có xu hướng chọn M sao chỗi mỗi tác vụ riêng lẻ là 16MB đến 64MB dữ liệu đầu vào, và làm cho R là bội số nhỏ của số lượng worker dự kiến sử dụng. trong bài toán này là M = 200,000 và R = 5000 sử dụng 2000 máy chủ

### 3.6. Backup Tasks
> - Một trong những nguyên nhân phổ biến làm kéo dài thời gian hoàn thành tổng thể của một hoạt động MapReduce là một "straggler": một máy mất một thời gian dài bất thường để hoàn thành một trong những tác vụ map hoặc reduce cuối cùng trong tính toán. Stragglers có thể phát sinh vì nhiều lý do. Ví dụ, một máy có đĩa cứng bị lỗi có thể gặp phải các lỗi có thể sửa chữa thường xuyên, làm chậm hiệu suất đọc từ 30 MB/s xuống còn 1 MB/s. Hệ thống lập lịch cụm có thể đã lập lịch các tác vụ khác trên máy, khiến nó thực hiện mã MapReduce chậm hơn do cạnh tranh về CPU, bộ nhớ, đĩa cục bộ hoặc băng thông mạng. Một vấn đề gần đây mà tác giả gặp phải là một lỗi trong mã khởi tạo máy khiến bộ nhớ đệm của bộ xử lý bị vô hiệu hóa: các tính toán trên các máy bị ảnh hưởng chậm lại hơn một trăm lần.
> - Tác giả có một cơ chế chung để giảm bớt vấn đề của các straggler. Khi một hoạt động MapReduce gần hoàn thành, master lập lịch các thực thi dự phòng của các tác vụ đang tiến hành còn lại. Tác vụ được đánh dấu là hoàn thành bất cứ khi nào thực thi chính hoặc thực thi dự phòng hoàn thành. Tác giả đã điều chỉnh cơ chế này để nó thường chỉ tăng tài nguyên tính toán được sử dụng bởi hoạt động không quá vài phần trăm. Tác giả nhận thấy rằng điều này giảm đáng kể thời gian hoàn thành các hoạt động MapReduce lớn. Ví dụ, chương trình sắp xếp được mô tả trong Phần 5.3 mất 44% thời gian hoàn thành khi cơ chế tác vụ dự phòng bị vô hiệu hóa.

## 4. Refinements
> - Mặc dù chức năng cơ bản cung cấp 1 cách đơn giản để viết hàm Map và Reduce là đủ cho cho nhu cầu, tác giả đã tìm thấy một số phần mở rộng hữu ích. Những phần mở rộng này được mô tả ở phần này

### 4.1. Partitioning Function
> - Người dùng chỉ định số lượng tác vụ task/ số lượng output file mà họ mông muốn(R). Dữ liệu phân vùng qua các tác vụ bằng cách sử dụng 1 hàm phân vùng trên khóa trung gian. 1 hàm phân vùng mặc định được cung cấp bằng cách sử dụng hashing (ví dụ: hash(key) mod R). Việc này dẫn dến các phân vùng khá cân bằng. Tuy nhiên, trong 1 số trường hợp, viêc phân vùng dữ liệu theo cách khác lại hữu ích. Ví dụ, thỉnh thoảng đầu ra key là URL, và ta muốn tất cả các mục cho một máy chủ cụ thể kết thúc trong cùng một tệp đầu ra. Để hỗ trợ các tình huống như vậy, người dùng của thư viện MapReduce có thể cung cấp một hàm phân vùng đặc biệt. Ví dụ, sử dụng "hash(Hostname(urlkey)) mod R" làm hàm phân vùng sẽ khiến tất cả các URL từ cùng một máy chủ kết thúc trong cùng một tệp đầu ra.

### 4.2. Ordering Guarantees
> - Tác giả đảm bảo rằng trong một phân vùng nhất định, các cặp khóa/giá trị trung gian được xử lý theo thứ tự tăng dần của khóa. Đảm bảo thứ tự này giúp dễ dàng tạo ra một tệp đầu ra được sắp xếp theo phân vùng, điều này hữu ích khi định dạng tệp đầu ra cần hỗ trợ tra cứu ngẫu nhiên hiệu quả theo khóa, hoặc người dùng của đầu ra thấy tiện lợi khi có dữ liệu được sắp xếp.

### 4.3. Combiner Function
> - Trong một vài trường hợp, có sự lạp lại quan trọng trong khóa trung gian xử lý bởi mỗi tác vụ map, và hàm Reduce do người dùng chỉ định là giao hoán và kết hợp. 1 ví dụ điển hình là đếm số lượng từ trong ví dụ ở phần 2.1. Vì tần suất của từ thường tuân theo phân phối Zipf, với mỗi taask sẽ tạo ra hàng trăm hoặc hàng ngàn bản ghi dưới dạng (the, 1). Tấ cả các số đếm này sẽ được gửi qua mạng thành 1 tác vụ task và sau đó cộng chúng lại với nhau bằng hàm Reduce để tạo ra 1 số. Tác giả cho phép người dùng chỉ định 1 hàm Combiner tùy chọn để kết hợp một phần dữ liệu trước khi gửi chúng qua mạng.
> - Hàm Combiner thực thi trên mỗi máy chủ (thực hiện tác vụ map). Thông thường, cùng một mã được sử dụng để triển khai cả hàm combiner và hàm reduce. Chỉ duy nhất 1 điểm khác biệt giữa hàm reduce và hàm combiner là cách thư viện MapReduce sẽ xử lý dữ liệu đầu ra của hàm. Đầu ra của hàm reduce được viết dưới dạng output file cuối cùng. Đầu ra của hàm combiner được viết dưới dạng file trung gian và gửi cho tác vụ reduce.
> - Việc kết hợp Combiner tăng tốc độ của một số loại hoạt động MapReduce.

### 4.4. Dạng Input và Output
> - Thư viện MapReduce cung cấp hỗ trợ cho việc đọc đầu vào dữ liệu ở nhiều định dạng khác nhau. Ví dụ, chế độ "text" xử lý mỗi dòng như một cặp key/value: key là độ lệch trong file và value là nội dung trong dòng đó. Một định dạng phổ biến khác được hỗ trợ lưu trữ một chuỗi các cặp key/value được sắp xếp theo key. Mỗi triển khai kiểu đầu vào biết cách chia nhỏ chúng thành các phạm vi có ý nhĩa để xử lý như các tác vụ map riêng biệt (ví dụ: Phạm vi của chế độ text đảm bảo việc chia nhỏ chỉ xảy ra tại các ranh giới dòng). Người dùng có thể thêm hỗ trợ cho 1 kiểu đầu vào mới bằng việc cung cấp 1 triển khai đơn gian của 1 reader interface, mặc dù nhiều người dùng chỉ dùng 1 phần nhỏ các loại đầu vào được định nghĩa trước.
> - 1 Reader không cần quan trong trong việc cung cấp dữ liệu được từ 1 file. Ví dụ, nó đơn gian là định nghĩa 1 reader đọc bản ghi từ 1 database, hoặc từ 1 cấu trúc dữ liệu được ánh xạ từ bộ nhớ.

### 4.5. Side-effect
> - Trong 1 vài trường hợp, người dùng tìm thấy tiện ích khi tạo ra file phụ trợ gióng như các đầu ra bổ sung từ các hoạt động map và/hoặc reduce. Ta dựa vào người viết chương trình để làm cho tác động phụ này atomic và idempotent. Thông thường ứng dụng viết 1 file tạm và đổi tên file khi đã hoàn thành.
> - Tác giả không cung cấp hỗ trợ cho atomic two-phase commit của nhiều tệp đầu ra được tạo ra bởi một tác vụ duy nhất. Do đó, các tác vụ tạo ra nhiều tệp đầu ra với các yêu cầu nhất quán giữa các tệp nên là xác định. Hạn chế này chưa bao giờ là một vấn đề trong thực tế.

### 4.6. Bỏ qua các bản ghi tệ
> - Đôi khi có nhiêu bug trong code của người dùng gây ra cho hàm Map hoặc Reduce gặp sự cố xác định trên một số bản ghi nhất định. Các lỗi như vậy ngăn cản hoạt động MapReduce hoàn thành. Hành động thông thường là sửa lỗi, nhưng đôi khi điều này không khả thi; có thể lỗi nằm trong một thư viện của bên thứ ba mà mã nguồn không có sẵn. Ngoài ra, đôi khi việc bỏ qua một vài bản ghi là chấp nhận được, ví dụ khi thực hiện phân tích thống kê trên một tập dữ liệu lớn. Tác giả cung cấp một chế độ thực thi tùy chọn trong đó thư viện MapReduce phát hiện các bản ghi gây ra sự cố xác định và bỏ qua các bản ghi này để tiếp tục tiến trình.
> - Mỗi quy trình worker cài đặt một trình xử lý tín hiệu bắt các lỗi phân đoạn và lỗi bus. Trước khi gọi một hàm Map hoặc Reduce của người dùng, thư viện MapReduce lưu số thứ tự của đối số trong một biến toàn cục. Nếu mã người dùng tạo ra một tín hiệu, trình xử lý tín hiệu gửi một gói UDP "hơi thở cuối cùng" chứa số thứ tự đến master MapReduce. Khi master đã thấy nhiều hơn một lỗi trên một bản ghi cụ thể, nó chỉ ra rằng bản ghi đó nên được bỏ qua khi nó phát hành lần thực thi lại tiếp theo của tác vụ Map hoặc Reduce tương ứng.

### 4.7. Thực thi cục bộ
> - Gỡ lỗi các vấn đề trong các hàm Map hoặc Reduce có thể khó khăn, vì tính toán thực tế xảy ra trong một hệ thống phân tán, thường trên hàng nghìn máy, với các quyết định phân công công việc được thực hiện động bởi master. Để giúp tạo điều kiện cho việc gỡ lỗi, lập hồ sơ và thử nghiệm quy mô nhỏ, tác giả đã phát triển một triển khai thay thế của thư viện MapReduce thực thi tuần tự tất cả công việc cho một hoạt động MapReduce trên máy cục bộ. Các điều khiển được cung cấp cho người dùng để tính toán có thể bị giới hạn ở các tác vụ map cụ thể. Người dùng gọi chương trình của họ với một cờ đặc biệt và sau đó có thể dễ dàng sử dụng bất kỳ công cụ gỡ lỗi hoặc thử nghiệm nào mà họ thấy hữu ích (ví dụ: gdb).

### 4.8. Thông tin trạng thái
> - Master chạy một máy chủ HTTP nội bộ và xuất một tập hợp các trang trạng thái cho ta sử dụng. Các trang trạng thái hiển thị tiến trình của tính toán, chẳng hạn như có bao nhiêu tác vụ đã hoàn thành, bao nhiêu đang tiến hành, số byte đầu vào, số byte dữ liệu trung gian, số byte đầu ra, tốc độ xử lý, v.v. Các trang cũng chứa các liên kết đến các tệp lỗi chuẩn và đầu ra chuẩn được tạo ra bởi mỗi tác vụ. Người dùng có thể sử dụng dữ liệu này để dự đoán thời gian tính toán sẽ mất bao lâu, và liệu có nên thêm tài nguyên vào tính toán hay không. Các trang này cũng có thể được sử dụng để xác định khi nào tính toán chậm hơn nhiều so với dự kiến.
> - Ngoài ra, trang trạng thái cấp cao nhất hiển thị các worker đã thất bại, và các tác vụ map và reduce mà họ đang xử lý khi họ thất bại. Thông tin này hữu ích khi cố gắng chẩn đoán lỗi trong mã người dùng.

### 4.9. Counter
> - Thư viện MapReduce cung cấp một cơ sở bộ đếm để đếm số lần xuất hiện của các sự kiện khác nhau. Ví dụ, mã người dùng có thể muốn đếm tổng số từ được xử lý hoặc số lượng tài liệu tiếng Đức được lập chỉ mục, v.v.
> - Để sử dụng cơ sở này, mã người dùng tạo một đối tượng bộ đếm có tên và sau đó tăng bộ đếm một cách thích hợp trong hàm Map và/hoặc Reduce. Ví dụ:
```cpp
Counter* uppercase;
uppercase = GetCounter("uppercase");

map(String name, String contents):
  for each word w in contents:
    if (IsCapitalized(w)):
      uppercase->Increment();
    EmitIntermediate(w, "1");
```
> - Các giá trị bộ đếm từ các máy worker riêng lẻ được truyền định kỳ đến master (được đính kèm vào phản hồi ping). Master tổng hợp các giá trị bộ đếm từ các tác vụ map và reduce thành công và trả lại chúng cho mã người dùng khi hoạt động MapReduce hoàn thành. Các giá trị bộ đếm hiện tại cũng được hiển thị trên trang trạng thái của master để con người có thể theo dõi tiến trình của tính toán trực tiếp. Khi tổng hợp các giá trị bộ đếm, master loại bỏ các tác động của các lần thực thi trùng lặp của cùng một tác vụ map hoặc reduce để tránh đếm hai lần. (Các lần thực thi trùng lặp có thể phát sinh từ việc sử dụng các tác vụ dự phòng và từ việc thực thi lại các tác vụ do lỗi).
> - Một số giá trị bộ đếm được duy trì tự động bởi thư viện MapReduce, chẳng hạn như số lượng cặp khóa/giá trị đầu vào được xử lý và số lượng cặp khóa/giá trị đầu ra được tạo ra.
> - Người dùng thấy cơ sở bộ đếm hữu ích để kiểm tra tính hợp lý của hành vi của các hoạt động MapReduce. Ví dụ, trong một số hoạt động MapReduce, mã người dùng có thể muốn đảm bảo rằng số lượng cặp đầu ra được tạo ra chính xác bằng với số lượng cặp đầu vào được xử lý, hoặc rằng tỷ lệ phần trăm tài liệu tiếng Đức được xử lý nằm trong một tỷ lệ chấp nhận được của tổng số tài liệu được xử lý.

## 5. Performance
> - Trong phần này, tác giả đo lường hiệu suất của MapReduce trên hai tính toán chạy trên một cụm máy lớn. Một tính toán tìm kiếm qua khoảng một terabyte dữ liệu để tìm một mẫu cụ thể. Tính toán còn lại sắp xếp khoảng một terabyte dữ liệu.
> - Hai chương trình này đại diện cho một tập hợp lớn các chương trình thực tế được viết bởi người dùng MapReduce - một lớp chương trình chuyển dữ liệu từ một biểu diễn này sang một biểu diễn khác, và một lớp khác trích xuất một lượng nhỏ dữ liệu thú vị từ một tập dữ liệu lớn.

### 5.1. Cấu hình Cluster
> - Tất cả các chương trình được thực thi trên một cụm gồm khoảng 1800 máy. Mỗi máy có hai bộ xử lý Intel Xeon 2GHz với Hyper-Threading được bật, 4GB bộ nhớ, hai đĩa IDE 160GB, và một liên kết Ethernet gigabit. Các máy chủ được sắp xếp trong một mạng chuyển mạch hình cây hai cấp với băng thông tổng hợp khoảng 100-200 Gbps tại gốc. Tất cả các máy đều ở cùng một cơ sở lưu trữ và do đó thời gian vòng lặp giữa bất kỳ cặp máy nào là dưới một mili giây.

### 5.2. Grep

![alt text](/image/figure-2-MapReduce(2004).png)
> - Chương trình grep quét qua 10 mũ 10 bản ghi 100 byte, tìm kiếm một mẫu ba ký tự tương đối hiếm (mẫu xuất hiện trong 92.337 bản ghi). Đầu vào được chia thành các phần khoảng 64MB (M = 15000), và toàn bộ đầu ra được đặt trong một tệp (R = 1).
> - Hình 2 cho thấy tiến trình của tính toán theo thời gian. Trục Y hiển thị tốc độ mà dữ liệu đầu vào được quét. Tốc độ dần dần tăng lên khi nhiều máy hơn được chỉ định cho tính toán MapReduce này, và đạt đỉnh trên 30 GB/s khi 1764 worker đã được chỉ định. Khi các tác vụ map hoàn thành, tốc độ bắt đầu giảm và đạt mức không khoảng 80 giây vào tính toán. Toàn bộ tính toán mất khoảng 150 giây từ khi bắt đầu đến khi kết thúc. Điều này bao gồm khoảng một phút chi phí khởi động. Chi phí này là do việc truyền chương trình đến tất cả các máy worker, và các độ trễ tương tác với GFS để mở tập hợp 1000 tệp đầu vào và lấy thông tin cần thiết cho tối ưu hóa địa phương.

### 5.3. Sort
![alt text](/image/figure-3-MapReduce(2004).png)
> - Chương trình Sort sắp xếp 10 mũ 10 bản ghi 100-byte. Chương trình là mô phỏng theo chuẩn TeraSort.
> - Chương trình sắp xếp bao gồm ít hơn 50 dòng code người dùng. Hàm Map gồm 3 dòng trích xuất một khóa sắp xếp 10-byte từ 1 dòng text và đưa ra khóa và dòng văn bản gốc dưới dạng cặp key/value. Tác giả sử dụng 1 hàm nhận diện tích hợp Reduce. Hàm này chuyển cặp key/value trung gian không thay đổi thành đầu ra cặp key/value. Đầu ra cuối cùng được ghi vào một tập hợp các tệp GFS sao chép hai chiều (tức là, 2 terabyte được ghi dưới dạng đầu ra của chương trình).
> - Như trước đây, dữ liệu đầu vào được chia thành các phần 64MB (M = 15000). Tác giả phân vùng đầu ra đã sắp xếp thành 4000 tệp (R = 4000). Hàm phân vùng sử dụng các byte ban đầu của khóa để phân chia nó thành một trong các phần R.
> - Hàm phân vùng cho chuẩn này có kiến thức tích hợp về phân phối các khóa. Trong một chương trình sắp xếp chung, tác giả sẽ thêm một hoạt động MapReduce trước đó để thu thập một mẫu của các khóa và sử dụng phân phối của các khóa mẫu để tính toán các điểm chia cho lần sắp xếp cuối cùng.
> - Hình 3(a) cho thấy tiến trình của 1 hoạt động bình của chương trình sắp xếp. Biểu đồ trên cùng bên trái hiển thị tốc độ mà đầu vào được đọc. Tốc độ đạt đỉnh khoảng 13 GB/s và giảm dần khá nhanh vì tất cả các tác vụ map hoàn thành trước khi 200 giây trôi qua. Lưu ý rằng tốc độ đầu vào thấp hơn cho grep. Điều này là do các tác vụ sắp xếp map tốn khoảng nửa thời gian của chúng và băng thông I/O để ghi đầu ra trung gian vào đĩa cục bộ. Đầu ra trung gian tương ứng cho grep có kích thước không đáng kể.
> - Biểu đồ giữa bên trái hiển thị tốc độ mà dữ liệu được gửi qua mạng từ tác vụ task đến tác vụ reduce. Việc chuyển tác vụ bắt đầu ngay khi tác vụ map đầu tiên được hoàn thành. Đỉnh đầu tiên trong biểu đồ là cho batch đầu tiên của khoảng 1700 tác vụ reduce (toàn bộ MapReduce được chỉ định khoảng 1700 máy, và mỗi máy thực hiện tối đa một tác vụ reduce tại một thời điểm). hoảng 300 giây vào tính toán, một số tác vụ reduce của lô đầu tiên hoàn thành và bắt đầu chuyển dữ liệu cho các tác vụ reduce còn lại. Tất cả việc chuyển dữ liệu hoàn thành khoảng 600 giây vào tính toán.
> - Biểu đồ dưới cùng bên trái hiển thị tốc độ mà dữ liệu đã sắp xếp được ghi vào các tệp đầu ra cuối cùng bởi các tác vụ reduce. Có một độ trễ giữa kết thúc giai đoạn chuyển dữ liệu đầu tiên và bắt đầu giai đoạn ghi vì các máy bận sắp xếp dữ liệu trung gian. Việc ghi tiếp tục với tốc độ khoảng 2-4 GB/s trong một thời gian. Tất cả các việc ghi hoàn thành khoảng 850 giây vào tính toán. Bao gồm chi phí khởi động, toàn bộ tính toán mất 891 giây. Điều này tương tự với kết quả tốt nhất hiện tại được báo cáo là 1057 giây cho chuẩn TeraSort.
> - Một vài điều cần lưu ý: tốc độ đầu vào cao hơn tốc độ chuyển dữ liệu và tốc độ đầu ra vì tối ưu hóa trên local - hầu hết dữ liệu được đọc từ đĩa cục bộ và bỏ qua mạng tương đối hạn chế về băng thông. Tốc độ chuyển dữ liệu cao hơn tốc độ đầu ra vì giai đoạn đầu ra ghi hai bản sao của dữ liệu đã sắp xếp (tác giả tạo hai bản sao của đầu ra vì lý do độ tin cậy và khả dụng). Tác giả ghi hai bản sao vì đó là cơ chế để đảm bảo độ tin cậy và khả dụng được cung cấp bởi hệ thống tệp cơ bản. Yêu cầu băng thông mạng để ghi dữ liệu sẽ giảm nếu hệ thống tệp cơ bản sử dụng mã hóa xóa thay vì sao chép.

### 5.4. Effect of Backup Tasks
> - Trong hình 3(b), thể hiện 1 hoạt động của chương trình sắp xếp với tác vụ backup đã tắt. Luồng thực thi là như nhau được thể hiện trong hình 3(a), trừ việc đó là 1 thời gian dài không có hoạt động ghi nào xảy ra. Sau 960s, tất cả trừ 5 tác vụ reduce đã hoàn thành. Mất 300s để những tác vụ cuối cùng hoàn thành. Toàn bộ tính toán mất 1283 giây, tăng 44% về thời gian trôi qua.

### 5.5. Machine Failures
> - Trong hình 3(c), thể hiện 1 hoạt động của chương trình sắp xếp khi tác giả đã tắt 200 trong 1796 worker trong vài phút. Hệ thống lập lịch cụm cơ bản ngay lập tức khởi động lại các quy trình worker mới trên các máy này (vì chỉ có các quy trình bị giết, các máy vẫn hoạt động bình thường).
> - Các lỗi worker xuất hiện dưới dạng tốc độ đầu vào âm vì một số công việc map đã hoàn thành trước đó biến mất (vì các worker map tương ứng đã bị giết) và cần phải được thực hiện lại. Việc thực hiện lại công việc map này diễn ra tương đối nhanh chóng. Toàn bộ tính toán hoàn thành trong 933 giây bao gồm chi phí khởi động (chỉ tăng 5% so với thời gian thực thi bình thường).

## 6. Kinh nghiệm
> - Tác giả viết chương trình đầu tiên của thư viện MapReduce vào tháng 2 năm 2003, và đã thực hiện cải tiến đáng kế vào tháng 8 năm 2003, bao gồm tối ưu local, cân bằng tải động của tác vụ thực thi trên các máy chủ worker,... Kể từ thời điểm đó, tác giả thấy rất ngạc nhiên về mức độ trải dài của thư viện MapReduce cho các vấn đề mà tác giả đang làm việc. Thư viện đã được sử dụng trải dài trong nhiều lĩnh vực trong google, bao gồm:
>> - Các vấn đề học máy quy mô lớn
>> - Các vấn đề phân cụm cho các sản phẩm Google News và Froogle
>> - Trích xuất dữ liệu lớn sử dụng để tạo ra các báo cái về truy vấn phổ biến (vd: Google Zeitgeist)
>> - Trích xuất của thông tin web cho những thí nghiệm và sản phẩm mới
>> - Các tính toán đồ thị quy mô lớn

![alt text](/image/figure-4-MapReduce(2004).png)

> - Hình 4 thể hiện sử tăng trưởng đáng kể về số lượng các chương trình MapReduce riêng biệt được kiểm tra vào hệ thống quản lý mã nguồn của tác gia theo thời gian, từ 0 vào đầu năm 2003 đến 900 trường hợp riêng biệt vào cuối tháng 11 năm 2004. MapReduce đã thành công bởi vì nó cho phép  có thể viết một chương đơn giản và chạy nó hiệu quả trên hàng nghìn máy chủ trong vòng nửa giờ, tăng tốc đáng kể chu kỳ phát triển và thử nghiệm. Hơn nữa, nó cho phép các lập trình viên không có kinh nghiệm với các hệ thống phân tán và/hoặc song song dễ dàng tận dụng lượng tài nguyên lớn.

### 6.1. Lập chỉ mục quy mô lớn
> - Một trong những ứng dụng quan trọng nhất của MapReduce cho đến nay là việc viết lại hoàn toàn hệ thống lập chỉ mục sản xuất tạo ra các cấu trúc dữ liệu được sử dụng cho dịch vụ tìm kiếm web của Google. Hệ thống lập chỉ mục nhận đầu vào là một tập hợp lớn các tài liệu đã được hệ thống thu thập lấy về, được lưu trữ dưới dạng một tập hợp các tệp GFS. Nội dung thô của các tài liệu này là hơn 20 terabyte dữ liệu. Quá trình lập chỉ mục chạy dưới dạng một chuỗi từ năm đến mười hoạt động MapReduce. Sử dụng MapReduce (thay vì các bước phân tán ad-hoc trong phiên bản trước của hệ thống lập chỉ mục) đã mang lại một số lợi ích:
>> - Mã lập chỉ mục đơn giản hơn, nhỏ hơn và dễ hiểu hơn, vì mã xử lý khả năng chịu lỗi, phân phối và song song được ẩn trong thư viện MapReduce. Ví dụ, kích thước của một giai đoạn tính toán giảm từ khoảng 3800 dòng mã C++ xuống còn khoảng 700 dòng khi được biểu diễn bằng MapReduce.
>> - Hiệu suất của thư viện MapReduce đủ tốt để có thể giữ các tính toán không liên quan về mặt khái niệm riêng biệt, thay vì trộn chúng lại với nhau để tránh các bước bổ sung qua dữ liệu. Điều này làm cho việc thay đổi quá trình lập chỉ mục trở nên dễ dàng. Ví dụ, một thay đổi mất vài tháng để thực hiện trong hệ thống lập chỉ mục cũ chỉ mất vài ngày để triển khai trong hệ thống mới.
>> - Quá trình lập chỉ mục trở nên dễ dàng hơn nhiều để vận hành, vì hầu hết các vấn đề gây ra bởi lỗi máy, máy chậm và sự cố mạng được xử lý tự động bởi thư viện MapReduce mà không cần can thiệp của người vận hành. Hơn nữa, việc cải thiện hiệu suất của quá trình lập chỉ mục bằng cách thêm các máy mới vào cụm lập chỉ mục trở nên dễ dàng.

### 7. Related Word
### 8. Kết luận
> - Mô hình lập trình MapReduce đã được sử dụng thành công tại Google cho nhiều mục đích khác nhau. Tác giả cho rằng sự thành công này đến từ một số lý do. Thứ nhất, mô hình này dễ sử dụng, ngay cả đối với các lập trình viên không có kinh nghiệm với các hệ thống phân tán và song song, vì nó ẩn đi các chi tiết về song song hóa, khả năng chịu lỗi, tối ưu hóa địa phương và cân bằng tải. Thứ hai, một loạt các vấn đề có thể dễ dàng được biểu diễn dưới dạng các tính toán MapReduce. Ví dụ, MapReduce được sử dụng để tạo ra dữ liệu cho dịch vụ tìm kiếm web của Google, để sắp xếp, khai thác dữ liệu, học máy và nhiều hệ thống khác. Thứ ba, tác giả và cộng sự đã phát triển một triển khai của MapReduce có thể mở rộng đến các cụm máy lớn bao gồm hàng nghìn máy. Triển khai này sử dụng hiệu quả các tài nguyên máy và do đó phù hợp để sử dụng cho nhiều vấn đề tính toán lớn gặp phải tại Google.
> - Tác giả đã học được một số điều từ công việc này. Thứ nhất, việc hạn chế mô hình lập trình làm cho việc song song hóa và phân phối các tính toán trở nên dễ dàng và làm cho các tính toán này chịu lỗi. Thứ hai, băng thông mạng là một tài nguyên khan hiếm. Do đó, một số tối ưu hóa trong hệ thống nhằm giảm lượng dữ liệu được gửi qua mạng: tối ưu hóa địa phương cho phép đọc dữ liệu từ các đĩa cục bộ và việc ghi một bản sao duy nhất của dữ liệu trung gian vào đĩa cục bộ tiết kiệm băng thông mạng. Thứ ba, thực thi dư thừa có thể được sử dụng để giảm tác động của các máy chậm và để xử lý các lỗi máy và mất dữ liệu.