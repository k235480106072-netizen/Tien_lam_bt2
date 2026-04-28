# BÀI KIỂM TRA SỐ 2 – HỆ QUẢN TRỊ CSDL SQL SERVER

**Họ và tên:** Vi Trần Tiến

**Mã sinh viên:** K235480106072

**Lớp:** K59KMT.K01

**Đề tài:** Hệ thống **Quản lý nhân sự** 
**Ngày nộp:** 03/05/2026

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
| SP | Mô tả |
|----|-------|
| **sp_help** | Hiển thị cấu trúc, cột, ràng buộc của một đối tượng (bảng, view, …). |
| **sp_rename** | Đổi tên một đối tượng (bảng, cột, …). |
| **sp_who** | Kiểm tra các session đang kết nối tới SQL Server. |



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
<img width="1917" height="1079" alt="image" src="https://github.com/user-attachments/assets/ab1cc450-e75b-4e26-87dc-02852ac2abe6" />

> ![Ảnh chụp màn hình: Thêm 1 nhân viên và SELECT lại bảng PhongBan thấy số lượng tăng 1]



### 4.2 Hiện tượng đệ quy (Recursive Trigger)
**Kịch bản**: 
Tôi đã thử tạo Trigger A trên bảng `[PhongBan]` tự động cập nhật lại bảng `[NhanVien]` (ví dụ: set trạng thái nghỉ việc nếu phòng đóng cửa), và Trigger B trên `[NhanVien]` (như trên) lại cập nhật số lượng của `[PhongBan]`.

**Kết quả**: 
SQL Server báo lỗi vượt quá giới hạn mức lồng (Maximum nesting level exceeded) do vòng lặp vô tận: A gọi B -> B gọi A -> A gọi B... (Tối đa 32 mức lồng trong SQL Server).

**Nhận xét**: 
Cần hết sức cẩn thận khi thiết kế Trigger. Hạn chế tối đa việc thiết kế các Trigger chéo nhau (vòng lặp) để tránh gây treo hệ thống (Deadlock) và tràn bộ nhớ.

> **Kết quả thực thi**:
> ![Ảnh chụp màn hình: Thông báo lỗi đệ quy Trigger (Maximum nesting level exceeded)]

---



## PHẦN 5: CURSOR VÀ DUYỆT DỮ LIỆU



### 5.1 Sử dụng Cursor duyệt và in thông báo
**Mục đích:** Duyệt từng nhân viên kèm mức lương hiện tại để in thông báo cá nhân hóa qua lệnh `PRINT`. Đây là tình huống điển hình mà Cursor phù hợp – khi cần xử lý logic riêng biệt cho từng dòng (ví dụ: gửi email, ghi log, v.v.).

**Luồng xử lý:**
1. Khai báo biến `@TenNV` và `@LuongHienTai` để chứa dữ liệu từng dòng.
2. Khai báo Cursor `cur_ThongBaoLuong` lấy dữ liệu từ JOIN `NhanVien` và `LuongThang`.
3. Mở Cursor → `FETCH NEXT` vào biến → vòng lặp `WHILE @@FETCH_STATUS = 0` in thông báo → `FETCH NEXT` tiếp.
4. Đóng (`CLOSE`) và giải phóng (`DEALLOCATE`) Cursor.

```sql
DECLARE @TenNV NVARCHAR(100);
DECLARE @LuongHienTai MONEY;

-- Khai báo Cursor
DECLARE cur_ThongBaoLuong CURSOR FOR
    SELECT nv.[HoTen], lt.[Luong]
    FROM dbo.[NhanVien] nv
    JOIN dbo.[LuongThang] lt ON nv.[MaNV] = lt.[MaNV];

OPEN cur_ThongBaoLuong;
FETCH NEXT FROM cur_ThongBaoLuong INTO @TenNV, @LuongHienTai;

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT N'Chào ' + @TenNV + N', mức lương của bạn tháng này là: ' + CAST(@LuongHienTai AS NVARCHAR);
    
    FETCH NEXT FROM cur_ThongBaoLuong INTO @TenNV, @LuongHienTai;
END

CLOSE cur_ThongBaoLuong;
DEALLOCATE cur_ThongBaoLuong;
```

> **Kết quả thực thi**:
<img width="1917" height="1079" alt="image" src="https://github.com/user-attachments/assets/15528b7f-a5b4-4edc-9357-686d70bccbef" />

Tab Messages hiển thị dòng chữ Chào... mức lương của bạn là...



### 5.2 So sánh với Set-based (Lệnh SQL thông thường)
Giả sử bài toán là: **Tăng 10% lương cho tất cả nhân viên.**

**Dùng Cursor**:
Phải mở con trỏ, lặp qua từng dòng, đọc dữ liệu, cập nhật, rồi chuyển sang dòng tiếp theo (Row-by-Agonizing-Row).

**Dùng Set-based (UPDATE)**:
```sql
UPDATE dbo.[LuongThang]
SET [Luong] = [Luong] * 1.1;
```
![Uploading image.png…]()


**So sánh hiệu năng**:
- **Tốc độ**: Lệnh `UPDATE` (Set-based) xử lý toàn bộ tập hợp dữ liệu cùng một lúc, nhanh hơn Cursor từ 10-100 lần trên tập dữ liệu lớn vì nó tối ưu hóa I/O và Transaction Log.
- **Tài nguyên**: Cursor khóa tài nguyên lâu hơn và tốn nhiều RAM/CPU để duy trì trạng thái từng dòng.

**Khi nào nên dùng Cursor?**
Nên dùng Cursor khi bài toán **không thể giải quyết bằng SQL Set-based**, ví dụ:
- Gọi API bên ngoài (External Stored Procedure) cho từng dòng dữ liệu.
- Bài toán gửi Email cá nhân hóa (đọc thông tin người dùng, render template mail riêng, đính kèm file PDF hóa đơn lương khác biệt cho từng người rồi gọi hàm Database Mail gửi đi). Trong trường hợp này, `UPDATE` không thể gửi email, bắt buộc phải dùng vòng lặp / Cursor.

> **Kết quả thực thi**:
> ![Ảnh chụp màn hình: So sánh thời gian thực thi (Execution Plan) giữa Cursor và SQL Command]
