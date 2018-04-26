## Cần biết: 
Trước tiên phải hiểu là MySQL Replication không phải là giải pháp giải quyết mọi bài toán về quá tải hệ thống cơ sở dữ liệu. 
Để mở rộng một hệ thống ta có hai phương pháp mở rộng là scale up và scale out. 
Ban đầu: tưởng tượng chỉ có một máy chủ: thì hai phương pháp trên là:
Scale up là với một máy chủ thì làm cách nào đó tối ưu, tăng hiệu năng để nó có thể phục vụ nhiều hơn số lượng kết nối, truy vấn.
Scale out là giải pháp tăng số lượng server và dùng các giải pháp load-balacer để phân phối truy vấn ra nhiều server. Như là: Ban đầu có 1 server có khả năng phục vụ 500 truy vấn. Nếu dựng thêm 5 server nữa có cấu hình tương tự, đặt thêm một Load Balancing phía trước để phân phối thì có khả năng hệ thống có thể phục vụ đc 5x500 truy vấn đồng thời.
=> MySQL Replication là một giải pháp scale out (tăng số lượng instance MySQL) nhưng không phải bài toán nào cũng dùng được. Các bài toán mà MySQL Replication sẽ giải quyết tốt:
* Scale Read
* Data Report
* Real time backup
Một số khái niệm:

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

  
