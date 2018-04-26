## Cần biết: 
Trước tiên phải hiểu là MySQL Replication không phải là giải pháp giải quyết mọi bài toán về quá tải hệ thống cơ sở dữ liệu. 

Để mở rộng một hệ thống ta có hai phương pháp mở rộng là scale up và scale out. 

Ban đầu: tưởng tượng chỉ có một máy chủ: thì hai phương pháp trên là:

* Scale up là với một máy chủ thì làm cách nào đó tối ưu, tăng hiệu năng để nó có thể phục vụ nhiều hơn số lượng kết nối, truy vấn.

* Scale out là giải pháp tăng số lượng server và dùng các giải pháp load-balacer để phân phối truy vấn ra nhiều server. Như là: Ban đầu có 1 server có khả năng phục vụ 500 truy vấn. Nếu dựng thêm 5 server nữa có cấu hình tương tự, đặt thêm một Load Balancing phía trước để phân phối thì có khả năng hệ thống có thể phục vụ đc 5x500 truy vấn đồng thời.

=> MySQL Replication là một giải pháp scale out (tăng số lượng instance MySQL) nhưng không phải bài toán nào cũng dùng được. Các bài toán mà MySQL Replication sẽ giải quyết tốt:

* Scale Read
* Data Report
* Real time backup

## Một số khái niệm:

### Scale Real: 
* Scale Read thường gặp ở các ứng dụng mà số truy vấn đọc dữ liệu nhiều hơn ghi, tỉ lệ read/write có thể 80/20 hoặc hơn. Các ứng dụng thường gặp là báo, trang tin tức.
* Với scale read ta sẽ chỉ có một Master instance phục vụ cho việc đọc/ghi dữ liệu. Có thể có một hoặc nhiều Slave instance chỉ phục vụ cho việc đọc dữ liệu
* Một số ứng dụng write nhiều (thương mại điện tử) cũng có sử dụng MySQL Replication để scale out hệ thống

### Data Report
* Một số hệ thống cho phép một số người (leader, manager, người làm report, thống kê, data) truy cập vào dữ liệu để phục vụ cho công việc của họ. Nhưng việc chọc thẳng vào dữ liệu database sẽ rất nguy hiểm vì:

        Vô tình chỉnh sửa làm sai lệnh dữ liệu (nếu có quyền insert, update)
        Vô tình thực thi các câu truy vấn tốn nhiều tài nguyên, thời gian truy vấn dài làm treo hệ thống

* Việc setup một máy chủ làm data report (application cũng sẽ không kết nối tới server này) làm giảm thiểu 2 rủi ro trên.

###  Real time backup

* Với cơ sở dữ liệu lớn việc backup không thể thực hiện thường xuyên được 
* Real time backup là một giải pháp bổ sung cho offline backup, chạy đồng thời cả 2 phương pháp này để bảo đảm sự an toàn cho dữ liệu.

## Mô hình hoạt động

  Mô hình 1:
  
  Mô hình 2:
  
  Nhận xet: cả hai mô hình chỉ có con master database phục vụ Write dữ liệu. Con slave để Read dữ liệu và có thể bố trí từng con Web backend read đến từng con Slave hay cho một con Haproxy-LB trước từng con DB để phân phối kết nối vào từng con DB theo thuật toán đang dùng của Haproxy

## Cách hoạt động:

Hiểu cơ bản: 
        
* Tất cả các thay đổi trên cơ sở dữ liệu master sẽ được ghi lại dưới dạng file log binary, slave đọc file log đó, thực hiện những thao tác trong file log, việc ghi, đọc và thực thi trong file log này dưới dạng binary được thực hiện rất nhanh.

* Replication dựa trên các con master lưu giữ theo dõi tất cả những thay đổi cơ sở dữ liệu của nó (cập nhật, xóa, vv) trong bản ghi nhị phân của nó. Các bản ghi nhị phân như là các record của tất cả các sự kiện làm thay đổi cấu trúc cơ sở dữ liệu hoặc nội dung (dữ liệu) từ thời điểm các máy chủ đã bắt đầu thực thi. 

P/S: Thông thường, câu SELECT không được ghi lại bởi vì chúng không phải thay đổi cấu trúc cũng như nội dung của cơ sở dữ liệu.

* Mỗi slave kết nối đến các master yêu cầu một bản sao của bản ghi nhị phân. Đó là, nó kéo các dữ liệu từ các master, chứ không phải là master đẩy dữ liệu đến các slave. Các slave cũng thực hiện các sự kiện từ các bản ghi nhị phân mà nó nhận được => Bảng được tạo ra hoặc cấu trúc thay đổi và dữ liệu đã chèn hay đã xóa và kể cả cập nhật thì đều giống hệt theo những thay đổi mà ban đầu đã được thực hiện trên master

## Chi tiết hoạt động:

### Trên Master:
 Mô hình thực hiện:
 
 Db3.
 
* B1: Các kết nối từ web app tới Master DB sẽ mở một Session_Thread khi có nhu cầu ghi dữ liệu. Session_Thread sẽ ghi các statement trạng thái SQL vào một file binlog (ví dụ với format của binlog là statement-based hoặc mix). Binlog được lưu trữ trong data_dir (cấu hình my.cnf) và có thể được cấu hình các thông số như kích thước tối đa bao nhiêu, lưu lại trên server bao nhiêu ngày.

* B2: Master DB sẽ mở một Dump_Thread và gửi binlog tới cho I/O_Thread mỗi khi I/O_Thread từ Slave DB yêu cầu dữ liệu

### Trên Slave:

* B3: Trên mỗi Slave DB sẽ mở một I/O_Thread kết nối tới Master DB thông qua network, giao thức TCP (với MySQL 5.5 replication chỉ hỗ trợ Single_Thread nên mỗi Slave DB sẽ chỉ mở duy nhất một kết nối tới Master DB, các phiên bản sau 5.6, 5.7 hỗ trợ mở đồng thời nhiều kết nối hơn) để yêu cầu binlog.

* Sau khi Dump_Thread gửi binlog tới I/O_Thead, I/O_Thread sẽ có nhiệm vụ đọc binlog này và ghi vào relaylog. Đồng thời trên Slave sẽ mở một SQL_Thread, SQL_Thread có nhiệm vụ đọc các event từ relaylog và apply các event đó vào Slave => quá trình replication hoàn thành.
