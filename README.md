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


```sql
/* Bảng PhongBan */
CREATE TABLE [PhongBan] (
    [MaPhong]          INT          PRIMARY KEY,
    [TenPhong]         NVARCHAR(100) NOT NULL,
    [SoLuongNhanVien]  INT           DEFAULT (0)   -- sẽ được Trigger cập nhật
);
GO
```

/* Bảng NhanVien */
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

/* Bảng LuongThang */
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

> **Kết quả thực thi**:






## PHẦN 2: FUNCTON (Hàm)



### 2.1 Built‑in Functions (các hàm có sẵn)
| Hàm | Mô tả | Ví dụ |
|-----|-------|-------|
| **GETDATE()** | Trả về ngày‑giờ hiện tại của server | `SELECT GETDATE();` |
| **LEN(string)** | Độ dài chuỗi ký tự | `SELECT LEN(N'Hello');` |
| **ROUND(number, d)** | Làm tròn số tới *d* chữ số thập phân | `SELECT ROUND(123.4567,2);` |
| **COALESCE(expr1,expr2, …)** | Trả về giá trị không NULL đầu tiên | `SELECT COALESCE(NULL, 5, 10);` |



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

> **Kết quả thực thi**:
> ![Ảnh chụp màn hình: Kết quả hàm fn_TinhTuoi]

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
```

> **Kết quả thực thi**:
> ![Ảnh chụp màn hình: Kết quả fn_NhanVienTheoPhong]

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
```

> **Kết quả thực thi**:
> ![Ảnh chụp màn hình: Kết quả fn_TinhThueThuNhap]

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
> ![Ảnh chụp màn hình: Thực thi sp_ThemNhanVien]

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
> ![Ảnh chụp màn hình: Gọi thủ tục sp_TongQuyLuong và in ra biến OUTPUT]

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
> ![Ảnh chụp màn hình: Kết quả bảng báo cáo từ sp_BaoCaoNhanSu]

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
> ![Ảnh chụp màn hình: Tab Messages hiển thị dòng chữ Chào... mức lương của bạn là...]



### 5.2 So sánh với Set-based (Lệnh SQL thông thường)
Giả sử bài toán là: **Tăng 10% lương cho tất cả nhân viên.**

**Dùng Cursor**:
Phải mở con trỏ, lặp qua từng dòng, đọc dữ liệu, cập nhật, rồi chuyển sang dòng tiếp theo (Row-by-Agonizing-Row).

**Dùng Set-based (UPDATE)**:
```sql
UPDATE dbo.[LuongThang]
SET [Luong] = [Luong] * 1.1;
```

**So sánh hiệu năng**:
- **Tốc độ**: Lệnh `UPDATE` (Set-based) xử lý toàn bộ tập hợp dữ liệu cùng một lúc, nhanh hơn Cursor từ 10-100 lần trên tập dữ liệu lớn vì nó tối ưu hóa I/O và Transaction Log.
- **Tài nguyên**: Cursor khóa tài nguyên lâu hơn và tốn nhiều RAM/CPU để duy trì trạng thái từng dòng.

**Khi nào nên dùng Cursor?**
Nên dùng Cursor khi bài toán **không thể giải quyết bằng SQL Set-based**, ví dụ:
- Gọi API bên ngoài (External Stored Procedure) cho từng dòng dữ liệu.
- Bài toán gửi Email cá nhân hóa (đọc thông tin người dùng, render template mail riêng, đính kèm file PDF hóa đơn lương khác biệt cho từng người rồi gọi hàm Database Mail gửi đi). Trong trường hợp này, `UPDATE` không thể gửi email, bắt buộc phải dùng vòng lặp / Cursor.

> **Kết quả thực thi**:
> ![Ảnh chụp màn hình: So sánh thời gian thực thi (Execution Plan) giữa Cursor và SQL Command]