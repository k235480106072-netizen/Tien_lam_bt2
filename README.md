# BÀI KIỂM TRA SỐ 2 – HỆ QUẢN TRỊ CSDL SQL SERVER

**Họ và tên:** Vi Trần Tiến

**Mã sinh viên:** K235480106072

**Lớp:** K59KMT.K01

**Đề tài:** Quản lý nhân sự


---



## TỔNG QUAN ĐỀ TÀI



### Mục tiêu
Xây dựng hệ thống **Quản lý nhân sự** trên nền tảng SQL Server, cho phép quản lý thông tin phòng ban, nhân viên và bảng lương hằng tháng. Đề tài minh họa việc áp dụng các kỹ thuật nâng cao của SQL Server bao gồm: **User-Defined Function**, **Stored Procedure**, **Trigger** và **Cursor**.



### Phạm vi và nghiệp vụ chính
- **Quản lý phòng ban**: Lưu trữ danh sách phòng ban, tự động đếm số lượng nhân viên thuộc mỗi phòng.

- **Quản lý nhân viên**: Thêm, sửa, xóa nhân viên; gắn nhân viên với phòng ban; tính tuổi nhân viên.

- **Quản lý lương**: Ghi nhận lương tháng, tính tổng quỹ lương, tính thuế thu nhập lũy tiến.

- **Báo cáo**: Xuất báo cáo nhân sự tổng hợp (phòng ban – nhân viên – lương) và in thông báo lương cá nhân.



### Phương pháp thực hiện
1. **Thiết kế CSDL** – Xác định các thực thể (PhongBan, NhanVien, LuongThang), thiết lập quan hệ PK/FK và ràng buộc CHECK.

2. **Viết Function** – Tạo 3 loại hàm do người dùng định nghĩa (Scalar, Inline Table-valued, Multi-statement Table-valued) phục vụ tính toán nghiệp vụ.

3. **Viết Stored Procedure** – Đóng gói logic nghiệp vụ (thêm nhân viên, tính tổng quỹ lương, báo cáo) vào các thủ tục để tái sử dụng và bảo mật.

4. **Viết Trigger** – Tự động hóa cập nhật sĩ số phòng ban khi có thay đổi dữ liệu nhân viên; phân tích hiện tượng đệ quy.

5. **Sử dụng Cursor** – Duyệt từng dòng dữ liệu để in thông báo cá nhân hóa; so sánh hiệu năng với cách tiếp cận Set-based.



### Công nghệ sử dụng
| Thành phần | Công nghệ |
|------------|-----------|
| Hệ quản trị CSDL | Microsoft SQL Server |
| Công cụ phát triển | SQL Server Management Studio (SSMS) |
| Ngôn ngữ | T-SQL (Transact-SQL) |

---



## PHẦN 1: THIẾT KẾ VÀ KHỞI TẠO CẤU TRÚC DỮ LIỆU



### 1.1 Logic thiết kế
| Thực thể | Mô tả | Quan hệ |
|----------|-------|---------|
| **[PhongBan]** | Thông tin phòng ban (mã, tên, số lượng nhân viên) | 1 📶 * Nhiều **[NhanVien]** |
| **[NhanVien]** | Thông tin cá nhân, phòng ban, ngày sinh | Nhiều 📶 * 1 **[PhongBan]**, 1 📶 * Nhiều **[LuongThang]** |
| **[LuongThang]** | Bảng lương tháng của mỗi nhân viên | Nhiều 📶 * 1 **[NhanVien]** |

*PK = Primary Key, FK = Foreign Key, Check = ràng buộc kiểm tra giá trị.*



### 1.2 Mã SQL khởi tạo
```sql
/* Tạo database */
CREATE DATABASE [QuanLyNhanSu_K235480106072];
GO
USE [QuanLyNhanSu_K235480106072];

```
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/9b489776-ee2c-46d2-95be-f578aad87726" />
Tạo cơ sở dữ liệu



```sql
/* Bảng PhongBan */
CREATE TABLE [PhongBan] (
    [MaPhong]          INT          PRIMARY KEY,
    [TenPhong]         NVARCHAR(100) NOT NULL,
    [SoLuongNhanVien]  INT           DEFAULT (0)   -- sẽ được Trigger cập nhật
);
GO
```
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/376dd666-2c12-4c42-88f9-f19321e994a4" />
Bảng PhongBan




/* Bảng NhanVien */

```sql
CREATE TABLE [NhanVien] (
    [MaNV]            INT           PRIMARY KEY IDENTITY(1,1),
    [HoTen]           NVARCHAR(100) NOT NULL,
    [NgaySinh]        DATE          NULL,
    [MaPhong]         INT           NOT NULL,
    CONSTRAINT [FK_NhanVien_PhongBan] 
        FOREIGN KEY ([MaPhong]) REFERENCES [PhongBan]([MaPhong]),
    CONSTRAINT [CK_NgaySinh] 
        CHECK ([NgaySinh] < GETDATE())               -- Ngày sinh phải < ngày hiện tại
);
GO
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/a89cf659-5b78-4bd7-a202-793ee90f912e" />

Bảng NhanVien


/* Bảng LuongThang */


```sql
CREATE TABLE [LuongThang] (
    [MaLuong]         INT           PRIMARY KEY IDENTITY(1,1),
    [MaNV]            INT           NOT NULL,
    [Thang]           DATE          NOT NULL,
    [Luong]           MONEY         NOT NULL,
    CONSTRAINT [FK_Luong_NhanVien] 
        FOREIGN KEY ([MaNV]) REFERENCES [NhanVien]([MaNV]),
    CONSTRAINT [CK_Luong_Positive] 
        CHECK ([Luong] > 0)                         -- lương phải > 0
);
GO
```


<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/2cd6602c-d782-4b1f-bd7b-3ff99f83cf39" />


Bảng LuongThang


## Chèn dữ liệu mẫu

```sql
-- Insert sample data into PhongBan
INSERT INTO dbo.[PhongBan] ([MaPhong], [TenPhong]) VALUES
-- Thêm 40 phòng ban mẫu
(1, N'Phòng Kỹ Thuật'),
(2, N'Phòng Kế Toán'),
(3, N'Phòng Nhân Sự'),
(4, N'Phòng Marketing'),
(5, N'Phòng Bán Hàng'),
(6, N'Phòng Hành Chính'),
(7, N'Phòng Nghiên Cứu'),
(8, N'Phòng Dịch Vụ'),
(9, N'Phòng Thiết Kế'),
(10, N'Phòng Phát Triển'),
(11, N'Phòng Vận Hành'),
(12, N'Phòng Quản Lý Dự Án'),
(13, N'Phòng Kiểm Định'),
(14, N'Phòng Đào Tạo'),
(15, N'Phòng An Ninh'),
(16, N'Phòng Y Tế'),
(17, N'Phòng Môi Trường'),
(18, N'Phòng Thể Dục'),
(19, N'Phòng Văn Hóa'),
(20, N'Phòng Thông Tin'),
(21, N'Phòng Pháp Lý'),
(22, N'Phòng Đấu Thầu'),
(23, N'Phòng Năng Lượng'),
(24, N'Phòng Công Nghệ'),
(25, N'Phòng Quản Trị'),
(26, N'Phòng Bảo Trì'),
(27, N'Phòng Vật Tư'),
(28, N'Phòng Kế Hoạch'),
(29, N'Phòng Tài Chính'),
(30, N'Phòng Đầu Tư'),
(31, N'Phòng Quỹ'),
(32, N'Phòng Dự Báo'),
(33, N'Phòng Thư Kiến'),
(34, N'Phòng Mua Sắm'),
(35, N'Phòng Thẩm Định'),
(36, N'Phòng Hậu Cần'),
(37, N'Phòng Quản Lý Rủi Ro'),
(38, N'Phòng Thông Tin Kỹ Thuật'),
(39, N'Phòng Phản Hồi'),
(40, N'Phòng Khác');

-- Insert sample data into NhanVien (40 nhân viên)
INSERT INTO dbo.[NhanVien] ([HoTen], [NgaySinh], [MaPhong]) VALUES
(N'Nguyen Van A', '1990-01-15', 1),
(N'Le Thi B', '1992-05-20', 2),
(N'Tran Van C', '1988-09-30', 3),
(N'Pham Thi D', '1991-03-12', 4),
(N'Hoang Van E', '1993-07-08', 5),
(N'Vu Thi F', '1989-11-25', 6),
(N'Dinh Van G', '1994-02-14', 7),
(N'Nguyen Thi H', '1995-06-19', 8),
(N'Le Van I', '1990-09-05', 9),
(N'Tran Thi J', '1992-12-22', 10),
(N'Pham Van K', '1991-04-03', 11),
(N'Hoang Thi L', '1993-08-17', 12),
(N'Vu Van M', '1994-10-30', 13),
(N'Dinh Thi N', '1996-01-11', 14),
(N'Nguyen Van O', '1990-02-28', 15),
(N'Le Van P', '1992-05-06', 16),
(N'Tran Thi Q', '1995-07-23', 17),
(N'Pham Van R', '1991-09-14', 18),
(N'Hoang Thi S', '1993-11-27', 19),
(N'Vu Van T', '1994-03-05', 20),
(N'Dinh Van U', '1990-06-18', 21),
(N'Nguyen Thi V', '1992-08-31', 22),
(N'Le Van W', '1995-10-12', 23),
(N'Tran Van X', '1991-12-25', 24),
(N'Pham Thi Y', '1993-02-07', 25),
(N'Hoang Van Z', '1994-04-20', 26),
(N'Vu Thi AA', '1990-07-02', 27),
(N'Dinh Van BB', '1992-09-15', 28),
(N'Nguyen Van CC', '1995-11-28', 29),
(N'Le Thi DD', '1991-01-10', 30),
(N'Tran Van EE', '1993-03-23', 31),
(N'Pham Van FF', '1994-05-05', 32),
(N'Hoang Thi GG', '1990-08-18', 33),
(N'Vu Van HH', '1992-10-31', 34),
(N'Dinh Thi II', '1995-12-13', 35),
(N'Nguyen Van JJ', '1991-02-26', 36),
(N'Le Van KK', '1993-04-10', 37),
(N'Tran Thi LL', '1994-06-22', 38),
(N'Pham Van MM', '1990-09-04', 39),
(N'Hoang Van NN', '1992-11-18', 40);

-- Insert sample data into LuongThang (40 bản ghi lương)
INSERT INTO dbo.[LuongThang] ([MaNV], [Thang], [Luong]) VALUES
-- Cập nhật lương tháng 5/2026 cho 40 nhân viên
(1, '2026-05-01', 10000000),
(2, '2026-05-01', 12000000),
(3, '2026-05-01', 11000000),
(4, '2026-05-01', 10500000),
(5, '2026-05-01', 11500000),
(6, '2026-05-01', 9500000),
(7, '2026-05-01', 10800000),
(8, '2026-05-01', 11200000),
(9, '2026-05-01', 10300000),
(10, '2026-05-01', 11900000),
(11, '2026-05-01', 10100000),
(12, '2026-05-01', 10700000),
(13, '2026-05-01', 11100000),
(14, '2026-05-01', 10600000),
(15, '2026-05-01', 10400000),
(16, '2026-05-01', 11700000),
(17, '2026-05-01', 10200000),
(18, '2026-05-01', 10900000),
(19, '2026-05-01', 11300000),
(20, '2026-05-01', 11800000),
(21, '2026-05-01', 10050000),
(22, '2026-05-01', 11550000),
(23, '2026-05-01', 10850000),
(24, '2026-05-01', 11250000),
(25, '2026-05-01', 10450000),
(26, '2026-05-01', 10650000),
(27, '2026-05-01', 10950000),
(28, '2026-05-01', 11150000),
(29, '2026-05-01', 10350000),
(30, '2026-05-01', 10750000),
(31, '2026-05-01', 11050000),
(32, '2026-05-01', 11280000),
(33, '2026-05-01', 11520000),
(34, '2026-05-01', 11760000),
(35, '2026-05-01', 10120000),
(36, '2026-05-01', 10380000),
(37, '2026-05-01', 10640000),
(38, '2026-05-01', 10900000),
(39, '2026-05-01', 11160000),
(40, '2026-05-01', 11420000);
GO
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/e87770f2-eddf-420c-8aa8-049c03d19a90" />
Chèn dữ liệu vào các bảng

## PHẦN 2: FUNCTON (Hàm)



### 2.1 Built‑in Functions (các hàm có sẵn)
# Trả lời: Các loại Built-in Function trong SQL Server

Trong SQL Server có rất nhiều **Built-in Function (hàm có sẵn)** giúp xử lý dữ liệu nhanh chóng mà không cần tự viết lại logic.  
Các hàm này được chia thành các nhóm chính như sau:

---

## 🔹 1. Hàm xử lý chuỗi (String Functions)

**Chức năng:** Xử lý dữ liệu dạng chuỗi ký tự  


## 🔹 2. Hàm số học (Mathematical Functions)

**Chức năng:** Thực hiện các phép toán số  


## 🔹 3. Hàm ngày giờ (Date and Time Functions)

**Chức năng:** Xử lý dữ liệu ngày tháng  

## 🔹 4. Hàm chuyển đổi kiểu (Conversion Functions)

**Chức năng:** Chuyển đổi giữa các kiểu dữ liệu  

## 🔹 5. Hàm tổng hợp (Aggregate Functions)

**Chức năng:** Tính toán trên nhiều dòng dữ liệu  


## 🔹 6. Hàm logic (Logical Functions)

**Chức năng:** Xử lý điều kiện  

## 🔹 7. Hàm hệ thống (System Functions)

**Chức năng:** Trả về thông tin hệ thống

## ✅ Kết luận
Các Built-in Function trong SQL Server giúp:
- Xử lý dữ liệu nhanh chóng  
- Giảm độ phức tạp khi viết truy vấn  
- Tăng hiệu suất và tính linh hoạt của hệ thống
- 
| Hàm | Mô tả | Ví dụ |
|-----|-------|-------|
| **GETDATE()** | Trả về ngày‑giờ hiện tại của server | `SELECT GETDATE();` |
| **LEN(string)** | Độ dài chuỗi ký tự | `SELECT LEN(N'Hello');` |
| **ROUND(number, d)** | Làm tròn số tới *d* chữ số thập phân | `SELECT ROUND(123.4567,2);` |


<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/d538e099-3b8f-4ec3-9eb0-ce81d673cf95" />

Trả về ngày‑giờ hiện tại của server

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/71b62659-b5c8-48a3-9a31-653937eb633c" />


Độ dài chuỗi ký tự

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/6603bcd4-bfe9-4ba9-a002-ed76a0f83f5d" />

Làm tròn số tới *d* chữ số thập phân

### 2.2 User‑Defined Functions
#### 2.2.1 Scalar Function – Tính tuổi nhân viên

**Mục đích:** Nhận vào ngày sinh của nhân viên, trả về số tuổi chính xác (kiểu `INT`). Hàm sử dụng `DATEDIFF(YEAR,…)` kết hợp điều chỉnh trường hợp chưa qua sinh nhật trong năm hiện tại (so sánh tháng và ngày) để đảm bảo kết quả đúng tuyệt đối.

| Tham số | Kiểu | Mô tả |
|---------|------|-------|
| `@NgaySinh` | `DATE` | Ngày sinh của nhân viên cần tính tuổi |
| **Trả về** | `INT` | Số tuổi tính đến thời điểm hiện tại |

```sql
CREATE FUNCTION dbo.fn_TinhTuoi(@NgaySinh DATE)
RETURNS INT
AS
BEGIN
    RETURN DATEDIFF(YEAR, @NgaySinh, GETDATE())
           - CASE 
                WHEN MONTH(@NgaySinh) > MONTH(GETDATE())
                  OR (MONTH(@NgaySinh) = MONTH(GETDATE())
                      AND DAY(@NgaySinh) > DAY(GETDATE())) THEN 1 
                ELSE 0 
             END;
END;
GO
```

Hàm thực thi


SELECT dbo.fn_TinhTuoi('1990-01-01') AS Tuoi;


> **Kết quả thực thi**:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/9a2495ae-e9a4-40ce-b0e0-7b4b0bac9694" />

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/8a0c66af-326a-464d-9111-7932c611bf7a" />
Tạo và thực thi hàm fn_TinhTuoi


#### 2.2.2 Inline Table‑valued Function – Lọc nhân viên theo phòng

**Mục đích:** Trả về danh sách nhân viên thuộc một phòng ban cụ thể. Vì là Inline TVF nên SQL Server có thể tối ưu hóa như một View có tham số – hiệu năng rất tốt khi JOIN với các bảng khác.

| Tham số | Kiểu | Mô tả |
|---------|------|-------|
| `@MaPhong` | `INT` | Mã phòng ban cần lọc |
| **Trả về** | `TABLE` | Bảng gồm các cột `MaNV`, `HoTen`, `NgaySinh` |

```sql
CREATE FUNCTION dbo.fn_NhanVienTheoPhong(@MaPhong INT)
RETURNS TABLE
AS
RETURN
SELECT  [MaNV], [HoTen], [NgaySinh]
FROM    dbo.[NhanVien]
WHERE   [MaPhong] = @MaPhong;
GO

SELECT * FROM dbo.fn_NhanVienTheoPhong(1);

```

> **Kết quả thực thi**:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/a20b3b00-5663-42b9-8471-c1524c671a67" />

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/057be245-1eb7-436f-9943-a8429e688886" />

Tạo và thực thi hàm trả về danh sách nhân viên theo phòng

#### 2.2.3 Multi‑statement Table‑valued Function – Tính thuế thu nhập lũy tiến

**Mục đích:** Tính thuế thu nhập cá nhân theo biểu lũy tiến đơn giản hóa (2 bậc). Hàm trả về bảng kết quả gồm thu nhập và thuế tương ứng của từng mức, giúp minh bạch cách tính thuế.

| Tham số | Kiểu | Mô tả |
|---------|------|-------|
| `@Luong` | `MONEY` | Mức lương cần tính thuế |
| **Trả về** | `TABLE` | Bảng gồm `ThuNhapTungMuc` và `ThueMuc` |

**Luồng xử lý:**
- Nếu lương ≤ 5 triệu → thuế 5% toàn bộ.
- Nếu lương > 5 triệu → 5 triệu đầu chịu 5%, phần còn lại chịu 10%.

```sql
CREATE FUNCTION dbo.fn_TinhThueThuNhap(@Luong MONEY)
RETURNS @Thue TABLE (ThuNhapTungMuc MONEY, ThueMuc MONEY)
AS
BEGIN
    /* Thu nhập tới 5 mil: 5% */
    IF @Luong <= 5000000
        INSERT @Thue VALUES (@Luong, @Luong * 0.05);
    ELSE
    BEGIN
        INSERT @Thue VALUES (5000000, 5000000 * 0.05);
        /* Phần còn lại: 10% */
        INSERT @Thue VALUES (@Luong - 5000000, (@Luong - 5000000) * 0.10);
    END
    RETURN;
END;
GO

SELECT * FROM dbo.fn_TinhThueThuNhap(6000000);
```
> **Kết quả thực thi**:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/5b105989-56e7-49ad-87e3-e8cd1ccbca90" />


<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/b02a5344-894e-4f40-a411-77b79847166f" />

 Kết quả tạo và thực thi hàm fn_TinhThueThuNhap

---



## PHẦN 3: STORED PROCEDURES (SP)



### 3.1 System Stored Procedures (hệ thống)

Trong SQL Server có rất nhiều **Stored Procedure (SP) có sẵn** gọi là **System Stored Procedure**, được chia thành một số nhóm chính như sau:

## 🔹 Các loại System Stored Procedure

- **Metadata SP** (truy vấn thông tin hệ thống):  
  Ví dụ: `sp_help`, `sp_columns`, `sp_tables`  

- **Security SP** (quản lý bảo mật, quyền):  
  Ví dụ: `sp_addlogin`, `sp_adduser`, `sp_addrolemember`  

- **Database Management SP** (quản lý cơ sở dữ liệu):  
  Ví dụ: `sp_rename`, `sp_spaceused`, `sp_databases`  

- **Execution SP** (thực thi lệnh động):  
  Ví dụ: `sp_executesql`  

- **System Monitoring SP** (theo dõi hệ thống):  
  Ví dụ: `sp_who`, `sp_who2`  

---

## Ví dụ 1: sp_help

**Chức năng:**  
Hiển thị thông tin chi tiết của một bảng hoặc đối tượng trong database.

**Cách dùng:**
```sql
sp_help 'ten_bang'
```

**Ví dụ:**
```sql
sp_help 'NhanVien'
```

**Giải thích:**  
Lệnh này sẽ trả về:
- Danh sách các cột trong bảng  
- Kiểu dữ liệu  
- Ràng buộc (constraints)  
- Index  

---


<img width="1914" height="1079" alt="image" src="https://github.com/user-attachments/assets/4548b02d-a204-4fa8-95f7-4936f3085c09" />
Hiển thị thông tin chi tiết của một bảng hoặc đối tượng trong database

## Ví dụ 2: sp_helptext

**Chức năng:**  

Hiển thị nội dung mã nguồn của Stored Procedure, View hoặc Trigger.

**Cách dùng:**
```sql
sp_helptext 'ten_object'
```

**Ví dụ:**
```sql
sp_helptext 'sp_TinhLuong'
```

**Giải thích:**  
Lệnh này giúp:
- Xem lại code đã viết  
- Kiểm tra logic của Stored Procedure  
- Hỗ trợ debug và chỉnh sửa  

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/afdc37e3-ca72-4d26-ab20-d1e15cc0e43c" />

Hiển thị nội dung mã nguồn của Stored Procedure

---

## Kết luận

System Stored Procedure giúp:

- Quản lý database hiệu quả
  
- Kiểm tra cấu trúc và dữ liệu nhanh chóng
  
- Hỗ trợ lập trình và debug hệ thống  


### 3.2 User‑Defined Stored Procedures

#### 3.2.1 SP_ThemNhanVien – Thêm nhân viên mới (có kiểm tra logic)

**Mục đích:** Thêm một nhân viên mới vào hệ thống. Trước khi INSERT, thủ tục kiểm tra phòng ban có tồn tại hay không – nếu không sẽ báo lỗi bằng `RAISERROR` và dừng xử lý, đảm bảo tính toàn vẹn dữ liệu.

| Tham số | Kiểu | Hướng | Mô tả |
|---------|------|-------|-------|
| `@HoTen` | `NVARCHAR(100)` | IN | Họ tên nhân viên |
| `@NgaySinh` | `DATE` | IN | Ngày sinh |
| `@MaPhong` | `INT` | IN | Mã phòng ban nhân viên thuộc về |

```sql
CREATE PROCEDURE dbo.sp_ThemNhanVien
    @HoTen      NVARCHAR(100),
    @NgaySinh   DATE,
    @MaPhong    INT
AS
BEGIN
    SET NOCOUNT ON;

    /* Kiểm tra phòng ban tồn tại */
    IF NOT EXISTS (SELECT 1 FROM dbo.[PhongBan] WHERE [MaPhong] = @MaPhong)
    BEGIN
        RAISERROR (N'Phòng ban không tồn tại.', 16, 1);
        RETURN;
    END;

    INSERT INTO dbo.[NhanVien] ([HoTen], [NgaySinh], [MaPhong])
    VALUES (@HoTen, @NgaySinh, @MaPhong);
END;
GO
```

> **Kết quả thực thi**:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/a6fd31cb-8237-4736-a8e7-f0ec59b4615a" />
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/b5fc2d1d-d6a9-4cff-b974-50399755b729" />

Thực thi sp_ThemNhanVien tự động thêm nhân viên

#### 3.2.2 sp_TongQuyLuong – Tổng quỹ lương (OUTPUT)

**Mục đích:** Tính tổng quỹ lương của toàn công ty trong một tháng cụ thể. Kết quả được trả qua tham số `OUTPUT` thay vì Result Set, giúp chương trình gọi (ứng dụng hoặc script khác) nhận giá trị trực tiếp vào biến.

| Tham số | Kiểu | Hướng | Mô tả |
|---------|------|-------|-------|
| `@Thang` | `DATE` | IN | Tháng cần tính (lấy MONTH và YEAR) |
| `@TongLuong` | `MONEY` | OUTPUT | Tổng lương trả về |

**Luồng xử lý:** Sử dụng `SUM()` trên bảng `[LuongThang]`, lọc theo tháng/năm. Nếu không có dữ liệu, `ISNULL` gán kết quả = 0.

```sql
CREATE PROCEDURE dbo.sp_TongQuyLuong
    @Thang      DATE,
    @TongLuong  MONEY OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT @TongLuong = SUM([Luong])
    FROM dbo.[LuongThang]
    WHERE MONTH([Thang]) = MONTH(@Thang) AND YEAR([Thang]) = YEAR(@Thang);
    
    -- Nếu không có dữ liệu, gán bằng 0
    SET @TongLuong = ISNULL(@TongLuong, 0);
END;
GO

-- Cách gọi:
-- DECLARE @TongTien MONEY;
-- EXEC dbo.sp_TongQuyLuong '2026-05-01', @TongTien OUTPUT;
-- PRINT N'Tổng lương: ' + CAST(@TongTien AS NVARCHAR);
```

> **Kết quả thực thi**:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/53c117b2-e75d-474e-8a76-fd073e4b2bb7" />


<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/9d27bb23-e7b4-4875-838d-f2283a418cba" />

Gọi thủ tục sp_TongQuyLuong và in ra biến OUTPUT

#### 3.2.3 sp_BaoCaoNhanSu – Join 3 bảng để báo cáo

**Mục đích:** Xuất báo cáo nhân sự tổng hợp cho một tháng, bao gồm tên phòng ban, họ tên nhân viên, tuổi (gọi lại `fn_TinhTuoi`) và lương. Kết quả trả về dưới dạng Result Set, sắp xếp theo phòng ban rồi lương giảm dần.

| Tham số | Kiểu | Hướng | Mô tả |
|---------|------|-------|-------|
| `@Thang` | `DATE` | IN | Tháng cần lập báo cáo |

**Luồng xử lý:** JOIN 3 bảng `PhongBan → NhanVien → LuongThang`, lọc theo tháng/năm, gọi hàm `fn_TinhTuoi` để tính tuổi trực tiếp trong SELECT.

```sql
CREATE PROCEDURE dbo.sp_BaoCaoNhanSu
    @Thang DATE
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        pb.[TenPhong],
        nv.[HoTen],
        dbo.fn_TinhTuoi(nv.[NgaySinh]) AS [Tuoi],
        lt.[Luong]
    FROM dbo.[PhongBan] pb
    JOIN dbo.[NhanVien] nv ON pb.[MaPhong] = nv.[MaPhong]
    JOIN dbo.[LuongThang] lt ON nv.[MaNV] = lt.[MaNV]
    WHERE MONTH(lt.[Thang]) = MONTH(@Thang) AND YEAR(lt.[Thang]) = YEAR(@Thang)
    ORDER BY pb.[TenPhong], lt.[Luong] DESC;
END;
GO
```

> **Kết quả thực thi**:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/179105ef-aa1c-4de2-ad7a-52b982aff275" />
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/6342d930-c772-4427-a5be-2492d5055732" />

Kết quả bảng báo cáo từ sp_BaoCaoNhanSu

---



## PHẦN 4: TRIGGER VÀ XỬ LÝ NGHIỆP VỤ



### 4.1 Trigger tự động cập nhật sĩ số
Khi có thao tác INSERT hoặc DELETE trên bảng `[NhanVien]`, Trigger này sẽ tự động điều chỉnh giá trị cột `[SoLuongNhanVien]` trong bảng `[PhongBan]`.

```sql
CREATE TRIGGER trg_CapNhatSiSo
ON dbo.[NhanVien]
AFTER INSERT, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- Tăng số lượng cho phòng vừa có nhân viên mới
    IF EXISTS(SELECT 1 FROM inserted)
    BEGIN
        UPDATE pb
        SET pb.[SoLuongNhanVien] = pb.[SoLuongNhanVien] + 
            (SELECT COUNT(*) FROM inserted i WHERE i.[MaPhong] = pb.[MaPhong])
        FROM dbo.[PhongBan] pb
        JOIN inserted i ON pb.[MaPhong] = i.[MaPhong];
    END

    -- Giảm số lượng cho phòng vừa bị xóa nhân viên
    IF EXISTS(SELECT 1 FROM deleted)
    BEGIN
        UPDATE pb
        SET pb.[SoLuongNhanVien] = pb.[SoLuongNhanVien] - 
            (SELECT COUNT(*) FROM deleted d WHERE d.[MaPhong] = pb.[MaPhong])
        FROM dbo.[PhongBan] pb
        JOIN deleted d ON pb.[MaPhong] = d.[MaPhong];
    END
END;
GO
```

> **Kết quả thực thi**:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/fab486ae-be5a-447d-9572-6eefe7594da3" />

 Thêm 1 nhân viên và SELECT lại bảng PhongBan thấy số lượng tăng 1



###  Thử viết Trigger tạo vòng lặp đệ quy (Circular Trigger)

#### Kịch bản thử nghiệm

Để minh hoạ hiện tượng đệ quy Trigger, ta sẽ tạo hai Trigger hoạt động **ngược chiều nhau**:

| Trigger | Đặt trên bảng | Sự kiện | Hành động |
|---------|--------------|---------|-----------|
| `trg_PhongBan_UpdateNhanVien` | `[PhongBan]` | `AFTER UPDATE` | Khi `SoLuongNhanVien` thay đổi → cập nhật cột `[MaPhong]` ngược lại sang `[NhanVien]` (giả lập: di chuyển toàn bộ NV sang phòng mặc định nếu phòng bị đặt = 0 nhân viên) |
| `trg_NhanVien_CapNhatSiSo` | `[NhanVien]` | `AFTER INSERT, UPDATE, DELETE` | Khi `[NhanVien]` thay đổi → cập nhật lại `SoLuongNhanVien` trong `[PhongBan]` |

Hai trigger này tạo thành **vòng lặp**:  
`INSERT NhanVien` → `trg_NhanVien_CapNhatSiSo` cập nhật `PhongBan` → `trg_PhongBan_UpdateNhanVien` cập nhật `NhanVien` → `trg_NhanVien_CapNhatSiSo` lại kích hoạt → ... vô tận.

---

#### Bước 1 – Tạo Trigger A trên bảng `[PhongBan]`

**Logic:** Khi `SoLuongNhanVien` của một phòng bị cập nhật về `0` (phòng trống), Trigger A tự động chuyển tất cả nhân viên còn sót lại của phòng đó sang `MaPhong = 1` (phòng mặc định).

```sql
-- =============================================
-- TRIGGER A: PhongBan → NhanVien
-- Khi SoLuongNhanVien = 0, chuyển NV về phòng 1
-- =============================================
CREATE TRIGGER trg_PhongBan_UpdateNhanVien
ON dbo.[PhongBan]
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Chỉ xử lý khi SoLuongNhanVien bị cập nhật về 0
    IF UPDATE([SoLuongNhanVien])
    BEGIN
        UPDATE dbo.[NhanVien]
        SET [MaPhong] = 1           -- chuyển về phòng mặc định (MaPhong = 1)
        WHERE [MaPhong] IN (
            SELECT i.[MaPhong]
            FROM inserted  i
            JOIN deleted   d ON i.[MaPhong] = d.[MaPhong]
            WHERE i.[SoLuongNhanVien] = 0
              AND d.[SoLuongNhanVien] > 0  -- chỉ khi vừa giảm về 0
        );
    END
END;
GO
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/43621551-70c7-4540-ac67-db0a6732ef13" />


Tạo Trigger A trên bảng
---

#### Bước 2 – Tạo Trigger B trên bảng `[NhanVien]`

**Logic:** Bất cứ khi nào có hàng được `INSERT`, `UPDATE` hoặc `DELETE` trên `[NhanVien]`, Trigger B sẽ tính lại và cập nhật `SoLuongNhanVien` cho tất cả phòng ban bị ảnh hưởng.

```sql
-- =============================================
-- TRIGGER B: NhanVien → PhongBan
-- Tính lại SoLuongNhanVien sau mọi thay đổi NV
-- =============================================
CREATE TRIGGER trg_NhanVien_CapNhatSiSo
ON dbo.[NhanVien]
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- Gộp danh sách MaPhong bị ảnh hưởng từ cả inserted lẫn deleted
    DECLARE @AffectedPhong TABLE ([MaPhong] INT);

    INSERT INTO @AffectedPhong
    SELECT [MaPhong] FROM inserted
    UNION
    SELECT [MaPhong] FROM deleted;

    -- Cập nhật lại đếm thực tế từ bảng NhanVien
    UPDATE pb
    SET pb.[SoLuongNhanVien] = (
        SELECT COUNT(*)
        FROM dbo.[NhanVien] nv
        WHERE nv.[MaPhong] = pb.[MaPhong]
    )
    FROM dbo.[PhongBan] pb
    WHERE pb.[MaPhong] IN (SELECT [MaPhong] FROM @AffectedPhong);
END;
GO
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/43703960-c82d-45f6-86f5-8fa80f1e3003" />


Tạo Trigger B trên bảng
---

#### Bước 3 – Kích hoạt vòng lặp và quan sát

Thực thi lệnh INSERT để kích hoạt chuỗi đệ quy:

```sql
-- Lệnh này sẽ kích hoạt chuỗi trigger đệ quy
INSERT INTO dbo.[NhanVien] ([HoTen], [NgaySinh], [MaPhong])
VALUES (N'Test Recursive', '2000-01-01', 2);
GO
```

---

> **Kết quả thực thi**:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/73bde0b8-e20b-4682-b485-8e9488ad2d67" />


Thông báo lỗi Msg 217 – Maximum nesting level exceeded khi thực thi INSERT

---

#### Giải thích thông báo lỗi

| Trường | Giá trị | Ý nghĩa |
|--------|---------|---------|
| **Msg** | 217 | Mã lỗi nội bộ của SQL Server |
| **Level** | 16 | Mức độ nghiêm trọng (severity): Lỗi do người dùng gây ra, có thể khắc phục |
| **State** | 1 | Trạng thái phụ dùng để debug nội bộ |
| **Procedure** | `trg_NhanVien_CapNhatSiSo` | Trigger đang thực thi khi tràn ngăn xếp |
| **Nội dung** | *Maximum nesting level exceeded (limit 32)* | Đã chạm đến giới hạn lồng nhau tối đa 32 cấp |

**Diễn giải luồng thực thi:**

```
INSERT INTO NhanVien  (lệnh người dùng)

trg_NhanVien_CapNhatSiSo  → UPDATE PhongBan.SoLuongNhanVien

trg_PhongBan_UpdateNhanVien → UPDATE NhanVien.MaPhong

trg_NhanVien_CapNhatSiSo  → UPDATE PhongBan.SoLuongNhanVien

trg_PhongBan_UpdateNhanVien → UPDATE NhanVien.MaPhong
  ...

SQL Server dừng lại và ném ra Msg 217
```

SQL Server cho phép tối đa **32 cấp lồng nhau** (bao gồm cả stored procedure, function và trigger). Khi chuỗi đệ quy chạm đến cấp 32 mà không có điều kiện dừng, hệ thống buộc phải kết thúc và **rollback toàn bộ transaction** – tức là bản ghi INSERT ban đầu cũng bị huỷ.

####  Nhận xét về hiện tượng đệ quy Trigger

| Vấn đề | Phân tích |
|--------|-----------|
| **Nguyên nhân gốc** | Hai trigger cập nhật qua lại lẫn nhau mà không có điều kiện dừng, tạo ra vòng lặp vô tận |
| **Hậu quả** | SQL Server ném lỗi Msg 217 và rollback toàn bộ transaction; dữ liệu không bị hỏng nhưng thao tác bị từ chối |
| **Giới hạn hệ thống** | Tối đa 32 cấp lồng nhau – đây là cơ chế bảo vệ ngăn tràn call-stack |
| **Rủi ro thực tế** | Trên hệ thống nhiều transaction đồng thời, vòng lặp trigger còn có thể gây **Deadlock** nếu hai session cùng khóa hai bảng theo thứ tự ngược nhau |
| **Khuyến nghị** | (1) Thiết kế trigger theo **một chiều** – tránh cập nhật chéo bảng; (2) Nếu bắt buộc phải cập nhật sang bảng khác, thêm điều kiện `IF UPDATE(col)` và kiểm tra `inserted`/`deleted` để đảm bảo chỉ kích hoạt khi thực sự cần; (3) Dùng **cột cờ** (flag column) hoặc kiểm tra `@@NESTLEVEL` để thoát sớm khi đã ở cấp lồng > 1 |

**Ví dụ kỹ thuật phòng tránh đệ quy bằng `@@NESTLEVEL`:**

```sql
-- Kỹ thuật chống đệ quy: chỉ chạy ở cấp lồng = 1
CREATE TRIGGER trg_Safe_Example
ON dbo.[PhongBan]
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    -- Nếu trigger đang được gọi lại từ một trigger khác thì thoát ngay
    IF @@NESTLEVEL > 1 RETURN;

    -- Logic nghiệp vụ thực sự ở đây...
END;
GO
```
#### Dọn dẹp – Xoá các Trigger thử nghiệm

Sau khi quan sát, cần xoá ngay hai trigger thử nghiệm để tránh ảnh hưởng đến các phần tiếp theo:

```sql
-- Xóa trigger thử nghiệm
DROP TRIGGER IF EXISTS dbo.trg_PhongBan_UpdateNhanVien;
DROP TRIGGER IF EXISTS dbo.trg_NhanVien_CapNhatSiSo;
GO

-- Kiểm tra lại – không còn trigger nào trong danh sách
SELECT name, type_desc, parent_class_desc
FROM sys.triggers
WHERE parent_id IN (
    OBJECT_ID('dbo.PhongBan'),
    OBJECT_ID('dbo.NhanVien')
);
GO
```

> **Kết quả thực thi**:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/38aaf3d9-33b9-4e01-bb4a-d1cef311e359" />

Kết quả SELECT sys.triggers – danh sách rỗng sau khi DROP

> **Kết luận:** Trigger là công cụ mạnh nhưng cần được thiết kế **rất cẩn thận**. Quy tắc an toàn là: mỗi trigger chỉ cập nhật dữ liệu theo **một hướng** (từ bảng con lên bảng cha hoặc ngược lại, không bao giờ cả hai chiều đồng thời trên cùng một cặp bảng).

---



## PHẦN 5: CURSOR VÀ DUYỆT DỮ LIỆU



### 5.1 Sử dụng CURSOR – In thông báo lương cá nhân hoá kèm tính thuế lũy tiến

#### 5.1.1 Bài toán và logic đặt ra

**Yêu cầu nghiệp vụ:** Cuối mỗi tháng, bộ phận kế toán cần in **phiếu lương cá nhân** cho từng nhân viên, bao gồm:
- Họ tên nhân viên
- Tên phòng ban
- Lương gộp (gross)
- Thuế thu nhập cá nhân (TNCN) tính theo biểu lũy tiến
- Lương thực nhận (net = gross – thuế)
- Xếp loại thu nhập: `Thấp` / `Trung bình` / `Cao`

**Biểu thuế lũy tiến áp dụng:**

| Bậc | Thu nhập chịu thuế | Thuế suất |
|-----|-------------------|-----------|
| 1 | ≤ 5.000.000 đ | 5% |
| 2 | 5.000.001 – 10.000.000 đ | 10% trên phần vượt |
| 3 | > 10.000.000 đ | 15% trên phần vượt tiếp theo |

**Tại sao phải dùng CURSOR?** Vì mỗi nhân viên cần một thông điệp `PRINT` **cá nhân hoá riêng biệt**, kèm theo logic rẽ nhánh dựa trên mức lương từng người. Không thể dùng một lệnh `SELECT` hay `UPDATE` thuần tuý để tạo ra từng dòng văn bản khác nhau cho từng người.

---

#### 5.1.2 Code CURSOR đầy đủ

```sql
-- =============================================
-- CURSOR: In phiếu lương cá nhân hóa
-- Duyệt qua từng nhân viên tháng 05/2026
-- =============================================

DECLARE
    @MaNV           INT,
    @TenNV          NVARCHAR(100),
    @TenPhong       NVARCHAR(100),
    @LuongGop       MONEY,
    @Thue           MONEY,
    @LuongNet       MONEY,
    @XepLoai        NVARCHAR(20),
    @ThongBao       NVARCHAR(500),
    @DemNV          INT = 0;

-- Khai báo CURSOR
DECLARE cur_PhieuLuong CURSOR
    LOCAL               -- chỉ dùng trong batch này
    STATIC              -- snapshot dữ liệu tại thời điểm OPEN, tránh dirty read
    READ_ONLY           -- không cần cập nhật qua cursor
    FORWARD_ONLY        -- chỉ duyệt tiến, tối ưu bộ nhớ
FOR
    SELECT
        nv.[MaNV],
        nv.[HoTen],
        pb.[TenPhong],
        lt.[Luong]
    FROM dbo.[NhanVien]  nv
    JOIN dbo.[PhongBan]  pb ON nv.[MaPhong] = pb.[MaPhong]
    JOIN dbo.[LuongThang] lt ON nv.[MaNV]  = lt.[MaNV]
    WHERE MONTH(lt.[Thang]) = 5
      AND YEAR(lt.[Thang])  = 2026
    ORDER BY pb.[TenPhong], nv.[HoTen];

-- ---- Mở Cursor ----
OPEN cur_PhieuLuong;

PRINT REPLICATE(N'=', 60);
PRINT N'       BẢNG THÔNG BÁO LƯƠNG THÁNG 05/2026';
PRINT REPLICATE(N'=', 60);

-- ---- Fetch dòng đầu tiên ----
FETCH NEXT FROM cur_PhieuLuong
    INTO @MaNV, @TenNV, @TenPhong, @LuongGop;

-- ---- Vòng lặp duyệt từng nhân viên ----
WHILE @@FETCH_STATUS = 0
BEGIN
    SET @DemNV = @DemNV + 1;

    -- Bước 1: Tính thuế lũy tiến theo 3 bậc
    SET @Thue = 0;

    IF @LuongGop <= 5000000
    BEGIN
        -- Bậc 1: toàn bộ chịu 5%
        SET @Thue = @LuongGop * 0.05;
    END
    ELSE IF @LuongGop <= 10000000
    BEGIN
        -- Bậc 1 + Bậc 2
        SET @Thue = (5000000 * 0.05)
                  + ((@LuongGop - 5000000) * 0.10);
    END
    ELSE
    BEGIN
        -- Bậc 1 + Bậc 2 + Bậc 3
        SET @Thue = (5000000  * 0.05)
                  + (5000000  * 0.10)
                  + ((@LuongGop - 10000000) * 0.15);
    END

    -- Bước 2: Tính lương thực nhận
    SET @LuongNet = @LuongGop - @Thue;

    -- Bước 3: Xếp loại thu nhập
    SET @XepLoai =
        CASE
            WHEN @LuongGop < 10000000  THEN N'Thấp'
            WHEN @LuongGop < 12000000  THEN N'Trung bình'
            ELSE                            N'Cao'
        END;

    -- Bước 4: In phiếu lương cá nhân
    PRINT REPLICATE(N'-', 60);
    SET @ThongBao = N'[' + CAST(@DemNV AS NVARCHAR) + N'] '
                  + @TenNV + N'  |  ' + @TenPhong;
    PRINT @ThongBao;

    PRINT N'   Lương gộp    : '
        + FORMAT(@LuongGop, N'#,##0') + N' VND';

    PRINT N'   Thuế TNCN    : '
        + FORMAT(@Thue,     N'#,##0') + N' VND';

    PRINT N'   Lương thực nhận: '
        + FORMAT(@LuongNet, N'#,##0') + N' VND';

    PRINT N'   Xếp loại     : ' + @XepLoai;

    -- Bước 5: In cảnh báo riêng nếu lương thấp
    IF @XepLoai = N'Thấp'
        PRINT N'   ⚠ Lưu ý: Nhân viên thuộc diện xem xét hỗ trợ phúc lợi.';

    -- Fetch dòng tiếp theo
    FETCH NEXT FROM cur_PhieuLuong
        INTO @MaNV, @TenNV, @TenPhong, @LuongGop;
END

-- ---- Đóng và giải phóng CURSOR ----
CLOSE     cur_PhieuLuong;
DEALLOCATE cur_PhieuLuong;

PRINT REPLICATE(N'=', 60);
PRINT N'Đã xử lý: ' + CAST(@DemNV AS NVARCHAR) + N' nhân viên.';
PRINT REPLICATE(N'=', 60);
GO
```

---

#### 5.1.3 Phân tích logic từng bước

| Bước | Thao tác | Mục đích |
|------|----------|----------|
| **Khai báo biến** | `DECLARE @MaNV, @TenNV, ...` | Chứa dữ liệu từng dòng khi FETCH |
| **Khai báo Cursor** | `LOCAL STATIC READ_ONLY FORWARD_ONLY` | Tối ưu bộ nhớ; `STATIC` đảm bảo dữ liệu nhất quán trong suốt vòng lặp |
| **OPEN** | `OPEN cur_PhieuLuong` | SQL Server thực thi câu SELECT và lưu kết quả vào temporary storage |
| **FETCH NEXT** | Trước vòng lặp và cuối vòng lặp | Lấy một dòng vào biến; `@@FETCH_STATUS = 0` nghĩa là fetch thành công |
| **Tính thuế** | `IF @LuongGop <= 5M ... ELSE IF ...` | Logic rẽ nhánh riêng cho từng nhân viên – không thể viết trong một `UPDATE` thuần tuý |
| **PRINT** | Nhiều lệnh `PRINT` khác nhau | Tạo output văn bản cá nhân hoá từng người |
| **CLOSE / DEALLOCATE** | Sau vòng lặp | **Bắt buộc** – giải phóng tài nguyên khóa bảng và bộ nhớ |

> **Kết quả thực thi**:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/6443a2d2-93bb-47ba-8471-8222b04cfc14" />
Tab Messages trong SSMS hiển thị danh sách phiếu lương cá nhân hoá với thuế và xếp loại cho từng nhân viên

---

### 5.2 Giải quyết bài toán tương tự KHÔNG dùng CURSOR

#### 5.2.1 Bài toán tương đương – Tăng lương và gắn xếp loại

Giả sử bài toán đơn giản hơn: **Cập nhật bảng phụ `LuongXepLoai` (tạo mới) ghi nhận xếp loại thu nhập và lương sau thuế của từng nhân viên tháng 05/2026**, không cần in từng dòng ra màn hình.

**Tạo bảng phụ để lưu kết quả:**

```sql
-- Tạo bảng tạm để lưu kết quả xếp loại
CREATE TABLE dbo.[LuongXepLoai] (
    [MaNV]      INT          NOT NULL,
    [Thang]     DATE         NOT NULL,
    [LuongGop]  MONEY        NOT NULL,
    [Thue]      MONEY        NOT NULL,
    [LuongNet]  MONEY        NOT NULL,
    [XepLoai]   NVARCHAR(20) NOT NULL,
    PRIMARY KEY ([MaNV], [Thang])
);
GO
```

**Giải quyết bằng Set-based SQL (không dùng CURSOR):**

```sql
-- =============================================
-- SET-BASED: Tính thuế + xếp loại cho TẤT CẢ
-- nhân viên chỉ bằng một lệnh INSERT...SELECT
-- =============================================
INSERT INTO dbo.[LuongXepLoai] ([MaNV], [Thang], [LuongGop], [Thue], [LuongNet], [XepLoai])
SELECT
    lt.[MaNV],
    lt.[Thang],
    lt.[Luong]                              AS [LuongGop],

    -- Tính thuế lũy tiến bằng CASE WHEN (set-based)
    CASE
        WHEN lt.[Luong] <= 5000000
            THEN lt.[Luong] * 0.05
        WHEN lt.[Luong] <= 10000000
            THEN (5000000 * 0.05) + ((lt.[Luong] - 5000000) * 0.10)
        ELSE
            (5000000 * 0.05) + (5000000 * 0.10) + ((lt.[Luong] - 10000000) * 0.15)
    END                                     AS [Thue],

    -- Lương thực nhận
    lt.[Luong] -
    CASE
        WHEN lt.[Luong] <= 5000000
            THEN lt.[Luong] * 0.05
        WHEN lt.[Luong] <= 10000000
            THEN (5000000 * 0.05) + ((lt.[Luong] - 5000000) * 0.10)
        ELSE
            (5000000 * 0.05) + (5000000 * 0.10) + ((lt.[Luong] - 10000000) * 0.15)
    END                                     AS [LuongNet],

    -- Xếp loại
    CASE
        WHEN lt.[Luong] < 10000000  THEN N'Thấp'
        WHEN lt.[Luong] < 12000000  THEN N'Trung bình'
        ELSE                             N'Cao'
    END                                     AS [XepLoai]

FROM dbo.[LuongThang] lt
WHERE MONTH(lt.[Thang]) = 5
  AND YEAR(lt.[Thang])  = 2026;
GO

-- Kiểm tra kết quả
SELECT
    nv.[HoTen],
    pb.[TenPhong],
    lx.[LuongGop],
    lx.[Thue],
    lx.[LuongNet],
    lx.[XepLoai]
FROM dbo.[LuongXepLoai] lx
JOIN dbo.[NhanVien]     nv ON lx.[MaNV]    = nv.[MaNV]
JOIN dbo.[PhongBan]     pb ON nv.[MaPhong] = pb.[MaPhong]
ORDER BY pb.[TenPhong], lx.[LuongGop] DESC;
GO
```

> **Kết quả thực thi**:

<img width="1917" height="1079" alt="image" src="https://github.com/user-attachments/assets/7aa4da21-8954-4413-b594-90588c414b20" />

Bảng kết quả SELECT hiển thị đầy đủ HoTen, TenPhong, LuongGop, Thue, LuongNet, XepLoai cho 40 nhân viên

---

#### 5.2.2 So sánh tốc độ thực thi CURSOR vs Set-based

Để đo thời gian, bật **Statistics Time và IO** trước khi chạy cả hai đoạn code:

```sql
-- Bật đo thời gian và I/O
SET STATISTICS TIME ON;
SET STATISTICS IO ON;
GO

-- ---- Chạy đoạn CURSOR (lặp 40 dòng) ----
-- (dán lại toàn bộ đoạn code CURSOR ở mục 5.1.2 vào đây)
-- ...

-- ---- Chạy đoạn Set-based ----
TRUNCATE TABLE dbo.[LuongXepLoai]; -- reset dữ liệu
INSERT INTO dbo.[LuongXepLoai] (...) SELECT ...;  -- (code ở trên)
GO

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```

**Kết quả dự kiến (với 40 bản ghi):**

| Phương pháp | CPU time (ms) | Elapsed time (ms) | Logical reads |
|-------------|:------------:|:-----------------:|:-------------:|
| **CURSOR** | ~15–30 ms | ~20–50 ms | ~80–160 reads (40 lần đọc riêng lẻ) |
| **Set-based** | ~1–3 ms | ~2–5 ms | ~8–12 reads (1 lần quét bảng) |

> **Kết quả thực thi**:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/754817d3-3767-4819-94be-5131b026d5da" />

Tab Messages trong SSMS hiển thị kết quả STATISTICS TIME và IO – so sánh rõ CPU time và Logical reads giữa CURSOR và Set-based


**Kết luận so sánh:**
- Trên 40 bản ghi, CURSOR đã chậm hơn Set-based khoảng **5–15 lần** về elapsed time.
- Trên tập dữ liệu lớn (hàng chục ngàn bản ghi), khoảng cách này có thể lên đến **50–100 lần**, vì CURSOR không thể tận dụng tối ưu hóa song song (parallelism) của SQL Server.
- **Nguyên tắc:** Nếu bài toán có thể biểu diễn bằng một câu SQL tập hợp (Set-based), luôn ưu tiên cách đó.

---

### 5.3 Bài toán chỉ CURSOR mới giải quyết được

#### 5.3.1 Bài toán: Gửi email thông báo lương cá nhân qua Database Mail

**Mô tả:**  
Sau khi chốt lương tháng 05/2026, hệ thống phải **tự động gửi email** đến địa chỉ email cá nhân của **từng nhân viên**, nội dung email là phiếu lương riêng (họ tên, phòng, lương gộp, thuế, lương net). Mỗi email phải được **render riêng** với tên người nhận và số liệu riêng.

**Tại sao Set-based SQL không làm được?**  
Gửi email đòi hỏi gọi stored procedure `msdb.dbo.sp_send_dbmail` với tham số khác nhau cho từng người. Không có lệnh `UPDATE` hay `INSERT` nào có thể gọi SP bên ngoài cho từng dòng. Đây là trường hợp **bắt buộc** phải dùng CURSOR hoặc vòng lặp `WHILE`.

---

#### 5.3.2 Chuẩn bị – Thêm cột Email vào bảng NhanVien

```sql
-- Thêm cột Email vào NhanVien (nếu chưa có)
ALTER TABLE dbo.[NhanVien]
    ADD [Email] NVARCHAR(200) NULL;
GO

-- Cập nhật email mẫu cho 40 nhân viên
UPDATE dbo.[NhanVien]
SET [Email] = LOWER(
        REPLACE([HoTen], N' ', N'.') + N'@company.vn'
    );
GO

-- Kiểm tra
SELECT TOP 5 [MaNV], [HoTen], [Email] FROM dbo.[NhanVien];
GO
```

> **Kết quả thực thi**:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/1d42e78b-68a7-44a8-83ff-a238aa58268c" />

SELECT TOP 5 hiển thị MaNV, HoTen, Email – mỗi nhân viên có địa chỉ email riêng dạng ten.ho@company.vn

---

#### 5.3.3 Code CURSOR gửi email cá nhân hóa

```sql
-- =============================================
-- CURSOR: Gửi email thông báo lương cá nhân
-- Chỉ CURSOR (hoặc WHILE loop) mới làm được!
-- =============================================

DECLARE
    @MaNV       INT,
    @TenNV      NVARCHAR(100),
    @Email      NVARCHAR(200),
    @TenPhong   NVARCHAR(100),
    @LuongGop   MONEY,
    @Thue       MONEY,
    @LuongNet   MONEY,
    @NoiDung    NVARCHAR(MAX),
    @TieuDe     NVARCHAR(300),
    @DemGui     INT = 0,
    @DemLoi     INT = 0;

DECLARE cur_GuiEmail CURSOR
    LOCAL STATIC READ_ONLY FORWARD_ONLY
FOR
    SELECT
        nv.[MaNV],
        nv.[HoTen],
        nv.[Email],
        pb.[TenPhong],
        lt.[Luong]
    FROM dbo.[NhanVien]   nv
    JOIN dbo.[PhongBan]   pb ON nv.[MaPhong] = pb.[MaPhong]
    JOIN dbo.[LuongThang] lt ON nv.[MaNV]    = lt.[MaNV]
    WHERE MONTH(lt.[Thang]) = 5
      AND YEAR(lt.[Thang])  = 2026
      AND nv.[Email] IS NOT NULL;     -- chỉ gửi cho NV có email

OPEN cur_GuiEmail;
FETCH NEXT FROM cur_GuiEmail
    INTO @MaNV, @TenNV, @Email, @TenPhong, @LuongGop;

WHILE @@FETCH_STATUS = 0
BEGIN
    BEGIN TRY
        -- Tính thuế lũy tiến (logic riêng từng người)
        SET @Thue =
            CASE
                WHEN @LuongGop <= 5000000
                    THEN @LuongGop * 0.05
                WHEN @LuongGop <= 10000000
                    THEN (5000000 * 0.05) + ((@LuongGop - 5000000) * 0.10)
                ELSE
                    (5000000 * 0.05) + (5000000 * 0.10)
                    + ((@LuongGop - 10000000) * 0.15)
            END;

        SET @LuongNet = @LuongGop - @Thue;

        -- Render nội dung email HTML cá nhân hoá
        SET @TieuDe  = N'[Thông báo lương] Tháng 05/2026 – ' + @TenNV;
        SET @NoiDung = N'<html><body>'
            + N'<p>Kính gửi <strong>' + @TenNV + N'</strong>,</p>'
            + N'<p>Phòng: <em>' + @TenPhong + N'</em></p>'
            + N'<table border="1" cellpadding="5">'
            + N'<tr><td>Lương gộp</td><td align="right">'
                + FORMAT(@LuongGop, N'#,##0') + N' VND</td></tr>'
            + N'<tr><td>Thuế TNCN</td><td align="right">'
                + FORMAT(@Thue,     N'#,##0') + N' VND</td></tr>'
            + N'<tr><td><strong>Lương thực nhận</strong></td>'
                + N'<td align="right"><strong>'
                + FORMAT(@LuongNet, N'#,##0') + N' VND</strong></td></tr>'
            + N'</table>'
            + N'<p>Trân trọng,<br/>Phòng Kế Toán</p>'
            + N'</body></html>';

        -- Gọi Database Mail để gửi email cho từng người
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name  = N'CompanyMailProfile',  -- tên profile Database Mail
            @recipients    = @Email,
            @subject       = @TieuDe,
            @body          = @NoiDung,
            @body_format   = N'HTML';

        SET @DemGui = @DemGui + 1;
        PRINT N'✓ Đã gửi email cho: ' + @TenNV + N' (' + @Email + N')';

    END TRY
    BEGIN CATCH
        SET @DemLoi = @DemLoi + 1;
        PRINT N'✗ Lỗi gửi email cho: ' + @TenNV
            + N' | Error: ' + ERROR_MESSAGE();
        -- Ghi log lỗi vào bảng nếu cần
    END CATCH;

    FETCH NEXT FROM cur_GuiEmail
        INTO @MaNV, @TenNV, @Email, @TenPhong, @LuongGop;
END

CLOSE     cur_GuiEmail;
DEALLOCATE cur_GuiEmail;

PRINT N'--- Kết quả gửi email ---';
PRINT N'Thành công : ' + CAST(@DemGui  AS NVARCHAR) + N' email';
PRINT N'Thất bại   : ' + CAST(@DemLoi AS NVARCHAR) + N' email';
GO
```

> **Kết quả thực thi**:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/4fe2eb60-6c9b-480d-8905-6903667b14f7" />

Tab Messages hiển thị từng dòng "✓ Đã gửi email cho: [Tên]" cho 40 nhân viên và tổng kết cuối cùng


---

#### 5.3.4 Phân tích – Tại sao Set-based SQL không thể thay thế?

```sql
-- Thử viết Set-based để gửi email – KHÔNG THỂ LÀM ĐƯỢC
-- Ví dụ sai này minh hoạ giới hạn của Set-based:

SELECT
    nv.[Email],
    N'Lương của bạn: ' + CAST(lt.[Luong] AS NVARCHAR) AS [NoiDung],
    -- ❌ Không có cách nào gọi sp_send_dbmail trong SELECT!
    -- ❌ sp_send_dbmail là Stored Procedure, không phải Function
    -- ❌ SQL Server không cho phép gọi SP trong SELECT/UPDATE/INSERT
    msdb.dbo.sp_send_dbmail(...)   -- CÚ PHÁP NÀY KHÔNG HỢP LỆ
FROM dbo.[NhanVien] nv
JOIN dbo.[LuongThang] lt ON nv.[MaNV] = lt.[MaNV];
```

**Lý do CURSOR là bắt buộc:**

| Ràng buộc | Giải thích |
|-----------|-----------|
| `sp_send_dbmail` là Stored Procedure | Chỉ được gọi bằng `EXEC`, không thể dùng trong `SELECT`, `UPDATE`, `WHERE` |
| Mỗi email có nội dung HTML khác nhau | Cần biến chuỗi riêng cho từng người – không thể vector hoá |
| Cần xử lý lỗi độc lập từng email | `BEGIN TRY / CATCH` bên trong vòng lặp; nếu 1 email lỗi vẫn tiếp tục gửi cho người khác |
| Cần đếm thành công / thất bại | Biến đếm `@DemGui`, `@DemLoi` chỉ cập nhật được trong vòng lặp |

---

#### 5.3.5 Tổng kết – Khi nào dùng CURSOR, khi nào dùng Set-based

| Tiêu chí | Set-based SQL | CURSOR |
|----------|:------------:|:------:|
| Tốc độ xử lý dữ liệu lớn | ✅ Rất nhanh | ❌ Chậm (Row-by-Row) |
| Cập nhật hàng loạt (INSERT/UPDATE/DELETE) | ✅ Tối ưu | ❌ Không cần thiết |
| Tính toán phức tạp cùng loại cho mọi dòng | ✅ Dùng CASE/CROSS APPLY | ✅ Cũng làm được |
| Gọi Stored Procedure riêng cho từng dòng | ❌ Không thể | ✅ Bắt buộc |
| Gửi email / gọi API bên ngoài từng dòng | ❌ Không thể | ✅ Bắt buộc |
| In văn bản cá nhân hoá từng dòng (PRINT) | ❌ Không thể | ✅ Bắt buộc |
| Xử lý lỗi độc lập từng dòng | ❌ Khó | ✅ TRY/CATCH trong vòng lặp |
| Bài toán phụ thuộc kết quả dòng trước | ❌ Rất khó | ✅ Có thể dùng biến tích luỹ |

> **Kết luận:** CURSOR không phải lựa chọn hiệu quả cho bài toán xử lý tập hợp dữ liệu thuần tuý, nhưng là **công cụ không thể thiếu** khi logic yêu cầu tương tác bên ngoài SQL (gọi API, gửi email, ghi file, gọi OS), xử lý tuần tự có trạng thái, hoặc in output cá nhân hoá từng dòng.
