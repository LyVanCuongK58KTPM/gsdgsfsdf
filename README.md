Tiếp tục mạch tài liệu của bài tập lớn, dưới đây là hướng dẫn chi tiết, kịch bản T-SQL và các bước thực hiện cho phân hệ Nhập dữ liệu, Truy vấn, Kiểm tra phân mảnh và Kiểm tra đồng bộ.Nội dung này được thiết kế theo đúng quy chuẩn báo cáo, chuẩn hóa theo các tên bảng, thực thể vật lý mà bạn đã cấu hình thành công ở các bước trước:CHƯƠNG 5: HƯỚNG DẪN VẬN HÀNH VÀ KIỂM TRA HỆ THỐNG PHÂN TÁN5.1. Quy trình Nhập dữ liệu (Data Insertion)Để đảm bảo tính toàn vẹn dữ liệu và đúng kiến trúc phân tán, quy trình nhập liệu được phân chia nghiêm ngặt theo phân cấp quản lý:5.1.1. Nhập liệu danh mục tập trung (Thực hiện tại Server TNU)Các dữ liệu mang tính chất danh mục khung, quy định toàn vùng hoặc quản lý sĩ số trần bắt buộc phải được khởi tạo từ máy chủ gốc trung tâm.
```SQL
USE DANGKY_TINCHI_TNU;
GO

-- Nhập một môn học mới vào danh mục khung toàn vùng
INSERT INTO MON_HOC (MaMon, TenMon, SoTinChi) 
VALUES ('MH05', N'An toàn bảo mật hệ thống phân tán', 3);

-- Nhập một lớp học phần mới và gán sĩ số trần ban đầu
INSERT INTO LOPHOC_PHAN (MaLHP, MaMon, SoLuongMax, SoLuongConLai, MaGV)
VALUES ('LHP05', 'MH05', 50, 50, 'GV_TNUT01');
GO
```
5.1.2. Nhập liệu tự trị địa phương (Thực hiện tại Server Chi nhánh TNUT / ICTU)Các dữ liệu liên quan đến phân hoạch phòng học vật lý tại địa phương hoặc phiếu đăng ký của sinh viên sẽ được nhập trực tiếp tại các site con để tối ưu tốc độ và đáp ứng tính năng ngoại tuyến.SQL-- CHẠY TẠI MÁY TNUT: Phòng Đào tạo TNUT tự xếp phòng học cho lớp học phần
```
USE QL_DANGKY_TINCHI_TNUT;
GO
INSERT INTO LICH_DAY_GIANG_VIEN (MaLHP, MaGV, PhongHoc, NgayBatDau, CaDuyet)
VALUES ('LHP05', 'GV_TNUT01', 'Phòng 401 - Nhà C1 TNUT', '2026-09-10', 2);
GO
```
5.2. Quy trình Truy vấn dữ liệu phân tán (Distributed Query)Hệ thống sử dụng các cổng truyền thông kết nối từ xa kết hợp phân hệ View và Stored Procedure để người dùng cuối (Sinh viên/Giáo vụ) có thể tra cứu thông tin toàn cục mà không cần quan tâm đến vị trí đặt dữ liệu vật lý.5.2.1. Tra cứu danh sách môn học và lớp học phần thời gian thựcSinh viên tại chi nhánh TNUT thực hiện mở giao diện đăng ký, ứng dụng sẽ gọi phân hệ View phân tán để lấy dữ liệu từ xa:SQLUSE QL_DANGKY_TINCHI_TNUT;
GO
-- Gọi View phân tán kết nối Linked Server lên máy mẹ
```
SELECT MaLHP, TenMon, SoTinChi, SoLuongConLai 
FROM v_DanhSachLopHocPhan 
WHERE MaMon = 'MH05';
GO
```
5.2.2. Gọi thủ tục kết xuất lịch dạy tích hợp của giảng viênHệ thống tự động Join dữ liệu tên giảng viên lưu tại máy mẹ TNU và lịch phòng học lưu tại máy con TNUT thông qua Stored Procedure:
```SQL
USE QL_DANGKY_TINCHI_TNUT;
GO
-- Xem lịch công tác của giảng viên GV_TNUT01
EXEC sp_TraCuuLichDayGiangVien @MaGV = 'GV_TNUT01';
GO
```
5.3. Kiểm tra phân mảnh dữ liệu (Data Fragmentation Verification)Mục này nhằm chứng minh dữ liệu đã được phân hoạch chính xác về mặt vật lý lên các ổ đĩa của từng Server trong mạng ảo ZeroTier theo đúng thiết kế ban đầu.5.3.1. Kiểm tra tính độc lập của Phân mảnh ngang nguyên thủy (SINH_VIEN)Mục tiêu: Chứng minh hồ sơ sinh viên trường nào thì nằm duy nhất tại ổ đĩa của trường đó để tăng tính bảo mật và tối ưu dung lượng lưu trữ cục bộ.Câu lệnh kiểm tra: Thực hiện chạy lệnh SELECT kiểm tra tại chỗ trên hai site chi nhánh:SQL-- 
```
CHẠY TẠI MÁY CHI NHÁNH TNUT
USE QL_DANGKY_TINCHI_TNUT;
GO
SELECT * FROM SINH_VIEN WHERE MaTruong = 'TNUT'; -- KẾT QUẢ: Hiện đầy đủ SV của TNUT
SELECT * FROM SINH_VIEN WHERE MaTruong = 'ICTU'; -- KẾT QUẢ: Trống trơn (0 dòng)

-- CHẠY TẠI MÁY CHI NHÁNH ICTU
USE QL_DANGKY_TINCHI_ICTU;
GO
SELECT * FROM SINH_VIEN WHERE MaTruong = 'ICTU'; -- KẾT QUẢ: Hiện đầy đủ SV của ICTU
SELECT * FROM SINH_VIEN WHERE MaTruong = 'TNUT'; -- KẾT QUẢ: Trống trơn (0 dòng)
```
5.3.2. Kiểm tra phân mảnh dữ liệu tự trị (LICH_DAY_GIANG_VIEN)Chứng minh quyền xếp phòng học hoàn toàn thuộc về địa phương, máy chủ trung tâm không can thiệp vật lý:Thực hiện chạy lệnh 
```
SELECT * FROM LICH_DAY_GIANG_VIEN
```
 trên máy mẹ TNU $\rightarrow$ Hệ thống sẽ báo lỗi hoặc không tìm thấy bảng, vì bảng này chỉ tồn tại vật lý tại ổ đĩa local của máy con TNUT và ICTU.5.4. Kiểm tra trạng thái đồng bộ (Replication Synchronization Verification)Phần này hướng dẫn cách giám sát và cưỡng bức hệ thống hợp nhất dữ liệu hai chiều (Merge Replication) để kiểm tra luồng thông mạch mạng.5.4.1. Giám sát thông qua công cụ đồ họa trực quan (Replication Monitor)Tại máy chủ trung tâm TNU, click chuột phải vào thư mục Replication $\rightarrow$ Chọn Launch Replication Monitor.Tại cây thư mục bên trái, click chọn vào tên Ấn bản Pub_Merge_DangKyTinChi.Quan sát tab All Subscriptions ở khung bên phải:Cột Status: Phải hiển thị biểu tượng màu xanh lá cây hoặc chữ Synchronized (Đã đồng bộ thành công).Cột Connection: Hiển thị trạng thái kết nối thông qua tài khoản sa mạng ảo.5.4.2. Cưỡng bức đồng bộ thủ công để kiểm tra tức thìKhi sinh viên bấm đăng ký ngoại tuyến trong lúc đứt mạng, ngay sau khi kết nối lại mạng ảo ZeroTier, ta thực hiện ép hệ thống đồng bộ ngay lập tức để nghiệm thu dữ liệu:Tại giao diện Replication Monitor, chuột phải vào dòng trạng thái của chi nhánh TNUT $\rightarrow$ Chọn Start Synchronizing.Hệ thống sẽ mở một cửa sổ tiến trình chạy ngầm. Quản trị viên tiến hành đọc dòng thông báo tại khung Last status message:[100%] Merge Agent đang thực hiện đối chiếu bản ghi... Upload 1 changes to Publisher... Download 0 changes to Subscriber... Quá trình hợp nhất hoàn tất thành công.Ngay sau khi dòng chữ trên xuất hiện, thực hiện truy vấn tại máy mẹ TNU để kiểm tra xem phiếu đăng ký của sinh viên chi nhánh đã cập bến an toàn chưa:SQL
 ```
 USE DANGKY_TINCHI_TNU;
GO
-- Kiểm tra phiếu đăng ký đã được gom về trung tâm
SELECT * FROM DANG_KY;
GO
```
